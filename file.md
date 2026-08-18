@use '../../styles/_variables' as *;

// ===========================================================================
// Sidebar — matches History / Reports / Comparison / New Evaluation design
// system: ink/paper palette, ultramarine signal accent, mono instrument
// labels, hover-lift.
//
// All neutral/surface/border colors now come from the shared "ink" design
// system tokens in _variables.scss (which resolve to the CSS custom
// properties in _theme.scss), so the sidebar is dark-mode aware without any
// local hex values. Only $signal/$signal-2 (brand accent) and $danger
// (status accent) remain — those are already flat constants shared across
// themes, sourced from _variables.scss rather than redeclared here.
//
// Font scaling: `.sidebar` sets a single base font-size. All descendant
// font-sizes are expressed in `em` (relative to that base), so bumping
// `.sidebar`'s font-size (e.g. on wide screens) scales the whole component
// proportionally from one place.
// ===========================================================================

// base font-size the sidebar's internal `em` scale is built on
$sidebar-base-font: 0.875rem;

%micro {
  font-family: $font-mono;
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

.sidebar {
  width: $sidebar-width;
  background: $card;
  border-right: 1px solid $line;
  display: flex;
  flex-direction: column;
  flex-shrink: 0;

  // master scale control — every em-based font-size below responds to this
  font-size: $sidebar-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  &__logo {
    padding: 22px 24px;
    display: flex;
    align-items: center;
    gap: 11px;
    font-family: $font-display;
    font-size: 1.2143em; // 1.0625rem / 0.875rem
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $ink;
    border-bottom: 1px solid $line;
    text-decoration: none;
    cursor: pointer;
    transition: opacity 0.15s ease;

    &:hover { opacity: 0.82; }
  }

  &__mark {
    flex-shrink: 0;
    width: 30px;
    height: 30px;
    border-radius: 9px;
    display: flex;
    align-items: center;
    justify-content: center;
    color: #fff;
    background: $ink-solid;
    font-size: 0.9286em; // 0.875rem / 0.875rem base → matches original abs size
    position: relative;
    overflow: hidden;

    &::after {
      content: '';
      position: absolute;
      inset: 0;
      background: linear-gradient(140deg, transparent 40%, rgba($signal, 0.9) 140%);
    }
  }

  &__nav {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding: 14px 12px;
    display: flex;
    flex-direction: column;
    gap: 2px;
  }

  &__section {
    @extend %micro;
    font-size: 0.7143em; // 0.625rem / 0.875rem
    color: $ink-3;
    padding: 20px 14px 8px;
  }

  &__foot {
    flex-shrink: 0;
    padding: 16px;
    border-top: 1px solid $line;
  }

  &__theme-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 6px 4px 14px;
    font-size: 0.8571em; // 0.75rem / 0.875rem
    font-weight: 650;
    color: $ink-2;
  }

  &__user {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 9px 10px;
    border-radius: 12px;
    border: 1px solid transparent;
    transition: background 0.15s ease, border-color 0.15s ease;

    &:hover { background: $paper; border-color: $line; }
  }

  &__avatar {
    flex-shrink: 0;
    width: 36px;
    height: 36px;
    border-radius: 10px;
    display: flex;
    align-items: center;
    justify-content: center;
    background: $ink-solid;
    color: #fff;
    font-family: $font-display;
    font-size: 0.9286em; // 0.8125rem / 0.875rem
    font-weight: 700;
    position: relative;
    overflow: hidden;

    &::after {
      content: '';
      position: absolute;
      inset: 0;
      background: linear-gradient(140deg, transparent 40%, rgba($signal, 0.9) 140%);
    }
  }

  &__user-info {
    flex: 1;
    min-width: 0;
  }

  &__user-name {
    font-size: 0.9286em; // 0.8125rem / 0.875rem
    font-weight: 700;
    color: $ink;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__user-email {
    font-family: $font-mono;
    font-size: 0.7857em; // 0.6875rem / 0.875rem
    color: $ink-3;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__logout {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 28px;
    height: 28px;
    border-radius: 8px;
    border: 1px solid transparent;
    background: transparent;
    color: $ink-3;
    cursor: pointer;
    transition: background 0.15s ease, color 0.15s ease, border-color 0.15s ease;

    &:hover {
      background: $danger-wash;
      border-color: rgba($danger, 0.2);
      color: $danger;
    }
  }
}

.nav-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 10px 14px;
  border-radius: 10px;
  font-size: 0.9643em; // 0.84375rem / 0.875rem
  font-weight: 550;
  color: $ink-2;
  cursor: pointer;
  transition: color 0.15s ease, background 0.15s ease;
  border: none;
  background: none;
  width: 100%;
  text-align: left;
  text-decoration: none;

  svg { flex-shrink: 0; color: $ink-3; transition: color 0.15s ease; }

  &:hover {
    color: $ink;
    background: $paper;
    svg { color: $ink-2; }
  }

  &.active {
    color: $signal-active;
    background: $wash;
    font-weight: 700;
    box-shadow: inset 2.5px 0 0 $signal-active;

    svg { color: $signal-active; }
  }
}

@media (max-width: 768px) {
  .sidebar { display: none; }
}












//theme.scss
//_theme.scss
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
//
// --ink-solid / --ink-solid-hover / --ink-solid-ok are a special case:
// they back "always dark" surfaces (option-type icon chips, primary
// buttons, toast notifications) that must stay legible against content
// that itself changes color with the theme — using the themed --ink-1
// there would go near-white (and invisible) in dark mode. They're still
// expressed as CSS custom properties for the same centralization reason
// as everything else here, but deliberately hold the *same* value in
// both the light and dark blocks below — that's intentional, not a
// missed override.
//
// --signal-active is a similar special case, but the reverse: it's the
// accent color used for "active/selected" state (e.g. Sidebar nav-item).
// Unlike other accents it is NOT identical across themes — flat $signal
// (#2B2BF5) reads fine on light surfaces but loses contrast/legibility on
// dark ones, so dark mode gets a brightened tint instead. --signal-wash
// (the pale background behind the active state) is bumped in dark mode
// for the same reason — the light-mode alpha is too faint to read against
// a dark card.
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

  // "always dark" surfaces — same value in both themes, see note above
  --ink-solid: #14161B;
  --ink-solid-hover: #000000;
  --ink-solid-ok: #34D399;

  // active/selected-state accent — same as brand signal in light mode, but
  // brightened in dark mode where flat $signal reads dull against dark
  // surfaces (used for active nav item text/icon/indicator)
  --signal-active: #2B2BF5;
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
  // bumped from 0.18 — the light-mode alpha was too faint to read as an
  // active-state background against a dark card
  --signal-wash: rgba(43, 43, 245, 0.28);

  // "always dark" surfaces — intentionally identical to :root above
  --ink-solid: #14161B;
  --ink-solid-hover: #000000;
  --ink-solid-ok: #34D399;

  // brightened tint of $signal — flat #2B2BF5 loses legibility/contrast on
  // dark surfaces (used for active nav item text/icon/indicator)
  --signal-active: #6C6CFF;
}














//variables.scss
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
// Active/selected-state accent — brand $signal in light mode, brightened
// in dark mode where flat $signal loses contrast against dark surfaces.
// Use this (not $signal) for active nav items, selected tabs, etc.
$signal-active: var(--signal-active);
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
// Violet accent — currently used by New Evaluation's Agent-type icon chip
// (&__option-icon--agent). Added here alongside $sky-ink/$rose-ink rather
// than staying hardcoded locally, following the same flat-constant
// convention (brand accents don't vary by theme).
$violet-ink: #6D28D9;
// "Always dark" surfaces — see the note in _theme.scss above these CSS
// vars. Used where content must stay legible against a background that
// intentionally does NOT invert with the theme (primary buttons, "always
// dark" icon chips, toast notifications) — using the themed $ink there
// would go near-white (and invisible) in dark mode.
$ink-solid:       var(--ink-solid);
$ink-solid-hover: var(--ink-solid-hover);
$ink-solid-ok:    var(--ink-solid-ok);
