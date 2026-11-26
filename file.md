//environment.js
// Environment settings of the UI. Change the API server here.
// Plain constants only: vite.config.js imports this file too, so keep it free of browser or Vite-only code.

// Address of the data generation API server (uvicorn app.main:app).
export const API_HOST = "http://107.99.131.90:8080";

// Path prefix of the data generation API on that server.
export const API_PATH = "/api/datagen";

// How the browser reaches API_HOST:
//   false (default): calls go to /api/datagen on the page's own origin. `npm run dev` / `npm run preview`
//                    forward them to API_HOST, and a build served by the API server itself is same-origin.
//                    No CORS setup is needed on the server.
//   true:            calls go straight to API_HOST from the browser. Use it when the built UI is hosted
//                    somewhere else; the API server must then allow this origin (CORS).
export const CALL_API_HOST_DIRECTLY = false;

// Results page data: false = the real API above; true = built-in dummy data (to try the screens without a server).
export const USE_DEMO_RESULTS = false;









//source.js
// Where the Results page gets its data from: the real /api/datagen backend (API_HOST in ../environment.js),
// or the dummy data in demoData.js when USE_DEMO_RESULTS is true there (handy without a server).
import { USE_DEMO_RESULTS } from "../environment.js";
import { api as realApi, fileUrl as realFileUrl, imageUrl as realImageUrl, post as realPost } from "../api.js";
import { demoApi, demoFileUrl, demoImageUrl, demoPost } from "./demoData.js";

export const DEMO = USE_DEMO_RESULTS;

export const api = DEMO ? demoApi : realApi;
export const post = DEMO ? demoPost : realPost;
export const fileUrl = DEMO ? demoFileUrl : realFileUrl;
export const imageUrl = DEMO ? demoImageUrl : realImageUrl;












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
    // the job first: the API answers 409 "not ready" for a running job and for a failed one alike, so its status tells them apart
    let live = true;
    api(`/jobs/${jobId}`).then((j) => { if (live) setJob(j); return j; }).catch(() => null)
      .then((j) => api(`/jobs/${jobId}/report`).then((r) => { if (live) setReport(r); }).catch((e) => {
        if (!live) return;
        const st = j?.status;
        const running = st ? st === "queued" || st === "running" : e.status === 409;
        setProblem(running ? { kind: "running", text: "This job is still running. Its results appear here as soon as it finishes." }
          : { kind: "failed", text: st === "cancelled" ? "This job was cancelled before it produced a dataset." : st === "failed" ? "This job failed before it produced a dataset." : e.message });
      }));
    return () => { live = false; };
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
              {DEMO && <span className="rs-demo" title="Dummy data: set USE_DEMO_RESULTS to false in src/environment.js to use the real API">Demo data</span>}
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

// every kept row, fetched in pages: the API returns at most 200 rows per call
async function allRows(jobId, status = "kept") {
  const PAGE = 200, out = [];
  for (let offset = 0; ; offset += PAGE) {
    const x = await api(`/jobs/${jobId}/rows?status=${status}&offset=${offset}&limit=${PAGE}`);
    out.push(...(x.rows || []));
    if (!x.rows?.length || out.length >= (x.total ?? 0)) return out;
  }
}

// ------------------------------------------------------------------ step 5: accept and download
function FinishStep({ jobId, job, r, counts, accepted, setAccepted, goReview }) {
  const toast = useToast();
  const [busy, run] = useBusy();
  const [all, setAll] = useState(null);
  useEffect(() => { let live = true; allRows(jobId).then((x) => live && setAll(x)).catch(() => {}); return () => { live = false; }; }, [jobId]);
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
