//GeneratePage.jsx
// Generate Dataset page: a guided, step-by-step setup with a live summary, and a progress view once a job runs.
// Same API as the original UI (/api/datagen); the form <-> JobRequest logic is in ../model.js.
import { useCallback, useEffect, useLayoutEffect, useRef, useState } from "react";
import { useStepper } from "../stepper.js";
import { API, api, post, upload } from "../api.js";
import { QA_TEMPLATE, T_FORMATS, T_INPUTS, T_OUTPUTS, buildRequest, defaultForm, endpointsForTest, formFromRequest, isQa, visionUsed } from "../model.js";
import { Icon, useToast } from "../ui.jsx";
import { emit, useBus } from "../bus.js";
import { AdvancedStep, DatasetTypeStep, DocumentsStep, MixStep, ModelsStep, SizeStep } from "./steps.jsx";
import Summary from "./Summary.jsx";
import RunProgress from "./RunProgress.jsx";
import "./generate.css";

export const STEPS = [
  { key: "documents", title: "Documents", icon: "doc", lead: "Upload the files the model should learn from, and choose what happens to the pictures inside them." },
  { key: "models", title: "Models", icon: "server", lead: "The teacher model writes and judges the Q&A pairs; the vision model reads images. Both must already be running." },
  { key: "dataset", title: "Dataset type", icon: "layers", lead: "Choose what one training row looks like. Each card shows an example made from an HR leave-policy document." },
  { key: "size", title: "Size & split", icon: "chart", lead: "Choose how many rows to create, and how they are shared between training and testing." },
  { key: "mix", title: "Training mix", icon: "mix", lead: "Add extra rows next to the document Q&A, so the model stays helpful and well-behaved." },
  { key: "advanced", title: "Advanced", icon: "gear", lead: "Everything here has sensible defaults. Change only what you need." },
];

const jobFromUrl = () => new URL(location.href).searchParams.get("job");

export default function GeneratePage({ openResults }) {
  const toast = useToast();
  const [defaults, setDefaults] = useState(null);
  // dataset templates: Q&A until GET /templates answers (an older server has Q&A only)
  const [tpl, setTpl] = useState({ templates: [QA_TEMPLATE], inputs: T_INPUTS, outputs: T_OUTPUTS, formats: T_FORMATS });
  const loadTemplates = useCallback(async () => {
    try { const r = await api("/templates"); if (r?.templates?.length) setTpl(r); } catch { /* older server: Q&A only */ }
  }, []);
  useEffect(() => { loadTemplates(); }, [loadTemplates]);
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
  // the soft card behind the active step and the progress line are measured from the real step buttons
  const [stepperRef, ind] = useStepper();
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
    const m = e.message;
    const key = /endpoint|vision/i.test(m) ? "models" : /split/i.test(m) ? "size" : /question types|template|input|output|label|json|variety|checks/i.test(m) ? "dataset"
      : /decline|paraphrase|general/i.test(m) ? "mix" : "advanced";
    issues.push({ step: STEPS.findIndex((x) => x.key === key), text: m });
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

  const stepProps = { form, set, setIn, setForm, files, setFiles, defaults, ep, models, testEndpoints, onReset: resetForm, tpl, reloadTemplates: loadTemplates, goStep: setStep };
  const cur = STEPS[step];
  const status = stepStatus(form, files, folders, ep);
  return (
    <div className="gen">
      <nav className="stepper" aria-label="Setup steps" ref={stepperRef} style={{ "--p": step / (STEPS.length - 1) }}>
        <span className="stp-progress" aria-hidden="true"><i style={ind ? { width: ind.end } : undefined} /></span>
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
            {cur.key === "dataset" && <DatasetTypeStep {...stepProps} />}
            {cur.key === "size" && <SizeStep {...stepProps} />}
            {cur.key === "mix" && <MixStep {...stepProps} />}
            {cur.key === "advanced" && <AdvancedStep {...stepProps} />}
          </div>
          <footer className="gen-ft">
            <button type="button" className="btn ghost" disabled={step === 0} onClick={() => setStep(step - 1)}><Icon name="back" size={15} />Back</button>
            <div className="grow" />
            {step < STEPS.length - 1
              ? <button type="button" className="btn primary next" onClick={() => setStep(step + 1)}>Next: {STEPS[step + 1].title}<Icon name="arrow" size={15} /></button>
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
    dataset: form.template?.input === "question" && !form.types.length ? { warn: true, note: "Pick a kind of question" }
      : { ok: true, note: isQa(form.template) ? `Q&A · ${form.types.length} kind${form.types.length > 1 ? "s" : ""}` : form.template.name },
    size: split !== 100 ? { warn: true, note: "Split must total 100" }
      : { ok: true, note: `Up to ${Number(form.maxPairs || 0).toLocaleString()} ${isQa(form.template) ? "pairs" : "rows"}` },
    mix: { ok: true, note: extras ? `${extras} extra${extras > 1 ? "s" : ""} on` : "No extras" },
    advanced: { ok: false, note: "Optional" },
  };
}



















//ResultsPage.jsx
// Results page as a guided, step-by-step flow (like Generate): pick a dataset, see the overview,
// review the pairs, look at the insights if you like, then accept and download.
import { useCallback, useEffect, useLayoutEffect, useRef, useState } from "react";
import { useStepper } from "../stepper.js";
import { api, DEMO, fileUrl, post } from "./source.js";
import { emit } from "../bus.js";
import { fmtN } from "../model.js";
import { Empty, Icon, useBusy, useToast } from "../ui.jsx";
import { Bars, Funnel, ImagePath, ScoreStacks, SplitTable, StackBar, TypeDonut, useCount } from "./Insights.jsx";
import Pair from "./Pair.jsx";
import SourceDrawer from "./SourceDrawer.jsx";
import "./results.css";

const STEPS = [
  { key: "pick", title: "Choose dataset", icon: "database", lead: "Pick the generated dataset you want to look at. Only finished jobs have results." },
  { key: "overview", title: "Overview", icon: "chart", lead: "A quick look at what was generated, and anything to check before training." },
  { key: "review", title: "Review pairs", icon: "list", lead: "Read the question-and-answer pairs. Approve the good ones, reject the bad ones, or edit them. You can skip this step." },
  { key: "insights", title: "Insights", icon: "layers", lead: "Optional. Charts that show where the questions came from, how they scored, and why some were removed." },
  { key: "finish", title: "Accept & download", icon: "download", lead: "Write the final dataset with your reviews applied, and download the files for training." },
];
const SHOW = [["kept", "Kept"], ["review", "To review"], ["dropped", "Dropped"], ["all", "All"]];
const MORE = [["duplicate", "Duplicates"], ["image", "From images"]];
const TYPES = [["", "All types"], ["factual", "Factual"], ["explanation", "Explanation"], ["how-to", "How-to"], ["from_images", "From images"],
  ["comparison", "Comparison"], ["decline", "Decline"], ["general", "General knowledge"]];
const LIMIT = 10;
const KEY = "slm-results-job";
const STATUS = { done: ["Finished", "ok"], running: ["Still running", "run"], queued: ["Queued", "run"], failed: ["Failed", "bad"], cancelled: ["Cancelled", "muted"] };
const firstJob = () => new URL(location.href).searchParams.get("job") || localStorage.getItem(KEY);
const when = (iso) => { try { return new Date(iso).toLocaleString(undefined, { day: "numeric", month: "short", hour: "2-digit", minute: "2-digit" }); } catch { return ""; } };

export default function ResultsPage() {
  const toast = useToast();
  const [jobs, setJobs] = useState(null);
  const [jobId, setJobId] = useState(firstJob);
  const [job, setJob] = useState(null);
  const [report, setReport] = useState(null);
  const [problem, setProblem] = useState(null); // { kind: "running" | "failed", text }
  const [step, setStep] = useState(0);
  const [counts, setCounts] = useState(null);
  const [accepted, setAccepted] = useState(null);
  const [drawer, setDrawer] = useState(null);

  // direction of travel, so each step slides in from the right side
  const prev = useRef(0);
  const dir = step >= prev.current ? "fwd" : "back";
  useEffect(() => { prev.current = step; }, [step]);

  useEffect(() => {
    api("/jobs").then((list) => {
      setJobs(list);
      setJobId((id) => (id && list.some((j) => j.id === id) ? id : (list.find((j) => j.status === "done") || list[0])?.id || null));
    }).catch((e) => { setJobs([]); toast(`API not reachable: ${e.message}`, "error"); });
  }, [toast]);

  useEffect(() => {
    setJob(null); setReport(null); setProblem(null); setCounts(null); setAccepted(null);
    if (!jobId) return;
    localStorage.setItem(KEY, jobId);
    api(`/jobs/${jobId}`).then(setJob).catch(() => {});
    api(`/jobs/${jobId}/report`).then(setReport)
      .catch((e) => setProblem(e.status === 409 ? { kind: "running", text: "This job is still running. Its results appear here as soon as it finishes." } : { kind: "failed", text: e.message }));
  }, [jobId]);
  // counts of the review tabs, loaded once so the steps can show progress
  useEffect(() => {
    if (!report) return;
    api(`/jobs/${jobId}/rows?status=kept&limit=1`).then((x) => setCounts((c) => c || x.counts)).catch(() => {});
  }, [report, jobId]);
  useEffect(() => { emit("results-job", jobId ? { id: jobId, name: job?.name || job?.request?.name } : null); }, [jobId, job]);

  // the soft card behind the active step and the progress line are measured from the real step buttons
  const [stepperRef, ind] = useStepper();

  const openGenerate = (id) => { const u = new URL(location.href); id ? u.searchParams.set("job", id) : u.searchParams.delete("job"); u.hash = "#/generate"; location.href = u.toString(); };
  const onCounts = useCallback((c) => setCounts(c), []);
  const ready = !!report;
  const cur = STEPS[step];
  const reviewed = counts ? (counts.kept ?? 0) - (counts.review ?? 0) : 0;
  const notes = {
    pick: jobId ? { ok: ready, note: job?.name || jobs?.find((j) => j.id === jobId)?.name || jobId } : { note: "Pick a dataset" },
    overview: ready ? { ok: step > 1, note: `${fmtN(report.kept)} rows` } : { note: "—" },
    review: ready ? { ok: step > 2, note: counts ? `${fmtN(reviewed)} of ${fmtN(counts.kept)} reviewed` : "Optional" } : { note: "—" },
    insights: ready ? { ok: step > 3, note: "Optional" } : { note: "—" },
    finish: ready ? { ok: !!accepted, note: accepted ? "Accepted" : "Not yet" } : { note: "—" },
  };

  if (jobs && jobs.length === 0) {
    return <section className="card rs-empty"><Empty icon="results" title="No datasets yet">Generate your first dataset and its results show up here.
      <a className="btn primary" href="#/generate"><Icon name="plus" size={14} />New dataset</a></Empty></section>;
  }

  return (
    <div className="gen rs">
      <nav className="stepper" aria-label="Results steps" ref={stepperRef} style={{ "--p": step / (STEPS.length - 1) }}>
        <span className="stp-progress" aria-hidden="true"><i style={ind ? { width: ind.end } : undefined} /></span>
        {ind && <span className="stp-ind" aria-hidden="true" style={{ transform: `translate(${ind.left}px, ${ind.top}px)`, width: ind.width, height: ind.height }} />}
        {STEPS.map((s, i) => {
          const st = notes[s.key];
          return (
            <button type="button" key={s.key} className={`stp${i === step ? " on" : ""}${st.ok ? " ok" : ""}`} disabled={i > 0 && !ready} onClick={() => setStep(i)}>
              <span className="stp-n">{st.ok && i !== step ? <Icon name="check" size={14} stroke={2.6} className="pop" /> : <span className="pop">{i + 1}</span>}</span>
              <span className="stp-t"><b>{s.title}</b><small>{st.note}</small></span>
            </button>
          );
        })}
      </nav>

      <section className="card gen-main rs-main">
        <header className={`gen-hd anim-${dir}`} key={`hd${step}`}>
          <span className="gen-hd-ico"><Icon name={cur.icon} size={20} /></span>
          <div className="gen-hd-t">
            <div className="gen-hd-meta">
              <span className="step-n">Step {step + 1} of {STEPS.length}</span>
              <span className="step-dots" aria-hidden="true">{STEPS.map((s2, i) => <i key={s2.key} className={i < step ? "done" : i === step ? "on" : ""} />)}</span>
              {DEMO && <span className="rs-demo" title="Dummy data: set DEMO to false in src/results/source.js to use the real API">Demo data</span>}
            </div>
            <h2>{cur.title}</h2><p>{cur.lead}</p>
          </div>
        </header>

        <div className={`gen-bd anim-${dir}`} key={`bd${step}`}>
          {cur.key === "pick" && <PickStep jobs={jobs} jobId={jobId} onPick={setJobId} problem={problem} job={job} openGenerate={openGenerate} />}
          {cur.key === "overview" && ready && <OverviewStep r={report} onSource={() => setDrawer("documents")} />}
          {cur.key === "review" && ready && <Review jobId={jobId} job={job} onCounts={onCounts} />}
          {cur.key === "insights" && ready && <Insights r={report} />}
          {cur.key === "finish" && ready && <FinishStep jobId={jobId} job={job} r={report} counts={counts} accepted={accepted} setAccepted={setAccepted} goReview={() => setStep(2)} />}
        </div>

        <footer className="gen-ft">
          <button type="button" className="btn ghost" disabled={step === 0} onClick={() => setStep(step - 1)}><Icon name="back" size={15} />Back</button>
          <div className="grow" />
          {step === 2 && <button type="button" className="btn ghost" onClick={() => setStep(4)}>Skip to accept</button>}
          {step < STEPS.length - 1
            ? <button type="button" className="btn primary next" disabled={!ready} onClick={() => setStep(step + 1)}>Next: {STEPS[step + 1].title}<Icon name="arrow" size={15} /></button>
            : <a className="btn ghost" href="#/generate"><Icon name="plus" size={14} />Generate another dataset</a>}
        </footer>
      </section>
      <SourceDrawer jobId={jobId} job={job} tab={drawer} setTab={setDrawer} onClose={() => setDrawer(null)} />
    </div>
  );
}

// ------------------------------------------------------------------ step 1: choose a dataset
function PickStep({ jobs, jobId, onPick, problem, job, openGenerate }) {
  if (!jobs) return <div className="rs-skel rows"><i /><i /></div>;
  return (
    <div className="rs-pickstep">
      <div className="rs-jobs" role="radiogroup" aria-label="Datasets">
        {jobs.map((j) => {
          const [l, tone] = STATUS[j.status] || [j.status, "muted"];
          return (
            <button type="button" role="radio" aria-checked={j.id === jobId} key={j.id} className={`rs-job${j.id === jobId ? " on" : ""}`} onClick={() => onPick(j.id)}>
              <span className="rs-radio" aria-hidden="true" />
              <span className="rs-job-t"><b>{j.name || j.id}</b><small className="mono">{j.id}{j.created ? ` · ${when(j.created)}` : ""}</small></span>
              <span className={`rs-status ${tone}`}><span className={`rs-dot ${tone}`} />{l}</span>
            </button>
          );
        })}
      </div>
      {problem && (
        <div className={`rs-callout ${problem.kind}`}>
          <Icon name={problem.kind === "running" ? "clock" : "alert"} size={16} />
          <div><b>{problem.kind === "running" ? "Not finished yet" : "This job has no results"}</b><p>{job?.error || problem.text}</p></div>
          <button type="button" className="btn small" onClick={() => openGenerate(jobId)}>{problem.kind === "running" ? "Follow the progress" : "Open it on Generate"}</button>
        </div>
      )}
    </div>
  );
}

// ------------------------------------------------------------------ step 2: the key numbers and what to check
function OverviewStep({ r, onSource }) {
  const f = r.funnel || {}, sp = r.splits || {};
  const removed = Object.values(r.drop_reasons || {}).reduce((a, b) => a + b, 0);
  const kept = useCount(r.kept);
  const warns = r.warnings || [];
  return (
    <>
      <section className="rs-sum">
        <div><span>Rows in the dataset</span><b className="num">{fmtN(Math.round(kept))}</b><small>from {fmtN(f.questions_generated)} questions</small></div>
        <div><span>Average grounded score</span><b className="num">{r.avg_grounded ?? "—"}<em>/ 5</em></b><small>how well answers match the source</small></div>
        <div><span>Train · validation · test</span><b className="num">{fmtN(sp.train || 0)}<em> · {fmtN(sp.val || 0)} · {fmtN(sp.test || 0)}</em></b><small>split by {r.split_by || "section"}</small></div>
        <div><span>Removed</span><b className="num">{fmtN(removed)}</b><small>duplicates and low scores</small></div>
      </section>
      {warns.length > 0 ? (
        <div className="rs-callout warn"><Icon name="alert" size={16} />
          <div><b>Check before training</b><ul>{warns.map((w) => <li key={w}>{w}</li>)}</ul></div></div>
      ) : (
        <div className="rs-callout ok"><Icon name="check" size={16} /><div><b>Nothing to worry about</b><p>No warnings for this dataset.</p></div></div>
      )}
      <div className="rs-hint">
        <Icon name="book" size={16} />
        <span>Want to see what the questions were made from? Open the parsed documents, images and knowledge base.</span>
        <button type="button" className="btn small" onClick={onSource}>Open source data</button>
      </div>
    </>
  );
}

// ------------------------------------------------------------------ step 5: accept and download
function FinishStep({ jobId, job, r, counts, accepted, setAccepted, goReview }) {
  const toast = useToast();
  const [busy, run] = useBusy();
  const [all, setAll] = useState(null);
  useEffect(() => { api(`/jobs/${jobId}/rows?status=kept&limit=10000`).then((x) => setAll(x.rows)).catch(() => {}); }, [jobId]);
  const n = (f) => (all ? all.filter(f).length : 0);
  const approved = n((x) => x.review?.action === "approve"), rejected = n((x) => x.review?.action === "reject"), edited = n((x) => x.review?.edited);
  const open = all ? all.length - approved - rejected : counts?.review ?? 0;
  const final = (r.kept ?? 0) - rejected;
  const accept = () => run(async () => {
    try { setAccepted(await post(`/jobs/${jobId}/accept`)); toast("Dataset written"); } catch (e) { toast(e.message, "error"); }
  });
  const files = [["train.jsonl", "Train set", "for training the model"], ["val.jsonl", "Validation set", "for checking during training"], ["test.jsonl", "Test set", "for the final evaluation"],
    ["quality_report.json", "Quality report", "all the numbers from this page"], ...(accepted ? [["all.jsonl", "Everything", "all rows in one file"]] : [])];
  return (
    <>
      <div className="rs-fin">
        <div className="rs-fin-n"><b className="num">{fmtN(final)}</b><span>rows will be written</span></div>
        <ul className="rs-fin-l">
          <li><i className="ok" /><b className="num">{fmtN(approved)}</b> approved{edited ? ` (${fmtN(edited)} edited)` : ""}</li>
          <li><i className="bad" /><b className="num">{fmtN(rejected)}</b> rejected, left out</li>
          <li><i /><b className="num">{fmtN(open)}</b> not reviewed, kept as they are</li>
        </ul>
        {open > 0 && <button type="button" className="btn small ghost" onClick={goReview}><Icon name="back" size={13} />Back to review</button>}
      </div>

      <div className={`rs-accept${accepted ? " done" : ""}`}>
        <span className="rs-num">1</span>
        <div className="grow"><b>{accepted ? "Dataset written" : "Accept the dataset"}</b>
          <p>{accepted ? <>Wrote {fmtN(accepted.rows)} rows to <span className="mono">{accepted.path}</span></> : "Saves the final files on the server, with your reviews applied."}</p></div>
        <button type="button" className={`btn ${accepted ? "" : "primary"}`} disabled={busy || job?.status !== "done"} onClick={accept}>
          <Icon name={accepted ? "refresh" : "check"} size={15} />{busy ? "Writing…" : accepted ? "Accept again" : "Accept dataset"}</button>
      </div>

      <div className="rs-dl">
        <div className="rs-dl-h"><span className="rs-num">2</span><div><b>Download the files</b><p>Use these to fine-tune your model.</p></div></div>
        <div className="rs-files">
          {files.map(([fn, l, d]) => (
            <a key={fn} className="rs-file" href={fileUrl(jobId, fn)} download={fn}>
              <span className="rs-file-ico"><Icon name="doc" size={16} /></span>
              <span className="rs-file-t"><b>{l}</b><small>{d}</small></span>
              <span className="mono rs-file-n">{fn}</span>
              <Icon name="download" size={15} className="rs-file-dl" />
            </a>
          ))}
        </div>
      </div>
    </>
  );
}

function Insights({ r }) {
  const f = r.funnel || {}, im = r.image_funnel || {};
  const mix = r.mix || { documents: r.kept - (r.paraphrases || 0) - (r.declines || 0), paraphrases: r.paraphrases || 0, decline: r.declines || 0, general: 0 };
  return (
    <div className="rs-ins">
      <Panel title="Where the questions went" d={0}>
        <Funnel steps={[["Generated", f.questions_generated], ["After dedupe", f.after_question_dedupe], ["Answered", f.answered], ["Passed the judge", f.passed_judge], ["In the dataset", f.kept, true]]} />
      </Panel>
      <Panel title="Judge scores of kept pairs" d={1}><ScoreStacks scores={r.scores_of_kept || {}} /></Panel>
      <Panel title="What the dataset is made of" d={2}>
        <StackBar parts={[["From documents", mix.documents, "var(--mix-1)"], ["Paraphrases", mix.paraphrases, "var(--mix-2)"], ["Declines", mix.decline, "var(--mix-3)"], ["General", mix.general, "var(--mix-4)"]]} />
      </Panel>
      <Panel title="Question types" d={3}><TypeDonut types={r.types || {}} /></Panel>
      <Panel title="Why pairs were removed" d={4}>
        <Bars empty="Nothing was removed." rows={Object.entries(r.drop_reasons || {}).sort((a, b) => b[1] - a[1]).map(([k, v]) => [k, v, /duplicate/.test(k) ? "muted" : "accent"])} />
      </Panel>
      <Panel title="Images to Q&A pairs" right={im.read_images === false ? "images ignored" : im.vision_analysis ? "vision model" : "OCR only"} d={5}><ImagePath im={im} /></Panel>
      <Panel title="Split per section" right={r.split_by ? `by ${r.split_by}` : ""} d={6} wide><SplitTable cov={r.split_coverage || {}} /></Panel>
    </div>
  );
}

function Panel({ title, right, d = 0, wide, children }) {
  return (
    <section className={`rs-panel${wide ? " wide" : ""}`} style={{ "--d": d }}>
      <header><h3>{title}</h3><div className="grow" />{right && <span className="rs-note-r">{right}</span>}</header>
      {children}
    </section>
  );
}

// ------------------------------------------------------------------ review list
function Review({ jobId, job, onCounts }) {
  const toast = useToast();
  const [filter, setFilter] = useState({ status: "kept", type: "", q: "", offset: 0 });
  const [rows, setRows] = useState(null);
  const [loading, setLoading] = useState(true);

  const load = useCallback(() => {
    const qs = new URLSearchParams({ status: filter.status, offset: filter.offset, limit: LIMIT });
    if (filter.type) qs.set("type", filter.type);
    if (filter.q) qs.set("q", filter.q);
    setLoading(true);
    api(`/jobs/${jobId}/rows?${qs}`).then(setRows).catch((e) => { if (e.status !== 409) toast(e.message, "error"); }).finally(() => setLoading(false));
  }, [jobId, filter, toast]);
  useEffect(() => { const t = setTimeout(load, filter.q ? 300 : 0); return () => clearTimeout(t); }, [load, filter.q]);

  useEffect(() => { if (rows?.counts) onCounts?.(rows.counts); }, [rows, onCounts]);
  const setF = (k, v) => setFilter((x) => ({ ...x, [k]: v, offset: k === "offset" ? v : 0 }));
  const c = rows?.counts || {};
  const more = MORE.find(([k]) => k === filter.status);
  return (
    <section className="rs-review">
      <div className="rs-filters">
        <div className="rs-seg" role="tablist">
          {SHOW.map(([k, l]) => (
            <button type="button" role="tab" aria-selected={filter.status === k} key={k} className={filter.status === k ? "on" : ""} onClick={() => setF("status", k)}>
              {l}<span className="num">{fmtN(c[k] ?? 0)}</span></button>
          ))}
          {more && <button type="button" className="on" role="tab" aria-selected="true">{more[1]}<span className="num">{fmtN(c[more[0]] ?? 0)}</span></button>}
        </div>
        <div className="grow" />
        <label className="search"><Icon name="search" size={14} /><input placeholder="Search" value={filter.q} onChange={(e) => setF("q", e.target.value)} /></label>
        <label className="rs-select" title="Filter">
          <select value={filter.type || (more ? `@${more[0]}` : "")} aria-label="Filter"
            onChange={(e) => { const v = e.target.value; if (v.startsWith("@")) setFilter((x) => ({ ...x, status: v.slice(1), type: "", offset: 0 })); else setF("type", v); }}>
            <optgroup label="Question type">{TYPES.map(([v, l]) => <option key={v} value={v}>{l}</option>)}</optgroup>
            <optgroup label="Show only">{MORE.map(([k, l]) => <option key={k} value={`@${k}`}>{l} ({fmtN(c[k] ?? 0)})</option>)}</optgroup>
          </select>
        </label>
      </div>

      <div className={`rs-list${loading ? " is-loading" : ""}`} key={`${filter.status}|${filter.type}|${filter.offset}`}>
        {!rows ? <div className="rs-skel rows"><i /><i /><i /></div>
          : rows.rows.length === 0 ? <Empty icon="search" title="Nothing here">Try another filter or search.</Empty>
          : rows.rows.map((row, i) => (
            <Pair key={row.q_id} i={i} row={row} jobId={jobId} job={job}
              onChange={(review) => setRows((x) => {
                const was = !!row.review?.action, now = !!review?.action;
                const counts = { ...x.counts, review: (x.counts?.review ?? 0) + (was && !now ? 1 : !was && now ? -1 : 0) };
                return { ...x, counts, rows: x.rows.map((y) => (y.q_id === row.q_id ? { ...y, review } : y)) };
              })} />
          ))}
      </div>

      {rows && rows.total > LIMIT && (
        <footer className="rs-pager">
          <span className="muted small"><b className="num">{filter.offset + 1}–{Math.min(filter.offset + LIMIT, rows.total)}</b> of <b className="num">{fmtN(rows.total)}</b></span>
          <div className="grow" />
          <button type="button" className="btn small ghost" disabled={filter.offset === 0} onClick={() => setF("offset", Math.max(0, filter.offset - LIMIT))}><Icon name="back" size={13} />Previous</button>
          <button type="button" className="btn small ghost" disabled={filter.offset + LIMIT >= rows.total} onClick={() => setF("offset", filter.offset + LIMIT)}>Next<Icon name="arrow" size={13} /></button>
        </footer>
      )}
    </section>
  );
}























//rich.css
/* Rich layer: depth, texture and finish. Mostly neutral; coral only as a thin accent.
   Loaded last, so it refines styles.css, shell.css and generate.css without replacing them. */

:root {
  --lift-1: 0 1px 2px rgba(16, 24, 40, .05), 0 1px 1px rgba(16, 24, 40, .03);
  --lift-2: 0 1px 2px rgba(16, 24, 40, .04), 0 4px 10px -2px rgba(16, 24, 40, .06), 0 12px 28px -12px rgba(16, 24, 40, .10);
  --lift-3: 0 2px 4px rgba(16, 24, 40, .04), 0 12px 24px -6px rgba(16, 24, 40, .10), 0 28px 56px -24px rgba(16, 24, 40, .16);
  --grad-soft: linear-gradient(135deg, rgba(var(--grad-a-rgb), .10), rgba(var(--grad-b-rgb), .10));
}

/* ---- canvas: a faint dot grid with a soft warm glow in the top corner */
main.main { background:
  radial-gradient(900px 420px at 100% -10%, rgba(var(--grad-a-rgb), .055), transparent 60%),
  radial-gradient(700px 380px at 30% -20%, rgba(var(--grad-b-rgb), .035), transparent 60%),
  radial-gradient(var(--dot) 1px, transparent 1.2px) 0 0 / 22px 22px,
  var(--bg); }

/* ---- sidebar: soft paper with a light inner edge */
.sidebar { background:
  linear-gradient(180deg, rgba(var(--paper-rgb), .96), rgba(250, 250, 251, .96)),
  radial-gradient(var(--dot) 1px, transparent 1.2px) 0 0 / 18px 18px;
  border-right: 1px solid var(--hairline); box-shadow: inset -1px 0 0 rgba(var(--paper-rgb), .8), 4px 0 24px -18px rgba(16, 24, 40, .25); position: relative; z-index: 2; }
.sb-top { padding-bottom: 18px; border-bottom: 1px solid var(--line-2); margin: 0 4px; }
.sb-nav a { font-weight: 550; }
.sb-ind { box-shadow: var(--edge), 0 0 0 1px var(--hairline), 0 2px 4px rgba(16, 24, 40, .04), 0 8px 18px -10px rgba(16, 24, 40, .18); }
.sb-nav-ico { border: 1px solid transparent; }
.sb-nav a.on .sb-nav-ico { background: linear-gradient(180deg, var(--surface), var(--accent-50)); border-color: var(--accent-100);
  box-shadow: 0 2px 6px -2px rgba(var(--accent-rgb), .35); }

/* ---- header: frosted glass over the canvas, title with a fine gradient underline */
.ph { background: rgba(var(--paper-rgb), .72); backdrop-filter: saturate(1.6) blur(14px); -webkit-backdrop-filter: saturate(1.6) blur(14px);
  border-bottom: 1px solid var(--hairline); box-shadow: 0 1px 0 rgba(var(--paper-rgb), .7) inset, 0 6px 20px -16px rgba(16, 24, 40, .25); z-index: 1; }
.ph::after { background: linear-gradient(90deg, rgba(var(--grad-a-rgb), .55), rgba(var(--grad-b-rgb), .35) 22%, transparent 55%); }
.ph-ico { position: relative; background: linear-gradient(180deg, var(--surface), var(--surface-2)); border: 0; box-shadow: var(--edge), 0 0 0 1px var(--hairline), var(--lift-2); }
.ph-ico::before { content: ""; position: absolute; inset: -1px; border-radius: inherit; padding: 1px; background: linear-gradient(135deg, rgba(var(--grad-a-rgb), .55), rgba(var(--grad-b-rgb), .15) 60%, transparent);
  -webkit-mask: linear-gradient(#000 0 0) content-box, linear-gradient(#000 0 0); -webkit-mask-composite: xor; mask-composite: exclude; pointer-events: none; }
.ph h1 { font-weight: 700; letter-spacing: -.4px; }
.ph-state { background: rgba(var(--paper-rgb), .8); box-shadow: var(--edge), var(--lift-1); border-color: var(--hairline); }
.ph-side .btn.ghost { border: 1px solid transparent; }
.ph-side .btn.ghost:hover { background: rgba(var(--paper-rgb), .9); border-color: var(--hairline); box-shadow: var(--lift-1); }

/* ---- cards: layered elevation and a light top edge */
.card { border-color: var(--hairline); box-shadow: var(--edge), var(--lift-2); border-radius: 18px; }
.gen-main, .summary { box-shadow: var(--edge), var(--lift-3); }
.gen-ft { background: rgba(250, 250, 251, .85); backdrop-filter: blur(10px); -webkit-backdrop-filter: blur(10px); border-radius: 0 0 18px 18px; }

/* ---- stepper: elevated rail, a progress line and ringed numbers */
.stepper { border-color: var(--hairline); box-shadow: var(--edge), var(--lift-2); border-radius: 18px; background: rgba(var(--paper-rgb), .92); overflow: hidden; }
.stp-progress { position: absolute; left: 0; right: 0; bottom: 0; height: 3px; background: var(--line-2); }
.stp-progress i { display: block; height: 100%; width: calc(10% + var(--p, 0) * 90%); background: var(--grad); border-radius: 0 3px 3px 0;
  transition: width .5s cubic-bezier(.2, .8, .2, 1); }
.stp-ind { background: linear-gradient(180deg, var(--surface), var(--surface-2)); box-shadow: var(--edge), 0 0 0 1px var(--hairline), 0 6px 14px -8px rgba(16, 24, 40, .22); }
.stp-n { box-shadow: 0 0 0 4px var(--surface); }
.stp.on .stp-n { background: var(--grad); color: #fff; border-color: transparent; box-shadow: 0 0 0 4px rgba(var(--accent-rgb), .12), 0 4px 10px -3px rgba(var(--accent-rgb), .5); }
.stp.ok:not(.on) .stp-n { background: linear-gradient(180deg, var(--surface), var(--green-50)); border-color: var(--green-200); }

/* ---- step header: icon tile with a gradient ring, and step dots */
.gen-hd { background: linear-gradient(180deg, rgba(250, 250, 251, .9), rgba(var(--paper-rgb), 0)); padding-top: 24px; padding-bottom: 20px; }
.gen-hd-ico { position: relative; width: 52px; height: 52px; border-radius: 15px; border: 0; color: var(--accent-600);
  background: linear-gradient(160deg, var(--surface) 30%, var(--accent-50)); box-shadow: var(--edge), 0 0 0 1px var(--hairline), var(--lift-2); }
.gen-hd-ico::before { content: ""; position: absolute; inset: -1px; border-radius: inherit; padding: 1px; background: linear-gradient(135deg, rgba(var(--grad-a-rgb), .6), rgba(var(--grad-b-rgb), .2) 60%, transparent);
  -webkit-mask: linear-gradient(#000 0 0) content-box, linear-gradient(#000 0 0); -webkit-mask-composite: xor; mask-composite: exclude; }
.gen-hd-meta { display: flex; align-items: center; gap: 10px; }
.step-dots { display: inline-flex; gap: 4px; }
.step-dots i { width: 6px; height: 6px; border-radius: 99px; background: var(--line); transition: width .35s cubic-bezier(.2, .8, .2, 1), background .3s; }
.step-dots i.done { background: var(--accent-200); }
.step-dots i.on { width: 18px; background: var(--grad); }
.gen-hd h2 { font-size: 24px; letter-spacing: -.4px; }

/* ---- section headings: a short accent tick */
.g-sec-hd h3 { position: relative; padding-left: 12px; }
.g-sec-hd h3::before { content: ""; position: absolute; left: 0; top: 4px; bottom: 4px; width: 3px; border-radius: 3px; background: var(--grad); opacity: .85; }

/* ---- drop zone: a small illustration of stacked documents */
.dropzone { border: 1.5px dashed var(--line-strong); background:
  radial-gradient(400px 160px at 50% 0%, rgba(var(--grad-a-rgb), .05), transparent 70%),
  radial-gradient(var(--dot) 1px, transparent 1.2px) 0 0 / 16px 16px, var(--surface-2);
  padding: 30px 20px 26px; }
.dropzone:hover, .dropzone.drag { border-color: var(--accent-400); background:
  radial-gradient(420px 180px at 50% 0%, rgba(var(--grad-a-rgb), .09), transparent 70%),
  radial-gradient(var(--dot) 1px, transparent 1.2px) 0 0 / 16px 16px, var(--tint-0); }
.dz-art { position: relative; width: 120px; height: 82px; margin-bottom: 4px; }
.dz-sheet { position: absolute; top: 6px; width: 52px; height: 66px; border-radius: 8px; background: var(--surface); border: 1px solid var(--hairline);
  box-shadow: var(--lift-2); display: grid; align-content: start; gap: 5px; padding: 12px 9px; transition: transform .35s cubic-bezier(.34, 1.4, .64, 1); }
.dz-sheet i { display: block; height: 4px; border-radius: 3px; background: var(--line); }
.dz-sheet i:nth-child(2) { width: 75%; } .dz-sheet i:last-child { width: 55%; }
.dz-sheet b { font-size: 9px; font-weight: 800; letter-spacing: .4px; color: #fff; background: var(--grad); border-radius: 4px; padding: 1px 4px; justify-self: start; margin-bottom: 2px; }
.dz-sheet.s1 { left: 10px; transform: rotate(-10deg); }
.dz-sheet.s2 { right: 10px; transform: rotate(10deg); }
.dz-sheet.s3 { left: 34px; top: 0; z-index: 1; box-shadow: var(--lift-3); }
.dz-badge { position: absolute; z-index: 2; right: 22px; bottom: -2px; width: 30px; height: 30px; border-radius: 50%; display: grid; place-items: center; color: #fff;
  background: var(--grad); box-shadow: 0 0 0 3px var(--surface), 0 6px 14px -4px rgba(var(--accent-rgb), .6); transition: transform .3s cubic-bezier(.34, 1.4, .64, 1); }
.dropzone:hover .dz-sheet.s1, .dropzone.drag .dz-sheet.s1 { transform: rotate(-16deg) translate(-6px, 2px); }
.dropzone:hover .dz-sheet.s2, .dropzone.drag .dz-sheet.s2 { transform: rotate(16deg) translate(6px, 2px); }
.dropzone:hover .dz-sheet.s3, .dropzone.drag .dz-sheet.s3 { transform: translateY(-5px); }
.dropzone:hover .dz-badge, .dropzone.drag .dz-badge { transform: translateY(-3px) scale(1.06); }
.dz-types span { background: rgba(var(--paper-rgb), .9); box-shadow: var(--lift-1); border-color: var(--hairline); }
.fcard { box-shadow: var(--edge), var(--lift-1); border-color: var(--hairline); border-radius: 14px; }

/* ---- choice cards: a fine gradient border when chosen */
.choice { border-color: var(--hairline); box-shadow: var(--edge), var(--lift-1); border-radius: 16px; position: relative; }
.choice:hover { box-shadow: var(--edge), var(--lift-2); }
.choice.on { border-color: transparent; background: linear-gradient(180deg, var(--surface), var(--tint-0)); box-shadow: var(--edge), var(--lift-2), 0 0 0 3px rgba(var(--accent-rgb), .06); }
.choice.on::before { content: ""; position: absolute; inset: -1.5px; border-radius: 17px; padding: 1.5px; background: var(--grad);
  -webkit-mask: linear-gradient(#000 0 0) content-box, linear-gradient(#000 0 0); -webkit-mask-composite: xor; mask-composite: exclude; pointer-events: none; }
.choice-ico { border: 1px solid var(--hairline); background: linear-gradient(180deg, var(--surface), var(--surface-2)); box-shadow: var(--lift-1); }
.choice.on .choice-ico { background: linear-gradient(160deg, var(--surface) 20%, var(--accent-50)); border-color: var(--accent-100); }
.choice.on .choice-mark { background: var(--grad); box-shadow: 0 3px 8px -2px rgba(var(--accent-rgb), .5); }

/* ---- option panels, model cards, folds */
.opt, .epcard, .fold, .rule2 { border-color: var(--hairline); box-shadow: var(--edge), var(--lift-1); border-radius: 16px; }
.opt.on, .fold[open], .epcard:hover { box-shadow: var(--edge), var(--lift-2); }
.opt-ico, .fold summary .opt-ico { border: 1px solid var(--hairline); background: linear-gradient(180deg, var(--surface), var(--surface-2)); box-shadow: var(--lift-1); }
.opt.on .opt-ico { background: linear-gradient(160deg, var(--surface) 20%, var(--accent-50)); border-color: var(--accent-100); }
.opt-bd { background: linear-gradient(180deg, rgba(250, 250, 251, .7), transparent); border-top-style: solid; border-top-color: var(--line-2); }
.mixsum { background: linear-gradient(180deg, var(--surface), var(--surface-2)); border-color: var(--hairline); box-shadow: var(--edge), var(--lift-1); }
.mixbar { height: 10px; box-shadow: inset 0 1px 2px rgba(16, 24, 40, .08); }

/* ---- form controls: soft inset, crisp focus */
input:not([type=checkbox]):not([type=file]), select, textarea { border-color: var(--line-strong); box-shadow: inset 0 1px 2px rgba(16, 24, 40, .04); background-color: #fff; }
input:focus, select:focus, textarea:focus { border-color: var(--accent-400); box-shadow: 0 0 0 4px rgba(var(--accent-rgb), .12), inset 0 1px 2px rgba(16, 24, 40, .03); }
.seg { background: linear-gradient(180deg, var(--surface-3), var(--surface-2)); box-shadow: inset 0 1px 2px rgba(16, 24, 40, .06); border-color: var(--hairline); }
.seg button.on { box-shadow: 0 1px 2px rgba(16, 24, 40, .1), 0 2px 6px -2px rgba(16, 24, 40, .12), 0 0 0 1px rgba(16, 24, 40, .04); }
.switch { box-shadow: inset 0 1px 2px rgba(16, 24, 40, .15); }
.switch span { box-shadow: 0 1px 3px rgba(16, 24, 40, .25), 0 0 0 .5px rgba(16, 24, 40, .06); }
.chip { box-shadow: var(--lift-1); border-color: var(--hairline); }
.gen .chip.on { background: linear-gradient(180deg, var(--surface), var(--tint-0)); border-color: var(--accent-200); }

/* ---- buttons: depth on the main actions */
.btn { box-shadow: var(--edge), var(--lift-1); border-color: var(--line-strong); }
.btn.ghost { box-shadow: none; }
.btn.dark { background: linear-gradient(180deg, var(--dark-btn-a), var(--dark-btn-b)); border-color: var(--dark-btn-b); color: var(--dark-btn-ink); box-shadow: inset 0 1px 0 rgba(255, 255, 255, .12), 0 4px 10px -4px rgba(16, 24, 40, .5); }
.btn.primary { box-shadow: inset 0 1px 0 rgba(255, 255, 255, .25), 0 6px 16px -6px rgba(var(--accent-rgb), .6); }
.btn.primary.lg, .btn.primary.next { position: relative; overflow: hidden; }
.btn.primary.lg::after, .btn.primary.next::after { content: ""; position: absolute; top: 0; bottom: 0; left: -40%; width: 30%; transform: skewX(-20deg);
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, .35), transparent); transition: left .7s ease; }
.btn.primary.lg:hover::after, .btn.primary.next:hover:not(:disabled)::after { left: 120%; }

/* ---- summary: frosted header with a faint warm wash, tidy rows */
.summary { background: rgba(var(--paper-rgb), .94); }
.sum-hd { background: radial-gradient(320px 140px at 100% 0%, rgba(var(--grad-a-rgb), .08), transparent 70%),
  radial-gradient(260px 120px at 0% 0%, rgba(var(--grad-b-rgb), .05), transparent 70%), var(--surface); border-bottom-color: var(--line-2); }
.sum-ico { border: 1px solid var(--hairline); background: linear-gradient(180deg, var(--surface), var(--surface-2)); box-shadow: var(--lift-1); }
.sum-ft { background: linear-gradient(180deg, var(--surface-2), var(--surface-3)); }
.sum-ready { padding: 10px 12px; border-radius: 12px; background: linear-gradient(180deg, var(--green-50), var(--green-50)); border: 1px solid var(--green-200); }
.sum-issues { box-shadow: var(--edge), var(--lift-1); }

/* ---- run view */
.run-hero { background: radial-gradient(500px 200px at 100% 0%, rgba(var(--grad-a-rgb), .06), transparent 70%), var(--surface); }
.run-bar { height: 10px; box-shadow: inset 0 1px 2px rgba(16, 24, 40, .08); }
.tl-dot { box-shadow: 0 0 0 4px var(--surface); }
