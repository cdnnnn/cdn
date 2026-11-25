//Landing.jsx
// Landing page: the first screen. "Sign in with SSO" (a demo sign-in, no real auth) opens the app at #/generate.
import { useState } from "react";
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
  const [busy, setBusy] = useState(false);
  const signIn = (e) => {
    e?.preventDefault();
    if (busy) return;
    setBusy(true);
    setTimeout(() => { window.location.hash = "#/generate"; }, 1300);
  };
  return (
    <div className="landing">
      <header className="lp-nav">
        <div className="lp-wrap lp-nav-in">
          <Logo size={46} />
          <nav className="lp-links">
            <a href="#how">How it works</a>
            <a href="#features">Features</a>
          </nav>
          <ThemeToggle />
          <SsoButton busy={busy} onClick={signIn} />
        </div>
      </header>

      <section className="lp-hero">
        <div className="lp-wrap lp-hero-in">
          <div className="lp-hero-text">
            <span className="lp-eyebrow"><Icon name="spark" size={14} />Synthetic data for small language models</span>
            <h1>Turn your documents into a <em>fine-tuning dataset</em></h1>
            <p>Upload your documents and get clean, grounded question-and-answer pairs, checked by a judge model and ready to train on.</p>
            <div className="lp-cta">
              <a className="btn primary lg" href="#/generate" onClick={signIn}><Icon name="play" size={13} />Get started</a>
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
            <a className="btn primary lg" href="#/generate" onClick={signIn}>Get started<Icon name="arrow" size={15} /></a>
          </div>
        </div>
      </section>

      <footer className="lp-foot">
        <div className="lp-wrap lp-foot-in"><Logo size={32} /><span className="muted small">Data generation for SLM fine-tuning</span></div>
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

// Demo single sign-on button. Shows "Signing you in…" for a moment, then opens the app as Prakash.
function SsoButton({ busy, onClick }) {
  return (
    <button type="button" className={`btn sso${busy ? " busy" : ""}`} onClick={onClick} disabled={busy} aria-live="polite">
      <span className="sso-ic">{busy ? <i className="sso-spin" /> : <Icon name="lock" size={14} />}</span>
      {busy ? "Signing you in…" : "Sign in with SSO"}
    </button>
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

/* Demo SSO sign-in button (header) */
.btn.sso { gap: 9px; padding-left: 8px; background: var(--ink); border-color: var(--ink); color: var(--surface); font-weight: 650;
  box-shadow: 0 6px 18px -8px color-mix(in srgb, var(--ink) 55%, transparent); transition: transform .2s, box-shadow .2s, background .2s; }
.btn.sso:hover:not(:disabled) { background: var(--ink); color: var(--surface); transform: translateY(-1px);
  box-shadow: 0 10px 24px -10px color-mix(in srgb, var(--accent) 70%, transparent); }
.btn.sso:disabled { opacity: 1; cursor: progress; }
.sso-ic { width: 24px; height: 24px; border-radius: 8px; display: grid; place-items: center; background: var(--grad); color: #fff; flex: none; }
.sso-spin { width: 12px; height: 12px; border-radius: 50%; border: 2px solid rgb(255 255 255 / .35); border-top-color: #fff; animation: sso-spin .7s linear infinite; }
@keyframes sso-spin { to { transform: rotate(360deg); } }
.btn.sso.busy { background: color-mix(in srgb, var(--ink) 88%, var(--accent)); }
@media (prefers-reduced-motion: reduce) { .sso-spin { animation-duration: 2s; } }
























//ui.jsx
// Small shared components: icons, switch, segmented control, chips, field, modal, drawer, empty state, toasts.
import { createContext, useCallback, useContext, useEffect, useRef, useState } from "react";

const P = {
  lock: <><rect x="5" y="10.5" width="14" height="10" rx="2.5" /><path d="M8.5 10.5V7.5a3.5 3.5 0 0 1 7 0v3M12 14.5v2.5" /></>,
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
