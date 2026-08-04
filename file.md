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














//Datasets.scss
@use '../../../styles/variables' as *;

.datasets-page {
  display: flex;
  flex-direction: column;
  gap: 16px;
  // Caps the page at a comfortable working width and centers it, so on very
  // wide viewports (1800px+) the sidebar/detail columns don't stretch into an
  // unusably wide layout — the extra space becomes gutters instead.
  max-width: 1680px;
  margin-left: auto;
  margin-right: auto;
  // Same flex chain as History/Models/Providers: main-layout__body / workspace-layout /
  // workspace-layout__content already resolve to the exact viewport height,
  // so height: 100% fills it precisely. workspace-layout__content also carries
  // a 3rem bottom padding, which would leave a gap below this page — pull most
  // of that back in, keeping a small 0.75rem breathing-room strip at the bottom.
  height: calc(100% + 3rem - 0.75rem);
  margin-bottom: calc(-3rem + 0.75rem);
  min-height: 0;

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
      color: #fff;

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

  /* ---------- segmented category bar ---------- */
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

  /* ---------- master-detail layout ---------- */
  &__layout {
    flex: 1;
    display: grid;
    grid-template-columns: 340px 1fr;
    gap: 16px;
    min-height: 0;
    overflow: hidden;
  }

  /* ---------- sidebar list ---------- */
  &__sidebar {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 16px;
    box-shadow: $shadow-xs;
    height: 100%;
    min-height: 0;
    display: flex;
    flex-direction: column;
    overflow: hidden;
  }

  &__sidebar-list {
    flex: 1;
    display: flex;
    flex-direction: column;
    min-height: 0;
    overflow-y: auto;
  }

  &__item {
    display: flex;
    flex-direction: column;
    gap: 6px;
    padding: 14px 16px;
    text-align: left;
    border: none;
    border-bottom: 1px solid $border-subtle;
    border-left: 3px solid transparent;
    background: $bg-main;
    width: 100%;
    cursor: pointer;
    transition: background 0.12s ease, border-color 0.12s ease;

    &:last-child {
      border-bottom: none;
    }

    &:hover {
      background: $bg-subtle;
    }

    &--active {
      background: $primary-light;
      border-left-color: $primary;

      .datasets-page__item-name {
        color: $primary;
      }
    }
  }

  &__item-top {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__item-name {
    font-size: 0.84375rem;
    font-weight: 600;
    color: $text-primary;
    line-height: 1.35;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__item-featured {
    flex-shrink: 0;
    display: inline-flex;
    color: $warning;
  }

  &__item-meta {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
    font-size: 0.75rem;
    color: $text-tertiary;
  }

  &__item-questions {
    flex-shrink: 0;
  }

  /* ---------- tags (shared by sidebar + detail) ---------- */
  &__tag {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 4px;
    font-size: 0.625rem;
    font-weight: 600;
    border-radius: 999px;
    padding: 2px 8px;

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

    &--featured {
      color: $warning;
      background: $warning-subtle;
    }
  }

  /* ---------- detail panel ---------- */
  &__detail {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 16px;
    box-shadow: $shadow-xs;
    padding: 26px 28px;
    height: 100%;
    min-height: 0;
    overflow-y: auto;
  }

  &__detail-head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
    padding-bottom: 18px;
    margin-bottom: 16px;
    border-bottom: 1px solid $border-subtle;
    position: sticky;
    top: -26px;
    padding-top: 26px;
    margin-top: -26px;
    background: $bg-main;
    z-index: 1;
  }

  &__detail-head-left {
    display: flex;
    flex-direction: column;
    gap: 8px;
    min-width: 0;
  }

  &__detail-tags {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;

    .datasets-page__tag {
      font-size: 0.6875rem;
      padding: 3px 10px;
    }
  }

  &__detail-name {
    font-size: 1.25rem;
    font-weight: 800;
    letter-spacing: -0.01em;
    line-height: 1.3;
    color: $text-primary;
  }

  &__detail-desc {
    font-size: 0.84375rem;
    line-height: 1.6;
    color: $text-secondary;
    margin-bottom: 24px;
  }

  /* ---------- section titles ---------- */
  &__section-title {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 0.8125rem;
    font-weight: 700;
    color: $text-primary;
    margin: 0 0 12px;

    svg {
      color: $primary;
      flex-shrink: 0;
    }

    &:not(:first-of-type) {
      margin-top: 24px;
    }
  }

  /* ---------- stat cards ---------- */
  &__stat-row {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 12px;
    margin-bottom: 24px;
  }

  &__stat-card {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-left: 3px solid $primary;
    border-radius: 10px;
    padding: 14px 16px;
    display: flex;
    align-items: center;
    gap: 12px;
    min-width: 0;
  }

  &__stat-icon {
    flex-shrink: 0;
    width: 34px;
    height: 34px;
    border-radius: 9px;
    background: $primary-light;
    color: $primary;
    display: grid;
    place-items: center;
  }

  &__stat-body {
    display: flex;
    flex-direction: column;
    gap: 2px;
    min-width: 0;
  }

  &__stat-card-label {
    font-size: 0.6875rem;
    font-weight: 600;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $text-tertiary;
  }

  &__stat-card-value {
    font-size: 1.125rem;
    font-weight: 800;
    color: $text-primary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;

    &--sm {
      font-size: 0.9375rem;
      font-weight: 700;
    }
  }

  /* ---------- meta list ---------- */
  &__meta-list {
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  &__meta-item {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 0.8125rem;
    color: $text-secondary;

    svg {
      flex-shrink: 0;
      color: $text-tertiary;
    }
  }

  /* ---------- empty state ---------- */
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

  /* ---------- task list — uniform grid cards ---------- */
  &__task-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
    gap: 12px;
    margin-bottom: 24px;
  }

  &__task-card {
    height: 128px;
    background: $bg-subtle;
    border: 1px solid $border-subtle;
    border-radius: 12px;
    padding: 14px 16px;
    display: flex;
    flex-direction: column;
    gap: 8px;
    transition: transform 0.13s ease, border-color 0.13s ease, box-shadow 0.13s ease, background 0.13s ease;

    &:hover {
      transform: translateY(-2px);
      border-color: $border-strong;
      box-shadow: $shadow-md;
      background: $bg-main;
    }
  }

  &__task-card-top {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__task-card-name {
    font-size: 0.8125rem;
    font-weight: 700;
    color: $text-primary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__task-card-idx {
    flex-shrink: 0;
    font-size: 0.65625rem;
    font-weight: 800;
    color: $text-tertiary;
  }

  &__task-card-value {
    font-size: 0.78125rem;
    color: $text-secondary;
    line-height: 1.5;
    display: -webkit-box;
    -webkit-line-clamp: 4;
    -webkit-box-orient: vertical;
    overflow: hidden;
  }

  /* ---------- responsive ---------- */
  @media (max-width: 900px) {
    &__layout {
      grid-template-columns: 1fr;
      grid-template-rows: 16rem 1fr;
      overflow: visible;
    }

    &__sidebar,
    &__detail {
      height: auto;
      max-height: 22rem;
    }

    &__stat-row {
      grid-template-columns: repeat(2, 1fr);
    }
  }

  @media (max-width: 520px) {
    &__detail {
      padding: 18px 16px;
    }

    &__detail-head {
      flex-direction: column;
    }

    &__stat-row {
      grid-template-columns: 1fr;
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

    &__item-name {
      font-size: 0.90625rem;
    }

    &__detail-name {
      font-size: 1.375rem;
    }

    &__detail-desc {
      font-size: 0.90625rem;
    }

    &__stat-card-value {
      font-size: 1.375rem;
    }

    &__stat-card-value--sm {
      font-size: 1.0625rem;
    }

    &__task-card-name {
      font-size: 0.875rem;
    }

    &__task-card-value {
      font-size: 0.84375rem;
    }
  }
}

@keyframes datasets-page-spin {
  to {
    transform: rotate(360deg);
  }
}


















//Models.scss
@use '../../../styles/variables' as *;

.models-page {
  display: flex;
  flex-direction: column;
  gap: 18px;
  // Caps the page at a comfortable working width and centers it, so on very
  // wide viewports (1800px+) the sidebar/detail columns don't stretch into an
  // unusably wide layout — the extra space becomes gutters instead.
  max-width: 1680px;
  margin-left: auto;
  margin-right: auto;
  // Same flex chain as History: main-layout__body / workspace-layout /
  // workspace-layout__content already resolve to the exact viewport height,
  // so height: 100% fills it precisely. workspace-layout__content also carries
  // a 3rem bottom padding, which would leave a gap below this page — pull most
  // of that back in, keeping a small 0.75rem breathing-room strip at the bottom.
  height: calc(100% + 3rem - 0.75rem);
  margin-bottom: calc(-3rem + 0.75rem);
  min-height: 0;

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
    flex-shrink: 0;
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
    margin-bottom: 3px;
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
    display: inline-flex;
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

  /* ---------- master-detail layout ---------- */
  &__layout {
    flex: 1;
    display: grid;
    grid-template-columns: 340px 1fr;
    gap: 16px;
    min-height: 0;
    overflow: hidden;
  }

  /* ---------- sidebar list ---------- */
  &__sidebar {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 16px;
    box-shadow: $shadow-xs;
    height: 100%;
    min-height: 0;
    display: flex;
    flex-direction: column;
    overflow: hidden;
  }

  &__sidebar-list {
    flex: 1;
    display: flex;
    flex-direction: column;
    min-height: 0;
    overflow-y: auto;
  }

  &__item {
    display: flex;
    flex-direction: column;
    gap: 6px;
    padding: 14px 16px;
    text-align: left;
    border: none;
    border-bottom: 1px solid $border-subtle;
    border-left: 3px solid transparent;
    background: $bg-main;
    width: 100%;
    cursor: pointer;
    transition: background 0.12s ease, border-color 0.12s ease;

    &:last-child {
      border-bottom: none;
    }

    &:hover {
      background: $bg-subtle;
    }

    &--active {
      background: $primary-light;
      border-left-color: $primary;

      .models-page__item-name {
        color: $primary;
      }
    }
  }

  &__item-top {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__item-name {
    font-size: 0.84375rem;
    font-weight: 600;
    color: $text-primary;
    line-height: 1.35;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__item-provider {
    flex-shrink: 0;
    font-size: 0.65625rem;
    font-weight: 600;
    color: $text-tertiary;
  }

  &__item-meta {
    display: flex;
    align-items: center;
    justify-content: space-between;
    font-size: 0.75rem;
    color: $text-tertiary;
  }

  &__item-score {
    font-weight: 700;
    color: $success;
  }

  /* ---------- detail panel ---------- */
  &__detail {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 16px;
    box-shadow: $shadow-xs;
    padding: 26px 28px;
    height: 100%;
    min-height: 0;
    overflow-y: auto;
  }

  &__detail-head {
    padding-bottom: 18px;
    margin-bottom: 16px;
    border-bottom: 1px solid $border-subtle;
    position: sticky;
    top: -26px;
    padding-top: 26px;
    margin-top: -26px;
    background: $bg-main;
    z-index: 1;
  }

  &__detail-head-left {
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  &__detail-name {
    font-size: 1.25rem;
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $text-primary;
  }

  &__detail-version {
    font-family: $font-mono;
    font-size: 0.75rem;
    color: $text-tertiary;
  }

  &__detail-desc {
    font-size: 0.84375rem;
    line-height: 1.6;
    color: $text-secondary;
    margin-bottom: 18px;
  }

  &__provider-badge {
    width: fit-content;
    font-size: 0.6875rem;
    font-weight: 600;
    color: $primary;
    background: $primary-light;
    border-radius: 999px;
    padding: 3px 10px;
  }

  /* ---------- capability pills ---------- */
  &__caps {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    margin-bottom: 24px;
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

  /* ---------- stat cards ---------- */
  &__section-title {
    font-size: 0.8125rem;
    font-weight: 700;
    color: $text-primary;
    margin-bottom: 12px;
  }

  &__stat-row {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 12px;
    margin-bottom: 24px;
  }

  &__stat-card {
    background: $bg-subtle;
    border-radius: 12px;
    padding: 14px 16px;
    display: flex;
    flex-direction: column;
    gap: 4px;
    min-width: 0;
  }

  &__stat-card-label {
    font-size: 0.6875rem;
    font-weight: 600;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $text-tertiary;
  }

  &__stat-card-value {
    font-size: 1.25rem;
    font-weight: 800;
    color: $text-primary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;

    &--sm {
      font-size: 1rem;
      font-weight: 700;
    }

    &--accent {
      color: $success;
    }
  }

  /* ---------- agent score bar ---------- */
  &__agent-score {
    display: flex;
    align-items: center;
    gap: 12px;
  }

  &__agent-score-bar-track {
    flex: 1;
    height: 8px;
    border-radius: 999px;
    background: $bg-inset;
    overflow: hidden;
  }

  &__agent-score-bar-fill {
    height: 100%;
    border-radius: 999px;
    background: $primary;
  }

  &__agent-score-value {
    font-size: 0.9375rem;
    font-weight: 700;
    color: $success;
    flex-shrink: 0;
    width: 48px;
    text-align: right;
  }

  /* ---------- empty state ---------- */
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
  }

  /* ---------- responsive ---------- */
  @media (max-width: 900px) {
    &__layout {
      grid-template-columns: 1fr;
      grid-template-rows: 16rem 1fr;
      overflow: visible;
    }

    &__sidebar,
    &__detail {
      height: auto;
      max-height: 22rem;
    }

    &__stat-row {
      grid-template-columns: repeat(2, 1fr);
    }
  }

  @media (max-width: 520px) {
    &__detail {
      padding: 18px 16px;
    }

    &__stat-row {
      grid-template-columns: 1fr;
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

    &__item-name {
      font-size: 0.90625rem;
    }

    &__detail-name {
      font-size: 1.375rem;
    }

    &__detail-desc {
      font-size: 0.90625rem;
    }

    &__cap-pill {
      font-size: 0.78125rem;
    }

    &__stat-card-value {
      font-size: 1.375rem;
    }

    &__stat-card-value--sm {
      font-size: 1.0625rem;
    }

    &__agent-score-value {
      font-size: 1.03125rem;
    }
  }
}


















//Providers.scss
@use '../../../styles/variables' as *;

.providers-page {
  display: flex;
  flex-direction: column;
  gap: 16px;
  // Caps the page at a comfortable working width and centers it, so on very
  // wide viewports (1800px+) the sidebar/detail columns don't stretch into an
  // unusably wide layout — the extra space becomes gutters instead.
  max-width: 1680px;
  margin-left: auto;
  margin-right: auto;
  // Same flex chain as History/Models: main-layout__body / workspace-layout /
  // workspace-layout__content already resolve to the exact viewport height,
  // so height: 100% fills it precisely. workspace-layout__content also carries
  // a 3rem bottom padding, which would leave a gap below this page — pull most
  // of that back in, keeping a small 0.75rem breathing-room strip at the bottom.
  height: calc(100% + 3rem - 0.75rem);
  margin-bottom: calc(-3rem + 0.75rem);
  min-height: 0;

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
    flex-shrink: 0;
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
    margin-bottom: 3px;
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

  /* ---------- search ---------- */
  &__search {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 9px;
    width: 300px;
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

  /* ---------- master-detail layout ---------- */
  &__layout {
    flex: 1;
    display: grid;
    grid-template-columns: 340px 1fr;
    gap: 16px;
    min-height: 0;
    overflow: hidden;
  }

  /* ---------- sidebar list ---------- */
  &__sidebar {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 16px;
    box-shadow: $shadow-xs;
    height: 100%;
    min-height: 0;
    display: flex;
    flex-direction: column;
    overflow: hidden;
  }

  &__sidebar-list {
    flex: 1;
    display: flex;
    flex-direction: column;
    min-height: 0;
    overflow-y: auto;
  }

  &__item {
    display: flex;
    align-items: flex-start;
    gap: 12px;
    padding: 14px 16px;
    text-align: left;
    border: none;
    border-bottom: 1px solid $border-subtle;
    border-left: 3px solid transparent;
    background: $bg-main;
    width: 100%;
    cursor: pointer;
    transition: background 0.12s ease, border-color 0.12s ease;

    &:last-child {
      border-bottom: none;
    }

    &:hover {
      background: $bg-subtle;
    }

    &--active {
      background: $primary-light;
      border-left-color: $primary;

      .providers-page__item-name {
        color: $primary;
      }
    }
  }

  &__item-body {
    flex: 1;
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 6px;
  }

  /* ---------- avatar ---------- */
  &__avatar {
    flex-shrink: 0;
    width: 34px;
    height: 34px;
    border-radius: 10px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    font-size: 0.75rem;
    font-weight: 700;
    letter-spacing: -0.01em;

    &--lg {
      width: 46px;
      height: 46px;
      border-radius: 12px;
      font-size: 1rem;
    }

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

  &__item-top {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__item-name {
    font-size: 0.84375rem;
    font-weight: 600;
    color: $text-primary;
    line-height: 1.35;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__item-meta {
    display: flex;
    align-items: center;
    justify-content: space-between;
    font-size: 0.75rem;
    color: $text-tertiary;
  }

  /* ---------- status tag (shared by sidebar + detail) ---------- */
  &__tag {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 4px;
    font-size: 0.6875rem;
    font-weight: 600;
    border-radius: 999px;
    padding: 3px 10px;

    &--jade {
      color: $success;
      background: $success-subtle;
    }

    &--gray {
      color: $text-tertiary;
      background: $bg-inset;
    }
  }

  /* ---------- detail panel ---------- */
  &__detail {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 16px;
    box-shadow: $shadow-xs;
    padding: 26px 28px;
    height: 100%;
    min-height: 0;
    overflow-y: auto;
  }

  &__detail-head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
    padding-bottom: 18px;
    margin-bottom: 16px;
    border-bottom: 1px solid $border-subtle;
    position: sticky;
    top: -26px;
    padding-top: 26px;
    margin-top: -26px;
    background: $bg-main;
    z-index: 1;
  }

  &__detail-head-left {
    display: flex;
    align-items: center;
    gap: 14px;
  }

  &__detail-head-text {
    display: flex;
    flex-direction: column;
    gap: 8px;

    .providers-page__tag {
      width: fit-content;
    }
  }

  &__detail-name {
    font-size: 1.25rem;
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $text-primary;
  }

  &__detail-actions {
    display: flex;
    gap: 8px;
    flex-shrink: 0;
    flex-wrap: wrap;
  }

  &__detail-desc {
    font-size: 0.84375rem;
    line-height: 1.6;
    color: $text-secondary;
    margin-bottom: 24px;
  }

  /* ---------- stat cards ---------- */
  &__section-title {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 0.8125rem;
    font-weight: 700;
    color: $text-primary;
    margin-bottom: 12px;

    &:not(:first-of-type) {
      margin-top: 24px;
    }

    svg {
      opacity: 0.8;
    }
  }

  /* ---------- capability pills ---------- */
  &__caps {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
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

  /* ---------- model list ---------- */
  &__model-list {
    display: flex;
    flex-direction: column;
    gap: 1px;
    border: 1px solid $border-subtle;
    border-radius: 12px;
    overflow: hidden;
  }

  &__model-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
    padding: 12px 14px;
    background: $bg-main;
    border-bottom: 1px solid $border-subtle;

    &:last-child {
      border-bottom: none;
    }

    &:hover {
      background: $bg-subtle;
    }
  }

  &__model-row-main {
    display: flex;
    align-items: baseline;
    gap: 8px;
    min-width: 0;
  }

  &__model-name {
    font-size: 0.8125rem;
    font-weight: 600;
    color: $text-primary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__model-version {
    flex-shrink: 0;
    font-family: $font-mono;
    font-size: 0.6875rem;
    color: $text-tertiary;
  }

  &__model-row-stats {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 16px;
  }

  &__model-pricing {
    font-size: 0.75rem;
    color: $text-secondary;
  }

  &__model-accuracy {
    font-size: 0.8125rem;
    font-weight: 700;
    color: $success;
    width: 42px;
    text-align: right;
  }

  &__stat-row {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 12px;
    margin-bottom: 8px;
  }

  &__stat-card {
    background: $bg-subtle;
    border-radius: 12px;
    padding: 14px 16px;
    display: flex;
    flex-direction: column;
    gap: 4px;
    min-width: 0;
  }

  &__stat-card-label {
    display: flex;
    align-items: center;
    gap: 5px;
    font-size: 0.6875rem;
    font-weight: 600;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $text-tertiary;

    svg {
      flex-shrink: 0;
      opacity: 0.8;
    }
  }

  &__stat-card-value {
    font-size: 1.25rem;
    font-weight: 800;
    color: $text-primary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;

    &--sm {
      font-size: 1rem;
      font-weight: 700;
    }

    &--accent {
      color: $success;
    }
  }

  /* ---------- inline connect form ---------- */
  &__connect-form {
    margin-top: 8px;
    display: flex;
    flex-direction: column;
    gap: 8px;
    max-width: 22rem;
  }

  &__field-label {
    font-size: 0.71875rem;
    font-weight: 600;
    color: $text-secondary;
  }

  &__input-wrap {
    display: flex;
    align-items: center;
    gap: 8px;
    border: 1px solid $border-default;
    border-radius: 8px;
    padding: 0 11px;
    background: $bg-main;
    color: $text-tertiary;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &:focus-within {
      border-color: $primary;
      box-shadow: 0 0 0 3px $primary-light;
    }
  }

  &__input {
    flex: 1;
    width: 100%;
    border: none;
    outline: none;
    padding: 8px 0;
    font-size: 0.8125rem;
    font-family: $font-body;
    color: $text-primary;
    background: transparent;

    &::placeholder {
      color: #a8b1bb;
    }
  }

  &__form-actions {
    display: flex;
    justify-content: flex-end;
    gap: 8px;
    margin-top: 2px;
  }

  /* ---------- buttons ---------- */
  &__btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 6px;
    font-family: $font-body;
    font-size: 0.8125rem;
    font-weight: 600;
    padding: 8px 14px;
    border-radius: 8px;
    border: 1px solid transparent;
    cursor: pointer;
    transition: background 0.14s ease, border-color 0.14s ease, color 0.14s ease;

    &--outline {
      background: $bg-main;
      border-color: $border-default;
      color: $text-primary;

      &:hover {
        border-color: $text-primary;
      }
    }

    &--danger-outline {
      background: $bg-main;
      border-color: $border-default;
      color: $text-tertiary;

      &:hover {
        border-color: $danger;
        color: $danger;
        background: $danger-subtle;
      }
    }

    &--primary {
      background: $primary;
      border-color: $primary;
      color: #fff;

      &:hover {
        background: $primary-hover;
        border-color: $primary-hover;
      }
    }
  }

  /* ---------- empty state ---------- */
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
  }

  /* ---------- responsive ---------- */
  @media (max-width: 900px) {
    &__layout {
      grid-template-columns: 1fr;
      grid-template-rows: 16rem 1fr;
      overflow: visible;
    }

    &__sidebar,
    &__detail {
      height: auto;
      max-height: 22rem;
    }

    &__stat-row {
      grid-template-columns: repeat(2, 1fr);
    }
  }

  @media (max-width: 520px) {
    &__detail {
      padding: 18px 16px;
    }

    &__detail-head {
      flex-direction: column;
    }

    &__stat-row {
      grid-template-columns: 1fr;
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

    &__item-name {
      font-size: 0.90625rem;
    }

    &__detail-name {
      font-size: 1.375rem;
    }

    &__detail-desc {
      font-size: 0.90625rem;
    }

    &__stat-card-value {
      font-size: 1.375rem;
    }

    &__stat-card-value--sm {
      font-size: 1.0625rem;
    }

    &__cap-pill {
      font-size: 0.78125rem;
    }

    &__model-name {
      font-size: 0.875rem;
    }
  }
}























//History.scss
@use '../../../styles/variables' as *;

.history {
  display: flex;
  flex-direction: column;
  gap: 18px;
  // Caps the page at a comfortable reading/working width and centers it, so
  // on very wide viewports (1800px+) the sidebar/detail columns don't stretch
  // into an unusably wide layout — instead the extra space becomes gutters.
  max-width: 1680px;
  margin-left: auto;
  margin-right: auto;
  // .main-layout__body / .workspace-layout / .workspace-layout__content form an
  // unbroken flex chain sized to the viewport (header/footer are fixed and offset
  // via margin), so height: 100% here resolves to the visible content area.
  // workspace-layout__content also has a 3rem bottom padding, which would
  // otherwise leave a large gap below .history — pull most of that back in,
  // keeping just a small 0.75rem breathing-room strip at the true bottom edge.
  height: calc(100% + 3rem - 0.75rem);
  margin-bottom: calc(-3rem + 0.75rem);
  min-height: 0;

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
    flex-shrink: 0;
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
    margin-bottom: 3px;
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

  /* ---------- generic buttons ---------- */
  &__btn {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    font-family: $font-body;
    font-size: 0.8125rem;
    font-weight: 600;
    padding: 9px 14px;
    border-radius: 8px;
    border: 1px solid $border-default;
    background: $bg-main;
    color: $text-secondary;
    cursor: pointer;
    white-space: nowrap;
    transition: background 0.14s ease, border-color 0.14s ease, color 0.14s ease;

    &:hover {
      border-color: $text-primary;
      color: $text-primary;
    }

    &--primary {
      background: $primary;
      border-color: $primary;
      color: #fff;

      &:hover {
        background: $primary-hover;
        border-color: $primary-hover;
        color: #fff;
      }
    }

    &--danger:hover {
      border-color: $danger;
      color: $danger;
      background: $danger-subtle;
    }

    &--push {
      margin-left: auto;
    }
  }

  /* ---------- filters ---------- */
  &__filters {
    flex-shrink: 0;
    display: flex;
    flex-wrap: wrap;
    gap: 10px;
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

  /* ---------- custom dropdown (used by <Select />) ---------- */
  &-select {
    position: relative;
    flex-shrink: 0;

    &__trigger {
      width: 100%;
      display: flex;
      align-items: center;
      justify-content: space-between;
      gap: 8px;
      border: 1px solid $border-default;
      border-radius: 10px;
      padding: 9px 12px;
      background: $bg-main;
      font-size: 0.8125rem;
      font-weight: 500;
      font-family: $font-body;
      color: $text-primary;
      cursor: pointer;
      transition: border-color 0.14s ease, box-shadow 0.14s ease;

      &:hover {
        border-color: $border-strong;
      }

      &--open {
        border-color: $primary;
        box-shadow: 0 0 0 3px $primary-light;
      }
    }

    &__chevron {
      flex-shrink: 0;
      color: $text-tertiary;
      transition: transform 0.16s ease;
    }

    &__trigger--open &__chevron {
      transform: rotate(180deg);
    }

    &__menu {
      position: absolute;
      top: calc(100% + 6px);
      left: 0;
      right: 0;
      z-index: 20;
      background: $bg-main;
      border: 1px solid $border-subtle;
      border-radius: 10px;
      box-shadow: $shadow-lg;
      padding: 5px;
      display: flex;
      flex-direction: column;
      gap: 1px;
    }

    &__option {
      display: flex;
      align-items: center;
      justify-content: space-between;
      gap: 8px;
      width: 100%;
      text-align: left;
      padding: 8px 10px;
      border: none;
      border-radius: 7px;
      background: transparent;
      font-size: 0.8125rem;
      font-family: $font-body;
      color: $text-secondary;
      cursor: pointer;
      transition: background 0.12s ease, color 0.12s ease;

      &:hover {
        background: $bg-subtle;
        color: $text-primary;
      }

      &--active {
        color: $primary;
        font-weight: 600;

        svg {
          color: $primary;
        }
      }
    }
  }

  /* ---------- master-detail layout ---------- */
  &__layout {
    flex: 1;
    display: grid;
    grid-template-columns: 340px 1fr;
    gap: 16px;
    min-height: 0;
    overflow: hidden;
  }

  /* ---------- sidebar list ---------- */
  &__sidebar {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 16px;
    box-shadow: $shadow-xs;
    height: 100%;
    min-height: 0;
    display: flex;
    flex-direction: column;
    overflow: hidden;
  }

  &__sidebar-list {
    flex: 1;
    display: flex;
    flex-direction: column;
    min-height: 0;
    overflow-y: auto;
  }

  &__item {
    display: flex;
    flex-direction: column;
    gap: 6px;
    padding: 14px 16px;
    text-align: left;
    border: none;
    border-bottom: 1px solid $border-subtle;
    border-left: 3px solid transparent;
    background: $bg-main;
    width: 100%;
    cursor: pointer;
    transition: background 0.12s ease, border-color 0.12s ease;

    &:last-child {
      border-bottom: none;
    }

    &:hover {
      background: $bg-subtle;
    }

    &--active {
      background: $primary-light;
      border-left-color: $primary;

      .history__item-name {
        color: $primary;
      }
    }
  }

  &__item-top {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__item-name {
    font-size: 0.84375rem;
    font-weight: 600;
    color: $text-primary;
    line-height: 1.35;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__item-meta {
    display: flex;
    align-items: center;
    justify-content: space-between;
    font-size: 0.75rem;
    color: $text-tertiary;
  }

  &__item-score {
    font-weight: 700;
    color: $success;
  }

  /* ---------- type badge (shared by sidebar + detail) ---------- */
  &__type-badge {
    flex-shrink: 0;
    font-size: 0.625rem;
    font-weight: 700;
    border-radius: 999px;
    padding: 2px 8px;

    &--violet {
      color: $violet;
      background: $violet-light;
    }

    &--blue {
      color: $primary;
      background: $primary-light;
    }

    &--amber {
      color: $warning;
      background: $warning-subtle;
    }
  }

  /* ---------- detail panel ---------- */
  &__detail {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 16px;
    box-shadow: $shadow-xs;
    padding: 26px 28px;
    height: 100%;
    min-height: 0;
    overflow-y: auto;
  }

  &__detail-head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
    padding-bottom: 18px;
    margin-bottom: 20px;
    border-bottom: 1px solid $border-subtle;
    position: sticky;
    top: -26px;
    padding-top: 26px;
    margin-top: -26px;
    background: $bg-main;
    z-index: 1;
  }

  &__detail-head-left {
    display: flex;
    flex-direction: column;
    gap: 8px;

    .history__type-badge {
      width: fit-content;
    }
  }

  &__detail-name {
    font-size: 1.25rem;
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $text-primary;
  }

  &__detail-date {
    font-size: 0.8125rem;
    color: $text-tertiary;
  }

  &__detail-actions {
    display: flex;
    gap: 8px;
    flex-shrink: 0;
    flex-wrap: wrap;
  }

  /* ---------- stat cards ---------- */
  &__stat-row {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 12px;
    margin-bottom: 24px;
  }

  &__stat-card {
    background: $bg-subtle;
    border-radius: 12px;
    padding: 14px 16px;
    display: flex;
    flex-direction: column;
    gap: 4px;
    min-width: 0;
  }

  &__stat-card-label {
    font-size: 0.6875rem;
    font-weight: 600;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $text-tertiary;
  }

  &__stat-card-value {
    font-size: 1.25rem;
    font-weight: 800;
    color: $text-primary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;

    &--sm {
      font-size: 1rem;
      font-weight: 700;
    }

    &--accent {
      color: $success;
    }
  }

  /* ---------- results table ---------- */
  &__section-title {
    font-size: 0.8125rem;
    font-weight: 700;
    color: $text-primary;
    margin-bottom: 12px;
  }

  &__table-wrap {
    overflow-x: auto;
  }

  &__table {
    width: 100%;
    border-collapse: collapse;

    thead th {
      text-align: left;
      font-size: 0.6875rem;
      font-weight: 700;
      letter-spacing: 0.05em;
      text-transform: uppercase;
      color: $text-tertiary;
      padding: 10px 14px;
      background: $bg-subtle;
      white-space: nowrap;

      &:first-child {
        border-radius: 8px 0 0 8px;
      }

      &:last-child {
        border-radius: 0 8px 8px 0;
      }
    }

    tbody td {
      padding: 12px 14px;
      font-size: 0.84375rem;
      color: $text-secondary;
      border-bottom: 1px solid $border-subtle;
      white-space: nowrap;
    }

    tbody tr:last-child td {
      border-bottom: none;
    }
  }

  &__cell-strong {
    font-weight: 600;
    color: $text-primary;
  }

  &__rank-pill {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 22px;
    height: 22px;
    border-radius: 6px;
    background: $bg-inset;
    color: $text-tertiary;
    font-size: 0.6875rem;
    font-weight: 700;

    &--1 {
      background: $primary-light;
      color: $primary;
    }
  }

  &__score-cell {
    color: $success;
    font-weight: 700;
  }

  /* ---------- empty state ---------- */
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
  }

  /* ---------- responsive ---------- */
  @media (max-width: 900px) {
    &__layout {
      grid-template-columns: 1fr;
      grid-template-rows: 16rem 1fr;
      overflow: visible;
    }

    &__sidebar,
    &__detail {
      height: auto;
      max-height: 22rem;
    }

    &__stat-row {
      grid-template-columns: repeat(2, 1fr);
    }
  }

  @media (max-width: 520px) {
    &__detail {
      padding: 18px 16px;
    }

    &__detail-head {
      flex-direction: column;
    }

    &__stat-row {
      grid-template-columns: 1fr;
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

    &__item-name {
      font-size: 0.90625rem;
    }

    &__detail-name {
      font-size: 1.375rem;
    }

    &__stat-card-value {
      font-size: 1.375rem;
    }

    &__stat-card-value--sm {
      font-size: 1.0625rem;
    }

    &__table {
      font-size: 0.90625rem;
    }

    &__cell-strong {
      font-size: 0.90625rem;
    }
  }
}






















//Header.scss
@use '../../styles/variables' as *;

.app-header {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  z-index: $z-header;
  height: 60px;
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.75rem;
  padding: 0 1.125rem;
  background: $bg-header-glass;
  backdrop-filter: blur(0.75rem);
  border-bottom: 1px solid $border-subtle;

  &__brand {
    display: flex;
    align-items: center;
    gap: 0.625rem;
    text-decoration: none;
    color: $text-primary;
    flex-shrink: 0;
  }

  &__brand-mark {
    width: 32px;
    height: 32px;
    border-radius: 0.5rem;
    background: linear-gradient(155deg, $primary 0%, $primary-hover 100%);
    color: #fff;
    display: grid;
    place-items: center;
    box-shadow: $shadow-xs, inset 0 0 0 0.0625rem rgba(255, 255, 255, 0.14);
  }

  &__brand-name {
    font-family: $font-display;
    font-weight: 700;
    font-size: 1.0925rem;
    letter-spacing: -0.02em;
    white-space: nowrap;
  }

  &__right {
    display: flex;
    align-items: center;
    gap: 0.625rem;
    flex-shrink: 0;
  }

  /* ---------- theme toggle ---------- */
  &__theme-toggle {
    position: relative;
    flex-shrink: 0;
    width: 52px;
    height: 28px;
    border-radius: 999px;
    border: 1px solid $border-default;
    background: $bg-subtle;
    box-shadow: inset 0 0.0625rem 0.125rem rgba(0, 0, 0, 0.12), inset 0 -0.0625rem 0 rgba(255, 255, 255, 0.06);
    cursor: pointer;
    display: block;
    padding: 0;
    transition: border-color 0.14s ease;

    &:hover {
      border-color: $border-strong;
    }

    &:focus-visible {
      outline: none;
      border-color: $primary;
      box-shadow: inset 0 0.0625rem 0.125rem rgba(0, 0, 0, 0.12), inset 0 -0.0625rem 0 rgba(255, 255, 255, 0.06),
        0 0 0 0.1875rem $primary-subtle;
    }
  }

  &__theme-static {
    position: absolute;
    top: 50%;
    transform: translateY(-50%);
    color: $text-tertiary;
    opacity: 0.7;
    pointer-events: none;

    &--sun {
      left: 7px;
    }

    &--moon {
      right: 7px;
    }
  }

  &__theme-knob {
    position: absolute;
    top: 3px;
    left: 3px;
    width: 20px;
    height: 20px;
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    color: #fff;
    transition: transform 0.18s ease, background 0.18s ease, box-shadow 0.18s ease;

    &::before {
      content: '';
      position: absolute;
      inset: 0.0625rem 0.0625rem auto 0.0625rem;
      height: 45%;
      border-radius: 999px 999px 40% 40%;
      background: linear-gradient(to bottom, rgba(255, 255, 255, 0.55), rgba(255, 255, 255, 0));
      pointer-events: none;
    }

    &--light {
      background: linear-gradient(155deg, #ffcf6b 0%, $warning 100%);
      box-shadow: 0 0.125rem 0.375rem rgba(183, 121, 31, 0.45), 0 0 0 0.0625rem rgba(255, 255, 255, 0.25) inset;
    }

    &--dark {
      background: linear-gradient(155deg, $primary-hover 0%, $primary 100%);
      box-shadow: 0 0.125rem 0.5rem rgba(87, 108, 250, 0.55), 0 0 0 0.0625rem rgba(255, 255, 255, 0.18) inset;
      transform: translateX(24px);
    }
  }

  &__user {
    position: relative;
    flex-shrink: 0;
  }

  &__avatar {
    width: 34px;
    height: 34px;
    border-radius: 50%;
    border: 1px solid $border-default;
    background: $primary-light;
    color: $primary;
    font-family: $font-display;
    font-size: 0.8125rem;
    font-weight: 700;
    letter-spacing: 0.01em;
    display: grid;
    place-items: center;
    cursor: pointer;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &:hover {
      border-color: $primary;
    }

    &--open,
    &:focus-visible {
      outline: none;
      border-color: $primary;
      box-shadow: 0 0 0 0.1875rem $primary-subtle;
    }
  }

  &__dropdown {
    position: absolute;
    top: calc(100% + 0.625rem);
    right: 0;
    width: 232px;
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 0.75rem;
    box-shadow: $shadow-lg;
    padding: 0.625rem;
    z-index: $z-header;
  }

  &__drop-user {
    display: flex;
    align-items: center;
    gap: 0.625rem;
    padding: 0.375rem 0.375rem 0.625rem;
  }

  &__drop-avatar {
    width: 36px;
    height: 36px;
    flex-shrink: 0;
    border-radius: 50%;
    background: $primary-light;
    color: $primary;
    font-family: $font-display;
    font-size: 0.84375rem;
    font-weight: 700;
    display: grid;
    place-items: center;
  }

  &__drop-info {
    min-width: 0;
  }

  &__drop-name {
    font-size: 0.90625rem;
    font-weight: 600;
    color: $text-primary;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  &__drop-role {
    font-size: 0.78125rem;
    color: $text-tertiary;
    margin-top: 0.125rem;
  }

  &__drop-divider {
    height: 1px;
    background: $border-subtle;
    margin: 0.125rem 0.375rem 0.5rem;
  }

  &__drop-item {
    display: flex;
    align-items: center;
    gap: 0.5625rem;
    width: 100%;
    padding: 0.5rem 0.625rem;
    border: none;
    border-radius: 0.5rem;
    background: transparent;
    color: $danger;
    font-size: 0.875rem;
    font-weight: 500;
    cursor: pointer;
    transition: background 0.14s ease;

    &:hover {
      background: $danger-subtle;
    }
  }

  @media (max-width: 620px) {
    &__brand-name {
      font-size: 1.0125rem;
    }
  }

  /* ---------- ultra-wide: nudge key text sizes up a touch ---------- */
  @media (min-width: 1800px) {
    &__brand-name {
      font-size: 1.1875rem;
    }

    &__drop-name {
      font-size: 0.96875rem;
    }

    &__drop-role {
      font-size: 0.84375rem;
    }
  }
}


















//Footer.scss
@use '../../styles/variables' as *;

.app-footer {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  z-index: $z-footer;
  height: $footer-height;
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.75rem;
  padding: 0 1.125rem;
  background: $bg-main;
  border-top: 1px solid $border-subtle;

  &__version,
  &__copyright {
    font-size: 0.71875rem;
    color: $text-tertiary;
    white-space: nowrap;
  }

  /* ---------- ultra-wide: nudge text size up a touch ---------- */
  @media (min-width: 1800px) {
    &__version,
    &__copyright {
      font-size: 0.8125rem;
    }
  }
}
















//Spinner.scss
@use '../../styles/variables' as *;

.spinner {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;

  &__ring {
    border-radius: 50%;
    border: 3px solid $border-default;
    border-top-color: $primary;
    animation: spinner-rotate 0.8s linear infinite;
  }

  &__label {
    font-size: 0.8125rem;
    color: $text-tertiary;
    font-weight: 500;
  }

  /* ---------- ultra-wide: nudge label size up a touch ---------- */
  @media (min-width: 1800px) {
    &__label {
      font-size: 0.90625rem;
    }
  }
}

@keyframes spinner-rotate {
  to {
    transform: rotate(360deg);
  }
}




















//AuthSpinner.scss
@use '../../styles/variables' as *;

.auth-spinner {
  position: fixed;
  inset: 0;
  z-index: 1000;
  display: grid;
  place-items: center;
  background: $bg-page;

  &__card {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 1.125rem;
    padding: 2.5rem 3rem;
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 1.25rem;
    box-shadow: $shadow-lg;
  }

  &__mark {
    width: 52px;
    height: 52px;
    border-radius: 1rem;
    display: grid;
    place-items: center;
    color: $on-primary;
    background: linear-gradient(155deg, $primary 0%, $primary-hover 100%);
    box-shadow: $shadow-md, inset 0 0 0 0.0625rem rgba(255, 255, 255, 0.14);
    animation: auth-spinner-pulse 2.2s ease-in-out infinite;
  }

  &__ring {
    position: relative;
    width: 32px;
    height: 32px;
  }

  &__arc {
    position: absolute;
    inset: 0;
    border-radius: 50%;
    border: 3px solid $border-default;
    border-top-color: $primary;
    animation: auth-spinner-spin 0.8s linear infinite;
  }

  &__label {
    font-family: $font-display;
    font-size: 0.9375rem;
    font-weight: 700;
    color: $text-primary;
    letter-spacing: -0.01em;
  }

  &__sub {
    margin-top: -0.75rem;
    font-size: 0.8125rem;
    color: $text-tertiary;
  }

  /* ---------- ultra-wide: nudge text sizes up a touch ---------- */
  @media (min-width: 1800px) {
    &__label {
      font-size: 1.03125rem;
    }

    &__sub {
      font-size: 0.90625rem;
    }
  }
}

@keyframes auth-spinner-spin {
  to {
    transform: rotate(360deg);
  }
}

@keyframes auth-spinner-pulse {
  0%,
  100% {
    transform: scale(1);
    box-shadow: $shadow-md, inset 0 0 0 0.0625rem rgba(255, 255, 255, 0.14);
  }
  50% {
    transform: scale(1.05);
    box-shadow: $shadow-lg, inset 0 0 0 0.0625rem rgba(255, 255, 255, 0.18);
  }
}



























//ssologin.scss
@use '../../styles/variables' as *;

.sso-login {
  position: relative;
  min-height: 100vh;
  display: grid;
  place-items: center;
  padding: 2rem;
  background: $bg-page;
  overflow: hidden;

  &__grid {
    position: absolute;
    inset: 0;
    background-image: linear-gradient(to right, $border-subtle 0.0625rem, transparent 0.0625rem),
      linear-gradient(to bottom, $border-subtle 0.0625rem, transparent 0.0625rem);
    background-size: 3.375rem 3.375rem;
    mask-image: radial-gradient(60% 55% at 50% 40%, #000 22%, transparent 72%);
    -webkit-mask-image: radial-gradient(60% 55% at 50% 40%, #000 22%, transparent 72%);
    pointer-events: none;
  }

  &__card {
    position: relative;
    width: 100%;
    max-width: 380px;
    display: flex;
    flex-direction: column;
    align-items: center;
    text-align: center;
    gap: 0.25rem;
    padding: 2.75rem 2.25rem 2.25rem;
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 1.25rem;
    box-shadow: $shadow-lg;
  }

  &__mark {
    width: 52px;
    height: 52px;
    border-radius: 1rem;
    display: grid;
    place-items: center;
    color: $on-primary;
    background: linear-gradient(155deg, $primary 0%, $primary-hover 100%);
    box-shadow: $shadow-md, inset 0 0 0 0.0625rem rgba(255, 255, 255, 0.14);
    margin-bottom: 1.25rem;
  }

  &__title {
    font-family: $font-display;
    font-size: 1.375rem;
    font-weight: 800;
    letter-spacing: -0.02em;
    color: $text-primary;

    span {
      color: $primary;
    }
  }

  &__sub {
    margin-top: 0.25rem;
    font-size: 0.84375rem;
    color: $text-tertiary;
  }

  &__banner {
    width: 100%;
    display: flex;
    align-items: flex-start;
    gap: 0.5rem;
    margin-top: 1.5rem;
    padding: 0.6875rem 0.8125rem;
    border-radius: 0.625rem;
    font-size: 0.8125rem;
    line-height: 1.45;
    text-align: left;

    svg {
      flex-shrink: 0;
      margin-top: 0.0625rem;
    }

    &--error {
      background: $danger-subtle;
      color: $danger;
    }

    &--info {
      background: $success-subtle;
      color: $success;
    }
  }

  &__signin-btn {
    width: 100%;
    margin-top: 1.75rem;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 0.5625rem;
    font-family: $font-body;
    font-size: 0.90625rem;
    font-weight: 600;
    color: $on-primary;
    background: $primary;
    border: 1px solid $primary;
    border-radius: 0.625rem;
    padding: 0.75rem 1rem;
    cursor: pointer;
    transition: background 0.14s ease, border-color 0.14s ease;

    &:hover {
      background: $primary-hover;
      border-color: $primary-hover;
    }

    &:focus-visible {
      outline: none;
      box-shadow: 0 0 0 0.1875rem $primary-subtle;
    }
  }

  &__hint {
    margin-top: 1.25rem;
    font-size: 0.75rem;
    line-height: 1.5;
    color: $text-tertiary;
  }

  /* ---------- ultra-wide: nudge text sizes up a touch ---------- */
  @media (min-width: 1800px) {
    &__title {
      font-size: 1.5rem;
    }

    &__sub {
      font-size: 0.90625rem;
    }

    &__banner {
      font-size: 0.875rem;
    }

    &__signin-btn {
      font-size: 0.96875rem;
    }

    &__hint {
      font-size: 0.8125rem;
    }
  }
}
