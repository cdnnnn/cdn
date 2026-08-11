//theme
// ─────────────────────────────────────────────────────────────────────────
// Theme tokens, exposed as CSS custom properties rather than SCSS variables
// so they can respond to a runtime dark-mode toggle (SCSS variables are
// resolved once at build time and can't change afterward).
//
// Every component already consumes these indirectly via the SCSS
// variables in _variables.scss (`$bg`, `$surface`, `$text-primary`, etc.),
// which now just point at `var(--bg)`, `var(--surface)`, etc. — so the
// whole app becomes theme-aware without touching individual component
// stylesheets. Only the neutral/background/text/border/shadow tokens and
// the "pale" badge-background tints are themed; brand accent colors
// (indigo, emerald, amber, red, sky, rose) stay constant across both
// themes since they already have enough contrast against both.
//
// The "ink" design system used by History/Reports/Comparison/New
// Evaluation/Sidebar/Landing/Dashboard/Datasets/Model Catalog/Providers
// follows the same convention: neutrals (--ink-1/2/3, --paper, --card,
// --line, --line-2) and status washes are themed here; the flat accent
// colors (signal, ok, amber, danger, sky, rose) stay constant and are
// declared locally in each component's SCSS as plain hex.
// ─────────────────────────────────────────────────────────────────────────

:root {
  --bg: #F7F8FC;
  --surface: #FFFFFF;
  --surface-alt: #F1F4F9;
  --surface-hover: #F8F9FD;

  --border: #E5E7EB;
  --border-light: #F3F4F6;

  --text-primary: #111827;
  --text-secondary: #6B7280;
  --text-muted: #9CA3AF;

  --indigo-pale: #E9EBF8;
  --amber-pale: #FFFBEB;
  --emerald-pale: #ECFDF5;
  --red-pale: #FEF2F2;
  --sky-pale: #F0F9FF;
  --rose-pale: #FFF1F2;

  --shadow-2: 0 2px 8px rgba(0, 0, 0, .06), 0 1px 2px rgba(0, 0, 0, .04);
  --shadow-3: 0 8px 24px rgba(0, 0, 0, .08), 0 2px 6px rgba(0, 0, 0, .04);
  --shadow-4: 0 16px 48px rgba(0, 0, 0, .1), 0 4px 12px rgba(0, 0, 0, .05);

  // ---- "ink" design system (History/Reports/Comparison/Sidebar/etc.) -----
  --ink-1: #14161B;
  --ink-2: #565B66;
  --ink-3: #8A909B;
  --paper: #F5F6F8;
  --card: #FFFFFF;
  --line: #E6E8EC;
  --line-2: #EEF0F3;

  // status washes (near-white in light mode)
  --ok-wash: #E7F7EF;
  --amber-wash: #FDF3E3;
  --danger-wash: #FDECEC;
  --sky-wash: #E6F3FB;
  --rose-wash: #FCE7F3;
  --ink-wash: #EEF0F2;
  --signal-wash: #ECEDFF;
}

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

  // Subtle tinted overlays instead of near-white washes, so badge/tag
  // backgrounds read correctly against a dark surface.
  --indigo-pale: rgba(76, 99, 199, .18);
  --amber-pale: rgba(245, 158, 11, .16);
  --emerald-pale: rgba(16, 185, 129, .16);
  --red-pale: rgba(239, 68, 68, .16);
  --sky-pale: rgba(14, 165, 233, .16);
  --rose-pale: rgba(244, 63, 94, .16);

  --shadow-2: 0 2px 8px rgba(0, 0, 0, .5), 0 1px 2px rgba(0, 0, 0, .4);
  --shadow-3: 0 8px 24px rgba(0, 0, 0, .55), 0 2px 6px rgba(0, 0, 0, .4);
  --shadow-4: 0 16px 48px rgba(0, 0, 0, .6), 0 4px 12px rgba(0, 0, 0, .45);

  // ---- "ink" design system, dark ------------------------------------------
  --ink-1: #ECEDF2;
  --ink-2: #A7ADC0;
  --ink-3: #7C859B;
  --paper: #0F1420;
  --card: #161B2A;
  --line: #262D42;
  --line-2: #1E2438;

  // tinted overlays instead of near-white washes
  --ok-wash: rgba(15, 169, 104, 0.16);
  --amber-wash: rgba(224, 134, 0, 0.16);
  --danger-wash: rgba(220, 38, 38, 0.18);
  --sky-wash: rgba(3, 105, 161, 0.18);
  --rose-wash: rgba(219, 39, 119, 0.18);
  --ink-wash: rgba(138, 144, 155, 0.14);
  --signal-wash: rgba(43, 43, 245, 0.18);
}



















//variable
// Design tokens. Neutral/surface/text/border/shadow/pale tokens resolve to
// CSS custom properties (defined in _theme.scss) so they respond to the
// runtime dark-mode toggle; brand accent colors stay constant across
// themes since they already contrast well against both.
$bg: var(--bg);
$surface: var(--surface);
$surface-alt: var(--surface-alt);
$surface-hover: var(--surface-hover);

$indigo: #1428A0;
$indigo-light: #4C63C7;
$indigo-dark: #0E1C74;
$violet: #2B45C9;
$indigo-pale: var(--indigo-pale);

$amber: #F59E0B;
$amber-dark: #D97706;
$amber-pale: var(--amber-pale);

$emerald: #10B981;
$emerald-dark: #059669;
$emerald-pale: var(--emerald-pale);

$red: #EF4444;
$red-pale: var(--red-pale);
$sky: #0EA5E9;
$sky-pale: var(--sky-pale);
$rose: #F43F5E;
$rose-pale: var(--rose-pale);

$border: var(--border);
$border-light: var(--border-light);

$text-primary: var(--text-primary);
$text-secondary: var(--text-secondary);
$text-muted: var(--text-muted);

$shadow-2: var(--shadow-2);
$shadow-3: var(--shadow-3);
$shadow-4: var(--shadow-4);

$footer-height: 44px;
$sidebar-width: 256px;
// Small gap reclaimed above the fixed footer when a page's scroll shell
// pulls back the workspace content wrapper's bottom padding — see
// .pg-shell in global.scss.
$page-bottom-reclaim: 0.75rem;

$grad-primary: linear-gradient(135deg, #1428A0, #2B45C9);
$grad-warm: linear-gradient(135deg, #F59E0B, #F97316);
$grad-cool: linear-gradient(135deg, #10B981, #0EA5E9);

$font-display: 'Segoe UI', Roboto, Arial, sans-serif;
$font-body: 'Segoe UI', Roboto, Arial, sans-serif;
$font-mono: 'Segoe UI', Roboto, Arial, sans-serif;

// ---------------------------------------------------------------------------
// "Ink" design system tokens — used by History, Reports, Comparison,
// New Evaluation, Sidebar, Landing, Dashboard, Datasets, Model Catalog,
// and Providers. Neutrals are theme-aware (via _theme.scss CSS vars);
// accent colors are flat constants shared across light/dark.
//
// These are provided here as a convenience — components that already
// declare their own local $ink/$paper/etc. block don't need to import
// this section, but new components can use these directly instead of
// redeclaring the block.
// ---------------------------------------------------------------------------
$ink:      var(--ink-1);
$ink-2:    var(--ink-2);
$ink-3:    var(--ink-3);
$paper:    var(--paper);
$card:     var(--card);
$line:     var(--line);
$line-2:   var(--line-2);

$signal:   #2B2BF5;
$signal-2: #1C1CC7;
$wash:     var(--signal-wash);

$ok:       #0FA968;
$ok-wash:  var(--ok-wash);

$amber-ink: #E08600;
$amber-ink-wash: var(--amber-wash);

$danger:   #DC2626;
$danger-wash: var(--danger-wash);

$ink-wash: var(--ink-wash);

$sky-ink:  #0369A1;
$sky-ink-wash: var(--sky-wash);

$rose-ink: #DB2777;
$rose-ink-wash: var(--rose-wash);










