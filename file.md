//index.html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>SLM Studio · Data generation</title>
    <!-- apply the saved light/dark theme before the page paints, so it never flashes -->
    <script>
      try { var t = localStorage.getItem("slm-theme") || (matchMedia("(prefers-color-scheme: dark)").matches ? "dark" : "light"); document.documentElement.dataset.theme = t; } catch (e) {}
    </script>
    <link rel="icon" type="image/svg+xml" href="/assets/logo-slm-studio.svg" />
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>











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
import "./theme-dark.css";
import ThemeToggle from "./ThemeToggle.jsx";

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
        <span className="ph-sep" aria-hidden="true" />
        <ThemeToggle />
      </div>
    </header>
  );
}

// Pieces 14–17 fill this page.
function ResultsPage() {
  return <section className="card"><Empty icon="results" title="Results page">Nothing here yet. The summary, Q&amp;A review and charts are added piece by piece.</Empty></section>;
}
















//Landing.jsx
// Landing page: the first screen. "Get started" opens the app at #/generate.
import Logo from "./Logo.jsx";
import ThemeToggle from "./ThemeToggle.jsx";
import { Icon } from "./ui.jsx";

const STEPS = [
  ["upload", "Add your documents", "Drop PDFs, Word files, Markdown, text or HTML, or point to a folder on the server."],
  ["gear", "Choose what to generate", "Pick question types, answer length and the training mix: paraphrases, polite declines and general knowledge."],
  ["check", "Review and export", "Check every Q&A pair, approve or edit it, then download a ready-to-train train / validation / test split."],
];

const FEATURES = [
  ["image", "Reads images too", "A vision model describes charts, screenshots and diagrams, so they become questions as well."],
  ["shield", "Quality judge", "Every answer is scored for grounding, correctness and clarity. Weak pairs are dropped automatically."],
  ["repeat", "Paraphrases", "The same question in new words, so the model understands how people really ask."],
  ["x", "Polite declines", "Teach the model what it must not answer, and exactly how it should say no."],
  ["globe", "General knowledge", "Keep everyday answers alive while the model learns your documents."],
  ["layers", "Clean splits", "Split by fact or by section, with duplicates removed, ready for fine-tuning."],
];

export default function Landing() {
  return (
    <div className="landing">
      <header className="lp-nav">
        <div className="lp-wrap lp-nav-in">
          <Logo size={40} withName={false} />
          <nav className="lp-links">
            <a href="#how">How it works</a>
            <a href="#features">Features</a>
          </nav>
          <ThemeToggle />
          <a className="btn primary" href="#/generate">Open the app<Icon name="arrow" size={15} /></a>
        </div>
      </header>

      <section className="lp-hero">
        <div className="lp-wrap lp-hero-in">
          <div className="lp-hero-text">
            <span className="lp-eyebrow"><Icon name="spark" size={14} />Synthetic data for small language models</span>
            <h1>Turn your documents into a <em>fine-tuning dataset</em></h1>
            <p>Upload your documents and get clean, grounded question-and-answer pairs, checked by a judge model and ready to train on.</p>
            <div className="lp-cta">
              <a className="btn primary lg" href="#/generate"><Icon name="play" size={13} />Get started</a>
              <a className="btn lg" href="#how">See how it works</a>
            </div>
          </div>
          <HeroPreview />
        </div>
      </section>

      <section className="lp-sec" id="how">
        <div className="lp-wrap">
          <div className="lp-sec-h"><span className="lp-eyebrow">How it works</span><h2>Three steps from documents to dataset</h2></div>
          <div className="lp-steps">
            {STEPS.map(([icon, title, text], i) => (
              <div className="lp-step" key={title}>
                <div className="lp-step-top"><span className="lp-step-n">{i + 1}</span><span className="lp-ico"><Icon name={icon} size={18} /></span></div>
                <h3>{title}</h3><p>{text}</p>
              </div>
            ))}
          </div>
        </div>
      </section>

      <section className="lp-sec alt" id="features">
        <div className="lp-wrap">
          <div className="lp-sec-h"><span className="lp-eyebrow">Features</span><h2>Everything a good training set needs</h2></div>
          <div className="lp-features">
            {FEATURES.map(([icon, title, text]) => (
              <div className="lp-feature" key={title}><span className="lp-ico"><Icon name={icon} size={18} /></span><div><h3>{title}</h3><p>{text}</p></div></div>
            ))}
          </div>
        </div>
      </section>

      <section className="lp-sec">
        <div className="lp-wrap">
          <div className="lp-band">
            <div><h2>Ready to build your first dataset?</h2><p>Start with a few documents. You can preview the result on 3 chunks before running the full job.</p></div>
            <a className="btn primary lg" href="#/generate">Get started<Icon name="arrow" size={15} /></a>
          </div>
        </div>
      </section>

      <footer className="lp-foot">
        <div className="lp-wrap lp-foot-in"><Logo size={28} withName={false} /><span className="muted small">Data generation for SLM fine-tuning</span></div>
      </footer>
    </div>
  );
}

// A small illustrative card in the hero: documents flowing into Q&A pairs.
function HeroPreview() {
  return (
    <div className="lp-preview" aria-hidden="true">
      <div className="pv-card">
        <div className="pv-head"><span className="pv-dots"><i /><i /><i /></span><span className="small muted">hr_policies · dataset</span></div>
        <div className="pv-files">
          <span className="pv-file"><b className="ext pdf">PDF</b>leave_policy.pdf</span>
          <span className="pv-file"><b className="ext docx">DOCX</b>leave_requests.docx</span>
        </div>
        <div className="pv-arrow"><Icon name="arrow" size={16} /></div>
        <div className="pv-qa">
          <div className="pv-q"><small>Question</small>How many leave days do new employees get?</div>
          <div className="pv-a"><small>Answer</small>New employees earn 2 days of leave per month, starting from their first day.</div>
          <div className="pv-scores"><span className="score shi">grounded 5</span><span className="score shi">correct 5</span><span className="score shi">clear 5</span></div>
        </div>
      </div>
      <div className="pv-stat"><span className="lp-ico green"><Icon name="check" size={16} /></span><div><b>1,248</b><small>pairs ready to train</small></div></div>
    </div>
  );
}

















//Logo.jsx
// Application logo, loaded from public/assets/.
//
// To use your own logo, replace public/assets/logo.svg with your file.
// If your file is not an SVG (for example logo.png), put it in public/assets/ and change LOGO_FILE below.
// The landing page and the sidebar both use this component.
// In dark mode LOGO_FILE_DARK is used (same logo with light text).
import { useTheme } from "./theme.js";

const LOGO_FILE = "logo-slm-studio-wide.svg";
const LOGO_FILE_DARK = "logo-slm-studio-wide-dark.svg";
const url = (f) => `${import.meta.env.BASE_URL}assets/${f}`;

// size is the logo height in px. The width follows the image's own shape, so wide or tall logos are never squeezed.
export default function Logo({ size = 36, withName = true, sub = "Data generation" }) {
  const [theme] = useTheme();
  const LOGO_URL = url(theme === "dark" ? LOGO_FILE_DARK : LOGO_FILE);
  return (
    <span className="logo">
      <img src={LOGO_URL} alt="SLM Studio" className="logo-img" style={{ height: size }} />
      {withName && <span className="logo-text"><b>SLM Studio</b>{sub && <small>{sub}</small>}</span>}
    </span>
  );
}





















//styles.css
/* SLM Studio · data generation — clean light theme */

/* ================================================================== tokens */
:root {
  /* neutrals */
  --bg: #f6f7f9;
  --surface: #ffffff;
  --surface-2: #fafbfc;
  --surface-3: #f3f4f6;
  --line: #e6e8ec;
  --line-2: #eff1f4;
  --line-strong: #d5d9e0;
  --ink: #111827;
  --ink-2: #374151;
  --muted: #6b7280;
  --faint: #9ca3af;

  /* ── THEME: Sunset coral, from linear-gradient(135deg, #F97316, #EC4899) ──
     Every theme color lives in this block. */
  --grad-a: #F97316;
  --grad-b: #EC4899;
  --grad-a-rgb: 249, 115, 22;
  --grad-b-rgb: 236, 72, 153;
  --grad: linear-gradient(135deg, var(--grad-a), var(--grad-b));
  --grad-hover: linear-gradient(135deg, #EA580C, #DB2777);
  --accent: #F2545B;
  --accent-rgb: 242, 84, 91;
  --accent-600: #D9265F;
  --accent-400: #FB7F7A;
  --accent-200: #FECDD3;
  --accent-100: #FFE4E6;
  --accent-50: #FFF4F1;
  --accent-ink: #4C0519;
  --tint-0: #FFFAF8;
  --tint-1: #FFF1EA;
  --tint-2: #FDF2F8;

  /* supporting tones */
  --violet: #DB2777;
  --violet-50: #FDF2F8;
  --violet-100: #FCE7F3;
  --violet-ink: #831843;

  /* training-mix and split colors */
  --mix-1: #F2545B;
  --mix-2: #F9B8A8;
  --mix-3: #94A3B8;
  --mix-4: #CBD5E1;
  --split-1: #F2545B;
  --split-2: #64748B;
  --split-3: #94A3B8;
  /* ── end of theme ── */

  --green: #10b981;
  --green-600: #059669;
  --green-50: #ecfdf5;
  --amber: #f59e0b;
  --orange: #ea580c;
  --orange-50: #fff7ed;
  --red: #dc2626;
  --red-50: #fef2f2;
  --red-200: #fbd5d5;
  --red-ink: #991b1b;
  --green-200: #c8eedd;
  --green-ink: #065f46;
  --amber-50: #fffbeb;
  --amber-200: #fde68a;
  --amber-ink: #92400e;
  --amber-ink-2: #78350f;

  /* depth and surfaces (overridden by the dark theme) */
  --paper-rgb: 255, 255, 255;
  --hairline: rgba(16, 24, 40, .08);
  --edge: inset 0 1px 0 rgba(255, 255, 255, .9);
  --dot: rgba(16, 24, 40, .055);
  --dark-btn-a: #2a3140;
  --dark-btn-b: #111827;
  --dark-btn-ink: #ffffff;
  --scroll: #d9dce2;
  --scroll-hover: #c3c8d0;
  --switch-off: #d9dde3;
  --switch-off-hover: #cbd0d8;
  --blue: #2563eb;
  --blue-50: #eff6ff;
  color-scheme: light;

  /* shape and depth */
  --radius-sm: 8px;
  --radius: 12px;
  --radius-lg: 16px;
  --shadow-xs: 0 1px 2px rgba(16, 24, 40, .04);
  --shadow: 0 1px 2px rgba(16, 24, 40, .04), 0 1px 3px rgba(16, 24, 40, .04);
  --shadow-md: 0 4px 12px -2px rgba(16, 24, 40, .08), 0 2px 4px -2px rgba(16, 24, 40, .04);
  --shadow-lg: 0 24px 48px -12px rgba(16, 24, 40, .18);
  --ring: 0 0 0 3px rgba(var(--accent-rgb), .16);

  /* ── FONTS: the Windows system font (Segoe UI), with system fallbacks on other platforms.
     --font-jakarta is kept for reference; to try it again set --font to var(--font-jakarta). ── */
  --font-system: "Segoe UI Variable Text", "Segoe UI", system-ui, -apple-system, "Helvetica Neue", Arial, sans-serif;
  --font-jakarta: "Plus Jakarta Sans Variable", "Plus Jakarta Sans", "Segoe UI Variable Text", "Segoe UI", system-ui, sans-serif;
  --font: var(--font-system);
  --mono: "Cascadia Mono", Consolas, "SFMono-Regular", ui-monospace, monospace;

  font-family: var(--font);
  letter-spacing: .005em;
  color: var(--ink);
  background: var(--bg);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  font-feature-settings: "cv11", "ss01";
}
* { box-sizing: border-box; }
html, body, #root { margin: 0; height: 100%; }
body { font-size: 15px; line-height: 1.5; background: var(--bg); }
button, input, select, textarea { font: inherit; color: inherit; }
a { color: var(--accent); }
::selection { background: var(--accent-100); }
:focus-visible { outline: 2px solid var(--accent-400); outline-offset: 2px; }
::-webkit-scrollbar { width: 10px; height: 10px; }
::-webkit-scrollbar-thumb { background: var(--scroll); border-radius: 99px; border: 3px solid transparent; background-clip: content-box; }
::-webkit-scrollbar-thumb:hover { background-color: var(--scroll-hover); }
::-webkit-scrollbar-track { background: transparent; }

.grow { flex: 1; }
.muted { color: var(--muted); }
.small { font-size: 13px; }
.mono { font-family: var(--mono); font-size: 13px; letter-spacing: -.01em; }
.num { font-family: var(--mono); font-variant-numeric: tabular-nums; letter-spacing: -.02em; }
.ellipsis { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; min-width: 0; }
.ico { flex: none; display: block; }
.err-t { color: var(--red); }
.ok-t { color: var(--green-600); display: inline-flex; gap: 4px; align-items: center; }

/* ================================================================== shell */
.app { display: grid; grid-template-columns: 256px minmax(0, 1fr); height: calc(100 * var(--vh, 1vh)); }

/* ---- sidebar */
.sidebar {
  background: var(--surface); border-right: 1px solid var(--line);
  display: flex; flex-direction: column; gap: 20px; padding: 18px 14px 14px; overflow: auto; min-height: 0;
}
.brand { display: flex; gap: 11px; align-items: center; padding: 2px 6px 4px; text-decoration: none; color: inherit; border-radius: 10px; }
.brand-mark { width: 34px; height: 34px; border-radius: 10px; display: grid; place-items: center; color: #fff; flex: none;
  background: var(--grad);
  box-shadow: 0 4px 10px -2px rgba(var(--accent-rgb), .45), inset 0 1px 0 rgba(255, 255, 255, .25); }
.brand b { display: block; font-size: 15.5px; font-weight: 650; letter-spacing: -.1px; color: var(--ink); line-height: 1.25; }
.brand span { font-size: 13px; color: var(--muted); }

.side-label { padding: 0 10px 6px; font-size: 12px; font-weight: 600; letter-spacing: .5px; text-transform: uppercase; color: var(--faint); }
.nav { display: grid; gap: 2px; }
.nav a, .nav button { display: flex; align-items: center; gap: 10px; padding: 8px 10px; border-radius: var(--radius-sm); color: var(--ink-2);
  text-decoration: none; background: none; border: 0; cursor: pointer; text-align: left; font-weight: 500; font-size: 14.5px;
  transition: background .12s, color .12s; }
.nav a .ico, .nav button .ico { color: var(--faint); transition: color .12s; }
.nav a:hover, .nav button:hover:not(:disabled) { background: var(--surface-3); color: var(--ink); }
.nav a:hover .ico, .nav button:hover:not(:disabled) .ico { color: var(--ink-2); }
.nav a.on { background: var(--accent-50); color: var(--accent-600); font-weight: 600; }
.nav a.on .ico { color: var(--accent); }
.nav button:disabled { opacity: .45; cursor: default; }
.nav-n { margin-left: auto; font-size: 12px; font-weight: 600; background: var(--surface-3); color: var(--ink-2); padding: 1px 8px; border-radius: 99px; }
.nav a.on .nav-n { background: var(--surface); color: var(--accent-600); box-shadow: 0 0 0 1px var(--accent-100); }

.side-sec { display: flex; flex-direction: column; min-height: 0; flex: 1; }
.side-h { display: flex; align-items: center; justify-content: space-between; padding: 0 6px 8px 10px; font-size: 12px; font-weight: 600;
  letter-spacing: .5px; text-transform: uppercase; color: var(--faint); }
.side-new { display: inline-flex; align-items: center; gap: 4px; background: var(--surface); border: 1px solid var(--line); color: var(--ink-2);
  border-radius: 7px; padding: 3px 8px; font-size: 13px; cursor: pointer; text-transform: none; letter-spacing: 0; font-weight: 600;
  box-shadow: var(--shadow-xs); transition: border-color .12s, background .12s; }
.side-new:hover { border-color: var(--line-strong); background: var(--surface-2); }
.side-new.icon { padding: 4px; border-color: transparent; box-shadow: none; background: none; color: var(--muted); }
.side-new.icon:hover { background: var(--surface-3); color: var(--ink); }
.joblist { display: grid; gap: 2px; overflow: auto; min-height: 0; align-content: start; }
.jobitem { display: flex; gap: 10px; align-items: center; padding: 7px 10px; border-radius: var(--radius-sm); background: none; border: 1px solid transparent;
  cursor: pointer; color: var(--ink-2); text-align: left; min-width: 0; transition: background .12s; }
.jobitem:hover { background: var(--surface-3); }
.jobitem.on { background: var(--surface); border-color: var(--line); box-shadow: var(--shadow); color: var(--ink); }
.ji-t { display: grid; min-width: 0; flex: 1; }
.ji-t b { font-weight: 600; font-size: 14px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.ji-t small { font-size: 12.5px; color: var(--faint); font-family: var(--mono); overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.side-empty { font-size: 13.5px; color: var(--muted); padding: 10px; border: 1px dashed var(--line); border-radius: var(--radius-sm); }
.side-ep { background: var(--surface-2); border: 1px solid var(--line); border-radius: var(--radius); padding: 10px 4px 8px; }
.side-ep .side-h { padding-bottom: 4px; }
.ep-line { display: flex; align-items: center; gap: 9px; padding: 4px 10px; font-size: 14px; color: var(--ink-2); font-weight: 500; }
.ep-line small { margin-left: auto; color: var(--muted); font-size: 12.5px; font-variant-numeric: tabular-nums; font-weight: 400; }

.dot { width: 8px; height: 8px; border-radius: 50%; background: var(--line-strong); flex: none; }
.dot.ok { background: var(--green); box-shadow: 0 0 0 3px rgba(16, 185, 129, .16); }
.dot.bad { background: #f87171; box-shadow: 0 0 0 3px rgba(248, 113, 113, .16); }
.dot.run { background: var(--accent); animation: pulse 1.3s infinite; }
.dot.muted { background: var(--line); }
@keyframes pulse { 0% { box-shadow: 0 0 0 0 rgba(var(--accent-rgb), .35); } 70% { box-shadow: 0 0 0 6px rgba(var(--accent-rgb), 0); } 100% { box-shadow: 0 0 0 0 rgba(var(--accent-rgb), 0); } }

/* ---- main column */
.main { display: flex; flex-direction: column; min-width: 0; height: calc(100 * var(--vh, 1vh)); }
.topbar { display: flex; align-items: center; gap: 10px; padding: 14px 28px; background: rgba(var(--paper-rgb), .85);
  backdrop-filter: saturate(1.4) blur(8px); border-bottom: 1px solid var(--line); min-height: 68px; }
.tb-title { min-width: 0; }
.crumbs { font-size: 13px; color: var(--muted); display: flex; align-items: center; gap: 6px; }
.crumbs .sep { color: var(--line-strong); }
.crumbs .mono { font-size: 12.5px; color: var(--ink-2); background: var(--surface-3); padding: 0 6px; border-radius: 5px; }
.tb-title h1 { margin: 2px 0 0; font-size: 20.5px; font-weight: 650; letter-spacing: -.3px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.pill { display: inline-flex; align-items: center; gap: 6px; font-size: 13px; font-weight: 600; padding: 4px 10px; border-radius: 99px;
  background: var(--surface); color: var(--ink-2); border: 1px solid var(--line); white-space: nowrap; }
.pill::before { content: ""; width: 6px; height: 6px; border-radius: 50%; background: var(--faint); }
.pill.mode::before { display: none; }
.pill.ok { background: var(--green-50); color: var(--green-600); border-color: #c8eedd; }
.pill.ok::before { background: var(--green); }
.pill.bad { background: var(--red-50); color: var(--red); border-color: #fbd5d5; }
.pill.bad::before { background: var(--red); }
.pill.run { background: var(--accent-50); color: var(--accent-600); border-color: var(--accent-100); }
.pill.run::before { background: var(--accent); animation: pulse 1.3s infinite; }
.tabs-top { display: flex; gap: 2px; background: var(--surface-3); padding: 3px; border-radius: 10px; margin-left: 8px; border: 1px solid var(--line-2); }
.tabs-top a { display: inline-flex; align-items: center; gap: 6px; padding: 6px 14px; border-radius: 7px; text-decoration: none; color: var(--muted);
  font-weight: 600; font-size: 14px; transition: color .12s, background .12s; }
.tabs-top a:hover { color: var(--ink); }
.tabs-top a.on { background: var(--surface); color: var(--ink); box-shadow: 0 1px 2px rgba(16, 24, 40, .08), 0 0 0 1px rgba(16, 24, 40, .03); }
.tabs-top a.on .ico { color: var(--accent); }
.content { flex: 1; min-height: 0; overflow: auto; padding: 22px 28px 26px; }
.content.fit { overflow: hidden; }

/* ================================================================== cards and controls */
.card { background: var(--surface); border: 1px solid var(--line); border-radius: var(--radius-lg); box-shadow: var(--shadow);
  display: flex; flex-direction: column; min-height: 0; min-width: 0; }
.card-hd { display: flex; align-items: center; gap: 12px; padding: 12px 18px; border-bottom: 1px solid var(--line-2); min-height: 60px; }
.card-t { display: grid; min-width: 0; }
.card-t h2 { margin: 0; font-size: 15.5px; font-weight: 650; letter-spacing: -.1px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.card-t small { color: var(--muted); font-size: 13.5px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.card-ico { width: 34px; height: 34px; border-radius: 10px; display: grid; place-items: center; background: var(--accent-50); color: var(--accent);
  flex: none; box-shadow: inset 0 0 0 1px var(--accent-100); }
.card-ico.indigo { background: var(--violet-50); color: var(--violet); box-shadow: inset 0 0 0 1px var(--violet-100); }
.card-ico.green { background: var(--green-50); color: var(--green-600); box-shadow: inset 0 0 0 1px var(--green-200); }
.card-ico.sm { width: 28px; height: 28px; border-radius: 8px; }
.step-n { font-size: 11.5px; font-weight: 700; color: var(--accent); letter-spacing: .5px; text-transform: uppercase; line-height: 1.3; }
.card-bd { padding: 16px 18px 18px; overflow: auto; overflow-x: hidden; flex: 1; min-height: 0; display: grid; gap: 16px; align-content: start; }
.card-bd.tight { gap: 10px; }

.btn { display: inline-flex; align-items: center; justify-content: center; gap: 7px; height: 36px; padding: 0 14px; border-radius: 9px;
  border: 1px solid var(--line); background: var(--surface); color: var(--ink-2); font-weight: 600; font-size: 14px; cursor: pointer;
  text-decoration: none; white-space: nowrap; box-shadow: var(--shadow-xs);
  transition: background .12s, border-color .12s, box-shadow .12s, color .12s, transform .06s; }
.btn:hover:not(:disabled) { border-color: var(--line-strong); background: var(--surface-2); color: var(--ink); }
.btn:active:not(:disabled) { transform: translateY(.5px); }
.btn.primary { background: var(--grad); border-color: transparent; color: #fff;
  box-shadow: 0 1px 2px rgba(16, 24, 40, .08), 0 2px 8px -2px rgba(var(--accent-rgb), .5), inset 0 1px 0 rgba(255, 255, 255, .14); }
.btn.primary:hover:not(:disabled) { background: var(--grad-hover); border-color: transparent; color: #fff; }
.btn.soft { background: var(--accent-50); border-color: var(--accent-100); color: var(--accent-600); box-shadow: none; }
.btn.soft:hover:not(:disabled) { background: var(--accent-100); border-color: var(--accent-200); color: var(--accent-600); }
.btn.ghost { background: transparent; border-color: transparent; color: var(--ink-2); box-shadow: none; }
.btn.ghost:hover:not(:disabled) { background: var(--surface-3); border-color: transparent; }
.btn.danger { color: var(--red); }
.btn.danger:hover:not(:disabled) { background: var(--red-50); color: var(--red); }
.btn.small { height: 30px; padding: 0 10px; font-size: 13.5px; border-radius: 8px; gap: 5px; }
.btn.wide { width: 100%; }
.btn:disabled { opacity: .5; cursor: not-allowed; box-shadow: none; }
.icon-btn { display: grid; place-items: center; width: 30px; height: 30px; border-radius: 8px; border: 0; background: none; color: var(--muted);
  cursor: pointer; transition: background .12s, color .12s; }
.icon-btn:hover { background: var(--surface-3); color: var(--ink); }
.link { background: none; border: 0; padding: 0; color: var(--accent); font-weight: 600; cursor: pointer; display: inline-flex; align-items: center; gap: 4px; font-size: 13.5px; }
.link:hover { color: var(--accent-600); text-decoration: underline; text-underline-offset: 2px; }
.vsep { width: 1px; height: 24px; background: var(--line); margin: 0 4px; }
.btn-group { display: inline-flex; border: 1px solid var(--line); border-radius: 9px; background: var(--surface); box-shadow: var(--shadow-xs); overflow: hidden; }
.btn-group .btn { border: 0; border-radius: 0; box-shadow: none; height: 34px; }
.btn-group .btn + .btn { border-left: 1px solid var(--line-2); }

.field { display: grid; gap: 6px; min-width: 0; align-content: start; }
.field-l { font-size: 13.5px; font-weight: 600; color: var(--ink-2); }
.field-h { font-size: 13px; color: var(--muted); overflow-wrap: anywhere; line-height: 1.4; }
.span2 { grid-column: span 2; }
input:not([type=checkbox]):not([type=file]), select, textarea {
  width: 100%; height: 36px; padding: 0 11px; border: 1px solid var(--line-strong); border-radius: 9px; background: var(--surface); outline: none; min-width: 0;
  box-shadow: var(--shadow-xs); transition: border-color .12s, box-shadow .12s; }
textarea { height: auto; padding: 9px 11px; resize: vertical; line-height: 1.5; }
input::placeholder, textarea::placeholder { color: var(--faint); }
input:hover:not(:focus):not(:disabled), select:hover:not(:focus):not(:disabled), textarea:hover:not(:focus):not(:disabled) { border-color: var(--scroll-hover); }
input:focus, select:focus, textarea:focus { border-color: var(--accent-400); box-shadow: var(--ring); }
select { appearance: none; padding-right: 32px; cursor: pointer;
  background: var(--surface) url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='%236b7280' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'%3E%3Cpath d='m7 10 5 5 5-5'/%3E%3C/svg%3E") no-repeat right 9px center; }
input.mini { width: 64px !important; height: 30px !important; padding: 0 8px !important; text-align: center; }
select.compact { width: auto; height: 34px; font-size: 14px; }
.grid2 { display: grid; grid-template-columns: minmax(0, 1fr) minmax(0, 1fr); gap: 14px; }
.grid3 { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 16px; }

.switch { width: 38px; height: 22px; border-radius: 99px; border: 0; background: var(--switch-off); position: relative; cursor: pointer; flex: none;
  transition: background .18s; padding: 0; box-shadow: inset 0 1px 2px rgba(16, 24, 40, .08); }
.switch span { position: absolute; top: 3px; left: 3px; width: 16px; height: 16px; border-radius: 50%; background: #fff;
  box-shadow: 0 1px 3px rgba(16, 24, 40, .2); transition: left .18s cubic-bezier(.3, .7, .4, 1); }
.switch:hover:not(:disabled) { background: var(--switch-off-hover); }
.switch.on, .switch.on:hover:not(:disabled) { background: var(--grad); }
.switch.on span { left: 19px; }
.switch:disabled { opacity: .5; cursor: not-allowed; }

.seg { display: inline-flex; background: var(--surface-3); border-radius: 9px; padding: 3px; gap: 2px; width: 100%; border: 1px solid var(--line-2); }
.seg button { flex: 1; border: 0; background: none; height: 28px; padding: 0 10px; border-radius: 7px; font-weight: 600; font-size: 13.5px; color: var(--muted);
  cursor: pointer; white-space: nowrap; transition: color .12s, background .12s; }
.seg button:hover { color: var(--ink); }
.seg button.on { background: var(--surface); color: var(--accent-600); box-shadow: 0 1px 2px rgba(16, 24, 40, .1), 0 0 0 1px rgba(16, 24, 40, .03); }
.seg.small { width: auto; margin-bottom: 6px; }
.seg.small button { height: 24px; padding: 0 10px; font-size: 13px; }

.chips { display: flex; flex-wrap: wrap; gap: 6px; }
.chip { display: inline-flex; align-items: center; gap: 5px; height: 28px; padding: 0 11px; border-radius: 99px; border: 1px solid var(--line);
  font-size: 13.5px; font-weight: 500; color: var(--ink-2); background: var(--surface); cursor: pointer; user-select: none; transition: all .12s; }
.chip:hover { border-color: var(--line-strong); background: var(--surface-2); }
.chip.on { background: var(--accent-50); border-color: var(--accent-200); color: var(--accent-600); font-weight: 600; }
.chip.static { cursor: default; font-size: 12.5px; height: 22px; padding: 0 9px; background: var(--surface-3); border-color: transparent; color: var(--ink-2); }
.chip-x { border: 0; background: none; display: grid; place-items: center; padding: 0 0 0 2px; color: inherit; cursor: pointer; opacity: .6; }
.chip-x:hover { opacity: 1; }
.chip-note { font-weight: 400; }
.tag { display: inline-flex; align-items: center; gap: 4px; font-size: 12px; font-weight: 600; padding: 2px 8px; border-radius: 6px;
  background: var(--surface-3); color: var(--ink-2); white-space: nowrap; line-height: 1.5; }
.tag.ok { background: var(--green-50); color: var(--green-600); }
.tag.bad { background: var(--red-50); color: var(--red); }
.tag.run { background: var(--accent-50); color: var(--accent-600); }
.tag.muted { background: var(--surface-3); color: var(--faint); }

/* ================================================================== generate */
.generate { height: 100%; display: grid; grid-template-rows: minmax(0, 1fr) auto; gap: 18px; }
.cols { border: 0; margin: 0; padding: 0; min-width: 0; min-height: 0; display: grid; gap: 18px;
  grid-template-columns: minmax(0, .95fr) minmax(0, .9fr) minmax(0, 1.2fr); grid-template-rows: minmax(0, 1fr); }
.cols:disabled .card-bd { opacity: .55; pointer-events: none; }

.drop { display: flex; gap: 14px; align-items: center; padding: 18px; border: 1.5px dashed var(--line-strong); border-radius: var(--radius);
  background: var(--surface-2); cursor: pointer; transition: border-color .15s, background .15s; }
.drop:hover, .drop.drag { border-color: var(--accent-400); background: var(--accent-50); }
.drop-ico { width: 42px; height: 42px; border-radius: 11px; display: grid; place-items: center; background: var(--surface); color: var(--accent);
  box-shadow: var(--shadow-md); border: 1px solid var(--line); flex: none; transition: transform .15s; }
.drop:hover .drop-ico, .drop.drag .drop-ico { transform: translateY(-1px); }
.drop b { color: var(--ink); font-weight: 600; } .drop u { color: var(--accent); text-decoration: none; font-weight: 600; }
.drop small { display: block; color: var(--muted); font-size: 13px; margin-top: 2px; }
.files { display: grid; gap: 6px; max-height: 132px; overflow: auto; }
.file { display: flex; gap: 10px; align-items: center; padding: 8px 10px; border: 1px solid var(--line); border-radius: 10px; background: var(--surface);
  text-align: left; min-width: 0; overflow: hidden; transition: border-color .12s, box-shadow .12s, background .12s; }
.file.pick { cursor: pointer; width: 100%; }
.file.pick:hover { border-color: var(--line-strong); background: var(--surface-2); }
.file.on, .file.on:hover { border-color: var(--accent-400); background: var(--surface); box-shadow: var(--ring); }
.file-n { display: grid; flex: 1; min-width: 0; font-weight: 600; font-size: 14px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.file-n small { font-weight: 400; color: var(--muted); font-size: 12.5px; overflow: hidden; text-overflow: ellipsis; }
.ext { font-size: 10.5px; font-weight: 700; letter-spacing: .3px; width: 36px; text-align: center; padding: 5px 0; border-radius: 7px; background: var(--surface-3);
  color: var(--ink-2); flex: none; overflow: hidden; }
.ext.pdf { background: var(--red-50); color: var(--red); } .ext.docx { background: var(--blue-50); color: var(--blue); }
.ext.html { background: var(--orange-50); color: var(--orange); } .ext.md, .ext.txt { background: var(--green-50); color: var(--green-600); }

.models { border: 1px solid var(--line); border-radius: var(--radius); padding: 12px 14px; display: grid; gap: 8px; background: var(--surface-2); }
.models-h { display: flex; align-items: center; justify-content: space-between; font-size: 13.5px; font-weight: 600; color: var(--ink-2); }
.models-h span { display: inline-flex; gap: 7px; align-items: center; }
.models-h .ico { color: var(--muted); }
.model-row { display: grid; grid-template-columns: 10px 58px minmax(0, 1fr) auto; gap: 9px; align-items: center; font-size: 14px;
  padding: 7px 10px; background: var(--surface); border: 1px solid var(--line-2); border-radius: 9px; }
.model-row b { font-weight: 600; }
.model-row .mono { color: var(--muted); font-size: 13px; }
.models-f { display: flex; align-items: center; gap: 8px; justify-content: space-between; padding-top: 2px; }

.feature { display: grid; grid-template-columns: 38px minmax(0, 1fr) auto; gap: 12px; align-items: center; padding: 12px 14px;
  border: 1px solid var(--line); border-radius: var(--radius); background: var(--surface-2); transition: background .15s, border-color .15s; }
.feature.on { background: linear-gradient(135deg, var(--accent-50), var(--tint-0) 60%, var(--surface)); border-color: var(--accent-100); }
.feature-ico { width: 38px; height: 38px; border-radius: 10px; display: grid; place-items: center; background: var(--surface); color: var(--muted);
  border: 1px solid var(--line); transition: all .15s; }
.feature.on .feature-ico { background: var(--grad); color: #fff; border-color: transparent; box-shadow: 0 4px 10px -3px rgba(var(--accent-rgb), .5); }
.feature b { display: block; font-size: 14.5px; font-weight: 600; } .feature small { color: var(--muted); font-size: 13px; display: block; line-height: 1.4; }

.split3 { display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); gap: 8px; }
.inset { position: relative; }
.inset small { position: absolute; top: 6px; left: 11px; font-size: 11.5px; color: var(--muted); font-weight: 600; pointer-events: none; letter-spacing: .2px; }
.inset input { height: 46px !important; padding: 18px 11px 4px !important; font-weight: 600; font-variant-numeric: tabular-nums; }

.tile { border: 1px solid var(--line); border-radius: var(--radius); padding: 10px 12px; display: grid; gap: 9px; background: var(--surface);
  transition: border-color .15s, box-shadow .15s, background .15s; }
.tile:hover { border-color: var(--line-strong); }
.tile.on { border-color: var(--accent-200); background: linear-gradient(180deg, var(--tint-0), var(--surface)); box-shadow: 0 0 0 3px rgba(var(--accent-rgb), .06); }
.tile-top { display: grid; grid-template-columns: 32px minmax(0, 1fr) auto; gap: 12px; align-items: center; }
.tile-ico { width: 32px; height: 32px; border-radius: 9px; display: grid; place-items: center; background: var(--surface-3); color: var(--muted); transition: all .15s; }
.tile.on .tile-ico { background: var(--grad); color: #fff; box-shadow: 0 4px 10px -3px rgba(var(--accent-rgb), .5); }
.tile-t b { display: block; font-size: 14.5px; font-weight: 600; }
.tile-t small { display: block; color: var(--muted); font-size: 13px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.tile-bd { display: grid; gap: 9px; padding: 10px 0 2px; border-top: 1px dashed var(--line); }
.tile-bd .chips { gap: 5px; }
.tile-bd .chip { height: 26px; padding: 0 10px; font-size: 13px; }
.inline { display: flex; align-items: center; gap: 8px; font-size: 13.5px; color: var(--ink-2); flex-wrap: wrap; }
.addtopic { display: flex; gap: 6px; flex: 1; min-width: 180px; margin-left: 8px; }
.addtopic input { height: 30px !important; padding: 0 10px !important; font-size: 13.5px; }

.mixbox { border: 1px solid var(--line); border-radius: var(--radius); padding: 12px 14px; display: grid; gap: 9px; background: var(--surface-2); }
.mixbox-h { display: flex; justify-content: space-between; align-items: center; font-size: 13.5px; }
.mixbox-h b { font-weight: 600; }
.mix { display: grid; gap: 8px; }
.mixbar { display: flex; height: 8px; border-radius: 99px; overflow: hidden; background: var(--line-2); gap: 2px; }
.mixbar i { display: block; height: 100%; transition: width .3s; border-radius: 99px; }
.mixleg { display: flex; flex-wrap: wrap; gap: 4px 14px; font-size: 13px; color: var(--muted); }
.mixleg b { color: var(--ink); font-weight: 600; font-variant-numeric: tabular-nums; }
.mixleg span::before { content: ""; display: inline-block; width: 8px; height: 8px; border-radius: 3px; margin-right: 6px; background: var(--c, currentColor); }

.warn { display: flex; gap: 10px; align-items: flex-start; font-size: 13.5px; color: var(--amber-ink); background: var(--amber-50);
  border: 1px solid var(--amber-200); border-radius: var(--radius); padding: 10px 12px; line-height: 1.45; }
.warn .ico { margin-top: 1px; color: var(--amber); }
.warn b { color: var(--amber-ink-2); font-weight: 600; }
.warn.block { margin-bottom: 18px; padding: 12px 16px; }
.warn ul { margin: 4px 0 0; padding-left: 18px; }
.small-warn { margin-top: 10px; font-size: 13px; }
.info { display: flex; gap: 9px; align-items: center; font-size: 14px; color: var(--accent-600); background: var(--accent-50);
  border: 1px solid var(--accent-100); border-radius: var(--radius); padding: 10px 14px; margin-bottom: 16px; }
.info.small { font-size: 13px; margin: 0; padding: 8px 12px; }
.error { font-size: 13.5px; background: var(--red-50); color: var(--red-ink); border: 1px solid var(--red-200); border-radius: 10px; padding: 9px 12px; white-space: pre-wrap; }

/* ---- run panel (docked at the bottom of Generate) */
.runpanel { padding: 14px 18px 16px; gap: 12px; box-shadow: var(--shadow-md); }
.run-top { display: flex; align-items: center; gap: 8px; }
.run-dot { width: 36px; height: 36px; border-radius: 10px; flex: none; margin-right: 4px; display: grid; place-items: center;
  background: var(--surface-3); color: var(--faint); position: relative; }
.run-dot::after { content: ""; width: 10px; height: 10px; border-radius: 50%; background: currentColor; }
.run-dot.run { background: var(--accent-50); color: var(--accent); }
.run-dot.run::after { animation: pulse 1.3s infinite; }
.run-dot.ok { background: var(--green-50); color: var(--green); }
.run-dot.bad { background: var(--red-50); color: var(--red); }
.run-t { display: grid; min-width: 0; }
.run-t b { font-size: 15px; font-weight: 650; } .run-t span { color: var(--muted); font-size: 13.5px; font-variant-numeric: tabular-nums; }
.track { display: grid; gap: 6px; }
.step { display: grid; gap: 6px; min-width: 0; }
.step > b { display: block; height: 5px; border-radius: 99px; background: var(--line-2); overflow: hidden; }
.step > b > i { display: block; height: 100%; width: 0; background: var(--accent); border-radius: 99px; transition: width .3s; }
.step.now > b > i { background: linear-gradient(90deg, var(--grad-a), var(--grad-b)); }
.step.done > b > i { width: 100%; background: var(--green); }
.step.fail > b > i { width: 100%; background: var(--red); }
.step.skip > b { background: repeating-linear-gradient(45deg, var(--line-2) 0 4px, var(--surface) 4px 8px); }
.step > span { display: flex; align-items: center; gap: 3px; font-size: 12px; font-weight: 500; color: var(--faint); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.step.done > span { color: var(--green-600); } .step.now > span { color: var(--accent); font-weight: 600; }
.step.fail > span { color: var(--red); font-weight: 600; } .step.skip > span { color: var(--faint); }

/* ================================================================== modal / drawer */
.overlay { position: fixed; inset: 0; background: rgba(17, 24, 39, .32); backdrop-filter: blur(3px); z-index: 50; display: grid; place-items: center;
  padding: 24px; animation: fade .15s ease-out; }
@keyframes fade { from { opacity: 0; } }
.modal { background: var(--surface); border-radius: var(--radius-lg); box-shadow: var(--shadow-lg); border: 1px solid var(--line);
  width: min(640px, 100%); max-height: calc(100 * var(--vh, 1vh) - 48px); display: flex; flex-direction: column; animation: pop .18s cubic-bezier(.2, .8, .3, 1); }
@keyframes pop { from { transform: translateY(8px) scale(.985); opacity: 0; } }
.modal.wide { width: min(1020px, 100%); }
.modal-hd { display: flex; gap: 12px; align-items: flex-start; justify-content: space-between; padding: 18px 22px; border-bottom: 1px solid var(--line-2); }
.modal-hd h2 { margin: 0; font-size: 18.5px; font-weight: 650; letter-spacing: -.2px; }
.modal-hd p { margin: 3px 0 0; color: var(--muted); font-size: 14px; max-width: 760px; }
.modal-bd { padding: 20px 22px; overflow: auto; min-height: 0; }
.modal-ft { display: flex; gap: 8px; align-items: center; justify-content: flex-end; padding: 14px 22px; border-top: 1px solid var(--line-2);
  flex-wrap: wrap; background: var(--surface-2); border-radius: 0 0 var(--radius-lg) var(--radius-lg); }
.adv { display: grid; grid-template-columns: 220px minmax(0, 1fr); gap: 24px; min-height: 400px; }
.adv-nav { display: grid; gap: 2px; align-content: start; padding-right: 16px; border-right: 1px solid var(--line-2); }
.adv-nav button { display: flex; gap: 10px; align-items: center; padding: 9px 11px; border-radius: var(--radius-sm); border: 0; background: none; cursor: pointer;
  text-align: left; color: var(--ink-2); font-weight: 500; font-size: 14px; transition: background .12s, color .12s; }
.adv-nav button .ico { color: var(--faint); }
.adv-nav button:hover { background: var(--surface-3); }
.adv-nav button.on { background: var(--accent-50); color: var(--accent-600); font-weight: 600; }
.adv-nav button.on .ico { color: var(--accent); }
.adv-bd { display: grid; gap: 16px; align-content: start; }
.lead { margin: 0; color: var(--muted); font-size: 14px; line-height: 1.55; }
.test-row { display: flex; align-items: center; justify-content: space-between; gap: 12px; padding: 12px 14px; background: var(--surface-2);
  border: 1px solid var(--line); border-radius: var(--radius); }
.rules { display: grid; gap: 12px; }
.rule { border: 1px solid var(--line); border-radius: var(--radius); padding: 14px 16px; display: grid; gap: 12px; background: var(--surface); box-shadow: var(--shadow-xs); transition: opacity .15s; }
.rule.off { opacity: .55; background: var(--surface-2); }
.rule-top { display: flex; gap: 12px; align-items: center; }
.rule-name { max-width: 300px; font-weight: 600; }

.drawer-wrap { place-items: stretch end; padding: 0; }
.drawer { width: min(1180px, 94 * var(--vw, 1vw)); height: calc(100 * var(--vh, 1vh)); background: var(--bg); display: flex; flex-direction: column; box-shadow: var(--shadow-lg);
  border-left: 1px solid var(--line); animation: slide .22s cubic-bezier(.2, .8, .3, 1); }
@keyframes slide { from { transform: translateX(32px); opacity: .5; } }
.drawer-hd { display: flex; align-items: center; gap: 16px; padding: 14px 22px; background: var(--surface); border-bottom: 1px solid var(--line); min-height: 64px; }
.drawer-hd h2 { margin: 0; font-size: 17.5px; font-weight: 650; letter-spacing: -.2px; }
.drawer-bd { padding: 22px; overflow: auto; flex: 1; min-height: 0; }

.toasts { position: fixed; right: 22px; bottom: 22px; display: grid; gap: 8px; z-index: 80; }
.toast { display: flex; gap: 10px; align-items: center; background: var(--surface); color: var(--ink); padding: 11px 14px 11px 12px; border-radius: var(--radius);
  font-size: 14px; font-weight: 500; box-shadow: var(--shadow-lg); border: 1px solid var(--line); max-width: 440px; animation: slide .2s ease-out; }
.toast .ico { width: 22px; height: 22px; padding: 4px; border-radius: 50%; background: var(--green-50); color: var(--green-600); }
.toast.error .ico { background: var(--red-50); color: var(--red); }

/* ================================================================== results */
.results { display: grid; gap: 0; }
.stats { display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); gap: 16px; margin-bottom: 18px; }
.stat { background: var(--surface); border: 1px solid var(--line); border-radius: var(--radius-lg); box-shadow: var(--shadow); padding: 16px 18px;
  display: flex; flex-direction: column; gap: 12px; min-width: 0; }
.stat-ico { width: 34px; height: 34px; border-radius: 10px; display: grid; place-items: center; flex: none; }
.stat-ico.blue { background: var(--accent-50); color: var(--accent); } .stat-ico.green { background: var(--green-50); color: var(--green-600); }
.stat-ico.violet { background: var(--violet-50); color: var(--violet); } .stat-ico.orange { background: var(--orange-50); color: var(--orange); }
.stat-b { display: grid; gap: 2px; min-width: 0; flex: 1; align-content: start; }
.stat-b > b { font-size: 28px; font-weight: 700; letter-spacing: -.3px; line-height: 1.15; font-variant-numeric: tabular-nums; }
.stat-b > span { font-size: 14px; color: var(--ink-2); font-weight: 500; }
.stat-b > small { font-size: 13px; color: var(--muted); }
.stat-b .mix { margin-top: 8px; }

.acceptbar { flex-direction: row; align-items: center; gap: 10px; padding: 14px 18px; margin-bottom: 18px; flex-wrap: wrap;
  background: linear-gradient(100deg, var(--surface) 40%, var(--accent-50)); border-color: var(--accent-100); }
.accept-t { display: grid; gap: 2px; min-width: 0; }
.accept-t b { font-size: 16px; font-weight: 650; }
.res-grid { display: grid; grid-template-columns: minmax(0, 1.65fr) minmax(320px, 1fr); gap: 18px; align-items: start; }
.side-col { display: grid; gap: 16px; }
.panel .card-hd { padding: 12px 16px; min-height: 52px; }
.panel .card-t h2 { font-size: 14.5px; }
.panel .card-bd { padding: 14px 16px 16px; }

.search { display: flex; align-items: center; gap: 7px; border: 1px solid var(--line-strong); border-radius: 9px; padding: 0 10px; color: var(--faint);
  background: var(--surface); box-shadow: var(--shadow-xs); height: 34px; transition: border-color .12s, box-shadow .12s; }
.search:focus-within { border-color: var(--accent-400); box-shadow: var(--ring); }
.search input { border: 0 !important; box-shadow: none !important; padding: 0 !important; height: 32px !important; width: 180px !important; background: none; }

.pilltabs { display: flex; gap: 2px; flex-wrap: nowrap; overflow-x: auto; max-width: 100%; scrollbar-width: none; background: var(--surface-3); padding: 3px; border-radius: 10px; border: 1px solid var(--line-2); width: fit-content; }
.pilltabs button { display: inline-flex; align-items: center; gap: 6px; border: 0; background: none; color: var(--muted); height: 28px; padding: 0 10px; flex: none; white-space: nowrap;
  border-radius: 7px; font-weight: 600; font-size: 13.5px; cursor: pointer; transition: color .12s, background .12s; }
.pilltabs button:hover { color: var(--ink); }
.pilltabs button span { font-size: 12px; color: var(--faint); font-weight: 600; background: var(--surface); padding: 0 6px; border-radius: 99px;
  min-width: 20px; text-align: center; font-variant-numeric: tabular-nums; border: 1px solid var(--line-2); }
.pilltabs button.on { background: var(--surface); color: var(--ink); box-shadow: 0 1px 2px rgba(16, 24, 40, .1), 0 0 0 1px rgba(16, 24, 40, .03); }
.pilltabs button.on span { background: var(--accent-50); color: var(--accent-600); border-color: var(--accent-100); }
.inline-tabs { margin-left: 4px; }

.pair { border: 1px solid var(--line); border-radius: var(--radius); padding: 14px 16px; display: grid; gap: 10px; background: var(--surface);
  transition: box-shadow .15s, border-color .15s; }
.pair:hover { border-color: var(--line-strong); box-shadow: var(--shadow-md); }
.pair.rejected { opacity: .6; background: var(--surface-2); }
.pair-meta { display: flex; gap: 8px; align-items: center; font-size: 13px; color: var(--muted); min-width: 0; }
.pair-meta > .mono:first-child { font-size: 12.5px; color: var(--ink-2); background: var(--surface-3); padding: 1px 6px; border-radius: 5px; }
.tag.t-from_images { background: var(--violet-50); color: var(--violet); }
.tag.t-decline { background: var(--orange-50); color: var(--orange); }
.tag.t-general { background: var(--green-50); color: var(--green-600); }
.state { display: inline-flex; align-items: center; gap: 5px; font-size: 12.5px; font-weight: 600; padding: 2px 9px; border-radius: 99px;
  background: var(--surface-3); color: var(--muted); white-space: nowrap; }
.state::before { content: ""; width: 6px; height: 6px; border-radius: 50%; background: currentColor; opacity: .8; }
.state.ok { background: var(--green-50); color: var(--green-600); } .state.bad { background: var(--red-50); color: var(--red); }

.msg { display: grid; grid-template-columns: 76px minmax(0, 1fr); gap: 12px; align-items: start; }
.msg > span { font-size: 11.5px; font-weight: 700; letter-spacing: .5px; text-transform: uppercase; color: var(--faint); padding-top: 9px; }
.msg p { margin: 0; white-space: pre-wrap; padding: 8px 12px; border-radius: 4px 12px 12px 12px; line-height: 1.55; }
.msg.u p { background: var(--surface-3); color: var(--ink); font-weight: 500; }
.msg.a > span { color: var(--accent); }
.msg.a p { background: var(--accent-50); color: #4C0519; box-shadow: inset 0 0 0 1px var(--accent-100); }
.msg.a p.muted { background: var(--surface-2); color: var(--muted); box-shadow: inset 0 0 0 1px var(--line-2); }
.msg.t > span { color: var(--violet); }
.msg.t p { background: var(--violet-50); font-size: 13.5px; color: var(--violet-ink); box-shadow: inset 0 0 0 1px var(--violet-100); }
.msg textarea { margin: 0; }
.pair-foot { display: flex; gap: 6px; align-items: center; flex-wrap: wrap; padding-top: 10px; border-top: 1px solid var(--line-2); }
.score { font-size: 12px; font-weight: 600; padding: 2px 8px; border-radius: 6px; font-variant-numeric: tabular-nums; }
.score.shi { background: var(--green-50); color: var(--green-600); } .score.smid { background: var(--amber-50); color: var(--amber-ink); } .score.slo { background: var(--red-50); color: var(--red); }
.ctx { font-size: 12.5px; color: var(--muted); display: inline-flex; gap: 4px; align-items: center; flex-wrap: wrap; }
.cite { font-size: 12px; padding: 1px 7px; border-radius: 5px; background: var(--surface-3); color: var(--ink-2); font-family: var(--mono); }
.cite.img { background: var(--violet-50); color: var(--violet); }
.why { font-size: 13px; color: var(--muted); font-style: italic; padding: 6px 10px; border-left: 2px solid var(--line-strong); }
.pair-act { display: flex; gap: 6px; justify-content: flex-end; }
.pager { display: flex; align-items: center; gap: 8px; padding-top: 4px; }

.bars { display: grid; gap: 9px; }
.bar-row { display: grid; grid-template-columns: minmax(0, 150px) minmax(0, 1fr) 46px; gap: 12px; align-items: center; font-size: 13.5px; }
.bar-l { color: var(--ink-2); overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.bar-t { height: 8px; background: var(--line-2); border-radius: 99px; overflow: hidden; }
.bar-t i { display: block; height: 100%; background: var(--accent); border-radius: 99px; transition: width .4s; }
.bar-v { font-weight: 600; color: var(--ink); text-align: right; font-variant-numeric: tabular-nums; }
.scorebars { display: grid; gap: 9px; }
.scorebars .bar-row { grid-template-columns: 100px minmax(0, 1fr); }
.stack { display: flex; height: 18px; border-radius: 6px; overflow: hidden; background: var(--line-2); gap: 1px; }
.stack i { display: grid; place-items: center; font-style: normal; font-size: 11px; font-weight: 600; }
.score-leg { display: flex; gap: 12px; font-size: 12px; color: var(--muted); padding: 4px 0 0 112px; }
.score-leg span::before { content: ""; display: inline-block; width: 9px; height: 9px; border-radius: 3px; margin-right: 5px; background: var(--c, currentColor); vertical-align: -1px; }

.tbl { width: 100%; border-collapse: separate; border-spacing: 0; font-size: 13.5px; }
.tbl th { text-align: left; font-size: 12px; color: var(--muted); font-weight: 600; text-transform: uppercase; letter-spacing: .4px;
  padding: 8px 10px; border-bottom: 1px solid var(--line); background: var(--surface-2); }
.tbl th:first-child { border-top-left-radius: 8px; } .tbl th:last-child { border-top-right-radius: 8px; }
.tbl td { padding: 8px 10px; border-bottom: 1px solid var(--line-2); vertical-align: top; font-variant-numeric: tabular-nums; }
.tbl tbody tr:hover td { background: var(--surface-2); }
.tbl tbody tr:last-child td { border-bottom: 0; }
.tbl td:first-child { max-width: 220px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.tbl tr.bad td { color: var(--red); font-weight: 600; }

.empty { display: grid; justify-items: center; gap: 6px; padding: 72px 20px; color: var(--muted); text-align: center; }
.empty b { color: var(--ink); font-size: 16px; font-weight: 600; }
.empty-ico { width: 56px; height: 56px; border-radius: 16px; display: grid; place-items: center; background: var(--surface); color: var(--accent);
  margin-bottom: 6px; border: 1px solid var(--line); box-shadow: var(--shadow-md); }

/* ================================================================== source data */
.src-grid { display: grid; grid-template-columns: 340px minmax(0, 1fr); gap: 18px; align-items: start; }
.src-col { display: grid; gap: 18px; min-width: 0; }
.src-col .card-bd { gap: 8px; }
.sec-row { display: flex; justify-content: space-between; gap: 10px; width: 100%; padding: 7px 10px; border: 0; border-radius: 7px; background: none;
  cursor: pointer; text-align: left; font-size: 14px; color: var(--ink-2); transition: background .12s, color .12s; }
.sec-row:hover { color: var(--accent-600); background: var(--accent-50); }
div.md { background: var(--surface-2); border: 1px solid var(--line); border-radius: var(--radius); padding: 14px 16px; font: 13.5px/1.65 var(--mono);
  white-space: pre-wrap; max-height: 460px; overflow: auto; color: var(--ink-2); }
.md-h { color: var(--accent-600); font-weight: 700; }
.md-img { display: block; background: var(--violet-50); color: var(--violet-ink); border-radius: 8px; padding: 8px 10px; margin: 4px 0; border: 1px solid var(--violet-100); }
.imgcard { display: grid; grid-template-columns: 160px minmax(0, 1fr); gap: 16px; padding: 12px; border: 1px solid var(--line); border-radius: var(--radius); background: var(--surface); }
.thumb { border: 1px solid var(--line); border-radius: 9px; background: var(--surface-2); height: 112px; overflow: hidden; }
.thumb img { width: 100%; height: 100%; object-fit: contain; display: block; }
.img-b { display: grid; gap: 6px; min-width: 0; font-size: 13.5px; }
.img-h { display: flex; gap: 8px; align-items: center; flex-wrap: wrap; }
.img-b p { margin: 0; color: var(--ink-2); }
.img-b p small { display: block; font-size: 11.5px; font-weight: 600; text-transform: uppercase; color: var(--faint); letter-spacing: .5px; margin-bottom: 1px; }
.kb { display: grid; gap: 18px; }
.overview { margin: 0; font-size: 15px; color: var(--ink-2); line-height: 1.65; }
.doccards { display: grid; grid-template-columns: repeat(auto-fill, minmax(260px, 1fr)); gap: 14px; }
.doccard { background: var(--surface); border: 1px solid var(--line); border-radius: var(--radius); padding: 14px 16px; display: grid; gap: 8px;
  font-size: 13.5px; box-shadow: var(--shadow); transition: box-shadow .15s, border-color .15s; }
.doccard:hover { border-color: var(--line-strong); box-shadow: var(--shadow-md); }
.doccard > b { font-size: 14.5px; font-weight: 600; }
.doccard p { margin: 0; color: var(--ink-2); line-height: 1.55; }
.kb-grid { display: grid; grid-template-columns: minmax(0, 1fr) minmax(0, 1.2fr); gap: 18px; align-items: start; }
.cov-doc { display: grid; gap: 8px; padding-bottom: 12px; border-bottom: 1px solid var(--line-2); }
.cov-doc:last-child { border-bottom: 0; padding-bottom: 0; }
.cov-doc > b { font-size: 14px; font-weight: 600; }

/* ================================================================== smaller screens */
@media (max-width: 1240px) {
  .cols { grid-template-columns: minmax(0, 1fr) minmax(0, 1fr); grid-template-rows: auto; }
  .cols > .card:last-child { grid-column: 1 / -1; }
  .content.fit { overflow: auto; }
  .generate { height: auto; }
  .card-bd { overflow: visible; }
  .stats { grid-template-columns: repeat(2, minmax(0, 1fr)); }
  .res-grid, .src-grid, .kb-grid { grid-template-columns: minmax(0, 1fr); }
}
@media (max-width: 900px) {
  .app { grid-template-columns: minmax(0, 1fr); }
  .sidebar { display: none; }
  .cols { grid-template-columns: minmax(0, 1fr); }
  .run-top { flex-wrap: wrap; }
  .adv { grid-template-columns: minmax(0, 1fr); }
  .adv-nav { border-right: 0; padding-right: 0; }
  .grid3 { grid-template-columns: minmax(0, 1fr) minmax(0, 1fr); }
  .topbar, .content { padding-left: 16px; padding-right: 16px; }
}
@media (max-height: 760px) and (min-width: 1241px) {
  .content.fit { overflow: auto; }
  .generate { height: auto; }
  .card-bd { overflow: visible; }
}

/* ---- large screens: on displays wider than 1800px everything is drawn about 7% larger
   (text, spacing and icons together, so proportions stay the same). Change --zoom to tune it. */
@media (min-width: 1800px) {
  html { --zoom: 1.07; zoom: var(--zoom); --vh: calc(1vh / var(--zoom)); --vw: calc(1vw / var(--zoom)); }
}























//rich.css
/* Rich layer: depth, texture and finish. Mostly neutral; coral only as a thin accent.
   Loaded last, so it refines styles.css, shell.css and generate.css without replacing them. */

:root {
  --lift-1: 0 1px 2px rgba(16, 24, 40, .05), 0 1px 1px rgba(16, 24, 40, .03);
  --lift-2: 0 1px 2px rgba(16, 24, 40, .04), 0 4px 10px -2px rgba(16, 24, 40, .06), 0 12px 28px -12px rgba(16, 24, 40, .10);
  --lift-3: 0 2px 4px rgba(16, 24, 40, .04), 0 12px 24px -6px rgba(16, 24, 40, .10), 0 28px 56px -24px rgba(16, 24, 40, .16);
  --grad-soft: linear-gradient(135deg, rgba(249, 115, 22, .10), rgba(236, 72, 153, .10));
}

/* ---- canvas: a faint dot grid with a soft warm glow in the top corner */
main.main { background:
  radial-gradient(900px 420px at 100% -10%, rgba(249, 115, 22, .055), transparent 60%),
  radial-gradient(700px 380px at 30% -20%, rgba(236, 72, 153, .035), transparent 60%),
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
.ph::after { background: linear-gradient(90deg, rgba(249, 115, 22, .55), rgba(236, 72, 153, .35) 22%, transparent 55%); }
.ph-ico { position: relative; background: linear-gradient(180deg, var(--surface), var(--surface-2)); border: 0; box-shadow: var(--edge), 0 0 0 1px var(--hairline), var(--lift-2); }
.ph-ico::before { content: ""; position: absolute; inset: -1px; border-radius: inherit; padding: 1px; background: linear-gradient(135deg, rgba(249, 115, 22, .55), rgba(236, 72, 153, .15) 60%, transparent);
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
.dropzone { border: 1.5px dashed var(--line-strong); background:
  radial-gradient(400px 160px at 50% 0%, rgba(249, 115, 22, .05), transparent 70%),
  radial-gradient(var(--dot) 1px, transparent 1.2px) 0 0 / 16px 16px, var(--surface-2);
  padding: 30px 20px 26px; }
.dropzone:hover, .dropzone.drag { border-color: var(--accent-400); background:
  radial-gradient(420px 180px at 50% 0%, rgba(249, 115, 22, .09), transparent 70%),
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
.btn.primary.lg { position: relative; overflow: hidden; }
.btn.primary.lg::after { content: ""; position: absolute; top: 0; bottom: 0; left: -40%; width: 30%; transform: skewX(-20deg);
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, .35), transparent); transition: left .7s ease; }
.btn.primary.lg:hover::after { left: 120%; }

/* ---- summary: frosted header with a faint warm wash, tidy rows */
.summary { background: rgba(var(--paper-rgb), .94); }
.sum-hd { background: radial-gradient(320px 140px at 100% 0%, rgba(249, 115, 22, .08), transparent 70%),
  radial-gradient(260px 120px at 0% 0%, rgba(236, 72, 153, .05), transparent 70%), var(--surface); border-bottom-color: var(--line-2); }
.sum-ico { border: 1px solid var(--hairline); background: linear-gradient(180deg, var(--surface), var(--surface-2)); box-shadow: var(--lift-1); }
.sum-ft { background: linear-gradient(180deg, var(--surface-2), var(--surface-3)); }
.sum-ready { padding: 10px 12px; border-radius: 12px; background: linear-gradient(180deg, var(--green-50), var(--green-50)); border: 1px solid var(--green-200); }
.sum-issues { box-shadow: var(--edge), var(--lift-1); }

/* ---- run view */
.run-hero { background: radial-gradient(500px 200px at 100% 0%, rgba(249, 115, 22, .06), transparent 70%), var(--surface); }
.run-bar { height: 10px; box-shadow: inset 0 1px 2px rgba(16, 24, 40, .08); }
.tl-dot { box-shadow: 0 0 0 4px var(--surface); }


















//landing.css
/* Landing page and logo placeholder. Uses the tokens from styles.css. */

/* ---- logo (the file is public/assets/logo.svg) */
.logo { display: inline-flex; align-items: center; gap: 11px; min-width: 0; color: inherit; text-decoration: none; }
.logo-img { display: block; flex: none; width: auto; max-width: 200px; object-fit: contain; }
.logo-text { display: grid; min-width: 0; line-height: 1.25; }
.logo-text b { font-size: 16px; font-weight: 650; letter-spacing: -.2px; color: var(--ink); }
.logo-text small { font-size: 13px; color: var(--muted); }

/* ---- page */
.landing { height: calc(100 * var(--vh, 1vh)); overflow: auto; background: var(--surface); scroll-behavior: smooth; }
.lp-wrap { max-width: 1160px; margin: 0 auto; padding: 0 32px; }
.lp-eyebrow { display: inline-flex; align-items: center; gap: 6px; font-size: 13.5px; font-weight: 600; color: var(--accent-600); }
.btn.lg { height: 44px; padding: 0 20px; font-size: 15px; border-radius: 10px; }

/* nav */
.lp-nav { position: sticky; top: 0; z-index: 10; background: rgba(var(--paper-rgb), .85); backdrop-filter: saturate(1.4) blur(10px); border-bottom: 1px solid var(--line-2); }
.lp-nav-in { display: flex; align-items: center; gap: 28px; height: 68px; }
.lp-links { display: flex; gap: 4px; margin-left: auto; }
.lp-links a { padding: 7px 12px; border-radius: 8px; color: var(--ink-2); text-decoration: none; font-weight: 500; font-size: 15px; }
.lp-links a:hover { background: var(--surface-3); color: var(--ink); }

/* hero */
.lp-hero { position: relative; overflow: hidden; padding: 84px 0 96px;
  background: radial-gradient(900px 420px at 85% 10%, rgba(var(--grad-a-rgb), .12), transparent 70%),
              radial-gradient(700px 380px at 0% 100%, rgba(var(--grad-b-rgb), .08), transparent 70%), var(--surface); }
.lp-hero::before { content: ""; position: absolute; inset: 0; pointer-events: none; opacity: .5;
  background-image: linear-gradient(var(--line-2) 1px, transparent 1px), linear-gradient(90deg, var(--line-2) 1px, transparent 1px);
  background-size: 44px 44px; mask-image: radial-gradient(ellipse at 50% 0%, #000 30%, transparent 75%); }
.lp-hero-in { position: relative; display: grid; grid-template-columns: minmax(0, 1.05fr) minmax(0, 1fr); gap: 56px; align-items: center; }
.lp-hero-text .lp-eyebrow { padding: 5px 12px; border-radius: 99px; background: var(--accent-50); border: 1px solid var(--accent-100); }
.lp-hero h1 { margin: 20px 0 16px; font-size: 54px; line-height: 1.08; letter-spacing: -.7px; font-weight: 700; }
.lp-hero h1 em { font-style: normal; background: var(--grad); -webkit-background-clip: text; background-clip: text; color: transparent; }
.lp-hero p { margin: 0; font-size: 18.5px; line-height: 1.6; color: var(--muted); max-width: 520px; }
.lp-cta { display: flex; gap: 10px; margin-top: 32px; flex-wrap: wrap; }

/* hero preview */
.lp-preview { position: relative; padding: 10px 28px 36px 0; }
.pv-card { background: var(--surface); border: 1px solid var(--line); border-radius: 18px; box-shadow: 0 30px 60px -20px rgba(16, 24, 40, .22), 0 0 0 6px rgba(var(--paper-rgb), .6);
  padding: 16px 18px 18px; display: grid; gap: 14px; }
.pv-head { display: flex; align-items: center; gap: 12px; padding-bottom: 12px; border-bottom: 1px solid var(--line-2); }
.pv-dots { display: inline-flex; gap: 5px; } .pv-dots i { width: 9px; height: 9px; border-radius: 50%; background: var(--line); }
.pv-files { display: flex; gap: 8px; flex-wrap: wrap; }
.pv-file { display: inline-flex; align-items: center; gap: 8px; padding: 6px 10px 6px 6px; border: 1px solid var(--line); border-radius: 10px; font-size: 13.5px; font-weight: 500; background: var(--surface-2); }
.pv-arrow { display: grid; place-items: center; color: var(--accent-400); transform: rotate(90deg); height: 16px; }
.pv-qa { display: grid; gap: 8px; }
.pv-q, .pv-a { padding: 10px 12px; border-radius: 4px 12px 12px 12px; font-size: 14px; line-height: 1.5; }
.pv-q small, .pv-a small { display: block; font-size: 11px; font-weight: 700; letter-spacing: .5px; text-transform: uppercase; margin-bottom: 2px; }
.pv-q { background: var(--surface-3); font-weight: 500; } .pv-q small { color: var(--faint); }
.pv-a { background: var(--accent-50); color: var(--accent-ink); box-shadow: inset 0 0 0 1px var(--accent-100); } .pv-a small { color: var(--accent); }
.pv-scores { display: flex; gap: 6px; flex-wrap: wrap; }
.pv-stat { position: absolute; right: 0; bottom: 0; display: flex; gap: 10px; align-items: center; background: var(--surface); border: 1px solid var(--line);
  border-radius: 14px; padding: 10px 16px 10px 10px; box-shadow: var(--shadow-lg); }
.pv-stat b { display: block; font-size: 19.5px; letter-spacing: -.3px; } .pv-stat small { color: var(--muted); font-size: 13px; }

/* sections */
.lp-ico { width: 38px; height: 38px; border-radius: 11px; display: grid; place-items: center; background: var(--accent-50); color: var(--accent);
  box-shadow: inset 0 0 0 1px var(--accent-100); flex: none; }
.lp-ico.green { background: var(--green-50); color: var(--green-600); box-shadow: inset 0 0 0 1px #d1f2e4; }
.lp-sec { padding: 88px 0; }
.lp-sec.alt { background: var(--bg); border-top: 1px solid var(--line-2); border-bottom: 1px solid var(--line-2); }
.lp-sec-h { text-align: center; margin-bottom: 44px; }
.lp-sec-h h2 { margin: 8px 0 0; font-size: 34.5px; letter-spacing: -.2px; font-weight: 700; }
.lp-sec h3 { margin: 0 0 6px; font-size: 17.5px; font-weight: 650; letter-spacing: -.2px; }
.lp-sec p { margin: 0; color: var(--muted); font-size: 15px; line-height: 1.6; }

.lp-steps { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 20px; }
.lp-step { padding: 24px; border: 1px solid var(--line); border-radius: 16px; background: var(--surface); box-shadow: var(--shadow); }
.lp-step-top { display: flex; align-items: center; justify-content: space-between; margin-bottom: 20px; }
.lp-step-n { font-size: 14px; font-weight: 700; color: var(--faint); width: 30px; height: 30px; border-radius: 50%; display: grid; place-items: center; border: 1px solid var(--line); }

.lp-features { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 16px; }
.lp-feature { display: flex; gap: 14px; padding: 22px; border-radius: 16px; background: var(--surface); border: 1px solid var(--line); transition: box-shadow .15s, border-color .15s; }
.lp-feature:hover { border-color: var(--line-strong); box-shadow: var(--shadow-md); }

.lp-band { display: flex; align-items: center; justify-content: space-between; gap: 24px; padding: 40px 44px; border-radius: 20px;
  background: linear-gradient(120deg, var(--tint-1), var(--tint-2) 60%, var(--surface)); border: 1px solid var(--accent-100); }
.lp-band h2 { margin: 0 0 6px; font-size: 28px; letter-spacing: -.3px; font-weight: 700; }

.lp-foot { border-top: 1px solid var(--line-2); padding: 26px 0; }
.lp-foot-in { display: flex; align-items: center; justify-content: space-between; gap: 16px; }

@media (max-width: 960px) {
  .lp-hero-in { grid-template-columns: minmax(0, 1fr); }
  .lp-hero h1 { font-size: 41px; }
  .lp-steps, .lp-features { grid-template-columns: minmax(0, 1fr); }
  .lp-band { flex-direction: column; align-items: flex-start; }
  .lp-links { display: none; }
  .lp-nav-in .btn { margin-left: auto; }
}

/* JetBrains Mono for file names and numbers in the preview */
.pv-file, .pv-stat b { font-family: var(--mono); letter-spacing: -.02em; }
















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
.testbar.good { background: var(--green-50); border-color: var(--green-200); color: var(--green-ink); }
.testbar.bad { background: var(--red-50); border-color: var(--red-200); color: var(--red-ink); }

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
.sum-issues { display: grid; gap: 6px; padding: 12px; border-radius: 12px; background: var(--amber-50); border: 1px solid var(--amber-200); }
.sum-issues b { display: flex; align-items: center; gap: 7px; font-size: 13.5px; color: var(--amber-ink); }
.sum-issues button { display: flex; align-items: center; justify-content: space-between; gap: 8px; text-align: left; border: 0; background: none; padding: 2px 0;
  font-size: 13px; color: var(--amber-ink); cursor: pointer; }
.sum-issues button:hover { color: var(--amber-ink-2); text-decoration: underline; }
.sum-note { margin: 0; font-size: 12.5px; color: var(--muted); text-align: center; }

/* ---- run view */
.runview { display: grid; gap: 20px; max-width: 1100px; margin: 0 auto; }
.run-loading { flex-direction: row; align-items: center; gap: 10px; padding: 24px; color: var(--muted); }
.run-hero { padding: 24px 26px; gap: 20px; }

.run-hero-top { display: flex; align-items: center; gap: 16px; }
.run-badge { width: 56px; height: 56px; border-radius: 16px; display: grid; place-items: center; color: #fff; flex: none; background: var(--faint); }
.run-badge.run { background: var(--accent-50); color: var(--accent-600); box-shadow: inset 0 0 0 1px var(--accent-100); }
.run-badge.run .ico { animation: spin 2.4s linear infinite; }
.run-badge.ok { background: var(--green-50); color: var(--green-600); box-shadow: inset 0 0 0 1px var(--green-200); }
.run-badge.bad { background: var(--red-50); color: var(--red); box-shadow: inset 0 0 0 1px var(--red-200); }
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
.btn.dark { background: var(--dark-btn-b); border-color: var(--dark-btn-b); color: var(--dark-btn-ink); }
.btn.dark:hover:not(:disabled) { background: var(--dark-btn-a); border-color: var(--dark-btn-a); color: var(--dark-btn-ink); }
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






















//theme.js
// Light / dark theme. The choice is saved in this browser; until you pick one, the app follows the system setting.
import { useEffect, useState } from "react";

const KEY = "slm-theme";
const media = () => window.matchMedia("(prefers-color-scheme: dark)");
export const savedTheme = () => { try { return localStorage.getItem(KEY); } catch { return null; } };
export const currentTheme = () => savedTheme() || (media().matches ? "dark" : "light");

export function applyTheme(t) {
  const el = document.documentElement;
  el.classList.add("theme-switching");                 // fade colors for a moment while switching
  el.dataset.theme = t;
  window.dispatchEvent(new CustomEvent("slm:theme", { detail: t }));
  setTimeout(() => el.classList.remove("theme-switching"), 450);
}

export function useTheme() {
  const [theme, setTheme] = useState(currentTheme);
  useEffect(() => {
    const onTheme = (e) => setTheme(e.detail);
    const onSystem = () => { if (!savedTheme()) applyTheme(media().matches ? "dark" : "light"); };
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

















//ThemeToggle.jsx
// Sun / moon switch for light and dark mode.
import { useTheme } from "./theme.js";

export default function ThemeToggle() {
  const [theme, toggle] = useTheme();
  const dark = theme === "dark";
  return (
    <button type="button" className={`theme-toggle${dark ? " is-dark" : ""}`} onClick={toggle}
      title={dark ? "Switch to light mode" : "Switch to dark mode"} aria-label={dark ? "Switch to light mode" : "Switch to dark mode"}>
      <span className="tt-knob">
        <svg className="tt-sun" viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round"><circle cx="12" cy="12" r="4" /><path d="M12 2.5v2M12 19.5v2M4.6 4.6 6 6M18 18l1.4 1.4M2.5 12h2M19.5 12h2M4.6 19.4 6 18M18 6l1.4-1.4" /></svg>
        <svg className="tt-moon" viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M20 14.5A8 8 0 0 1 9.5 4a8 8 0 1 0 10.5 10.5z" /></svg>
      </span>
    </button>
  );
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
