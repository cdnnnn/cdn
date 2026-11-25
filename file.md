//src/customize.js
// User customization: colour theme and font. Kept in sessionStorage, so it lasts for this browser tab's
// session and resets when the tab is closed. The defaults are the original coral theme and the system font.
import { useEffect, useState } from "react";
import "./custom-fonts.css";
import "./palettes.css";

const KEY = "slm-custom";

export const PALETTES = [
  { id: "coral", name: "Sunset coral", note: "Warm orange to pink", a: "#F97316", b: "#EC4899" },
  { id: "ocean", name: "Ocean blue", note: "Sky blue to indigo", a: "#0EA5E9", b: "#6366F1" },
  { id: "violet", name: "Amethyst", note: "Violet to fuchsia", a: "#8B5CF6", b: "#D946EF" },
  { id: "emerald", name: "Emerald", note: "Green to cyan", a: "#10B981", b: "#06B6D4" },
];

export const FONTS = [
  { id: "system", name: "Segoe UI", note: "System font", stack: '"Segoe UI Variable Text", "Segoe UI", system-ui, -apple-system, "Helvetica Neue", Arial, sans-serif' },
  { id: "inter", name: "Inter", note: "Neutral and very legible", stack: '"Inter Pick", "Segoe UI", system-ui, sans-serif' },
  { id: "jakarta", name: "Plus Jakarta Sans", note: "Friendly and modern", stack: '"Plus Jakarta Sans Pick", "Segoe UI", system-ui, sans-serif' },
  { id: "manrope", name: "Manrope", note: "Geometric and crisp", stack: '"Manrope Pick", "Segoe UI", system-ui, sans-serif' },
  { id: "dmsans", name: "DM Sans", note: "Soft and compact", stack: '"DM Sans Pick", "Segoe UI", system-ui, sans-serif' },
];

export const DEFAULTS = { palette: "coral", font: "system" };

const valid = (c) => ({
  palette: PALETTES.some((p) => p.id === c?.palette) ? c.palette : DEFAULTS.palette,
  font: FONTS.some((f) => f.id === c?.font) ? c.font : DEFAULTS.font,
});

export function readCustom() {
  try { return valid(JSON.parse(sessionStorage.getItem(KEY) || "null")); } catch { return { ...DEFAULTS }; }
}

function apply(c) {
  const el = document.documentElement;
  if (c.palette === DEFAULTS.palette) delete el.dataset.palette; else el.dataset.palette = c.palette;
  if (c.font === DEFAULTS.font) delete el.dataset.font; else el.dataset.font = c.font;
}

export function saveCustom(next) {
  const c = valid(next);
  try {
    if (c.palette === DEFAULTS.palette && c.font === DEFAULTS.font) sessionStorage.removeItem(KEY);
    else sessionStorage.setItem(KEY, JSON.stringify(c));
  } catch { /* storage blocked: still apply for this page */ }
  const el = document.documentElement;
  el.classList.add("theme-switching");                 // colours fade for a moment instead of jumping
  apply(c);
  setTimeout(() => el.classList.remove("theme-switching"), 450);
  window.dispatchEvent(new CustomEvent("slm:custom", { detail: c }));
  return c;
}

// apply the saved choice as soon as the app loads (index.html also does this before the first paint)
apply(readCustom());

export function useCustom() {
  const [c, setC] = useState(readCustom);
  useEffect(() => {
    const h = (e) => setC(e.detail);
    window.addEventListener("slm:custom", h);
    return () => window.removeEventListener("slm:custom", h);
  }, []);
  return [c, (patch) => setC(saveCustom({ ...c, ...patch })), () => setC(saveCustom(DEFAULTS))];
}






















//Customize.jsx
// Customize panel: light/dark, colour theme and font. Opens from the profile menu (Settings) and the
// palette button in the page header. Choices apply at once and are kept for this browser session.
import { useCallback, useState } from "react";
import { emit, useBus } from "./bus.js";
import { DEFAULTS, FONTS, PALETTES, useCustom } from "./customize.js";
import { useTheme } from "./theme.js";
import { Icon, Modal } from "./ui.jsx";
import "./customize.css";

export const openCustomize = () => emit("customize");

export function CustomizeButton() {
  return (
    <button type="button" className="cz-open" onClick={openCustomize} title="Customize colours and font" aria-label="Customize colours and font">
      <Icon name="palette" size={16} />
    </button>
  );
}

export default function CustomizePanel() {
  const [open, setOpen] = useState(false);
  useBus("customize", useCallback(() => setOpen(true), []));
  const close = useCallback(() => setOpen(false), []);
  const [c, set, reset] = useCustom();
  const [theme, toggle] = useTheme();
  const isDefault = c.palette === DEFAULTS.palette && c.font === DEFAULTS.font;
  return (
    <Modal open={open} onClose={close} title="Customize" subtitle="Choose how SLM Studio looks. Changes apply right away and are kept for this browser session."
      footer={<>
        <span className="cz-note"><Icon name="clock" size={14} />Saved for this session only</span>
        <div className="grow" />
        <button type="button" className="btn ghost" onClick={reset} disabled={isDefault}><Icon name="repeat" size={14} />Reset to default</button>
        <button type="button" className="btn primary" onClick={close}>Done</button>
      </>}>
      <div className="cz">
        <section className="cz-sec">
          <header><h3>Appearance</h3><p>Light or dark mode.</p></header>
          <div className="cz-modes" role="radiogroup" aria-label="Appearance">
            {[["light", "Light"], ["dark", "Dark"]].map(([k, l]) => (
              <button type="button" key={k} role="radio" aria-checked={theme === k} className={`cz-mode m-${k}${theme === k ? " on" : ""}`} onClick={() => theme !== k && toggle()}>
                <span className="cz-mode-pv" aria-hidden="true"><i /><b /><b /><em /></span>
                <span className="cz-mode-l"><span className="cz-tick"><Icon name="check" size={12} /></span>{l}</span>
              </button>
            ))}
          </div>
        </section>

        <section className="cz-sec">
          <header><h3>Colour theme</h3><p>Used for buttons, highlights, charts and the logo.</p></header>
          <div className="cz-pals" role="radiogroup" aria-label="Colour theme">
            {PALETTES.map((p) => (
              <button type="button" key={p.id} role="radio" aria-checked={c.palette === p.id} className={`cz-pal${c.palette === p.id ? " on" : ""}`}
                style={{ "--pa": p.a, "--pb": p.b }} onClick={() => set({ palette: p.id })}>
                <span className="cz-pal-pv" aria-hidden="true">
                  <span className="cz-orb" />
                  <span className="cz-lines"><i /><i /></span>
                  <span className="cz-chip">Next</span>
                </span>
                <span className="cz-pal-t">
                  <b><span>{p.name}</span>{p.id === DEFAULTS.palette && <small className="cz-def">Default</small>}</b>
                  <small>{p.note}</small>
                </span>
                <span className="cz-tick"><Icon name="check" size={12} /></span>
              </button>
            ))}
          </div>
        </section>

        <section className="cz-sec">
          <header><h3>Font</h3><p>The typeface for all text in the app.</p></header>
          <div className="cz-fonts" role="radiogroup" aria-label="Font">
            {FONTS.map((f) => (
              <button type="button" key={f.id} role="radio" aria-checked={c.font === f.id} className={`cz-font${c.font === f.id ? " on" : ""}`}
                style={{ fontFamily: f.stack }} onClick={() => set({ font: f.id })}>
                <span className="cz-aa" aria-hidden="true">Aa</span>
                <span className="cz-font-t">
                  <b><span>{f.name}</span>{f.id === DEFAULTS.font && <small className="cz-def">Default</small>}</b>
                  <small>{f.note}</small>
                </span>
                <span className="cz-tick"><Icon name="check" size={12} /></span>
              </button>
            ))}
          </div>
        </section>
      </div>
    </Modal>
  );
}





















//customize.css
/* Customize panel */
.cz { display: grid; gap: 24px; }
.cz-sec > header { margin-bottom: 11px; }
.cz-sec h3 { margin: 0; font-size: 14.5px; font-weight: 700; color: var(--ink); }
.cz-sec p { margin: 2px 0 0; font-size: 13px; color: var(--muted); }
.cz-note { display: inline-flex; align-items: center; gap: 6px; font-size: 12.5px; color: var(--muted); }

/* shared: the tick in the corner of the chosen option */
.cz-tick { width: 20px; height: 20px; border-radius: 50%; display: grid; place-items: center; flex: none; color: #fff; background: var(--grad);
  opacity: 0; transform: scale(.5); transition: opacity .2s, transform .3s cubic-bezier(.34, 1.4, .64, 1); }
.on > .cz-tick, .on .cz-tick { opacity: 1; transform: none; }
.cz-def { margin-left: 7px; padding: 1px 6px; border-radius: 99px; font-size: 10.5px; font-weight: 650; letter-spacing: .2px; vertical-align: 1px;
  background: var(--surface-3); color: var(--muted); font-family: var(--font); }

.cz-pal, .cz-font, .cz-mode { position: relative; text-align: left; cursor: pointer; border: 1px solid var(--line); background: var(--surface); border-radius: 14px;
  color: var(--ink); font: inherit; transition: border-color .2s, box-shadow .2s, transform .2s cubic-bezier(.22, 1, .36, 1), background .2s; }
.cz-pal:hover, .cz-font:hover, .cz-mode:hover { transform: translateY(-2px); border-color: var(--line-strong); box-shadow: 0 8px 20px -14px rgb(16 24 40 / .35); }
.cz-pal.on, .cz-font.on, .cz-mode.on { border-color: transparent; box-shadow: 0 0 0 2px var(--accent), 0 10px 24px -16px rgba(var(--accent-rgb), .7); }
.cz-pal:focus-visible, .cz-font:focus-visible, .cz-mode:focus-visible { outline: none; box-shadow: var(--ring), 0 0 0 1px var(--accent); }

/* appearance */
.cz-modes { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; }
.cz-mode { padding: 10px; display: grid; gap: 9px; }
.cz-mode-pv { height: 64px; border-radius: 9px; position: relative; overflow: hidden; display: block; border: 1px solid var(--line-2); }
.m-light .cz-mode-pv { background: #f6f7f9; } .m-dark .cz-mode-pv { background: #0c0e13; border-color: #222631; }
.cz-mode-pv i { position: absolute; left: 0; top: 0; bottom: 0; width: 26%; }
.m-light .cz-mode-pv i { background: #fff; border-right: 1px solid #e6e8ec; } .m-dark .cz-mode-pv i { background: #14171f; border-right: 1px solid #2a2f3b; }
.cz-mode-pv b { position: absolute; left: 33%; height: 7px; border-radius: 4px; }
.cz-mode-pv b:nth-of-type(1) { top: 14px; width: 40%; } .cz-mode-pv b:nth-of-type(2) { top: 28px; width: 55%; }
.m-light .cz-mode-pv b { background: #e6e8ec; } .m-dark .cz-mode-pv b { background: #2a2f3b; }
.cz-mode-pv em { position: absolute; left: 33%; bottom: 10px; width: 26%; height: 11px; border-radius: 6px; background: var(--grad); }
.cz-mode-l { display: flex; align-items: center; gap: 8px; font-weight: 650; font-size: 13.5px; }
.cz-mode-l .cz-tick { width: 18px; height: 18px; }
.cz-mode:not(.on) .cz-mode-l .cz-tick { display: none; }

/* colour themes */
.cz-pals { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 12px; }
.cz-pal { display: grid; grid-template-columns: auto minmax(0, 1fr); align-items: center; gap: 12px; padding: 12px; }
.cz-pal-pv { width: 80px; height: 56px; border-radius: 10px; position: relative; overflow: hidden; display: block;
  background: linear-gradient(135deg, color-mix(in srgb, var(--pa) 14%, var(--surface)), color-mix(in srgb, var(--pb) 12%, var(--surface))); border: 1px solid var(--line-2); }
.cz-orb { position: absolute; left: 9px; top: 9px; width: 18px; height: 18px; border-radius: 6px; background: linear-gradient(135deg, var(--pa), var(--pb));
  box-shadow: 0 4px 10px -4px var(--pb); }
.cz-lines { position: absolute; left: 33px; top: 11px; display: grid; gap: 4px; }
.cz-lines i { display: block; height: 5px; border-radius: 3px; background: color-mix(in srgb, var(--ink) 18%, transparent); width: 36px; }
.cz-lines i + i { width: 24px; }
.cz-chip { position: absolute; right: 8px; bottom: 7px; padding: 2px 8px; border-radius: 6px; font-size: 10px; font-weight: 700; color: #fff;
  background: linear-gradient(90deg, var(--pa), var(--pb)); }
.cz-pal-t, .cz-font-t { display: grid; gap: 2px; min-width: 0; }
.cz-pal-t b, .cz-font-t b { font-size: 14px; font-weight: 650; display: flex; align-items: center; min-width: 0; white-space: nowrap; }
.cz-pal-t b > span, .cz-font-t b > span { overflow: hidden; text-overflow: ellipsis; }
.cz-def { flex: none; }
.cz-pal-t small, .cz-font-t small { font-size: 12.5px; color: var(--muted); }
.cz-pal:hover .cz-orb { transform: rotate(-8deg) scale(1.08); } .cz-orb { transition: transform .35s cubic-bezier(.34, 1.4, .64, 1); }

/* fonts */
.cz-fonts { display: grid; grid-template-columns: repeat(auto-fill, minmax(200px, 1fr)); gap: 10px; }
.cz-font { display: grid; grid-template-columns: auto minmax(0, 1fr); align-items: center; gap: 12px; padding: 11px 12px; }
.cz-aa { width: 44px; height: 44px; border-radius: 11px; display: grid; place-items: center; font-size: 21px; font-weight: 700; letter-spacing: -.5px;
  background: var(--surface-3); color: var(--ink); transition: background .2s, color .2s; }
.cz-font.on .cz-aa { background: var(--grad); color: #fff; }

/* header button that opens the panel */
.cz-open { width: 34px; height: 34px; border-radius: 10px; display: grid; place-items: center; border: 1px solid var(--line); background: var(--surface); color: var(--ink-2);
  cursor: pointer; transition: color .2s, border-color .2s, transform .3s cubic-bezier(.34, 1.4, .64, 1), box-shadow .2s; }
.cz-open:hover { color: var(--accent-600); border-color: rgba(var(--accent-rgb), .45); transform: rotate(-12deg); box-shadow: 0 6px 16px -10px rgba(var(--accent-rgb), .8); }

@media (max-width: 720px) { .cz-pals, .cz-modes { grid-template-columns: 1fr; } }

/* the tick sits in the top-right corner, so names get the full width */
.cz-pal > .cz-tick, .cz-font > .cz-tick { position: absolute; top: 8px; right: 8px; width: 18px; height: 18px; }



















//palettes.css
/* Colour themes for the Customize panel. Sunset coral is the default and lives in styles.css / theme-dark.css;
   each block below swaps only the theme colours (accent, gradient, tints, logo) for light and, after it, dark mode. */

/* ── Ocean blue: sky blue to indigo ── */
:root[data-palette="ocean"] {
  --grad-a: #0EA5E9; --grad-b: #6366F1; --grad-a-rgb: 14, 165, 233; --grad-b-rgb: 99, 102, 241;
  --grad: linear-gradient(135deg, var(--grad-a), var(--grad-b)); --grad-hover: linear-gradient(135deg, #0284C7, #4F46E5);
  --accent: #2F6FEB; --accent-rgb: 47, 111, 235; --accent-600: #1D4ED8; --accent-400: #5B8DEF;
  --accent-200: #C7D7FE; --accent-100: #E0E9FF; --accent-50: #F2F6FF; --accent-ink: #172554;
  --tint-0: #F8FBFF; --tint-1: #EEF5FF; --tint-2: #F1F0FF;
  --violet: #4F46E5; --violet-50: #EEF2FF; --violet-100: #E0E7FF; --violet-ink: #312E81;
  --mix-1: #2F6FEB; --mix-2: #A5C4F7; --split-1: #2F6FEB;
  --lg-l1a: #7DD3FC; --lg-l1b: #0EA5E9; --lg-l1c: #3B82F6; --lg-l1e: #0369A1;
  --lg-l2a: #60A5FA; --lg-l2b: #2563EB; --lg-l2e: #1E40AF; --lg-l2s: 30, 64, 175;
  --lg-l3a: #A5B4FC; --lg-l3b: #6366F1; --lg-l3e: #3730A3; --lg-l3s: 55, 48, 163;
  --lg-halo: 59, 130, 246; --lg-glow: 224, 242, 254;
}
:root[data-theme="dark"][data-palette="ocean"] {
  --accent: #5B8DF6; --accent-rgb: 91, 141, 246; --accent-600: #8FB1FF; --accent-400: #6E9AF5;
  --accent-200: #233A66; --accent-100: #1A2A4A; --accent-50: #141C30; --accent-ink: #DBE6FF;
  --tint-0: #14171d; --tint-1: #161c27; --tint-2: #191a2a;
  --violet: #818CF8; --violet-50: #1b1c33; --violet-100: #26284a; --violet-ink: #E0E7FF;
  --mix-1: #5B8DF6; --mix-2: #4E6A9E; --split-1: #5B8DF6;
}

/* ── Amethyst: violet to fuchsia ── */
:root[data-palette="violet"] {
  --grad-a: #8B5CF6; --grad-b: #D946EF; --grad-a-rgb: 139, 92, 246; --grad-b-rgb: 217, 70, 239;
  --grad: linear-gradient(135deg, var(--grad-a), var(--grad-b)); --grad-hover: linear-gradient(135deg, #7C3AED, #C026D3);
  --accent: #7C4DEB; --accent-rgb: 124, 77, 235; --accent-600: #6D28D9; --accent-400: #A78BFA;
  --accent-200: #DDD6FE; --accent-100: #EDE9FE; --accent-50: #F7F4FF; --accent-ink: #2E1065;
  --tint-0: #FCFAFF; --tint-1: #F5F0FF; --tint-2: #FBF0FD;
  --violet: #C026D3; --violet-50: #FDF4FF; --violet-100: #FAE8FF; --violet-ink: #701A75;
  --mix-1: #7C4DEB; --mix-2: #C9B6F7; --split-1: #7C4DEB;
  --lg-l1a: #C4B5FD; --lg-l1b: #8B5CF6; --lg-l1c: #7C3AED; --lg-l1e: #5B21B6;
  --lg-l2a: #E879F9; --lg-l2b: #C026D3; --lg-l2e: #86198F; --lg-l2s: 134, 25, 143;
  --lg-l3a: #A78BFA; --lg-l3b: #6D28D9; --lg-l3e: #4C1D95; --lg-l3s: 76, 29, 149;
  --lg-halo: 139, 92, 246; --lg-glow: 237, 233, 254;
}
:root[data-theme="dark"][data-palette="violet"] {
  --accent: #A07CF8; --accent-rgb: 160, 124, 248; --accent-600: #C4B0FF; --accent-400: #9B7CF2;
  --accent-200: #3B2C66; --accent-100: #2A2048; --accent-50: #1C1730; --accent-ink: #EDE6FF;
  --tint-0: #17161d; --tint-1: #1c1826; --tint-2: #1f1726;
  --violet: #E879F9; --violet-50: #2a1530; --violet-100: #3d1d45; --violet-ink: #FAE8FF;
  --mix-1: #A07CF8; --mix-2: #6E5AA8; --split-1: #A07CF8;
}

/* ── Emerald: green to cyan ── */
:root[data-palette="emerald"] {
  --grad-a: #10B981; --grad-b: #06B6D4; --grad-a-rgb: 16, 185, 129; --grad-b-rgb: 6, 182, 212;
  --grad: linear-gradient(135deg, var(--grad-a), var(--grad-b)); --grad-hover: linear-gradient(135deg, #059669, #0891B2);
  --accent: #0E9F6E; --accent-rgb: 14, 159, 110; --accent-600: #047857; --accent-400: #34C795;
  --accent-200: #A7F3D0; --accent-100: #D1FAE5; --accent-50: #F0FDF7; --accent-ink: #064E3B;
  --tint-0: #F8FEFB; --tint-1: #EEFBF4; --tint-2: #ECFBFD;
  --violet: #0891B2; --violet-50: #ECFEFF; --violet-100: #CFFAFE; --violet-ink: #164E63;
  --mix-1: #0E9F6E; --mix-2: #9FE3C8; --split-1: #0E9F6E;
  --lg-l1a: #6EE7B7; --lg-l1b: #10B981; --lg-l1c: #0D9488; --lg-l1e: #047857;
  --lg-l2a: #2DD4BF; --lg-l2b: #0D9488; --lg-l2e: #115E59; --lg-l2s: 17, 94, 89;
  --lg-l3a: #67E8F9; --lg-l3b: #0891B2; --lg-l3e: #155E75; --lg-l3s: 21, 94, 117;
  --lg-halo: 16, 185, 129; --lg-glow: 209, 250, 229;
}
:root[data-theme="dark"][data-palette="emerald"] {
  --accent: #2FC48D; --accent-rgb: 47, 196, 141; --accent-600: #6EE7B7; --accent-400: #3CCB98;
  --accent-200: #164A38; --accent-100: #113327; --accent-50: #0E221B; --accent-ink: #D1FAE5;
  --tint-0: #131a18; --tint-1: #14201c; --tint-2: #131f22;
  --violet: #22D3EE; --violet-50: #0f2429; --violet-100: #15343b; --violet-ink: #CFFAFE;
  --mix-1: #2FC48D; --mix-2: #3F7F68; --split-1: #2FC48D;
}

/* ── fonts ── the system font (Segoe UI) is the default, set in styles.css */
:root[data-font="inter"] { --font: "Inter Pick", "Segoe UI", system-ui, sans-serif; }
:root[data-font="jakarta"] { --font: "Plus Jakarta Sans Pick", "Segoe UI", system-ui, sans-serif; }
:root[data-font="manrope"] { --font: "Manrope Pick", "Segoe UI", system-ui, sans-serif; }
:root[data-font="dmsans"] { --font: "DM Sans Pick", "Segoe UI", system-ui, sans-serif; }




















//custom-fonts.css
/* Fonts for the Customize panel, bundled locally (no CDN). A browser only downloads a font once it is picked.
   Licences (SIL OFL 1.1) are next to the files in assets/fonts/. */
@font-face { font-family: 'Plus Jakarta Sans Pick'; font-style: normal; font-display: swap; font-weight: 200 800;
  src: url(./assets/fonts/plus-jakarta-sans-latin-ext-wght-normal.woff2) format('woff2-variations'); unicode-range: U+0100-02BA,U+02BD-02C5,U+02C7-02CC,U+02CE-02D7,U+02DD-02FF,U+0304,U+0308,U+0329,U+1D00-1DBF,U+1E00-1E9F,U+1EF2-1EFF,U+2020,U+20A0-20AB,U+20AD-20C0,U+2113,U+2C60-2C7F,U+A720-A7FF; }
@font-face { font-family: 'Plus Jakarta Sans Pick'; font-style: normal; font-display: swap; font-weight: 200 800;
  src: url(./assets/fonts/plus-jakarta-sans-latin-wght-normal.woff2) format('woff2-variations'); unicode-range: U+0000-00FF,U+0131,U+0152-0153,U+02BB-02BC,U+02C6,U+02DA,U+02DC,U+0304,U+0308,U+0329,U+2000-206F,U+20AC,U+2122,U+2191,U+2193,U+2212,U+2215,U+FEFF,U+FFFD; }
@font-face { font-family: 'Inter Pick'; font-style: normal; font-display: swap; font-weight: 100 900;
  src: url(./assets/fonts/inter-latin-ext-wght-normal.woff2) format('woff2-variations'); unicode-range: U+0100-02BA,U+02BD-02C5,U+02C7-02CC,U+02CE-02D7,U+02DD-02FF,U+0304,U+0308,U+0329,U+1D00-1DBF,U+1E00-1E9F,U+1EF2-1EFF,U+2020,U+20A0-20AB,U+20AD-20C0,U+2113,U+2C60-2C7F,U+A720-A7FF; }
@font-face { font-family: 'Inter Pick'; font-style: normal; font-display: swap; font-weight: 100 900;
  src: url(./assets/fonts/inter-latin-wght-normal.woff2) format('woff2-variations'); unicode-range: U+0000-00FF,U+0131,U+0152-0153,U+02BB-02BC,U+02C6,U+02DA,U+02DC,U+0304,U+0308,U+0329,U+2000-206F,U+20AC,U+2122,U+2191,U+2193,U+2212,U+2215,U+FEFF,U+FFFD; }
@font-face { font-family: 'Manrope Pick'; font-style: normal; font-display: swap; font-weight: 200 800;
  src: url(./assets/fonts/manrope-latin-ext-wght-normal.woff2) format('woff2-variations'); unicode-range: U+0100-02BA,U+02BD-02C5,U+02C7-02CC,U+02CE-02D7,U+02DD-02FF,U+0304,U+0308,U+0329,U+1D00-1DBF,U+1E00-1E9F,U+1EF2-1EFF,U+2020,U+20A0-20AB,U+20AD-20C0,U+2113,U+2C60-2C7F,U+A720-A7FF; }
@font-face { font-family: 'Manrope Pick'; font-style: normal; font-display: swap; font-weight: 200 800;
  src: url(./assets/fonts/manrope-latin-wght-normal.woff2) format('woff2-variations'); unicode-range: U+0000-00FF,U+0131,U+0152-0153,U+02BB-02BC,U+02C6,U+02DA,U+02DC,U+0304,U+0308,U+0329,U+2000-206F,U+20AC,U+2122,U+2191,U+2193,U+2212,U+2215,U+FEFF,U+FFFD; }
@font-face { font-family: 'DM Sans Pick'; font-style: normal; font-display: swap; font-weight: 100 1000;
  src: url(./assets/fonts/dm-sans-latin-ext-wght-normal.woff2) format('woff2-variations'); unicode-range: U+0100-02BA,U+02BD-02C5,U+02C7-02CC,U+02CE-02D7,U+02DD-02FF,U+0304,U+0308,U+0329,U+1D00-1DBF,U+1E00-1E9F,U+1EF2-1EFF,U+2020,U+20A0-20AB,U+20AD-20C0,U+2113,U+2C60-2C7F,U+A720-A7FF; }
@font-face { font-family: 'DM Sans Pick'; font-style: normal; font-display: swap; font-weight: 100 1000;
  src: url(./assets/fonts/dm-sans-latin-wght-normal.woff2) format('woff2-variations'); unicode-range: U+0000-00FF,U+0131,U+0152-0153,U+02BB-02BC,U+02C6,U+02DA,U+02DC,U+0304,U+0308,U+0329,U+2000-206F,U+20AC,U+2122,U+2191,U+2193,U+2212,U+2215,U+FEFF,U+FFFD; }




















 
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
      // colour theme and font from the Customize panel (kept for this browser session)
      try { var c = JSON.parse(sessionStorage.getItem("slm-custom") || "null"); if (c && c.palette && c.palette !== "coral") document.documentElement.dataset.palette = c.palette; if (c && c.font && c.font !== "system") document.documentElement.dataset.font = c.font; } catch (e) {}
    </script>
    <link rel="icon" type="image/svg+xml" href="/assets/logo-slm-studio.svg" />
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>















//main.jsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import App from "./App.jsx";
import { ToastProvider } from "./ui.jsx";
import "./styles.css";
import "./landing.css";
import "./customize.js";
import CustomizePanel from "./Customize.jsx";

createRoot(document.getElementById("root")).render(
  <StrictMode>
    <ToastProvider>
      <App />
      <CustomizePanel />
    </ToastProvider>
  </StrictMode>,
);












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




















//Landing.jsx
// Landing page: the first screen. "Sign in with SSO" (a demo sign-in, no real auth) opens the app at #/generate.
import { useState } from "react";
import Logo from "./Logo.jsx";
import ThemeToggle from "./ThemeToggle.jsx";
import { CustomizeButton } from "./Customize.jsx";
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
          <CustomizeButton />
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

















//Sidebar.jsx
// Sidebar: the logo, the two pages and the signed-in user. It can be collapsed to icons only; the choice is remembered.
import { useEffect, useRef, useState } from "react";
import { openCustomize } from "./Customize.jsx";
import { Icon, useToast } from "./ui.jsx";
import Logo from "./Logo.jsx";
import "./sidebar.css";

const KEY = "slm-sidebar";
// a dummy user until sign-in exists
const USER = { name: "Prakash", role: "AI Department", initials: "P" };

export default function Sidebar({ route, pages }) {
  const [collapsed, setCollapsed] = useState(() => localStorage.getItem(KEY) === "collapsed");
  useEffect(() => { localStorage.setItem(KEY, collapsed ? "collapsed" : "open"); }, [collapsed]);
  // keyboard shortcut: Ctrl/Cmd + B toggles the sidebar
  useEffect(() => {
    const k = (e) => { if ((e.ctrlKey || e.metaKey) && e.key.toLowerCase() === "b") { e.preventDefault(); setCollapsed((c) => !c); } };
    window.addEventListener("keydown", k);
    return () => window.removeEventListener("keydown", k);
  }, []);

  return (
    <aside className={`sidebar${collapsed ? " is-collapsed" : ""}`}>
      <div className="sb-top">
        <a className="sb-brand" href="#/" title="Back to the home page"><Logo size={42} /></a>
      </div>
      <button type="button" className="sb-toggle" onClick={() => setCollapsed((c) => !c)}
        aria-label={collapsed ? "Expand sidebar" : "Collapse sidebar"} aria-expanded={!collapsed} data-tip={collapsed ? "Expand (Ctrl+B)" : "Collapse (Ctrl+B)"}>
        <Icon name="chevron" size={14} stroke={2.2} />
      </button>

      <nav className="sb-nav" style={{ "--i": Math.max(0, Object.keys(pages).indexOf(route)) }}>
        <span className="sb-ind" aria-hidden="true" />
        {Object.entries(pages).map(([k, p]) => (
          <a key={k} className={route === k ? "on" : ""} href={`#/${k}`} data-tip={p.nav} aria-label={p.nav}>
            <span className="sb-nav-ico"><Icon name={p.icon} size={17} /></span><span className="sb-label">{p.nav}</span>
          </a>
        ))}
      </nav>

      <UserCard collapsed={collapsed} />
    </aside>
  );
}

function UserCard({ collapsed }) {
  const toast = useToast();
  const [open, setOpen] = useState(false);
  const ref = useRef(null);
  useEffect(() => {
    if (!open) return undefined;
    const h = (e) => { if (!ref.current?.contains(e.target)) setOpen(false); };
    const k = (e) => { if (e.key === "Escape") setOpen(false); };
    document.addEventListener("mousedown", h); document.addEventListener("keydown", k);
    return () => { document.removeEventListener("mousedown", h); document.removeEventListener("keydown", k); };
  }, [open]);
  const demo = (what) => { setOpen(false); toast(`${what} is not available in this demo yet`); };
  return (
    <div className="sb-user" ref={ref}>
      {open && (
        <div className="sb-user-menu" role="menu">
          <div className="sb-user-menu-h"><span className="sb-avatar sm">{USER.initials}</span><div><b>{USER.name}</b><small>{USER.role}</small></div></div>
          <button type="button" role="menuitem" onClick={() => demo("Your profile")}><Icon name="eye" size={15} />Your profile</button>
          <button type="button" role="menuitem" onClick={() => { setOpen(false); openCustomize(); }}><Icon name="palette" size={15} />Customize</button>
          <div className="sb-user-sep" />
          <button type="button" role="menuitem" className="danger" onClick={() => demo("Sign out")}><Icon name="back" size={15} />Sign out</button>
        </div>
      )}
      <button type="button" className={`sb-user-btn${open ? " on" : ""}`} onClick={() => setOpen((o) => !o)} aria-haspopup="menu" aria-expanded={open}
        data-tip={collapsed && !open ? USER.name : undefined}>
        <span className="sb-avatar">{USER.initials}<i className="sb-online" aria-label="Online" /></span>
        <span className="sb-user-t sb-label"><b>{USER.name}</b><small>{USER.role}</small></span>
        <span className="sb-user-chev sb-label"><Icon name="chevron" size={14} /></span>
      </button>
    </div>
  );
}

















//ui.jsx
// Small shared components: icons, switch, segmented control, chips, field, modal, drawer, empty state, toasts.
import { createContext, useCallback, useContext, useEffect, useRef, useState } from "react";

const P = {
  palette: <><path d="M12 3.5a8.5 8.5 0 1 0 0 17c1.2 0 1.8-.8 1.8-1.7 0-.5-.2-.9-.5-1.2-.3-.3-.5-.7-.5-1.2 0-.9.8-1.7 1.7-1.7h2a4 4 0 0 0 4-4c0-4-3.8-7.2-8.5-7.2z" /><circle cx="7.6" cy="11" r="1.1" /><circle cx="10" cy="7.4" r="1.1" /><circle cx="14.4" cy="7.4" r="1.1" /></>,
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























//logo.css
/* SLM Studio logo in HTML/CSS. --s is the logo height; --u is one unit of a 64-unit grid.
   Colours come from the --lg-* tokens (set per colour theme in palettes.css); the values after the commas are the default coral. */
.slm-logo { --u: calc(var(--s) / 64); display: inline-flex; align-items: center; gap: calc(var(--u) * 11); line-height: 1; text-decoration: none; user-select: none; }

/* ---- the mark: three stacked layers seen in isometric view */
.slm-mark { position: relative; flex: none; width: var(--s); height: var(--s); }
.slm-halo { position: absolute; inset: 8%; border-radius: 50%; background: radial-gradient(circle, rgba(var(--lg-halo, 244, 63, 94), .32), transparent 68%);
  filter: blur(calc(var(--u) * 4)); opacity: .7; }

/* each layer sits in a wrapper; the wrapper moves, the layer keeps its fixed isometric shape */
.slm-lw { position: absolute; inset: 0; }
.slm-layer { position: absolute; left: 50%; width: calc(var(--u) * 34); height: calc(var(--u) * 34); border-radius: calc(var(--u) * 7);
  transform: translate(-50%, -50%) scaleY(.56) rotate(45deg); }
/* bottom: the raw documents */
.slm-layer.l3 { top: calc(var(--u) * 45); background: linear-gradient(135deg, var(--lg-l3a, #F472B6), var(--lg-l3b, #DB2777));
  box-shadow: calc(var(--u) * 2.6) calc(var(--u) * 2.6) 0 var(--lg-l3e, #9D174D), calc(var(--u) * 5) calc(var(--u) * 5) calc(var(--u) * 8) rgba(var(--lg-l3s, 157, 23, 77), .35); opacity: .62; }
/* middle: the model */
.slm-layer.l2 { top: calc(var(--u) * 33); background: linear-gradient(135deg, var(--lg-l2a, #FB7185), var(--lg-l2b, #E11D48));
  box-shadow: calc(var(--u) * 2.6) calc(var(--u) * 2.6) 0 var(--lg-l2e, #9F1239), calc(var(--u) * 4) calc(var(--u) * 4) calc(var(--u) * 6) rgba(var(--lg-l2s, 159, 18, 57), .3); opacity: .86; }
/* top: the finished dataset, with a light sweep */
.slm-layer.l1 { top: calc(var(--u) * 21); overflow: hidden; background: linear-gradient(135deg, var(--lg-l1a, #FDBA74) 0%, var(--lg-l1b, #F97316) 45%, var(--lg-l1c, #F43F5E) 100%);
  box-shadow: calc(var(--u) * 2.6) calc(var(--u) * 2.6) 0 var(--lg-l1e, #C2410C), inset calc(var(--u) * 1.2) calc(var(--u) * 1.2) 0 rgba(255, 255, 255, .45); }
.slm-layer.l1::after { content: ""; position: absolute; inset: -40%; background: linear-gradient(90deg, transparent 40%, rgba(255, 255, 255, .75) 50%, transparent 60%);
  translate: -100% 0; }
/* the glowing core on the top layer */
.slm-core { position: absolute; left: 50%; top: 50%; width: calc(var(--u) * 9); height: calc(var(--u) * 9); border-radius: calc(var(--u) * 2.4);
  translate: -50% -50%; background: #fff; box-shadow: 0 0 calc(var(--u) * 6) rgba(255, 255, 255, .95), 0 0 calc(var(--u) * 12) rgba(var(--lg-glow, 255, 237, 213), .8); }

/* the spark of generation, top right */
.slm-spark { position: absolute; right: calc(var(--u) * -2); top: calc(var(--u) * -1); width: calc(var(--u) * 15); height: calc(var(--u) * 15);
  background: linear-gradient(135deg, var(--lg-l1a, #FDBA74), var(--lg-l1c, #F43F5E));
  clip-path: polygon(50% 0, 59% 41%, 100% 50%, 59% 59%, 50% 100%, 41% 59%, 0 50%, 41% 41%); }

/* ---- the word mark */
.slm-word { display: inline-flex; align-items: baseline; gap: calc(var(--u) * 7); font-family: "Segoe UI Variable Display", "Segoe UI", system-ui, sans-serif;
  font-size: calc(var(--u) * 32); font-weight: 800; letter-spacing: calc(var(--u) * -.8); white-space: nowrap; }
.slm-word b { font-weight: 800; color: var(--ink, #111827); transition: color .35s; }
.slm-word em { font-style: normal; font-weight: 700; background: linear-gradient(90deg, var(--grad-a, #F97316), var(--grad-b, #EC4899), var(--grad-a, #F97316)); background-size: 200% 100%;
  -webkit-background-clip: text; background-clip: text; color: transparent; }

/* ---- motion: one slow, smooth loop that never stops
   - the whole mark floats gently
   - the layers rise and settle one after another, like a soft wave moving up the stack
   - a light sweep glides over the top layer, the core glows and dims, the spark turns slowly and twinkles
   All loops use the same 6s beat (or multiples of it) so they stay in step. */
.slm-logo.is-animated .slm-mark { animation: slm-hover 6s ease-in-out infinite; }
.slm-logo.is-animated .slm-lw { animation: slm-wave 6s ease-in-out infinite; }
.slm-logo.is-animated .w2 { animation-delay: -.5s; }
.slm-logo.is-animated .w1 { animation-delay: -1s; }
.slm-logo.is-animated .slm-layer.l1::after { animation: slm-sweep 6s cubic-bezier(.45, 0, .55, 1) infinite; }
.slm-logo.is-animated .slm-core { animation: slm-pulse 3s ease-in-out infinite; }
.slm-logo.is-animated .slm-spark { animation: slm-turn 12s linear infinite, slm-twinkle 3s ease-in-out infinite; }
.slm-logo.is-animated .slm-halo { animation: slm-halo 6s ease-in-out infinite; }
.slm-logo.is-animated .slm-word em { animation: slm-text 12s ease-in-out infinite; }

@keyframes slm-hover { 0%, 100% { translate: 0 calc(var(--u) * 1); } 50% { translate: 0 calc(var(--u) * -1.5); } }
@keyframes slm-wave { 0%, 100% { transform: translateY(0); } 35% { transform: translateY(calc(var(--u) * -3)); } 65% { transform: translateY(calc(var(--u) * .4)); } }
@keyframes slm-sweep { 0%, 40% { translate: -100% 0; } 80%, 100% { translate: 100% 0; } }
@keyframes slm-pulse { 0%, 100% { scale: 1; opacity: 1; } 50% { scale: .8; opacity: .7; } }
@keyframes slm-turn { to { rotate: 360deg; } }
@keyframes slm-twinkle { 0%, 100% { scale: 1; opacity: 1; } 50% { scale: .7; opacity: .65; } }
@keyframes slm-halo { 0%, 100% { opacity: .5; scale: .92; } 50% { opacity: .9; scale: 1.06; } }
@keyframes slm-text { 0%, 100% { background-position: 0% 0; } 50% { background-position: 100% 0; } }

/* hover: the mark grows a touch */
.slm-mark { transition: scale .5s cubic-bezier(.22, 1, .36, 1); }
a:hover > .slm-logo .slm-mark, .slm-logo:hover .slm-mark { scale: 1.08; }

/* dark mode: brighter halo, lighter back layers so the stack reads on a dark background */
[data-theme="dark"] .slm-halo { background: radial-gradient(circle, rgba(var(--lg-halo, 244, 63, 94), .5), transparent 68%); }
[data-theme="dark"] .slm-layer.l3 { opacity: .8; }
[data-theme="dark"] .slm-layer.l2 { opacity: .95; }

@media (prefers-reduced-motion: reduce) { .slm-logo * { animation: none !important; } }

/* never inherit link underline */
.sb-brand, .sb-brand:hover, .slm-logo, .slm-logo * { text-decoration: none !important; }

















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
.btn.primary.lg { position: relative; overflow: hidden; }
.btn.primary.lg::after { content: ""; position: absolute; top: 0; bottom: 0; left: -40%; width: 30%; transform: skewX(-20deg);
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, .35), transparent); transition: left .7s ease; }
.btn.primary.lg:hover::after { left: 120%; }

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
  radial-gradient(900px 420px at 100% -10%, rgba(var(--grad-a-rgb), .07), transparent 60%),
  radial-gradient(700px 380px at 30% -20%, rgba(var(--grad-b-rgb), .05), transparent 60%),
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
[data-theme="dark"] .btn.primary { box-shadow: inset 0 1px 0 rgba(255, 255, 255, .2), 0 8px 20px -8px rgba(var(--grad-b-rgb), .55); }
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























//results/Insights.jsx
// Charts for the Results page, drawn with Recharts. Colours come from the theme (light/dark) and the coral palette.
import { useEffect, useState } from "react";
import { Bar, BarChart, Cell, LabelList, Pie, PieChart, ResponsiveContainer, Tooltip, XAxis, YAxis } from "recharts";
import { fmtN } from "../model.js";
import { SCORES } from "./Pair.jsx";

const calm = () => typeof matchMedia === "function" && matchMedia("(prefers-reduced-motion: reduce)").matches;

// counts a number up from 0 with an ease-out, for the headline figures
export function useCount(target, ms = 900) {
  const [v, setV] = useState(calm() ? target : 0);
  useEffect(() => {
    if (typeof target !== "number" || calm()) { setV(target); return undefined; }
    let raf, t0;
    const step = (t) => { t0 ??= t; const k = Math.min(1, (t - t0) / ms); setV(target * (1 - (1 - k) ** 3)); if (k < 1) raf = requestAnimationFrame(step); };
    raf = requestAnimationFrame(step);
    return () => cancelAnimationFrame(raf);
  }, [target, ms]);
  return v;
}

export function Meter({ value }) {
  return <span className="rs-meter"><i style={{ "--w": `${Math.max(0, Math.min(1, value)) * 100}%` }} /></span>;
}

export function Donut({ value, label, sub }) {
  const r = 26, c = 2 * Math.PI * r;
  return (
    <span className="rs-donut">
      <svg viewBox="0 0 64 64" aria-hidden="true">
        <defs><linearGradient id="rs-g" x1="0" y1="0" x2="1" y2="1"><stop offset="0" stopColor="var(--grad-a)" /><stop offset="1" stopColor="var(--grad-b)" /></linearGradient></defs>
        <circle cx="32" cy="32" r={r} className="trk" />
        <circle cx="32" cy="32" r={r} className="val" style={{ "--c": c, "--o": c * (1 - Math.max(0, Math.min(1, value))) }} />
      </svg>
      <span className="rs-donut-t"><b className="num">{label}</b>{sub && <small>{sub}</small>}</span>
    </span>
  );
}


// ---- theme -------------------------------------------------------------------------------------
// Categorical palette (coral first), checked for colour-blind separation in both modes.
const CAT = {
  light: ["#F2545B", "#257DE0", "#00B894", "#F2A600", "#6A4CE2", "#EE7FAC", "#008A17"],
  dark: ["#EB6263", "#338CED", "#00A888", "#D28E05", "#9887EF", "#E24D8B", "#0AAC23"],
};
// One-hue coral ramp for judge scores 1 → 5 (low → high).
const RAMP = {
  light: ["#FDE3E4", "#F9B4B7", "#F5858A", "#F2545B", "#C2333A"],
  dark: ["#4A2529", "#7C3439", "#B4464C", "#EB6263", "#FF9A9C"],
};
const TOK = ["ink", "ink-2", "muted", "faint", "line", "line-2", "surface", "surface-3", "accent"];
// colour helpers, so the charts follow the colour theme picked in Customize
const hex = (h) => { const m = /^#?([0-9a-f]{6})$/i.exec(String(h).trim()); return m ? [0, 2, 4].map((i) => parseInt(m[1].slice(i, i + 2), 16)) : null; };
const toHex = (c) => `#${c.map((v) => Math.round(Math.max(0, Math.min(255, v))).toString(16).padStart(2, "0")).join("")}`;
const mix = (a, b, k) => toHex(a.map((v, i) => v * k + b[i] * (1 - k)));
const hue = ([r, g, b]) => { r /= 255; g /= 255; b /= 255; const mx = Math.max(r, g, b), mn = Math.min(r, g, b), d = mx - mn; if (!d) return 0;
  const h = mx === r ? ((g - b) / d) % 6 : mx === g ? (b - r) / d + 2 : (r - g) / d + 4; return (h * 60 + 360) % 360; };
const readTheme = () => {
  const el = document.documentElement, cs = getComputedStyle(el);
  const mode = el.dataset.theme === "dark" ? "dark" : "light";
  const t = { mode, cat: CAT[mode], ramp: RAMP[mode] };
  TOK.forEach((k) => { t[k] = cs.getPropertyValue(`--${k}`).trim(); });
  const acc = hex(t.accent), surf = hex(t.surface);
  if (el.dataset.palette && acc && surf) {
    // score ramp: one hue (the theme accent), light to strong
    const end = mode === "dark" ? [255, 255, 255] : [0, 0, 0];
    t.ramp = mode === "dark" ? [mix(acc, surf, .22), mix(acc, surf, .42), mix(acc, surf, .68), t.accent, mix(acc, end, .62)]
      : [mix(acc, surf, .16), mix(acc, surf, .38), mix(acc, surf, .66), t.accent, mix(acc, end, .78)];
    // categories: the theme accent takes slot 1 and coral takes the place of the hue closest to it
    const cat = [...CAT[mode]], h0 = hue(acc);
    let best = 1, bd = 999;
    cat.forEach((c, i) => { if (!i) return; const d = Math.abs(((hue(hex(c)) - h0 + 540) % 360) - 180); if (d < bd) { bd = d; best = i; } });
    cat[best] = cat[0]; cat[0] = t.accent;
    t.cat = cat;
  }
  return t;
};
function useChartTheme() {
  const [t, setT] = useState(readTheme);
  useEffect(() => {
    const mo = new MutationObserver(() => setT(readTheme()));
    mo.observe(document.documentElement, { attributes: true, attributeFilter: ["data-theme", "data-palette", "class", "style"] });
    return () => mo.disconnect();
  }, []);
  return t;
}
const cap = (s) => String(s).charAt(0).toUpperCase() + String(s).slice(1);
const anim = () => ({ isAnimationActive: !calm(), animationDuration: 1000, animationEasing: "ease-out" });
const tick = (t) => ({ fill: t["ink-2"], fontSize: 12.5, fontFamily: "inherit" });

// ---- tooltip -----------------------------------------------------------------------------------
function Tip({ active, payload, label, fmt }) {
  if (!active || !payload?.length) return null;
  const rows = fmt ? fmt(payload, label) : payload.map((p) => ({ c: p.payload?.fill || p.color, l: p.name, v: fmtN(p.value) }));
  const title = rows.title ?? label;
  return (
    <div className="rs-tip">
      {title != null && title !== "" && <b className="rs-tip-h">{title}</b>}
      {(rows.items || rows).map((r) => <span key={r.l}><i style={{ background: r.c }} />{r.l}<b className="num">{r.v}</b></span>)}
    </div>
  );
}
const tipProps = (t, fmt) => ({ content: <Tip fmt={fmt} />, cursor: { fill: t["surface-3"], radius: 6 }, isAnimationActive: false, wrapperStyle: { outline: "none", zIndex: 5 } });

// ---- what the dataset is made of: one 100% stacked bar ---------------------------------------
export function StackBar({ parts, note }) {
  const t = useChartTheme();
  const live = parts.filter((p) => p[1]);
  const tot = live.reduce((a, p) => a + p[1], 0) || 1;
  const row = Object.fromEntries(live.map(([l, v]) => [l, v]));
  const col = (k) => t.cat[[0, 3, 1, 2][k] ?? k];
  return (
    <div className="rs-stack">
      <ResponsiveContainer width="100%" height={34}>
        <BarChart layout="vertical" data={[row]} margin={{ top: 4, right: 0, bottom: 4, left: 0 }} barCategoryGap={0}>
          <XAxis type="number" hide domain={[0, tot]} />
          <YAxis type="category" hide />
          <Tooltip {...tipProps(t, (p) => ({ title: "", items: p.map((x) => ({ c: x.color, l: x.name, v: `${fmtN(x.value)} · ${Math.round((100 * x.value) / tot)}%` })) }))} cursor={false} shared={false} />
          {live.map(([l], k) => (
            <Bar key={l} dataKey={l} name={l} stackId="s" fill={col(parts.findIndex((p) => p[0] === l))} stroke={t.surface} strokeWidth={2} {...anim()}
              radius={live.length === 1 ? 6 : k === 0 ? [6, 0, 0, 6] : k === live.length - 1 ? [0, 6, 6, 0] : 0} />
          ))}
        </BarChart>
      </ResponsiveContainer>
      <div className="rs-legend">
        {parts.map(([l, v], k) => <span key={l}><i style={{ background: col(k) }} />{l}<b className="num">{fmtN(v)}</b></span>)}
      </div>
      {note && <small className="muted">{note}</small>}
    </div>
  );
}

// ---- where the questions went: funnel as horizontal bars --------------------------------------
export function Funnel({ steps }) {
  const t = useChartTheme();
  const data = steps.map(([l, v, end], k) => {
    const prev = k ? steps[k - 1][1] : null;
    const drop = prev != null && v != null && prev > 0 && v < prev ? Math.round((100 * (prev - v)) / prev) : null;
    return { l, v: v || 0, fin: !!end, drop };
  });
  const label = ({ x, y, width, height, index }) => {
    const d = data[index];
    return (
      <text x={x + width + 8} y={y + height / 2} dy="0.35em" fontSize={12.5} fontFamily="inherit">
        <tspan fill={t.ink} fontWeight={650}>{fmtN(d.v)}</tspan>
        {d.drop != null && <tspan fill={t.muted} fontSize={11.5} dx={6}>−{d.drop}%</tspan>}
      </text>
    );
  };
  return (
    <ResponsiveContainer width="100%" height={data.length * 38 + 6}>
      <BarChart layout="vertical" data={data} margin={{ top: 0, right: 78, bottom: 0, left: 0 }} barCategoryGap={8}>
        <XAxis type="number" hide domain={[0, "dataMax"]} />
        <YAxis type="category" dataKey="l" width={118} tickLine={false} axisLine={false} tick={tick(t)} />
        <Tooltip {...tipProps(t, (p) => { const d = p[0].payload; return { title: d.l, items: [{ c: d.fin ? t.accent : t.ramp[2], l: "Questions", v: fmtN(d.v) }, ...(d.drop != null ? [{ c: t.faint, l: "Lost from previous step", v: `${d.drop}%` }] : [])] }; })} />
        <Bar dataKey="v" name="Questions" radius={[0, 6, 6, 0]} background={{ fill: t["surface-3"], radius: 6 }} {...anim()}>
          {data.map((d) => <Cell key={d.l} fill={d.fin ? t.accent : t.ramp[1]} />)}
          <LabelList dataKey="v" content={label} />
        </Bar>
      </BarChart>
    </ResponsiveContainer>
  );
}

// ---- judge scores: 100% stacked bar per criterion, 1 → 5 ------------------------------------
export function ScoreStacks({ scores }) {
  const t = useChartTheme();
  const rows = Object.entries(scores);
  if (!rows.length) return <div className="muted small">No judged pairs kept.</div>;
  const data = rows.map(([k, m]) => {
    const tot = Object.values(m).reduce((a, b) => a + b, 0) || 1;
    const d = { name: SCORES[k] || k, tot, avg: Object.entries(m).reduce((a, [n, c]) => a + Number(n) * c, 0) / tot };
    [1, 2, 3, 4, 5].forEach((n) => { d[`s${n}`] = (100 * (m[n] || 0)) / tot; d[`c${n}`] = m[n] || 0; });
    return d;
  });
  const fmt = (p) => { const d = p[0].payload; return { title: `${d.name} · average ${d.avg.toFixed(1)}`, items: [5, 4, 3, 2, 1].filter((n) => d[`c${n}`]).map((n) => ({ c: t.ramp[n - 1], l: `Score ${n}`, v: `${d[`c${n}`]} · ${Math.round(d[`s${n}`])}%` })) }; };
  return (
    <div className="rs-scores">
      <ResponsiveContainer width="100%" height={data.length * 34 + 4}>
        <BarChart layout="vertical" data={data} margin={{ top: 0, right: 0, bottom: 0, left: 0 }} barCategoryGap={9}>
          <XAxis type="number" hide domain={[0, 100]} />
          <YAxis yAxisId="l" type="category" dataKey="name" width={108} tickLine={false} axisLine={false} tick={tick(t)} />
          <YAxis yAxisId="r" orientation="right" type="category" dataKey="avg" width={34} tickLine={false} axisLine={false}
            tick={{ ...tick(t), fill: t.ink, fontWeight: 650 }} tickFormatter={(v) => Number(v).toFixed(1)} />
          <Tooltip {...tipProps(t, fmt)} />
          {[1, 2, 3, 4, 5].map((n) => (
            <Bar key={n} yAxisId="l" dataKey={`s${n}`} name={`Score ${n}`} stackId="s" fill={t.ramp[n - 1]} stroke={t.surface} strokeWidth={2} {...anim()} />
          ))}
        </BarChart>
      </ResponsiveContainer>
      <div className="rs-sleg">{[1, 2, 3, 4, 5].map((n) => <span key={n}><i style={{ background: t.ramp[n - 1] }} />{n}</span>)}<span className="muted">· score, low to high · average on the right</span></div>
    </div>
  );
}

// ---- question types: donut ------------------------------------------------------------------
const TYPE_ORDER = ["factual", "how-to", "explanation", "comparison", "from_images", "decline", "general"];
export function TypeDonut({ types }) {
  const t = useChartTheme();
  const rows = Object.entries(types).filter((r) => r[1]).sort((a, b) => b[1] - a[1]);
  const tot = rows.reduce((a, r) => a + r[1], 0);
  if (!tot) return <div className="muted small">No questions yet.</div>;
  // colour follows the type, never its rank
  const col = (k) => { const i = TYPE_ORDER.indexOf(k); return i < 0 ? t.faint : t.cat[i]; };
  const data = rows.map(([k, v]) => ({ name: k.replace(/_/g, " "), key: k, value: v }));
  return (
    <div className="rs-types">
      <div className="rs-pie">
        <ResponsiveContainer width="100%" height="100%">
          <PieChart>
            <Tooltip {...tipProps(t, (p) => ({ title: "", items: [{ c: col(p[0].payload.key), l: cap(p[0].name), v: `${fmtN(p[0].value)} · ${Math.round((100 * p[0].value) / tot)}%` }] }))} cursor={false} />
            <Pie data={data} dataKey="value" nameKey="name" innerRadius="66%" outerRadius="100%" paddingAngle={data.length > 1 ? 2 : 0} cornerRadius={4}
              stroke={t.surface} strokeWidth={2} startAngle={90} endAngle={-270} {...anim()} animationDuration={1100}>
              {data.map((d) => <Cell key={d.key} fill={col(d.key)} />)}
            </Pie>
          </PieChart>
        </ResponsiveContainer>
        <span className="rs-pie-c"><b className="num">{fmtN(tot)}</b><small>pairs</small></span>
      </div>
      <div className="rs-tlist">
        {data.map((d) => (
          <span key={d.key}><i style={{ background: col(d.key) }} />{d.name}<b className="num">{fmtN(d.value)}</b><small className="num">{Math.round((100 * d.value) / tot)}%</small></span>
        ))}
      </div>
    </div>
  );
}

// ---- simple horizontal bars (drop reasons) ---------------------------------------------------
export function Bars({ rows, empty }) {
  const t = useChartTheme();
  if (!rows.length) return <div className="muted small">{empty}</div>;
  const data = rows.map(([l, v, c]) => ({ l, v: v || 0, fill: c === "muted" ? t.faint : t.accent }));
  return (
    <ResponsiveContainer width="100%" height={data.length * 32 + 4}>
      <BarChart layout="vertical" data={data} margin={{ top: 0, right: 36, bottom: 0, left: 0 }} barCategoryGap={9}>
        <XAxis type="number" hide domain={[0, "dataMax"]} />
        <YAxis type="category" dataKey="l" width={150} tickLine={false} axisLine={false} tick={{ ...tick(t), textAnchor: "end" }}
          tickFormatter={(s) => { const x = s.charAt(0).toUpperCase() + s.slice(1); return x.length > 22 ? `${x.slice(0, 21)}…` : x; }} />
        <Tooltip {...tipProps(t, (p) => ({ title: p[0].payload.l, items: [{ c: p[0].payload.fill, l: "Pairs removed", v: fmtN(p[0].value) }] }))} />
        <Bar dataKey="v" name="Pairs" radius={[0, 5, 5, 0]} maxBarSize={14} background={{ fill: t["line-2"], radius: 5 }} {...anim()}>
          {data.map((d) => <Cell key={d.l} fill={d.fill} />)}
          <LabelList dataKey="v" position="right" offset={8} fill={t.ink} fontSize={12.5} fontWeight={650} formatter={fmtN} />
        </Bar>
      </BarChart>
    </ResponsiveContainer>
  );
}

export function ImagePath({ im }) {
  const steps = [["Found", im.found], ["Read", im.read], ["Chunks", im.image_chunks], ["Questions", im.questions], ["Answered", im.answered], ["Kept", im.kept]];
  return (
    <>
      <div className="rs-ipath">
        {steps.map(([l, v], k) => <div key={l} className={k === steps.length - 1 ? "end" : ""} style={{ "--k": k }}><b className="num">{fmtN(v ?? 0)}</b><small>{l}</small></div>)}
      </div>
      {(im.hint || im.without_image_data > 0) && <div className="rs-note">{[im.hint, im.without_image_data ? `${im.without_image_data} picture(s) had no image data.` : ""].filter(Boolean).join(" ")}</div>}
      {!im.hint && Object.keys(im.skip_reasons || {}).length > 0 && <div className="muted small">Skipped: {Object.entries(im.skip_reasons).map(([k, v]) => `${v} × ${k}`).join(" · ")}</div>}
    </>
  );
}

export function SplitTable({ cov }) {
  const rows = Object.entries(cov);
  if (!rows.length) return <div className="muted small">No sections.</div>;
  return (
    <table className="rs-tbl">
      <thead><tr><th>Section</th><th>Train</th><th>Val</th><th>Test</th></tr></thead>
      <tbody>{rows.map(([sec, c]) => (
        <tr key={sec} className={!c.train && (c.val || c.test) ? "bad" : ""}><td title={sec}>{sec}</td>
          <td className="num">{fmtN(c.train || 0)}</td><td className="num">{fmtN(c.val || 0)}</td><td className="num">{fmtN(c.test || 0)}</td></tr>
      ))}</tbody>
    </table>
  );
}























//results/results.css
/* ================================================================== Results page (simple version)
   Calm surfaces, coral only where it means something (the dataset, high scores, the main action).
   Every colour comes from the theme tokens, so light and dark mode both work. */
.rs { display: grid; gap: 18px; --sc-1: color-mix(in srgb, var(--accent) 10%, var(--line)); --sc-2: color-mix(in srgb, var(--accent) 28%, var(--surface));
  --sc-3: color-mix(in srgb, var(--accent) 50%, var(--surface)); --sc-4: color-mix(in srgb, var(--accent) 75%, var(--surface)); --sc-5: var(--accent); }
.rs .num { font-variant-numeric: tabular-nums; }

/* ---- entrance: cards rise in one after another */
.rs-in { animation: rs-rise .55s cubic-bezier(.22, 1, .36, 1) both; animation-delay: calc(var(--d, 0) * 55ms); }
@keyframes rs-rise { from { opacity: 0; transform: translateY(12px); } to { opacity: 1; transform: none; } }

.rs-stack { display: grid; gap: 12px; }
.rs-stack-bar { display: flex; height: 10px; border-radius: 99px; overflow: hidden; gap: 2px; background: var(--line-2); }
.rs-stack-bar i { width: var(--w); background: var(--c); transform-origin: left; animation: rs-grow .9s cubic-bezier(.22, 1, .36, 1) both; animation-delay: calc(.25s + var(--k) * 90ms); }
.rs-legend { display: grid; grid-template-columns: 1fr 1fr; gap: 6px 14px; }
.rs-legend span { display: flex; align-items: center; gap: 7px; font-size: 13px; color: var(--ink-2); min-width: 0; }
.rs-legend i { width: 8px; height: 8px; border-radius: 3px; background: var(--c); flex: none; }
.rs-legend b { margin-left: auto; font-weight: 650; color: var(--ink); }
.rs-stack > small { font-size: 12.5px; }
.rs-funnel { display: grid; gap: 8px; }
.rs-fstep { display: grid; grid-template-columns: 112px minmax(0, 1fr) 30px 40px; gap: 10px; align-items: center; font-size: 13px; position: relative; }
.rs-fl { color: var(--ink-2); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.rs-ft { height: 22px; border-radius: 7px; background: var(--surface-3); overflow: hidden; }
.rs-ft i { display: block; height: 100%; width: var(--w); border-radius: 7px; background: color-mix(in srgb, var(--accent) 22%, var(--surface));
  transform-origin: left; animation: rs-grow .9s cubic-bezier(.22, 1, .36, 1) both; animation-delay: calc(.3s + var(--k) * 90ms); }
.rs-fstep.end .rs-ft i { background: var(--grad); }
.rs-fstep b { text-align: right; font-weight: 650; }
.rs-fd { font-size: 11.5px; font-weight: 600; color: var(--faint); text-align: right; }

.rs-scores { display: grid; gap: 9px; }
.rs-srow { display: grid; grid-template-columns: 104px minmax(0, 1fr) 30px; gap: 10px; align-items: center; font-size: 13px; }
.rs-sl { color: var(--ink-2); }
.rs-sbar { display: flex; height: 12px; border-radius: 99px; overflow: hidden; gap: 2px; background: var(--line-2); }
.rs-sbar i { width: var(--w); background: var(--c); transform-origin: left; animation: rs-grow .8s cubic-bezier(.22, 1, .36, 1) both; animation-delay: calc(.3s + var(--k) * 70ms); }
.rs-srow b { text-align: right; font-weight: 650; }
.rs-sleg { display: flex; gap: 10px; flex-wrap: wrap; font-size: 12px; color: var(--muted); padding-top: 4px; }
.rs-sleg span { display: inline-flex; align-items: center; gap: 4px; }
.rs-sleg i { width: 9px; height: 9px; border-radius: 3px; background: var(--c); }

.rs-types { display: flex; gap: 18px; align-items: center; }
.rs-tlist { display: grid; gap: 7px; flex: 1; min-width: 0; }
.rs-tlist span { display: grid; grid-template-columns: 10px minmax(0, 1fr) auto 36px; gap: 8px; align-items: center; font-size: 13px; color: var(--ink-2); text-transform: capitalize; }
.rs-tlist i { width: 9px; height: 9px; border-radius: 3px; background: var(--c); }
.rs-tlist b { font-weight: 650; color: var(--ink); } .rs-tlist small { color: var(--muted); text-align: right; font-size: 12px; }

.rs-bars { display: grid; gap: 9px; }
.rs-brow { display: grid; grid-template-columns: minmax(0, 140px) minmax(0, 1fr) 32px; gap: 10px; align-items: center; font-size: 13px; }
.rs-bl { color: var(--ink-2); overflow: hidden; text-overflow: ellipsis; white-space: nowrap; text-transform: capitalize; }
.rs-bt { height: 8px; border-radius: 99px; background: var(--line-2); overflow: hidden; }
.rs-bt i { display: block; height: 100%; width: var(--w); border-radius: 99px; background: var(--c); transform-origin: left;
  animation: rs-grow .8s cubic-bezier(.22, 1, .36, 1) both; animation-delay: calc(.3s + var(--k) * 80ms); }
.rs-brow b { text-align: right; font-weight: 650; }

.rs-ipath { display: grid; grid-template-columns: repeat(6, 1fr); gap: 0; position: relative; }
.rs-ipath::before { content: ""; position: absolute; left: 8%; right: 8%; top: 17px; height: 2px; background: linear-gradient(90deg, var(--line), color-mix(in srgb, var(--accent) 45%, var(--line))); }
.rs-ipath > div { display: grid; justify-items: center; gap: 5px; position: relative; animation: rs-rise .5s cubic-bezier(.22, 1, .36, 1) both; animation-delay: calc(.3s + var(--k) * 70ms); }
.rs-ipath b { width: 36px; height: 36px; border-radius: 50%; display: grid; place-items: center; font-size: 14px; font-weight: 700; background: var(--surface);
  border: 1px solid var(--line-strong); color: var(--ink); box-shadow: 0 0 0 4px var(--surface); }
.rs-ipath .end b { background: var(--grad); color: #fff; border-color: transparent; box-shadow: 0 0 0 4px var(--surface), 0 4px 10px -3px rgba(var(--accent-rgb), .6); }
.rs-ipath small { font-size: 11.5px; color: var(--muted); }
.rs-note { font-size: 12.5px; color: var(--amber-ink); background: var(--amber-50); border: 1px solid var(--amber-200); border-radius: 9px; padding: 7px 10px; }

.rs-tbl { width: 100%; border-collapse: separate; border-spacing: 0; font-size: 13px; }
.rs-tbl th { text-align: left; font-size: 11px; font-weight: 700; letter-spacing: .5px; text-transform: uppercase; color: var(--faint); padding: 0 8px 8px; border-bottom: 1px solid var(--line); }
.rs-tbl th:not(:first-child), .rs-tbl td:not(:first-child) { text-align: right; width: 52px; }
.rs-tbl td { padding: 8px; border-bottom: 1px solid var(--line-2); color: var(--ink-2); }
.rs-tbl td:first-child { max-width: 170px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; color: var(--ink); font-weight: 500; }
.rs-tbl tr:last-child td { border-bottom: 0; }
.rs-tbl tbody tr { transition: background .12s; } .rs-tbl tbody tr:hover { background: var(--surface-2); }
.rs-tbl tr.bad td { color: var(--red); }

@keyframes rs-grow { from { transform: scaleX(0); } }
.rs-skel { display: grid; grid-template-columns: 1fr 1fr; gap: 14px; padding: 12px; }
.rs-skel i { height: 110px; border-radius: 14px; background: linear-gradient(90deg, var(--surface-3) 0%, var(--surface-2) 50%, var(--surface-3) 100%); background-size: 200% 100%;
  animation: rs-shimmer 1.4s ease-in-out infinite; }
.rs-skel.rows { grid-template-columns: 1fr; padding: 0; } .rs-skel.rows i { height: 120px; }
@keyframes rs-shimmer { from { background-position: 100% 0; } to { background-position: -100% 0; } }
.rs-empty { padding: 8px; }
.rs-empty .empty > span { display: grid; justify-items: center; gap: 14px; }
.rs-chev { color: var(--faint); transform: rotate(90deg); transition: transform .25s cubic-bezier(.22, 1, .36, 1); flex: none; }
.rs-chev.sm { margin-left: -2px; }

/* ---- the four key numbers, in one calm strip */
.rs-sum { display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); background: var(--surface); border: 1px solid var(--line); border-radius: 14px; animation: rs-rise .5s cubic-bezier(.22, 1, .36, 1) both; }
.rs-sum > div { display: grid; gap: 4px; padding: 18px 22px; min-width: 0; }
.rs-sum > div + div { border-left: 1px solid var(--line-2); }
.rs-sum span { font-size: 13px; color: var(--muted); }
.rs-sum b { font-family: var(--font); font-size: 28px; font-weight: 700; letter-spacing: -.6px; line-height: 1.15; color: var(--ink); }
.rs-sum b em { font-style: normal; font-size: 16px; font-weight: 500; color: var(--muted); letter-spacing: 0; margin-left: 3px; }
.rs-sum small { font-size: 12.5px; color: var(--faint); }
.rs-sum > div:first-child b { background: var(--grad); -webkit-background-clip: text; background-clip: text; color: transparent; }

/* ---- review */
.rs-review { display: grid; gap: 14px; animation: rs-rise .45s cubic-bezier(.22, 1, .36, 1) both; }
.rs-filters { display: flex; align-items: center; gap: 10px; flex-wrap: wrap; }
.rs-seg { display: inline-flex; gap: 2px; padding: 3px; border-radius: 10px; background: var(--surface-3); border: 1px solid var(--line-2); }
.rs-seg button { display: inline-flex; align-items: center; gap: 6px; height: 30px; padding: 0 12px; border: 0; border-radius: 7px; background: none; cursor: pointer;
  font: inherit; font-size: 13.5px; font-weight: 600; color: var(--muted); transition: color .15s, background .15s, box-shadow .15s; }
.rs-seg button:hover { color: var(--ink); }
.rs-seg button.on { background: var(--surface); color: var(--ink); box-shadow: 0 1px 2px rgba(16, 24, 40, .1); }
.rs-seg button span { font-size: 12px; color: var(--faint); font-weight: 600; }
.rs-seg button.on span { color: var(--accent-600); }
.rs-filters .search { height: 36px; }
.rs-filters .search input { width: 170px !important; background: transparent !important; color: var(--ink); }
.rs-select { display: inline-flex; align-items: center; height: 36px; border: 1px solid var(--line-strong); border-radius: 9px; background: var(--surface); box-shadow: var(--shadow-xs); }
.rs-select:focus-within { border-color: var(--accent-400); box-shadow: var(--ring); }
.rs-select select { border: 0 !important; box-shadow: none !important; background-color: transparent; height: 34px; padding: 0 28px 0 12px !important; font-size: 13.5px; color: var(--ink); }

.rs-list { display: grid; background: var(--surface); border: 1px solid var(--line); border-radius: 14px; overflow: hidden; transition: opacity .2s; }
.rs-list.is-loading { opacity: .6; }
.rs-list > .empty { padding: 48px 20px; }
.rs-list > .rs-skel { padding: 12px; }
.rs-pager { display: flex; align-items: center; gap: 8px; padding: 0 4px; }

/* ---- one pair: a row in the list */
.qa { display: grid; grid-template-columns: minmax(0, 1fr) auto; gap: 20px; padding: 18px 20px; position: relative; transition: background .2s;
  animation: rs-rise .4s cubic-bezier(.22, 1, .36, 1) both; animation-delay: calc(var(--i, 0) * 35ms); }
.qa + .qa { border-top: 1px solid var(--line-2); }
.qa:hover { background: var(--surface-2); }
.qa::before { content: ""; position: absolute; left: 0; top: 18px; bottom: 18px; width: 3px; border-radius: 0 3px 3px 0; background: transparent; transition: background .2s; }
.qa.is-approve::before { background: var(--green); }
.qa.is-reject::before { background: var(--red); }
.qa.is-reject .qa-q, .qa.is-reject .qa-a { opacity: .5; }
.qa.is-reject .qa-q { text-decoration: line-through; text-decoration-color: color-mix(in srgb, var(--red) 50%, transparent); }
.qa-main { display: grid; gap: 6px; min-width: 0; }
.qa-q { margin: 0; font-size: 15.5px; font-weight: 600; color: var(--ink); line-height: 1.45; }
.qa-a { margin: 0; font-size: 14.5px; color: var(--ink-2); line-height: 1.6; white-space: pre-wrap; }
.qa.is-out .qa-a { color: var(--muted); }
.qa-why { margin: 2px 0 0; font-size: 13px; color: var(--red-ink); padding-left: 10px; border-left: 2px solid var(--red-200); }
.qa-meta { display: flex; align-items: center; gap: 8px; margin-top: 4px; font-size: 12.5px; color: var(--muted); min-width: 0; flex-wrap: wrap; }
.qa-meta > span.ellipsis { max-width: 360px; }
.qa-type { padding: 1px 8px; border-radius: 99px; background: var(--surface-3); color: var(--ink-2); font-weight: 500; text-transform: capitalize; }
.qa-avg { font-weight: 650; color: var(--ink-2); }
.qa-avg small { font-weight: 400; color: var(--faint); margin-left: 1px; }
.qa-more-btn { display: inline-flex; align-items: center; gap: 3px; border: 0; background: none; padding: 0; cursor: pointer; font: inherit; font-size: 12.5px; font-weight: 600; color: var(--accent-600); }
.qa-more-btn svg { transform: rotate(90deg); transition: transform .2s; }
.qa-more-btn.on svg { transform: rotate(-90deg); }
.qa-more { display: grid; gap: 12px; margin-top: 8px; padding: 14px; border-radius: 12px; background: var(--surface-2); border: 1px solid var(--line-2); animation: rs-rise .3s ease both; }
.qa:hover .qa-more { background: var(--surface); }
.qa-scores { display: grid; grid-template-columns: repeat(auto-fill, minmax(190px, 1fr)); gap: 8px 18px; }
.qa-score { display: grid; grid-template-columns: 1fr auto 16px; gap: 8px; align-items: center; font-size: 13px; color: var(--ink-2); }
.qa-score b { text-align: right; font-weight: 650; }
.qa-score small { grid-column: 1 / -1; font-size: 12px; color: var(--muted); margin-top: -3px; }
.qa-pips { display: inline-flex; gap: 2px; }
.qa-pips i { width: 12px; height: 5px; border-radius: 3px; background: var(--line); }
.qa-score.hi .qa-pips i.on { background: var(--accent); }
.qa-score.mid .qa-pips i.on { background: var(--amber); }
.qa-score.lo .qa-pips i.on { background: var(--red); }
.qa-kv { display: grid; grid-template-columns: 80px minmax(0, 1fr); gap: 10px; font-size: 13px; }
.qa-kv > span { color: var(--faint); font-weight: 600; font-size: 12px; padding-top: 2px; }
.qa-kv p { margin: 0; color: var(--ink-2); }
.qa-chips { display: flex; flex-wrap: wrap; gap: 5px; align-items: center; }

.qa-edit { display: grid; gap: 10px; animation: rs-rise .3s ease both; }
.qa-edit label { display: grid; gap: 4px; }
.qa-edit span { font-size: 12px; font-weight: 600; color: var(--faint); }
.qa-edit textarea { width: 100%; resize: vertical; }
.qa-edit-act { display: flex; gap: 6px; justify-content: flex-end; }

.qa-side { display: flex; flex-direction: column; align-items: flex-end; gap: 10px; }
.qa-state { font-size: 12px; font-weight: 600; padding: 2px 9px; border-radius: 99px; white-space: nowrap; background: var(--surface-3); color: var(--muted); }
.qa-state.ok { background: var(--green-50); color: var(--green-600); } .qa-state.bad { background: var(--red-50); color: var(--red); }
.qa-act { display: flex; gap: 4px; }
.qa-btn { width: 32px; height: 32px; display: grid; place-items: center; border-radius: 9px; border: 1px solid var(--line); background: var(--surface); color: var(--muted);
  cursor: pointer; transition: color .15s, background .15s, border-color .15s, transform .15s; }
.qa-btn:hover:not(:disabled) { color: var(--ink); border-color: var(--line-strong); }
.qa-btn:active:not(:disabled) { transform: scale(.94); }
.qa-btn:disabled { opacity: .5; cursor: default; }
.qa-btn.yes:hover:not(:disabled) { color: var(--green-600); border-color: var(--green-200); background: var(--green-50); }
.qa-btn.no:hover:not(:disabled) { color: var(--red); border-color: var(--red-200); background: var(--red-50); }
.qa-btn.yes.on, .qa-btn.yes.on:hover:not(:disabled) { color: #fff; background: var(--green-600); border-color: var(--green-600); }
.qa-btn.no.on, .qa-btn.no.on:hover:not(:disabled) { color: #fff; background: var(--red); border-color: var(--red); }

/* ---- insights */
.rs-ins { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 16px; align-items: start; }
.rs-panel { display: grid; gap: 14px; align-content: start; padding: 18px 20px; background: var(--surface); border: 1px solid var(--line); border-radius: 14px; animation: rs-rise .5s cubic-bezier(.22, 1, .36, 1) both; animation-delay: calc(var(--d, 0) * 50ms); }
.rs-panel.wide { grid-column: 1 / -1; }
.rs-panel > header { display: flex; align-items: center; gap: 8px; }
.rs-panel h3 { margin: 0; font-size: 14.5px; font-weight: 650; }
.rs-note-r { font-size: 12.5px; color: var(--muted); }

/* ---- guided flow pieces */
.rs-main { min-height: 520px; }
.rs .stp:disabled { cursor: default; opacity: .55; }
.rs-demo { font-size: 11.5px; font-weight: 600; color: var(--muted); padding: 1px 8px; border-radius: 99px; border: 1px dashed var(--line-strong); }
.rs-dot { width: 7px; height: 7px; border-radius: 50%; background: var(--faint); flex: none; display: inline-block; }
.rs-dot.ok { background: var(--green); } .rs-dot.run { background: var(--accent); } .rs-dot.bad { background: var(--red); }

.rs-pickstep { display: grid; gap: 16px; max-width: 760px; }
.rs-jobs { display: grid; gap: 10px; }
.rs-job { display: flex; align-items: center; gap: 14px; padding: 14px 16px; border-radius: 12px; border: 1px solid var(--line); background: var(--surface);
  cursor: pointer; text-align: left; font: inherit; color: var(--ink); transition: border-color .15s, box-shadow .15s, background .15s; }
.rs-job:hover { border-color: var(--line-strong); box-shadow: var(--shadow-md); }
.rs-job.on { border-color: var(--accent-400); box-shadow: var(--ring); background: color-mix(in srgb, var(--accent) 3%, var(--surface)); }
.rs-radio { width: 18px; height: 18px; border-radius: 50%; border: 2px solid var(--line-strong); flex: none; display: grid; place-items: center; transition: border-color .15s; }
.rs-radio::after { content: ""; width: 8px; height: 8px; border-radius: 50%; background: var(--grad); transform: scale(0); transition: transform .25s cubic-bezier(.34, 1.56, .64, 1); }
.rs-job.on .rs-radio { border-color: var(--accent); } .rs-job.on .rs-radio::after { transform: scale(1); }
.rs-job-t { display: grid; gap: 2px; flex: 1; min-width: 0; }
.rs-job-t b { font-size: 15px; font-weight: 600; }
.rs-job-t small, [data-theme="dark"] .rs-job-t small.mono { font-size: 12px; color: var(--muted); background: none; padding: 0; }
.rs-status { display: inline-flex; align-items: center; gap: 6px; font-size: 12.5px; font-weight: 600; color: var(--muted); white-space: nowrap; }
.rs-status.ok { color: var(--green-600); } .rs-status.bad { color: var(--red); } .rs-status.run { color: var(--accent-600); }

.rs-callout { display: flex; align-items: flex-start; gap: 12px; padding: 14px 16px; border-radius: 12px; border: 1px solid var(--line); background: var(--surface-2); font-size: 14px; }
.rs-callout > svg { margin-top: 2px; flex: none; }
.rs-callout > div { flex: 1; min-width: 0; }
.rs-callout b { display: block; font-weight: 600; margin-bottom: 2px; }
.rs-callout p { margin: 0; color: var(--ink-2); }
.rs-callout ul { margin: 4px 0 0; padding-left: 18px; line-height: 1.6; }
.rs-callout.warn { background: var(--amber-50); border-color: var(--amber-200); color: var(--amber-ink-2); } .rs-callout.warn b, .rs-callout.warn > svg { color: var(--amber-ink); }
.rs-callout.ok { background: var(--green-50); border-color: var(--green-200); } .rs-callout.ok > svg, .rs-callout.ok b { color: var(--green-600); }
.rs-callout.failed { background: var(--red-50); border-color: var(--red-200); } .rs-callout.failed > svg, .rs-callout.failed b { color: var(--red); }
.rs-callout.running > svg, .rs-callout.running b { color: var(--accent-600); }
.rs-hint { display: flex; align-items: center; gap: 12px; padding: 12px 16px; border-radius: 12px; border: 1px dashed var(--line-strong); color: var(--ink-2); font-size: 14px; }
.rs-hint > svg { color: var(--accent); flex: none; } .rs-hint > span { flex: 1; }

.rs-fin { display: flex; align-items: center; gap: 28px; flex-wrap: wrap; padding: 18px 22px; border-radius: 14px; border: 1px solid var(--line); background: var(--surface-2); }
.rs-fin-n { display: grid; }
.rs-fin-n b { font-size: 40px; font-weight: 750; letter-spacing: -1px; line-height: 1; background: var(--grad); -webkit-background-clip: text; background-clip: text; color: transparent; font-family: var(--font); }
.rs-fin-n span { font-size: 13px; color: var(--muted); margin-top: 4px; }
.rs-fin-l { list-style: none; margin: 0; padding: 0 0 0 28px; border-left: 1px solid var(--line); display: grid; gap: 6px; font-size: 14px; color: var(--ink-2); flex: 1; }
.rs-fin-l li { display: flex; align-items: center; gap: 8px; }
.rs-fin-l i { width: 8px; height: 8px; border-radius: 50%; background: var(--line-strong); } .rs-fin-l i.ok { background: var(--green); } .rs-fin-l i.bad { background: var(--red); }
.rs-fin-l b { font-weight: 650; color: var(--ink); min-width: 18px; }
.rs-num { width: 28px; height: 28px; border-radius: 50%; display: grid; place-items: center; font-size: 13px; font-weight: 700; background: var(--surface-3); color: var(--ink-2); flex: none; }
.rs-accept { display: flex; align-items: center; gap: 14px; padding: 16px 18px; border-radius: 14px; border: 1px solid color-mix(in srgb, var(--accent) 22%, var(--line));
  background: color-mix(in srgb, var(--accent) 4%, var(--surface)); }
.rs-accept .rs-num { background: var(--grad); color: #fff; }
.rs-accept.done { border-color: var(--green-200); background: var(--green-50); } .rs-accept.done .rs-num { background: var(--green-600); }
.rs-accept b, .rs-dl-h b { font-size: 15px; font-weight: 650; }
.rs-accept p, .rs-dl-h p { margin: 2px 0 0; font-size: 13.5px; color: var(--muted); }
.rs-accept .mono { color: var(--green-600); }
.rs-dl { display: grid; gap: 12px; padding: 16px 18px; border-radius: 14px; border: 1px solid var(--line); }
.rs-dl-h { display: flex; align-items: center; gap: 14px; }
.rs-files { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 10px; padding-left: 42px; }
.rs-file { display: flex; align-items: center; gap: 12px; padding: 11px 14px; border-radius: 11px; border: 1px solid var(--line); background: var(--surface); color: var(--ink);
  text-decoration: none; transition: border-color .15s, box-shadow .15s, transform .15s; }
.rs-file:hover { border-color: var(--accent-200); box-shadow: var(--shadow-md); transform: translateY(-1px); }
.rs-file-ico { width: 34px; height: 34px; border-radius: 9px; display: grid; place-items: center; background: var(--accent-50); color: var(--accent-600); flex: none; }
.rs-file-t { display: grid; flex: 1; min-width: 0; } .rs-file-t b { font-size: 14px; font-weight: 600; } .rs-file-t small { font-size: 12.5px; color: var(--muted); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.rs-file-n { font-size: 11.5px; color: var(--faint); }
.rs-file-dl { color: var(--faint); transition: color .15s, transform .2s; flex: none; }
.rs-file:hover .rs-file-dl { color: var(--accent); transform: translateY(2px); }

@media (max-width: 1100px) { .rs-files { grid-template-columns: 1fr; padding-left: 0; } .rs-sum { grid-template-columns: 1fr 1fr; } .rs-sum > div:nth-child(3) { border-left: 0; } .rs-sum > div:nth-child(n+3) { border-top: 1px solid var(--line-2); } .rs-ins { grid-template-columns: 1fr; } }
@media (max-width: 760px) { .qa { grid-template-columns: 1fr; } .qa-side { flex-direction: row; justify-content: space-between; } }
@media (prefers-reduced-motion: reduce) { .rs *, .rs { animation: none !important; transition: none !important; } }

/* Recharts */
.rs-pie { width: 128px; height: 128px; flex: none; position: relative; }
.rs-pie-c { position: absolute; inset: 0; display: grid; place-content: center; text-align: center; line-height: 1.1; pointer-events: none; }
.rs-pie-c b { font-size: 20px; font-weight: 750; color: var(--ink); } .rs-pie-c small { font-size: 11px; color: var(--muted); }
.rs .recharts-wrapper, .rs .recharts-surface { overflow: visible; }
.rs .recharts-wrapper *:focus { outline: none; }
.rs-tip { background: var(--surface); border: 1px solid var(--line); border-radius: 10px; padding: 9px 11px; display: grid; gap: 5px; min-width: 150px;
  box-shadow: 0 10px 30px -10px rgb(0 0 0 / .25); font-size: 12.5px; color: var(--ink-2); }
.rs-tip-h { font-size: 12.5px; font-weight: 700; color: var(--ink); text-transform: capitalize; }
.rs-tip span { display: grid; grid-template-columns: 9px minmax(0, 1fr) auto; gap: 8px; align-items: center; }
.rs-tip i { width: 9px; height: 9px; border-radius: 3px; }
.rs-tip span b { font-weight: 650; color: var(--ink); text-transform: none; }
