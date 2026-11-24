//Logo.jsx
// Application logo, loaded from public/assets/.
//
// To use your own logo, replace public/assets/logo.svg with your file.
// If your file is not an SVG (for example logo.png), put it in public/assets/ and change LOGO_FILE below.
// The landing page and the sidebar both use this component.
const LOGO_FILE = "logo.svg";
const LOGO_URL = `${import.meta.env.BASE_URL}assets/${LOGO_FILE}`;

// size is the logo height in px. The width follows the image's own shape, so wide or tall logos are never squeezed.
export default function Logo({ size = 36, withName = true, sub = "Data generation" }) {
  return (
    <span className="logo">
      <img src={LOGO_URL} alt="Logo" className="logo-img" style={{ height: size }} />
      {withName && <span className="logo-text"><b>SLM-Finetuning</b>{sub && <small>{sub}</small>}</span>}
    </span>
  );
}











//Landing.jsx
// Landing page: the first screen. "Get started" opens the app at #/generate.
import Logo from "./Logo.jsx";
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



















//landing.css
/* Landing page and logo placeholder. Uses the tokens from styles.css. */

/* ---- logo (the file is public/assets/logo.svg) */
.logo { display: inline-flex; align-items: center; gap: 11px; min-width: 0; color: inherit; text-decoration: none; }
.logo-img { display: block; flex: none; width: auto; max-width: 200px; object-fit: contain; }
.logo-text { display: grid; min-width: 0; line-height: 1.25; }
.logo-text b { font-size: 16px; font-weight: 650; letter-spacing: -.2px; color: var(--ink); }
.logo-text small { font-size: 13px; color: var(--muted); }

/* ---- page */
.landing { height: 100vh; overflow: auto; background: var(--surface); scroll-behavior: smooth; }
.lp-wrap { max-width: 1160px; margin: 0 auto; padding: 0 32px; }
.lp-eyebrow { display: inline-flex; align-items: center; gap: 6px; font-size: 13.5px; font-weight: 600; color: var(--accent-600); }
.btn.lg { height: 44px; padding: 0 20px; font-size: 15px; border-radius: 10px; }

/* nav */
.lp-nav { position: sticky; top: 0; z-index: 10; background: rgba(255, 255, 255, .85); backdrop-filter: saturate(1.4) blur(10px); border-bottom: 1px solid var(--line-2); }
.lp-nav-in { display: flex; align-items: center; gap: 28px; height: 68px; }
.lp-links { display: flex; gap: 4px; margin-left: auto; }
.lp-links a { padding: 7px 12px; border-radius: 8px; color: var(--ink-2); text-decoration: none; font-weight: 500; font-size: 15px; }
.lp-links a:hover { background: var(--surface-3); color: var(--ink); }

/* hero */
.lp-hero { position: relative; overflow: hidden; padding: 84px 0 96px;
  background: radial-gradient(900px 420px at 85% 10%, rgba(129, 140, 248, .16), transparent 70%),
              radial-gradient(700px 380px at 0% 100%, rgba(16, 185, 129, .08), transparent 70%), var(--surface); }
.lp-hero::before { content: ""; position: absolute; inset: 0; pointer-events: none; opacity: .5;
  background-image: linear-gradient(var(--line-2) 1px, transparent 1px), linear-gradient(90deg, var(--line-2) 1px, transparent 1px);
  background-size: 44px 44px; mask-image: radial-gradient(ellipse at 50% 0%, #000 30%, transparent 75%); }
.lp-hero-in { position: relative; display: grid; grid-template-columns: minmax(0, 1.05fr) minmax(0, 1fr); gap: 56px; align-items: center; }
.lp-hero-text .lp-eyebrow { padding: 5px 12px; border-radius: 99px; background: var(--accent-50); border: 1px solid var(--accent-100); }
.lp-hero h1 { margin: 20px 0 16px; font-size: 54px; line-height: 1.08; letter-spacing: -1.4px; font-weight: 700; }
.lp-hero h1 em { font-style: normal; background: linear-gradient(90deg, var(--accent), var(--violet)); -webkit-background-clip: text; background-clip: text; color: transparent; }
.lp-hero p { margin: 0; font-size: 18.5px; line-height: 1.6; color: var(--muted); max-width: 520px; }
.lp-cta { display: flex; gap: 10px; margin-top: 32px; flex-wrap: wrap; }

/* hero preview */
.lp-preview { position: relative; padding: 10px 28px 36px 0; }
.pv-card { background: var(--surface); border: 1px solid var(--line); border-radius: 18px; box-shadow: 0 30px 60px -20px rgba(16, 24, 40, .22), 0 0 0 6px rgba(255, 255, 255, .6);
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
.pv-a { background: var(--accent-50); color: #1e1b4b; box-shadow: inset 0 0 0 1px var(--accent-100); } .pv-a small { color: var(--accent); }
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
.lp-sec-h h2 { margin: 8px 0 0; font-size: 34.5px; letter-spacing: -.8px; font-weight: 700; }
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
  background: linear-gradient(120deg, var(--accent-50), #f5f3ff 60%, var(--surface)); border: 1px solid var(--accent-100); }
.lp-band h2 { margin: 0 0 6px; font-size: 28px; letter-spacing: -.6px; font-weight: 700; }

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



















//App.jsx
// Routing: the landing page at #/ (the first screen), and the app shell with sidebar and top bar at #/generate and #/results.
// Features are added one piece at a time; see README.md for the build order.
import { useEffect, useState } from "react";
import { Empty, Icon } from "./ui.jsx";
import Landing from "./Landing.jsx";
import Logo from "./Logo.jsx";

const PAGES = {
  generate: { label: "Generate", nav: "Generate dataset", icon: "spark", title: "Generate a dataset from documents" },
  results: { label: "Results", nav: "Results", icon: "chart", title: "Results" },
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
      <aside className="sidebar">
        <a className="brand" href="#/" title="Back to the home page"><Logo size={36} withName={false} /></a>
        <nav className="nav">
          <div className="side-label">Workspace</div>
          {Object.entries(PAGES).map(([k, p]) => (
            <a key={k} className={route === k ? "on" : ""} href={`#/${k}`}><Icon name={p.icon} />{p.nav}</a>
          ))}
        </nav>
        {/* Piece 4: recent jobs list goes here */}
        <div className="grow" />
        {/* Piece 5: model endpoint status goes here */}
      </aside>

      <main className="main">
        <header className="topbar">
          <div className="tb-title">
            <div className="crumbs">{page.nav}<span className="sep">/</span>New job</div>
            <h1>{page.title}</h1>
          </div>
          <div className="grow" />
          <div className="tabs-top">
            {Object.entries(PAGES).map(([k, p]) => (
              <a key={k} className={route === k ? "on" : ""} href={`#/${k}`}><Icon name={p.icon} size={14} />{p.label}</a>
            ))}
          </div>
        </header>
        <div className="content">
          {route === "generate" ? <GeneratePage /> : <ResultsPage />}
        </div>
      </main>
    </div>
  );
}

// Pieces 6–13 fill this page.
function GeneratePage() {
  return <section className="card"><Empty icon="spark" title="Generate page">Nothing here yet. The document, question and training-mix sections are added piece by piece.</Empty></section>;
}

// Pieces 14–17 fill this page.
function ResultsPage() {
  return <section className="card"><Empty icon="chart" title="Results page">Nothing here yet. The summary, Q&amp;A review and charts are added piece by piece.</Empty></section>;
}
