/* Sharp form fields: inputs, dropdowns, text areas and segmented controls.
   Crisp 1px borders with real contrast, tighter corners, flat backgrounds (no soft inner shadow),
   solid text and a clear two-tone focus ring in the current palette's accent.
   Loaded after styles.css / rich.css / theme-dark.css; the :root prefix makes these rules win. */
:root {
  --field-radius: 6px;
  --field-bg: #ffffff;
  --field-line: #b8c0cc;
  --field-line-hover: #7d8696;
  --field-ink: #0b1220;
  --field-ph: #7b8494;
  --field-disabled: #f1f3f6;
  --field-chev: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='%231f2937' stroke-width='2.4' stroke-linecap='round' stroke-linejoin='round'%3E%3Cpath d='m7 10 5 5 5-5'/%3E%3C/svg%3E");
}
:root[data-theme="dark"] {
  --field-bg: #0a0c11;
  --field-line: #4a5264;
  --field-line-hover: #7a8396;
  --field-ink: #f5f7fa;
  --field-ph: #858da0;
  --field-disabled: #161a22;
  --field-chev: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='%23e5e7eb' stroke-width='2.4' stroke-linecap='round' stroke-linejoin='round'%3E%3Cpath d='m7 10 5 5 5-5'/%3E%3C/svg%3E");
}

:root input:not([type=checkbox]):not([type=file]):not([type=radio]):not([type=range]),
:root select, :root textarea {
  border: 1px solid var(--field-line); border-radius: var(--field-radius);
  background-color: var(--field-bg); color: var(--field-ink); box-shadow: none;
  font-weight: 500; letter-spacing: -.005em;
  -webkit-font-smoothing: antialiased; -moz-osx-font-smoothing: grayscale; text-rendering: optimizeLegibility;
  transition: border-color .1s, box-shadow .1s, background-color .1s;
}
:root input::placeholder, :root textarea::placeholder { color: var(--field-ph); font-weight: 400; opacity: 1; }

:root input:not([type=checkbox]):not([type=file]):not([type=radio]):not([type=range]):hover:not(:focus):not(:disabled), :root select:hover:not(:focus):not(:disabled), :root textarea:hover:not(:focus):not(:disabled) {
  border-color: var(--field-line-hover);
}
/* focus: solid accent edge plus a thin, tight halo, so the active field is unmistakable */
:root input:not([type=checkbox]):not([type=file]):not([type=radio]):not([type=range]):focus, :root select:focus, :root textarea:focus {
  border-color: var(--accent); outline: none;
  box-shadow: 0 0 0 1px var(--accent), 0 0 0 4px rgba(var(--accent-rgb), .16);
}
:root input:not([type=checkbox]):not([type=file]):not([type=radio]):not([type=range]):disabled, :root select:disabled, :root textarea:disabled {
  background-color: var(--field-disabled); color: var(--muted); cursor: not-allowed; border-style: dashed;
}

/* dropdowns: darker, bolder chevron that matches the text */
:root select {
  background-image: var(--field-chev); background-repeat: no-repeat; background-position: right 9px center; background-size: 16px;
}
:root select option { color: var(--ink); background: var(--surface); font-weight: 500; }

/* the label inside the split / min-max number boxes */
:root .inset small { color: var(--ink-2); font-weight: 600; }

/* segmented controls (On / Off, Fact / Section, ...) */
:root .seg { border-radius: 8px; border: 1px solid var(--field-line); background: var(--surface-3); box-shadow: none; }
:root .seg button { border-radius: 5px; }
:root .seg button.on { box-shadow: 0 0 0 1px var(--field-line), 0 1px 2px rgba(16, 24, 40, .12); }
:root[data-theme="dark"] .seg { background: var(--field-bg); }
:root[data-theme="dark"] .seg button.on { background: var(--surface-3); box-shadow: 0 0 0 1px var(--field-line); }

/* keep the borderless search box borderless */
:root .search input { border: 0 !important; box-shadow: none !important; background: none !important; }
:root .search { border-radius: var(--field-radius); border-color: var(--field-line); background: var(--field-bg); }
:root .search:focus-within { border-color: var(--accent); box-shadow: 0 0 0 1px var(--accent), 0 0 0 4px rgba(var(--accent-rgb), .16); }

/* ---- field widths: each control is as wide as its content needs, not as wide as the column.
   Only max-widths (plus fit-content for toggles), so narrow screens behave exactly as before. */
:root {
  --w-num: 200px;      /* numbers: chunk sizes, temperatures, counts */
  --w-text: 560px;     /* URLs, names, keys, short text */
  --w-select: 480px;   /* dropdowns */
  --w-area: 820px;     /* prompts and other multi-line text */
}
.field > input[type=number] { max-width: var(--w-num); }
.field > input:not([type=number]):not([type=checkbox]):not([type=file]) { max-width: var(--w-text); }
.field > select { max-width: var(--w-select); }
.field > textarea { max-width: var(--w-area); }
/* On / Off, Fact / Section, No / Yes: just wide enough for the options */
.field > .seg { width: fit-content; min-width: 200px; max-width: 100%; }
.field > .seg button { min-width: 92px; }
/* grouped number boxes */
.split4 { max-width: 640px; }
.tpl-minmax { max-width: 320px; }
/* template window: it is already bounded, so text fields use its full column; only the short id is compact */
.tpl-form .field > input:not([type=number]):not([type=checkbox]):not([type=file]) { max-width: none; }
.tpl-form .field > input.mono:not([type=number]):not([type=checkbox]):not([type=file]) { max-width: 300px; }
