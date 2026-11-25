//src/palettes.css
/* Colour themes for the Customize panel. Sunset coral is the default and lives in styles.css / theme-dark.css;
   each block below swaps only the theme colours (accent, gradient, tints, logo) for light and, after it, dark mode. */

/* ── Royal blue: one colour, #2B2BF5, and its shades ── */
:root[data-palette="royal"] {
  --grad-a: #5C5CF8; --grad-b: #2B2BF5; --grad-a-rgb: 92, 92, 248; --grad-b-rgb: 43, 43, 245;
  --grad: linear-gradient(135deg, var(--grad-a), var(--grad-b)); --grad-hover: linear-gradient(135deg, #3B3BF6, #1C1CC9);
  --accent: #2B2BF5; --accent-rgb: 43, 43, 245; --accent-600: #1F1FD1; --accent-400: #6B6BF8;
  --accent-200: #CACAFD; --accent-100: #E3E3FE; --accent-50: #F3F3FF; --accent-ink: #0E0E5C;
  --tint-0: #FAFAFF; --tint-1: #F1F1FF; --tint-2: #EEEEFE;
  --violet: #4646F6; --violet-50: #F0F0FF; --violet-100: #E3E3FE; --violet-ink: #13137A;
  --mix-1: #2B2BF5; --mix-2: #B4B4FB; --split-1: #2B2BF5;
  --lg-l1a: #A5A5FC; --lg-l1b: #4B4BF7; --lg-l1c: #2B2BF5; --lg-l1e: #1A1AB0;
  --lg-l2a: #7C7CF9; --lg-l2b: #2B2BF5; --lg-l2e: #16169A; --lg-l2s: 22, 22, 154;
  --lg-l3a: #B9B9FC; --lg-l3b: #5050F7; --lg-l3e: #1F1FA8; --lg-l3s: 31, 31, 168;
  --lg-halo: 43, 43, 245; --lg-glow: 227, 227, 254;
}
:root[data-theme="dark"][data-palette="royal"] {
  /* lifted shades of the same blue, so text and lines stay readable on the dark background */
  --grad-a: #7A7AFA; --grad-b: #3B3BF6; --grad-a-rgb: 122, 122, 250; --grad-b-rgb: 59, 59, 246;
  --accent: #7373FA; --accent-rgb: 115, 115, 250; --accent-600: #A3A3FF; --accent-400: #8585F9;
  --accent-200: #26266B; --accent-100: #1C1C4A; --accent-50: #151532; --accent-ink: #E0E0FF;
  --tint-0: #15151d; --tint-1: #181828; --tint-2: #19192b;
  --violet: #8C8CFB; --violet-50: #1b1b36; --violet-100: #25254d; --violet-ink: #E3E3FE;
  --mix-1: #7373FA; --mix-2: #4A4AA8; --split-1: #7373FA;
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

/* ── fonts ── the system font (Segoe UI) is the default, set in styles.css.
   The other fonts come from the @fontsource-variable npm packages (imported in customize.js);
   their family names end in "Variable". */
:root[data-font="inter"] { --font: "Inter Variable", "Segoe UI", system-ui, sans-serif; }
:root[data-font="jakarta"] { --font: "Plus Jakarta Sans Variable", "Segoe UI", system-ui, sans-serif; }
:root[data-font="manrope"] { --font: "Manrope Variable", "Segoe UI", system-ui, sans-serif; }
:root[data-font="dmsans"] { --font: "DM Sans Variable", "Segoe UI", system-ui, sans-serif; }




















//src/customize.js
// User customization: colour theme and font. Kept in sessionStorage, so it lasts for this browser tab's
// session and resets when the tab is closed. The defaults are the original coral theme and the system font.
import { useEffect, useState } from "react";
// Fonts from npm (@fontsource-variable/*). Each package only declares its @font-face rules;
// the browser downloads a font file only once that font is picked, so unused fonts cost nothing.
import "@fontsource-variable/inter";
import "@fontsource-variable/plus-jakarta-sans";
import "@fontsource-variable/manrope";
import "@fontsource-variable/dm-sans";
import "./palettes.css";

const KEY = "slm-custom";

export const PALETTES = [
  { id: "coral", name: "Sunset coral", note: "Warm orange to pink", a: "#F97316", b: "#EC4899" },
  { id: "royal", name: "Royal blue", note: "One blue, #2B2BF5, in shades", a: "#5C5CF8", b: "#2B2BF5" },
  { id: "violet", name: "Amethyst", note: "Violet to fuchsia", a: "#8B5CF6", b: "#D946EF" },
  { id: "emerald", name: "Emerald", note: "Green to cyan", a: "#10B981", b: "#06B6D4" },
];

export const FONTS = [
  { id: "system", name: "Segoe UI", note: "System font", stack: '"Segoe UI Variable Text", "Segoe UI", system-ui, -apple-system, "Helvetica Neue", Arial, sans-serif' },
  { id: "inter", name: "Inter", note: "Neutral and very legible", stack: '"Inter Variable", "Segoe UI", system-ui, sans-serif' },
  { id: "jakarta", name: "Plus Jakarta Sans", note: "Friendly and modern", stack: '"Plus Jakarta Sans Variable", "Segoe UI", system-ui, sans-serif' },
  { id: "manrope", name: "Manrope", note: "Geometric and crisp", stack: '"Manrope Variable", "Segoe UI", system-ui, sans-serif' },
  { id: "dmsans", name: "DM Sans", note: "Soft and compact", stack: '"DM Sans Variable", "Segoe UI", system-ui, sans-serif' },
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
