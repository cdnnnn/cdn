//theme.js
// Light / dark theme. The choice is saved in this browser; until you pick one, the app follows the system setting.
import { useEffect, useState } from "react";

const KEY = "slm-theme";
const media = () => window.matchMedia("(prefers-color-scheme: dark)");
export const savedTheme = () => { try { return localStorage.getItem(KEY); } catch { return null; } };
export const currentTheme = () => savedTheme() || (media().matches ? "dark" : "light");

// Switching to dark sweeps in diagonally from the bottom-left corner to the top right;
// switching to light sweeps from the top-right corner to the bottom left.
// Uses the View Transitions API where the browser has it (Chrome, Edge); elsewhere the colors simply fade.
const SWEEP = {
  dark: ["polygon(0% 100%, 0% 100%, 0% 100%)", "polygon(0% 100%, 0% -100%, 200% 100%)"],
  light: ["polygon(100% 0%, 100% 0%, 100% 0%)", "polygon(100% 0%, 100% 200%, -100% 0%)"],
};

function setTheme(t) {
  document.documentElement.dataset.theme = t;
  window.dispatchEvent(new CustomEvent("slm:theme", { detail: t }));
}

export function applyTheme(t, { animate = true } = {}) {
  const el = document.documentElement;
  const reduce = window.matchMedia("(prefers-reduced-motion: reduce)").matches;
  if (animate && !reduce && document.startViewTransition) {
    el.classList.add("theme-sweep");
    const vt = document.startViewTransition(() => setTheme(t));
    vt.ready.then(() => {
      el.animate({ clipPath: SWEEP[t] }, { duration: 750, easing: "cubic-bezier(.65, 0, .35, 1)", pseudoElement: "::view-transition-new(root)" });
    }).catch(() => {});
    vt.finished.finally(() => el.classList.remove("theme-sweep"));
    return;
  }
  el.classList.add("theme-switching");                 // fallback: fade colors for a moment
  setTheme(t);
  setTimeout(() => el.classList.remove("theme-switching"), 450);
}

export function useTheme() {
  const [theme, setTheme] = useState(currentTheme);
  useEffect(() => {
    const onTheme = (e) => setTheme(e.detail);
    const onSystem = () => { if (!savedTheme()) applyTheme(media().matches ? "dark" : "light", { animate: false }); };
    window.addEventListener("slm:theme", onTheme);
    media().addEventListener("change", onSystem);
    return () => { window.removeEventListener("slm:theme", onTheme); media().removeEventListener("change", onSystem); };
  }, []);
  const toggle = () => {
    const t = theme === "dark" ? "light" : "dark";
    try { localStorage.setItem(KEY, t); } catch { /* private mode */ }
    applyTheme(t);
  };
  return [theme, toggle];
}
























//theme-dark.css
/* Dark theme and the light/dark switch.
   The dark theme swaps the color tokens; the coral gradient stays the same, so both modes share one identity.
   Selected with <html data-theme="dark">, set by src/theme.js (and before first paint by index.html). */

:root[data-theme="dark"] {
  color-scheme: dark;

  /* neutrals: deep ink-blue greys, a little warm */
  --bg: #0c0e13;
  --surface: #14171f;
  --surface-2: #191d26;
  --surface-3: #212531;
  --line: #2a2f3b;
  --line-2: #222631;
  --line-strong: #373d4b;
  --ink: #eef0f4;
  --ink-2: #c9ced8;
  --muted: #959cab;
  --faint: #6c7486;

  /* coral accent, lifted for contrast on dark */
  --accent: #FB6B70;
  --accent-rgb: 251, 107, 112;
  --accent-600: #FF8E92;
  --accent-400: #F57A7E;
  --accent-200: #5f2a33;
  --accent-100: #3f1f27;
  --accent-50: #2a171d;
  --accent-ink: #ffd9dc;
  --tint-0: #1a171b;
  --tint-1: #231a1d;
  --tint-2: #231922;

  --violet: #F472B6;
  --violet-50: #2a1724;
  --violet-100: #3e1f34;
  --violet-ink: #fbcfe8;

  --mix-2: #E8A090;
  --mix-3: #64748B;
  --mix-4: #475569;
  --split-2: #94A3B8;
  --split-3: #64748B;

  /* status colors */
  --green: #34d399;
  --green-600: #4ade9f;
  --green-50: #0f261e;
  --green-200: #1d4a39;
  --green-ink: #a7f3d0;
  --amber: #fbbf24;
  --amber-50: #2a2111;
  --amber-200: #5b4416;
  --amber-ink: #fcd34d;
  --amber-ink-2: #fde68a;
  --orange: #fb923c;
  --orange-50: #2b1c10;
  --red: #f87171;
  --red-50: #2c1518;
  --red-200: #5c2429;
  --red-ink: #fca5a5;
  --blue: #60a5fa;
  --blue-50: #13203a;

  /* depth: darker shadows, a faint light edge instead of a white one */
  --paper-rgb: 20, 23, 31;
  --hairline: rgba(255, 255, 255, .07);
  --edge: inset 0 1px 0 rgba(255, 255, 255, .045);
  --dot: rgba(255, 255, 255, .045);
  --shadow-xs: 0 1px 2px rgba(0, 0, 0, .3);
  --shadow: 0 1px 2px rgba(0, 0, 0, .3), 0 1px 3px rgba(0, 0, 0, .25);
  --shadow-md: 0 6px 16px -4px rgba(0, 0, 0, .5);
  --shadow-lg: 0 28px 56px -12px rgba(0, 0, 0, .65);
  --lift-1: 0 1px 2px rgba(0, 0, 0, .35);
  --lift-2: 0 1px 2px rgba(0, 0, 0, .3), 0 6px 14px -4px rgba(0, 0, 0, .35), 0 16px 32px -16px rgba(0, 0, 0, .5);
  --lift-3: 0 2px 4px rgba(0, 0, 0, .3), 0 14px 28px -8px rgba(0, 0, 0, .45), 0 30px 60px -24px rgba(0, 0, 0, .6);
  --scroll: #2f3442;
  --scroll-hover: #414857;
  --switch-off: #343a48;
  --switch-off-hover: #3e4555;

  /* the "dark" button becomes a light one, so it still stands out */
  --dark-btn-a: #ffffff;
  --dark-btn-b: #e8eaef;
  --dark-btn-ink: #111827;
}

/* ---- dark-only touches */
[data-theme="dark"] main.main { background:
  radial-gradient(900px 420px at 100% -10%, rgba(249, 115, 22, .07), transparent 60%),
  radial-gradient(700px 380px at 30% -20%, rgba(236, 72, 153, .05), transparent 60%),
  radial-gradient(var(--dot) 1px, transparent 1.2px) 0 0 / 22px 22px,
  var(--bg); }
[data-theme="dark"] .sidebar { background:
  linear-gradient(180deg, rgba(20, 23, 31, .97), rgba(16, 19, 26, .97)),
  radial-gradient(var(--dot) 1px, transparent 1.2px) 0 0 / 18px 18px;
  box-shadow: 4px 0 24px -18px rgba(0, 0, 0, .8); }
[data-theme="dark"] .ph { background: rgba(16, 19, 26, .72); box-shadow: 0 8px 24px -18px rgba(0, 0, 0, .9); }
[data-theme="dark"] .gen-ft { background: rgba(20, 23, 31, .85); }
[data-theme="dark"] .gen-hd { background: linear-gradient(180deg, rgba(255, 255, 255, .02), transparent); }
[data-theme="dark"] .stp-n { box-shadow: 0 0 0 4px var(--surface); }
[data-theme="dark"] .seg { background: var(--bg); box-shadow: inset 0 1px 2px rgba(0, 0, 0, .4); }
[data-theme="dark"] .seg button.on { background: var(--surface-3); box-shadow: 0 1px 2px rgba(0, 0, 0, .4), 0 0 0 1px var(--hairline); }
[data-theme="dark"] input:not([type=checkbox]):not([type=file]), [data-theme="dark"] select, [data-theme="dark"] textarea {
  background-color: var(--bg); box-shadow: inset 0 1px 2px rgba(0, 0, 0, .35); }
[data-theme="dark"] .switch span { background: #f3f4f6; }
[data-theme="dark"] .btn.primary { box-shadow: inset 0 1px 0 rgba(255, 255, 255, .2), 0 8px 20px -8px rgba(236, 72, 153, .55); }
[data-theme="dark"] .splitbar span { color: #fff; }
[data-theme="dark"] .pv-card { box-shadow: 0 30px 60px -20px rgba(0, 0, 0, .7), 0 0 0 6px rgba(255, 255, 255, .03); }
[data-theme="dark"] .dz-sheet { background: var(--surface-3); }
[data-theme="dark"] .mono, [data-theme="dark"] .run-title .mono { background: var(--surface-3); }

/* ---- switching: colors fade instead of jumping */
.theme-switching, .theme-switching *, .theme-switching *::before, .theme-switching *::after {
  transition: background-color .35s ease, border-color .35s ease, color .35s ease, fill .35s ease, box-shadow .35s ease !important; }

/* ---- diagonal sweep (View Transitions): the new theme is revealed over the old one by src/theme.js */
::view-transition-old(root), ::view-transition-new(root) { animation: none; mix-blend-mode: normal; }
::view-transition-new(root) { z-index: 2; }
::view-transition-old(root) { z-index: 1; }
.theme-sweep *:not(.theme-toggle, .theme-toggle *) { transition: none !important; }

/* ---- the sun / moon switch */
.theme-toggle { position: relative; width: 56px; height: 30px; flex: none; border-radius: 99px; border: 1px solid var(--hairline); cursor: pointer; padding: 0;
  background: linear-gradient(180deg, var(--surface-3), var(--surface-2)); box-shadow: inset 0 1px 2px rgba(16, 24, 40, .08); transition: background .3s, border-color .3s; }
.theme-toggle:hover { border-color: var(--line-strong); }
.tt-knob { position: absolute; top: 2px; left: 2px; width: 24px; height: 24px; border-radius: 50%; display: grid; place-items: center; color: #fff;
  background: var(--grad); box-shadow: 0 2px 6px -1px rgba(var(--accent-rgb), .55);
  transition: transform .4s cubic-bezier(.34, 1.4, .64, 1), background .3s; }
.tt-knob svg { position: absolute; transition: opacity .25s, transform .4s cubic-bezier(.34, 1.4, .64, 1); }
.tt-moon { opacity: 0; transform: rotate(-60deg) scale(.6); }
.theme-toggle.is-dark .tt-knob { transform: translateX(26px); background: linear-gradient(135deg, #6366F1, #A855F7); box-shadow: 0 2px 8px -1px rgba(129, 140, 248, .6); }
.theme-toggle.is-dark .tt-sun { opacity: 0; transform: rotate(60deg) scale(.6); }
.theme-toggle.is-dark .tt-moon { opacity: 1; transform: none; }
.ph-sep { width: 1px; height: 24px; background: var(--line); margin: 0 2px; }
@media (max-width: 760px) { .lp-nav-in .theme-toggle { margin-left: auto; } .lp-nav-in .btn { margin-left: 0 !important; } }























//GeneratePage.jsx
// Generate Dataset page: a guided, step-by-step setup with a live summary, and a progress view once a job runs.
// Same API as the original UI (/api/datagen); the form <-> JobRequest logic is in ../model.js.
import { useCallback, useEffect, useLayoutEffect, useRef, useState } from "react";
import { API, api, post, upload } from "../api.js";
import { buildRequest, defaultForm, endpointsForTest, formFromRequest, visionUsed } from "../model.js";
import { Icon, useToast } from "../ui.jsx";
import { emit, useBus } from "../bus.js";
import { AdvancedStep, DocumentsStep, MixStep, ModelsStep, QuestionsStep } from "./steps.jsx";
import Summary from "./Summary.jsx";
import RunProgress from "./RunProgress.jsx";
import "./generate.css";

export const STEPS = [
  { key: "documents", title: "Documents", icon: "doc", lead: "Upload the files the model should learn from, or point to a folder on the server." },
  { key: "models", title: "Models", icon: "server", lead: "The teacher model writes and judges the Q&A pairs; the vision model reads images. Both must already be running." },
  { key: "questions", title: "Questions", icon: "chat", lead: "Choose which kinds of questions to ask, how long the answers are, and how the dataset is split." },
  { key: "mix", title: "Training mix", icon: "mix", lead: "Add extra rows next to the document Q&A, so the model stays helpful and well-behaved." },
  { key: "advanced", title: "Advanced", icon: "gear", lead: "Everything here has sensible defaults. Change only what you need." },
];

const jobFromUrl = () => new URL(location.href).searchParams.get("job");

export default function GeneratePage({ openResults }) {
  const toast = useToast();
  const [defaults, setDefaults] = useState(null);
  const [form, setForm] = useState(() => defaultForm(null));
  const [files, setFiles] = useState([]);
  const [step, setStep] = useState(0);
  const [ep, setEp] = useState({ state: "idle" });
  const [models, setModels] = useState({ teacher: [], vision: [] });
  const [jobId, setJobId] = useState(jobFromUrl);
  // remember the previous step, so the step content slides in from the right direction
  const prevStep = useRef(0);
  const dir = step >= prevStep.current ? "fwd" : "back";
  useEffect(() => { prevStep.current = step; }, [step]);
  // the soft card behind the active step is placed by measuring that step, so it always sits under the right one
  const stepperRef = useRef(null);
  const [ind, setInd] = useState(null);
  useLayoutEffect(() => {
    const place = () => {
      const el = stepperRef.current?.querySelector(".stp.on");
      const next = el ? { left: el.offsetLeft, top: el.offsetTop, width: el.offsetWidth, height: el.offsetHeight } : null;
      setInd((cur) => (cur && next && cur.left === next.left && cur.top === next.top && cur.width === next.width && cur.height === next.height) || cur === next ? cur : next);
    };
    place();
    window.addEventListener("resize", place);
    return () => window.removeEventListener("resize", place);
  });
  const [job, setJob] = useState(null);
  const [progress, setProgress] = useState({});
  const [starting, setStarting] = useState(false);
  const [startErr, setStartErr] = useState("");
  const formRef = useRef(form);
  formRef.current = form;
  const set = (k, v) => setForm((f) => ({ ...f, [k]: v }));
  const setIn = (group, k, v) => setForm((f) => ({ ...f, [group]: { ...f[group], [k]: v } }));

  // ---- start-up: defaults, and follow a job that is already running
  useEffect(() => {
    (async () => {
      let d = null;
      try { d = await api("/defaults"); } catch (e) { toast(`API not reachable: ${e.message}`, "error"); }
      setDefaults(d || { input_roots: [] });
      setForm(defaultForm(d));
      if (!jobFromUrl()) {
        try {
          const list = await api("/jobs");
          const active = list.find((j) => j.status === "queued" || j.status === "running");
          if (active) setJobId(active.id);
        } catch { /* the toast above already says the API is down */ }
      }
    })();
  }, []); // eslint-disable-line

  // keep ?job= in the address bar so a refresh comes back to the same job
  useEffect(() => {
    const u = new URL(location.href);
    jobId ? u.searchParams.set("job", jobId) : u.searchParams.delete("job");
    history.replaceState(null, "", u);
  }, [jobId]);

  // ---- the selected job and its live progress (server-sent events)
  const refreshJob = useCallback(async (id) => {
    if (!id) return;
    try { setJob(await api(`/jobs/${id}`)); } catch (e) { if (e.status === 404) setJobId(null); }
  }, []);
  useEffect(() => { setJob(null); setProgress({}); if (jobId) refreshJob(jobId); }, [jobId, refreshJob]);
  useEffect(() => {
    if (!jobId) return undefined;
    const es = new EventSource(`${API}/jobs/${encodeURIComponent(jobId)}/events`);
    let t;
    es.onmessage = (m) => {
      const ev = JSON.parse(m.data);
      if (ev.type === "end") { es.close(); refreshJob(jobId); return; }
      if (ev.type === "step_progress") setProgress((p) => ({ ...p, [ev.step]: { done: ev.done, total: ev.total } }));
      else { clearTimeout(t); t = setTimeout(() => refreshJob(jobId), 250); }
    };
    return () => { clearTimeout(t); es.close(); };
  }, [jobId, refreshJob]);

  // ---- endpoints: test now, and again whenever an endpoint setting changes
  const testSeq = useRef(0);
  const testEndpoints = useCallback(async (f = formRef.current) => {
    if (!f.teacherUrl.trim()) { setEp({ state: "done", message: "Enter the teacher URL.", bad: true }); return; }
    const seq = ++testSeq.current;
    setEp((e) => ({ ...e, state: "testing" }));
    try {
      const res = await post("/endpoints/test", { endpoints: endpointsForTest(f) });
      if (seq !== testSeq.current) return;
      const t = res.find((x) => x.role === "teacher"), v = res.find((x) => x.role === "vision");
      setModels({ teacher: t?.models || [], vision: v?.models || [] });
      const pick = (list, re) => list.find((m) => re.test(m)) || list[0];
      setForm((cur) => {
        const next = { ...cur };
        if (t?.ok && t.models.length && !t.models.includes(cur.teacherModel)) next.teacherModel = pick(t.models, /^(?!.*(vl|vision)).*/i);
        if (v?.ok && v.models.length && !v.models.includes(cur.visionModel)) next.visionModel = pick(v.models, /vl|vision/i);
        return next;
      });
      const why = (x, role) => (x.ok ? (x.models.length ? "" : `${role}: no models served`) : `${role}: ${x.error || "not reachable"}`);
      const problems = [why(t, "Teacher"), v ? why(v, "Vision") : visionUsed(f) ? "Vision: enter the URL, or turn vision analysis off" : ""].filter(Boolean);
      setEp({ state: "done", teacher: t, vision: v, message: problems.join(" · "), bad: problems.length > 0 });
    } catch (e) {
      if (seq === testSeq.current) setEp({ state: "done", message: `Test failed: ${e.message}`, bad: true });
    }
  }, []);
  const epKey = [form.teacherUrl, form.visionUrl, form.apiKey, visionUsed(form)].join("|");
  const firstTest = useRef(true);
  useEffect(() => {
    if (!defaults) return undefined;
    const t = setTimeout(() => testEndpoints(), firstTest.current ? 0 : 700);
    firstTest.current = false;
    return () => clearTimeout(t);
  }, [epKey, defaults, testEndpoints]);

  // ---- what is missing before a job can start
  const folders = form.folders.split(/\n/).map((s) => s.trim()).filter(Boolean);
  const issues = [];
  if (!files.length && !folders.length) issues.push({ step: 0, text: "Add documents: upload files or enter a server folder." });
  try { buildRequest(form, ["check"]); } catch (e) {
    const s = /endpoint|vision/i.test(e.message) ? 1 : /question types|split/i.test(e.message) ? 2 : /decline|paraphrase|general/i.test(e.message) ? 3 : 4;
    issues.push({ step: s, text: e.message });
  }

  // ---- actions
  const startJob = async (preview) => {
    setStartErr("");
    if (issues.length) { setStep(issues[0].step); setStartErr(issues[0].text); return; }
    setStarting(true);
    try {
      const list = [...folders];
      if (files.length) list.unshift((await upload(files)).folder);
      const res = await post(`/jobs${preview ? "?preview=3" : ""}`, buildRequest(form, list));
      setFiles([]);
      toast(`Job ${res.job_id} started${preview ? " (preview on 3 chunks)" : ""}`);
      setJobId(res.job_id);
    } catch (e) {
      setStartErr(e.message);
    } finally { setStarting(false); }
  };
  const cancelJob = async () => { await post(`/jobs/${jobId}/cancel`); toast("Cancel requested: the job stops after the current item"); };
  const retryJob = async () => { await post(`/jobs/${jobId}/retry`); toast("Job re-queued: finished steps are skipped"); setProgress({}); refreshJob(jobId); };
  const newJob = () => { setJobId(null); setStep(0); setStartErr(""); };
  const reuseSettings = () => { if (job) { setForm(formFromRequest(job.request, defaults)); toast(`Settings of ${job.job_id} loaded`); } newJob(); };
  const resetForm = () => { setForm(defaultForm(defaults)); toast("Settings reset to the defaults"); };

  // ---- tell the sidebar and header what is going on, and listen to them
  const vUsed = visionUsed(form);
  useEffect(() => { emit("ep", { ...ep, teacherModel: form.teacherModel, visionModel: vUsed ? form.visionModel : "" }); }, [ep, form.teacherModel, form.visionModel, vUsed]);
  useEffect(() => { emit("job", { id: jobId, name: job?.name || job?.request?.name, status: job?.status, step, ready: issues.length === 0 }); });
  useEffect(() => { emit("jobs-changed"); }, [jobId, job?.status]);
  useBus("open-job", useCallback((id) => { setStartErr(""); setJobId(id); }, []));
  useBus("new-job", useCallback(() => { setJobId(null); setStep(0); setStartErr(""); }, []));
  useBus("retest", useCallback(() => testEndpoints(), [testEndpoints]));

  if (jobId) {
    return <RunProgress job={job} jobId={jobId} progress={progress} onCancel={cancelJob} onRetry={retryJob}
      onNew={newJob} onReuse={reuseSettings} openResults={openResults} />;
  }

  const stepProps = { form, set, setIn, setForm, files, setFiles, defaults, ep, models, testEndpoints, onReset: resetForm };
  const cur = STEPS[step];
  const status = stepStatus(form, files, folders, ep);
  return (
    <div className="gen">
      <nav className="stepper" aria-label="Setup steps" ref={stepperRef} style={{ "--p": step / (STEPS.length - 1) }}>
        <span className="stp-progress" aria-hidden="true"><i /></span>
        {ind && <span className="stp-ind" aria-hidden="true" style={{ transform: `translate(${ind.left}px, ${ind.top}px)`, width: ind.width, height: ind.height }} />}
        {STEPS.map((s, i) => {
          const st = status[s.key];
          return (
            <button type="button" key={s.key} className={`stp${i === step ? " on" : ""}${st.ok ? " ok" : ""}${st.warn ? " attn" : ""}`} onClick={() => setStep(i)}>
              <span className="stp-n">{st.ok && i !== step ? <Icon name="check" size={14} stroke={2.6} className="pop" /> : <span className="pop">{i + 1}</span>}</span>
              <span className="stp-t"><b>{s.title}</b><small>{st.note}</small></span>
            </button>
          );
        })}
      </nav>

      <div className="gen-grid">
        <section className="card gen-main">
          <header className={`gen-hd anim-${dir}`} key={`hd${step}`}>
            <span className="gen-hd-ico"><Icon name={cur.icon} size={20} /></span>
            <div className="gen-hd-t">
              <div className="gen-hd-meta">
                <span className="step-n">Step {step + 1} of {STEPS.length}</span>
                <span className="step-dots" aria-hidden="true">{STEPS.map((s2, i) => <i key={s2.key} className={i < step ? "done" : i === step ? "on" : ""} />)}</span>
              </div>
              <h2>{cur.title}</h2><p>{cur.lead}</p>
            </div>
          </header>
          <div className={`gen-bd anim-${dir}`} key={`bd${step}`}>
            {cur.key === "documents" && <DocumentsStep {...stepProps} />}
            {cur.key === "models" && <ModelsStep {...stepProps} />}
            {cur.key === "questions" && <QuestionsStep {...stepProps} />}
            {cur.key === "mix" && <MixStep {...stepProps} />}
            {cur.key === "advanced" && <AdvancedStep {...stepProps} />}
          </div>
          <footer className="gen-ft">
            <button type="button" className="btn ghost" disabled={step === 0} onClick={() => setStep(step - 1)}><Icon name="back" size={15} />Back</button>
            <div className="grow" />
            {step < STEPS.length - 1
              ? <button type="button" className="btn dark" onClick={() => setStep(step + 1)}>Next: {STEPS[step + 1].title}<Icon name="arrow" size={15} /></button>
              : <button type="button" className="btn primary" disabled={starting} onClick={() => startJob(false)}><Icon name="play" size={13} />{starting ? "Starting…" : "Start job"}</button>}
          </footer>
        </section>

        <Summary form={form} files={files} folders={folders} ep={ep} issues={issues} starting={starting} startErr={startErr}
          onStart={startJob} goStep={setStep} />
      </div>
    </div>
  );
}

function stepStatus(form, files, folders, ep) {
  const n = files.length + folders.length;
  const split = form.split.reduce((a, b) => a + (Number(b) || 0), 0);
  const extras = [form.para.enabled, form.decline.enabled, form.general.enabled].filter(Boolean).length;
  const epOk = ep.state === "done" && !ep.bad;
  return {
    documents: n ? { ok: true, note: [files.length && `${files.length} file${files.length > 1 ? "s" : ""}`, folders.length && `${folders.length} folder${folders.length > 1 ? "s" : ""}`].filter(Boolean).join(" · ") } : { note: "Add your files" },
    models: ep.state === "testing" ? { note: "Testing…" } : epOk ? { ok: true, note: "Endpoints online" } : { warn: ep.state === "done", note: ep.state === "done" ? "Check the endpoints" : "Not tested yet" },
    questions: form.types.length && split === 100 ? { ok: true, note: `${form.types.length} question type${form.types.length > 1 ? "s" : ""}` } : { warn: true, note: !form.types.length ? "Pick a question type" : "Split must total 100" },
    mix: { ok: true, note: extras ? `${extras} extra${extras > 1 ? "s" : ""} on` : "Document Q&A only" },
    advanced: { ok: false, note: "Optional" },
  };
}
