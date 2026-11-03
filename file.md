//learn.js
// Learning from usage. Loaded after skills.js (shares its globals).
// Four sources, all stored only in this browser (chrome.storage.local "knowledge"):
//   run   - what worked / what failed in the agent's own tasks
//   watch - "Learn this app": the user's normal use, turned into task suggestions + facts
//   fix   - corrections from the user (typed, or shown by recording)
// The notes for the current site are added to the model's instructions on the next task.

// own element helper, so this file does not depend on the load order of skills.js
function lnEl(tag, cls, text) {
  const e = document.createElement(tag);
  if (cls) e.className = cls;
  if (text != null) e.textContent = text;
  return e;
}
let KNOW = {}; // { host: [ note ] }   note = { id, kind: flow|avoid|tip|field|page, text, req?, src, at, uses, ok? }
let LEARN_AUTO = true;
let failLog = []; // failed actions of the current agent run
let lastFlowId = null;
const NOTE_CAP = 40;
var WATCH_GAP = 90000; // idle time that separates two tasks while watching

async function knowLoad() {
  try {
    const o = await chrome.storage.local.get(["knowledge", "learnAuto"]);
    KNOW = o.knowledge && typeof o.knowledge === "object" ? o.knowledge : {};
    LEARN_AUTO = o.learnAuto !== false;
  } catch (e) { KNOW = {}; }
  return KNOW;
}
async function knowSave() { try { await chrome.storage.local.set({ knowledge: KNOW }); } catch (e) {} }
const nid = () => "n" + Date.now().toString(36) + Math.random().toString(36).slice(2, 5);
const normNote = (t) => String(t || "").toLowerCase().replace(/[^a-z0-9À-￿]+/g, " ").trim();

function addNote(host, note) {
  if (!host || !note || !note.text) return null;
  const list = (KNOW[host] = KNOW[host] || []);
  note.text = String(note.text).replace(/\s+/g, " ").trim().slice(0, 420);
  const key = normNote(note.text);
  const same = list.find((n) => normNote(n.text) === key);
  if (same) { same.at = Date.now(); same.uses = (same.uses || 0) + 1; return same; }
  Object.assign(note, { id: nid(), at: Date.now(), uses: 0 });
  list.push(note);
  if (list.length > NOTE_CAP) {
    // forget the least useful, oldest note (confirmed / corrected ones last)
    const rank = (n) => (n.src === "fix" ? 3 : n.ok ? 2 : 1) * 1e13 + (n.uses || 0) * 1e11 + n.at;
    list.sort((a, b) => rank(b) - rank(a));
    list.length = NOTE_CAP;
  }
  return note;
}

// ---------- using what was learned ----------
const wordsOf = (s) => new Set(normNote(s).split(" ").filter((w) => w.length > 2));
function overlap(a, b) {
  const A = wordsOf(a), B = wordsOf(b);
  if (!A.size || !B.size) return 0;
  let n = 0;
  A.forEach((w) => { if (B.has(w)) n++; });
  return n / Math.min(A.size, B.size);
}
function knowledgePrompt() {
  if (!agentHost || !LEARN_AUTO) return "";
  const list = KNOW[agentHost] || [];
  if (!list.length) return "";
  const req = lastRequestText || "";
  const flows = list.filter((n) => n.kind === "flow").map((n) => ({ n, s: overlap(req, n.req || n.text) + (n.ok ? 0.1 : 0) })).filter((x) => x.s >= 0.4).sort((a, b) => b.s - a.s).slice(0, 2).map((x) => x.n);
  const facts = list.filter((n) => n.kind !== "flow").sort((a, b) => (b.src === "fix") - (a.src === "fix") || (b.uses || 0) - (a.uses || 0) || b.at - a.at).slice(0, 8);
  const pick = [...flows, ...facts];
  if (!pick.length) return "";
  let out = "";
  for (const n of pick) {
    const line = "- " + (n.kind === "avoid" ? "AVOID: " : n.kind === "flow" ? "Worked before: " : "") + n.text + "\n";
    if ((out + line).length > 1500) break;
    out += line;
  }
  return `\n\nLearned about this site (${agentHost}) from earlier use. These are hints, not guarantees: verify against the current page, and follow the user's request first.\n${out}`;
}

// ---------- learning from the agent's own runs ----------
const shortLbl = (d) => String((d && (d.label || d.placeholder || d.name || d.id)) || "").slice(0, 40);
function flowText(steps) {
  const parts = [];
  for (const st of steps) {
    const l = shortLbl(st.d);
    if (st.t === "click") parts.push(`click “${l}”`);
    else if (st.t === "type") parts.push(st.param ? `type {${l}}` : `type “${String(st.value).slice(0, 20)}” in “${l}”`);
    else if (st.t === "select") parts.push(st.param ? `choose {${l}}` : `choose “${String(st.value).slice(0, 20)}” in “${l}”`);
    else if (st.t === "key") parts.push(`press ${st.key}`);
  }
  return parts.join(" → ").slice(0, 380);
}
function learnAfterRun(host, url) {
  lastFlowId = null;
  if (RUNNER || !LEARN_AUTO || !host || !outcome) return;
  let changed = false;
  if (outcome.status === "done" && !skillRunUsed && actionLog.length >= 2) {
    const d = draftFromRun(host, url);
    if (d.steps.length >= 2) {
      const n = addNote(host, { kind: "flow", req: (lastRequestText || "").slice(0, 120), text: `For “${(lastRequestText || "").slice(0, 100)}”: ${flowText(d.steps)}`, src: "run" });
      if (n) { lastFlowId = n.id; changed = true; }
    }
  }
  // a failure that was later fixed by a different action: remember what did not work and what did
  let lessons = 0;
  for (const f of failLog) {
    if (lessons >= 2) break;
    const later = actionLog.find((a) => a.at > f.at && a.name === f.name);
    if (!later) continue;
    const fl = shortLbl(f.el && f.el.d), ll = shortLbl(later.el && later.el.d);
    if (!fl && !ll) continue;
    addNote(host, { kind: "avoid", text: `${f.name} on ${fl ? "“" + fl + "”" : "that element"} failed (${String(f.text).replace(/\s+/g, " ").slice(0, 80)})${ll && ll !== fl ? `; “${ll}” worked instead` : "; a retry worked"}.`, src: "run" });
    lessons++; changed = true;
  }
  if (changed) knowSave().then(renderKnow);
  showFeedback(host);
}

// ---------- feedback: "Was that right?" / teach me ----------
function showFeedback(host) {
  if (RUNNER || !outcome) return;
  document.querySelectorAll(".kfb").forEach((n) => n.remove());
  const div = addMessage("note", "");
  div.classList.add("kfb");
  const flowId = lastFlowId, req = lastRequestText;
  div.append(lnEl("span", "", i18n("kn.fb") + " "));
  const yes = lnEl("button", "yes", i18n("kn.yes"));
  const no = lnEl("button", "no", i18n("kn.no"));
  yes.onclick = () => {
    const n = (KNOW[host] || []).find((x) => x.id === flowId);
    if (n) { n.ok = true; knowSave().then(renderKnow); }
    div.textContent = i18n("kn.thanks");
  };
  no.onclick = () => {
    // a wrong run must not be remembered as a good flow
    if (flowId && KNOW[host]) { KNOW[host] = KNOW[host].filter((x) => x.id !== flowId); knowSave().then(renderKnow); }
    div.textContent = "";
    div.append(lnEl("div", "", i18n("kn.teachHow")));
    const ta = lnEl("input");
    ta.placeholder = i18n("kn.tellPh");
    const tell = lnEl("button", "yes", i18n("kn.save"));
    const show = lnEl("button", "no", i18n("kn.showMe"));
    tell.onclick = async () => {
      const t = ta.value.trim();
      if (!t) { ta.focus(); return; }
      addNote(host, { kind: "avoid", text: `When asked “${String(req).slice(0, 80)}”: ${t}`, src: "fix" });
      await knowSave(); renderKnow();
      div.textContent = i18n("kn.fixSaved");
    };
    show.onclick = () => { div.remove(); startRecording({ teach: { req, host } }); };
    const row = lnEl("div", "tbtns");
    row.append(tell, show);
    div.append(ta, row);
    ta.focus();
  };
  div.append(yes, no);
}

// ---------- "Learn this app": watch the user ----------
const sig = (steps) => steps.filter((s) => s.t !== "key" || s.key === "Enter").map((s) => s.t + ":" + (shortLbl(s.d) || (s.d && s.d.tag) || "")).slice(0, 60);
function lcsRatio(a, b) {
  if (!a.length || !b.length) return 0;
  const dp = Array.from({ length: a.length + 1 }, () => new Array(b.length + 1).fill(0));
  for (let i = 1; i <= a.length; i++) for (let j = 1; j <= b.length; j++) dp[i][j] = a[i - 1] === b[j - 1] ? dp[i - 1][j - 1] + 1 : Math.max(dp[i - 1][j], dp[i][j - 1]);
  return dp[a.length][b.length] / Math.max(a.length, b.length);
}
function segmentSteps(steps) {
  const segs = [];
  let cur = [];
  for (const s of steps) {
    const gap = cur.length ? s.at - cur[cur.length - 1].at : 0;
    if (cur.length && gap > WATCH_GAP) { segs.push(cur); cur = []; }
    cur.push(s);
  }
  if (cur.length) segs.push(cur);
  return segs;
}
function analyzeWatch(data, rec) {
  const steps = (data.steps || []).filter((s) => s && s.d);
  const tasks = [];
  for (const seg of segmentSteps(steps)) {
    if (seg.length < 3) continue;
    const sg = sig(seg);
    const twin = tasks.find((t) => lcsRatio(t.sig, sg) >= 0.8);
    if (twin) { twin.count++; twin.segs.push(seg); } else tasks.push({ sig: sg, count: 1, segs: [seg] });
  }
  tasks.sort((a, b) => b.count - a.count || b.segs[0].length - a.segs[0].length);
  // facts
  const notes = [];
  const seen = new Set();
  for (const s of steps) {
    if (s.choices && s.choices.length > 1 && (s.t === "select" || s.pick || s.kind === "radio")) {
      const f = (s.t === "select" ? (s.d.fl || shortLbl(s.d)) : s.kind === "radio" ? (s.fl || "choice") : s.fl) || "";
      if (!f || seen.has("c" + f)) continue;
      seen.add("c" + f);
      notes.push({ kind: "field", text: `“${f}” is a ${s.multi ? "multi-select" : "choice"} field. Options: ${s.choices.slice(0, 12).join(" | ")}${s.choices.length > 12 ? " | …" : ""}.`, src: "watch" });
    }
    if (s.ro && s.t === "type") {
      const f = s.d.fl || shortLbl(s.d);
      if (f && !seen.has("d" + f)) { seen.add("d" + f); notes.push({ kind: "tip", text: `“${f}” is a read-only date field: open its calendar icon and pick the day (typing does nothing).`, src: "watch" }); }
    }
    if (s.icon && s.near && !seen.has("i" + s.near.id + s.near.name)) {
      seen.add("i" + s.near.id + s.near.name);
    }
  }
  const pages = {};
  for (const s of steps) { try { const p = new URL(s.url).pathname; pages[p] = (pages[p] || 0) + 1; } catch (e) {} }
  const top = Object.entries(pages).sort((a, b) => b[1] - a[1]).slice(0, 5).map((x) => x[0]);
  if (top.length > 1) notes.push({ kind: "page", text: `Pages you use most here: ${top.join(", ")}`, src: "watch" });
  tasks.slice(0, 6).forEach((t) => notes.push({ kind: "flow", req: "", text: `Common task (${t.count}×): ${flowText(t.segs[0].map((s) => ({ ...s, param: undefined }))).slice(0, 300)}`, src: "watch" }));
  return { total: steps.length, tasks, notes };
}
function finishWatch(data, rec) {
  const res = analyzeWatch(data, rec);
  if (!res.total) { addMessage("note", i18n("kn.watchEmpty")); return; }
  const card = addMessage("skillcard", "");
  card.append(lnEl("div", "title", i18n("kn.watchTitle", { n: res.total })));
  res.tasks.slice(0, 6).forEach((t, i) => {
    const row = lnEl("div", "srow");
    row.append(lnEl("span", "stext", i18n("kn.taskN", { i: i + 1, steps: t.segs[0].length }) + (t.count > 1 ? " · " + i18n("kn.repeated", { n: t.count }) : "") + " — " + flowText(t.segs[0]).slice(0, 110)));
    const b = lnEl("button", "yes", i18n("kn.saveSkill"));
    b.onclick = () => {
      const seg = t.segs[0];
      const draft = draftFromRecording({ steps: seg, reqs: [] }, { host: rec.host, startUrl: seg[0].url || rec.startUrl });
      openSkillEditor(draft);
    };
    row.append(b);
    card.append(row);
  });
  const boxes = [];
  if (res.notes.length) {
    card.append(lnEl("div", "small", i18n("kn.factsTitle")));
    res.notes.forEach((n) => {
      const lab = lnEl("label", "mrow");
      const cb = lnEl("input");
      cb.type = "checkbox";
      cb.checked = n.kind !== "flow";
      lab.append(cb, document.createTextNode(" " + n.text));
      card.append(lab);
      boxes.push([cb, n]);
    });
  }
  const btns = lnEl("div", "btns");
  const save = lnEl("button", "yes", i18n("kn.saveNotes"));
  const close = lnEl("button", "no", i18n("sk.cancel"));
  save.onclick = async () => {
    let k = 0;
    for (const [cb, n] of boxes) if (cb.checked) { addNote(rec.host, { ...n }); k++; }
    await knowSave(); renderKnow();
    card.remove();
    addMessage("note", i18n("kn.notesSaved", { n: k }));
  };
  close.onclick = () => card.remove();
  btns.append(save, close);
  card.append(btns);
}

// ---------- the "Learned" panel: view, delete, export, import ----------
function renderKnow() {
  const box = $("knowList");
  if (!box || !$("knowTitle")) return; // an older sidepanel.html without the Learned panel
  box.textContent = "";
  const hosts = Object.keys(KNOW).filter((h) => KNOW[h] && KNOW[h].length).sort((a, b) => (b === agentHost) - (a === agentHost) || a.localeCompare(b));
  const total = hosts.reduce((s, h) => s + KNOW[h].length, 0);
  $("knowTitle").textContent = i18n("kn.title", { n: total });
  const topRow = lnEl("div", "skform");
  const auto = lnEl("label", "mrow");
  const cb = lnEl("input");
  cb.type = "checkbox";
  cb.checked = LEARN_AUTO;
  cb.onchange = async () => { LEARN_AUTO = cb.checked; try { await chrome.storage.local.set({ learnAuto: LEARN_AUTO }); } catch (e) {} };
  auto.append(cb, document.createTextNode(" " + i18n("kn.auto")));
  topRow.append(auto);
  box.append(topRow);
  if (!hosts.length) box.append(lnEl("div", "small", i18n("kn.empty")));
  for (const h of hosts) {
    const row = lnEl("div", "skill");
    const hd = lnEl("div", "sktop");
    hd.append(lnEl("b", "", h));
    const clr = lnEl("a", "sdel", i18n("kn.clearSite"));
    clr.href = "#";
    clr.onclick = async (e) => { e.preventDefault(); if (!confirm(i18n("kn.clearConfirm", { host: h }))) return; delete KNOW[h]; await knowSave(); renderKnow(); };
    hd.append(clr);
    row.append(hd);
    for (const n of KNOW[h]) {
      const r = lnEl("div", "srow");
      r.append(lnEl("span", "stext", (n.kind === "avoid" ? "⚠ " : n.kind === "flow" ? "↻ " : "• ") + n.text + (n.src ? "  (" + n.src + (n.ok ? " ✓" : "") + ")" : "")));
      const del = lnEl("a", "sdel", "✕");
      del.href = "#";
      del.onclick = async (e) => { e.preventDefault(); KNOW[h] = KNOW[h].filter((x) => x.id !== n.id); if (!KNOW[h].length) delete KNOW[h]; await knowSave(); renderKnow(); };
      r.append(del);
      row.append(r);
    }
    box.append(row);
  }
  const bt = lnEl("div", "tbtns");
  const ex = lnEl("button", "no", i18n("kn.export"));
  const im = lnEl("button", "no", i18n("kn.import"));
  ex.onclick = async () => {
    await skillsLoad();
    const blob = new Blob([JSON.stringify({ pageAgent: 1, exported: new Date().toISOString(), knowledge: KNOW, skills: SKILLS }, null, 1)], { type: "application/json" });
    const a = document.createElement("a");
    a.href = URL.createObjectURL(blob);
    a.download = "page-agent-learned.json";
    a.click();
    setTimeout(() => URL.revokeObjectURL(a.href), 2000);
  };
  im.onclick = () => { const f = $("knowFile"); if (f) f.click(); };
  bt.append(ex, im);
  box.append(bt, lnEl("div", "small", i18n("kn.exportWarn")));
}
async function importKnow(file) {
  try {
    const j = JSON.parse(await file.text());
    if (!j || j.pageAgent !== 1) throw new Error("not a Page Agent export");
    await knowLoad(); await skillsLoad();
    let notes = 0, skills = 0;
    for (const [h, list] of Object.entries(j.knowledge || {})) for (const n of Array.isArray(list) ? list : []) if (n && n.text && addNote(h, { kind: n.kind || "tip", text: n.text, req: n.req, src: n.src || "import", ok: n.ok })) notes++;
    for (const sk of Array.isArray(j.skills) ? j.skills : []) {
      if (!sk || !sk.name || !Array.isArray(sk.steps) || !sk.host) continue;
      SKILLS = SKILLS.filter((s) => !(s.host === sk.host && s.name.toLowerCase() === String(sk.name).toLowerCase()));
      SKILLS.push({ ...sk, id: sk.id && !SKILLS.some((s) => s.id === sk.id) ? sk.id : "sk_" + Date.now().toString(36) + skills });
      skills++;
    }
    await knowSave(); await skillsSave(); renderKnow();
    addMessage("note", i18n("kn.imported", { notes, skills }));
  } catch (e) { addMessage("err", i18n("kn.importFail", { msg: e.message })); }
}

function learnInit() {
  knowLoad().then(renderKnow);
  if ($("knowBtn") && $("knowBox")) {
    $("knowBtn").onclick = (e) => { e.preventDefault(); const b = $("knowBox"); b.hidden = false; b.open = !b.open; renderKnow(); };
    if ($("watchBtn")) $("watchBtn").onclick = (e) => { e.preventDefault(); try { if (recording) stopRecording(); else Promise.resolve(startRecording({ watch: true })).catch((x) => addMessage("err", String(x && x.message || x))); } catch (x) { addMessage("err", String(x && x.message || x)); } };
    if ($("knowFile")) $("knowFile").onchange = (e) => { const f = e.target.files && e.target.files[0]; if (f) importKnow(f); e.target.value = ""; };
  }
  i18nOnChange(renderKnow);
}
learnInit();

// ---------- self-check: files from different versions are the usual reason for "nothing happens" ----------
function selfCheck() {
  const missing = [];
  for (const id of ["recBtn", "recBar", "recText", "recStop", "skillsBox", "skillsList", "watchBtn", "knowBtn", "knowBox", "knowList", "knowTitle", "knowFile"]) if (!document.getElementById(id)) missing.push("#" + id + " (sidepanel.html)");
  const fns = { startRecording: "skills.js", stopRecording: "skills.js", mkEl: "skills.js", openSkillEditor: "skills.js", draftFromRecording: "skills.js", executeSkill: "skills.js", askSkillInputs: "skills.js", recorderFn: "inpage.js", replayStepFn: "inpage.js", fieldsFn: "inpage.js", snapshotFn: "inpage.js", callLLM: "llm.js", loadLite: "sidepanel.js", addMessage: "sidepanel.js" };
  for (const [f, file] of Object.entries(fns)) { let ok = false; try { ok = typeof (0, eval)("typeof " + f + ' === "function" ? ' + f + " : undefined") === "function"; } catch (e) {} if (!ok) missing.push(f + " (" + file + ")"); }
  for (const k of ["kn.watch", "sk.askTitle", "set.lite", "sk.report"]) { try { if (i18n(k) === k) missing.push("text " + k + " (lang-en.js / lang-ko.js)"); } catch (e) {} }
  if (missing.length) {
    try { addMessage("err", "Some extension files are missing or from an older version: " + missing.join(", ") + ". Replace the whole extension folder with the latest zip, then reload the extension."); } catch (e) {}
  }
  return missing;
}
// show script errors in the panel instead of failing silently
window.addEventListener("error", (e) => {
  try { addMessage("err", "Panel error: " + (e.message || "?") + " (" + String(e.filename || "").split("/").pop() + ":" + (e.lineno || "?") + ")"); } catch (x) {}
});
window.addEventListener("unhandledrejection", (e) => {
  try { addMessage("err", "Panel error: " + String((e.reason && e.reason.message) || e.reason).slice(0, 200)); } catch (x) {}
});
setTimeout(selfCheck, 400);



















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
    "\n\nSaved skills. When one matches the request, call run_skill with its name and the parameter values you know from the user's request (omit unknown ones: the user is shown a form to confirm or fill in the inputs before it runs). It replays a recorded, proven path and is faster and more reliable than doing the steps yourself. A skill for another site runs automatically in that site's tab (it is found or opened for you), so you may use it from any tab. If a skill fails, continue manually from where it stopped.\n" +
    list.map((s) => `- ${s.name}${s.host !== agentHost ? ` [site ${s.host}]` : ""}${s.description ? ": " + s.description : ""}. Parameters: ${s.params.length ? s.params.map((p) => `${p.name}${p.label ? ` (${p.label})` : ""}${p.default ? ` default "${String(p.default).slice(0, 40)}"` : ""}${p.bool ? " yes/no" : ""}${p.multi ? " (several values: separate with ;)" : ""}${p.choices && p.choices.length ? ` options: ${p.choices.slice(0, 12).join(" | ")}` : ""}`).join(", ") : "none"}`).join("\n")
  );
}
const RUN_SKILL_TOOL = {
  name: "run_skill",
  description: "Run a saved skill (a recorded sequence of steps for this site). Give the skill name and a params object with a value for each parameter.",
  input_schema: { type: "object", properties: { name: { type: "string" }, params: { type: "object" } }, required: ["name"] },
};
const skillTools = () => (SKILLS.length ? [RUN_SKILL_TOOL] : []);

// ---------- multi-value / yes-no inputs ----------
const YES_RE = /^\s*(y|yes|true|1|on|checked|tick|ticked|select|selected)\b/i;
function splitMulti(v, choices) {
  if (Array.isArray(v)) return v.map((x) => String(x).trim()).filter(Boolean);
  const str = String(v == null ? "" : v);
  if (/[;\n|]/.test(str)) return str.split(/\s*(?:;|\n|\|)\s*/).filter(Boolean);
  if (choices && choices.includes(str.trim())) return [str.trim()];
  return str.split(/\s*,\s*/).filter(Boolean);
}

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
function recBarShow(n, watch) {
  const bar = $("recBar");
  bar.hidden = false;
  $("recText").textContent = i18n(watch ? "kn.watching" : "sk.recording", { n: n || 0 });
}
async function startRecording(opts) {
  if (running || recording) return;
  opts = opts || {};
  const tab = await getActiveTab();
  if (!tab || !/^https?:/.test(tab.url || "")) { addMessage("err", i18n("err.unreadable")); return; }
  const hint = document.querySelector(".hint");
  if (hint) hint.remove();
  agentTabId = tab.id;
  recording = { tabId: tab.id, host: hostOf(tab.url), startUrl: tab.url, watch: !!opts.watch, teach: opts.teach || null };
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
      recBarShow(r.result && r.result.steps ? r.result.steps.length : 0, recording && recording.watch);
    } catch (e) {}
  }, 1500);
  recBarShow(0, recording.watch);
  addMessage("note", i18n(recording.watch ? "kn.watchStart" : recording.teach ? "kn.teachStarted" : "sk.recStarted"));
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
  if (rec.watch) { finishWatch(data, rec); return; }
  const draft = draftFromRecording(data, rec);
  if (rec.teach) { draft.name = String(rec.teach.req || "").slice(0, 60); draft.description = String(rec.teach.req || "").slice(0, 140); }
  T("skill-recorded", { steps: draft.steps.length, params: draft.params.length, api: !!draft.api });
  openSkillEditor(draft);
}

// Turn the recorded events into a skill draft: parameters from typed values, plus an optional direct-request fast path.
function draftFromRecording(data, rec) {
  const steps = data.steps.map((s) => ({ t: s.t, d: s.d, value: s.value, key: s.key, sensitive: !!s.sensitive, effect: s.effect, at: s.at, url: s.url, icon: s.icon, near: s.near, ro: s.ro, pick: s.pick, choices: s.choices, fl: s.fl, kind: s.kind, checked: s.checked, box: s.box, multi: s.multi }));
  const params = [];
  const used = new Set();
  for (const st of steps) {
    if ((st.t === "type" || st.t === "select") && !st.sensitive && st.value) {
      const pl = (st.d && st.d.fl) || lbl(st.d);
      let n = slug(pl), k = 2;
      while (used.has(n)) n = slug(pl) + "_" + k++;
      used.add(n);
      st.param = n;
      params.push({ name: n, label: pl, default: st.value, choices: st.choices, multi: !!st.multi });
    }
  }
  // Date pickers: clicks that lead up to a value appearing in a read-only field (open the picker, pick a day) are best-effort at replay;
  // the field itself is then set from the parameter, so a picker that cannot be driven does not break the skill.
  steps.forEach((st, j) => {
    if (!(st.t === "type" && st.ro && st.param)) return;
    const nums = String(st.value).match(/\d+/g) || [];
    for (let k = j - 1, prevAt = st.at; k >= 0 && steps[k].t === "click" && prevAt - steps[k].at < 20000; prevAt = steps[k].at, k--) {
      steps[k].opt = true;
      const lab = String(steps[k].d && steps[k].d.label || "").trim();
      if (/^\d{1,2}$/.test(lab)) {
        const idx = nums.findIndex((n) => Number(n) === Number(lab));
        if (idx >= 0) { steps[k].dayOf = st.param; steps[k].dayIdx = idx; steps[k].d.daySel = true; }
      }
    }
  });
  const draft = { id: "sk_" + Date.now().toString(36), name: "", description: "", host: rec.host, startUrl: rec.startUrl, params, steps, api: null, useApi: true, created: Date.now(), uses: 0, ok: 0, fail: 0 };
  // Choices made in custom dropdowns / lists / radio groups and ticked checkboxes: each one is an input too.
  const uniq = (label, fallback) => { let n = slug(label) || fallback, k = 2; const b = n; while (used.has(n)) n = b + "_" + k++; used.add(n); return n; };
  steps.forEach((st, i) => {
    if (st.t !== "click" || st.opt || st.dayOf || st.param || st.grp) return;
    if (st.kind === "checkbox") {
      const n = uniq(lbl(st.d), "option");
      st.param = n;
      params.push({ name: n, label: String(lbl(st.d)).slice(0, 40), default: st.checked ? "yes" : "no", bool: true });
      return;
    }
    if (st.kind === "radio") {
      const n = uniq(st.fl || "choice", "choice");
      st.param = n;
      params.push({ name: n, label: String(st.fl || "Choice").slice(0, 40), default: st.d.label, choices: st.choices });
      return;
    }
    const pv0 = steps[i - 1];
    if (!st.pick && pv0 && (pv0.pick || pv0.grp) && pv0.t === "click") { st.loose = true; return; } // closes the list: its label shows the current choice
    if (!st.pick) return;
    const pv = steps[i - 1];
    if (pv && pv.pick && pv.box && pv.box === st.box && (pv.param || pv.grp)) {
      // another pick from the same open list, with no new opener: a multi-select
      const ln = pv.param || pv.grp, P = params.find((x) => x.name === ln);
      st.grp = ln;
      if (P) { P.multi = true; P.default = P.default + "; " + st.d.label; }
      return;
    }
    if (dynLink(draft, i).label || dynLink(draft, i).ctx) return; // a typeahead suggestion: already follows what was typed
    const opener = i > 0 && steps[i - 1].t === "click" ? steps[i - 1] : null;
    const label = (opener && (opener.fl || lbl(opener.d))) || st.fl || "Option";
    const n = uniq(label, "option");
    st.param = n;
    params.push({ name: n, label: String(label).slice(0, 40), default: st.d.label, choices: st.choices });
    if (opener) opener.loose = true; // the opener often shows the current value, which will differ next time
  });
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
    if (st.t === "click" && st.kind === "checkbox") return;
    if (st.t === "click" && st.param) { st.d.dyn = st.param; return; }
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
  if (r.kind === "checkbox" && r.param && values[r.param] != null) r.want = YES_RE.test(String(values[r.param]));
  if (r.dayOf && values[r.dayOf] != null) {
    const nums = String(values[r.dayOf]).match(/\d+/g) || [];
    if (nums[r.dayIdx] != null) { d.label = String(Number(nums[r.dayIdx])); d.id = d.testid = d.css = d.name = ""; d.nth = 0; }
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
      if ((st.t === "type" || st.t === "select" || (st.t === "click" && (st.pick || st.param || st.kind))) && !st.sensitive) {
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
            if (!draft.params.some((p) => p.name === st.param)) draft.params.push({ name: st.param, label: lbl(st.d), default: st.kind === "checkbox" ? (st.checked ? "yes" : "no") : st.t === "click" ? st.d.label : st.value, choices: st.choices, bool: st.kind === "checkbox", multi: !!st.multi });
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
      del.onclick = (e) => { e.preventDefault(); const s = draft.steps[i]; draft.steps.splice(i, 1); if (s.param) { draft.params = draft.params.filter((p) => p.name !== s.param); draft.steps = draft.steps.filter((x) => x.grp !== s.param); } draft.api = null; render(); };
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
    draft.params.forEach((p) => { p.multi = draft.steps.some((s) => s.grp === p.name) || draft.steps.some((s) => s.param === p.name && s.t === "select" && s.multi); });
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

// One input control for a skill parameter: text (with suggestions), yes/no, or a tick-list for multi-selects.
function paramInput(sk, p, value, need, form) {
  const cur = value == null ? "" : String(value);
  if (p.bool) {
    const sel = mkEl("select");
    for (const [v, k] of [["yes", "sk.yes"], ["no", "sk.no"]]) { const o = mkEl("option", "", i18n(k)); o.value = v; sel.append(o); }
    sel.value = YES_RE.test(cur) ? "yes" : "no";
    return { node: sel, get: () => sel.value, focus: () => sel.focus(), mark: () => {} };
  }
  if (p.multi && p.choices && p.choices.length) {
    const wrap = mkEl("div", "multi");
    const have = splitMulti(cur, p.choices).map((x) => x.toLowerCase());
    const boxes = p.choices.map((c) => {
      const lab = mkEl("label", "mrow");
      const cb = mkEl("input");
      cb.type = "checkbox";
      cb.value = c;
      cb.checked = have.includes(c.toLowerCase());
      lab.append(cb, document.createTextNode(" " + c));
      wrap.append(lab);
      return cb;
    });
    // values the model gave that are not in the recorded list are kept as extra text
    const extra = splitMulti(cur, p.choices).filter((x) => !p.choices.some((c) => c.toLowerCase() === x.toLowerCase()));
    const ex = mkEl("input");
    ex.placeholder = i18n("sk.otherValues");
    ex.value = extra.join("; ");
    wrap.append(ex);
    return { node: wrap, get: () => boxes.filter((b) => b.checked).map((b) => b.value).concat(splitMulti(ex.value)).join("; "), focus: () => boxes[0].focus(), mark: (on) => wrap.classList.toggle("need", on) };
  }
  const inp = mkEl("input");
  inp.value = cur;
  if (p.multi) inp.placeholder = i18n("sk.multiHint");
  if (p.choices && p.choices.length) {
    const dl = mkEl("datalist");
    dl.id = "dl_" + sk.id + "_" + p.name + "_" + Math.random().toString(36).slice(2, 6);
    p.choices.forEach((c) => { const o = mkEl("option"); o.value = c; dl.append(o); });
    inp.setAttribute("list", dl.id);
    form.append(dl);
  }
  if (need) inp.classList.add("need");
  return { node: inp, get: () => inp.value.trim(), focus: () => inp.focus(), mark: (on) => inp.classList.toggle("need", on), el: inp };
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
      const pi = paramInput(sk, p, p.default || "", false, form);
      inputs[p.name] = pi;
      form.append(lab, pi.node);
    }
    const btns = mkEl("div", "tbtns");
    const run = mkEl("button", "yes", i18n("sk.runNow"));
    run.onclick = () => {
      const values = {};
      for (const p of sk.params) values[p.name] = inputs[p.name].get();
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
    addReportButton();
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
    if (Array.isArray(v)) v = v.join("; ");
    if (v == null || v === "") v = p.default;
    if (v == null || v === "") missing.push(p.name);
    values[p.name] = v == null ? "" : String(v);
  }
  // Ask the user to confirm / fill the inputs before anything runs (skipped for unattended runs, and with auto-approve when nothing is missing).
  if (sk.params.length && !RUNNER && (missing.length || !$("autoApprove").checked)) {
    const got = await askSkillInputs(sk, values, missing);
    if (!got) return { ok: false, denied: true, error: true, text: "The user cancelled running this skill. Do not retry it. Ask how to proceed." };
    Object.assign(values, got);
  } else if (missing.length) return { error: true, text: `Skill "${sk.name}" needs a value for: ${missing.join(", ")}. Ask the user, then call run_skill again.` };
  tab = await resolveSkillTab(sk, tab);
  agentTabId = tab.id;
  const res = await executeSkill(sk, values, tab);
  if (!RUNNER) addReportButton();
  let snap = "";
  try { snap = await takeSnapshot(); } catch (e) {}
  return { error: !res.ok, text: res.text + (snap ? "\n\n" + snap : "") };
}

// Shows the skill's inputs (pre-filled from the request / recording) and waits for the user to confirm or change them.
function askSkillInputs(sk, values, missing) {
  return new Promise((resolve) => {
    const div = addMessage("approval", "");
    div.append(mkEl("div", "title", i18n("sk.askTitle", { name: sk.name })));
    div.append(mkEl("div", "skhint", i18n(missing.length ? "sk.askMissing" : "sk.askCheck")));
    const form = mkEl("div", "skform");
    const inputs = {};
    for (const p of sk.params) {
      const lab = mkEl("label", "", (p.label || p.name) + (missing.includes(p.name) ? " *" : "") + (p.multi ? " " + i18n("sk.multiTag") : ""));
      const pi = paramInput(sk, p, values[p.name], missing.includes(p.name), form);
      inputs[p.name] = pi;
      form.append(lab, pi.node);
    }
    const btns = mkEl("div", "btns");
    const yes = mkEl("button", "yes", i18n("sk.runNow"));
    const no = mkEl("button", "no", i18n("btn.deny"));
    const done = (res) => { yes.disabled = no.disabled = true; form.querySelectorAll("input,select").forEach((x) => { x.disabled = true; }); approvalResolver = null; resolve(res); };
    yes.onclick = () => {
      const out = {};
      for (const p of sk.params) out[p.name] = inputs[p.name].get();
      const empty = sk.params.filter((p) => !out[p.name]);
      if (empty.length) { empty.forEach((p) => inputs[p.name].mark(true)); inputs[empty[0].name].focus(); return; }
      done(out);
    };
    no.onclick = () => done(null);
    approvalResolver = (ok) => done(ok ? null : null);
    btns.append(yes, no);
    div.append(form, btns);
    const first = sk.params.find((p) => missing.includes(p.name)) || sk.params[0];
    try { inputs[first.name].focus(); } catch (e) {}
  });
}

async function stepApproval(sk, st, values, tab, i) {
  if ($("autoApprove").checked) { logLine(i18n("log.auto", { action: stepText(st, st.param ? values[st.param] : null) })); return true; }
  const filled = sk.params.length ? sk.params.map((p) => `  ${p.label || p.name}: ${values[p.name]}`).join("\n") : "  " + i18n("d.none");
  return await requestApproval(stepText(st, st.param ? values[st.param] : null), i18n("why.final"), `${i18n("d.page")}: ${tab.title}\n${tab.url}\n\n${i18n("act.skill", { name: sk.name })}\n${i18n("d.fields")}:\n${filled}`);
}

// ---------- debug report ----------
let RUN = null;
const compactD = (d) => { if (!d) return d; const o = {}; for (const k of ["tag", "role", "type", "id", "name", "label", "placeholder", "ctx", "css", "dyn", "dynValue", "daySel", "nth"]) if (d[k]) o[k] = d[k]; return o; };
function reportSkill(sk) {
  return { name: sk.name, host: sk.host, startUrl: sk.startUrl, useApi: sk.useApi, hasApi: !!sk.api, params: sk.params, steps: sk.steps.map((st, i) => ({ n: i + 1, t: st.t, d: compactD(st.d), value: st.sensitive ? "(secret)" : st.value, key: st.key, param: st.param, opt: st.opt, grp: st.grp, kind: st.kind, multi: st.multi, pick: st.pick, icon: st.icon, ro: st.ro, loose: st.loose, dayOf: st.dayOf, effect: st.effect, url: st.url })) };
}
async function buildReport() {
  let page = null;
  try { const [r] = await chrome.scripting.executeScript({ target: { tabId: agentTabId }, func: fieldsFn }); page = r.result; } catch (e) { page = { error: e.message }; }
  let ver = "";
  try { ver = chrome.runtime.getManifest().version; } catch (e) {}
  const rep = { tool: "Page Agent " + ver, note: "Contains the values you entered and the field values visible on the page (passwords hidden). Review before sharing.", run: RUN, skill: RUN && RUN.sk ? reportSkill(RUN.sk) : null, pageNow: page };
  if (rep.run) { rep.run = { ...rep.run }; delete rep.run.sk; }
  return JSON.stringify(rep, null, 1);
}
function addReportButton() {
  document.querySelectorAll(".reportbtn").forEach((n) => n.remove());
  const div = addMessage("note", "");
  div.classList.add("reportbtn");
  const b = mkEl("button", "no", i18n("sk.report"));
  b.onclick = async () => {
    const text = await buildReport();
    let ok = false;
    try { await navigator.clipboard.writeText(text); ok = true; } catch (e) {}
    if (!ok) { const ta = mkEl("textarea"); ta.value = text; ta.style.width = "100%"; ta.rows = 8; div.append(ta); ta.select(); try { ok = document.execCommand("copy"); } catch (e) {} }
    b.textContent = ok ? i18n("sk.reportCopied") : i18n("sk.reportManual");
  };
  div.append(b);
}

async function executeSkill(sk, values, tab) {
  const t0 = Date.now();
  RUN = { sk, skill: sk.name, started: new Date().toISOString(), site: sk.host, values, mode: "ui", steps: [], outcome: "" };
  T("skill-run", { name: sk.name, steps: sk.steps.length, api: !!(sk.api && sk.useApi), values });
  let result;
  try { result = await executeSkillInner(sk, values, tab); } catch (e) { result = { ok: false, text: `Skill "${sk.name}" failed: ${e.message}` }; }
  await skillsLoad();
  const cur = SKILLS.find((s) => s.id === sk.id);
  if (cur) { cur.uses = (cur.uses || 0) + 1; if (result.ok) cur.ok = (cur.ok || 0) + 1; else if (!result.denied) cur.fail = (cur.fail || 0) + 1; await skillsSave(); }
  T("skill-done", { ok: result.ok, ms: Date.now() - t0, text: String(result.text).slice(0, 200) });
  if (RUN) { RUN.mode = result.mode || RUN.mode; RUN.ok = !!result.ok; RUN.ms = Date.now() - t0; RUN.outcome = String(result.text).slice(0, 400); }
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
  let noEffect = null;
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
    if (st.grp) continue; // an extra pick of a multi-select: handled together with its first pick
    const val = st.param ? values[st.param] : st.value;
    const P0 = st.param && sk.params.find((p) => p.name === st.param);
    const multiVals = P0 && P0.multi && st.t === "click" ? splitMulti(values[st.param], P0.choices) : null;
    if (multiVals && !multiVals.length) { logLine("↷ " + stepText(st, "")); continue; }
    const rs = resolveStep(st, values);
    const txt = stepText(rs, val);
    if (st.t === "type" && st.sensitive) return { ok: false, text: `Skill "${sk.name}" stopped at step ${i + 1}: "${lbl(st.d)}" is a sensitive field. Ask the user to fill it in, then continue manually. Done so far: ${done.join("; ") || "(nothing)"}.` };
    if (stepIsFinal(st) && !(await stepApproval(sk, st, values, tab, i))) return { ok: false, denied: true, text: "The user denied this action. Do not retry it. Ask how to proceed." };
    logLine(`▶ ${i + 1}/${sk.steps.length} ${txt}`);
    if (st.opt) {
      // best-effort step (opening / driving a date picker): a failure here is fine, the date field is set from the parameter afterwards
      let r0 = { ok: false };
      try { [{ result: r0 }] = await chrome.scripting.executeScript({ target: { tabId: agentTabId }, func: replayStepFn, args: [rs, null, 1800] }); } catch (e) {}
      if (!r0 || !r0.ok) logLine("↷ " + i18n("sk.optSkipped", { n: i + 1 }));
      await settle(agentTabId, true);
      done.push(txt);
      continue;
    }
    let sigBefore = "";
    if (st.t === "click" && st.effect && !st.opt && !st.icon) { try { [{ result: sigBefore }] = await chrome.scripting.executeScript({ target: { tabId: agentTabId }, func: pageSigFn }); } catch (e) {} }
    const tryStep = async (timeout, rstep) => {
      const t1 = Date.now();
      let out;
      try {
        const [r] = await chrome.scripting.executeScript({ target: { tabId: agentTabId }, func: replayStepFn, args: [rstep || resolveStep(st, values), val == null ? null : String(val), timeout] });
        out = r.result || { ok: false, msg: "no result" };
      } catch (e) { out = { ok: false, msg: e.message }; }
      if (RUN) RUN.steps.push({ step: i + 1, of: sk.steps.length, action: txt, target: compactD(rstep ? rstep.d : rs.d), ok: out.ok, msg: out.msg, ms: Date.now() - t1 });
      return out;
    };
    let res;
    if (multiVals) {
      res = { ok: true, msg: "" };
      for (const mv of multiVals) {
        res = await tryStep(6000, resolveStep(st, { ...values, [st.param]: mv }));
        if (!res.ok) break;
        await sleep(150);
      }
    } else res = await tryStep(i === i0 ? 2500 : 6000);
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
      const suspect = noEffect ? ` Note: step ${noEffect.i + 1} (${noEffect.txt}) showed no visible change and may be the cause.` : "";
      return { ok: false, text: `Skill "${sk.name}" stopped at step ${i + 1}/${sk.steps.length} (${txt}): ${res.msg}${suspect} Done so far: ${done.join("; ") || "(nothing)"}. Continue this task manually from the current page.` };
    }
    if (st.t === "click" && st.effect) await sleep(350); // let a form postback / navigation actually begin before checking whether the tab is ready
    await settle(agentTabId, st.t !== "click" && st.t !== "key");
    if (newTabId) { agentTabId = newTabId; newTabId = null; await settle(agentTabId); }
    if (st.t === "click" && st.effect && sigBefore) {
      // The recording showed this click changing the page. Give a slow result (server postback, partial update) time to arrive.
      // If nothing visibly changes we do NOT click again (that could repeat an action) and we do not fail yet: the next step
      // proves whether the click mattered; if that step cannot be found, the message names this click as the suspect.
      let changed = false;
      for (let k = 0; k < 6 && !changed; k++) {
        let sig = "";
        try { [{ result: sig }] = await chrome.scripting.executeScript({ target: { tabId: agentTabId }, func: pageSigFn }); } catch (e) { changed = true; break; }
        changed = sig !== sigBefore;
        if (!changed) await sleep(250);
      }
      if (!changed) {
        noEffect = { i, txt };
        T("skill-no-effect", { i, text: txt }); // quiet: it is only reported if a later step fails or at the end
      } else if (noEffect && noEffect.i < i) noEffect = null;
    }
    done.push(txt);
    if (st.t === "type" && sk.params.length) typedLog.push({ label: lbl(st.d), value: String(val) });
  }
  return { ok: true, mode: "ui", text: `Skill "${sk.name}" completed all ${sk.steps.length} steps.${noEffect ? ` Note: step ${noEffect.i + 1} (${noEffect.txt}) showed no visible change; check the result.` : ""}` };
}

// ---------- wiring ----------
function skillsInit() {
  skillsLoad().then(renderSkills);
  if ($("recBtn")) {
    $("recBtn").onclick = (e) => { e.preventDefault(); try { if (recording) stopRecording(); else Promise.resolve(startRecording()).catch((x) => addMessage("err", String(x && x.message || x))); } catch (x) { addMessage("err", String(x && x.message || x)); } };
    $("recStop").onclick = () => stopRecording();
    $("skillsBtn").onclick = (e) => { e.preventDefault(); const b = $("skillsBox"); b.hidden = false; b.open = !b.open; };
  }
  try {
    chrome.storage.onChanged.addListener((ch, area) => { if (area === "local" && ch.skills) { SKILLS = Array.isArray(ch.skills.newValue) ? ch.skills.newValue : []; renderSkills(); } });
  } catch (e) {}
  i18nOnChange(renderSkills);
}
skillsInit();
