//fields.css
/* Sharp form fields: inputs, dropdowns, text areas and segmented controls.
   Crisp 1px borders with real contrast, tighter corners, flat backgrounds (no soft inner shadow),
   solid text and a clear two-tone focus ring in the current palette's accent.
   Loaded after styles.css / rich.css / theme-dark.css; the :root prefix makes these rules win. */
:root {
  --field-radius: 6px;
  --field-bg: #ffffff;
  --field-line: #b8c0cc;
  --field-line-hover: #7d8696;
  --field-ink: #0b1220;
  --field-ph: #7b8494;
  --field-disabled: #f1f3f6;
  --field-chev: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='%231f2937' stroke-width='2.4' stroke-linecap='round' stroke-linejoin='round'%3E%3Cpath d='m7 10 5 5 5-5'/%3E%3C/svg%3E");
}
:root[data-theme="dark"] {
  --field-bg: #0a0c11;
  --field-line: #4a5264;
  --field-line-hover: #7a8396;
  --field-ink: #f5f7fa;
  --field-ph: #858da0;
  --field-disabled: #161a22;
  --field-chev: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='%23e5e7eb' stroke-width='2.4' stroke-linecap='round' stroke-linejoin='round'%3E%3Cpath d='m7 10 5 5 5-5'/%3E%3C/svg%3E");
}

:root input:not([type=checkbox]):not([type=file]):not([type=radio]):not([type=range]),
:root select, :root textarea {
  border: 1px solid var(--field-line); border-radius: var(--field-radius);
  background-color: var(--field-bg); color: var(--field-ink); box-shadow: none;
  font-weight: 500; letter-spacing: -.005em;
  -webkit-font-smoothing: antialiased; -moz-osx-font-smoothing: grayscale; text-rendering: optimizeLegibility;
  transition: border-color .1s, box-shadow .1s, background-color .1s;
}
:root input::placeholder, :root textarea::placeholder { color: var(--field-ph); font-weight: 400; opacity: 1; }

:root input:not([type=checkbox]):not([type=file]):not([type=radio]):not([type=range]):hover:not(:focus):not(:disabled), :root select:hover:not(:focus):not(:disabled), :root textarea:hover:not(:focus):not(:disabled) {
  border-color: var(--field-line-hover);
}
/* focus: solid accent edge plus a thin, tight halo, so the active field is unmistakable */
:root input:not([type=checkbox]):not([type=file]):not([type=radio]):not([type=range]):focus, :root select:focus, :root textarea:focus {
  border-color: var(--accent); outline: none;
  box-shadow: 0 0 0 1px var(--accent), 0 0 0 4px rgba(var(--accent-rgb), .16);
}
:root input:not([type=checkbox]):not([type=file]):not([type=radio]):not([type=range]):disabled, :root select:disabled, :root textarea:disabled {
  background-color: var(--field-disabled); color: var(--muted); cursor: not-allowed; border-style: dashed;
}

/* dropdowns: darker, bolder chevron that matches the text */
:root select {
  background-image: var(--field-chev); background-repeat: no-repeat; background-position: right 9px center; background-size: 16px;
}
:root select option { color: var(--ink); background: var(--surface); font-weight: 500; }

/* the label inside the split / min-max number boxes */
:root .inset small { color: var(--ink-2); font-weight: 600; }

/* segmented controls (On / Off, Fact / Section, ...) */
:root .seg { border-radius: 8px; border: 1px solid var(--field-line); background: var(--surface-3); box-shadow: none; }
:root .seg button { border-radius: 5px; }
:root .seg button.on { box-shadow: 0 0 0 1px var(--field-line), 0 1px 2px rgba(16, 24, 40, .12); }
:root[data-theme="dark"] .seg { background: var(--field-bg); }
:root[data-theme="dark"] .seg button.on { background: var(--surface-3); box-shadow: 0 0 0 1px var(--field-line); }

/* keep the borderless search box borderless */
:root .search input { border: 0 !important; box-shadow: none !important; background: none !important; }
:root .search { border-radius: var(--field-radius); border-color: var(--field-line); background: var(--field-bg); }
:root .search:focus-within { border-color: var(--accent); box-shadow: 0 0 0 1px var(--accent), 0 0 0 4px rgba(var(--accent-rgb), .16); }






















//App.jsx
// Routing: the landing page at #/ (the first screen), and the app shell with sidebar and top bar at #/generate and #/results.
// Features are added one piece at a time; see README.md for the build order.
import { useCallback, useEffect, useState } from "react";
import { Icon } from "./ui.jsx";
import Landing from "./Landing.jsx";
import Sidebar from "./Sidebar.jsx";
import { emit, lastOf, useBus } from "./bus.js";
import GeneratePage, { STEPS } from "./generate/GeneratePage.jsx";
import "./shell.css";
import "./rich.css";
import "./theme-dark.css";
import "./fields.css";
import ThemeToggle from "./ThemeToggle.jsx";
import { CustomizeButton } from "./Customize.jsx";
import ResultsPage from "./results/ResultsPage.jsx";

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
  const [rjob, setRjob] = useState(() => lastOf("results-job") || null);
  useBus("results-job", useCallback((j) => setRjob(j), []));
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
            {!onGen && rjob?.id && <><Icon name="chevron" size={12} /><span className="num">{rjob.name || rjob.id}</span></>}
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
        <span className="ph-sep" aria-hidden="true" />
        <CustomizeButton />
        <ThemeToggle />
      </div>
    </header>
  );
}
