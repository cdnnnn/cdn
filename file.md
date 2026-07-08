//_tokens.scss

// Every color token below resolves through a CSS custom property so theme
// switching is automatic: set `[data-theme='dark']` (or leave it off / set
// to 'light') on any ancestor element — typically <html> or <body> — and
// every component using these tokens repaints without any component-level
// changes. The actual custom property values live in global.scss under
// `:root` (light, default) and `[data-theme='dark']` (dark overrides).
$border: var(--db-analytics-border);
$border-strong: var(--db-analytics-border-strong);
$border-stronger: var(--db-analytics-border-stronger);
$text-primary: var(--db-analytics-text-primary);
$text-secondary: var(--db-analytics-text-secondary);
$text-muted: var(--db-analytics-text-muted);
$surface-0: var(--db-analytics-surface-0);
$surface-1: var(--db-analytics-surface-1);
$surface-2: var(--db-analytics-surface-2);
$accent: var(--db-analytics-accent);
$accent-bg: var(--db-analytics-accent-bg);
$success: var(--db-analytics-success);
$warning: var(--db-analytics-warning);
$danger: var(--db-analytics-danger);
$radius: 8px;
$radius-lg: 12px;
$gradient-primary: var(--db-analytics-gradient-primary);












//global.scss


// NOTE: this file is only relevant for running DBAnalytics as a standalone app.
// When merging into an existing project that already sets box-sizing,
// font-family, and root height at the application level, skip importing
// this file — DBAnalytics.module.scss no longer depends on it.
//
// Theme variables: every color used across the DBAnalytics components reads
// from these CSS custom properties (see src/styles/_tokens.scss). Toggling
// dark mode is just a matter of setting `data-theme="dark"` on <html> or
// <body> — no component code needs to know which theme is active.
//
//   document.documentElement.setAttribute('data-theme', 'dark');
//   document.documentElement.setAttribute('data-theme', 'light'); // or remove it

:root {
  --db-analytics-border: rgba(11, 11, 11, 0.1);
  --db-analytics-border-strong: rgba(11, 11, 11, 0.16);
  --db-analytics-border-stronger: rgba(11, 11, 11, 0.24);
  --db-analytics-text-primary: #0b0b0b;
  --db-analytics-text-secondary: #52514e;
  --db-analytics-text-muted: #898781;
  --db-analytics-surface-0: #f5f4f0;
  --db-analytics-surface-1: #fcfcfb;
  --db-analytics-surface-2: #ffffff;
  --db-analytics-accent: #2a78d6;
  --db-analytics-accent-bg: #e6f1fb;
  --db-analytics-success: #0ca30c;
  --db-analytics-warning: #eda100;
  --db-analytics-danger: #e34948;
  --db-analytics-gradient-primary: linear-gradient(135deg, #7c3aed 0%, #4f46e5 50%, #2563eb 100%);
}

[data-theme='dark'] {
  --db-analytics-border: rgba(255, 255, 255, 0.1);
  --db-analytics-border-strong: rgba(255, 255, 255, 0.16);
  --db-analytics-border-stronger: rgba(255, 255, 255, 0.24);
  --db-analytics-text-primary: #f5f5f5;
  --db-analytics-text-secondary: #b6b4ae;
  --db-analytics-text-muted: #86847e;
  --db-analytics-surface-0: #141414;
  --db-analytics-surface-1: #0c0c0c;
  --db-analytics-surface-2: #000000;
  --db-analytics-accent: #5b9eec;
  --db-analytics-accent-bg: rgba(79, 70, 229, 0.16);
  --db-analytics-success: #3ecf5f;
  --db-analytics-warning: #f0b429;
  --db-analytics-danger: #f16060;
  --db-analytics-gradient-primary: linear-gradient(135deg, #8b5cf6 0%, #6366f1 50%, #3b82f6 100%);
}

* {
  box-sizing: border-box;
}
