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
