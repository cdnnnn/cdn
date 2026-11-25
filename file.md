//logo.css
/* SLM Studio logo in HTML/CSS. --s is the logo height; --u is one unit of a 64-unit grid. */
.slm-logo { --u: calc(var(--s) / 64); display: inline-flex; align-items: center; gap: calc(var(--u) * 11); line-height: 1; text-decoration: none; user-select: none; }

/* ---- the mark: three stacked layers seen in isometric view */
.slm-mark { position: relative; flex: none; width: var(--s); height: var(--s); }
.slm-halo { position: absolute; inset: 8%; border-radius: 50%; background: radial-gradient(circle, rgba(244, 63, 94, .32), transparent 68%);
  filter: blur(calc(var(--u) * 4)); opacity: .7; }

.slm-layer { position: absolute; left: 50%; width: calc(var(--u) * 34); height: calc(var(--u) * 34); border-radius: calc(var(--u) * 7);
  transform: translate(-50%, -50%) scaleY(.56) rotate(45deg); transition: translate .45s cubic-bezier(.34, 1.45, .64, 1); }
/* bottom: the raw documents */
.slm-layer.l3 { top: calc(var(--u) * 45); background: linear-gradient(135deg, #F472B6, #DB2777);
  box-shadow: calc(var(--u) * 2.6) calc(var(--u) * 2.6) 0 #9D174D, calc(var(--u) * 5) calc(var(--u) * 5) calc(var(--u) * 8) rgba(157, 23, 77, .35); opacity: .62; }
/* middle: the model */
.slm-layer.l2 { top: calc(var(--u) * 33); background: linear-gradient(135deg, #FB7185, #E11D48);
  box-shadow: calc(var(--u) * 2.6) calc(var(--u) * 2.6) 0 #9F1239, calc(var(--u) * 4) calc(var(--u) * 4) calc(var(--u) * 6) rgba(159, 18, 57, .3); opacity: .86; }
/* top: the finished dataset, with a light sweep */
.slm-layer.l1 { top: calc(var(--u) * 21); overflow: hidden; background: linear-gradient(135deg, #FDBA74 0%, #F97316 45%, #F43F5E 100%);
  box-shadow: calc(var(--u) * 2.6) calc(var(--u) * 2.6) 0 #C2410C, inset calc(var(--u) * 1.2) calc(var(--u) * 1.2) 0 rgba(255, 255, 255, .45); }
.slm-layer.l1::after { content: ""; position: absolute; inset: -40%; background: linear-gradient(90deg, transparent 40%, rgba(255, 255, 255, .75) 50%, transparent 60%);
  translate: -100% 0; }
/* the glowing core on the top layer */
.slm-core { position: absolute; left: 50%; top: 50%; width: calc(var(--u) * 9); height: calc(var(--u) * 9); border-radius: calc(var(--u) * 2.4);
  translate: -50% -50%; background: #fff; box-shadow: 0 0 calc(var(--u) * 6) rgba(255, 255, 255, .95), 0 0 calc(var(--u) * 12) rgba(255, 237, 213, .8); }

/* the spark of generation, top right */
.slm-spark { position: absolute; right: calc(var(--u) * -2); top: calc(var(--u) * -1); width: calc(var(--u) * 15); height: calc(var(--u) * 15);
  background: linear-gradient(135deg, #FDBA74, #F43F5E);
  clip-path: polygon(50% 0, 59% 41%, 100% 50%, 59% 59%, 50% 100%, 41% 59%, 0 50%, 41% 41%); }

/* ---- the word mark */
.slm-word { display: inline-flex; align-items: baseline; gap: calc(var(--u) * 7); font-family: "Segoe UI Variable Display", "Segoe UI", system-ui, sans-serif;
  font-size: calc(var(--u) * 32); font-weight: 800; letter-spacing: calc(var(--u) * -.8); white-space: nowrap; }
.slm-word b { font-weight: 800; color: var(--ink, #111827); transition: color .35s; }
.slm-word em { font-style: normal; font-weight: 700; background: linear-gradient(90deg, #F97316, #EC4899, #F97316); background-size: 200% 100%;
  -webkit-background-clip: text; background-clip: text; color: transparent; }

/* ---- motion: one slow, smooth loop that never stops
   - the whole mark floats gently
   - the layers rise and settle one after another, like a soft wave moving up the stack
   - a light sweep glides over the top layer, the core glows and dims, the spark turns slowly and twinkles
   All loops use the same 6s beat (or multiples of it) so they stay in step. */
.slm-logo.is-animated .slm-mark { animation: slm-hover 6s ease-in-out infinite; }
.slm-logo.is-animated .slm-layer { animation: slm-drop .9s cubic-bezier(.22, 1, .36, 1) backwards, slm-wave 6s ease-in-out infinite; }
.slm-logo.is-animated .l3 { animation-delay: 0s, 0s; }
.slm-logo.is-animated .l2 { animation-delay: .15s, -.5s; }
.slm-logo.is-animated .l1 { animation-delay: .3s, -1s; }
.slm-logo.is-animated .slm-layer.l1::after { animation: slm-sweep 6s cubic-bezier(.45, 0, .55, 1) infinite; }
.slm-logo.is-animated .slm-core { animation: slm-pulse 3s ease-in-out infinite; }
.slm-logo.is-animated .slm-spark { animation: slm-pop .7s cubic-bezier(.22, 1, .36, 1) .6s backwards, slm-turn 12s linear infinite, slm-twinkle 3s ease-in-out infinite; }
.slm-logo.is-animated .slm-halo { animation: slm-halo 6s ease-in-out infinite; }
.slm-logo.is-animated .slm-word em { animation: slm-text 12s ease-in-out infinite; }

@keyframes slm-drop { from { translate: 0 calc(var(--u) * -12); opacity: 0; } }
@keyframes slm-hover { 0%, 100% { translate: 0 calc(var(--u) * 1); } 50% { translate: 0 calc(var(--u) * -1.5); } }
@keyframes slm-wave { 0%, 100% { translate: 0 0; } 35% { translate: 0 calc(var(--u) * -3); } 65% { translate: 0 calc(var(--u) * .4); } }
@keyframes slm-sweep { 0%, 40% { translate: -100% 0; } 80%, 100% { translate: 100% 0; } }
@keyframes slm-pulse { 0%, 100% { scale: 1; opacity: 1; } 50% { scale: .8; opacity: .7; } }
@keyframes slm-pop { from { scale: 0; opacity: 0; } }
@keyframes slm-turn { to { rotate: 360deg; } }
@keyframes slm-twinkle { 0%, 100% { scale: 1; opacity: 1; } 50% { scale: .7; opacity: .65; } }
@keyframes slm-halo { 0%, 100% { opacity: .5; scale: .92; } 50% { opacity: .9; scale: 1.06; } }
@keyframes slm-text { 0%, 100% { background-position: 0% 0; } 50% { background-position: 100% 0; } }

/* hover: the mark grows a touch */
.slm-mark { transition: scale .5s cubic-bezier(.22, 1, .36, 1); }
a:hover > .slm-logo .slm-mark, .slm-logo:hover .slm-mark { scale: 1.08; }

/* dark mode: brighter halo, lighter back layers so the stack reads on a dark background */
[data-theme="dark"] .slm-halo { background: radial-gradient(circle, rgba(244, 63, 94, .5), transparent 68%); }
[data-theme="dark"] .slm-layer.l3 { opacity: .8; }
[data-theme="dark"] .slm-layer.l2 { opacity: .95; }

@media (prefers-reduced-motion: reduce) { .slm-logo * { animation: none !important; } }

/* never inherit link underline */
.sb-brand, .sb-brand:hover, .slm-logo, .slm-logo * { text-decoration: none !important; }





















//Sidebar.jsx
// Sidebar: the logo and the two pages.
import { Icon } from "./ui.jsx";
import Logo from "./Logo.jsx";

export default function Sidebar({ route, pages }) {
  return (
    <aside className="sidebar">
      <div className="sb-top">
        <a className="sb-brand" href="#/" title="Back to the home page"><Logo size={42} /></a>
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
          <Logo size={46} />
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
