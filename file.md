/* ═══════════════════════════════════════════════════════════════════════
   DARK THEME TOKEN OPTIONS
   Drop ONE of these [data-theme='dark'] blocks in, replacing your current
   one. Light-mode :root block is unchanged in all four — only dark tokens
   differ. Brand accents (indigo/emerald/amber/red/sky/rose) still stay
   constant per your original convention; only wash/neutral tokens shift.
   ═══════════════════════════════════════════════════════════════════════ */


/* ───────────────────────────────────────────────────────────────────────
   OPTION A — Current (baseline, included for reference/diffing)
   ─────────────────────────────────────────────────────────────────────── */
[data-theme='dark'] {
  --bg: #0B0F1A;
  --surface: #131826;
  --surface-alt: #1B2136;
  --surface-hover: #1F2540;

  --border: #262D42;
  --border-light: #1E2438;

  --text-primary: #F3F4F6;
  --text-secondary: #A7ADC0;
  --text-muted: #7C859B;

  --indigo-pale: rgba(76, 99, 199, .18);
  --amber-pale: rgba(245, 158, 11, .16);
  --emerald-pale: rgba(16, 185, 129, .16);
  --red-pale: rgba(239, 68, 68, .16);
  --sky-pale: rgba(14, 165, 233, .16);
  --rose-pale: rgba(244, 63, 94, .16);

  --shadow-2: 0 2px 8px rgba(0, 0, 0, .5), 0 1px 2px rgba(0, 0, 0, .4);
  --shadow-3: 0 8px 24px rgba(0, 0, 0, .55), 0 2px 6px rgba(0, 0, 0, .4);
  --shadow-4: 0 16px 48px rgba(0, 0, 0, .6), 0 4px 12px rgba(0, 0, 0, .45);

  --ink-1: #ECEDF2;
  --ink-2: #A7ADC0;
  --ink-3: #7C859B;
  --paper: #0F1420;
  --card: #161B2A;
  --line: #262D42;
  --line-2: #1E2438;

  --ok-wash: rgba(15, 169, 104, 0.16);
  --amber-wash: rgba(224, 134, 0, 0.16);
  --danger-wash: rgba(220, 38, 38, 0.18);
  --sky-wash: rgba(3, 105, 161, 0.18);
  --rose-wash: rgba(219, 39, 119, 0.18);
  --ink-wash: rgba(138, 144, 155, 0.14);
  --signal-wash: rgba(43, 43, 245, 0.18);
}


/* ───────────────────────────────────────────────────────────────────────
   OPTION B — Slate Neutral (GitHub-dark inspired)
   Cooler, desaturated neutrals with slightly higher line contrast.
   ─────────────────────────────────────────────────────────────────────── */
[data-theme='dark'] {
  --bg: #0D1117;
  --surface: #161B22;
  --surface-alt: #1C2128;
  --surface-hover: #21262D;

  --border: #30363D;
  --border-light: #21262D;

  --text-primary: #E6EDF3;
  --text-secondary: #8B949E;
  --text-muted: #6E7681;

  --indigo-pale: rgba(88, 101, 242, .18);
  --amber-pale: rgba(210, 153, 34, .16);
  --emerald-pale: rgba(63, 185, 80, .16);
  --red-pale: rgba(248, 81, 73, .16);
  --sky-pale: rgba(56, 139, 253, .16);
  --rose-pale: rgba(219, 88, 138, .16);

  --shadow-2: 0 2px 8px rgba(0, 0, 0, .55), 0 1px 2px rgba(0, 0, 0, .45);
  --shadow-3: 0 8px 24px rgba(0, 0, 0, .6), 0 2px 6px rgba(0, 0, 0, .45);
  --shadow-4: 0 16px 48px rgba(0, 0, 0, .65), 0 4px 12px rgba(0, 0, 0, .5);

  --ink-1: #E6EDF3;
  --ink-2: #8B949E;
  --ink-3: #6E7681;
  --paper: #0D1117;
  --card: #161B22;
  --line: #30363D;
  --line-2: #21262D;

  --ok-wash: rgba(63, 185, 80, 0.15);
  --amber-wash: rgba(210, 153, 34, 0.15);
  --danger-wash: rgba(248, 81, 73, 0.15);
  --sky-wash: rgba(56, 139, 253, 0.15);
  --rose-wash: rgba(219, 88, 138, 0.15);
  --ink-wash: rgba(139, 148, 158, 0.14);
  --signal-wash: rgba(88, 101, 242, 0.18);
}


/* ───────────────────────────────────────────────────────────────────────
   OPTION C — Warm Charcoal (iOS/Material style)
   True-black-leaning, warm-neutral greys. Less blue cast, calmer for
   long sessions.
   ─────────────────────────────────────────────────────────────────────── */
[data-theme='dark'] {
  --bg: #121212;
  --surface: #1C1C1E;
  --surface-alt: #232325;
  --surface-hover: #2A2A2C;

  --border: #2E2E30;
  --border-light: #242426;

  --text-primary: #F2F2F2;
  --text-secondary: #A1A1A6;
  --text-muted: #767678;

  --indigo-pale: rgba(94, 106, 210, .18);
  --amber-pale: rgba(255, 159, 10, .16);
  --emerald-pale: rgba(48, 209, 88, .16);
  --red-pale: rgba(255, 69, 58, .16);
  --sky-pale: rgba(100, 210, 255, .16);
  --rose-pale: rgba(255, 55, 95, .16);

  --shadow-2: 0 2px 8px rgba(0, 0, 0, .5), 0 1px 2px rgba(0, 0, 0, .4);
  --shadow-3: 0 8px 24px rgba(0, 0, 0, .55), 0 2px 6px rgba(0, 0, 0, .4);
  --shadow-4: 0 16px 48px rgba(0, 0, 0, .6), 0 4px 12px rgba(0, 0, 0, .45);

  --ink-1: #F2F2F2;
  --ink-2: #A1A1A6;
  --ink-3: #767678;
  --paper: #121212;
  --card: #1C1C1E;
  --line: #2E2E30;
  --line-2: #242426;

  --ok-wash: rgba(48, 209, 88, 0.16);
  --amber-wash: rgba(255, 159, 10, 0.16);
  --danger-wash: rgba(255, 69, 58, 0.18);
  --sky-wash: rgba(100, 210, 255, 0.16);
  --rose-wash: rgba(255, 55, 95, 0.16);
  --ink-wash: rgba(161, 161, 166, 0.14);
  --signal-wash: rgba(94, 106, 210, 0.18);
}


/* ───────────────────────────────────────────────────────────────────────
   OPTION D — Indigo Tint
   Keeps a subtle indigo cast in neutrals to echo the brand accent, so
   surfaces read as branded rather than generic dark grey.
   ─────────────────────────────────────────────────────────────────────── */
[data-theme='dark'] {
  --bg: #0E1024;
  --surface: #171A36;
  --surface-alt: #1D2140;
  --surface-hover: #232748;

  --border: #2A2E52;
  --border-light: #202444;

  --text-primary: #EDEEFB;
  --text-secondary: #A7ACD6;
  --text-muted: #7D82B0;

  --indigo-pale: rgba(99, 112, 230, .2);
  --amber-pale: rgba(251, 191, 36, .16);
  --emerald-pale: rgba(52, 211, 153, .16);
  --red-pale: rgba(248, 113, 113, .16);
  --sky-pale: rgba(96, 165, 250, .16);
  --rose-pale: rgba(251, 113, 133, .16);

  --shadow-2: 0 2px 8px rgba(0, 0, 4, .5), 0 1px 2px rgba(0, 0, 4, .4);
  --shadow-3: 0 8px 24px rgba(0, 0, 4, .55), 0 2px 6px rgba(0, 0, 4, .4);
  --shadow-4: 0 16px 48px rgba(0, 0, 4, .6), 0 4px 12px rgba(0, 0, 4, .45);

  --ink-1: #EDEEFB;
  --ink-2: #A7ACD6;
  --ink-3: #7D82B0;
  --paper: #0E1024;
  --card: #171A36;
  --line: #2A2E52;
  --line-2: #202444;

  --ok-wash: rgba(52, 211, 153, 0.16);
  --amber-wash: rgba(251, 191, 36, 0.16);
  --danger-wash: rgba(248, 113, 113, 0.18);
  --sky-wash: rgba(96, 165, 250, 0.18);
  --rose-wash: rgba(251, 113, 133, 0.18);
  --ink-wash: rgba(167, 172, 214, 0.14);
  --signal-wash: rgba(99, 112, 230, 0.2);
}
