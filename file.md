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

/* ---- diagonal sweep (View Transitions): the new theme is revealed over the old one by src/theme.js */
::view-transition-old(root), ::view-transition-new(root) { animation: none; mix-blend-mode: normal; }
::view-transition-new(root) { z-index: 2; }
::view-transition-old(root) { z-index: 1; }
.theme-sweep *:not(.theme-toggle, .theme-toggle *) { transition: none !important; }

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













//Logo.jsx
// SLM Studio logo, built in HTML and CSS (styles in logo.css), so it can animate and follow light / dark mode.
// The mark: two chat bubbles and a sparkle on the coral gradient tile; the word mark: "SLM" in ink and "Studio" in the gradient.
// size is the logo height in px; everything scales from it. markOnly shows just the tile, without the name.
// (withName is accepted for older call sites and ignored: the name is part of the logo.)
// (The favicon in the browser tab still uses public/assets/logo-slm-studio.svg, because browsers need an image file there.)
import "./logo.css";

export default function Logo({ size = 36, markOnly = false, animated = true }) {
  return (
    <span className={`slm-logo${animated ? " is-animated" : ""}`} style={{ "--s": `${size}px` }} role="img" aria-label="SLM Studio">
      <span className="slm-mark" aria-hidden="true">
        <span className="slm-glow" />
        <span className="slm-bubble back" />
        <span className="slm-bubble front" />
        <span className="slm-spark" />
        <span className="slm-shine" />
      </span>
      {!markOnly && <span className="slm-word" aria-hidden="true"><b>SLM</b><em>Studio</em></span>}
    </span>
  );
}













//logo.css
/* SLM Studio logo in HTML/CSS. --s is the logo height; --u is one unit of a 64-unit grid. */
.slm-logo { --u: calc(var(--s) / 64); display: inline-flex; align-items: center; gap: calc(var(--u) * 13); line-height: 1; text-decoration: none; user-select: none; }

/* ---- the mark: a rounded coral tile */
.slm-mark { position: relative; flex: none; width: var(--s); height: var(--s); border-radius: calc(var(--u) * 17); overflow: hidden; isolation: isolate;
  background: linear-gradient(135deg, #F97316 0%, #F43F5E 55%, #EC4899 100%);
  box-shadow: inset 0 calc(var(--u) * 1.5) 0 rgba(255, 255, 255, .28), inset 0 calc(var(--u) * -2) calc(var(--u) * 4) rgba(124, 18, 60, .25),
    0 calc(var(--u) * 4) calc(var(--u) * 12) calc(var(--u) * -3) rgba(236, 72, 153, .55);
  transition: transform .35s cubic-bezier(.34, 1.4, .64, 1), box-shadow .35s; }
.slm-glow { position: absolute; inset: 0; background: radial-gradient(circle at 28% 22%, rgba(255, 255, 255, .45), transparent 55%); }

/* chat bubbles: a soft one behind, a white one in front; each has a little tail */
.slm-bubble { position: absolute; width: calc(var(--u) * 27); height: calc(var(--u) * 19); border-radius: calc(var(--u) * 6); }
.slm-bubble::after { content: ""; position: absolute; bottom: calc(var(--u) * -5.5); width: calc(var(--u) * 7); height: calc(var(--u) * 7); background: inherit; }
.slm-bubble.back { left: calc(var(--u) * 12); top: calc(var(--u) * 12); background: rgba(255, 255, 255, .42); }
.slm-bubble.back::after { left: calc(var(--u) * 5); clip-path: polygon(0 0, 100% 0, 0 100%); }
.slm-bubble.front { right: calc(var(--u) * 12); top: calc(var(--u) * 28); background: #fff;
  box-shadow: 0 calc(var(--u) * 2) calc(var(--u) * 5) rgba(124, 18, 60, .25); }
.slm-bubble.front::after { right: calc(var(--u) * 5); clip-path: polygon(0 0, 100% 0, 100% 100%); }

/* the sparkle inside the front bubble */
.slm-spark { position: absolute; left: calc(var(--u) * 31); top: calc(var(--u) * 31.5); width: calc(var(--u) * 12); height: calc(var(--u) * 12);
  background: linear-gradient(135deg, #F97316, #EC4899);
  clip-path: polygon(50% 0, 61% 39%, 100% 50%, 61% 61%, 50% 100%, 39% 61%, 0 50%, 39% 39%); }

/* a light sweep that passes over the tile now and then */
.slm-shine { position: absolute; top: -20%; bottom: -20%; left: -60%; width: 40%; transform: skewX(-20deg);
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, .55), transparent); opacity: 0; }

/* ---- the word mark */
.slm-word { display: inline-flex; align-items: baseline; gap: calc(var(--u) * 7); font-family: "Segoe UI Variable Display", "Segoe UI", system-ui, sans-serif;
  font-size: calc(var(--u) * 30); font-weight: 800; letter-spacing: calc(var(--u) * -.8); white-space: nowrap; }
.slm-word b { font-weight: 800; color: var(--ink, #111827); transition: color .35s; }
.slm-word em { font-style: normal; font-weight: 700; background: linear-gradient(90deg, #F97316, #EC4899, #F97316); background-size: 200% 100%;
  -webkit-background-clip: text; background-clip: text; color: transparent; }

/* ---- motion: bubbles float, the sparkle twinkles, the shine sweeps, the gradient text drifts */
.slm-logo.is-animated .slm-bubble.back { animation: slm-float 4.5s ease-in-out infinite; }
.slm-logo.is-animated .slm-bubble.front { animation: slm-float 4.5s ease-in-out infinite -2.25s; }
.slm-logo.is-animated .slm-spark { animation: slm-twinkle 3.2s ease-in-out infinite; }
.slm-logo.is-animated .slm-shine { animation: slm-shine 7s ease-in-out infinite 1.2s; }
.slm-logo.is-animated .slm-word em { animation: slm-text 8s ease-in-out infinite; }
@keyframes slm-float { 0%, 100% { translate: 0 0; } 50% { translate: 0 calc(var(--u) * -2); } }
@keyframes slm-twinkle { 0%, 100% { transform: scale(1) rotate(0deg); filter: none; } 45% { transform: scale(1.22) rotate(45deg); filter: brightness(1.15); } 60% { transform: scale(.95) rotate(45deg); } }
@keyframes slm-shine { 0%, 70% { left: -60%; opacity: 0; } 72% { opacity: 1; } 90% { left: 130%; opacity: 1; } 91%, 100% { left: 130%; opacity: 0; } }
@keyframes slm-text { 0%, 100% { background-position: 0% 0; } 50% { background-position: 100% 0; } }

/* hover: the tile lifts and tilts a touch, the sparkle spins */
a:hover > .slm-logo .slm-mark, .slm-logo:hover .slm-mark { transform: translateY(calc(var(--u) * -1.5)) rotate(-4deg);
  box-shadow: inset 0 calc(var(--u) * 1.5) 0 rgba(255, 255, 255, .28), 0 calc(var(--u) * 7) calc(var(--u) * 16) calc(var(--u) * -3) rgba(236, 72, 153, .6); }
a:hover > .slm-logo .slm-spark, .slm-logo:hover .slm-spark { animation: slm-spin .8s cubic-bezier(.34, 1.4, .64, 1); }
@keyframes slm-spin { from { transform: rotate(0) scale(1); } 50% { transform: rotate(180deg) scale(1.3); } to { transform: rotate(360deg) scale(1); } }

/* dark mode: a slightly stronger glow so the tile still pops */
[data-theme="dark"] .slm-mark { box-shadow: inset 0 calc(var(--u) * 1.5) 0 rgba(255, 255, 255, .25), 0 calc(var(--u) * 4) calc(var(--u) * 16) calc(var(--u) * -2) rgba(236, 72, 153, .5); }

@media (prefers-reduced-motion: reduce) { .slm-logo * { animation: none !important; } }

/* never inherit link underline */
.sb-brand, .sb-brand:hover, .slm-logo, .slm-logo * { text-decoration: none !important; }
