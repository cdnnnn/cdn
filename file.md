//skills.js
// Skills: record a task once (by doing it, or save a successful agent run), then replay it fast and deterministically.
// Loaded after sidepanel.js (uses its globals: $, T, i18n, addMessage, requestApproval, settle, takeSnapshot, ...).
// Page-side helpers (recorderFn, replayStepFn, pageSigFn, fetchFn) live in inpage.js.

let SKILLS = [];
let recording = null; // { tabId, host, startUrl, timer, onUpdated } while the user is recording
let actionLog = []; // successful actions of the current agent run (so a good run can be saved as a skill)
let lastRequestText = "";
let skillRunUsed = false;
let agentHost = "";

const hostOf = (u) => { try { return new URL(u).host; } catch (e) { return ""; } };
const stripHash = (u) => String(u || "").split("#")[0];
const slug = (s) => (String(s || "").toLowerCase().replace(/[^a-z0-9À-￿]+/g, "_").replace(/^_+|_+$/g, "") || "value").slice(0, 30);

async function skillsLoad() {
  try { const { skills } = await chrome.storage.local.get("skills"); SKILLS = Array.isArray(skills) ? skills : []; } catch (e) { SKILLS = []; }
  return SKILLS;
}
async function skillsSave() {
  try { await chrome.storage.local.set({ skills: SKILLS }); } catch (e) {}
  renderSkills();
}
const skillsForHost = (h) => SKILLS.filter((s) => s.host === h);

// ---------- what the model is told ----------
function skillsPrompt() {
  const list = [...skillsForHost(agentHost), ...SKILLS.filter((s) => s.host !== agentHost)].slice(0, 25);
  if (!list.length) return "";
  return (
    "\n\nSaved skills. When one matches the request, call run_skill with its name and ALL its parameters (take values from the user's request; use the default if the user gave none). It replays a recorded, proven path and is faster and more reliable than doing the steps yourself. A skill for another site runs automatically in that site's tab (it is found or opened for you), so you may use it from any tab. If a skill fails, continue manually from where it stopped.\n" +
    list.map((s) => `- ${s.name}${s.host !== agentHost ? ` [site ${s.host}]` : ""}${s.description ? ": " + s.description : ""}. Parameters: ${s.params.length ? s.params.map((p) => `${p.name}${p.label ? ` (${p.label})` : ""}${p.default ? ` default "${String(p.default).slice(0, 40)}"` : ""}`).join(", ") : "none"}`).join("\n")
  );
}
const RUN_SKILL_TOOL = {
  name: "run_skill",
  description: "Run a saved skill (a recorded sequence of steps for this site). Give the skill name and a params object with a value for each parameter.",
  input_schema: { type: "object", properties: { name: { type: "string" }, params: { type: "object" } }, required: ["name"] },
};
const skillTools = () => (SKILLS.length ? [RUN_SKILL_TOOL] : []);

// ---------- describing steps ----------
const lbl = (d) => (d && (d.dynValue || d.label || d.placeholder || d.name || d.id || d.testid)) || (d && d.tag) || "";
function stepText(st, val) {
  const label = lbl(st.d);
  const v = val != null ? val : st.value;
  if (st.t === "click") return i18n("sk.click", { label });
  if (st.t === "type") return st.sensitive ? i18n("sk.typeSecret", { label }) : i18n("sk.type", { value: v, label });
  if (st.t === "select") return i18n("sk.select", { value: v, label });
  if (st.t === "key") return i18n("sk.key", { key: st.key, label });
  return st.t;
}
const stepIsFinal = (st) => st.t === "click" ? (isRiskyLabel(lbl(st.d)) || (st.d && st.d.type === "submit")) : st.t === "key" && st.key === "Enter";

// ---------- recording ----------
function recBarShow(n) {
  const bar = $("recBar");
  bar.hidden = false;
  $("recText").textContent = i18n("sk.recording", { n: n || 0 });
}
async function startRecording() {
  if (running || recording) return;
  const tab = await getActiveTab();
  if (!tab || !/^https?:/.test(tab.url || "")) { addMessage("err", i18n("err.unreadable")); return; }
  const hint = document.querySelector(".hint");
  if (hint) hint.remove();
  agentTabId = tab.id;
  recording = { tabId: tab.id, host: hostOf(tab.url), startUrl: tab.url };
  try {
    await chrome.scripting.executeScript({ target: { tabId: tab.id }, func: recorderFn, args: [{ cmd: "start" }], world: "MAIN" });
  } catch (e) { recording = null; addMessage("err", e.message); return; }
  recording.onUpdated = (id, info) => {
    if (recording && id === recording.tabId && info.status === "complete") {
      chrome.scripting.executeScript({ target: { tabId: id }, func: recorderFn, args: [{ cmd: "start", resume: true }], world: "MAIN" }).catch(() => {});
    }
  };
  chrome.tabs.onUpdated.addListener(recording.onUpdated);
  recording.timer = setInterval(async () => {
    try {
      const [r] = await chrome.scripting.executeScript({ target: { tabId: recording.tabId }, func: recorderFn, args: [{ cmd: "dump" }], world: "MAIN" });
      recBarShow(r.result && r.result.steps ? r.result.steps.length : 0);
    } catch (e) {}
  }, 1500);
  recBarShow(0);
  addMessage("note", i18n("sk.recStarted"));
}

async function stopRecording() {
  if (!recording) return;
  const rec = recording;
  recording = null;
  clearInterval(rec.timer);
  try { chrome.tabs.onUpdated.removeListener(rec.onUpdated); } catch (e) {}
  $("recBar").hidden = true;
  let data = null;
  try {
    const tid = rec.tabId;
    const [r] = await chrome.scripting.executeScript({ target: { tabId: tid }, func: recorderFn, args: [{ cmd: "stop" }], world: "MAIN" });
    data = r.result;
  } catch (e) { addMessage("err", e.message); return; }
  if (!data || !data.steps || !data.steps.length) { addMessage("note", i18n("sk.recEmpty")); return; }
  const draft = draftFromRecording(data, rec);
  T("skill-recorded", { steps: draft.steps.length, params: draft.params.length, api: !!draft.api });
  openSkillEditor(draft);
}

// Turn the recorded events into a skill draft: parameters from typed values, plus an optional direct-request fast path.
function draftFromRecording(data, rec) {
  const steps = data.steps.map((s) => ({ t: s.t, d: s.d, value: s.value, key: s.key, sensitive: !!s.sensitive, effect: s.effect, at: s.at, url: s.url }));
  const params = [];
  const used = new Set();
  for (const st of steps) {
    if ((st.t === "type" || st.t === "select") && !st.sensitive && st.value) {
      let n = slug(lbl(st.d)), k = 2;
      while (used.has(n)) n = slug(lbl(st.d)) + "_" + k++;
      used.add(n);
      st.param = n;
      params.push({ name: n, label: lbl(st.d), default: st.value });
    }
  }
  const draft = { id: "sk_" + Date.now().toString(36), name: "", description: "", host: rec.host, startUrl: rec.startUrl, params, steps, api: null, useApi: true, created: Date.now(), uses: 0, ok: 0, fail: 0 };
  draft.api = detectApi(data, steps, params, rec.startUrl);
  return draft;
}

const BEACON = /collect|analytics|telemetry|track|beacon|metric|sentry|segment|logging|\/log\b|gtm|pixel/i;
function detectApi(data, steps, params, startUrl) {
  let origin = "";
  try { origin = new URL(startUrl).origin; } catch (e) {}
  const reqs = (data.reqs || []).filter((r) => {
    try { return new URL(r.url).origin === origin; } catch (e) { return false; }
  }).filter((r) => !r.opaque && !BEACON.test(r.url) && r.status >= 200 && r.status < 300);
  if (!reqs.length) return null;
  const lastStepAt = steps.length ? steps[steps.length - 1].at : 0;
  // the data-changing request closest to the end of the recording is the "do it" request
  const req = reqs[reqs.length - 1];
  if (lastStepAt && req.at < steps[0].at) return null;
  const ct = (req.headers["content-type"] || "").toLowerCase();
  let body = req.body || "";
  let url = req.url;
  const found = new Set();
  const variants = (v) => {
    const out = [];
    if (ct.includes("json")) out.push(JSON.stringify(v).slice(1, -1));
    else if (ct.includes("x-www-form-urlencoded")) out.push(encodeURIComponent(v).replace(/%20/g, "+"), encodeURIComponent(v));
    out.push(v);
    return out;
  };
  for (const p of params) {
    if (String(p.default).length < 3) return null; // too short to locate reliably
    let hit = false;
    for (const vv of variants(p.default)) {
      if (vv && body.includes(vv)) { body = body.split(vv).join("{{" + p.name + "}}"); hit = true; break; }
    }
    if (!hit) {
      const enc = encodeURIComponent(p.default);
      if (url.includes(enc)) { url = url.split(enc).join("{{" + p.name + "}}"); hit = true; }
    }
    if (!hit) return null; // a parameter the request does not use: cannot be replayed directly
    found.add(p.name);
  }
  return { method: req.method, url, headers: req.headers, body: body || null, contentType: ct };
}

function fillTokens(str, values, enc) {
  return String(str).replace(/\{\{([^}]+)\}\}/g, (m, k) => (k in values ? enc(String(values[k])) : m));
}
function buildRequest(api, values) {
  const ct = api.contentType || "";
  const bodyEnc = ct.includes("json") ? (v) => JSON.stringify(v).slice(1, -1) : ct.includes("x-www-form-urlencoded") ? (v) => encodeURIComponent(v).replace(/%20/g, "+") : (v) => v;
  return { method: api.method, url: fillTokens(api.url, values, encodeURIComponent), headers: api.headers, body: api.body == null ? null : fillTokens(api.body, values, bodyEnc) };
}

// ---------- saving a successful agent run as a skill ----------
function draftFromRun(host, startUrl) {
  const steps = [];
  const params = [];
  const used = new Set();
  const req = (lastRequestText || "").toLowerCase();
  const addParam = (st, val) => {
    if (!val || String(val).length < 2 || !req.includes(String(val).toLowerCase())) return;
    let n = slug(lbl(st.d)), k = 2;
    while (used.has(n)) n = slug(lbl(st.d)) + "_" + k++;
    used.add(n);
    st.param = n;
    params.push({ name: n, label: lbl(st.d), default: String(val) });
  };
  for (const a of actionLog) {
    const d = a.el && a.el.d ? a.el.d : null;
    if (a.name === "click" && d) {
      const prev = steps[steps.length - 1];
      if (prev && prev.t === "click" && lbl(prev.d) === lbl(d) && prev.d.css === d.css) continue; // an immediate repeat is noise
      steps.push({ t: "click", d });
    } else if (a.name === "type_text" && d) {
      const st = { t: "type", d, value: a.input.text, sensitive: false };
      addParam(st, a.input.text);
      steps.push(st);
      if (a.input.submit) steps.push({ t: "key", key: "Enter", d });
    } else if (a.name === "select_option" && d) {
      const st = { t: "select", d, value: a.input.option };
      addParam(st, a.input.option);
      steps.push(st);
    } else if (a.name === "press_key" && d !== undefined) {
      steps.push({ t: "key", key: a.input.key, d: d || {} });
    }
  }
  return { id: "sk_" + Date.now().toString(36), name: "", description: (lastRequestText || "").slice(0, 140), host, startUrl, params, steps, api: null, useApi: false, created: Date.now(), uses: 0, ok: 0, fail: 0 };
}
function offerSaveSkill(host, startUrl) {
  if (RUNNER || skillRunUsed || actionLog.length < 2) return;
  const div = addMessage("approval", "");
  const t = document.createElement("div");
  t.className = "title";
  t.textContent = i18n("sk.offer");
  const btns = document.createElement("div");
  btns.className = "btns";
  const yes = document.createElement("button");
  yes.className = "yes";
  yes.textContent = i18n("sk.offerYes");
  const no = document.createElement("button");
  no.className = "no";
  no.textContent = i18n("sk.offerNo");
  yes.onclick = () => { div.remove(); const d = draftFromRun(host, startUrl); if (d.steps.length) openSkillEditor(d); };
  no.onclick = () => div.remove();
  btns.append(yes, no);
  div.append(t, btns);
}

// ---------- values that steer later steps (typeahead suggestions, rows named after the value) ----------
// A click on "Project Alpha" right after typing "Alpha" must follow the parameter next time ("Project Beta"), not repeat the recorded label.
const normT = (s) => String(s || "").replace(/\s+/g, " ").trim().toLowerCase();
function valueMatches(text, value) {
  const v = normT(value);
  if (v.length < 2) return false;
  const t = normT(text);
  return v.split(" ").filter(Boolean).every((tok) => t.includes(tok));
}
// For click step i: which earlier parameter does its label / row context follow?
function dynLink(draft, i) {
  const st = draft.steps[i];
  const out = {};
  if (st.t !== "click" || !st.d) return out;
  for (let j = i - 1; j >= 0; j--) {
    const pj = draft.steps[j];
    if (!pj.param) continue;
    const p = draft.params.find((x) => x.name === pj.param);
    if (!p) continue;
    if (!out.label && st.d.label && valueMatches(st.d.label, p.default)) out.label = p.name;
    else if (!out.ctx && st.d.ctx && valueMatches(st.d.ctx, p.default)) out.ctx = p.name;
    if (out.label && out.ctx) break;
  }
  return out;
}
function linkDynamic(draft) {
  draft.steps.forEach((st, i) => {
    if (!st.d) return;
    delete st.d.dyn; delete st.d.ctxDyn;
    const l = dynLink(draft, i);
    if (l.label) st.d.dyn = l.label;
    if (l.ctx) st.d.ctxDyn = l.ctx;
  });
}
// The step as it should be searched for on THIS run (current parameter values instead of the recorded ones).
function resolveStep(st, values) {
  const r = JSON.parse(JSON.stringify(st));
  const d = r.d || (r.d = {});
  if (d.dyn && values[d.dyn] != null) {
    d.dynValue = String(values[d.dyn]);
    d.id = d.testid = d.css = d.name = d.href = ""; d.nth = 0;
  }
  if (d.ctxDyn && values[d.ctxDyn] != null) {
    d.ctx = String(values[d.ctxDyn]);
    d.id = d.testid = d.css = ""; d.nth = 0;
  }
  return r;
}

// ---------- editor card ----------
function mkEl(tag, cls, text) {
  const e = document.createElement(tag);
  if (cls) e.className = cls;
  if (text != null) e.textContent = text;
  return e;
}
function openSkillEditor(draft) {
  const card = addMessage("skillcard", "");
  card.append(mkEl("div", "title", i18n("sk.edTitle")));
  const name = mkEl("input");
  name.placeholder = i18n("sk.name");
  name.value = draft.name || "";
  const desc = mkEl("input");
  desc.placeholder = i18n("sk.desc");
  desc.value = draft.description || "";
  card.append(mkEl("label", "", i18n("sk.name")), name, mkEl("label", "", i18n("sk.desc")), desc);
  const list = mkEl("div", "steps");
  const render = () => {
    list.textContent = "";
    draft.steps.forEach((st, i) => {
      const row = mkEl("div", "srow");
      row.append(mkEl("span", "sn", String(i + 1)));
      const lk = dynLink(draft, i);
      row.append(mkEl("span", "stext", stepText(st) + (lk.label || lk.ctx ? " " + i18n("sk.follows", { param: lk.label || lk.ctx }) : "")));
      if ((st.t === "type" || st.t === "select") && !st.sensitive) {
        const cb = mkEl("input");
        cb.type = "checkbox";
        cb.checked = !!st.param;
        cb.title = i18n("sk.param");
        const pn = mkEl("input", "pname");
        pn.value = st.param || slug(lbl(st.d));
        pn.hidden = !st.param;
        cb.onchange = () => {
          if (cb.checked) {
            st.param = pn.value || slug(lbl(st.d));
            if (!draft.params.some((p) => p.name === st.param)) draft.params.push({ name: st.param, label: lbl(st.d), default: st.value });
          } else {
            draft.params = draft.params.filter((p) => p.name !== st.param);
            st.param = null;
          }
          draft.api = null; // the direct request was derived with the old parameter set
          render();
        };
        pn.onchange = () => {
          const p = draft.params.find((x) => x.name === st.param);
          const nn = slug(pn.value);
          if (p) p.name = nn;
          st.param = nn;
          pn.value = nn;
        };
        row.append(cb, mkEl("span", "plabel", i18n("sk.param")), pn);
      } else if (st.sensitive) {
        row.append(mkEl("span", "plabel", i18n("sk.secret")));
      }
      const del = mkEl("a", "sdel", "✕");
      del.href = "#";
      del.title = i18n("sk.delete");
      del.onclick = (e) => { e.preventDefault(); const s = draft.steps[i]; draft.steps.splice(i, 1); if (s.param) draft.params = draft.params.filter((p) => p.name !== s.param); draft.api = null; render(); };
      row.append(del);
      list.append(row);
    });
  };
  render();
  card.append(list);
  let apiCb = null;
  if (draft.api) {
    const wrap = mkEl("label", "apiopt");
    apiCb = mkEl("input");
    apiCb.type = "checkbox";
    apiCb.checked = draft.useApi !== false;
    wrap.append(apiCb, document.createTextNode(" " + i18n("sk.useApi")));
    card.append(wrap, mkEl("div", "small", i18n("sk.apiInfo", { req: `${draft.api.method} ${draft.api.url.replace(/^https?:\/\/[^/]+/, "").slice(0, 80)}` })));
  }
  const btns = mkEl("div", "btns");
  const save = mkEl("button", "yes", i18n("sk.save"));
  const cancel = mkEl("button", "no", i18n("sk.cancel"));
  save.onclick = async () => {
    const nm = name.value.trim();
    if (!nm) { name.focus(); name.classList.add("bad"); return; }
    draft.name = nm.slice(0, 60);
    draft.description = desc.value.trim().slice(0, 200);
    draft.useApi = apiCb ? apiCb.checked : false;
    if (!draft.steps.length) { card.remove(); return; }
    // keep each parameter's default in sync with its (possibly edited) step
    draft.params = draft.params.filter((p) => draft.steps.some((s) => s.param === p.name));
    linkDynamic(draft);
    await skillsLoad();
    SKILLS = SKILLS.filter((s) => !(s.host === draft.host && s.name.toLowerCase() === draft.name.toLowerCase()));
    SKILLS.push(draft);
    await skillsSave();
    card.remove();
    addMessage("note", i18n("sk.saved", { name: draft.name }));
  };
  cancel.onclick = () => card.remove();
  btns.append(save, cancel);
  card.append(btns);
}

// ---------- the skills list ----------
function renderSkills() {
  const box = $("skillsList");
  if (!box) return;
  box.textContent = "";
  $("skillsTitle").textContent = i18n("sk.listTitle", { n: SKILLS.length });
  $("skillsBox").hidden = SKILLS.length === 0 && !recording;
  if (!SKILLS.length) return;
  for (const sk of SKILLS) {
    const row = mkEl("div", "skill");
    const top = mkEl("div", "sktop");
    top.append(mkEl("b", "", sk.name), mkEl("span", "skhost", sk.host));
    row.append(top);
    if (sk.description) row.append(mkEl("div", "skdesc", sk.description));
    row.append(mkEl("div", "skmeta", `${i18n("sk.stepsN", { n: sk.steps.length })}${sk.api && sk.useApi ? " · " + i18n("sk.fast") : ""} · ${i18n("sk.uses", { n: sk.uses || 0 })}${sk.fail ? " · " + i18n("sk.failN", { n: sk.fail }) : ""}`));
    const form = mkEl("div", "skform");
    const inputs = {};
    for (const p of sk.params) {
      const lab = mkEl("label", "", p.label || p.name);
      const inp = mkEl("input");
      inp.value = p.default || "";
      inputs[p.name] = inp;
      form.append(lab, inp);
    }
    const btns = mkEl("div", "tbtns");
    const run = mkEl("button", "yes", i18n("sk.runNow"));
    run.onclick = () => {
      const values = {};
      for (const p of sk.params) values[p.name] = inputs[p.name].value;
      runSkillFromList(sk, values);
    };
    const del = mkEl("button", "no", i18n("sk.delete"));
    del.onclick = async () => {
      if (!confirm(i18n("sk.deleteConfirm", { name: sk.name }))) return;
      SKILLS = SKILLS.filter((s) => s.id !== sk.id);
      await skillsSave();
    };
    btns.append(run, del);
    row.append(form, btns);
    box.append(row);
  }
}

async function runSkillFromList(sk, values) {
  if (running) return;
  const tab = await getActiveTab();
  if (!tab || !/^https?:/.test(tab.url || "")) { addMessage("err", i18n("err.unreadable")); return; }
  const hint = document.querySelector(".hint");
  if (hint) hint.remove();
  addMessage("user", i18n("sk.running", { name: sk.name }));
  setRunning(true);
  try {
    const target = await resolveSkillTab(sk, tab);
    agentTabId = target.id;
    typedLog = [];
    const res = await executeSkill(sk, values, target);
    addMessage(res.ok ? "note" : "err", res.ok ? i18n("sk.done", { name: sk.name }) + " " + (res.text || "") : res.text);
  } catch (e) {
    addMessage("err", e.message);
  } finally {
    setRunning(false);
  }
}

// A skill belongs to a site, not to the tab you happen to be looking at. Find (or open) that site's tab and work there.
async function resolveSkillTab(sk, tab) {
  if (hostOf(tab.url) === sk.host) return tab;
  let all = [];
  try { all = await chrome.tabs.query({}); } catch (e) {}
  const same = all.filter((t) => hostOf(t.url) === sk.host);
  const target = same.find((t) => stripHash(t.url) === stripHash(sk.startUrl)) || same[0];
  if (target) {
    try { await chrome.tabs.update(target.id, { active: true }); } catch (e) {}
    logLine(i18n("sk.switched", { host: sk.host }));
    return target;
  }
  logLine(i18n("sk.opening", { host: sk.host }));
  const t = await chrome.tabs.create({ url: sk.startUrl, active: true });
  await settle(t.id);
  return await chrome.tabs.get(t.id);
}

// ---------- running a skill ----------
async function doRunSkill(input, tab) {
  await skillsLoad();
  const host = hostOf(tab.url);
  const want = String(input.name || "").trim().toLowerCase();
  const byName = SKILLS.filter((s) => s.id === input.name || s.name.toLowerCase() === want);
  const sk = byName.find((s) => s.host === host) || byName[0];
  if (!sk) return { error: true, text: `No saved skill "${input.name}". Available: ${SKILLS.map((s) => s.name).join(", ") || "(none)"}. Continue manually.` };
  skillRunUsed = true;
  const given = input.params && typeof input.params === "object" ? input.params : {};
  const values = {};
  const missing = [];
  for (const p of sk.params) {
    let v = given[p.name];
    if (v == null || v === "") v = p.default;
    if (v == null || v === "") missing.push(p.name);
    values[p.name] = v == null ? "" : String(v);
  }
  if (missing.length) return { error: true, text: `Skill "${sk.name}" needs a value for: ${missing.join(", ")}. Ask the user, then call run_skill again.` };
  tab = await resolveSkillTab(sk, tab);
  agentTabId = tab.id;
  const res = await executeSkill(sk, values, tab);
  let snap = "";
  try { snap = await takeSnapshot(); } catch (e) {}
  return { error: !res.ok, text: res.text + (snap ? "\n\n" + snap : "") };
}

async function stepApproval(sk, st, values, tab, i) {
  if ($("autoApprove").checked) { logLine(i18n("log.auto", { action: stepText(st, st.param ? values[st.param] : null) })); return true; }
  const filled = sk.params.length ? sk.params.map((p) => `  ${p.label || p.name}: ${values[p.name]}`).join("\n") : "  " + i18n("d.none");
  return await requestApproval(stepText(st, st.param ? values[st.param] : null), i18n("why.final"), `${i18n("d.page")}: ${tab.title}\n${tab.url}\n\n${i18n("act.skill", { name: sk.name })}\n${i18n("d.fields")}:\n${filled}`);
}

async function executeSkill(sk, values, tab) {
  const t0 = Date.now();
  T("skill-run", { name: sk.name, steps: sk.steps.length, api: !!(sk.api && sk.useApi), values });
  let result;
  try { result = await executeSkillInner(sk, values, tab); } catch (e) { result = { ok: false, text: `Skill "${sk.name}" failed: ${e.message}` }; }
  await skillsLoad();
  const cur = SKILLS.find((s) => s.id === sk.id);
  if (cur) { cur.uses = (cur.uses || 0) + 1; if (result.ok) cur.ok = (cur.ok || 0) + 1; else if (!result.denied) cur.fail = (cur.fail || 0) + 1; await skillsSave(); }
  T("skill-done", { ok: result.ok, ms: Date.now() - t0, text: String(result.text).slice(0, 200) });
  return result;
}

async function executeSkillInner(sk, values, tab) {
  // 1) direct request, when the recording found one that fits every parameter
  if (sk.api && sk.useApi !== false) {
    const req = buildRequest(sk.api, values);
    let ok = true;
    if (!$("autoApprove").checked) {
      const body = req.body ? "\n\n" + String(req.body).slice(0, 600) : "";
      ok = await requestApproval(i18n("act.skill", { name: sk.name }), i18n("why.skillApi"), `${i18n("d.page")}: ${tab.title}\n${tab.url}\n\n${req.method} ${req.url}${body}`);
    } else logLine(i18n("log.auto", { action: i18n("act.skill", { name: sk.name }) }));
    if (!ok) return { ok: false, denied: true, text: "The user denied running this skill. Do not retry it. Ask how to proceed." };
    if (stopFlag) return { ok: false, text: "Stopped by user." };
    logLine("▶ " + i18n("sk.direct"));
    const [r] = await chrome.scripting.executeScript({ target: { tabId: agentTabId }, func: fetchFn, args: [req] });
    const res = r.result || { ok: false, status: 0, text: "no response" };
    T("skill-api", { status: res.status });
    if (res.ok) return { ok: true, mode: "api", text: `Direct request succeeded (HTTP ${res.status}). Response: ${res.text.slice(0, 400)}` };
    logLine("✖ " + i18n("sk.directFailed", { status: res.status }));
    if (res.status >= 400 && res.status < 500 && res.status !== 401 && res.status !== 403 && res.status !== 404) {
      return { ok: false, text: `Skill "${sk.name}": the app rejected the direct request (HTTP ${res.status}): ${res.text.slice(0, 300)}. Check the values; do not retry blindly.` };
    }
  } else if ($("stepMode").checked) {
    const ok = await requestApproval(i18n("act.skill", { name: sk.name }), i18n("why.step"), sk.steps.map((s, i) => `${i + 1}. ${stepText(s, s.param ? values[s.param] : null)}`).join("\n"));
    if (!ok) return { ok: false, denied: true, text: "The user denied running this skill. Do not retry it. Ask how to proceed." };
  }

  // 2) replay the recorded UI steps, checking each one
  const done = [];
  // Already past the first steps? (e.g. the recording began on page 1 and clicked through to page 2, and we are on page 2 now.)
  let i0 = 0;
  try {
    const here = stripHash((await chrome.tabs.get(agentTabId)).url);
    while (i0 < sk.steps.length - 1 && sk.steps[i0].t === "click" && stripHash(sk.steps[i0].url) !== here && stripHash(sk.steps[i0 + 1].url) === here) i0++;
    if (i0) logLine(i18n("sk.skipped", { n: i0 }));
  } catch (e) {}
  for (let i = i0; i < sk.steps.length; i++) {
    if (stopFlag) return { ok: false, text: "Stopped by user." };
    const st = sk.steps[i];
    const val = st.param ? values[st.param] : st.value;
    const rs = resolveStep(st, values);
    const txt = stepText(rs, val);
    if (st.t === "type" && st.sensitive) return { ok: false, text: `Skill "${sk.name}" stopped at step ${i + 1}: "${lbl(st.d)}" is a sensitive field. Ask the user to fill it in, then continue manually. Done so far: ${done.join("; ") || "(nothing)"}.` };
    if (stepIsFinal(st) && !(await stepApproval(sk, st, values, tab, i))) return { ok: false, denied: true, text: "The user denied this action. Do not retry it. Ask how to proceed." };
    logLine(`▶ ${i + 1}/${sk.steps.length} ${txt}`);
    let sigBefore = "";
    if (st.t === "click" && st.effect) { try { [{ result: sigBefore }] = await chrome.scripting.executeScript({ target: { tabId: agentTabId }, func: pageSigFn }); } catch (e) {} }
    const tryStep = async (timeout) => {
      try {
        const [r] = await chrome.scripting.executeScript({ target: { tabId: agentTabId }, func: replayStepFn, args: [resolveStep(st, values), val == null ? null : String(val), timeout] });
        return r.result || { ok: false, msg: "no result" };
      } catch (e) { return { ok: false, msg: e.message }; }
    };
    let res = await tryStep(i === i0 ? 2500 : 6000);
    if (!res.ok && i === i0) {
      // not on the starting page: go there (same site) and try again
      const cur = (await chrome.tabs.get(agentTabId)).url;
      if (sk.startUrl && stripHash(cur) !== stripHash(sk.startUrl) && hostOf(sk.startUrl) === sk.host) {
        await chrome.tabs.update(agentTabId, { url: sk.startUrl });
        await settle(agentTabId);
        res = await tryStep(6000);
      }
    }
    if (!res.ok) {
      T("skill-step-failed", { i, text: txt, msg: res.msg });
      return { ok: false, text: `Skill "${sk.name}" stopped at step ${i + 1}/${sk.steps.length} (${txt}): ${res.msg} Done so far: ${done.join("; ") || "(nothing)"}. Continue this task manually from the current page.` };
    }
    await settle(agentTabId, st.t !== "click" && st.t !== "key");
    if (newTabId) { agentTabId = newTabId; newTabId = null; await settle(agentTabId); }
    if (st.t === "click" && st.effect && sigBefore) {
      // the recording showed this click changes the page: make sure it did (results can arrive a moment later)
      let changed = false;
      for (let k = 0; k < 6 && !changed; k++) {
        let sig = "";
        try { [{ result: sig }] = await chrome.scripting.executeScript({ target: { tabId: agentTabId }, func: pageSigFn }); } catch (e) { changed = true; break; }
        changed = sig !== sigBefore;
        if (!changed) await sleep(300);
      }
      if (!changed) {
        T("skill-step-failed", { i, text: txt, msg: "no effect" });
        return { ok: false, text: `Skill "${sk.name}" stopped at step ${i + 1}/${sk.steps.length} (${txt}): the click had no effect on the page. Done so far: ${done.join("; ") || "(nothing)"}. Continue manually.` };
      }
    }
    done.push(txt);
    if (st.t === "type" && sk.params.length) typedLog.push({ label: lbl(st.d), value: String(val) });
  }
  return { ok: true, mode: "ui", text: `Skill "${sk.name}" completed all ${sk.steps.length} steps.` };
}

// ---------- wiring ----------
function skillsInit() {
  skillsLoad().then(renderSkills);
  if ($("recBtn")) {
    $("recBtn").onclick = (e) => { e.preventDefault(); if (recording) stopRecording(); else startRecording(); };
    $("recStop").onclick = () => stopRecording();
    $("skillsBtn").onclick = (e) => { e.preventDefault(); const b = $("skillsBox"); b.hidden = false; b.open = !b.open; };
  }
  try {
    chrome.storage.onChanged.addListener((ch, area) => { if (area === "local" && ch.skills) { SKILLS = Array.isArray(ch.skills.newValue) ? ch.skills.newValue : []; renderSkills(); } });
  } catch (e) {}
  i18nOnChange(renderSkills);
}
skillsInit();
























//lang en
// English (source language). Every other lang-*.js file must contain exactly these keys and the same {placeholders}.
I18N.en = {
  "lang.label": "Language",
  "lang.help": "Changes the language of the whole extension and the language the assistant replies in.",
  "hdr.newChat": "New chat",
  "hdr.debug": "Copy debug log",
  "hdr.settings": "Settings",
  "hdr.currentPage": "Current page",
  "hint.intro": "Ask about the page, or tell me what to do on it. For example:",
  "chip.1": "What tasks are assigned to John?",
  "chip.2": "Create a new record using the details I give you",
  "chip.3": "Book a ticket from Pune to Mumbai for tomorrow",
  "hint.safety": "I'll ask for your approval before submitting, paying, booking, deleting or sending anything, and I never type passwords, card numbers or OTPs.",
  "hint.tip": "Tip: add a time and I'll do it later, e.g. “in 5 minutes, submit this form”.",
  "opt.step": "Approve every action",
  "opt.auto": "Auto-approve actions",
  "opt.debug": "Show model replies (debug)",
  "input.ph": "Ask or instruct… (Enter to send, Shift+Enter for new line)",
  "btn.send": "Send",
  "btn.stop": "Stop",
  "btn.approve": "Approve",
  "btn.deny": "Deny",
  "appr.title": "Approve? {action}",
  "appr.approved": "Approved: {action}",
  "appr.denied": "Denied: {action}",
  "why.step": "“Approve every action” is on.",
  "why.final": "This looks like a final or irreversible action.",
  "why.submit": "This types and submits a form.",
  "why.enter": "Pressing Enter can submit a form.",
  "why.site": "This opens a different website.",
  "why.schedule": "This will run later, on this page, without you watching. Auto-approve is OFF for it, so it will pause and ask you before final actions.",
  "d.page": "Page",
  "d.fields": "Fields filled during this task",
  "d.none": "(none)",
  "d.willDo": "What it will do",
  "act.type": "type “{text}” into {target}",
  "act.typeSubmit": "type “{text}” into {target} and submit",
  "act.select": "select “{option}” in {target}",
  "act.click": "click {target}",
  "act.press": "press {key}",
  "act.open": "open {url}",
  "act.later": "schedule for later: “{text}”",
  "act.scheduleAt": "schedule for {when}",
  "log.auto": "✔ auto-approved: {action}",
  "note.stopped": "Stopped.",
  "note.newChat": "New chat. Ask about the current page or tell me what to do.",
  "note.noControls": "I can't see any clickable controls on this page{iframes}. It may still be loading, or the app may run inside an iframe.",
  "note.iframes": " (it contains {n} iframe(s); content inside iframes isn't visible to me)",
  "note.stepLimit": "Reached the {n}-step limit. Tell me to continue if you want me to keep going.",
  "note.notAction": "This looks like instructions rather than an action: the model did not call a tool. Tick “Show model replies”, or use Copy debug log and send it to the developer.",
  "note.debugCopied": "Debug log copied (no API key included). Paste it to the developer, after removing anything private.",
  "note.copyFailed": "Could not copy automatically.",
  "err.unreadable": "This page can't be read or controlled (browser-internal pages like chrome:// or the Web Store are blocked). Switch to a normal web page.",
  "run.title": "Scheduled task",
  "run.autoOn": "Running unattended with auto-approve ON.",
  "run.autoOff": "Running unattended. I will pause and ask you before final actions.",
  "run.cannotStart": "Could not start the scheduled task.",
  "run.window": "Page Agent: scheduled task",
  "confirm.auto": "Auto-approve lets the agent submit, book, pay, save, send and delete WITHOUT asking you first.\n\nPasswords, card numbers and OTPs are still never typed by the agent, and you can press Stop at any time.\n\nTurn on auto-approve?",
  "tasks.title": "Scheduled tasks ({n} active)",
  "tasks.clear": "clear finished",
  "tasks.at": "at {time}",
  "tasks.starting": "starting…",
  "tasks.in": "in {t}",
  "tasks.auto": "auto-approve",
  "tasks.waiting": "Waiting for your approval: {title}",
  "tasks.open": "Open task window",
  "tasks.cancel": "Cancel",
  "tasks.stop": "Stop",
  "tasks.result": "Result",
  "tasks.log": "Log",
  "st.scheduled": "Scheduled",
  "st.launching": "Starting",
  "st.running": "Running",
  "st.needs_approval": "Needs approval",
  "st.done": "Done",
  "st.failed": "Failed",
  "st.cancelled": "Stopped",
  "st.missed": "Missed",
  "st.interrupted": "Interrupted",
  "set.title": "Settings",
  "set.provider": "Provider type",
  "prov.anthropic": "Anthropic Messages API (or compatible)",
  "prov.openai": "OpenAI-compatible (/chat/completions)",
  "prov.custom": "Custom (my own request/response format)",
  "prov.help.anthropic": "Anthropic's API, or a proxy that uses the same /v1/messages format with tool use.",
  "prov.help.openai": "Any server with POST {base}/chat/completions and tool calling, e.g. https://your-host/v1. The model must support tool calling.",
  "prov.help.custom": "Any HTTP JSON API. You define the request body and where the reply text is in the response. Tools work through a JSON-in-text protocol, so the model only needs to follow instructions.",
  "set.baseUrl": "Base URL",
  "set.requestUrl": "Request URL (full endpoint, POST)",
  "set.urlHelp": "Leave empty for the official endpoint of the selected type.",
  "set.urlHelpCustom": "Required. {{model}} and {{apiKey}} can be used in the URL.",
  "set.apiKey": "API key",
  "set.apiKeyPh": "optional for local models",
  "set.apiKeyHelp": "Stored only in this browser (chrome.storage.local) and sent only to the URL above.",
  "set.model": "Model name",
  "set.modelPh": "your provider's model id",
  "set.headers": "Extra headers (optional, JSON)",
  "set.headersHelp": "For providers that use a different auth header or need custom headers. {{apiKey}} and {{model}} work in the Custom type.",
  "set.bodyTpl": "Request body template (JSON)",
  "set.bodyTplHelp": "Placeholders: \"{{model}}\", \"{{system}}\", \"{{messages}}\" (list of {role, content}), \"{{prompt}}\" (everything as one text), \"{{max_tokens}}\" (number), \"{{chat}}\" (system message + messages, for OpenAI-style APIs), \"{{tools}}\" (tool definitions in OpenAI format, only if your API supports native tools). If the template has none of {{system}}, {{chat}} or {{prompt}}, the instructions and tools are added to the first user message automatically. A value that is exactly a placeholder gets the real value; placeholders inside longer text are inserted as text.",
  "set.respPath": "Response text path",
  "set.respPathPh": "e.g. choices.0.message.content or data.reply (empty = auto-detect)",
  "set.respPathHelp": "Where the model's reply text is in the JSON response. If the response is plain text, leave empty.",
  "set.suffix": "Prompt suffix (optional, custom provider)",
  "set.suffixPh": "e.g. /no_think (Qwen3: turns off long reasoning)",
  "set.suffixHelp": "Added to the end of every message sent to the model, e.g. switches like /no_think.",
  "set.compact": "Compact page snapshots (for small-context models)",
  "set.compactHelp": "Sends fewer page elements and less text per step. Turn on if your model has a small context window (Ollama's default is only 2-4k tokens: also raise num_ctx to 16k+).",
  "btn.save": "Save",
  "btn.test": "Save & test connection",
  "set.saved": "Saved.",
  "set.testing": "Testing…",
  "set.connected": "Connected. Model replied: {reply}",
  "set.emptyReply": "(empty reply)",
  "set.foundAt": "Reply text was found at: {path}",
  "set.pathTip": "Tip: paste this into “Response text path” to lock it in.",
  "err.tplJson": "Request body template must be valid JSON.",
  "bg.finished": "Task finished",
  "bg.failed": "Task failed",
  "bg.stopped": "Task stopped",
  "bg.missed": "Task missed",
  "bg.interrupted": "Task interrupted",
  "bg.approval": "Approval needed",
  "bg.cancelledBefore": "Cancelled before it ran.",
  "bg.stoppedByYou": "Stopped by you.",
  "bg.done": "Finished.",
  "bg.noStart": "The task window did not start.",
  "bg.couldNotStart": "Could not start: {err}",
  "bg.missedMsg": "It was due at {when} but the browser was not running within {min} minutes of that time, so it was not run.",
  "bg.winClosed": "The task window was closed before the task finished.",
  "bg.restarted": "The browser was closed or restarted while this task was running.",
  "sk.record": "Record a skill",
  "sk.list": "Skills",
  "sk.stopRec": "Stop recording",
  "sk.recording": "● Recording… do the task on the page, then press Stop. ({n} steps)",
  "sk.recStarted": "Recording started. Do the task once, normally, on the page. Passwords are never saved.",
  "sk.recEmpty": "Nothing was recorded.",
  "sk.click": "click “{label}”",
  "sk.type": "type “{value}” into “{label}”",
  "sk.typeSecret": "type a secret into “{label}” (you fill this in)",
  "sk.select": "choose “{value}” in “{label}”",
  "sk.key": "press {key} in “{label}”",
  "sk.edTitle": "Save as skill",
  "sk.name": "Name",
  "sk.desc": "What it does",
  "sk.param": "parameter",
  "sk.secret": "secret: not saved",
  "sk.delete": "Delete",
  "sk.useApi": "Use the direct request when possible (much faster)",
  "sk.apiInfo": "Direct request found: {req}",
  "sk.save": "Save skill",
  "sk.cancel": "Cancel",
  "sk.saved": "Skill “{name}” saved. Ask for it in plain words, or run it from Skills.",
  "sk.offer": "That worked. Save these steps as a skill so next time it runs fast?",
  "sk.offerYes": "Save as skill",
  "sk.offerNo": "No thanks",
  "sk.listTitle": "Skills ({n})",
  "sk.stepsN": "{n} steps",
  "sk.fast": "direct request",
  "sk.uses": "used {n}×",
  "sk.failN": "{n} failed",
  "sk.runNow": "Run now",
  "sk.deleteConfirm": "Delete skill “{name}”?",
  "sk.wrongSite": "This skill belongs to {host}. Open that site first.",
  "sk.running": "Run skill: {name}",
  "sk.done": "Skill “{name}” finished.",
  "sk.direct": "Sending the direct request…",
  "sk.directFailed": "The direct request did not work (HTTP {status}); replaying the steps instead.",
  "act.skill": "run skill “{name}”",
  "why.skillApi": "This skill sends a request that changes data on the site.",
  "sk.follows": "(follows “{param}”)",
  "sk.switched": "Switching to your open tab on {host}…",
  "sk.opening": "Opening {host}…",
  "sk.skipped": "Already past the first {n} step(s); continuing from here.",
};























//lang ko
// Korean. Must contain exactly the same keys and {placeholders} as lang-en.js.
I18N.ko = {
  "lang.label": "언어",
  "lang.help": "확장 프로그램 전체의 언어와 어시스턴트가 답변하는 언어를 바꿉니다.",
  "hdr.newChat": "새 대화",
  "hdr.debug": "디버그 로그 복사",
  "hdr.settings": "설정",
  "hdr.currentPage": "현재 페이지",
  "hint.intro": "페이지에 대해 질문하거나, 페이지에서 할 일을 알려 주세요. 예를 들어:",
  "chip.1": "John에게 배정된 작업은 무엇인가요?",
  "chip.2": "내가 알려 주는 정보로 새 레코드를 만들어 줘",
  "chip.3": "내일 푸네에서 뭄바이로 가는 티켓을 예약해 줘",
  "hint.safety": "제출, 결제, 예약, 삭제, 전송을 하기 전에는 반드시 승인을 요청하며, 비밀번호, 카드 번호, OTP는 절대 입력하지 않습니다.",
  "hint.tip": "팁: 시간을 함께 말하면 나중에 실행합니다. 예: “5분 후에 이 양식을 제출해 줘”.",
  "opt.step": "모든 작업 승인",
  "opt.auto": "작업 자동 승인",
  "opt.debug": "모델 응답 표시(디버그)",
  "input.ph": "질문하거나 지시하세요… (Enter로 전송, Shift+Enter로 줄바꿈)",
  "btn.send": "보내기",
  "btn.stop": "중지",
  "btn.approve": "승인",
  "btn.deny": "거부",
  "appr.title": "승인하시겠습니까? {action}",
  "appr.approved": "승인됨: {action}",
  "appr.denied": "거부됨: {action}",
  "why.step": "“모든 작업 승인”이 켜져 있습니다.",
  "why.final": "최종 단계이거나 되돌릴 수 없는 작업으로 보입니다.",
  "why.submit": "양식에 입력하고 제출합니다.",
  "why.enter": "Enter 키를 누르면 양식이 제출될 수 있습니다.",
  "why.site": "다른 웹사이트를 엽니다.",
  "why.schedule": "나중에 이 페이지에서 사용자가 보지 않는 상태로 실행됩니다. 자동 승인은 꺼져 있으므로 최종 작업 전에 일시 중지하고 물어봅니다.",
  "d.page": "페이지",
  "d.fields": "이 작업 중 입력한 필드",
  "d.none": "(없음)",
  "d.willDo": "수행할 작업",
  "act.type": "{target}에 “{text}” 입력",
  "act.typeSubmit": "{target}에 “{text}” 입력 후 제출",
  "act.select": "{target}에서 “{option}” 선택",
  "act.click": "{target} 클릭",
  "act.press": "{key} 키 누르기",
  "act.open": "{url} 열기",
  "act.later": "나중에 실행 예약: “{text}”",
  "act.scheduleAt": "{when}에 실행 예약",
  "log.auto": "✔ 자동 승인됨: {action}",
  "note.stopped": "중지되었습니다.",
  "note.newChat": "새 대화입니다. 현재 페이지에 대해 질문하거나 할 일을 알려 주세요.",
  "note.noControls": "이 페이지에서 클릭할 수 있는 컨트롤이 보이지 않습니다{iframes}. 아직 로딩 중이거나 앱이 iframe 안에서 실행 중일 수 있습니다.",
  "note.iframes": " (iframe {n}개가 있으며, iframe 안의 내용은 볼 수 없습니다)",
  "note.stepLimit": "{n}단계 제한에 도달했습니다. 계속하려면 계속하라고 말씀해 주세요.",
  "note.notAction": "작업이 아니라 설명처럼 보입니다. 모델이 도구를 호출하지 않았습니다. “모델 응답 표시”를 켜거나 디버그 로그를 복사해 개발자에게 보내 주세요.",
  "note.debugCopied": "디버그 로그를 복사했습니다(API 키는 포함되지 않음). 개인 정보를 지운 뒤 개발자에게 붙여넣어 전달하세요.",
  "note.copyFailed": "자동으로 복사하지 못했습니다.",
  "err.unreadable": "이 페이지는 읽거나 제어할 수 없습니다(chrome:// 같은 브라우저 내부 페이지와 웹 스토어는 차단됩니다). 일반 웹 페이지로 이동하세요.",
  "run.title": "예약된 작업",
  "run.autoOn": "자동 승인을 켠 상태로 무인 실행 중입니다.",
  "run.autoOff": "무인 실행 중입니다. 최종 작업 전에 일시 중지하고 물어보겠습니다.",
  "run.cannotStart": "예약된 작업을 시작하지 못했습니다.",
  "run.window": "Page Agent: 예약된 작업",
  "confirm.auto": "자동 승인을 켜면 에이전트가 먼저 묻지 않고 제출, 예약, 결제, 저장, 전송, 삭제를 수행합니다.\n\n비밀번호, 카드 번호, OTP는 여전히 에이전트가 입력하지 않으며, 언제든 중지를 누를 수 있습니다.\n\n자동 승인을 켜시겠습니까?",
  "tasks.title": "예약된 작업 (진행 중 {n}개)",
  "tasks.clear": "완료된 항목 지우기",
  "tasks.at": "{time}에",
  "tasks.starting": "시작 중…",
  "tasks.in": "{t} 후",
  "tasks.auto": "자동 승인",
  "tasks.waiting": "승인 대기 중: {title}",
  "tasks.open": "작업 창 열기",
  "tasks.cancel": "취소",
  "tasks.stop": "중지",
  "tasks.result": "결과",
  "tasks.log": "로그",
  "st.scheduled": "예약됨",
  "st.launching": "시작 중",
  "st.running": "실행 중",
  "st.needs_approval": "승인 필요",
  "st.done": "완료",
  "st.failed": "실패",
  "st.cancelled": "중지됨",
  "st.missed": "놓침",
  "st.interrupted": "중단됨",
  "set.title": "설정",
  "set.provider": "제공자 유형",
  "prov.anthropic": "Anthropic Messages API (또는 호환)",
  "prov.openai": "OpenAI 호환 (/chat/completions)",
  "prov.custom": "사용자 지정 (직접 정의하는 요청/응답 형식)",
  "prov.help.anthropic": "Anthropic API 또는 도구 사용이 가능한 동일한 /v1/messages 형식의 프록시입니다.",
  "prov.help.openai": "POST {base}/chat/completions와 도구 호출을 지원하는 모든 서버입니다. 예: https://your-host/v1. 모델이 도구 호출을 지원해야 합니다.",
  "prov.help.custom": "모든 HTTP JSON API입니다. 요청 본문과 응답에서 답변 텍스트의 위치를 직접 정의합니다. 도구는 텍스트 속 JSON 프로토콜로 동작하므로 모델은 지시를 따르기만 하면 됩니다.",
  "set.baseUrl": "기본 URL",
  "set.requestUrl": "요청 URL (전체 엔드포인트, POST)",
  "set.urlHelp": "비워 두면 선택한 유형의 공식 엔드포인트를 사용합니다.",
  "set.urlHelpCustom": "필수입니다. URL에 {{model}}과 {{apiKey}}를 사용할 수 있습니다.",
  "set.apiKey": "API 키",
  "set.apiKeyPh": "로컬 모델은 선택 사항",
  "set.apiKeyHelp": "이 브라우저(chrome.storage.local)에만 저장되며 위의 URL로만 전송됩니다.",
  "set.model": "모델 이름",
  "set.modelPh": "제공자의 모델 ID",
  "set.headers": "추가 헤더 (선택, JSON)",
  "set.headersHelp": "다른 인증 헤더를 쓰거나 사용자 지정 헤더가 필요한 제공자용입니다. 사용자 지정 유형에서는 {{apiKey}}와 {{model}}을 사용할 수 있습니다.",
  "set.bodyTpl": "요청 본문 템플릿 (JSON)",
  "set.bodyTplHelp": "자리 표시자: \"{{model}}\", \"{{system}}\", \"{{messages}}\" ({role, content} 목록), \"{{prompt}}\" (전체를 하나의 텍스트로), \"{{max_tokens}}\" (숫자), \"{{chat}}\" (시스템 메시지 + 메시지, OpenAI 방식 API용), \"{{tools}}\" (OpenAI 형식의 도구 정의, API가 네이티브 도구를 지원할 때만). 템플릿에 {{system}}, {{chat}}, {{prompt}}가 하나도 없으면 지침과 도구가 첫 번째 사용자 메시지에 자동으로 추가됩니다. 값이 자리 표시자 하나로만 되어 있으면 실제 값으로 바뀌고, 긴 텍스트 안의 자리 표시자는 텍스트로 삽입됩니다.",
  "set.respPath": "응답 텍스트 경로",
  "set.respPathPh": "예: choices.0.message.content 또는 data.reply (비우면 자동 감지)",
  "set.respPathHelp": "JSON 응답에서 모델의 답변 텍스트가 있는 위치입니다. 응답이 일반 텍스트이면 비워 두세요.",
  "set.suffix": "프롬프트 접미사 (선택, 사용자 지정 제공자)",
  "set.suffixPh": "예: /no_think (Qwen3: 긴 추론 끄기)",
  "set.suffixHelp": "모델로 보내는 모든 메시지 끝에 추가됩니다. 예: /no_think 같은 스위치.",
  "set.compact": "페이지 스냅샷 간소화 (작은 컨텍스트 모델용)",
  "set.compactHelp": "단계마다 보내는 페이지 요소와 텍스트를 줄입니다. 모델의 컨텍스트 창이 작을 때 켜세요(Ollama 기본값은 2-4k 토큰뿐이므로 num_ctx도 16k 이상으로 올리세요).",
  "btn.save": "저장",
  "btn.test": "저장 및 연결 테스트",
  "set.saved": "저장되었습니다.",
  "set.testing": "테스트 중…",
  "set.connected": "연결되었습니다. 모델 응답: {reply}",
  "set.emptyReply": "(빈 응답)",
  "set.foundAt": "응답 텍스트 위치: {path}",
  "set.pathTip": "팁: 이 값을 “응답 텍스트 경로”에 붙여넣어 고정하세요.",
  "err.tplJson": "요청 본문 템플릿은 올바른 JSON이어야 합니다.",
  "bg.finished": "작업 완료",
  "bg.failed": "작업 실패",
  "bg.stopped": "작업 중지됨",
  "bg.missed": "작업을 놓침",
  "bg.interrupted": "작업 중단됨",
  "bg.approval": "승인 필요",
  "bg.cancelledBefore": "실행되기 전에 취소되었습니다.",
  "bg.stoppedByYou": "사용자가 중지했습니다.",
  "bg.done": "완료되었습니다.",
  "bg.noStart": "작업 창이 시작되지 않았습니다.",
  "bg.couldNotStart": "시작하지 못했습니다: {err}",
  "bg.missedMsg": "{when}에 실행될 예정이었지만 그 시각으로부터 {min}분 이내에 브라우저가 실행 중이지 않아 실행되지 않았습니다.",
  "bg.winClosed": "작업이 끝나기 전에 작업 창이 닫혔습니다.",
  "bg.restarted": "이 작업이 실행되는 동안 브라우저가 닫히거나 다시 시작되었습니다.",
  "sk.record": "스킬 녹화",
  "sk.list": "스킬",
  "sk.stopRec": "녹화 중지",
  "sk.recording": "● 녹화 중… 페이지에서 작업을 한 뒤 중지를 누르세요. ({n}단계)",
  "sk.recStarted": "녹화를 시작했습니다. 페이지에서 작업을 평소처럼 한 번 수행하세요. 비밀번호는 저장되지 않습니다.",
  "sk.recEmpty": "녹화된 내용이 없습니다.",
  "sk.click": "“{label}” 클릭",
  "sk.type": "“{label}”에 “{value}” 입력",
  "sk.typeSecret": "“{label}”에 비밀 값 입력 (직접 입력해야 함)",
  "sk.select": "“{label}”에서 “{value}” 선택",
  "sk.key": "“{label}”에서 {key} 키 누르기",
  "sk.edTitle": "스킬로 저장",
  "sk.name": "이름",
  "sk.desc": "하는 일",
  "sk.param": "매개변수",
  "sk.secret": "비밀 값: 저장되지 않음",
  "sk.delete": "삭제",
  "sk.useApi": "가능하면 직접 요청 사용 (훨씬 빠름)",
  "sk.apiInfo": "직접 요청을 찾았습니다: {req}",
  "sk.save": "스킬 저장",
  "sk.cancel": "취소",
  "sk.saved": "스킬 “{name}”을(를) 저장했습니다. 평소 말로 요청하거나 스킬 목록에서 실행하세요.",
  "sk.offer": "성공했습니다. 다음에 빠르게 실행되도록 이 단계를 스킬로 저장할까요?",
  "sk.offerYes": "스킬로 저장",
  "sk.offerNo": "괜찮습니다",
  "sk.listTitle": "스킬 ({n})",
  "sk.stepsN": "{n}단계",
  "sk.fast": "직접 요청",
  "sk.uses": "{n}회 사용",
  "sk.failN": "{n}회 실패",
  "sk.runNow": "지금 실행",
  "sk.deleteConfirm": "스킬 “{name}”을(를) 삭제할까요?",
  "sk.wrongSite": "이 스킬은 {host}용입니다. 먼저 해당 사이트를 여세요.",
  "sk.running": "스킬 실행: {name}",
  "sk.done": "스킬 “{name}” 완료.",
  "sk.direct": "직접 요청을 보내는 중…",
  "sk.directFailed": "직접 요청이 작동하지 않았습니다(HTTP {status}). 대신 단계를 재생합니다.",
  "act.skill": "스킬 “{name}” 실행",
  "why.skillApi": "이 스킬은 사이트의 데이터를 변경하는 요청을 보냅니다.",
  "sk.follows": "(“{param}” 값을 따름)",
  "sk.switched": "{host}에서 열려 있는 탭으로 전환합니다…",
  "sk.opening": "{host}을(를) 여는 중…",
  "sk.skipped": "처음 {n}단계는 이미 지났습니다. 여기서부터 계속합니다.",
};























//manifest.json
{
  "manifest_version": 3,
  "name": "Page Agent",
  "version": "1.0.3",
  "description": "Ask questions about the current page, or tell it to do things (fill forms, create records, book tickets), now or later, with approval before final actions.",
  "permissions": [
    "tabs",
    "scripting",
    "storage",
    "sidePanel",
    "alarms",
    "notifications"
  ],
  "host_permissions": [
    "<all_urls>",
    "https://api.anthropic.com/*"
  ],
  "background": {
    "service_worker": "background.js"
  },
  "side_panel": {
    "default_path": "sidepanel.html"
  },
  "options_page": "options.html",
  "action": {
    "default_title": "Page Agent",
    "default_icon": {
      "16": "icons/icon16.png",
      "32": "icons/icon32.png",
      "48": "icons/icon48.png"
    }
  },
  "icons": {
    "16": "icons/icon16.png",
    "32": "icons/icon32.png",
    "48": "icons/icon48.png",
    "128": "icons/icon128.png"
  }
}
