//rich.css
/* Rich layer: depth, texture and finish. Mostly neutral; coral only as a thin accent.
   Loaded last, so it refines styles.css, shell.css and generate.css without replacing them. */

:root {
  --edge: inset 0 1px 0 rgba(255, 255, 255, .9);
  --lift-1: 0 1px 2px rgba(16, 24, 40, .05), 0 1px 1px rgba(16, 24, 40, .03);
  --lift-2: 0 1px 2px rgba(16, 24, 40, .04), 0 4px 10px -2px rgba(16, 24, 40, .06), 0 12px 28px -12px rgba(16, 24, 40, .10);
  --lift-3: 0 2px 4px rgba(16, 24, 40, .04), 0 12px 24px -6px rgba(16, 24, 40, .10), 0 28px 56px -24px rgba(16, 24, 40, .16);
  --hairline: rgba(16, 24, 40, .08);
  --grad-soft: linear-gradient(135deg, rgba(249, 115, 22, .10), rgba(236, 72, 153, .10));
}

/* ---- canvas: a faint dot grid with a soft warm glow in the top corner */
main.main { background:
  radial-gradient(900px 420px at 100% -10%, rgba(249, 115, 22, .055), transparent 60%),
  radial-gradient(700px 380px at 30% -20%, rgba(236, 72, 153, .035), transparent 60%),
  radial-gradient(rgba(16, 24, 40, .055) 1px, transparent 1.2px) 0 0 / 22px 22px,
  var(--bg); }

/* ---- sidebar: soft paper with a light inner edge */
.sidebar { background:
  linear-gradient(180deg, rgba(255, 255, 255, .96), rgba(250, 250, 251, .96)),
  radial-gradient(rgba(16, 24, 40, .05) 1px, transparent 1.2px) 0 0 / 18px 18px;
  border-right: 1px solid var(--hairline); box-shadow: inset -1px 0 0 rgba(255, 255, 255, .8), 4px 0 24px -18px rgba(16, 24, 40, .25); position: relative; z-index: 2; }
.sb-top { padding-bottom: 18px; border-bottom: 1px solid var(--line-2); margin: 0 4px; }
.sb-nav a { font-weight: 550; }
.sb-ind { box-shadow: var(--edge), 0 0 0 1px var(--hairline), 0 2px 4px rgba(16, 24, 40, .04), 0 8px 18px -10px rgba(16, 24, 40, .18); }
.sb-nav-ico { border: 1px solid transparent; }
.sb-nav a.on .sb-nav-ico { background: linear-gradient(180deg, #fff, var(--accent-50)); border-color: var(--accent-100);
  box-shadow: 0 2px 6px -2px rgba(var(--accent-rgb), .35); }

/* ---- header: frosted glass over the canvas, title with a fine gradient underline */
.ph { background: rgba(255, 255, 255, .72); backdrop-filter: saturate(1.6) blur(14px); -webkit-backdrop-filter: saturate(1.6) blur(14px);
  border-bottom: 1px solid var(--hairline); box-shadow: 0 1px 0 rgba(255, 255, 255, .7) inset, 0 6px 20px -16px rgba(16, 24, 40, .25); z-index: 1; }
.ph::after { background: linear-gradient(90deg, rgba(249, 115, 22, .55), rgba(236, 72, 153, .35) 22%, transparent 55%); }
.ph-ico { position: relative; background: linear-gradient(180deg, #fff, #fbfbfc); border: 0; box-shadow: var(--edge), 0 0 0 1px var(--hairline), var(--lift-2); }
.ph-ico::before { content: ""; position: absolute; inset: -1px; border-radius: inherit; padding: 1px; background: linear-gradient(135deg, rgba(249, 115, 22, .55), rgba(236, 72, 153, .15) 60%, transparent);
  -webkit-mask: linear-gradient(#000 0 0) content-box, linear-gradient(#000 0 0); -webkit-mask-composite: xor; mask-composite: exclude; pointer-events: none; }
.ph h1 { font-weight: 700; letter-spacing: -.4px; }
.ph-state { background: rgba(255, 255, 255, .8); box-shadow: var(--edge), var(--lift-1); border-color: var(--hairline); }
.ph-side .btn.ghost { border: 1px solid transparent; }
.ph-side .btn.ghost:hover { background: rgba(255, 255, 255, .9); border-color: var(--hairline); box-shadow: var(--lift-1); }

/* ---- cards: layered elevation and a light top edge */
.card { border-color: var(--hairline); box-shadow: var(--edge), var(--lift-2); border-radius: 18px; }
.gen-main, .summary { box-shadow: var(--edge), var(--lift-3); }
.gen-ft { background: rgba(250, 250, 251, .85); backdrop-filter: blur(10px); -webkit-backdrop-filter: blur(10px); border-radius: 0 0 18px 18px; }

/* ---- stepper: elevated rail, a progress line and ringed numbers */
.stepper { border-color: var(--hairline); box-shadow: var(--edge), var(--lift-2); border-radius: 18px; background: rgba(255, 255, 255, .92); overflow: hidden; }
.stp-progress { position: absolute; left: 0; right: 0; bottom: 0; height: 3px; background: var(--line-2); }
.stp-progress i { display: block; height: 100%; width: calc(10% + var(--p, 0) * 90%); background: var(--grad); border-radius: 0 3px 3px 0;
  transition: width .5s cubic-bezier(.2, .8, .2, 1); }
.stp-ind { background: linear-gradient(180deg, #fff, var(--surface-2)); box-shadow: var(--edge), 0 0 0 1px var(--hairline), 0 6px 14px -8px rgba(16, 24, 40, .22); }
.stp-n { box-shadow: 0 0 0 4px #fff; }
.stp.on .stp-n { background: var(--grad); color: #fff; border-color: transparent; box-shadow: 0 0 0 4px rgba(var(--accent-rgb), .12), 0 4px 10px -3px rgba(var(--accent-rgb), .5); }
.stp.ok:not(.on) .stp-n { background: linear-gradient(180deg, #fff, var(--green-50)); border-color: #bfe8d4; }

/* ---- step header: icon tile with a gradient ring, and step dots */
.gen-hd { background: linear-gradient(180deg, rgba(250, 250, 251, .9), rgba(255, 255, 255, 0)); padding-top: 24px; padding-bottom: 20px; }
.gen-hd-ico { position: relative; width: 52px; height: 52px; border-radius: 15px; border: 0; color: var(--accent-600);
  background: linear-gradient(160deg, #fff 30%, var(--accent-50)); box-shadow: var(--edge), 0 0 0 1px var(--hairline), var(--lift-2); }
.gen-hd-ico::before { content: ""; position: absolute; inset: -1px; border-radius: inherit; padding: 1px; background: linear-gradient(135deg, rgba(249, 115, 22, .6), rgba(236, 72, 153, .2) 60%, transparent);
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
.dropzone { border: 1.5px dashed #d6dae1; background:
  radial-gradient(400px 160px at 50% 0%, rgba(249, 115, 22, .05), transparent 70%),
  radial-gradient(rgba(16, 24, 40, .06) 1px, transparent 1.2px) 0 0 / 16px 16px, #fcfcfd;
  padding: 30px 20px 26px; }
.dropzone:hover, .dropzone.drag { border-color: var(--accent-400); background:
  radial-gradient(420px 180px at 50% 0%, rgba(249, 115, 22, .09), transparent 70%),
  radial-gradient(rgba(16, 24, 40, .06) 1px, transparent 1.2px) 0 0 / 16px 16px, #fffdfc; }
.dz-art { position: relative; width: 120px; height: 82px; margin-bottom: 4px; }
.dz-sheet { position: absolute; top: 6px; width: 52px; height: 66px; border-radius: 8px; background: #fff; border: 1px solid var(--hairline);
  box-shadow: var(--lift-2); display: grid; align-content: start; gap: 5px; padding: 12px 9px; transition: transform .35s cubic-bezier(.34, 1.4, .64, 1); }
.dz-sheet i { display: block; height: 4px; border-radius: 3px; background: var(--line); }
.dz-sheet i:nth-child(2) { width: 75%; } .dz-sheet i:last-child { width: 55%; }
.dz-sheet b { font-size: 9px; font-weight: 800; letter-spacing: .4px; color: #fff; background: var(--grad); border-radius: 4px; padding: 1px 4px; justify-self: start; margin-bottom: 2px; }
.dz-sheet.s1 { left: 10px; transform: rotate(-10deg); }
.dz-sheet.s2 { right: 10px; transform: rotate(10deg); }
.dz-sheet.s3 { left: 34px; top: 0; z-index: 1; box-shadow: var(--lift-3); }
.dz-badge { position: absolute; z-index: 2; right: 22px; bottom: -2px; width: 30px; height: 30px; border-radius: 50%; display: grid; place-items: center; color: #fff;
  background: var(--grad); box-shadow: 0 0 0 3px #fff, 0 6px 14px -4px rgba(var(--accent-rgb), .6); transition: transform .3s cubic-bezier(.34, 1.4, .64, 1); }
.dropzone:hover .dz-sheet.s1, .dropzone.drag .dz-sheet.s1 { transform: rotate(-16deg) translate(-6px, 2px); }
.dropzone:hover .dz-sheet.s2, .dropzone.drag .dz-sheet.s2 { transform: rotate(16deg) translate(6px, 2px); }
.dropzone:hover .dz-sheet.s3, .dropzone.drag .dz-sheet.s3 { transform: translateY(-5px); }
.dropzone:hover .dz-badge, .dropzone.drag .dz-badge { transform: translateY(-3px) scale(1.06); }
.dz-types span { background: rgba(255, 255, 255, .9); box-shadow: var(--lift-1); border-color: var(--hairline); }
.fcard { box-shadow: var(--edge), var(--lift-1); border-color: var(--hairline); border-radius: 14px; }

/* ---- choice cards: a fine gradient border when chosen */
.choice { border-color: var(--hairline); box-shadow: var(--edge), var(--lift-1); border-radius: 16px; position: relative; }
.choice:hover { box-shadow: var(--edge), var(--lift-2); }
.choice.on { border-color: transparent; background: linear-gradient(180deg, #fff, #fffaf8); box-shadow: var(--edge), var(--lift-2), 0 0 0 3px rgba(var(--accent-rgb), .06); }
.choice.on::before { content: ""; position: absolute; inset: -1.5px; border-radius: 17px; padding: 1.5px; background: var(--grad);
  -webkit-mask: linear-gradient(#000 0 0) content-box, linear-gradient(#000 0 0); -webkit-mask-composite: xor; mask-composite: exclude; pointer-events: none; }
.choice-ico { border: 1px solid var(--hairline); background: linear-gradient(180deg, #fff, var(--surface-2)); box-shadow: var(--lift-1); }
.choice.on .choice-ico { background: linear-gradient(160deg, #fff 20%, var(--accent-50)); border-color: var(--accent-100); }
.choice.on .choice-mark { background: var(--grad); box-shadow: 0 3px 8px -2px rgba(var(--accent-rgb), .5); }

/* ---- option panels, model cards, folds */
.opt, .epcard, .fold, .rule2 { border-color: var(--hairline); box-shadow: var(--edge), var(--lift-1); border-radius: 16px; }
.opt.on, .fold[open], .epcard:hover { box-shadow: var(--edge), var(--lift-2); }
.opt-ico, .fold summary .opt-ico { border: 1px solid var(--hairline); background: linear-gradient(180deg, #fff, var(--surface-2)); box-shadow: var(--lift-1); }
.opt.on .opt-ico { background: linear-gradient(160deg, #fff 20%, var(--accent-50)); border-color: var(--accent-100); }
.opt-bd { background: linear-gradient(180deg, rgba(250, 250, 251, .7), transparent); border-top-style: solid; border-top-color: var(--line-2); }
.mixsum { background: linear-gradient(180deg, #fff, #fbfbfc); border-color: var(--hairline); box-shadow: var(--edge), var(--lift-1); }
.mixbar { height: 10px; box-shadow: inset 0 1px 2px rgba(16, 24, 40, .08); }

/* ---- form controls: soft inset, crisp focus */
input:not([type=checkbox]):not([type=file]), select, textarea { border-color: #dfe2e7; box-shadow: inset 0 1px 2px rgba(16, 24, 40, .04); background-color: #fff; }
input:focus, select:focus, textarea:focus { border-color: var(--accent-400); box-shadow: 0 0 0 4px rgba(var(--accent-rgb), .12), inset 0 1px 2px rgba(16, 24, 40, .03); }
.seg { background: linear-gradient(180deg, #f1f2f4, #f6f7f8); box-shadow: inset 0 1px 2px rgba(16, 24, 40, .06); border-color: var(--hairline); }
.seg button.on { box-shadow: 0 1px 2px rgba(16, 24, 40, .1), 0 2px 6px -2px rgba(16, 24, 40, .12), 0 0 0 1px rgba(16, 24, 40, .04); }
.switch { box-shadow: inset 0 1px 2px rgba(16, 24, 40, .15); }
.switch span { box-shadow: 0 1px 3px rgba(16, 24, 40, .25), 0 0 0 .5px rgba(16, 24, 40, .06); }
.chip { box-shadow: var(--lift-1); border-color: var(--hairline); }
.gen .chip.on { background: linear-gradient(180deg, #fff, #fffaf8); border-color: var(--accent-200); }

/* ---- buttons: depth on the main actions */
.btn { box-shadow: var(--edge), var(--lift-1); border-color: #dfe2e7; }
.btn.ghost { box-shadow: none; }
.btn.dark { background: linear-gradient(180deg, #2a3140, #111827); border-color: #111827; box-shadow: inset 0 1px 0 rgba(255, 255, 255, .12), 0 4px 10px -4px rgba(16, 24, 40, .5); }
.btn.primary { box-shadow: inset 0 1px 0 rgba(255, 255, 255, .25), 0 6px 16px -6px rgba(var(--accent-rgb), .6); }
.btn.primary.lg { position: relative; overflow: hidden; }
.btn.primary.lg::after { content: ""; position: absolute; top: 0; bottom: 0; left: -40%; width: 30%; transform: skewX(-20deg);
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, .35), transparent); transition: left .7s ease; }
.btn.primary.lg:hover::after { left: 120%; }

/* ---- summary: frosted header with a faint warm wash, tidy rows */
.summary { background: rgba(255, 255, 255, .94); }
.sum-hd { background: radial-gradient(320px 140px at 100% 0%, rgba(249, 115, 22, .08), transparent 70%),
  radial-gradient(260px 120px at 0% 0%, rgba(236, 72, 153, .05), transparent 70%), #fff; border-bottom-color: var(--line-2); }
.sum-ico { border: 1px solid var(--hairline); background: linear-gradient(180deg, #fff, var(--surface-2)); box-shadow: var(--lift-1); }
.sum-ft { background: linear-gradient(180deg, #fbfbfc, #f7f8f9); }
.sum-ready { padding: 10px 12px; border-radius: 12px; background: linear-gradient(180deg, #f3fcf7, #ecfaf2); border: 1px solid #c8eedd; }
.sum-issues { box-shadow: var(--edge), var(--lift-1); }

/* ---- run view */
.run-hero { background: radial-gradient(500px 200px at 100% 0%, rgba(249, 115, 22, .06), transparent 70%), #fff; }
.run-bar { height: 10px; box-shadow: inset 0 1px 2px rgba(16, 24, 40, .08); }
.tl-dot { box-shadow: 0 0 0 4px #fff; }




















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
import "./rich.css";

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




























//steps.jsx
// The five setup steps of the Generate Dataset page.
import { useRef, useState } from "react";
import { D_PRESETS, G_TOPICS, MIX, P_STYLES, host, mixEstimate, mixWarnings, short, visionUsed } from "../model.js";
import { Chips, Field, Icon, NumberInput, Segmented, Switch } from "../ui.jsx";

const ext = (n) => { const e = (n.split(".").pop() || "").toLowerCase(); return e === "htm" ? "html" : e === "markdown" ? "md" : e; };
const toggle = (list, k) => (list.includes(k) ? list.filter((x) => x !== k) : [...list, k]);
const mb = (b) => (b / 1048576 < 0.1 ? `${Math.max(1, Math.round(b / 1024))} KB` : `${(b / 1048576).toFixed(1)} MB`);

// ------------------------------------------------------------------ small building blocks
function Section({ title, hint, right, children }) {
  return (
    <section className="g-sec">
      <div className="g-sec-hd"><div><h3>{title}</h3>{hint && <p>{hint}</p>}</div>{right}</div>
      {children}
    </section>
  );
}

// a large selectable card: radio (single) or checkbox (multi) behaviour
function Choice({ on, onClick, icon, title, text, multi }) {
  return (
    <button type="button" role={multi ? "checkbox" : "radio"} aria-checked={on} className={`choice${on ? " on" : ""}`} onClick={onClick}>
      <span className="choice-top">
        {icon && <span className="choice-ico"><Icon name={icon} size={18} /></span>}
        <span className={`choice-mark${multi ? " sq" : ""}`}>{on && <Icon name="check" size={12} stroke={3} />}</span>
      </span>
      <b>{title}</b>
      {text && <small>{text}</small>}
    </button>
  );
}

function Toggle({ icon, title, text, on, onChange, children }) {
  return (
    <div className={`opt${on ? " on" : ""}`}>
      <div className="opt-top">
        <span className="opt-ico"><Icon name={icon} size={18} /></span>
        <div className="opt-t"><b>{title}</b><small>{text}</small></div>
        <Switch on={on} onChange={onChange} label={title} />
      </div>
      {on && children && <div className="opt-bd">{children}</div>}
    </div>
  );
}

function StatusBadge({ s, used = true, testing }) {
  let cls = "ok", text = "";
  if (!used) [cls, text] = ["muted", "Not used"];
  else if (testing) [cls, text] = ["run", "Testing…"];
  else if (!s) [cls, text] = ["muted", "Not tested"];
  else if (!s.ok) [cls, text] = ["bad", "Offline"];
  else if (!s.models?.length) [cls, text] = ["bad", "No models"];
  else text = `Online · ${s.latency_ms} ms`;
  return <span className={`badge ${cls}`}><i />{text}</span>;
}

export function MixBar({ mix }) {
  const parts = Object.entries(mix).filter(([, v]) => v > 0);
  const tot = parts.reduce((a, [, v]) => a + v, 0) || 1;
  return (
    <div className="mix">
      <div className="mixbar">{parts.map(([k, v]) => <i key={k} style={{ width: `${(100 * v) / tot}%`, background: MIX[k][1] }} title={`${MIX[k][0]} ${Math.round((100 * v) / tot)}%`} />)}</div>
      <div className="mixleg">{parts.map(([k, v]) => <span key={k} style={{ "--c": MIX[k][1] }}>{MIX[k][0]} <b>{Math.round((100 * v) / tot)}%</b></span>)}</div>
    </div>
  );
}

// ------------------------------------------------------------------ 1. documents
export function DocumentsStep({ form, set, files, setFiles, defaults }) {
  const input = useRef(null);
  const [drag, setDrag] = useState(false);
  const [showFolders, setShowFolders] = useState(!!form.folders.trim());
  // copy the FileList now: it is live, and clearing the input empties it
  const add = (list) => {
    const picked = Array.from(list || []);
    if (picked.length) setFiles((cur) => [...cur, ...picked.filter((f) => !cur.some((x) => x.name === f.name && x.size === f.size))]);
  };
  const roots = defaults?.input_roots || [];
  const total = files.reduce((a, f) => a + f.size, 0);
  return (
    <>
      <Section title="Upload files" hint="They are uploaded when the job starts."
        right={files.length > 0 && <span className="muted small">{files.length} file{files.length > 1 ? "s" : ""} · {mb(total)}</span>}>
        <div className={`dropzone${drag ? " drag" : ""}`} role="button" tabIndex={0} onClick={() => input.current.click()}
          onKeyDown={(e) => (e.key === "Enter" || e.key === " ") && (e.preventDefault(), input.current.click())}
          onDragOver={(e) => { e.preventDefault(); setDrag(true); }} onDragLeave={() => setDrag(false)}
          onDrop={(e) => { e.preventDefault(); setDrag(false); add(e.dataTransfer.files); }}>
          <span className="dz-art" aria-hidden="true">
            <span className="dz-sheet s1"><i /><i /><i /></span>
            <span className="dz-sheet s2"><i /><i /><i /></span>
            <span className="dz-sheet s3"><b>PDF</b><i /><i /><i /></span>
            <span className="dz-badge"><Icon name="upload" size={16} stroke={2.2} /></span>
          </span>
          <b>Drop files here, or <u>browse</u></b>
          <span className="dz-types">{["PDF", "DOCX", "TXT", "Markdown", "HTML"].map((t) => <span key={t}>{t}</span>)}</span>
          <input ref={input} type="file" multiple hidden accept=".pdf,.docx,.txt,.md,.markdown,.html,.htm" onChange={(e) => { add(e.target.files); e.target.value = ""; }} />
        </div>
        {files.length > 0 && (
          <div className="filegrid">
            {files.map((f, i) => (
              <div className="fcard" key={f.name + f.size}>
                <span className={`ext ${ext(f.name)}`}>{ext(f.name).toUpperCase()}</span>
                <span className="fcard-n" title={f.name}>{f.name}<small>{mb(f.size)}</small></span>
                <button type="button" className="icon-btn" aria-label={`Remove ${f.name}`} onClick={() => setFiles((x) => x.filter((_, k) => k !== i))}><Icon name="x" size={14} /></button>
              </div>
            ))}
          </div>
        )}
        {!showFolders
          ? <button type="button" className="link" onClick={() => setShowFolders(true)}><Icon name="folder" size={14} />Or use folders that are already on the server</button>
          : <Field label="Folders on the server (one per line)" hint={roots.length ? `Allowed: ${roots.join(" · ")}` : ""}>
            <textarea className="mono" rows={2} value={form.folders} onChange={(e) => set("folders", e.target.value)}
              placeholder={roots[0] ? `${roots[0]}${roots[0].includes("\\") ? "\\" : "/"}my_documents` : "/data/documents/my_documents"} />
          </Field>}
      </Section>

      <Section title="Dataset">
        <div className="grid2">
          <Field label="Dataset name" hint="Used for the job and the output files."><input value={form.name} onChange={(e) => set("name", e.target.value)} placeholder="e.g. hr_policies" /></Field>
          <Field label="Images inside the documents" hint={form.readImages ? "Charts, screenshots and diagrams are read too." : "Only the text is used."}>
            <Segmented value={form.readImages} onChange={(v) => set("readImages", v)} options={[[true, "Read them"], [false, "Ignore them"]]} />
          </Field>
        </div>
        <div className="choices two" role="radiogroup" aria-label="Dataset for">
          <Choice on={form.mode === "chat"} onClick={() => set("mode", "chat")} icon="chat" title="Chat fine-tuning" text="Question → answer. The usual choice for assistants." />
          <Choice on={form.mode === "reasoning"} onClick={() => set("mode", "reasoning")} icon="spark" title="Reasoning fine-tuning" text="Question → thinking trace → answer, for reasoning models." />
        </div>
      </Section>
    </>
  );
}

// ------------------------------------------------------------------ 2. models
export function ModelsStep({ form, set, ep, models, testEndpoints }) {
  const vUsed = visionUsed(form);
  const opts = (list, cur) => [...new Set([cur, ...(list || [])].filter(Boolean))];
  const testing = ep.state === "testing";
  return (
    <>
      <div className="epcards">
        <div className="epcard">
          <div className="epcard-hd">
            <span className="opt-ico on"><Icon name="server" size={18} /></span>
            <div className="opt-t"><b>Teacher model</b><small>Writes the questions and answers, then judges them</small></div>
          </div>
          <Field label="Endpoint URL"><input className="mono" value={form.teacherUrl} placeholder="http://gpu-host:8001/v1" onChange={(e) => set("teacherUrl", e.target.value)} /></Field>
          <Field label="Model"><select value={form.teacherModel} onChange={(e) => set("teacherModel", e.target.value)}>
            {opts(models.teacher, form.teacherModel).map((m) => <option key={m}>{m}</option>)}</select></Field>
          <div className="epcard-ft"><StatusBadge s={ep.teacher} testing={testing} /></div>
        </div>

        <div className={`epcard${vUsed ? "" : " off"}`}>
          <div className="epcard-hd">
            <span className={`opt-ico${vUsed ? " on" : ""}`}><Icon name="image" size={18} /></span>
            <div className="opt-t"><b>Vision model</b><small>Describes images, charts and screenshots</small></div>
            <Switch on={form.vision} onChange={(v) => set("vision", v)} label="Vision analysis" disabled={!form.readImages} />
          </div>
          {!form.readImages ? <p className="note"><Icon name="alert" size={14} />Images are ignored (see Documents), so the vision model is not used.</p>
            : !form.vision ? <p className="note"><Icon name="alert" size={14} />Off: OCR only. Text printed in images is kept; faster.</p>
              : <>
                <Field label="Endpoint URL"><input className="mono" value={form.visionUrl} placeholder="http://gpu-host:8002/v1" onChange={(e) => set("visionUrl", e.target.value)} /></Field>
                <Field label="Model"><select value={form.visionModel} onChange={(e) => set("visionModel", e.target.value)}>
                  {opts(models.vision, form.visionModel).map((m) => <option key={m}>{m}</option>)}</select></Field>
                <div className="epcard-ft"><StatusBadge s={ep.vision} used={vUsed} testing={testing} /></div>
              </>}
        </div>
      </div>

      <Section title="Connection" hint="OpenAI-compatible endpoints (for example vLLM). The tool only calls them; it never starts or loads a model.">
        <div className="grid2">
          <Field label="API key (optional)" hint="Sent as a Bearer token to both endpoints"><input type="password" autoComplete="off" value={form.apiKey} placeholder="none" onChange={(e) => set("apiKey", e.target.value)} /></Field>
          <Field label="Parallel requests" hint="How many calls run at the same time"><NumberInput value={form.maxParallel} min={1} max={64} onChange={(v) => set("maxParallel", v)} /></Field>
        </div>
        <div className={`testbar${ep.bad ? " bad" : ep.state === "done" ? " good" : ""}`}>
          <Icon name={ep.bad ? "alert" : ep.state === "done" ? "check" : "server"} size={16} />
          <span>{testing ? "Testing the endpoints…" : ep.message || (ep.state === "done"
            ? `All good: the teacher serves ${short(form.teacherModel)} at ${host(form.teacherUrl)}${vUsed ? `, vision serves ${short(form.visionModel)}` : ""}.`
            : "Endpoints are tested automatically when you change them.")}</span>
          <div className="grow" />
          <button type="button" className="btn small" disabled={testing} onClick={() => testEndpoints()}><Icon name="refresh" size={13} />Test again</button>
        </div>
      </Section>
    </>
  );
}

// ------------------------------------------------------------------ 3. questions
const QTYPES = [
  ["factual", "Factual", "Facts stated in the text", "list"],
  ["explanation", "Explanation", "Why and how things work", "book"],
  ["how-to", "How-to", "Step-by-step instructions", "layers"],
  ["from_images", "From images", "About charts, screenshots and diagrams", "image"],
  ["comparison", "Comparison", "Compare things across documents", "repeat"],
];
const SPLIT_COLORS = ["var(--split-1)", "var(--split-2)", "var(--split-3)"];

export function QuestionsStep({ form, set }) {
  const total = form.split.reduce((a, b) => a + (Number(b) || 0), 0);
  return (
    <>
      <Section title="Question types" hint="Pick one or more. The teacher writes a mix of them for every part of your documents.">
        <div className="choices five">
          {QTYPES.map(([k, l, t, ic]) => (
            <Choice multi key={k} on={form.types.includes(k)} onClick={() => set("types", toggle(form.types, k))} icon={ic} title={l} text={t} />
          ))}
        </div>
      </Section>

      <Section title="Answers">
        <div className="choices two" role="radiogroup" aria-label="Answer length">
          <Choice on={form.answerLength === "short"} onClick={() => set("answerLength", "short")} title="Short answers" text="One or two sentences, straight to the point." />
          <Choice on={form.answerLength === "detailed"} onClick={() => set("answerLength", "detailed")} title="Detailed answers" text="Full explanations with the relevant context." />
        </div>
        <div className="grid2">
          <Field label="Max Q&A pairs" hint="Upper limit for document Q&A pairs."><NumberInput value={form.maxPairs} min={1} onChange={(v) => set("maxPairs", v)} /></Field>
          <Field label="System prompt (optional)" hint="First message of every row. Use the same text when you serve the model.">
            <textarea rows={2} value={form.systemPrompt} onChange={(e) => set("systemPrompt", e.target.value)} placeholder="e.g. You are the HR assistant for Contoso." />
          </Field>
        </div>
      </Section>

      <Section title="Train / validation / test split" right={<span className={`badge ${total === 100 ? "ok" : "bad"}`}><i />{total === 100 ? "Totals 100%" : `Totals ${total}%, must be 100%`}</span>}>
        <div className="splitbar">{form.split.map((v, i) => (Number(v) > 0 ? <i key={i} style={{ flex: Number(v), background: SPLIT_COLORS[i] }} title={`${["Train", "Validation", "Test"][i]} ${v}%`}>{Number(v) >= 12 && <span>{["Train", "Validation", "Test"][i]} {v}%</span>}</i> : null))}</div>
        <div className="split4">
          {["Train %", "Validation %", "Test %"].map((l, i) => (
            <span className="inset" key={l}><small>{l}</small><NumberInput value={form.split[i]} min={0} max={100}
              onChange={(v) => set("split", form.split.map((x, k) => (k === i ? v : x)))} /></span>
          ))}
          <span className="inset"><small>Random seed</small><NumberInput value={form.seed} onChange={(v) => set("seed", v)} /></span>
        </div>
        <Field label="Split by" hint={form.splitBy === "fact" ? "Validation and test ask facts that were trained, in new words." : "Validation and test use whole sections that were never trained on."}>
          <Segmented value={form.splitBy} onChange={(v) => set("splitBy", v)} options={[["fact", "Fact"], ["section", "Section"]]} />
        </Field>
      </Section>
    </>
  );
}

// ------------------------------------------------------------------ 4. training mix
export function MixStep({ form, setIn, setForm }) {
  const [topic, setTopic] = useState("");
  const [open, setOpen] = useState(null);
  const g = form.general, d = form.decline, p = form.para;
  const own = g.topics.filter((t) => !G_TOPICS.some(([k]) => k === t));
  const addTopic = () => {
    const t = topic.trim();
    if (t && !g.topics.some((x) => x.toLowerCase() === t.toLowerCase())) setIn("general", "topics", [...g.topics, t]);
    setTopic("");
  };
  const setRules = (fn) => setForm((f) => ({ ...f, decline: { ...f.decline, rules: fn(f.decline.rules) } }));
  const upd = (i, k, v) => setRules((rs) => rs.map((r, j) => (j === i ? { ...r, [k]: v } : r)));
  const missing = D_PRESETS.filter((x) => !d.rules.some((r) => r.name.toLowerCase() === x.name.toLowerCase()));
  const mix = mixEstimate(form), warnings = mixWarnings(form);
  return (
    <>
      <div className="mixsum">
        <div className="mixsum-hd"><b>Expected mix</b><span className="muted small">{mix.paraphrases || mix.decline || mix.general ? "per 100 document Q&A pairs" : "document Q&A only"}</span></div>
        <MixBar mix={mix} />
      </div>
      {warnings.map((w) => <div className="warn" key={w}><Icon name="alert" size={15} />{w}</div>)}

      <Toggle icon="repeat" title="Paraphrases" text="The same question in new words, with the same answer" on={p.enabled} onChange={(v) => setIn("para", "enabled", v)}>
        <div className="inline"><span>New wordings per question</span><NumberInput className="mini" value={p.count} min={1} max={10} onChange={(v) => setIn("para", "count", v)} /></div>
        <Chips options={P_STYLES} value={p.styles} onToggle={(k) => setIn("para", "styles", toggle(p.styles, k))} />
      </Toggle>

      <Toggle icon="shield" title="Decline answers" text="Questions the model should politely refuse" on={d.enabled} onChange={(v) => setIn("decline", "enabled", v)}>
        <div className="inline"><span>Share of document pairs</span><NumberInput className="mini" value={d.share} min={1} max={30} onChange={(v) => setIn("decline", "share", v)} /><span>%</span></div>
        <div className="rules2">
          {d.rules.map((r, i) => (
            <div className={`rule2${r.on ? "" : " off"}${open === i ? " open" : ""}`} key={i}>
              <div className="rule2-top">
                <Switch on={r.on} onChange={(v) => upd(i, "on", v)} label={`Use ${r.name}`} />
                <button type="button" className="rule2-name" onClick={() => setOpen(open === i ? null : i)}>
                  <b>{r.name || "(no name)"}</b><small>{r.scope || "No scope yet"}</small>
                </button>
                <span className="tag">{r.reply_mode === "guided" ? "Teacher writes reply" : "Exact reply"}</span>
                <button type="button" className="icon-btn" aria-label={open === i ? "Close" : `Edit ${r.name}`} onClick={() => setOpen(open === i ? null : i)}><Icon name={open === i ? "x" : "edit"} size={14} /></button>
              </div>
              {open === i && (
                <div className="rule2-bd">
                  <Field label="Rule name"><input value={r.name} onChange={(e) => upd(i, "name", e.target.value)} /></Field>
                  <Field label="Decline questions like these"><textarea rows={2} value={r.scope} placeholder="e.g. Questions asking for the salary of a named colleague." onChange={(e) => upd(i, "scope", e.target.value)} /></Field>
                  <Field label="Reply">
                    <Segmented small value={r.reply_mode} onChange={(v) => upd(i, "reply_mode", v)} options={[["fixed", "Exact text"], ["guided", "Teacher writes it"]]} />
                    <textarea rows={2} value={r.reply} onChange={(e) => upd(i, "reply", e.target.value)}
                      placeholder={r.reply_mode === "guided" ? "How to reply, e.g. Politely refuse and point to the HR portal." : "The exact reply for every question of this rule."} />
                  </Field>
                  <div className="inline"><div className="grow" /><button type="button" className="btn small ghost danger" onClick={() => { setRules((rs) => rs.filter((_, j) => j !== i)); setOpen(null); }}><Icon name="trash" size={13} />Delete rule</button></div>
                </div>
              )}
            </div>
          ))}
        </div>
        <div className="inline">
          <button type="button" className="btn small" onClick={() => { setRules((rs) => [...rs, { name: `Rule ${rs.length + 1}`, scope: "", reply: "", reply_mode: "fixed", on: true }]); setOpen(d.rules.length); }}><Icon name="plus" size={13} />New rule</button>
          {missing.map((x) => <button type="button" key={x.name} className="btn small ghost" onClick={() => setRules((rs) => [...rs, { ...x, on: true }])}><Icon name="plus" size={13} />{x.name}</button>)}
        </div>
      </Toggle>

      <Toggle icon="globe" title="General knowledge" text="Keeps everyday answers: facts, how-tos, writing help" on={g.enabled} onChange={(v) => setIn("general", "enabled", v)}>
        <div className="inline"><span>Share of document pairs</span><NumberInput className="mini" value={g.share} min={1} max={50} onChange={(v) => setIn("general", "share", v)} /><span>%</span></div>
        <Chips options={[...G_TOPICS, ...own.map((t) => [t, t])]} value={g.topics} onToggle={(k) => setIn("general", "topics", toggle(g.topics, k))}
          onRemove={(k) => (own.includes(k) ? () => setIn("general", "topics", g.topics.filter((x) => x !== k)) : null)} />
        <div className="addtopic"><input value={topic} placeholder="Add your own topic…" onChange={(e) => setTopic(e.target.value)}
          onKeyDown={(e) => e.key === "Enter" && (e.preventDefault(), addTopic())} />
          <button type="button" className="btn small" onClick={addTopic} disabled={!topic.trim()}><Icon name="plus" size={13} />Add</button></div>
      </Toggle>
    </>
  );
}

// ------------------------------------------------------------------ 5. advanced
function Fold({ icon, title, hint, children, first }) {
  return (
    <details className="fold" open={first}>
      <summary><span className="opt-ico"><Icon name={icon} size={16} /></span><span className="opt-t"><b>{title}</b><small>{hint}</small></span><Icon name="chevron" size={16} className="fold-chev" /></summary>
      <div className="fold-bd">{children}</div>
    </details>
  );
}

export function AdvancedStep({ form, set, setIn, onReset }) {
  const onOff = (k) => <Segmented value={form[k]} onChange={(v) => set(k, v)} options={[[true, "On"], [false, "Off"]]} />;
  return (
    <>
      <div className="inline"><span className="muted">Open a section to change it.</span><div className="grow" />
        <button type="button" className="btn small ghost" onClick={onReset}><Icon name="refresh" size={13} />Reset everything to the defaults</button></div>
      <Fold first icon="image" title="Parsing & images" hint="OCR, image size limits">
        <div className="grid3">
          <Field label="OCR for scanned pages">{onOff("ocr")}</Field>
          <Field label="Skip images smaller than (px)"><NumberInput value={form.minImagePx} onChange={(v) => set("minImagePx", v)} /></Field>
          <Field label="Max images per document"><NumberInput value={form.maxImages} onChange={(v) => set("maxImages", v)} /></Field>
          {form.vision
            ? <Field label="Skip image types" hint="Types the vision model may return" className="span2"><input value={form.skipImageTypes} onChange={(e) => set("skipImageTypes", e.target.value)} /></Field>
            : <Field label="Keep image text with at least (characters)"><NumberInput value={form.minOcrChars} onChange={(v) => set("minOcrChars", v)} /></Field>}
        </div>
      </Fold>
      <Fold icon="book" title="Chunks & knowledge base" hint="Chunk sizes, summaries, glossary">
        <div className="grid3">
          <Field label="Chunk size min (tokens)"><NumberInput value={form.chunkMin} onChange={(v) => set("chunkMin", v)} /></Field>
          <Field label="Chunk size max (tokens)"><NumberInput value={form.chunkMax} onChange={(v) => set("chunkMax", v)} /></Field>
          <Field label="Merge pieces below (tokens)"><NumberInput value={form.mergeBelow} onChange={(v) => set("mergeBelow", v)} /></Field>
          <Field label="Related chunks per answer" hint="0 = no related-chunk search"><NumberInput value={form.related} min={0} max={8} onChange={(v) => set("related", v)} /></Field>
          <Field label="Document summary">{onOff("summary")}</Field>
          <Field label="Glossary">{onOff("glossary")}</Field>
          <Field label="Collection overview">{onOff("overview")}</Field>
        </div>
      </Fold>
      <Fold icon="spark" title="Generation" hint="Questions per chunk, temperatures, answer length">
        <div className="grid3">
          <Field label="One question per (tokens)"><NumberInput value={form.tpq} onChange={(v) => set("tpq", v)} /></Field>
          <Field label="Questions per chunk, min"><NumberInput value={form.perMin} onChange={(v) => set("perMin", v)} /></Field>
          <Field label="Questions per chunk, max"><NumberInput value={form.perMax} onChange={(v) => set("perMax", v)} /></Field>
          <Field label="Question temperature"><NumberInput step={0.1} value={form.tempQ} onChange={(v) => set("tempQ", v)} /></Field>
          <Field label="Answer temperature"><NumberInput step={0.1} value={form.tempA} onChange={(v) => set("tempA", v)} /></Field>
          <Field label="Max answer tokens"><NumberInput value={form.maxAnswerTokens} onChange={(v) => set("maxAnswerTokens", v)} /></Field>
          <Field label="Paraphrase temperature"><NumberInput step={0.1} value={form.para.temperature} onChange={(v) => setIn("para", "temperature", v)} /></Field>
          <Field label="General knowledge answers"><Segmented value={form.general.answer_length} onChange={(v) => setIn("general", "answer_length", v)} options={[["short", "Short"], ["detailed", "Detailed"]]} /></Field>
          <Field label="General answer temperature"><NumberInput step={0.1} value={form.general.temperature} onChange={(v) => setIn("general", "temperature", v)} /></Field>
        </div>
      </Fold>
      <Fold icon="check" title="Judge & duplicates" hint="Quality thresholds, near-duplicate removal">
        <div className="grid3">
          <Field label="Grounded score at least" hint="1–5"><NumberInput value={form.minGrounded} min={1} max={5} onChange={(v) => set("minGrounded", v)} /></Field>
          <Field label="Other scores at least" hint="1–5"><NumberInput value={form.minOther} min={1} max={5} onChange={(v) => set("minOther", v)} /></Field>
          <Field label="Max row length (tokens)"><NumberInput value={form.maxRowTokens} onChange={(v) => set("maxRowTokens", v)} /></Field>
          <Field label="Near-duplicate when Jaccard ≥"><NumberInput step={0.05} value={form.jaccard} onChange={(v) => set("jaccard", v)} /></Field>
          <Field label="Ask the teacher when Jaccard ≥" hint="1.0 = never ask"><NumberInput step={0.05} value={form.teacherJaccard} onChange={(v) => set("teacherJaccard", v)} /></Field>
        </div>
      </Fold>
      <Fold icon="download" title="Output" hint="Where the accepted dataset is saved">
        <Field label="Save the dataset as (on Accept)" hint="Default: the job folder's dataset/all.jsonl"><input className="mono" value={form.outPath} placeholder="<job folder>/dataset/all.jsonl" onChange={(e) => set("outPath", e.target.value)} /></Field>
      </Fold>
    </>
  );
}
