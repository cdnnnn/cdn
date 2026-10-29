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
  const list = skillsForHost(agentHost);
  if (!list.length) return "";
  return (
    "\n\nSaved skills for this site. When one matches the request, call run_skill with its name and ALL its parameters (take values from the user's request; use the default if the user gave none). It replays a recorded, proven path and is faster and more reliable than doing the steps yourself. If a skill fails, continue manually from where it stopped.\n" +
    list.map((s) => `- ${s.name}${s.description ? ": " + s.description : ""}. Parameters: ${s.params.length ? s.params.map((p) => `${p.name}${p.label ? ` (${p.label})` : ""}${p.default ? ` default "${String(p.default).slice(0, 40)}"` : ""}`).join(", ") : "none"}`).join("\n")
  );
}
const RUN_SKILL_TOOL = {
  name: "run_skill",
  description: "Run a saved skill (a recorded sequence of steps for this site). Give the skill name and a params object with a value for each parameter.",
  input_schema: { type: "object", properties: { name: { type: "string" }, params: { type: "object" } }, required: ["name"] },
};
const skillTools = () => (skillsForHost(agentHost).length ? [RUN_SKILL_TOOL] : []);

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
  if (hostOf(tab.url) !== sk.host) { addMessage("err", i18n("sk.wrongSite", { host: sk.host })); return; }
  const hint = document.querySelector(".hint");
  if (hint) hint.remove();
  addMessage("user", i18n("sk.running", { name: sk.name }));
  setRunning(true);
  try {
    agentTabId = tab.id;
    typedLog = [];
    const res = await executeSkill(sk, values, tab);
    addMessage(res.ok ? "note" : "err", res.ok ? i18n("sk.done", { name: sk.name }) + " " + (res.text || "") : res.text);
  } catch (e) {
    addMessage("err", e.message);
  } finally {
    setRunning(false);
  }
}

// ---------- running a skill ----------
async function doRunSkill(input, tab) {
  await skillsLoad();
  const host = hostOf(tab.url);
  const want = String(input.name || "").trim().toLowerCase();
  const sk = SKILLS.find((s) => s.host === host && (s.id === input.name || s.name.toLowerCase() === want));
  if (!sk) return { error: true, text: `No saved skill "${input.name}" for this site. Available: ${skillsForHost(host).map((s) => s.name).join(", ") || "(none)"}. Continue manually.` };
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
  for (let i = 0; i < sk.steps.length; i++) {
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
    let res = await tryStep(i === 0 ? 2500 : 6000);
    if (!res.ok && i === 0) {
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












//inpage.js
// Functions in this file are injected into the page via chrome.scripting.executeScript({func}).
// They must be fully self-contained (no references to outside variables).

function snapshotFn(maxText, maxEls, opts) {
  maxEls = maxEls || 200;
  // Shadow hosts are discovered ONCE per snapshot (a full "*" scan per query is far too slow on pages with thousands of nodes).
  let hostCache = null;
  function shadowHosts() {
    if (hostCache) return hostCache;
    hostCache = [];
    const stack = [document];
    while (stack.length) {
      const r = stack.pop();
      r.querySelectorAll("*").forEach((e) => { if (e.shadowRoot) { hostCache.push(e); stack.push(e.shadowRoot); } });
    }
    return hostCache;
  }
  function deepAll(selector, root, out) {
    root = root || document;
    out = out || [];
    root.querySelectorAll(selector).forEach((e) => out.push(e));
    if (root === document) for (const h of shadowHosts()) h.shadowRoot.querySelectorAll(selector).forEach((e) => out.push(e));
    return out;
  }

  deepAll("[data-aq-id]").forEach((e) => e.removeAttribute("data-aq-id"));
  const SEL =
    'a[href],button,input,select,textarea,summary,[role=button],[role=link],[role=checkbox],[role=radio],' +
    '[role=tab],[role=menuitem],[role=option],[role=combobox],[role=switch],[role=textbox],' +
    '[contenteditable=""],[contenteditable="true"],[onclick],[tabindex]:not([tabindex="-1"])';
  const SENS =
    /\bpass(word|code|phrase)\b|\bcard.?(number|no)\b|\bcc-(number|csc|exp)|\bcvv\b|\bcvc\b|\botp\b|one.?time.?(code|password)|\bssn\b|aadhaar|social.?security|contrase[ñn]a|clave de acceso|n[úu]mero de (tarjeta|cuenta)|mot de passe|code secret|num[ée]ro de carte|passwort|kennwort|kartennummer|prüfziffer|senha|n[úu]mero do cart[ãa]o|пароль|номер карты|одноразов|密码|口令|卡号|银行卡|验证码|身份证|パスワード|暗証|カード番号|認証コード|ワンタイム|كلمة (المرور|السر)|رقم البطاقة|رمز التحقق|पासवर्ड|कार्ड नंबर|ओटीपी|आधार|পাসওয়ার্ড|কার্ড নম্বর|ওটিপি|கடவுச்சொல்|அட்டை எண்|ఓటిపి|పాస్‌వర్డ్|పాస్వర్డ్|కార్డ్ నంబర్|पासवर्ड|કાર્ડ નંબર|પાસવર્ડ|ಪಾಸ್‌ವರ್ಡ್|ಪಾಸ್ವರ್ಡ್|ಕಾರ್ಡ್ ಸಂಖ್ಯೆ|പാസ്‌വേഡ്|പാസ്വേഡ്|കാർഡ് നമ്പർ|ਪਾਸਵਰਡ|ਕਾਰਡ ਨੰਬਰ|ପାସୱାର୍ଡ|କାର୍ଡ ନମ୍ବର|ଓଟିପି|비밀번호|패스워드|암호|카드 ?번호|인증 ?(번호|코드)|보안 ?코드|주민 ?등록|일회용/i;
  const clean = (s) => (s || "").replace(/\s+/g, " ").trim();
  const txt = (el) => (el.innerText !== undefined ? el.innerText : el.textContent) || "";
  const vh = window.innerHeight || 800;

  const isVisible = (el) => {
    const r = el.getBoundingClientRect();
    if (r.width < 2 || r.height < 2) return false;
    const cs = getComputedStyle(el);
    return cs.visibility !== "hidden" && cs.display !== "none" && cs.opacity !== "0";
  };

  const labelOf = (el) => {
    let l = el.getAttribute("aria-label");
    const lb = el.getAttribute("aria-labelledby");
    if (!l && lb) l = lb.split(" ").map((i) => { const n = document.getElementById(i); return n ? txt(n) : ""; }).join(" ");
    if (!l && el.labels && el.labels[0]) l = txt(el.labels[0]);
    if (!l && ["INPUT", "TEXTAREA", "SELECT"].includes(el.tagName)) l = el.placeholder || el.name || el.id;
    if (!l && el.tagName === "INPUT" && ["submit", "button"].includes(el.type)) l = el.value;
    if (!l) { const img = el.querySelector && el.querySelector("img[alt]"); l = txt(el) || el.title || el.alt || (img && img.alt) || ""; }
    l = clean(l);
    if (!l) {
      const svgTitle = el.querySelector && el.querySelector("svg title");
      l = clean(svgTitle ? svgTitle.textContent : "");
    }
    if (!l) {
      const hint = clean(el.getAttribute("data-testid") || el.getAttribute("data-test") || el.id || (typeof el.className === "string" ? el.className : ""));
      if (hint) l = "(icon: " + hint.slice(0, 40) + ")";
    }
    return l.slice(0, 80);
  };

  const base = new Set(deepAll(SEL));
  // Hidden checkbox/radio (custom switches): the visible label is what a user clicks.
  base.forEach((el) => {
    if (el.tagName === "INPUT" && (el.type === "checkbox" || el.type === "radio") && !isVisible(el) && el.labels && el.labels[0]) base.add(el.labels[0]);
  });
  // Clickable things with no semantic markup (div/span with a click handler show a pointer cursor).
  if (base.size <= 400) {
    const vw = window.innerWidth || 1200;
    let scanned = 0;
    for (const el of deepAll("div,span,li,label,svg,img,td,p,i,section,article")) {
      if (++scanned > 4000) break;
      if (base.has(el)) continue;
      if (getComputedStyle(el).cursor !== "pointer") continue;
      const par = el.parentElement;
      if (par && getComputedStyle(par).cursor === "pointer") continue;
      if (el.querySelector(SEL)) continue;
      const r = el.getBoundingClientRect();
      if (r.width * r.height > 0.5 * vw * vh) continue;
      base.add(el);
    }
  }
  const viewCache = new Map();
  const inView = (el) => { let v = viewCache.get(el); if (v === undefined) { const r = el.getBoundingClientRect(); v = r.bottom > 0 && r.top < vh; viewCache.set(el, v); } return v; };
  let cand = [...base].filter((el) => !(el.tagName === "INPUT" && el.type === "hidden"));
  cand.sort((a, b) => (a.compareDocumentPosition(b) & 4 ? -1 : 1));
  let nodes;
  if (cand.length > maxEls * 3) {
    // Huge page (thousands of cards/rows): only the first maxEls VISIBLE controls matter (on-screen first), so stop checking
    // computed styles once we have enough instead of testing every node.
    const a = [], b = [];
    for (const el of cand) (inView(el) ? a : b).push(el);
    nodes = [];
    for (const el of a) { if (nodes.length >= maxEls) break; if (isVisible(el)) nodes.push(el); }
    for (const el of b) { if (nodes.length >= maxEls) break; if (isVisible(el)) nodes.push(el); }
    nodes.sort((x, y) => (x.compareDocumentPosition(y) & 4 ? -1 : 1));
  } else {
    nodes = cand.filter(isVisible);
    if (nodes.length > maxEls) {
      const a = nodes.filter(inView);
      const b = nodes.filter((n) => !inView(n));
      const keep = new Set([...a, ...b].slice(0, maxEls));
      nodes = nodes.filter((n) => keep.has(n));
    }
  }

  // Compact "how to find this element again" descriptor, used when a successful run is saved as a skill.
  const okId = (id) => !!id && id.length < 60 && !/\d{3,}|^:|[:\s]/.test(id);
  const cssPath = (el) => {
    const parts = [];
    let n = el;
    for (let i = 0; n && n.nodeType === 1 && i < 6; i++, n = n.parentElement) {
      if (okId(n.id)) { parts.unshift("#" + String(n.id).replace(/[^\w-]/g, "\\$&")); break; }
      let s = n.tagName.toLowerCase();
      // position among same-tag siblings, counted by walking backwards (capped, so huge lists stay cheap)
      let idx = 1, steps = 0, sib = n.previousElementSibling, capped = false;
      while (sib && steps++ < 40) { if (sib.tagName === n.tagName) idx++; sib = sib.previousElementSibling; }
      if (sib) capped = true;
      if (!capped && (idx > 1 || (n.nextElementSibling && n.nextElementSibling.tagName === n.tagName))) s += ":nth-of-type(" + idx + ")";
      parts.unshift(s);
    }
    return parts.join(" > ");
  };
  const describeForSkill = (el, label) => {
    const row = el.closest && el.closest("tr,li,[role=row],[role=listitem],[role=dialog],dialog,form");
    let ctx = row && row !== el ? clean(txt(row)).slice(0, 50) : "";
    if (ctx === label) ctx = "";
    const d = {
      tag: el.tagName.toLowerCase(), role: el.getAttribute("role") || "", type: el.tagName === "INPUT" ? el.type : "",
      id: okId(el.id) ? el.id : "", name: el.getAttribute("name") || "",
      testid: el.getAttribute("data-testid") || el.getAttribute("data-test") || el.getAttribute("data-qa") || el.getAttribute("data-cy") || "",
      label: /^\(icon:/.test(label) ? "" : label, placeholder: el.getAttribute("placeholder") || "", ctx, nth: 0, css: "",
    };
    try { d.css = cssPath(el); } catch (e) {}
    if (el.tagName === "A" && el.getAttribute("href")) { try { d.href = new URL(el.href).pathname.slice(0, 100); } catch (e) {} }
    return d;
  };

  const elements = nodes.map((el, i) => {
    const id = i + 1;
    el.setAttribute("data-aq-id", String(id));
    const tag = el.tagName.toLowerCase();
    const type = tag === "input" ? (el.type || "text") : "";
    const label = labelOf(el);
    const sensitive =
      type === "password" || SENS.test([el.getAttribute("autocomplete"), el.name, el.id, label].join(" "));
    const o = { id, tag, role: el.getAttribute("role") || "", type, label, sensitive, inView: inView(el) };
    o.d = describeForSkill(el, label);
    if (!sensitive && (tag === "input" || tag === "textarea") && !["checkbox", "radio", "submit", "button"].includes(type)) {
      o.value = String(el.value || "").slice(0, 60);
    }
    if (tag === "select") {
      const sel = el.options[el.selectedIndex];
      o.value = sel ? clean(sel.text) : "";
      o.options = [...el.options].slice(0, 25).map((x) => clean(x.text));
    }
    if (tag === "label" && el.control && (el.control.type === "checkbox" || el.control.type === "radio")) { o.checked = !!el.control.checked; o.role = "toggle"; }
    else if (type === "checkbox" || type === "radio") o.checked = !!el.checked;
    else if (el.getAttribute("aria-checked")) o.checked = el.getAttribute("aria-checked") === "true";
    if (el.disabled || el.getAttribute("aria-disabled") === "true") o.disabled = true;
    if (el.required) o.required = true;
    if (tag === "a" && el.href) {
      try { const u = new URL(el.href); o.href = (u.host + u.pathname).slice(0, 80); } catch (e) {}
    }
    return o;
  });

  // Tour mode only: also expose page AREAS (nav, sidebar, header, tables, forms, headings...) so a guide can explain
  // the whole app, and tell each control which area/row it sits in so it is described accurately.
  let regions = [];
  if (opts && opts.tour) {
    const RSEL =
      "header,nav,aside,main,footer,section,form,table,dialog,h1,h2,h3," +
      "[role=banner],[role=navigation],[role=complementary],[role=main],[role=contentinfo],[role=region],[role=tablist],[role=toolbar]," +
      "[role=search],[role=grid],[role=table],[role=dialog],[role=menu],[role=tree]," +
      "[class*=sidebar],[class*=side-bar],[class*=navbar],[class*=toolbar],[class*=breadcrumb],[class*=drawer],[class*=modal],[class*=tabs]" +
      (opts.full ? ",h4,canvas,[class*=chart],[class*=card],[class*=widget],[class*=kpi],[class*=tile],[class*=panel],[class*=banner],[class*=alert]" : "");
    const KIND_OK = /^(header|nav|aside|main|footer|table|form|dialog|banner|navigation|complementary|contentinfo|tablist|toolbar|search|grid|menu|tree|canvas)$/;
    const regionEls = [];
    const regionOf = new Map();
    let n = elements.length;
    const vwid = window.innerWidth || 1200;
    for (const el of deepAll(RSEL)) {
      if (regions.length >= (opts.full ? 80 : 40)) break;
      if (el.hasAttribute("data-aq-id")) continue;
      const r = el.getBoundingClientRect();
      if (r.width < 60 || r.height < 24 || !isVisible(el)) continue;
      if (r.right <= 0 || r.left >= vwid) continue;
      const tag = el.tagName.toLowerCase();
      const kind = el.getAttribute("role") || (/^h[1-3]$/.test(tag) ? "heading" : tag);
      let label = clean(el.getAttribute("aria-label") || "");
      const lb = el.getAttribute("aria-labelledby");
      if (!label && lb) label = clean(lb.split(" ").map((i) => { const x = document.getElementById(i); return x ? txt(x) : ""; }).join(" "));
      if (!label && tag === "table") { const c = el.querySelector("caption"); label = clean(c ? txt(c) : ""); }
      if (!label && kind === "heading") label = clean(txt(el));
      if (!label) { const h = el.querySelector("h1,h2,h3,h4,legend"); label = clean(h ? txt(h) : ""); }
      if (!label && opts.full && !KIND_OK.test(kind)) label = clean(txt(el)).slice(0, 50); // cards/widgets: name them by their first text
      label = label.slice(0, 60);
      if (!label && !KIND_OK.test(kind)) continue; // an unlabeled generic <section>/div adds noise
      n++;
      el.setAttribute("data-aq-id", String(n));
      const reg = { id: n, kind, label };
      regions.push(reg);
      regionOf.set(el, reg);
      regionEls.push([el, reg]);
    }
    nodes.forEach((el, i) => {
      let p = el.parentElement || (el.getRootNode && el.getRootNode().host), g = 0;
      while (p && g++ < 14) {
        const reg = regionOf.get(p);
        if (reg && reg.kind !== "heading") { elements[i].area = reg.kind + (reg.label ? ' "' + reg.label + '"' : ""); break; }
        p = p.parentElement || (p.getRootNode && p.getRootNode().host);
      }
      const row = el.closest && el.closest("tr,li,[role=row],[role=listitem]");
      if (row) {
        const rt = clean(txt(row)).slice(0, 60);
        if (rt && rt !== elements[i].label) elements[i].ctx = rt;
      }
    });
    // reading order (document order) across controls AND areas, so a tour can walk the page top to bottom
    const seqItems = nodes.map((el, i) => [el, elements[i]]).concat(regionEls);
    seqItems.sort((a, b) => {
      if (a[0] === b[0]) return 0;
      const c = a[0].compareDocumentPosition(b[0]);
      return c & 4 ? -1 : c & 2 ? 1 : 0;
    });
    seqItems.forEach((x, k) => { x[1].seq = k; });
  }

  // Repeated structures (cards, rows, list items...). The model only ever sees a slice of a big page, so it needs to know
  // what repeats and how often, and can then use query_page to count/list them instantly instead of scrolling.
  let repeats = [], totalElements = 0;
  try {
    const all = document.body ? document.body.getElementsByTagName("*") : [];
    totalElements = all.length;
    const lim = Math.min(all.length, 60000);
    const cssEsc = (c) => (typeof CSS !== "undefined" && CSS.escape ? CSS.escape(c) : c.replace(/[^\w-]/g, "\\$&"));
    const keyCache = new Map();
    const counts = new Map();
    for (let i = 0; i < lim; i++) {
      const e = all[i];
      const cn = e.className;
      if (typeof cn !== "string" || !cn) continue;
      const ck = e.tagName + "|" + cn;
      let key = keyCache.get(ck);
      if (key === undefined) {
        const cls = cn.split(/\s+/).filter((c) => c && c.length <= 40 && !/\d/.test(c)).slice(0, 3);
        key = cls.length ? e.tagName.toLowerCase() + "." + cls.map((c) => cssEsc(c)).join(".") : "";
        keyCache.set(ck, key);
      }
      if (!key) continue;
      const c = counts.get(key);
      if (c) c.n++; else counts.set(key, { n: 1, first: e });
    }
    const cands = [...counts.entries()].filter(([, v]) => v.n >= 6).map(([sel, v]) => ({ sel, n: v.n, first: v.first, size: v.first.getElementsByTagName("*").length }));
    cands.sort((a, b) => b.n - a.n || b.size - a.size);
    repeats = cands.slice(0, 8).map((c) => {
      let n = c.n;
      try { n = document.querySelectorAll(c.sel).length; } catch (e) {}
      return { selector: c.sel, count: n, sample: clean(c.first.textContent || "").slice(0, 70) };
    });
  } catch (e) {}

  const body = (document.body ? txt(document.body) : "").replace(/\n{3,}/g, "\n\n");
  return {
    repeats,
    totalElements,
    url: location.href,
    title: document.title,
    scrollY: Math.round(window.scrollY),
    scrollMax: Math.max(0, Math.round(document.documentElement.scrollHeight - vh)),
    iframes: document.querySelectorAll("iframe").length,
    elements,
    regions,
    text: body.slice(0, maxText),
    totalChars: body.length,
  };
}

// Read-only aggregate query over the WHOLE page (not just the visible slice). Used for "how many ...", "list all ...", "which ... are ..." on big pages.
function queryFn(q) {
  const clean = (x) => (x || "").replace(/\s+/g, " ").trim();
  const out = { ok: true };
  try {
    const sel = String(q.selector || "").trim();
    if (!sel) return { ok: false, msg: "selector is required (a CSS selector such as .card or tr.row)." };
    const nodes = [];
    const walk = (root) => {
      root.querySelectorAll(sel).forEach((e) => nodes.push(e));
      root.querySelectorAll("*").forEach((e) => { if (e.shadowRoot) walk(e.shadowRoot); });
    };
    try { walk(document); } catch (e) { return { ok: false, msg: "Invalid CSS selector: " + e.message }; }
    let m = nodes;
    if (q.text_contains) {
      const needle = String(q.text_contains).toLowerCase();
      m = m.filter((e) => (e.textContent || "").toLowerCase().includes(needle));
    }
    if (q.visible_only) {
      m = m.filter((e) => {
        const r = e.getBoundingClientRect();
        if (r.width < 2 || r.height < 2) return false;
        const cs = getComputedStyle(e);
        return cs.visibility !== "hidden" && cs.display !== "none";
      });
    }
    out.count = m.length;
    out.matchedBeforeFilters = nodes.length;
    const nlist = Math.max(0, Math.min(Number(q.list) || 0, 50));
    if (nlist) out.items = m.slice(0, nlist).map((e) => clean(e.textContent).slice(0, 160));
    if (q.count_by) {
      const g = new Map();
      for (const e of m) {
        let t = "";
        try { const x = e.querySelector(String(q.count_by)); t = x ? clean(x.textContent).slice(0, 60) : "(none)"; } catch (err) { return { ok: false, msg: "Invalid count_by selector: " + err.message }; }
        g.set(t, (g.get(t) || 0) + 1);
      }
      out.groups = [...g.entries()].sort((a, b) => b[1] - a[1]).slice(0, 30).map(([value, n]) => ({ value, n }));
      out.distinct = g.size;
    }
    if (q.attribute) {
      const vals = m.slice(0, 50).map((e) => e.getAttribute(String(q.attribute))).filter((v) => v != null);
      out.attributes = vals.map((v) => String(v).slice(0, 120));
    }
    return out;
  } catch (e) {
    return { ok: false, msg: String(e && e.message || e) };
  }
}

// Resolves as soon as the page has stopped changing (no DOM mutations for quietMs), or after maxMs at the latest.
// Much faster than sleeping a fixed time after every action.
function quietFn(quietMs, maxMs) {
  return new Promise((resolve) => {
    const t0 = Date.now();
    let last = t0;
    const mo = new MutationObserver(() => { last = Date.now(); });
    try {
      mo.observe(document.documentElement, { subtree: true, childList: true, characterData: true, attributes: true, attributeFilter: ["class", "hidden", "open", "disabled", "aria-expanded", "aria-hidden", "aria-busy"] });
    } catch (e) {}
    const iv = setInterval(() => {
      const now = Date.now();
      if (now - last >= quietMs || now - t0 >= maxMs) { clearInterval(iv); mo.disconnect(); resolve(now - t0); }
    }, 25);
  });
}

function readFn(max) {
  const t = (document.body ? (document.body.innerText !== undefined ? document.body.innerText : document.body.textContent) : "") || "";
  const body = t.replace(/\n{3,}/g, "\n\n");
  return { text: body.slice(0, max), totalChars: body.length };
}

function actFn(name, a) {
  function deepAll(selector, root, out) {
    root = root || document;
    out = out || [];
    root.querySelectorAll(selector).forEach((e) => out.push(e));
    root.querySelectorAll("*").forEach((e) => { if (e.shadowRoot) deepAll(selector, e.shadowRoot, out); });
    return out;
  }
  const find = (id) => deepAll('[data-aq-id="' + id + '"]')[0] || null;
  const notFound = { ok: false, msg: "Element not found. The page changed; use ids from the latest snapshot." };
  const key = (el, k) => {
    for (const t of ["keydown", "keypress", "keyup"]) {
      el.dispatchEvent(new KeyboardEvent(t, { key: k, code: k, bubbles: true, cancelable: true }));
    }
  };
  const fire = (el) => {
    el.dispatchEvent(new Event("input", { bubbles: true }));
    el.dispatchEvent(new Event("change", { bubbles: true }));
  };
  try {
    if (name === "click") {
      const el = find(a.id);
      if (!el) return notFound;
      if (el.disabled) return { ok: false, msg: "Element is disabled." };
      if (el.scrollIntoView) el.scrollIntoView({ block: "center", inline: "center" });
      if (el.focus) el.focus({ preventScroll: true });
      try {
        const r = el.getBoundingClientRect();
        const o = { bubbles: true, cancelable: true, view: window, clientX: r.left + r.width / 2, clientY: r.top + r.height / 2, button: 0 };
        if (typeof PointerEvent === "function") el.dispatchEvent(new PointerEvent("pointerdown", { ...o, pointerType: "mouse", isPrimary: true }));
        el.dispatchEvent(new MouseEvent("mousedown", o));
        if (typeof PointerEvent === "function") el.dispatchEvent(new PointerEvent("pointerup", { ...o, pointerType: "mouse", isPrimary: true }));
        el.dispatchEvent(new MouseEvent("mouseup", o));
      } catch (e) {}
      el.click();
      return { ok: true, msg: "Clicked." };
    }
    if (name === "type_text") {
      const el = find(a.id);
      if (!el) return notFound;
      if (el.scrollIntoView) el.scrollIntoView({ block: "center" });
      if (el.focus) el.focus();
      const isTA = el instanceof HTMLTextAreaElement;
      if (isTA || el instanceof HTMLInputElement) {
        const proto = isTA ? HTMLTextAreaElement.prototype : HTMLInputElement.prototype;
        const set = Object.getOwnPropertyDescriptor(proto, "value").set;
        set.call(el, (a.append ? el.value : "") + a.text);
        fire(el);
      } else if (el.isContentEditable) {
        if (!a.append) document.execCommand("selectAll", false);
        document.execCommand("insertText", false, a.text);
      } else {
        return { ok: false, msg: "Element is not a text field." };
      }
      if (a.submit) {
        if (el.form && el.form.requestSubmit) el.form.requestSubmit();
        else key(el, "Enter");
      }
      return { ok: true, msg: "Typed." };
    }
    if (name === "select_option") {
      const el = find(a.id);
      if (!el) return notFound;
      if (el.tagName !== "SELECT") return { ok: false, msg: "Not a <select>. Click it and choose the option instead." };
      const want = String(a.option).trim().toLowerCase();
      const opts = [...el.options];
      const opt =
        opts.find((o) => o.text.trim().toLowerCase() === want || o.value.toLowerCase() === want) ||
        opts.find((o) => o.text.toLowerCase().includes(want));
      if (!opt) return { ok: false, msg: "No such option. Available: " + opts.slice(0, 25).map((o) => o.text.trim()).join(" | ") };
      const set = Object.getOwnPropertyDescriptor(HTMLSelectElement.prototype, "value").set;
      set.call(el, opt.value);
      fire(el);
      return { ok: true, msg: "Selected " + opt.text.trim() + "." };
    }
    if (name === "scroll") {
      const h = window.innerHeight || 800;
      if (a.direction === "top") window.scrollTo(0, 0);
      else if (a.direction === "bottom") window.scrollTo(0, document.documentElement.scrollHeight);
      else window.scrollBy(0, (a.direction === "up" ? -1 : 1) * Math.round(h * 0.8));
      return { ok: true, msg: "Scrolled." };
    }
    if (name === "press_key") {
      const el = document.activeElement || document.body;
      if (a.key === "Enter" && (el.tagName === "BUTTON" || el.tagName === "A")) el.click();
      else key(el, a.key);
      return { ok: true, msg: "Pressed " + a.key + "." };
    }
    return { ok: false, msg: "Unknown action " + name };
  } catch (e) {
    return { ok: false, msg: String(e) };
  }
}

// ---------- guided tour overlay (runs in the page; drawn in a shadow root, never touches the page's own DOM/styles) ----------
// steps: [{id?, title, text, click?}]  L: translated labels  o: {finished, rtl}
// A step that points at an element is only shown if that element is REALLY visible on screen (rendered, not clipped by a
// scroll area / collapsed panel, not covered). Otherwise the step is dropped, so the guide never highlights empty space.
// Reports back with chrome.runtime.sendMessage({__pa_tour:true, type:"advance"|"exit", reason}).
function tourFn(steps, L, o) {
  if (window.__paTour) window.__paTour.destroy();
  steps = steps.slice();
  const send = (m) => { try { chrome.runtime.sendMessage({ __pa_tour: true, ...m }); } catch (e) {} };
  const deepAll = (selector, root = document, out = []) => {
    root.querySelectorAll(selector).forEach((e) => out.push(e));
    root.querySelectorAll("*").forEach((e) => { if (e.shadowRoot) deepAll(selector, e.shadowRoot, out); });
    return out;
  };
  const findEl = (s) => (s && s.id != null ? deepAll('[data-aq-id="' + s.id + '"]')[0] || null : null);

  // ----- is this element really on screen? -----
  const okStyle = (c) => {
    if (!c || !c.isConnected || !c.getBoundingClientRect) return false;
    const cs = getComputedStyle(c);
    if (cs.display === "none" || cs.visibility === "hidden" || parseFloat(cs.opacity) < 0.05) return false;
    if (c.closest && c.closest("[hidden],[inert]")) return false;
    return true;
  };
  // visible part of the element after clipping by the viewport and by every overflow:hidden/auto/scroll ancestor
  const clipRect = (el, r) => {
    let Lf = Math.max(0, r.left), Tp = Math.max(0, r.top), Rt = Math.min(window.innerWidth, r.right), Bt = Math.min(window.innerHeight, r.bottom);
    let cur = el, p = el.parentNode, guard = 0;
    while (p && guard++ < 60) {
      if (p.nodeType === 11) { cur = p.host; p = p.host && p.host.parentNode; continue; }
      if (p.nodeType !== 1) break;
      if (getComputedStyle(cur).position === "fixed") break; // fixed elements ignore ancestors' overflow
      if (p !== document.documentElement && p !== document.body) {
        const cs = getComputedStyle(p);
        if (cs.overflowX !== "visible" || cs.overflowY !== "visible") {
          const pr = p.getBoundingClientRect();
          Lf = Math.max(Lf, pr.left); Tp = Math.max(Tp, pr.top); Rt = Math.min(Rt, pr.right); Bt = Math.min(Bt, pr.bottom);
        }
      }
      cur = p; p = p.parentNode;
    }
    return { left: Lf, top: Tp, right: Rt, bottom: Bt, width: Math.max(0, Rt - Lf), height: Math.max(0, Bt - Tp) };
  };
  const covered = (c, cr) => {
    if (typeof document.elementFromPoint !== "function") return false;
    const pts = [[0.5, 0.5], [0.2, 0.3], [0.8, 0.7]];
    for (const [fx, fy] of pts) {
      const t = document.elementFromPoint(cr.left + cr.width * fx, cr.top + cr.height * fy);
      if (!t) return false;
      const host = c.getRootNode && c.getRootNode().host;
      if (t === c || c.contains(t) || t.contains(c) || (host && (host === t || host.contains(t)))) return false;
    }
    return true;
  };
  // returns {el, rect} or null. With scroll=true, scrolls the element into view first (also inside scroll containers).
  const locate = (s, scroll) => {
    const el0 = findEl(s);
    if (!el0) return null;
    const cands = [el0];
    if (el0.tagName === "INPUT" && ["checkbox", "radio", "file"].includes(el0.type)) {
      if (el0.labels && el0.labels[0]) cands.push(el0.labels[0]);
      const lab = el0.closest && el0.closest("label");
      if (lab) cands.push(lab);
    }
    for (const c of cands) {
      if (!okStyle(c)) continue;
      let r = c.getBoundingClientRect();
      if (r.width < 4 || r.height < 4) continue;
      const need = (rr) => 0.5 * Math.min(rr.width * rr.height, 0.4 * window.innerWidth * window.innerHeight);
      let cr = clipRect(c, r);
      if (scroll && cr.width * cr.height < need(r)) {
        try { c.scrollIntoView({ block: "center", inline: "center", behavior: "instant" }); } catch (e) {}
        r = c.getBoundingClientRect();
        cr = clipRect(c, r);
      }
      if (cr.width < 4 || cr.height < 4 || cr.width * cr.height < need(r)) continue;
      if (covered(c, cr)) continue;
      return { el: c, orig: el0, rect: cr };
    }
    return null;
  };

  // ----- UI -----
  const host = document.createElement("div");
  host.setAttribute("data-pa-tour", "1");
  host.style.cssText = "all:initial;position:fixed;inset:0;z-index:2147483647;pointer-events:none;";
  host.dir = o && o.rtl ? "rtl" : "ltr";
  const root = host.attachShadow({ mode: "open" });
  const style = document.createElement("style");
  style.textContent = `
    :host{all:initial}
    *{box-sizing:border-box;font-family:system-ui,-apple-system,"Segoe UI",Roboto,sans-serif}
    .dim{position:fixed;inset:0;background:rgba(15,23,42,.55);pointer-events:none}
    .ring{position:fixed;border:3px solid #3b82f6;border-radius:8px;box-shadow:0 0 0 9999px rgba(15,23,42,.55);pointer-events:none;transition:all .18s ease}
    .tip{position:fixed;width:min(320px,calc(100vw - 24px));background:#fff;color:#1f2328;border-radius:12px;padding:12px 14px;box-shadow:0 10px 30px rgba(0,0,0,.35);pointer-events:auto;font-size:14px;line-height:1.45}
    .top{display:flex;justify-content:space-between;align-items:center;gap:8px;margin-bottom:4px}
    .count{font-size:12px;color:#57606a}
    .x{all:unset;cursor:pointer;font-size:18px;line-height:1;padding:2px 6px;border-radius:6px;color:#57606a}
    .x:hover{background:#eef2f7}
    h3{margin:0 0 4px;font-size:15px}
    p{margin:0 0 8px;white-space:pre-wrap;overflow-wrap:anywhere}
    .hintc{font-size:12px;color:#0b57d0;margin-bottom:8px}
    .btns{display:flex;justify-content:flex-end;gap:8px}
    button.b{all:unset;cursor:pointer;padding:6px 14px;border-radius:8px;font-size:13px;font-weight:600;background:#eef2f7;color:#1f2328}
    button.b.next{background:#0b57d0;color:#fff}
    button.b:focus-visible{outline:2px solid #0b57d0;outline-offset:2px}
    [hidden]{display:none!important}
    @media (prefers-color-scheme:dark){.tip{background:#1e2530;color:#e6edf3}.count,.x{color:#9aa7b4}.x:hover{background:#2b3544}.hintc{color:#7db4ff}button.b{background:#2b3544;color:#e6edf3}button.b.next{background:#2f6fe0;color:#fff}}
  `;
  const mk = (tag, cls, text) => { const e = document.createElement(tag); if (cls) e.className = cls; if (text != null) e.textContent = text; return e; };
  const dim = mk("div", "dim"), ring = mk("div", "ring"), tip = mk("div", "tip");
  tip.setAttribute("role", "dialog");
  tip.setAttribute("aria-live", "polite");
  const count = mk("span", "count"), xBtn = mk("button", "x", "×");
  xBtn.setAttribute("aria-label", L.exit);
  xBtn.title = L.exit;
  const top = mk("div", "top");
  top.append(count, xBtn);
  const h = mk("h3"), p = mk("p"), hint = mk("div", "hintc");
  const back = mk("button", "b back", L.back), next = mk("button", "b next");
  const btns = mk("div", "btns");
  btns.append(back, next);
  tip.append(top, h, p, hint, btns);
  root.append(style, dim, ring, tip);
  document.documentElement.appendChild(host);

  let i = 0, waiting = false, clickEls = [], clickFn = null, timer = null, shown = 0;
  const detach = () => { if (clickFn) clickEls.forEach((e) => e.removeEventListener("click", clickFn, true)); clickEls = []; clickFn = null; };

  function position() {
    if (waiting || !steps[i]) return;
    const s = steps[i];
    const loc = s.id != null ? locate(s, false) : null;
    const vw = window.innerWidth, vh = window.innerHeight, pad = 6, gap = 12;
    const tw = tip.offsetWidth || 320, th = tip.offsetHeight || 140;
    if (!loc) {
      ring.hidden = true; dim.hidden = false;
      tip.style.left = Math.max(12, (vw - tw) / 2) + "px";
      tip.style.top = Math.max(12, (vh - th) / 2) + "px";
      return;
    }
    const r = loc.rect;
    dim.hidden = true; ring.hidden = false;
    ring.style.left = r.left - pad + "px"; ring.style.top = r.top - pad + "px";
    ring.style.width = r.width + pad * 2 + "px"; ring.style.height = r.height + pad * 2 + "px";
    let ty = r.bottom + pad + gap;
    if (ty + th > vh - 8) ty = r.top - pad - gap - th;
    if (ty < 8) ty = Math.min(vh - th - 8, Math.max(8, r.bottom + pad + gap)); // no room above or below: overlap rather than leave the screen
    let tx = r.left + r.width / 2 - tw / 2;
    tx = Math.max(12, Math.min(vw - tw - 12, tx));
    tip.style.left = tx + "px"; tip.style.top = ty + "px";
  }

  function paint(loc) {
    detach();
    waiting = false;
    const s = steps[i], last = i === steps.length - 1;
    count.textContent = L.step.replace("{n}", ((o && o.offset) || 0) + i + 1).replace("{total}", ((o && o.offset) || 0) + steps.length + ((o && o.rest) || 0));
    h.textContent = s.title || ""; h.hidden = !s.title;
    p.textContent = s.text || "";
    const wantsClick = !!(s.click && loc);
    hint.textContent = wantsClick ? L.click : ""; hint.hidden = !wantsClick;
    back.hidden = i === 0;
    next.textContent = last && o && o.finished ? L.done : L.next;
    btns.hidden = false; xBtn.hidden = false;
    if (wantsClick) {
      clickEls = loc.orig === loc.el ? [loc.el] : [loc.el, loc.orig];
      clickFn = () => { setTimeout(() => advance("click"), 0); }; // let the page handle the click first
      clickEls.forEach((e) => e.addEventListener("click", clickFn, true));
    }
    position();
    try { next.focus({ preventScroll: true }); } catch (e) {}
  }

  // Show the first step at/after (dir>0) or at/before (dir<0) index k whose target can really be seen; drop the others.
  function show(k, dir) {
    while (k >= 0 && k < steps.length) {
      const s = steps[k];
      const loc = s.id == null ? null : locate(s, true);
      if (s.id == null || loc) { i = k; paint(loc); return true; }
      steps.splice(k, 1);
      if (k < i) i--; // keep i pointing at the step currently on screen
      if (dir < 0) k--;
    }
    return false;
  }

  function advance(reason) {
    detach();
    if (o && o.finished && reason === "next") { send({ type: "advance", reason, last: true, shown }); return destroy(); }
    // more to show: wait for the extension to prepare the next part
    waiting = true;
    ring.hidden = true; dim.hidden = true;
    count.textContent = ""; h.hidden = true; hint.hidden = true; btns.hidden = true; xBtn.hidden = true;
    p.textContent = L.wait;
    tip.style.left = Math.max(12, window.innerWidth - tip.offsetWidth - 16) + "px";
    tip.style.top = "16px";
    send({ type: "advance", reason, shown });
  }

  const onKey = (e) => { if (e.key === "Escape") { send({ type: "exit" }); destroy(); } };
  function destroy() {
    detach();
    clearInterval(timer);
    window.removeEventListener("keydown", onKey, true);
    window.removeEventListener("resize", position);
    window.removeEventListener("scroll", position, true);
    host.remove();
    if (window.__paTour && window.__paTour.host === host) window.__paTour = null;
  }
  next.onclick = () => { if (!show(i + 1, 1)) advance("next"); };
  back.onclick = () => { show(i - 1, -1); };
  xBtn.onclick = () => { send({ type: "exit" }); destroy(); };
  window.addEventListener("keydown", onKey, true);
  window.addEventListener("resize", position);
  window.addEventListener("scroll", position, true);
  timer = setInterval(position, 250);
  window.__paTour = { destroy, host };
  // Drop steps whose target cannot really be shown right now, up front, so "Step n of N" is honest and nothing points at empty space.
  steps = steps.filter((s) => s.id == null || locate(s, true));
  shown = steps.length;
  if (!show(0, 1)) advance("empty");
  return true;
}

function tourClearFn() {
  if (window.__paTour) window.__paTour.destroy();
  document.querySelectorAll("[data-pa-tour]").forEach((e) => e.remove());
  return true;
}

// ======================= Skills: recording, replay, direct requests =======================
// Everything below is injected into the page with chrome.scripting.executeScript, so each function is self-contained.

// Records what the user does on the page (clicks, typed values, selects, keys) plus mutating network requests.
// cmd: { cmd: "start", resume?: boolean } | { cmd: "dump" } | { cmd: "stop" }
function recorderFn(c) {
  const KEY = "__paRecState";
  const W = window;
  const clean = (s) => (s || "").replace(/\s+/g, " ").trim();
  const txt = (el) => clean(el.innerText !== undefined ? el.innerText : el.textContent);
  const okId = (id) => !!id && id.length < 60 && !/\d{3,}|^:|[:\s]/.test(id);
  const labelOf = (el) => {
    let l = el.getAttribute("aria-label");
    const lb = el.getAttribute("aria-labelledby");
    if (!l && lb) l = lb.split(" ").map((i) => { const n = document.getElementById(i); return n ? txt(n) : ""; }).join(" ");
    if (!l && el.labels && el.labels[0]) l = txt(el.labels[0]);
    if (!l && ["INPUT", "TEXTAREA", "SELECT"].includes(el.tagName)) l = el.placeholder || el.name || (okId(el.id) ? el.id : "");
    if (!l && el.tagName === "INPUT" && ["submit", "button"].includes(el.type)) l = el.value;
    if (!l) { const img = el.querySelector && el.querySelector("img[alt]"); l = txt(el) || el.title || el.alt || (img && img.alt) || ""; }
    return clean(l).slice(0, 80);
  };
  const cssPath = (el) => {
    const parts = [];
    let n = el;
    for (let i = 0; n && n.nodeType === 1 && i < 6; i++, n = n.parentElement) {
      if (okId(n.id)) { parts.unshift("#" + String(n.id).replace(/[^\w-]/g, "\\$&")); break; }
      let s = n.tagName.toLowerCase();
      const p = n.parentElement;
      if (p) { const same = [...p.children].filter((x) => x.tagName === n.tagName); if (same.length > 1) s += ":nth-of-type(" + (same.indexOf(n) + 1) + ")"; }
      parts.unshift(s);
    }
    return parts.join(" > ");
  };
  const describe = (el) => {
    const label = labelOf(el);
    const row = el.closest("tr,li,[role=row],[role=listitem],[role=dialog],dialog,form");
    let ctx = row && row !== el ? txt(row).slice(0, 50) : "";
    if (ctx === label) ctx = "";
    let nth = 0, same = 1;
    try {
      const all = [...document.querySelectorAll(el.tagName.toLowerCase())].filter((x) => labelOf(x) === label);
      same = all.length; nth = Math.max(0, all.indexOf(el));
    } catch (e) {}
    const d = {
      tag: el.tagName.toLowerCase(), role: el.getAttribute("role") || "", type: el.tagName === "INPUT" ? el.type : "",
      id: okId(el.id) ? el.id : "", name: el.getAttribute("name") || "",
      testid: el.getAttribute("data-testid") || el.getAttribute("data-test") || el.getAttribute("data-qa") || el.getAttribute("data-cy") || "",
      label, placeholder: el.getAttribute("placeholder") || "", ctx, nth, same, css: cssPath(el),
    };
    if (el.tagName === "A" && el.getAttribute("href")) { try { d.href = new URL(el.href).pathname.slice(0, 100); } catch (e) {} }
    return d;
  };
  const save = () => {
    try {
      const R = W.__paRec;
      sessionStorage.setItem(KEY, JSON.stringify({ steps: R.steps, reqs: R.reqs.slice(-40) }));
    } catch (e) {}
  };

  if (c.cmd === "dump" || c.cmd === "stop") {
    const R = W.__paRec;
    let out;
    if (R) out = { steps: R.steps, reqs: R.reqs, url: location.href, title: document.title };
    else { try { out = Object.assign({ url: location.href, title: document.title }, JSON.parse(sessionStorage.getItem(KEY) || "{}")); } catch (e) { out = { steps: [], reqs: [], url: location.href, title: document.title }; } }
    if (c.cmd === "stop") {
      if (R) { R.on = false; (R.off || []).forEach((f) => { try { f(); } catch (e) {} }); }
      try { sessionStorage.removeItem(KEY); } catch (e) {}
    }
    return out;
  }

  // ---- start / resume
  if (W.__paRec && W.__paRec.on) return { ok: true, already: true };
  let prev = null;
  if (c.resume) { try { prev = JSON.parse(sessionStorage.getItem(KEY) || "null"); } catch (e) {} }
  const R = (W.__paRec = { on: true, steps: (prev && prev.steps) || [], reqs: (prev && prev.reqs) || [], muts: 0, off: [] });
  const mo = new MutationObserver((list) => { R.muts += list.length; });
  mo.observe(document.documentElement, { subtree: true, childList: true, characterData: true });
  R.off.push(() => mo.disconnect());

  const push = (st) => {
    st.at = Date.now(); st.url = location.href;
    const m0 = R.muts, u0 = location.href;
    R.steps.push(st);
    setTimeout(() => { st.effect = R.muts - m0 > 2 || location.href !== u0; save(); }, 1200);
    save();
  };
  const isTextual = (el) => el.tagName === "TEXTAREA" || (el.tagName === "INPUT" && !["checkbox", "radio", "button", "submit", "reset", "file", "image", "range", "color"].includes(el.type));
  const flush = (el) => {
    if (!el || !(isTextual(el))) return;
    const sensitive = el.type === "password";
    const val = sensitive ? "(secret)" : el.value;
    if (el.__paLast === val || (el.__paLast === undefined && !el.value)) { el.__paLast = val; return; }
    el.__paLast = val;
    const d = describe(el);
    const last = R.steps[R.steps.length - 1];
    if (last && last.t === "type" && last.d.css === d.css && last.d.label === d.label) { last.value = sensitive ? "" : el.value; last.at = Date.now(); save(); return; }
    push({ t: "type", d, value: sensitive ? "" : el.value, sensitive });
  };
  const onChange = (e) => {
    const el = e.target;
    if (!el || !el.tagName) return;
    if (el.tagName === "SELECT") {
      const o = el.options[el.selectedIndex];
      push({ t: "select", d: describe(el), value: o ? clean(o.text) : el.value });
    } else flush(el);
  };
  const CLICKABLE = 'a,button,input,select,textarea,summary,label,[role=button],[role=link],[role=tab],[role=menuitem],[role=option],[role=checkbox],[role=radio],[role=switch],[role=treeitem],[role=gridcell],[onclick],li,[tabindex]';
  const onClick = (e) => {
    if (!e.isTrusted && !W.__paAllowSynthetic) return;
    flush(document.activeElement);
    let el = e.target && e.target.closest ? e.target.closest(CLICKABLE) : null;
    if (!el) el = e.target;
    if (!el || !el.tagName) return;
    if (isTextual(el) || el.tagName === "SELECT" || el.isContentEditable) return; // focusing a field is not a step; its value is
    const last = R.steps[R.steps.length - 1];
    if (el.tagName === "INPUT" && last && last.t === "click" && Date.now() - last.at < 150 && last.d.tag === "label") return;
    push({ t: "click", d: describe(el) });
  };
  const KEYS = ["Enter", "Escape", "ArrowDown", "ArrowUp"];
  const onKey = (e) => {
    if ((!e.isTrusted && !W.__paAllowSynthetic) || !KEYS.includes(e.key)) return;
    const el = e.target;
    if (!el || !el.tagName) return;
    if (e.key === "Enter" && !(el.tagName === "INPUT" || el.getAttribute("role") === "combobox" || el.getAttribute("role") === "textbox")) return;
    flush(el);
    push({ t: "key", key: e.key, d: describe(el) });
  };
  const onBlurCE = (e) => {
    const el = e.target;
    if (el && el.isContentEditable && el.tagName) {
      const v = clean(el.textContent);
      if (el.__paLast !== v && v) { el.__paLast = v; push({ t: "type", d: describe(el), value: v, sensitive: false }); }
    }
  };
  document.addEventListener("change", onChange, true);
  document.addEventListener("click", onClick, true);
  document.addEventListener("keydown", onKey, true);
  document.addEventListener("focusout", onBlurCE, true);
  R.off.push(() => {
    document.removeEventListener("change", onChange, true);
    document.removeEventListener("click", onClick, true);
    document.removeEventListener("keydown", onKey, true);
    document.removeEventListener("focusout", onBlurCE, true);
  });

  // ---- network: remember requests that change data (POST/PUT/PATCH/DELETE)
  const keepHeaders = (h) => {
    const out = {};
    try {
      const put = (k, v) => { k = String(k).toLowerCase(); if (k === "content-type" || k === "accept" || k.startsWith("x-")) out[k] = String(v); };
      if (!h) return out;
      if (typeof h.forEach === "function" && !Array.isArray(h)) h.forEach((v, k) => put(k, v));
      else if (Array.isArray(h)) h.forEach(([k, v]) => put(k, v));
      else Object.keys(h).forEach((k) => put(k, h[k]));
    } catch (e) {}
    return out;
  };
  const note = (method, url, headers, body) => {
    method = String(method || "GET").toUpperCase();
    if (["GET", "HEAD", "OPTIONS"].includes(method) || !R.on) return null;
    let abs = url;
    try { abs = new URL(url, location.href).href; } catch (e) {}
    let b = null;
    if (typeof body === "string") b = body.slice(0, 20000);
    else if (body && typeof body.toString === "function" && body instanceof URLSearchParams) b = body.toString();
    const r = { method, url: abs, headers: keepHeaders(headers), body: b, opaque: body != null && b == null, at: Date.now(), status: 0 };
    R.reqs.push(r);
    if (R.reqs.length > 80) R.reqs.shift();
    save();
    return r;
  };
  if (!W.__paNetPatched) {
    W.__paNetPatched = true;
    const of = W.fetch;
    W.fetch = function (input, init) {
      let rec = null;
      try {
        const f = W.__paNoteFetch;
        if (f) {
          const isReq = typeof Request !== "undefined" && input instanceof Request;
          rec = f(isReq ? input.method : init && init.method, isReq ? input.url : String(input), (init && init.headers) || (isReq ? input.headers : null), init && init.body);
        }
      } catch (e) {}
      const p = of.apply(this, arguments);
      if (rec) p.then((resp) => { rec.status = resp.status; }, () => {});
      return p;
    };
    const XO = XMLHttpRequest.prototype.open, XS = XMLHttpRequest.prototype.send, XH = XMLHttpRequest.prototype.setRequestHeader;
    XMLHttpRequest.prototype.open = function (m, u) { this.__pa = { m, u, h: {} }; return XO.apply(this, arguments); };
    XMLHttpRequest.prototype.setRequestHeader = function (k, v) { if (this.__pa) this.__pa.h[k] = v; return XH.apply(this, arguments); };
    XMLHttpRequest.prototype.send = function (body) {
      try {
        const f = W.__paNoteFetch;
        const rec = this.__pa && f ? f(this.__pa.m, this.__pa.u, this.__pa.h, body) : null;
        if (rec) this.addEventListener("loadend", () => { rec.status = this.status; });
      } catch (e) {}
      return XS.apply(this, arguments);
    };
  }
  W.__paNoteFetch = note;
  return { ok: true };
}

// Finds one recorded step's element on the current page and performs it. step = { t, d, value?, key? }.
async function replayStepFn(step, value, timeoutMs) {
  const clean = (s) => (s || "").replace(/\s+/g, " ").trim();
  const norm = (s) => clean(s).toLowerCase();
  const txt = (el) => clean(el.innerText !== undefined ? el.innerText : el.textContent);
  const okId = (id) => !!id && id.length < 60 && !/\d{3,}|^:|[:\s]/.test(id);
  const labelOf = (el) => {
    let l = el.getAttribute("aria-label");
    const lb = el.getAttribute("aria-labelledby");
    if (!l && lb) l = lb.split(" ").map((i) => { const n = document.getElementById(i); return n ? txt(n) : ""; }).join(" ");
    if (!l && el.labels && el.labels[0]) l = txt(el.labels[0]);
    if (!l && ["INPUT", "TEXTAREA", "SELECT"].includes(el.tagName)) l = el.placeholder || el.name || (okId(el.id) ? el.id : "");
    if (!l && el.tagName === "INPUT" && ["submit", "button"].includes(el.type)) l = el.value;
    if (!l) { const img = el.querySelector && el.querySelector("img[alt]"); l = txt(el) || el.title || el.alt || (img && img.alt) || ""; }
    return clean(l).slice(0, 80);
  };
  const visible = (el) => {
    const r = el.getBoundingClientRect();
    if (r.width < 2 || r.height < 2) return false;
    const cs = getComputedStyle(el);
    return cs.visibility !== "hidden" && cs.display !== "none";
  };
  const deepQ = (sel) => {
    const out = [...document.querySelectorAll(sel)];
    const stack = [document];
    while (stack.length) {
      const r = stack.pop();
      r.querySelectorAll("*").forEach((e) => { if (e.shadowRoot) { e.shadowRoot.querySelectorAll(sel).forEach((x) => out.push(x)); stack.push(e.shadowRoot); } });
    }
    return out;
  };
  const d = step.d || {};
  const find = () => {
    const attr = (n, v) => '[' + n + '="' + String(v).replace(/"/g, '\\"') + '"]';
    // exact identifiers first
    if (d.testid) {
      const q = deepQ(["data-testid", "data-test", "data-qa", "data-cy"].map((n) => attr(n, d.testid)).join(",")).filter(visible);
      if (q.length === 1) return q[0];
    }
    if (d.id) { const e = document.getElementById(d.id); if (e && visible(e)) return e; }
    // otherwise score the candidates of the same tag/role
    const sel = [d.tag || "*"].concat(d.role ? ['[role="' + d.role + '"]'] : []).join(",");
    const nl = norm(d.label), nc = norm(d.ctx).slice(0, 24);
    let best = [], bs = 0;
    for (const el of deepQ(sel).slice(0, 4000)) {
      let s = 0;
      if (d.testid && (el.getAttribute("data-testid") === d.testid || el.getAttribute("data-test") === d.testid)) s += 10;
      if (d.name && el.getAttribute("name") === d.name) s += 5;
      if (d.id && el.id === d.id) s += 8;
      const l = nl ? norm(labelOf(el)) : "";
      if (nl && l === nl) s += 6; else if (nl && l && (l.startsWith(nl) || nl.startsWith(l))) s += 3;
      if (d.placeholder && el.getAttribute("placeholder") === d.placeholder) s += 2;
      if (d.href && el.tagName === "A") { try { if (new URL(el.href).pathname === d.href) s += 4; } catch (e) {} }
      if (s < 4) continue;
      if (d.type && el.type === d.type) s += 1;
      if (nc) { const row = el.closest("tr,li,[role=row],[role=listitem],[role=dialog],dialog,form"); if (row && norm(txt(row)).includes(nc)) s += 3; }
      if (!visible(el)) continue;
      if (s > bs) { bs = s; best = [el]; } else if (s === bs) best.push(el);
    }
    if (best.length) return best[Math.min(d.nth || 0, best.length - 1)] || best[0];
    if (d.css) { try { const e = document.querySelector(d.css); if (e && visible(e) && (!nl || norm(labelOf(e)) === nl || !labelOf(e))) return e; } catch (e) {} }
    return null;
  };
  // Dynamic target: the element is named after a parameter value (a typeahead suggestion, a row) rather than the recorded label.
  const findDyn = () => {
    const toks = norm(d.dynValue).split(" ").filter(Boolean);
    if (!toks.length) return null;
    const want = norm(d.dynValue);
    let best = null, bs = -1e9;
    for (const el of deepQ('[role=option],[role=menuitem],[role=treeitem],[role=row],[role=link],[role=button],li,a,button,div,span,td,tr,p,label,h1,h2,h3,h4').slice(0, 8000)) {
      const text = norm(txt(el));
      if (!text || text.length > 220 || !toks.every((t) => text.includes(t))) continue;
      if (!visible(el)) continue;
      let s = 10 - Math.max(0, text.length - want.length) / 15;
      if (text === want) s += 4;
      if (el.closest('[role=listbox],[role=menu],[role=tree],[class*="suggest" i],[class*="option" i],[class*="dropdown" i],[class*="autocomplete" i],[class*="popup" i],[class*="result" i]')) s += 3;
      if (el.tagName.toLowerCase() === d.tag) s += 2;
      if (d.role && el.getAttribute("role") === d.role) s += 2;
      if (d.ctx) { const row = el.closest("tr,li,[role=row],[role=listitem],[role=dialog],dialog,form"); if (row && norm(txt(row)).includes(norm(d.ctx).slice(0, 24))) s += 1; }
      if (s > bs) { bs = s; best = el; }
    }
    return best;
  };
  const t0 = Date.now(), limit = timeoutMs || 6000;
  const bare = step.t === "key" && !d.tag && !d.css;
  const look = () => (bare ? document.activeElement || document.body : d.dynValue ? findDyn() : find());
  let el = look();
  while (!el && Date.now() - t0 < limit) { await new Promise((r) => setTimeout(r, 200)); el = look(); }
  if (!el) return { ok: false, msg: 'Could not find "' + (d.label || d.css || d.tag) + '" (' + step.t + ") on this page." };
  if (el.scrollIntoView) { try { el.scrollIntoView({ block: "center", inline: "center" }); } catch (e) {} }
  const press = (t) => {
    try {
      const r = t.getBoundingClientRect();
      const o = { bubbles: true, cancelable: true, view: window, clientX: r.left + r.width / 2, clientY: r.top + r.height / 2, button: 0 };
      if (typeof PointerEvent === "function") t.dispatchEvent(new PointerEvent("pointerdown", { ...o, pointerType: "mouse", isPrimary: true }));
      t.dispatchEvent(new MouseEvent("mousedown", o));
      if (typeof PointerEvent === "function") t.dispatchEvent(new PointerEvent("pointerup", { ...o, pointerType: "mouse", isPrimary: true }));
      t.dispatchEvent(new MouseEvent("mouseup", o));
    } catch (e) {}
    t.click();
  };
  const fire = (e) => { e.dispatchEvent(new Event("input", { bubbles: true })); e.dispatchEvent(new Event("change", { bubbles: true })); };
  try {
    if (step.t === "click") {
      if (el.disabled) return { ok: false, msg: '"' + (d.label || "element") + '" is disabled.' };
      if (el.focus) el.focus({ preventScroll: true });
      press(el);
      return { ok: true, msg: "Clicked." };
    }
    if (step.t === "type") {
      if (el.focus) el.focus();
      const v = value == null ? step.value || "" : String(value);
      const isTA = el instanceof HTMLTextAreaElement;
      if (isTA || el instanceof HTMLInputElement) {
        const proto = isTA ? HTMLTextAreaElement.prototype : HTMLInputElement.prototype;
        Object.getOwnPropertyDescriptor(proto, "value").set.call(el, v);
        fire(el);
        if (el.value !== v) return { ok: false, msg: "The field did not accept the value." };
        const lastCh = v.slice(-1) || " ";
        for (const ty of ["keydown", "keypress", "keyup"]) el.dispatchEvent(new KeyboardEvent(ty, { key: lastCh, bubbles: true, cancelable: true }));
      } else if (el.isContentEditable) {
        document.execCommand("selectAll", false);
        document.execCommand("insertText", false, v);
      } else return { ok: false, msg: "Not a text field." };
      return { ok: true, msg: "Typed." };
    }
    if (step.t === "select") {
      if (el.tagName !== "SELECT") return { ok: false, msg: "Not a <select>." };
      const want = norm(value == null ? step.value : value);
      const opts = [...el.options];
      const opt = opts.find((o) => norm(o.text) === want || norm(o.value) === want) || opts.find((o) => norm(o.text).includes(want));
      if (!opt) return { ok: false, msg: 'No option "' + want + '". Available: ' + opts.slice(0, 20).map((o) => clean(o.text)).join(" | ") };
      Object.getOwnPropertyDescriptor(HTMLSelectElement.prototype, "value").set.call(el, opt.value);
      fire(el);
      return { ok: true, msg: "Selected." };
    }
    if (step.t === "key") {
      if (el.focus) el.focus();
      for (const ty of ["keydown", "keypress", "keyup"]) el.dispatchEvent(new KeyboardEvent(ty, { key: step.key, code: step.key, bubbles: true, cancelable: true }));
      if (step.key === "Enter" && el.form && el.tagName === "INPUT" && el.form.requestSubmit) { try { el.form.requestSubmit(); } catch (e) {} }
      return { ok: true, msg: "Pressed " + step.key + "." };
    }
    return { ok: false, msg: "Unknown step type " + step.t };
  } catch (e) {
    return { ok: false, msg: String(e && e.message || e) };
  }
}

// A cheap fingerprint of the page, to check that a step really changed something.
function pageSigFn() {
  const t = (document.body ? document.body.innerText || document.body.textContent || "" : "").slice(0, 30000);
  let h = 5381;
  for (let i = 0; i < t.length; i++) h = ((h << 5) + h + t.charCodeAt(i)) | 0;
  return location.href + "|" + (document.body ? document.body.getElementsByTagName("*").length : 0) + "|" + h;
}

// Sends one recorded request again, from the page (same cookies / origin as the app).
async function fetchFn(req) {
  try {
    const r = await fetch(req.url, { method: req.method, headers: req.headers || {}, body: req.body == null ? undefined : req.body, credentials: "include" });
    const text = await r.text();
    return { ok: r.ok, status: r.status, text: text.slice(0, 800) };
  } catch (e) {
    return { ok: false, status: 0, text: String(e && e.message || e) };
  }
}




















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
};






















//manifest.json
{
  "manifest_version": 3,
  "name": "Page Agent",
  "version": "1.0.2",
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
