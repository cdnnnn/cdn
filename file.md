//Sidebar.jsx
// Sidebar: the logo and the two pages.
import { Icon } from "./ui.jsx";
import Logo from "./Logo.jsx";

export default function Sidebar({ route, pages }) {
  return (
    <aside className="sidebar">
      <div className="sb-top">
        <a className="sb-brand" href="#/" title="Back to the home page"><Logo size={34} withName={false} /></a>
      </div>
      <nav className="sb-nav" style={{ "--i": Math.max(0, Object.keys(pages).indexOf(route)) }}>
        <span className="sb-ind" aria-hidden="true" />
        {Object.entries(pages).map(([k, p]) => (
          <a key={k} className={route === k ? "on" : ""} href={`#/${k}`}>
            <span className="sb-nav-ico"><Icon name={p.icon} size={17} /></span>{p.nav}
          </a>
        ))}
      </nav>
    </aside>
  );
}












//Shell.css
/* App shell: sidebar and page header. Calm and neutral; coral is used for small accents only. */

/* ---- sidebar: logo and two tabs; a white card slides behind the active tab */
div.app { grid-template-columns: 248px minmax(0, 1fr); }
.sidebar { gap: 26px; padding: 20px 14px 14px; background: linear-gradient(180deg, var(--surface) 0%, var(--surface-2) 100%); border-right: 1px solid var(--line); }
.sb-top { display: flex; align-items: center; padding: 2px 8px 0; }
.sb-brand { display: inline-flex; align-items: center; border-radius: 10px; padding: 4px; margin: -4px; transition: background .15s; }
.sb-brand:hover { background: var(--surface-3); }

.sb-nav { position: relative; display: grid; gap: 4px; }
.sb-nav a { position: relative; z-index: 1; display: flex; align-items: center; gap: 12px; height: 46px; padding: 0 10px; border-radius: 12px; color: var(--ink-2);
  text-decoration: none; font-weight: 500; font-size: 15px; transition: color .2s, background .2s; }
.sb-nav a:not(.on):hover { background: rgba(17, 24, 39, .04); color: var(--ink); }
.sb-nav a.on { color: var(--ink); font-weight: 600; }
.sb-nav-ico { width: 30px; height: 30px; border-radius: 9px; display: grid; place-items: center; color: var(--muted); transition: color .2s, background .2s, transform .2s; }
.sb-nav a:hover .sb-nav-ico { transform: translateX(1px); }
.sb-nav a.on .sb-nav-ico { background: var(--accent-50); color: var(--accent-600); }
.sb-ind { position: absolute; left: 0; right: 0; top: 0; height: 46px; border-radius: 12px; background: var(--surface);
  box-shadow: 0 1px 2px rgba(16, 24, 40, .06), 0 0 0 1px var(--line), 0 6px 14px -8px rgba(16, 24, 40, .12);
  transform: translateY(calc(var(--i, 0) * 50px)); transition: transform .38s cubic-bezier(.2, .8, .2, 1); }
.sb-ind::before { content: ""; position: absolute; left: -14px; top: 12px; bottom: 12px; width: 3px; border-radius: 0 3px 3px 0; background: var(--grad); }

/* ---- page header */
.ph { display: flex; align-items: center; gap: 20px; padding: 20px 32px; background: var(--surface); border-bottom: 1px solid var(--line);
  position: relative; }
.ph::after { content: ""; position: absolute; left: 32px; right: 32px; bottom: -1px; height: 1px;
  background: linear-gradient(90deg, rgba(var(--accent-rgb), .35), transparent 40%); }
.ph-main { display: flex; align-items: center; gap: 16px; min-width: 0; flex: 1; }
.ph-ico { width: 48px; height: 48px; border-radius: 14px; display: grid; place-items: center; flex: none; color: var(--accent-600);
  background: linear-gradient(180deg, var(--surface), var(--surface-2)); border: 1px solid var(--line); box-shadow: var(--shadow-xs); }
.ph-t { min-width: 0; display: grid; gap: 2px; }
.ph-crumbs { display: flex; align-items: center; gap: 6px; font-size: 12.5px; color: var(--faint); }
.ph-crumbs a { color: var(--muted); text-decoration: none; }
.ph-crumbs a:hover { color: var(--ink); }
.ph-crumbs .ico { color: var(--line-strong); }
.ph-crumbs > span { color: var(--ink-2); font-weight: 500; }
.ph h1 { margin: 0; font-size: 23px; font-weight: 700; letter-spacing: -.3px; color: var(--ink); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.ph p { margin: 0; font-size: 14px; color: var(--muted); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.ph-side { display: flex; align-items: center; gap: 10px; flex: none; }
.ph-state { display: inline-flex; align-items: center; gap: 10px; height: 36px; padding: 0 12px 0 4px; border: 1px solid var(--line); border-radius: 10px;
  background: var(--surface-2); font-size: 13px; color: var(--muted); }
.ph-state-l { font-size: 11.5px; font-weight: 650; letter-spacing: .5px; text-transform: uppercase; color: var(--ink-2); background: var(--surface);
  border: 1px solid var(--line); border-radius: 7px; padding: 3px 8px; }
.ph-state-v b { color: var(--ink); font-weight: 600; }
.ph-state-dot { width: 8px; height: 8px; border-radius: 50%; background: #f59e0b; box-shadow: 0 0 0 3px rgba(245, 158, 11, .15); }
.ph-state-dot.ok { background: var(--green); box-shadow: 0 0 0 3px rgba(16, 185, 129, .15); }

@media (max-width: 1100px) { .ph p { display: none; } .ph { padding: 16px 20px; } }
main .content { padding: 24px 32px 28px; }
.ph p b { color: var(--ink-2); font-weight: 600; }














//bus.js
// Tiny app-wide event bus, so the sidebar and header can talk to the Generate page.
// Events: "ep" (model status), "job" (the job open on the Generate page), "jobs-changed",
//         "open-job" (id), "new-job", "retest". The last value of each event is kept for late listeners.
import { useEffect } from "react";

const last = {};
export const emit = (name, detail) => { last[name] = detail; window.dispatchEvent(new CustomEvent(`slm:${name}`, { detail })); };
export const lastOf = (name) => last[name];
export function useBus(name, handler) {
  useEffect(() => {
    const h = (e) => handler(e.detail);
    window.addEventListener(`slm:${name}`, h);
    return () => window.removeEventListener(`slm:${name}`, h);
  }, [name, handler]);
}














//App.jsx
// Routing: the landing page at #/ (the first screen), and the app shell with sidebar and top bar at #/generate and #/results.
// Features are added one piece at a time; see README.md for the build order.
import { useCallback, useEffect, useState } from "react";
import { Empty, Icon } from "./ui.jsx";
import Landing from "./Landing.jsx";
import Sidebar from "./Sidebar.jsx";
import { emit, lastOf, useBus } from "./bus.js";
import GeneratePage, { STEPS } from "./generate/GeneratePage.jsx";
import "./shell.css";

const PAGES = {
  generate: { label: "Generate", nav: "Generate Dataset", icon: "spark", title: "Generate Dataset",
    sub: "Turn your documents into grounded question-and-answer pairs for fine-tuning." },
  results: { label: "Results", nav: "Results", icon: "results", title: "Results",
    sub: "Review the generated pairs, their scores, and download the dataset." },
};
// "#how" and "#features" are anchors on the landing page, so anything that is not an app page shows the landing page.
const routeFromHash = () => { const r = (location.hash || "").replace(/^#\/?/, ""); return PAGES[r] ? r : "home"; };

export default function App() {
  const [route, setRoute] = useState(routeFromHash);
  useEffect(() => {
    const h = () => setRoute(routeFromHash());
    window.addEventListener("hashchange", h);
    return () => window.removeEventListener("hashchange", h);
  }, []);
  if (route === "home") return <Landing />;
  const page = PAGES[route];

  return (
    <div className="app">
      <Sidebar route={route} pages={PAGES} />

      <main className="main">
        <PageHeader route={route} page={page} />
        <div className="content" key={route}>
          {route === "generate" ? <GeneratePage openResults={() => { location.hash = "#/results"; }} /> : <ResultsPage />}
        </div>
      </main>
    </div>
  );
}

// Page header: breadcrumb, title and one line of context on the left; the state of the current job and actions on the right.
function PageHeader({ route, page }) {
  const [job, setJob] = useState(() => lastOf("job") || null);
  useBus("job", useCallback((j) => setJob(j), []));
  const onGen = route === "generate";
  const open = onGen && job?.id;
  return (
    <header className="ph">
      <div className="ph-main">
        <span className="ph-ico"><Icon name={page.icon} size={20} /></span>
        <div className="ph-t">
          <div className="ph-crumbs">
            <a href="#/">SLM Studio</a><Icon name="chevron" size={12} />
            <a href={`#/${route}`}>{page.nav}</a>
            {onGen && <><Icon name="chevron" size={12} />{open ? <span className="num">{job.id}</span> : <span>New job</span>}</>}
          </div>
          <h1>{page.title}</h1>
          <p>{open ? <>Following <b>{job.name || job.id}</b>. You can leave this page; the job keeps running on the server.</> : page.sub}</p>
        </div>
      </div>
      <div className="ph-side">
        {onGen && !open && job && (
          <span className="ph-state">
            <span className="ph-state-l">Draft</span>
            <span className="ph-state-v">Step <b className="num">{(job.step ?? 0) + 1}</b> of {STEPS.length}</span>
            <span className={`ph-state-dot ${job.ready ? "ok" : ""}`} title={job.ready ? "Ready to start" : "Something still needs your input"} />
          </span>
        )}
        {open && <button type="button" className="btn" onClick={() => emit("new-job")}><Icon name="plus" size={14} />New job</button>}
        {onGen && !open && <a className="btn ghost" href="#/results"><Icon name="results" size={14} />Results</a>}
        {!onGen && <a className="btn" href="#/generate"><Icon name="plus" size={14} />New dataset</a>}
      </div>
    </header>
  );
}

// Pieces 14–17 fill this page.
function ResultsPage() {
  return <section className="card"><Empty icon="results" title="Results page">Nothing here yet. The summary, Q&amp;A review and charts are added piece by piece.</Empty></section>;
}
















//GeneratePage.jsx
// Generate Dataset page: a guided, step-by-step setup with a live summary, and a progress view once a job runs.
// Same API as the original UI (/api/datagen); the form <-> JobRequest logic is in ../model.js.
import { useCallback, useEffect, useRef, useState } from "react";
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
      <nav className="stepper" aria-label="Setup steps" style={{ "--i": step }}>
        <span className="stp-ind" aria-hidden="true" />
        {STEPS.map((s, i) => {
          const st = status[s.key];
          return (
            <button type="button" key={s.key} className={`stp${i === step ? " on" : ""}${st.ok ? " ok" : ""}${st.warn ? " warn" : ""}`} onClick={() => setStep(i)}>
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
            <div><span className="step-n">Step {step + 1} of {STEPS.length}</span><h2>{cur.title}</h2><p>{cur.lead}</p></div>
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













//Sidebar.jsx
// Sidebar: the logo and the two pages.
import { Icon } from "./ui.jsx";
import Logo from "./Logo.jsx";

export default function Sidebar({ route, pages }) {
  return (
    <aside className="sidebar">
      <div className="sb-top">
        <a className="sb-brand" href="#/" title="Back to the home page"><Logo size={34} withName={false} /></a>
      </div>
      <nav className="sb-nav" style={{ "--i": Math.max(0, Object.keys(pages).indexOf(route)) }}>
        <span className="sb-ind" aria-hidden="true" />
        {Object.entries(pages).map(([k, p]) => (
          <a key={k} className={route === k ? "on" : ""} href={`#/${k}`}>
            <span className="sb-nav-ico"><Icon name={p.icon} size={17} /></span>{p.nav}
          </a>
        ))}
      </nav>
    </aside>
  );
}













//Shell.css
/* App shell: sidebar and page header. Calm and neutral; coral is used for small accents only. */

/* ---- sidebar: logo and two tabs; a white card slides behind the active tab */
div.app { grid-template-columns: 248px minmax(0, 1fr); }
.sidebar { gap: 26px; padding: 20px 14px 14px; background: linear-gradient(180deg, var(--surface) 0%, var(--surface-2) 100%); border-right: 1px solid var(--line); }
.sb-top { display: flex; align-items: center; padding: 2px 8px 0; }
.sb-brand { display: inline-flex; align-items: center; border-radius: 10px; padding: 4px; margin: -4px; transition: background .15s; }
.sb-brand:hover { background: var(--surface-3); }

.sb-nav { position: relative; display: grid; gap: 4px; }
.sb-nav a { position: relative; z-index: 1; display: flex; align-items: center; gap: 12px; height: 46px; padding: 0 10px; border-radius: 12px; color: var(--ink-2);
  text-decoration: none; font-weight: 500; font-size: 15px; transition: color .2s, background .2s; }
.sb-nav a:not(.on):hover { background: rgba(17, 24, 39, .04); color: var(--ink); }
.sb-nav a.on { color: var(--ink); font-weight: 600; }
.sb-nav-ico { width: 30px; height: 30px; border-radius: 9px; display: grid; place-items: center; color: var(--muted); transition: color .2s, background .2s, transform .2s; }
.sb-nav a:hover .sb-nav-ico { transform: translateX(1px); }
.sb-nav a.on .sb-nav-ico { background: var(--accent-50); color: var(--accent-600); }
.sb-ind { position: absolute; left: 0; right: 0; top: 0; height: 46px; border-radius: 12px; background: var(--surface);
  box-shadow: 0 1px 2px rgba(16, 24, 40, .06), 0 0 0 1px var(--line), 0 6px 14px -8px rgba(16, 24, 40, .12);
  transform: translateY(calc(var(--i, 0) * 50px)); transition: transform .38s cubic-bezier(.2, .8, .2, 1); }
.sb-ind::before { content: ""; position: absolute; left: -14px; top: 12px; bottom: 12px; width: 3px; border-radius: 0 3px 3px 0; background: var(--grad); }

/* ---- page header */
.ph { display: flex; align-items: center; gap: 20px; padding: 20px 32px; background: var(--surface); border-bottom: 1px solid var(--line);
  position: relative; }
.ph::after { content: ""; position: absolute; left: 32px; right: 32px; bottom: -1px; height: 1px;
  background: linear-gradient(90deg, rgba(var(--accent-rgb), .35), transparent 40%); }
.ph-main { display: flex; align-items: center; gap: 16px; min-width: 0; flex: 1; }
.ph-ico { width: 48px; height: 48px; border-radius: 14px; display: grid; place-items: center; flex: none; color: var(--accent-600);
  background: linear-gradient(180deg, var(--surface), var(--surface-2)); border: 1px solid var(--line); box-shadow: var(--shadow-xs); }
.ph-t { min-width: 0; display: grid; gap: 2px; }
.ph-crumbs { display: flex; align-items: center; gap: 6px; font-size: 12.5px; color: var(--faint); }
.ph-crumbs a { color: var(--muted); text-decoration: none; }
.ph-crumbs a:hover { color: var(--ink); }
.ph-crumbs .ico { color: var(--line-strong); }
.ph-crumbs > span { color: var(--ink-2); font-weight: 500; }
.ph h1 { margin: 0; font-size: 23px; font-weight: 700; letter-spacing: -.3px; color: var(--ink); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.ph p { margin: 0; font-size: 14px; color: var(--muted); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.ph-side { display: flex; align-items: center; gap: 10px; flex: none; }
.ph-state { display: inline-flex; align-items: center; gap: 10px; height: 36px; padding: 0 12px 0 4px; border: 1px solid var(--line); border-radius: 10px;
  background: var(--surface-2); font-size: 13px; color: var(--muted); }
.ph-state-l { font-size: 11.5px; font-weight: 650; letter-spacing: .5px; text-transform: uppercase; color: var(--ink-2); background: var(--surface);
  border: 1px solid var(--line); border-radius: 7px; padding: 3px 8px; }
.ph-state-v b { color: var(--ink); font-weight: 600; }
.ph-state-dot { width: 8px; height: 8px; border-radius: 50%; background: #f59e0b; box-shadow: 0 0 0 3px rgba(245, 158, 11, .15); }
.ph-state-dot.ok { background: var(--green); box-shadow: 0 0 0 3px rgba(16, 185, 129, .15); }

@media (max-width: 1100px) { .ph p { display: none; } .ph { padding: 16px 20px; } }
main .content { padding: 24px 32px 28px; }
.ph p b { color: var(--ink-2); font-weight: 600; }














//App.jsx
// Routing: the landing page at #/ (the first screen), and the app shell with sidebar and top bar at #/generate and #/results.
// Features are added one piece at a time; see README.md for the build order.
import { useCallback, useEffect, useState } from "react";
import { Empty, Icon } from "./ui.jsx";
import Landing from "./Landing.jsx";
import Sidebar from "./Sidebar.jsx";
import { emit, lastOf, useBus } from "./bus.js";
import GeneratePage, { STEPS } from "./generate/GeneratePage.jsx";
import "./shell.css";

const PAGES = {
  generate: { label: "Generate", nav: "Generate Dataset", icon: "spark", title: "Generate Dataset",
    sub: "Turn your documents into grounded question-and-answer pairs for fine-tuning." },
  results: { label: "Results", nav: "Results", icon: "results", title: "Results",
    sub: "Review the generated pairs, their scores, and download the dataset." },
};
// "#how" and "#features" are anchors on the landing page, so anything that is not an app page shows the landing page.
const routeFromHash = () => { const r = (location.hash || "").replace(/^#\/?/, ""); return PAGES[r] ? r : "home"; };

export default function App() {
  const [route, setRoute] = useState(routeFromHash);
  useEffect(() => {
    const h = () => setRoute(routeFromHash());
    window.addEventListener("hashchange", h);
    return () => window.removeEventListener("hashchange", h);
  }, []);
  if (route === "home") return <Landing />;
  const page = PAGES[route];

  return (
    <div className="app">
      <Sidebar route={route} pages={PAGES} />

      <main className="main">
        <PageHeader route={route} page={page} />
        <div className="content" key={route}>
          {route === "generate" ? <GeneratePage openResults={() => { location.hash = "#/results"; }} /> : <ResultsPage />}
        </div>
      </main>
    </div>
  );
}

// Page header: breadcrumb, title and one line of context on the left; the state of the current job and actions on the right.
function PageHeader({ route, page }) {
  const [job, setJob] = useState(() => lastOf("job") || null);
  useBus("job", useCallback((j) => setJob(j), []));
  const onGen = route === "generate";
  const open = onGen && job?.id;
  return (
    <header className="ph">
      <div className="ph-main">
        <span className="ph-ico"><Icon name={page.icon} size={20} /></span>
        <div className="ph-t">
          <div className="ph-crumbs">
            <a href="#/">SLM Studio</a><Icon name="chevron" size={12} />
            <a href={`#/${route}`}>{page.nav}</a>
            {onGen && <><Icon name="chevron" size={12} />{open ? <span className="num">{job.id}</span> : <span>New job</span>}</>}
          </div>
          <h1>{page.title}</h1>
          <p>{open ? <>Following <b>{job.name || job.id}</b>. You can leave this page; the job keeps running on the server.</> : page.sub}</p>
        </div>
      </div>
      <div className="ph-side">
        {onGen && !open && job && (
          <span className="ph-state">
            <span className="ph-state-l">Draft</span>
            <span className="ph-state-v">Step <b className="num">{(job.step ?? 0) + 1}</b> of {STEPS.length}</span>
            <span className={`ph-state-dot ${job.ready ? "ok" : ""}`} title={job.ready ? "Ready to start" : "Something still needs your input"} />
          </span>
        )}
        {open && <button type="button" className="btn" onClick={() => emit("new-job")}><Icon name="plus" size={14} />New job</button>}
        {onGen && !open && <a className="btn ghost" href="#/results"><Icon name="results" size={14} />Results</a>}
        {!onGen && <a className="btn" href="#/generate"><Icon name="plus" size={14} />New dataset</a>}
      </div>
    </header>
  );
}

// Pieces 14–17 fill this page.
function ResultsPage() {
  return <section className="card"><Empty icon="results" title="Results page">Nothing here yet. The summary, Q&amp;A review and charts are added piece by piece.</Empty></section>;
}
















//ui.jsx
// Small shared components: icons, switch, segmented control, chips, field, modal, drawer, empty state, toasts.
import { createContext, useCallback, useContext, useEffect, useRef, useState } from "react";

const P = {
  doc: <><path d="M6 3h8l4 4v14H6z" /><path d="M14 3v4h4M9 12h6M9 16h6" /></>,
  upload: <><path d="M12 16V5M7.5 9.5 12 5l4.5 4.5" /><path d="M5 15v4h14v-4" /></>,
  folder: <path d="M3 7.5V18a1 1 0 0 0 1 1h16a1 1 0 0 0 1-1V9a1 1 0 0 0-1-1h-8L10 6H4a1 1 0 0 0-1 1.5z" />,
  server: <><rect x="4" y="4" width="16" height="7" rx="2" /><rect x="4" y="13" width="16" height="7" rx="2" /><path d="M8 7.5h.01M8 16.5h.01" /></>,
  chat: <><path d="M4 5h16v11H9l-5 4z" /><path d="M9.5 9a2.5 2.5 0 1 1 3.3 2.4c-.5.2-.8.5-.8 1" /></>,
  image: <><rect x="3.5" y="4.5" width="17" height="15" rx="2" /><circle cx="9" cy="10" r="1.6" /><path d="m20 16-4.5-4.5L7 19.5" /></>,
  mix: <><circle cx="12" cy="12" r="8.5" /><path d="M12 3.5V12l6 4.5" /></>,
  repeat: <><path d="M17 3l3 3-3 3" /><path d="M4 11V9a3 3 0 0 1 3-3h13M7 21l-3-3 3-3" /><path d="M20 13v2a3 3 0 0 1-3 3H4" /></>,
  shield: <><path d="M12 3 5 6v6c0 4.5 3 7.5 7 9 4-1.5 7-4.5 7-9V6z" /><path d="m9.5 9.5 5 5M14.5 9.5l-5 5" /></>,
  globe: <><circle cx="12" cy="12" r="8.5" /><path d="M3.5 12h17M12 3.5c2.5 2.6 3.5 5.5 3.5 8.5s-1 5.9-3.5 8.5c-2.5-2.6-3.5-5.5-3.5-8.5s1-5.9 3.5-8.5z" /></>,
  gear: <><circle cx="12" cy="12" r="3" /><path d="M19.4 15a1.7 1.7 0 0 0 .3 1.8l.1.1a2 2 0 1 1-2.8 2.8l-.1-.1a1.7 1.7 0 0 0-1.8-.3 1.7 1.7 0 0 0-1 1.5V21a2 2 0 1 1-4 0v-.1a1.7 1.7 0 0 0-1.1-1.5 1.7 1.7 0 0 0-1.8.3l-.1.1a2 2 0 1 1-2.8-2.8l.1-.1a1.7 1.7 0 0 0 .3-1.8 1.7 1.7 0 0 0-1.5-1H3a2 2 0 1 1 0-4h.1a1.7 1.7 0 0 0 1.5-1.1 1.7 1.7 0 0 0-.3-1.8l-.1-.1a2 2 0 1 1 2.8-2.8l.1.1a1.7 1.7 0 0 0 1.8.3H9a1.7 1.7 0 0 0 1-1.5V3a2 2 0 1 1 4 0v.1a1.7 1.7 0 0 0 1 1.5 1.7 1.7 0 0 0 1.8-.3l.1-.1a2 2 0 1 1 2.8 2.8l-.1.1a1.7 1.7 0 0 0-.3 1.8V9a1.7 1.7 0 0 0 1.5 1H21a2 2 0 1 1 0 4h-.1a1.7 1.7 0 0 0-1.5 1z" /></>,
  play: <path d="M7 4.5v15l12-7.5z" />,
  eye: <><path d="M2.5 12S6 5.5 12 5.5 21.5 12 21.5 12 18 18.5 12 18.5 2.5 12 2.5 12z" /><circle cx="12" cy="12" r="2.8" /></>,
  check: <path d="m5 12.5 4.5 4.5L19 7.5" />,
  x: <path d="M6 6l12 12M18 6 6 18" />,
  plus: <path d="M12 5v14M5 12h14" />,
  trash: <><path d="M4.5 7h15M9.5 7V4.5h5V7M6.5 7l1 13h9l1-13" /></>,
  edit: <><path d="M4 20h4L19 9l-4-4L4 16z" /><path d="m13.5 6.5 4 4" /></>,
  refresh: <><path d="M20 11a8 8 0 1 0-2.3 5.7" /><path d="M20 4v7h-7" /></>,
  download: <><path d="M12 4v11M7.5 10.5 12 15l4.5-4.5" /><path d="M5 19h14" /></>,
  database: <><ellipse cx="12" cy="6" rx="7.5" ry="3" /><path d="M4.5 6v12c0 1.7 3.4 3 7.5 3s7.5-1.3 7.5-3V6M4.5 12c0 1.7 3.4 3 7.5 3s7.5-1.3 7.5-3" /></>,
  chart: <><path d="M4 20V4M4 20h16" /><path d="M8 16v-4M12 16V8M16 16v-6" /></>,
  results: <><rect x="5" y="4.5" width="14" height="16.5" rx="2.5" /><path d="M9 4.5V3.5a1 1 0 0 1 1-1h4a1 1 0 0 1 1 1v1" /><path d="m8.8 12.3 2.2 2.2 4.2-4.4" /><path d="M9 17.8h6" /></>,
  alert: <><path d="M12 3.5 2.5 20h19z" /><path d="M12 10v4.5M12 17.5h.01" /></>,
  search: <><circle cx="11" cy="11" r="6.5" /><path d="m20 20-4.2-4.2" /></>,
  list: <path d="M8 6h12M8 12h12M8 18h12M4 6h.01M4 12h.01M4 18h.01" />,
  spark: <><path d="M12 3v4M12 17v4M3 12h4M17 12h4" /><path d="m6 6 2.5 2.5M15.5 15.5 18 18M18 6l-2.5 2.5M8.5 15.5 6 18" /></>,
  layers: <><path d="m12 3 9 5-9 5-9-5z" /><path d="m3 13 9 5 9-5" /></>,
  arrow: <path d="M5 12h14M13 6l6 6-6 6" />,
  stop: <rect x="6" y="6" width="12" height="12" rx="2" />,
  chevron: <path d="m9 6 6 6-6 6" />,
  back: <path d="M19 12H5M11 6l-6 6 6 6" />,
  clock: <><circle cx="12" cy="12" r="8.5" /><path d="M12 7.5V12l3 2" /></>,
  book: <><path d="M4 5.5A2.5 2.5 0 0 1 6.5 3H20v15H6.5A2.5 2.5 0 0 0 4 20.5z" /><path d="M4 20.5A2.5 2.5 0 0 0 6.5 23H20v-5" /></>,
};

export function Icon({ name, size = 16, stroke = 1.8, className = "" }) {
  return (
    <svg className={`ico ${className}`} width={size} height={size} viewBox="0 0 24 24" fill={name === "play" || name === "stop" ? "currentColor" : "none"}
      stroke="currentColor" strokeWidth={stroke} strokeLinecap="round" strokeLinejoin="round" aria-hidden="true">{P[name]}</svg>
  );
}

export function Switch({ on, onChange, label, disabled }) {
  return (
    <button type="button" className={`switch${on ? " on" : ""}`} role="switch" aria-checked={!!on} aria-label={label}
      disabled={disabled} onClick={() => onChange(!on)}><span /></button>
  );
}

export function Segmented({ value, options, onChange, small }) {
  return (
    <div className={`seg${small ? " small" : ""}`} role="radiogroup">
      {options.map(([v, l, hint]) => (
        <button type="button" key={String(v)} role="radio" aria-checked={value === v} className={value === v ? "on" : ""}
          title={hint || undefined} onClick={() => onChange(v)}>{l}</button>
      ))}
    </div>
  );
}

export function Chips({ options, value, onToggle, onRemove }) {
  return (
    <div className="chips">
      {options.map(([k, l, extra]) => {
        const on = value.includes(k);
        return (
          <span key={k} role="checkbox" aria-checked={on} tabIndex={0} className={`chip${on ? " on" : ""}`}
            onClick={() => onToggle(k)} onKeyDown={(e) => (e.key === " " || e.key === "Enter") && (e.preventDefault(), onToggle(k))}>
            {on && <Icon name="check" size={12} stroke={2.4} />}{l}{extra}
            {onRemove && onRemove(k) && (
              <button type="button" className="chip-x" aria-label={`Remove ${l}`} onClick={(e) => { e.stopPropagation(); onRemove(k)(); }}>
                <Icon name="x" size={11} stroke={2.4} />
              </button>
            )}
          </span>
        );
      })}
    </div>
  );
}

export function Field({ label, hint, children, className = "" }) {
  return (
    // a div, not a label: a label would forward clicks to the first button of a segmented control
    <div className={`field ${className}`} role="group" aria-label={label}>
      <span className="field-l">{label}</span>
      {children}
      {hint && <span className="field-h">{hint}</span>}
    </div>
  );
}

export function NumberInput({ value, onChange, step = 1, min, max, ...rest }) {
  return <input type="number" value={value} step={step} min={min} max={max} onChange={(e) => onChange(e.target.value === "" ? "" : Number(e.target.value))} {...rest} />;
}

function useEscape(onClose, active) {
  useEffect(() => {
    if (!active) return undefined;
    const h = (e) => e.key === "Escape" && onClose();
    window.addEventListener("keydown", h);
    return () => window.removeEventListener("keydown", h);
  }, [onClose, active]);
}

export function Modal({ open, title, subtitle, onClose, children, footer, wide }) {
  useEscape(onClose, open);
  if (!open) return null;
  return (
    <div className="overlay" onMouseDown={(e) => e.target === e.currentTarget && onClose()}>
      <div className={`modal${wide ? " wide" : ""}`} role="dialog" aria-modal="true" aria-label={title}>
        <div className="modal-hd">
          <div><h2>{title}</h2>{subtitle && <p>{subtitle}</p>}</div>
          <button type="button" className="icon-btn" onClick={onClose} aria-label="Close"><Icon name="x" /></button>
        </div>
        <div className="modal-bd">{children}</div>
        {footer && <div className="modal-ft">{footer}</div>}
      </div>
    </div>
  );
}

export function Drawer({ open, title, onClose, children, tabs }) {
  useEscape(onClose, open);
  if (!open) return null;
  return (
    <div className="overlay drawer-wrap" onMouseDown={(e) => e.target === e.currentTarget && onClose()}>
      <aside className="drawer" role="dialog" aria-modal="true" aria-label={title}>
        <div className="drawer-hd"><h2>{title}</h2>{tabs}<div className="grow" />
          <button type="button" className="btn ghost" onClick={onClose}><Icon name="x" /> Close</button></div>
        <div className="drawer-bd">{children}</div>
      </aside>
    </div>
  );
}

export function Empty({ icon = "list", title, children }) {
  return <div className="empty"><span className="empty-ico"><Icon name={icon} size={22} /></span><b>{title}</b>{children && <span>{children}</span>}</div>;
}

// ------------------------------------------------------------------ toasts
const ToastCtx = createContext(() => {});
export const useToast = () => useContext(ToastCtx);
export function ToastProvider({ children }) {
  const [items, setItems] = useState([]);
  const id = useRef(0);
  const push = useCallback((text, kind = "info") => {
    const k = ++id.current;
    setItems((x) => [...x, { k, text, kind }]);
    setTimeout(() => setItems((x) => x.filter((t) => t.k !== k)), 4200);
  }, []);
  return (
    <ToastCtx.Provider value={push}>
      {children}
      <div className="toasts">{items.map((t) => <div key={t.k} className={`toast ${t.kind}`}><Icon name={t.kind === "error" ? "alert" : "check"} />{t.text}</div>)}</div>
    </ToastCtx.Provider>
  );
}

export function useBusy() {
  const [busy, setBusy] = useState(false);
  const run = useCallback(async (fn) => { setBusy(true); try { return await fn(); } finally { setBusy(false); } }, []);
  return [busy, run];
}




















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
      <nav className="stepper" aria-label="Setup steps" ref={stepperRef}>
        {ind && <span className="stp-ind" aria-hidden="true" style={{ transform: `translate(${ind.left}px, ${ind.top}px)`, width: ind.width, height: ind.height }} />}
        {STEPS.map((s, i) => {
          const st = status[s.key];
          return (
            <button type="button" key={s.key} className={`stp${i === step ? " on" : ""}${st.ok ? " ok" : ""}${st.warn ? " warn" : ""}`} onClick={() => setStep(i)}>
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
            <div><span className="step-n">Step {step + 1} of {STEPS.length}</span><h2>{cur.title}</h2><p>{cur.lead}</p></div>
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




















//generate.css
/* Generate Dataset page. Uses the tokens from styles.css (coral theme). */

.gen { display: grid; gap: 20px; max-width: 1360px; margin: 0 auto; }

/* ---- stepper */
.stepper { display: grid; grid-template-columns: repeat(5, minmax(0, 1fr)); gap: 6px; padding: 6px; background: var(--surface);
  border: 1px solid var(--line); border-radius: var(--radius-lg); box-shadow: var(--shadow); }
.stp { position: relative; display: flex; align-items: center; gap: 12px; padding: 10px 12px; border: 0; border-radius: 12px; background: none;
  cursor: pointer; text-align: left; min-width: 0; transition: background .12s; }
.stp:not(.on):hover { background: rgba(17, 24, 39, .025); }
.stp + .stp::before { content: ""; position: absolute; left: -6px; top: 50%; width: 6px; height: 2px; background: var(--line); }
.stp-n { width: 32px; height: 32px; border-radius: 50%; display: grid; place-items: center; flex: none; font-weight: 700; font-size: 14px;
  background: var(--surface-3); color: var(--muted); border: 1px solid var(--line); transition: all .15s; }
.stp-t { display: grid; min-width: 0; }
.stp-t b { font-size: 14.5px; font-weight: 600; color: var(--ink-2); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.stp-t small { font-size: 12.5px; color: var(--faint); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.stp.ok .stp-n { background: var(--surface); color: var(--green-600); border-color: var(--line-strong); }
.stp.ok .stp-t small { color: var(--muted); }
.stp.on { background: none; }
.stp.on .stp-n { background: var(--grad); color: #fff; border-color: transparent; box-shadow: none; }
.stp.on .stp-t b { color: var(--ink); }

/* ---- main card + summary */
.gen-grid { display: grid; grid-template-columns: minmax(0, 1fr) 340px; gap: 20px; align-items: start; }
.gen-main { min-height: 560px; }
.gen-hd { display: flex; gap: 16px; align-items: flex-start; padding: 22px 26px 18px; border-bottom: 1px solid var(--line-2); }
.gen-hd-ico { width: 44px; height: 44px; border-radius: 12px; display: grid; place-items: center; flex: none; background: var(--surface-2); color: var(--ink-2);
  border: 1px solid var(--line); }
.gen-hd h2 { margin: 2px 0 4px; font-size: 22px; font-weight: 700; letter-spacing: -.2px; }
.gen-hd p { margin: 0; color: var(--muted); font-size: 14.5px; max-width: 640px; }
.gen-bd { padding: 22px 26px 26px; display: grid; gap: 26px; align-content: start; }
.gen-ft { display: flex; align-items: center; gap: 10px; padding: 14px 26px; border-top: 1px solid var(--line-2); background: var(--surface-2);
  border-radius: 0 0 var(--radius-lg) var(--radius-lg); position: sticky; bottom: 0; }

/* sections inside a step */
.g-sec { display: grid; gap: 14px; }
.g-sec-hd { display: flex; align-items: flex-end; justify-content: space-between; gap: 12px; }
.g-sec-hd h3 { margin: 0; font-size: 15.5px; font-weight: 650; letter-spacing: -.1px; }
.g-sec-hd p { margin: 2px 0 0; font-size: 13.5px; color: var(--muted); }
.note { display: flex; gap: 8px; align-items: center; margin: 0; font-size: 13.5px; color: var(--muted); padding: 10px 12px; background: var(--surface-2);
  border: 1px dashed var(--line-strong); border-radius: 10px; }
.btn.lg { height: 44px; padding: 0 20px; font-size: 14.5px; border-radius: 10px; }

/* status badges */
.badge { display: inline-flex; align-items: center; gap: 6px; font-size: 12.5px; font-weight: 600; padding: 3px 10px; border-radius: 99px;
  background: var(--surface-3); color: var(--muted); white-space: nowrap; }
.badge i { width: 7px; height: 7px; border-radius: 50%; background: currentColor; }
.badge.ok { background: var(--green-50); color: var(--green-600); }
.badge.bad { background: var(--red-50); color: var(--red); }
.badge.run { background: var(--accent-50); color: var(--accent-600); }
.badge.run i { animation: pulse 1.3s infinite; }

/* choice cards */
.choices { display: grid; gap: 12px; }
.choices.two { grid-template-columns: repeat(2, minmax(0, 1fr)); }
.choices.five { grid-template-columns: repeat(auto-fill, minmax(170px, 1fr)); }
.choice { display: grid; gap: 4px; align-content: start; text-align: left; padding: 14px 16px; border: 1.5px solid var(--line); border-radius: 14px;
  background: var(--surface); cursor: pointer; transition: border-color .12s, box-shadow .12s, background .12s; }
.choice:hover { border-color: var(--line-strong); box-shadow: var(--shadow-md); }
.choice-top { display: flex; align-items: center; justify-content: space-between; margin-bottom: 6px; }
.choice-ico { width: 36px; height: 36px; border-radius: 10px; display: grid; place-items: center; background: var(--surface-3); color: var(--muted); transition: all .15s; }
.choice-mark { width: 20px; height: 20px; border-radius: 50%; border: 1.5px solid var(--line-strong); display: grid; place-items: center; color: #fff; margin-left: auto; transition: all .15s; }
.choice-mark.sq { border-radius: 6px; }
.choice b { font-size: 14.5px; font-weight: 600; color: var(--ink); }
.choice small { font-size: 13px; color: var(--muted); line-height: 1.45; }
.choice.on { border-color: var(--accent); background: var(--surface); box-shadow: 0 0 0 3px rgba(var(--accent-rgb), .07); }
.choice.on .choice-ico { background: var(--accent-50); color: var(--accent-600); }
.choice.on .choice-mark { background: var(--accent); border-color: transparent; }

/* documents */
.dropzone { display: grid; justify-items: center; gap: 10px; padding: 34px 20px; border: 1.5px dashed var(--line-strong); border-radius: 16px; text-align: center;
  background: var(--surface-2); cursor: pointer; transition: border-color .15s, background .15s, transform .15s; }
.dropzone:hover, .dropzone.drag { border-color: var(--accent-400); background: var(--tint-0); }
.dropzone.drag { transform: scale(1.005); }
.dz-ico { width: 56px; height: 56px; border-radius: 16px; display: grid; place-items: center; background: var(--surface); color: var(--accent-600);
  border: 1px solid var(--line); box-shadow: var(--shadow); }
.dropzone b { font-size: 16px; font-weight: 600; }
.dropzone u { text-decoration: none; color: var(--accent-600); }
.dz-types { display: flex; gap: 6px; flex-wrap: wrap; justify-content: center; }
.dz-types span { font-size: 12px; font-weight: 600; color: var(--muted); background: var(--surface); border: 1px solid var(--line); padding: 2px 9px; border-radius: 99px; }
.filegrid { display: grid; grid-template-columns: repeat(auto-fill, minmax(230px, 1fr)); gap: 8px; }
.fcard { display: flex; align-items: center; gap: 10px; padding: 9px 10px; border: 1px solid var(--line); border-radius: 12px; background: var(--surface); min-width: 0; }
.fcard-n { display: grid; flex: 1; min-width: 0; font-weight: 600; font-size: 14px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.fcard-n small { font-weight: 400; font-size: 12.5px; color: var(--muted); }

/* models */
.epcards { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 16px; }
.epcard { display: grid; gap: 14px; align-content: start; padding: 18px; border: 1px solid var(--line); border-radius: 16px; background: var(--surface); box-shadow: var(--shadow); }
.epcard.off { background: var(--surface-2); }
.epcard-hd { display: flex; align-items: center; gap: 12px; }
.epcard-ft { display: flex; justify-content: flex-end; }
.testbar { display: flex; align-items: center; gap: 10px; padding: 12px 14px; border-radius: 12px; border: 1px solid var(--line); background: var(--surface-2); font-size: 14px; color: var(--ink-2); }
.testbar.good { background: var(--green-50); border-color: #c8eedd; color: #065f46; }
.testbar.bad { background: var(--red-50); border-color: #fbd5d5; color: #991b1b; }

/* option toggles (mix) */
.opt-ico { width: 38px; height: 38px; border-radius: 11px; display: grid; place-items: center; flex: none; background: var(--surface-3); color: var(--muted); transition: all .15s; }
.opt-ico.on, .opt.on .opt-ico { background: var(--accent-50); color: var(--accent-600); }
.opt-t { display: grid; min-width: 0; flex: 1; }
.opt-t b { font-size: 15px; font-weight: 600; }
.opt-t small { font-size: 13px; color: var(--muted); }
.opt { border: 1px solid var(--line); border-radius: 16px; background: var(--surface); transition: border-color .15s, box-shadow .15s; }
.opt:hover { border-color: var(--line-strong); }
.opt.on { border-color: var(--line-strong); box-shadow: var(--shadow); }
.opt-top { display: flex; align-items: center; gap: 14px; padding: 14px 16px; }
.opt-bd { display: grid; gap: 12px; padding: 14px 16px 16px; border-top: 1px dashed var(--line); }
.mixsum { display: grid; gap: 10px; padding: 16px 18px; border-radius: 16px; background: var(--surface-2); border: 1px solid var(--line); }
.mixsum-hd { display: flex; justify-content: space-between; align-items: center; }
.addtopic { display: flex; gap: 8px; max-width: 420px; }
.rules2 { display: grid; gap: 8px; }
.rule2 { border: 1px solid var(--line); border-radius: 12px; background: var(--surface); transition: opacity .15s; }
.rule2.off { opacity: .6; background: var(--surface-2); }
.rule2.open { border-color: var(--line-strong); opacity: 1; }
.rule2-top { display: flex; align-items: center; gap: 12px; padding: 10px 12px; }
.rule2-name { display: grid; flex: 1; min-width: 0; text-align: left; border: 0; background: none; padding: 0; cursor: pointer; }
.rule2-name b { font-size: 14px; font-weight: 600; }
.rule2-name small { font-size: 12.5px; color: var(--muted); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.rule2-bd { display: grid; gap: 12px; padding: 4px 12px 12px; }

/* questions: split */
.splitbar { display: flex; height: 34px; border-radius: 10px; overflow: hidden; gap: 3px; background: var(--line-2); }
.splitbar i { display: grid; place-items: center; min-width: 0; font-style: normal; transition: flex .25s; }
.splitbar span { font-size: 12.5px; font-weight: 600; color: #fff; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; padding: 0 6px; text-shadow: 0 1px 2px rgba(0, 0, 0, .15); }
.split4 { display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); gap: 10px; }

/* advanced: folds */
.fold { border: 1px solid var(--line); border-radius: 14px; background: var(--surface); }
.fold[open] { box-shadow: var(--shadow); }
.fold summary { display: flex; align-items: center; gap: 14px; padding: 14px 16px; cursor: pointer; list-style: none; }
.fold summary::-webkit-details-marker { display: none; }
.fold summary .opt-ico { width: 34px; height: 34px; border-radius: 10px; }
.fold[open] summary .opt-ico { background: var(--surface-3); color: var(--ink-2); }
.fold-chev { color: var(--faint); transition: transform .15s; }
.fold[open] .fold-chev { transform: rotate(90deg); }
.fold-bd { padding: 4px 16px 18px; }

/* ---- summary panel */
.summary { position: sticky; top: 0; overflow: hidden; }
.sum-hd { padding: 20px 20px 16px; display: grid; gap: 4px; justify-items: start; background: var(--surface); border-bottom: 1px solid var(--line-2); }
.sum-hd h3 { margin: 0 0 6px; font-size: 19px; font-weight: 700; letter-spacing: -.3px; word-break: break-word; }
.sum-rows { display: grid; padding: 6px 8px; }
.sum-row { display: flex; gap: 12px; align-items: flex-start; padding: 11px 12px; border: 0; background: none; border-radius: 10px; text-align: left; cursor: pointer; width: 100%; }
.sum-row:hover { background: var(--surface-2); }
.sum-row + .sum-row { border-top: 1px solid var(--line-2); border-radius: 0; }
.sum-ico { width: 30px; height: 30px; border-radius: 9px; display: grid; place-items: center; background: var(--surface-3); color: var(--muted); flex: none; }
.sum-t { display: grid; gap: 3px; flex: 1; min-width: 0; font-size: 14px; color: var(--ink); }
.sum-t > small { font-size: 11.5px; font-weight: 600; text-transform: uppercase; letter-spacing: .5px; color: var(--faint); }
.sum-t em { color: var(--muted); font-style: normal; }
.sum-sub { display: block; font-size: 13px; color: var(--muted); }
.sum-ep { display: flex; align-items: center; gap: 8px; font-size: 13.5px; color: var(--muted); }
.sum-ep b { color: var(--ink); font-weight: 600; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; min-width: 0; }
.sum-edit { color: var(--faint); opacity: 0; margin-top: 8px; transition: opacity .12s; }
.sum-row:hover .sum-edit { opacity: 1; }
.sum-ft { display: grid; gap: 10px; padding: 16px 20px 20px; border-top: 1px solid var(--line-2); background: var(--surface-2); }
.sum-ready { display: flex; align-items: center; gap: 8px; font-weight: 600; font-size: 14px; color: var(--green-600); }
.sum-issues { display: grid; gap: 6px; padding: 12px; border-radius: 12px; background: #fffbeb; border: 1px solid #fde68a; }
.sum-issues b { display: flex; align-items: center; gap: 7px; font-size: 13.5px; color: #92400e; }
.sum-issues button { display: flex; align-items: center; justify-content: space-between; gap: 8px; text-align: left; border: 0; background: none; padding: 2px 0;
  font-size: 13px; color: #92400e; cursor: pointer; }
.sum-issues button:hover { color: #78350f; text-decoration: underline; }
.sum-note { margin: 0; font-size: 12.5px; color: var(--muted); text-align: center; }

/* ---- run view */
.runview { display: grid; gap: 20px; max-width: 1100px; margin: 0 auto; }
.run-loading { flex-direction: row; align-items: center; gap: 10px; padding: 24px; color: var(--muted); }
.run-hero { padding: 24px 26px; gap: 20px; }

.run-hero-top { display: flex; align-items: center; gap: 16px; }
.run-badge { width: 56px; height: 56px; border-radius: 16px; display: grid; place-items: center; color: #fff; flex: none; background: var(--faint); }
.run-badge.run { background: var(--accent-50); color: var(--accent-600); box-shadow: inset 0 0 0 1px var(--accent-100); }
.run-badge.run .ico { animation: spin 2.4s linear infinite; }
.run-badge.ok { background: var(--green-50); color: var(--green-600); box-shadow: inset 0 0 0 1px #c8eedd; }
.run-badge.bad { background: var(--red-50); color: var(--red); box-shadow: inset 0 0 0 1px #fbd5d5; }
@keyframes spin { to { transform: rotate(360deg); } }
.run-title .crumb { font-size: 12.5px; color: var(--muted); }
.run-title .mono { background: var(--surface-3); padding: 1px 6px; border-radius: 5px; }
.run-title h2 { margin: 4px 0 2px; font-size: 24px; font-weight: 700; letter-spacing: -.25px; }
.run-title p { margin: 0; font-size: 15px; color: var(--ink-2); }
.run-meter { display: grid; gap: 8px; }
.run-bar { height: 12px; border-radius: 99px; background: rgba(17, 24, 39, .06); overflow: hidden; }
.run-bar i { display: block; height: 100%; border-radius: 99px; background: var(--grad); transition: width .4s; }
.run-bar i.ok { background: var(--green); } .run-bar i.bad { background: var(--red); }
.run-meta { display: flex; align-items: center; justify-content: space-between; font-size: 14px; color: var(--muted); }
.run-meta b { font-size: 16px; color: var(--ink); font-variant-numeric: tabular-nums; }
.run-meta span { display: inline-flex; align-items: center; gap: 6px; }
.run-actions { display: flex; align-items: center; gap: 10px; flex-wrap: wrap; }
.tl { list-style: none; margin: 0; padding: 14px 22px 20px; display: grid; }
.tl-item { position: relative; display: flex; gap: 14px; padding: 8px 0; }
.tl-item:not(:last-child)::after { content: ""; position: absolute; left: 14px; top: 38px; bottom: -8px; width: 2px; background: var(--line-2); }
.tl-item.done:not(:last-child)::after { background: var(--line); }
.tl-dot { width: 30px; height: 30px; border-radius: 50%; display: grid; place-items: center; flex: none; font-size: 12.5px; font-weight: 700;
  background: var(--surface); color: var(--faint); border: 1.5px solid var(--line); z-index: 1; }
.tl-item.done .tl-dot { background: var(--surface); border-color: var(--line-strong); color: var(--green-600); }
.tl-item.now .tl-dot { background: var(--accent); border-color: transparent; color: #fff; animation: pulse 1.3s infinite; }
.tl-item.fail .tl-dot { background: var(--red); border-color: var(--red); color: #fff; }
.tl-item.skip .tl-dot { background: var(--surface-3); }
.tl-body { flex: 1; min-width: 0; display: grid; gap: 4px; padding-top: 4px; }
.tl-row { display: flex; align-items: baseline; gap: 10px; }
.tl-row b { font-size: 14.5px; font-weight: 600; flex: 1; }
.tl-item.wait .tl-row b, .tl-item.skip .tl-row b { color: var(--muted); font-weight: 500; }
.tl-item.now .tl-row b { color: var(--accent-600); }
.tl-time { font-size: 13px; color: var(--muted); font-variant-numeric: tabular-nums; }
.tl-body small { font-size: 13px; color: var(--muted); }
.tl-bar { height: 6px; border-radius: 99px; background: var(--line-2); overflow: hidden; max-width: 520px; }
.tl-bar i { display: block; height: 100%; background: var(--accent); border-radius: 99px; transition: width .3s; }

/* ---- smaller screens */
@media (max-width: 1240px) {
  .gen-grid { grid-template-columns: minmax(0, 1fr); }
  .summary { position: static; }
  .stp-t small { display: none; }
}
@media (max-width: 900px) {
  .stepper { grid-template-columns: repeat(5, auto); overflow-x: auto; }
  .stp-t { display: none; }
  .stp.on .stp-t { display: grid; }
  .choices.two, .epcards, .grid2 { grid-template-columns: minmax(0, 1fr); }
  .split4 { grid-template-columns: repeat(2, minmax(0, 1fr)); }
  .run-hero-top { flex-wrap: wrap; }
}

/* ---- calm accents: the gradient is kept for the main action only */
.btn.dark { background: var(--ink); border-color: var(--ink); color: #fff; }
.btn.dark:hover:not(:disabled) { background: #1f2937; border-color: #1f2937; color: #fff; }
.gen .switch.on, .gen .switch.on:hover:not(:disabled) { background: var(--accent); }
.gen .chip.on { background: var(--surface); border-color: var(--line-strong); color: var(--ink); }
.gen .chip.on .ico { color: var(--accent); }
.gen .seg button.on { color: var(--ink); }

/* ---- JetBrains Mono for IDs, numbers and file names */
.fcard-n, .fcard-n small, .tl-time, .run-meta b, .mixleg b, .split4 input, .sum-ep b, .run-title .mono { font-family: var(--mono); letter-spacing: -.02em; }
.fcard-n { font-size: 13.5px; font-weight: 500; }

/* ================================================================ motion
   Subtle and quick: things fade and rise a few pixels, the active step and tab slide.
   Everything is switched off for people who ask their system for reduced motion. */
:root { --ease-out: cubic-bezier(.2, .8, .2, 1); --ease-spring: cubic-bezier(.34, 1.4, .64, 1); }

@keyframes rise-in { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: none; } }
@keyframes fade-in { from { opacity: 0; } to { opacity: 1; } }
@keyframes slide-fwd { from { opacity: 0; transform: translateX(18px); } to { opacity: 1; transform: none; } }
@keyframes slide-back { from { opacity: 0; transform: translateX(-18px); } to { opacity: 1; transform: none; } }
@keyframes pop-in { 0% { opacity: 0; transform: scale(.4); } 100% { opacity: 1; transform: scale(1); } }
@keyframes open-down { from { opacity: 0; transform: translateY(-6px); } to { opacity: 1; transform: none; } }

/* page enter: stepper, main card and summary rise in one after the other */
.gen > .stepper { animation: rise-in .45s var(--ease-out) both; }
.gen-grid > .gen-main { animation: rise-in .5s var(--ease-out) .06s both; }
.gen-grid > .summary { animation: rise-in .5s var(--ease-out) .12s both; }
main .content > * { animation: fade-in .3s ease both; }

/* stepper: a soft card slides behind the active step */
.stepper { position: relative; }
.stp { z-index: 1; }
.stp.on { background: none; }
.stp-ind { position: absolute; top: 0; left: 0; border-radius: 12px; background: var(--surface-2); pointer-events: none;
  box-shadow: inset 0 0 0 1px var(--line); transition: transform .42s var(--ease-out), width .42s var(--ease-out); }
.stp-n { transition: background .25s, color .25s, border-color .25s, transform .2s var(--ease-spring); }
.stp:hover .stp-n { transform: scale(1.06); }
.stp-n .pop { animation: pop-in .35s var(--ease-spring) both; display: inline-flex; }
.stp-t b, .stp-t small { transition: color .25s; }

/* step content: header and body slide in from the direction you moved, sections follow in a short cascade */
.gen-hd.anim-fwd { animation: slide-fwd .38s var(--ease-out) both; }
.gen-hd.anim-back { animation: slide-back .38s var(--ease-out) both; }
.gen-bd.anim-fwd > *, .gen-bd.anim-back > * { animation: rise-in .45s var(--ease-out) both; }
.gen-bd > *:nth-child(1) { animation-delay: .04s; }
.gen-bd > *:nth-child(2) { animation-delay: .09s; }
.gen-bd > *:nth-child(3) { animation-delay: .14s; }
.gen-bd > *:nth-child(4) { animation-delay: .19s; }
.gen-bd > *:nth-child(n+5) { animation-delay: .24s; }

/* cards and options: gentle lift on hover, the tick pops in when chosen */
.choice { transition: border-color .2s, box-shadow .2s, background .2s, transform .2s var(--ease-out); }
.choice:hover { transform: translateY(-2px); }
.choice:active { transform: translateY(0) scale(.995); }
.choice-ico { transition: background .25s, color .25s, transform .25s var(--ease-spring); }
.choice.on .choice-ico { transform: scale(1.05); }
.choice.on .choice-mark .ico { animation: pop-in .3s var(--ease-spring) both; }
.fcard { animation: rise-in .35s var(--ease-out) both; transition: border-color .2s, box-shadow .2s, transform .2s var(--ease-out); }
.fcard:hover { border-color: var(--line-strong); box-shadow: var(--shadow-md); transform: translateY(-1px); }
.dz-ico { transition: transform .3s var(--ease-spring), box-shadow .3s; }
.dropzone:hover .dz-ico, .dropzone.drag .dz-ico { transform: translateY(-3px); box-shadow: var(--shadow-md); }
.epcard, .opt, .fold, .rule2 { transition: border-color .2s, box-shadow .25s, background .2s, opacity .2s; }
.epcard:hover { box-shadow: var(--shadow-md); }
.opt-ico { transition: background .25s, color .25s, transform .25s var(--ease-spring); }
.opt.on .opt-ico { transform: scale(1.04); }
.opt-bd, .rule2-bd, .fold-bd { animation: open-down .3s var(--ease-out) both; }
.chip { transition: background .2s, border-color .2s, color .2s, transform .15s var(--ease-out); }
.chip:active { transform: scale(.96); }
.chip.on .ico { animation: pop-in .25s var(--ease-spring) both; }
.fold-chev { transition: transform .3s var(--ease-out); }
.splitbar i { transition: flex .45s var(--ease-out); }
.mixbar i { transition: width .45s var(--ease-out); }
.switch span { transition: left .25s var(--ease-spring), background .2s; }
.switch { transition: background .25s; }

/* buttons: small press feedback; the main action lifts and its arrow nudges forward */
.btn { transition: background .2s, border-color .2s, color .2s, box-shadow .2s, transform .15s var(--ease-out); }
.btn:active:not(:disabled) { transform: scale(.98); }
.btn.primary:hover:not(:disabled), .btn.dark:hover:not(:disabled) { transform: translateY(-1px); box-shadow: 0 8px 18px -8px rgba(16, 24, 40, .35); }
.btn.primary:hover:not(:disabled) { box-shadow: 0 10px 22px -8px rgba(var(--accent-rgb), .55); }
.btn .ico { transition: transform .2s var(--ease-out); }
.btn:hover:not(:disabled) > .ico:last-child:not(:first-child) { transform: translateX(2px); }
.gen-ft .btn.ghost:hover:not(:disabled) > .ico:first-child { transform: translateX(-2px); }

/* summary: the status box swaps smoothly, bars glide to their new size */
.sum-row { transition: background .2s; }
.sum-ico { transition: background .2s, color .2s; }
.sum-row:hover .sum-ico { background: var(--accent-50); color: var(--accent-600); }
.sum-ready, .sum-issues { animation: rise-in .35s var(--ease-out) both; }
.sum-issues button { transition: color .15s, transform .15s; }

/* run view: hero and timeline rise in, steps cascade */
.runview > .run-hero { animation: rise-in .45s var(--ease-out) both; }
.runview > .card:not(.run-hero) { animation: rise-in .5s var(--ease-out) .08s both; }
.tl-item { animation: rise-in .4s var(--ease-out) both; }
.tl-item:nth-child(1) { animation-delay: .10s; } .tl-item:nth-child(2) { animation-delay: .13s; } .tl-item:nth-child(3) { animation-delay: .16s; }
.tl-item:nth-child(4) { animation-delay: .19s; } .tl-item:nth-child(5) { animation-delay: .22s; } .tl-item:nth-child(6) { animation-delay: .25s; }
.tl-item:nth-child(7) { animation-delay: .28s; } .tl-item:nth-child(n+8) { animation-delay: .31s; }
.run-bar i { transition: width .6s var(--ease-out); }
.run-badge { animation: pop-in .45s var(--ease-spring) .1s both; }


@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after { animation-duration: .01ms !important; animation-delay: 0s !important; transition-duration: .01ms !important; }
}






















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
      <nav className="stepper" aria-label="Setup steps" ref={stepperRef}>
        {ind && <span className="stp-ind" aria-hidden="true" style={{ transform: `translate(${ind.left}px, ${ind.top}px)`, width: ind.width, height: ind.height }} />}
        {STEPS.map((s, i) => {
          const st = status[s.key];
          return (
            <button type="button" key={s.key} className={`stp${i === step ? " on" : ""}${st.ok ? " ok" : ""}${st.warn ? " warn" : ""}`} onClick={() => setStep(i)}>
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
            <div><span className="step-n">Step {step + 1} of {STEPS.length}</span><h2>{cur.title}</h2><p>{cur.lead}</p></div>
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

























//generate.css
/* Generate Dataset page. Uses the tokens from styles.css (coral theme). */

.gen { display: grid; gap: 20px; max-width: 1360px; margin: 0 auto; }

/* ---- stepper */
.stepper { display: grid; grid-template-columns: repeat(5, minmax(0, 1fr)); gap: 6px; padding: 6px; background: var(--surface);
  border: 1px solid var(--line); border-radius: var(--radius-lg); box-shadow: var(--shadow); }
.stp { position: relative; display: flex; align-items: center; gap: 12px; padding: 10px 12px; border: 0; border-radius: 12px; background: none;
  cursor: pointer; text-align: left; min-width: 0; transition: background .12s; }
.stp:not(.on):hover { background: rgba(17, 24, 39, .025); }
.stp + .stp::before { content: ""; position: absolute; left: -6px; top: 50%; width: 6px; height: 2px; background: var(--line); }
.stp-n { width: 32px; height: 32px; border-radius: 50%; display: grid; place-items: center; flex: none; font-weight: 700; font-size: 14px;
  background: var(--surface-3); color: var(--muted); border: 1px solid var(--line); transition: all .15s; }
.stp-t { display: grid; min-width: 0; }
.stp-t b { font-size: 14.5px; font-weight: 600; color: var(--ink-2); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.stp-t small { font-size: 12.5px; color: var(--faint); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.stp.ok .stp-n { background: var(--surface); color: var(--green-600); border-color: var(--line-strong); }
.stp.ok .stp-t small { color: var(--muted); }
.stp.on { background: none; }
.stp.on .stp-n { background: var(--grad); color: #fff; border-color: transparent; box-shadow: none; }
.stp.on .stp-t b { color: var(--ink); }

/* ---- main card + summary */
.gen-grid { display: grid; grid-template-columns: minmax(0, 1fr) 340px; gap: 20px; align-items: start; }
.gen-main { min-height: 560px; }
.gen-hd { display: flex; gap: 16px; align-items: flex-start; padding: 22px 26px 18px; border-bottom: 1px solid var(--line-2); }
.gen-hd-ico { width: 44px; height: 44px; border-radius: 12px; display: grid; place-items: center; flex: none; background: var(--surface-2); color: var(--ink-2);
  border: 1px solid var(--line); }
.gen-hd h2 { margin: 2px 0 4px; font-size: 22px; font-weight: 700; letter-spacing: -.2px; }
.gen-hd p { margin: 0; color: var(--muted); font-size: 14.5px; max-width: 640px; }
.gen-bd { padding: 22px 26px 26px; display: grid; gap: 26px; align-content: start; }
.gen-ft { display: flex; align-items: center; gap: 10px; padding: 14px 26px; border-top: 1px solid var(--line-2); background: var(--surface-2);
  border-radius: 0 0 var(--radius-lg) var(--radius-lg); position: sticky; bottom: 0; }

/* sections inside a step */
.g-sec { display: grid; gap: 14px; }
.g-sec-hd { display: flex; align-items: flex-end; justify-content: space-between; gap: 12px; }
.g-sec-hd h3 { margin: 0; font-size: 15.5px; font-weight: 650; letter-spacing: -.1px; }
.g-sec-hd p { margin: 2px 0 0; font-size: 13.5px; color: var(--muted); }
.note { display: flex; gap: 8px; align-items: center; margin: 0; font-size: 13.5px; color: var(--muted); padding: 10px 12px; background: var(--surface-2);
  border: 1px dashed var(--line-strong); border-radius: 10px; }
.btn.lg { height: 44px; padding: 0 20px; font-size: 14.5px; border-radius: 10px; }

/* status badges */
.badge { display: inline-flex; align-items: center; gap: 6px; font-size: 12.5px; font-weight: 600; padding: 3px 10px; border-radius: 99px;
  background: var(--surface-3); color: var(--muted); white-space: nowrap; }
.badge i { width: 7px; height: 7px; border-radius: 50%; background: currentColor; }
.badge.ok { background: var(--green-50); color: var(--green-600); }
.badge.bad { background: var(--red-50); color: var(--red); }
.badge.run { background: var(--accent-50); color: var(--accent-600); }
.badge.run i { animation: pulse 1.3s infinite; }

/* choice cards */
.choices { display: grid; gap: 12px; }
.choices.two { grid-template-columns: repeat(2, minmax(0, 1fr)); }
.choices.five { grid-template-columns: repeat(auto-fill, minmax(170px, 1fr)); }
.choice { display: grid; gap: 4px; align-content: start; text-align: left; padding: 14px 16px; border: 1.5px solid var(--line); border-radius: 14px;
  background: var(--surface); cursor: pointer; transition: border-color .12s, box-shadow .12s, background .12s; }
.choice:hover { border-color: var(--line-strong); box-shadow: var(--shadow-md); }
.choice-top { display: flex; align-items: center; justify-content: space-between; margin-bottom: 6px; }
.choice-ico { width: 36px; height: 36px; border-radius: 10px; display: grid; place-items: center; background: var(--surface-3); color: var(--muted); transition: all .15s; }
.choice-mark { width: 20px; height: 20px; border-radius: 50%; border: 1.5px solid var(--line-strong); display: grid; place-items: center; color: #fff; margin-left: auto; transition: all .15s; }
.choice-mark.sq { border-radius: 6px; }
.choice b { font-size: 14.5px; font-weight: 600; color: var(--ink); }
.choice small { font-size: 13px; color: var(--muted); line-height: 1.45; }
.choice.on { border-color: var(--accent); background: var(--surface); box-shadow: 0 0 0 3px rgba(var(--accent-rgb), .07); }
.choice.on .choice-ico { background: var(--accent-50); color: var(--accent-600); }
.choice.on .choice-mark { background: var(--accent); border-color: transparent; }

/* documents */
.dropzone { display: grid; justify-items: center; gap: 10px; padding: 34px 20px; border: 1.5px dashed var(--line-strong); border-radius: 16px; text-align: center;
  background: var(--surface-2); cursor: pointer; transition: border-color .15s, background .15s, transform .15s; }
.dropzone:hover, .dropzone.drag { border-color: var(--accent-400); background: var(--tint-0); }
.dropzone.drag { transform: scale(1.005); }
.dz-ico { width: 56px; height: 56px; border-radius: 16px; display: grid; place-items: center; background: var(--surface); color: var(--accent-600);
  border: 1px solid var(--line); box-shadow: var(--shadow); }
.dropzone b { font-size: 16px; font-weight: 600; }
.dropzone u { text-decoration: none; color: var(--accent-600); }
.dz-types { display: flex; gap: 6px; flex-wrap: wrap; justify-content: center; }
.dz-types span { font-size: 12px; font-weight: 600; color: var(--muted); background: var(--surface); border: 1px solid var(--line); padding: 2px 9px; border-radius: 99px; }
.filegrid { display: grid; grid-template-columns: repeat(auto-fill, minmax(230px, 1fr)); gap: 8px; }
.fcard { display: flex; align-items: center; gap: 10px; padding: 9px 10px; border: 1px solid var(--line); border-radius: 12px; background: var(--surface); min-width: 0; }
.fcard-n { display: grid; flex: 1; min-width: 0; font-weight: 600; font-size: 14px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.fcard-n small { font-weight: 400; font-size: 12.5px; color: var(--muted); }

/* models */
.epcards { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 16px; }
.epcard { display: grid; gap: 14px; align-content: start; padding: 18px; border: 1px solid var(--line); border-radius: 16px; background: var(--surface); box-shadow: var(--shadow); }
.epcard.off { background: var(--surface-2); }
.epcard-hd { display: flex; align-items: center; gap: 12px; }
.epcard-ft { display: flex; justify-content: flex-end; }
.testbar { display: flex; align-items: center; gap: 10px; padding: 12px 14px; border-radius: 12px; border: 1px solid var(--line); background: var(--surface-2); font-size: 14px; color: var(--ink-2); }
.testbar.good { background: var(--green-50); border-color: #c8eedd; color: #065f46; }
.testbar.bad { background: var(--red-50); border-color: #fbd5d5; color: #991b1b; }

/* option toggles (mix) */
.opt-ico { width: 38px; height: 38px; border-radius: 11px; display: grid; place-items: center; flex: none; background: var(--surface-3); color: var(--muted); transition: all .15s; }
.opt-ico.on, .opt.on .opt-ico { background: var(--accent-50); color: var(--accent-600); }
.opt-t { display: grid; min-width: 0; flex: 1; }
.opt-t b { font-size: 15px; font-weight: 600; }
.opt-t small { font-size: 13px; color: var(--muted); }
.opt { border: 1px solid var(--line); border-radius: 16px; background: var(--surface); transition: border-color .15s, box-shadow .15s; }
.opt:hover { border-color: var(--line-strong); }
.opt.on { border-color: var(--line-strong); box-shadow: var(--shadow); }
.opt-top { display: flex; align-items: center; gap: 14px; padding: 14px 16px; }
.opt-bd { display: grid; gap: 12px; padding: 14px 16px 16px; border-top: 1px dashed var(--line); }
.mixsum { display: grid; gap: 10px; padding: 16px 18px; border-radius: 16px; background: var(--surface-2); border: 1px solid var(--line); }
.mixsum-hd { display: flex; justify-content: space-between; align-items: center; }
.addtopic { display: flex; gap: 8px; max-width: 420px; }
.rules2 { display: grid; gap: 8px; }
.rule2 { border: 1px solid var(--line); border-radius: 12px; background: var(--surface); transition: opacity .15s; }
.rule2.off { opacity: .6; background: var(--surface-2); }
.rule2.open { border-color: var(--line-strong); opacity: 1; }
.rule2-top { display: flex; align-items: center; gap: 12px; padding: 10px 12px; }
.rule2-name { display: grid; flex: 1; min-width: 0; text-align: left; border: 0; background: none; padding: 0; cursor: pointer; }
.rule2-name b { font-size: 14px; font-weight: 600; }
.rule2-name small { font-size: 12.5px; color: var(--muted); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.rule2-bd { display: grid; gap: 12px; padding: 4px 12px 12px; }

/* questions: split */
.splitbar { display: flex; height: 34px; border-radius: 10px; overflow: hidden; gap: 3px; background: var(--line-2); }
.splitbar i { display: grid; place-items: center; min-width: 0; font-style: normal; transition: flex .25s; }
.splitbar span { font-size: 12.5px; font-weight: 600; color: #fff; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; padding: 0 6px; text-shadow: 0 1px 2px rgba(0, 0, 0, .15); }
.split4 { display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); gap: 10px; }

/* advanced: folds */
.fold { border: 1px solid var(--line); border-radius: 14px; background: var(--surface); }
.fold[open] { box-shadow: var(--shadow); }
.fold summary { display: flex; align-items: center; gap: 14px; padding: 14px 16px; cursor: pointer; list-style: none; }
.fold summary::-webkit-details-marker { display: none; }
.fold summary .opt-ico { width: 34px; height: 34px; border-radius: 10px; }
.fold[open] summary .opt-ico { background: var(--surface-3); color: var(--ink-2); }
.fold-chev { color: var(--faint); transition: transform .15s; }
.fold[open] .fold-chev { transform: rotate(90deg); }
.fold-bd { padding: 4px 16px 18px; }

/* ---- summary panel */
.summary { position: sticky; top: 0; overflow: hidden; }
.sum-hd { padding: 20px 20px 16px; display: grid; gap: 4px; justify-items: start; background: var(--surface); border-bottom: 1px solid var(--line-2); }
.sum-hd h3 { margin: 0 0 6px; font-size: 19px; font-weight: 700; letter-spacing: -.3px; word-break: break-word; }
.sum-rows { display: grid; padding: 6px 8px; }
.sum-row { display: flex; gap: 12px; align-items: flex-start; padding: 11px 12px; border: 0; background: none; border-radius: 10px; text-align: left; cursor: pointer; width: 100%; }
.sum-row:hover { background: var(--surface-2); }
.sum-row + .sum-row { border-top: 1px solid var(--line-2); border-radius: 0; }
.sum-ico { width: 30px; height: 30px; border-radius: 9px; display: grid; place-items: center; background: var(--surface-3); color: var(--muted); flex: none; }
.sum-t { display: grid; gap: 3px; flex: 1; min-width: 0; font-size: 14px; color: var(--ink); }
.sum-t > small { font-size: 11.5px; font-weight: 600; text-transform: uppercase; letter-spacing: .5px; color: var(--faint); }
.sum-t em { color: var(--muted); font-style: normal; }
.sum-sub { display: block; font-size: 13px; color: var(--muted); }
.sum-ep { display: flex; align-items: center; gap: 8px; font-size: 13.5px; color: var(--muted); }
.sum-ep b { color: var(--ink); font-weight: 600; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; min-width: 0; }
.sum-edit { color: var(--faint); opacity: 0; margin-top: 8px; transition: opacity .12s; }
.sum-row:hover .sum-edit { opacity: 1; }
.sum-ft { display: grid; gap: 10px; padding: 16px 20px 20px; border-top: 1px solid var(--line-2); background: var(--surface-2); }
.sum-ready { display: flex; align-items: center; gap: 8px; font-weight: 600; font-size: 14px; color: var(--green-600); }
.sum-issues { display: grid; gap: 6px; padding: 12px; border-radius: 12px; background: #fffbeb; border: 1px solid #fde68a; }
.sum-issues b { display: flex; align-items: center; gap: 7px; font-size: 13.5px; color: #92400e; }
.sum-issues button { display: flex; align-items: center; justify-content: space-between; gap: 8px; text-align: left; border: 0; background: none; padding: 2px 0;
  font-size: 13px; color: #92400e; cursor: pointer; }
.sum-issues button:hover { color: #78350f; text-decoration: underline; }
.sum-note { margin: 0; font-size: 12.5px; color: var(--muted); text-align: center; }

/* ---- run view */
.runview { display: grid; gap: 20px; max-width: 1100px; margin: 0 auto; }
.run-loading { flex-direction: row; align-items: center; gap: 10px; padding: 24px; color: var(--muted); }
.run-hero { padding: 24px 26px; gap: 20px; }

.run-hero-top { display: flex; align-items: center; gap: 16px; }
.run-badge { width: 56px; height: 56px; border-radius: 16px; display: grid; place-items: center; color: #fff; flex: none; background: var(--faint); }
.run-badge.run { background: var(--accent-50); color: var(--accent-600); box-shadow: inset 0 0 0 1px var(--accent-100); }
.run-badge.run .ico { animation: spin 2.4s linear infinite; }
.run-badge.ok { background: var(--green-50); color: var(--green-600); box-shadow: inset 0 0 0 1px #c8eedd; }
.run-badge.bad { background: var(--red-50); color: var(--red); box-shadow: inset 0 0 0 1px #fbd5d5; }
@keyframes spin { to { transform: rotate(360deg); } }
.run-title .crumb { font-size: 12.5px; color: var(--muted); }
.run-title .mono { background: var(--surface-3); padding: 1px 6px; border-radius: 5px; }
.run-title h2 { margin: 4px 0 2px; font-size: 24px; font-weight: 700; letter-spacing: -.25px; }
.run-title p { margin: 0; font-size: 15px; color: var(--ink-2); }
.run-meter { display: grid; gap: 8px; }
.run-bar { height: 12px; border-radius: 99px; background: rgba(17, 24, 39, .06); overflow: hidden; }
.run-bar i { display: block; height: 100%; border-radius: 99px; background: var(--grad); transition: width .4s; }
.run-bar i.ok { background: var(--green); } .run-bar i.bad { background: var(--red); }
.run-meta { display: flex; align-items: center; justify-content: space-between; font-size: 14px; color: var(--muted); }
.run-meta b { font-size: 16px; color: var(--ink); font-variant-numeric: tabular-nums; }
.run-meta span { display: inline-flex; align-items: center; gap: 6px; }
.run-actions { display: flex; align-items: center; gap: 10px; flex-wrap: wrap; }
.tl { list-style: none; margin: 0; padding: 14px 22px 20px; display: grid; }
.tl-item { position: relative; display: flex; gap: 14px; padding: 8px 0; }
.tl-item:not(:last-child)::after { content: ""; position: absolute; left: 14px; top: 38px; bottom: -8px; width: 2px; background: var(--line-2); }
.tl-item.done:not(:last-child)::after { background: var(--line); }
.tl-dot { width: 30px; height: 30px; border-radius: 50%; display: grid; place-items: center; flex: none; font-size: 12.5px; font-weight: 700;
  background: var(--surface); color: var(--faint); border: 1.5px solid var(--line); z-index: 1; }
.tl-item.done .tl-dot { background: var(--surface); border-color: var(--line-strong); color: var(--green-600); }
.tl-item.now .tl-dot { background: var(--accent); border-color: transparent; color: #fff; animation: pulse 1.3s infinite; }
.tl-item.fail .tl-dot { background: var(--red); border-color: var(--red); color: #fff; }
.tl-item.skip .tl-dot { background: var(--surface-3); }
.tl-body { flex: 1; min-width: 0; display: grid; gap: 4px; padding-top: 4px; }
.tl-row { display: flex; align-items: baseline; gap: 10px; }
.tl-row b { font-size: 14.5px; font-weight: 600; flex: 1; }
.tl-item.wait .tl-row b, .tl-item.skip .tl-row b { color: var(--muted); font-weight: 500; }
.tl-item.now .tl-row b { color: var(--accent-600); }
.tl-time { font-size: 13px; color: var(--muted); font-variant-numeric: tabular-nums; }
.tl-body small { font-size: 13px; color: var(--muted); }
.tl-bar { height: 6px; border-radius: 99px; background: var(--line-2); overflow: hidden; max-width: 520px; }
.tl-bar i { display: block; height: 100%; background: var(--accent); border-radius: 99px; transition: width .3s; }

/* ---- smaller screens */
@media (max-width: 1240px) {
  .gen-grid { grid-template-columns: minmax(0, 1fr); }
  .summary { position: static; }
  .stp-t small { display: none; }
}
@media (max-width: 900px) {
  .stepper { grid-template-columns: repeat(5, auto); overflow-x: auto; }
  .stp-t { display: none; }
  .stp.on .stp-t { display: grid; }
  .choices.two, .epcards, .grid2 { grid-template-columns: minmax(0, 1fr); }
  .split4 { grid-template-columns: repeat(2, minmax(0, 1fr)); }
  .run-hero-top { flex-wrap: wrap; }
}

/* ---- calm accents: the gradient is kept for the main action only */
.btn.dark { background: var(--ink); border-color: var(--ink); color: #fff; }
.btn.dark:hover:not(:disabled) { background: #1f2937; border-color: #1f2937; color: #fff; }
.gen .switch.on, .gen .switch.on:hover:not(:disabled) { background: var(--accent); }
.gen .chip.on { background: var(--surface); border-color: var(--line-strong); color: var(--ink); }
.gen .chip.on .ico { color: var(--accent); }
.gen .seg button.on { color: var(--ink); }

/* ---- JetBrains Mono for IDs, numbers and file names */
.fcard-n, .fcard-n small, .tl-time, .run-meta b, .mixleg b, .split4 input, .sum-ep b, .run-title .mono { font-family: var(--mono); letter-spacing: -.02em; }
.fcard-n { font-size: 13.5px; font-weight: 500; }

/* ================================================================ motion
   Subtle and quick: things fade and rise a few pixels, the active step and tab slide.
   Everything is switched off for people who ask their system for reduced motion. */
:root { --ease-out: cubic-bezier(.2, .8, .2, 1); --ease-spring: cubic-bezier(.34, 1.4, .64, 1); }

@keyframes rise-in { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: none; } }
@keyframes fade-in { from { opacity: 0; } to { opacity: 1; } }
@keyframes slide-fwd { from { opacity: 0; transform: translateX(18px); } to { opacity: 1; transform: none; } }
@keyframes slide-back { from { opacity: 0; transform: translateX(-18px); } to { opacity: 1; transform: none; } }
@keyframes pop-in { 0% { opacity: 0; transform: scale(.4); } 100% { opacity: 1; transform: scale(1); } }
@keyframes open-down { from { opacity: 0; transform: translateY(-6px); } to { opacity: 1; transform: none; } }

/* page enter: stepper, main card and summary rise in one after the other */
.gen > .stepper { animation: rise-in .45s var(--ease-out) both; }
.gen-grid > .gen-main { animation: rise-in .5s var(--ease-out) .06s both; }
.gen-grid > .summary { animation: rise-in .5s var(--ease-out) .12s both; }
main .content > * { animation: fade-in .3s ease both; }

/* stepper: a soft card slides behind the active step */
.stepper { position: relative; }
.stp { z-index: 1; }
.stp.on { background: none; }
.stp-ind { position: absolute; top: 0; left: 0; border-radius: 12px; background: var(--surface-2); pointer-events: none;
  box-shadow: inset 0 0 0 1px var(--line); transition: transform .42s var(--ease-out), width .42s var(--ease-out); }
.stp-n { transition: background .25s, color .25s, border-color .25s, transform .2s var(--ease-spring); }
.stp:hover .stp-n { transform: scale(1.06); }
.stp-n .pop { animation: pop-in .35s var(--ease-spring) both; display: inline-flex; }
.stp-t b, .stp-t small { transition: color .25s; }

/* step content: header and body slide in from the direction you moved, sections follow in a short cascade */
.gen-hd.anim-fwd { animation: slide-fwd .38s var(--ease-out) both; }
.gen-hd.anim-back { animation: slide-back .38s var(--ease-out) both; }
.gen-bd.anim-fwd > *, .gen-bd.anim-back > * { animation: rise-in .45s var(--ease-out) both; }
.gen-bd > *:nth-child(1) { animation-delay: .04s; }
.gen-bd > *:nth-child(2) { animation-delay: .09s; }
.gen-bd > *:nth-child(3) { animation-delay: .14s; }
.gen-bd > *:nth-child(4) { animation-delay: .19s; }
.gen-bd > *:nth-child(n+5) { animation-delay: .24s; }

/* cards and options: gentle lift on hover, the tick pops in when chosen */
.choice { transition: border-color .2s, box-shadow .2s, background .2s, transform .2s var(--ease-out); }
.choice:hover { transform: translateY(-2px); }
.choice:active { transform: translateY(0) scale(.995); }
.choice-ico { transition: background .25s, color .25s, transform .25s var(--ease-spring); }
.choice.on .choice-ico { transform: scale(1.05); }
.choice.on .choice-mark .ico { animation: pop-in .3s var(--ease-spring) both; }
.fcard { animation: rise-in .35s var(--ease-out) both; transition: border-color .2s, box-shadow .2s, transform .2s var(--ease-out); }
.fcard:hover { border-color: var(--line-strong); box-shadow: var(--shadow-md); transform: translateY(-1px); }
.dz-ico { transition: transform .3s var(--ease-spring), box-shadow .3s; }
.dropzone:hover .dz-ico, .dropzone.drag .dz-ico { transform: translateY(-3px); box-shadow: var(--shadow-md); }
.epcard, .opt, .fold, .rule2 { transition: border-color .2s, box-shadow .25s, background .2s, opacity .2s; }
.epcard:hover { box-shadow: var(--shadow-md); }
.opt-ico { transition: background .25s, color .25s, transform .25s var(--ease-spring); }
.opt.on .opt-ico { transform: scale(1.04); }
.opt-bd, .rule2-bd, .fold-bd { animation: open-down .3s var(--ease-out) both; }
.chip { transition: background .2s, border-color .2s, color .2s, transform .15s var(--ease-out); }
.chip:active { transform: scale(.96); }
.chip.on .ico { animation: pop-in .25s var(--ease-spring) both; }
.fold-chev { transition: transform .3s var(--ease-out); }
.splitbar i { transition: flex .45s var(--ease-out); }
.mixbar i { transition: width .45s var(--ease-out); }
.switch span { transition: left .25s var(--ease-spring), background .2s; }
.switch { transition: background .25s; }

/* buttons: small press feedback; the main action lifts and its arrow nudges forward */
.btn { transition: background .2s, border-color .2s, color .2s, box-shadow .2s, transform .15s var(--ease-out); }
.btn:active:not(:disabled) { transform: scale(.98); }
.btn.primary:hover:not(:disabled), .btn.dark:hover:not(:disabled) { transform: translateY(-1px); box-shadow: 0 8px 18px -8px rgba(16, 24, 40, .35); }
.btn.primary:hover:not(:disabled) { box-shadow: 0 10px 22px -8px rgba(var(--accent-rgb), .55); }
.btn .ico { transition: transform .2s var(--ease-out); }
.btn:hover:not(:disabled) > .ico:last-child:not(:first-child) { transform: translateX(2px); }
.gen-ft .btn.ghost:hover:not(:disabled) > .ico:first-child { transform: translateX(-2px); }

/* summary: the status box swaps smoothly, bars glide to their new size */
.sum-row { transition: background .2s; }
.sum-ico { transition: background .2s, color .2s; }
.sum-row:hover .sum-ico { background: var(--accent-50); color: var(--accent-600); }
.sum-ready, .sum-issues { animation: rise-in .35s var(--ease-out) both; }
.sum-issues button { transition: color .15s, transform .15s; }

/* run view: hero and timeline rise in, steps cascade */
.runview > .run-hero { animation: rise-in .45s var(--ease-out) both; }
.runview > .card:not(.run-hero) { animation: rise-in .5s var(--ease-out) .08s both; }
.tl-item { animation: rise-in .4s var(--ease-out) both; }
.tl-item:nth-child(1) { animation-delay: .10s; } .tl-item:nth-child(2) { animation-delay: .13s; } .tl-item:nth-child(3) { animation-delay: .16s; }
.tl-item:nth-child(4) { animation-delay: .19s; } .tl-item:nth-child(5) { animation-delay: .22s; } .tl-item:nth-child(6) { animation-delay: .25s; }
.tl-item:nth-child(7) { animation-delay: .28s; } .tl-item:nth-child(n+8) { animation-delay: .31s; }
.run-bar i { transition: width .6s var(--ease-out); }
.run-badge { animation: pop-in .45s var(--ease-spring) .1s both; }


@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after { animation-duration: .01ms !important; animation-delay: 0s !important; transition-duration: .01ms !important; }
}
