//_variables.scss
// ============================================================
// SemcoEval — design tokens (converted from theme.css / landing.css)
// Base: 1rem = 16px
// ============================================================

// ---------- Brand / primary ----------
$primary: #1428a0;
$primary-hover: #1d37c9;
$primary-light: #eef1fe;
$primary-subtle: #e2e7fc;

// ---------- Surfaces ----------
$bg-page: #f6f7f9;
$bg-subtle: #f3f5f8;
$bg-inset: #edf0f4;
$bg-main: #ffffff;

// ---------- Borders ----------
$border-default: #dce0e7;
$border-subtle: #e9ecf1;
$border-strong: #c7cdd8;

// ---------- Text ----------
$text-primary: #0e1526;
$text-secondary: #46506b;
$text-tertiary: #7a8399;

// ---------- Status ----------
$success: #0f7a5a;
$success-subtle: #e4f4ee;
$warning: #b7791f;
$warning-subtle: #fdf3e0;
$danger: #c0303b;
$danger-subtle: #fcebec;

// ---------- Shadows ----------
$shadow-xs: 0 0.0625rem 0.125rem rgba(14, 21, 38, 0.04);
$shadow-sm: 0 0.0625rem 0.125rem rgba(14, 21, 38, 0.05);
$shadow-md: 0 0.125rem 0.25rem rgba(14, 21, 38, 0.05), 0 0.5rem 1.25rem -0.75rem rgba(14, 21, 38, 0.16);
$shadow-lg: 0 0.25rem 0.5rem rgba(14, 21, 38, 0.05), 0 1.125rem 2.75rem -1.375rem rgba(14, 21, 38, 0.24);
$shadow-xl: 0 1.75rem 4.375rem -1.875rem rgba(14, 21, 38, 0.34);

// ---------- Radius ----------
$radius-sm: 0.375rem;
$radius-md: 0.5rem;
$radius-lg: 0.75rem;
$radius-xl: 1rem;

// ---------- Typography ----------
$font-display: 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
$font-body: 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
$font-mono: 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;

// ---------- Layout ----------
$header-height: 60px;
$footer-height: 30px;
$sidebar-width: 240px;

// ---------- Z-index ----------
$z-header: 100;
$z-footer: 100;
$z-sidebar: 90;
















//global.scss
@use './variables' as *;

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
  font-size: 1rem;
  line-height: 1.55;
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
