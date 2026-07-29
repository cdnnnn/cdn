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
  font-size: calc(100% + 16px); // +1rem across the board, still respects browser zoom
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
