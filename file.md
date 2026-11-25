//stepper.js
// Measures the active step of a stepper so the soft card behind it and the coral progress line
// always line up with the real button, on any screen size, zoom level, sidebar state or font.
import { useLayoutEffect, useRef, useState } from "react";

const same = (a, b) => a === b || (a && b && a.left === b.left && a.top === b.top && a.width === b.width && a.height === b.height && a.end === b.end);

export function useStepper() {
  const ref = useRef(null);
  const [ind, setInd] = useState(null);
  const placeRef = useRef(() => {});
  placeRef.current = () => {
    const nav = ref.current;
    const el = nav?.querySelector(".stp.on");
    if (!nav || !el) { setInd(null); return; }
    // positions relative to the stepper itself, so they hold under CSS zoom too
    const nb = nav.getBoundingClientRect(), eb = el.getBoundingClientRect();
    const k = nav.offsetWidth ? nb.width / nav.offsetWidth : 1; // rendered px per CSS px
    const left = (eb.left - nb.left) / k - nav.clientLeft, top = (eb.top - nb.top) / k - nav.clientTop;
    const width = eb.width / k, height = eb.height / k;
    const n = { left: Math.round(left * 10) / 10, top: Math.round(top * 10) / 10, width: Math.round(width * 10) / 10, height: Math.round(height * 10) / 10,
      end: Math.round((left + width) * 10) / 10 };
    setInd((c) => (same(c, n) ? c : n));
  };
  const roRef = useRef(null), seen = useRef(null);
  // re-measure after every render (step changed, labels changed), and watch the stepper once it is on screen
  useLayoutEffect(() => {
    placeRef.current();
    const nav = ref.current, ro = roRef.current;
    if (ro && nav !== seen.current) {
      ro.disconnect(); seen.current = nav;
      if (nav) { ro.observe(nav); nav.querySelectorAll(".stp").forEach((x) => ro.observe(x)); }
    }
  });
  // ... and whenever the layout moves: window resize, stepper or step size changes, fonts finishing loading
  useLayoutEffect(() => {
    const place = () => placeRef.current();
    let raf = 0;
    const soon = () => { cancelAnimationFrame(raf); raf = requestAnimationFrame(place); };
    window.addEventListener("resize", soon);
    const ro = typeof ResizeObserver === "function" ? new ResizeObserver(soon) : null;
    roRef.current = ro; seen.current = null;
    if (ro && ref.current) { seen.current = ref.current; ro.observe(ref.current); ref.current.querySelectorAll(".stp").forEach((x) => ro.observe(x)); }
    document.fonts?.ready?.then(soon);
    document.fonts?.addEventListener?.("loadingdone", soon);
    // the sidebar animates its width; measure again once that settles
    const onEnd = (e) => { if (e.propertyName === "grid-template-columns" || e.propertyName === "width") soon(); };
    document.addEventListener("transitionend", onEnd);
    return () => {
      cancelAnimationFrame(raf); window.removeEventListener("resize", soon); ro?.disconnect(); roRef.current = null;
      document.fonts?.removeEventListener?.("loadingdone", soon); document.removeEventListener("transitionend", onEnd);
    };
  }, []);
  return [ref, ind];
}



















//GeneratePage.jsx
// Generate Dataset page: a guided, step-by-step setup with a live summary, and a progress view once a job runs.
// Same API as the original UI (/api/datagen); the form <-> JobRequest logic is in ../model.js.
import { useCallback, useEffect, useLayoutEffect, useRef, useState } from "react";
import { useStepper } from "../stepper.js";
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
            ? <button type="button" className="btn dark" disabled={!ready} onClick={() => setStep(step + 1)}>Next: {STEPS[step + 1].title}<Icon name="arrow" size={15} /></button>
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
