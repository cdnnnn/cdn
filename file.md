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
    if ($("watchBtn")) $("watchBtn").onclick = (e) => { e.preventDefault(); if (recording) stopRecording(); else startRecording({ watch: true }); };
    if ($("knowFile")) $("knowFile").onchange = (e) => { const f = e.target.files && e.target.files[0]; if (f) importKnow(f); e.target.value = ""; };
  }
  i18nOnChange(renderKnow);
}
learnInit();




















//manifest.json
{
  "manifest_version": 3,
  "name": "Page Agent",
  "version": "1.1.4",
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
