//global.scss
@use './variables' as *;

:root {
  color-scheme: light;

  --primary: #1428a0;
  --primary-hover: #1d37c9;
  --primary-light: #eef1fe;
  --primary-subtle: #e2e7fc;

  --violet: #7c3aed;
  --violet-light: #f3e8ff;

  --card-accent: #14a08d;

  --bg-page: #f6f7f9;
  --bg-subtle: #f3f5f8;
  --bg-inset: #edf0f4;
  --bg-main: #ffffff;
  --bg-header-glass: rgba(255, 255, 255, 0.88);

  --border-default: #dce0e7;
  --border-subtle: #e9ecf1;
  --border-strong: #c7cdd8;

  --text-primary: #0e1526;
  --text-secondary: #46506b;
  --text-tertiary: #7a8399;

  --success: #0f7a5a;
  --success-subtle: #e4f4ee;
  --warning: #b7791f;
  --warning-subtle: #fdf3e0;
  --danger: #c0303b;
  --danger-subtle: #fcebec;

  --shadow-xs: 0 0.0625rem 0.125rem rgba(14, 21, 38, 0.04);
  --shadow-sm: 0 0.0625rem 0.125rem rgba(14, 21, 38, 0.05);
  --shadow-md: 0 0.125rem 0.25rem rgba(14, 21, 38, 0.05), 0 0.5rem 1.25rem -0.75rem rgba(14, 21, 38, 0.16);
  --shadow-lg: 0 0.25rem 0.5rem rgba(14, 21, 38, 0.05), 0 1.125rem 2.75rem -1.375rem rgba(14, 21, 38, 0.24);
  --shadow-xl: 0 1.75rem 4.375rem -1.875rem rgba(14, 21, 38, 0.34);
}

[data-theme='dark'] {
  color-scheme: dark;

  --primary: #6c8cff;
  --primary-hover: #85a3ff;
  --primary-light: #141c38;
  --primary-subtle: #1d2748;

  --violet: #c4a6ff;
  --violet-light: #1c1733;

  --card-accent: #4fd6c0;

  // True-black page with near-black cards sitting just barely above it —
  // bg-subtle/bg-inset step up in small, even increments so table headers,
  // input wells, and hover states stay readable without the elevation
  // jumps feeling abrupt against a pure-black base.
  --bg-page: #000000;
  --bg-main: #0d0d0d;
  --bg-subtle: #131313;
  --bg-inset: #1a1a1a;
  --bg-header-glass: rgba(0, 0, 0, 0.75);

  // Borders carry more of the depth signal here than shadows do, since a
  // black shadow is invisible against a black page — so these are a touch
  // brighter/more frequent than a typical dark theme would need.
  --border-default: #292929;
  --border-subtle: #1c1c1c;
  --border-strong: #3d3d3d;

  --text-primary: #f5f5f5;
  --text-secondary: #a8a8a8;
  --text-tertiary: #767676;

  --success: #34d399;
  --success-subtle: #0c1f18;
  --warning: #fbbf4a;
  --warning-subtle: #241c0c;
  --danger: #fb7185;
  --danger-subtle: #2a1014;

  // Drop shadows barely register on true black, so depth here leans on the
  // inset top highlight (a hairline of light catching each card's top edge)
  // more than usual — bumped up from the previous dark palette so cards and
  // dropdowns still read as "lifted" rather than flat against the page.
  --shadow-xs: 0 1px 2px rgba(0, 0, 0, 0.5), inset 0 1px 0 rgba(255, 255, 255, 0.04);
  --shadow-sm: 0 1px 3px rgba(0, 0, 0, 0.55), inset 0 1px 0 rgba(255, 255, 255, 0.05);
  --shadow-md: 0 0.5rem 1.75rem -0.75rem rgba(0, 0, 0, 0.7), inset 0 1px 0 rgba(255, 255, 255, 0.06);
  --shadow-lg: 0 1.125rem 3rem -1.25rem rgba(0, 0, 0, 0.75), inset 0 1px 0 rgba(255, 255, 255, 0.07);
  --shadow-xl: 0 1.75rem 5rem -1.5rem rgba(0, 0, 0, 0.85), inset 0 1px 0 rgba(255, 255, 255, 0.08);
}

* ,
*::before,
*::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

// Slim scrollbars everywhere — applies to every scrollable element in the
// app (sidebars, detail panels, tables, dropdowns) automatically, since
// nothing needs to opt in individually. Firefox via scrollbar-width/-color,
// Chromium/Safari/Edge via the ::-webkit-scrollbar pseudo-elements. Colors
// use tokens, so this follows light/dark mode with no extra work.
* {
  scrollbar-width: thin;
  scrollbar-color: $border-strong transparent;
}

::-webkit-scrollbar {
  width: 8px;
  height: 8px;
}

::-webkit-scrollbar-track {
  background: transparent;
}

::-webkit-scrollbar-thumb {
  background: $border-strong;
  border-radius: 999px;
  border: 2px solid transparent;
  background-clip: padding-box;

  &:hover {
    background: $text-tertiary;
    background-clip: padding-box;
  }
}

html {
  -webkit-font-smoothing: antialiased;
  scroll-behavior: smooth;
  font-size: 100%; // 1rem = 16px, respects user browser settings
}

body {
  font-family: $font-body;
  background: $bg-main;
  color: $text-primary;
  font-size: 1.0625rem;
  line-height: 1.55;
  transition: background-color 0.16s ease, color 0.16s ease;
}

a {
  color: inherit;
}

button,
input,
select,
textarea {
  font-family: inherit;
}

h1,
h2,
h3 {
  font-family: $font-display;
  letter-spacing: -0.025em;
  line-height: 1.12;
  font-weight: 700;
}

/* numbers hold their columns without a monospaced face */
.n {
  font-variant-numeric: tabular-nums;
  font-feature-settings: 'tnum' 1, 'lnum' 1;
}

:where(a, button, input, select, textarea, [tabindex]):focus-visible {
  outline: 0.125rem solid $primary;
  outline-offset: 0.125rem;
  border-radius: 0.25rem;
}

::selection {
  background: $primary-subtle;
  color: $text-primary;
}

@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
























//variables.scss
// ============================================================
// SemcoEval — design tokens
// Base: 1rem = 16px
//
// Color / shadow tokens are aliases for CSS custom properties.
// The actual light + dark values live in global.scss (:root and
// [data-theme='dark']), so every component that already uses
// $primary, $bg-main, $shadow-md, etc. gets dark mode automatically —
// nothing here needs to change per-component.
// ============================================================

// ---------- Brand / primary ----------
$primary: var(--primary);
$primary-hover: var(--primary-hover);
$primary-light: var(--primary-light);
$primary-subtle: var(--primary-subtle);

// ---------- Secondary accent (used by violet badges/pills across pages) ----------
$violet: var(--violet);
$violet-light: var(--violet-light);

// Teal accent for the Datasets full-info card's left border — distinct from
// $primary so cards read as their own visual category, with a brightened
// equivalent hue for contrast against near-black in dark mode.
$card-accent: var(--card-accent);

// ---------- Surfaces ----------
$bg-page: var(--bg-page);
$bg-subtle: var(--bg-subtle);
$bg-inset: var(--bg-inset);
$bg-main: var(--bg-main);
$bg-header-glass: var(--bg-header-glass);

// ---------- Borders ----------
$border-default: var(--border-default);
$border-subtle: var(--border-subtle);
$border-strong: var(--border-strong);

// ---------- Text ----------
$text-primary: var(--text-primary);
$text-secondary: var(--text-secondary);
$text-tertiary: var(--text-tertiary);

// ---------- Status ----------
$success: var(--success);
$success-subtle: var(--success-subtle);
$warning: var(--warning);
$warning-subtle: var(--warning-subtle);
$danger: var(--danger);
$danger-subtle: var(--danger-subtle);

// ---------- On-fill text (mode-independent) ----------
// Text sitting on a saturated fill like $primary or $success stays white in
// both themes since those fills are always bright/saturated regardless of
// mode — this documents that intent instead of leaving raw #fff scattered
// through component files.
$on-primary: #fff;

// ---------- Shadows ----------
$shadow-xs: var(--shadow-xs);
$shadow-sm: var(--shadow-sm);
$shadow-md: var(--shadow-md);
$shadow-lg: var(--shadow-lg);
$shadow-xl: var(--shadow-xl);

// ---------- Radius (mode-independent) ----------
$radius-sm: 0.375rem;
$radius-md: 0.5rem;
$radius-lg: 0.75rem;
$radius-xl: 1rem;

// ---------- Typography (mode-independent) ----------
$font-display: 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
$font-body: 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
$font-mono: 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;

// ---------- Layout (mode-independent) ----------
$header-height: 60px;
$footer-height: 30px;
$sidebar-width: 240px;

// ---------- Z-index (mode-independent) ----------
$z-header: 100;
$z-footer: 100;
$z-sidebar: 90;



















//Datasets.scss
@use '../../../styles/variables' as *;

.datasets-page {
  display: flex;
  flex-direction: column;
  gap: 16px;

  /* ---------- header ---------- */
  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding-bottom: 18px;
    margin-bottom: 2px;
    border-bottom: 1px solid $border-subtle;
  }

  &__header-left {
    display: flex;
    flex-direction: column;
  }

  &__header-right {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 10px;
  }

  &__header-eyebrow {
    display: flex;
    align-items: center;
    gap: 8px;
    font-family: $font-mono;
    font-size: 0.6875rem;
    font-weight: 700;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    color: $primary;
    margin-bottom: 6px;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $primary;
    }
  }

  &__header-meta {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-secondary;
    background: $bg-subtle;
    border: 1px solid $border-subtle;
    border-radius: 999px;
    padding: 7px 13px;
    white-space: nowrap;
  }

  &__title {
    font-size: 21px;
    font-weight: 800;
    letter-spacing: -0.02em;
    color: $text-primary;
  }

  &__subtitle {
    margin-top: 3px;
    color: $text-secondary;
    font-size: 0.84375rem;
  }

  &__btn {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    font-family: $font-body;
    font-size: 0.8125rem;
    font-weight: 600;
    padding: 9px 14px;
    border-radius: 8px;
    border: 1px solid transparent;
    cursor: pointer;
    white-space: nowrap;
    transition: background 0.14s ease, border-color 0.14s ease, color 0.14s ease, opacity 0.14s ease;

    &:disabled {
      opacity: 0.6;
      cursor: not-allowed;
    }

    &--outline {
      background: $bg-main;
      border-color: $border-default;
      color: $text-secondary;

      &:hover:not(:disabled) {
        border-color: $text-primary;
        color: $text-primary;
      }
    }

    &--primary {
      background: $primary;
      border-color: $primary;
      color: $on-primary;

      &:hover {
        background: $primary-hover;
        border-color: $primary-hover;
      }
    }
  }

  /* ---------- filters ---------- */
  &__filters {
    flex-shrink: 0;
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 12px;
  }

  &__search {
    display: flex;
    align-items: center;
    gap: 9px;
    width: 280px;
    max-width: 100%;
    border: 1px solid $border-default;
    border-radius: 10px;
    padding: 9px 12px;
    background: $bg-main;
    color: $text-tertiary;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &:focus-within {
      border-color: $primary;
      box-shadow: 0 0 0 3px $primary-light;
    }

    input {
      flex: 1;
      border: none;
      outline: none;
      font-size: 0.8125rem;
      color: $text-primary;
      background: transparent;
      font-family: $font-body;
      min-width: 0;

      &::placeholder {
        color: $text-tertiary;
      }
    }
  }

  &__search-clear {
    flex-shrink: 0;
    width: 18px;
    height: 18px;
    border-radius: 50%;
    border: none;
    background: $bg-inset;
    color: $text-tertiary;
    display: grid;
    place-items: center;
    cursor: pointer;
    transition: background 0.14s ease, color 0.14s ease;

    &:hover {
      background: $border-default;
      color: $text-primary;
    }
  }

  &__seg {
    flex-shrink: 0;
    display: inline-flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 2px;
    padding: 3px;
    border: 1px solid $border-subtle;
    border-radius: 11px;
    background: $bg-subtle;
  }

  &__seg-item {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-family: $font-body;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-tertiary;
    background: transparent;
    border: none;
    border-radius: 8px;
    padding: 7px 12px;
    cursor: pointer;
    transition: background 0.14s ease, color 0.14s ease, box-shadow 0.14s ease;

    svg {
      opacity: 0.8;
    }

    &:hover {
      color: $text-primary;
    }

    &--active {
      background: $bg-main;
      color: $primary;
      box-shadow: $shadow-xs;

      svg {
        opacity: 1;
      }
    }
  }

  /* ---------- tags ---------- */
  &__tag {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 4px;
    font-size: 0.6875rem;
    font-weight: 700;
    border-radius: 999px;
    padding: 3px 10px;

    &--blue {
      color: $primary;
      background: $primary-light;
    }
  }

  /* ---------- capability pills (shared) ---------- */
  &__caps {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    margin-bottom: 16px;
  }

  &__cap-pill {
    font-size: 0.71875rem;
    font-weight: 600;
    border-radius: 999px;
    padding: 3px 10px;

    &--blue {
      color: $primary;
      background: $primary-light;
    }

    &--violet {
      color: $violet;
      background: $violet-light;
    }

    &--amber {
      color: $warning;
      background: $warning-subtle;
    }

    &--jade {
      color: $success;
      background: $success-subtle;
    }

    &--rose {
      color: $danger;
      background: $danger-subtle;
    }
  }

  /* ---------- full-info card grid ---------- */
  &__grid {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 16px;
  }

  &__card {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-left: 3px solid $card-accent;
    border-radius: 12px;
    padding: 18px 20px;
    box-shadow: $shadow-xs;
    transition: box-shadow 0.15s ease, transform 0.15s ease;

    &:hover {
      box-shadow: $shadow-md;
      transform: translateY(-2px);
    }
  }

  &__card-top {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 10px;
    margin-bottom: 8px;
  }

  &__card-name {
    font-size: 0.9375rem;
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $text-primary;
  }

  &__card-desc {
    font-size: 0.78125rem;
    color: $text-secondary;
    line-height: 1.55;
    margin-bottom: 14px;
  }

  &__card-stats {
    display: flex;
    gap: 20px;
    margin-bottom: 14px;
    padding-bottom: 14px;
    border-bottom: 1px solid $border-subtle;
  }

  &__card-stat-label {
    font-size: 0.625rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.04em;
    color: $text-tertiary;
  }

  &__card-stat-value {
    font-size: 0.9375rem;
    font-weight: 800;
    color: $text-primary;
    display: block;
    margin-top: 2px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;

    &--sm {
      font-size: 0.78125rem;
      font-weight: 700;
    }
  }

  &__card-section-label {
    font-size: 0.65625rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    color: $text-tertiary;
    margin-bottom: 8px;
    display: block;
  }

  &__card-tasks {
    display: flex;
    flex-direction: column;
    gap: 6px;
    margin-bottom: 10px;

    &--inline {
      flex-direction: row;
      flex-wrap: wrap;
      align-items: baseline;
      gap: 0 6px;
    }
  }

  &__card-task {
    font-size: 0.75rem;
    line-height: 1.55;

    b {
      color: $text-primary;
      font-weight: 700;
    }

    span {
      color: $text-secondary;
    }

    .datasets-page__card-tasks--inline & {
      display: inline;
      line-height: 1.7;

      &:not(:first-child)::before {
        content: '';
        display: inline-block;
        width: 4px;
        height: 4px;
        border-radius: 50%;
        background: $primary;
        margin-right: 6px;
        vertical-align: middle;
      }
    }
  }

  &__card-view-all {
    display: inline-flex;
    align-items: center;
    font-family: $font-body;
    font-size: 0.71875rem;
    font-weight: 700;
    color: $primary;
    background: transparent;
    border: none;
    padding: 0;
    margin-bottom: 14px;
    cursor: pointer;

    &:hover {
      text-decoration: underline;
    }
  }

  &__card-foot {
    padding-top: 12px;
    border-top: 1px solid $border-subtle;
  }

  &__card-foot-source {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    font-size: 0.6875rem;
    color: $text-tertiary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;

    svg {
      flex-shrink: 0;
    }
  }

  /* ---------- loading — plain, no border, just centers the spinner ---------- */
  &__loading {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 64px 20px;
  }

  &__empty {
    flex-shrink: 0;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 10px;
    padding: 52px 20px;
    border: 1px dashed $border-strong;
    border-radius: 14px;
    color: $text-tertiary;
    font-size: 0.84375rem;

    svg {
      color: $text-tertiary;
    }

    &--error {
      border-style: solid;
      border-color: $danger-subtle;
      background: $danger-subtle;
      color: $danger;

      svg {
        color: $danger;
      }
    }
  }

  &__spin {
    animation: datasets-page-spin 0.9s linear infinite;
  }

  /* ---------- tasks modal ---------- */
  &__overlay {
    position: fixed;
    inset: 0;
    z-index: 200;
    background: rgba(0, 0, 0, 0.45);
    backdrop-filter: blur(2px);
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 24px;
  }

  &__modal {
    width: 100%;
    max-width: 32rem;
    max-height: min(80vh, 40rem);
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 14px;
    box-shadow: $shadow-lg;
    display: flex;
    flex-direction: column;
    overflow: hidden;
  }

  &__modal-head {
    flex-shrink: 0;
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
    padding: 20px 22px 16px;
    border-bottom: 1px solid $border-subtle;

    .datasets-page__tag {
      margin-bottom: 8px;
    }
  }

  &__modal-title {
    font-size: 1.0625rem;
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $text-primary;
  }

  &__modal-sub {
    margin-top: 3px;
    font-size: 0.75rem;
    color: $text-tertiary;
  }

  &__modal-close {
    flex-shrink: 0;
    width: 30px;
    height: 30px;
    border-radius: 8px;
    border: 1px solid $border-default;
    background: $bg-main;
    color: $text-tertiary;
    display: grid;
    place-items: center;
    cursor: pointer;
    transition: border-color 0.14s ease, color 0.14s ease;

    &:hover {
      border-color: $text-primary;
      color: $text-primary;
    }
  }

  &__modal-body {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding: 18px 22px 22px;
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  /* ---------- responsive ---------- */
  @media (max-width: 1500px) {
    &__grid {
      grid-template-columns: repeat(2, 1fr);
    }
  }

  @media (max-width: 1000px) {
    &__grid {
      grid-template-columns: 1fr;
    }
  }

  @media (max-width: 640px) {
    &__card-stats {
      flex-wrap: wrap;
      gap: 14px;
    }
  }

  /* ---------- ultra-wide: nudge key text sizes up a touch ---------- */
  @media (min-width: 1800px) {
    &__title {
      font-size: 23px;
    }

    &__subtitle {
      font-size: 0.90625rem;
    }

    &__card-name {
      font-size: 1.03125rem;
    }

    &__card-desc {
      font-size: 0.84375rem;
    }

    &__card-stat-value {
      font-size: 1.03125rem;
    }

    &__card-stat-value--sm {
      font-size: 0.84375rem;
    }

    &__card-task {
      font-size: 0.8125rem;
    }

    &__cap-pill {
      font-size: 0.78125rem;
    }

    &__card-stat-label {
      font-size: 0.6875rem;
    }

    &__card-section-label {
      font-size: 0.71875rem;
    }

    &__card-view-all {
      font-size: 0.78125rem;
    }
  }
}

@keyframes datasets-page-spin {
  to {
    transform: rotate(360deg);
  }
}
