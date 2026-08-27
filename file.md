@use '../../styles/_variables' as *;

// ===========================================================================
// Create Metric — single-page builder (all sections visible at once).
// Left: overview rail with a redesigned "living timeline" stepper.
// Right: every section stacked, separated by dashed dividers, capped at
// a wider 1000px reading column.
//
// Font scaling follows the same convention as Model Catalog: `.cm` sets a
// single base font-size, every descendant font-size is expressed in `em`
// relative to that base, so bumping `.cm`'s font-size on wide screens
// scales the whole builder proportionally from one place.
// ===========================================================================

// Neutrals, accents, and washes all come from the shared "ink" block in
// _variables.scss (theme-aware via _theme.scss custom properties) — same
// tokens Model Catalog uses, no locally-declared colors. $amber is kept
// as a local alias only because this file's selectors were written
// against that name; it points at the same $amber-ink token as everyone
// else.
$amber: $amber-ink;
// Toast needs to stay legible against its own dark chip in both themes
// (unlike page surfaces, which flip), so it uses the ink-1 dark value
// directly rather than the theme-flipping $ink token.
$ink-solid: #14161B;

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

$soft: 0 1px 2px rgba(20, 22, 27, 0.05);
$lift: 0 18px 40px -20px rgba(20, 22, 27, 0.30);

// base font-size the whole builder's internal `em` scale is built on
$base-font: 0.8125rem; // matches Model Catalog / Custom Metrics Dashboard base

%micro {
  font-family: $mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

@keyframes cm-spin { to { transform: rotate(360deg); } }
@keyframes cm-fade-up { from { opacity: 0; transform: translateY(8px); } to { opacity: 1; transform: translateY(0); } }
@keyframes cm-pop { 0% { transform: scale(0.7); opacity: 0; } 60% { transform: scale(1.08); } 100% { transform: scale(1); opacity: 1; } }
@keyframes cm-modal-in { from { opacity: 0; transform: translateY(12px) scale(0.98); } to { opacity: 1; transform: translateY(0) scale(1); } }
@keyframes cm-check-pop { 0% { transform: scale(0.4); opacity: 0; } 70% { transform: scale(1.15); } 100% { transform: scale(1); opacity: 1; } }

.spin { animation: cm-spin 0.8s linear infinite; }

// ---------------------------------------------------------------------------
// shell — master scale control. Every em-based font-size below responds
// to this. On very wide screens, bumping it to 1rem scales everything.
// ---------------------------------------------------------------------------
.cm {
  font-size: $base-font;
  flex: 1;
  min-height: 0;
  display: flex;
  flex-direction: column;

  @media (min-width: 1800px) { font-size: 1rem; }
}

.builder {
  flex: 1;
  min-height: 0;
  display: grid;
  grid-template-columns: 300px 1fr;
  gap: 0;
  overflow: hidden;
}

// ---------------------------------------------------------------------------
// LEFT RAIL — vertical stepper with a connecting line that tracks the
// 12px gap between rows, flat solid colors (no gradients), and a clear
// done/active state — jump to any section, any time.
// ---------------------------------------------------------------------------
.rail {
  display: flex;
  flex-direction: column;
  min-height: 0;
  background: $card;
  border-right: 1px solid $line;
  overflow-y: auto;
}

.rail__head {
  padding: 26px 22px 20px;
  border-bottom: 1px solid $line;
}

.rail__eyebrow {
  @extend %micro;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  color: $signal;
  display: flex;
  align-items: center;
  gap: 7px;
  margin-bottom: 10px;

  &::before { content: ''; width: 14px; height: 2px; border-radius: 2px; background: $signal; }
}

.rail__sub {
  margin-top: 4px;
  font-size: 0.9615em; // 0.78125rem / 0.8125rem
  color: $ink-3;
  line-height: 1.5;
}

.rail__steps {
  flex: 1;
  padding: 18px 14px 20px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.rail-step {
  position: relative;
  display: flex;
  align-items: flex-start;
  gap: 14px;
  width: 100%;
  text-align: left;
  padding: 13px 14px 13px 12px;
  border-radius: 16px;
  border: 1.5px solid transparent;
  background: transparent;
  cursor: pointer;
  transition: background 0.2s ease, border-color 0.2s ease, transform 0.2s ease, box-shadow 0.2s ease;

  &:hover {
    background: $paper;
    border-color: $line;
    transform: translateX(3px);

    .rail-step__arrow { opacity: 1; transform: translateX(0); }
  }

  &--done {
    background: $wash;
    border-color: rgba($signal, 0.16);

    &:hover { border-color: rgba($signal, 0.35); }
  }

  // vertical connector: starts right below this marker, and reaches all
  // the way through the 12px row gap into the top of the next marker.
  &:not(:last-child)::after {
    content: '';
    position: absolute;
    left: 30px;
    top: 49px;
    bottom: -25px; // 12px row gap + 13px next step's top padding
    width: 2px;
    border-radius: 2px;
    background: $line;
    z-index: 0;
    transition: background 0.25s ease;
  }
  &--done:not(:last-child)::after {
    background: $signal;
  }
}

.rail-step__marker {
  position: relative;
  z-index: 1;
  flex-shrink: 0;
  width: 36px;
  height: 36px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-family: $mono;
  font-size: 1.2308em; // 1.0rem / 0.8125rem
  font-weight: 800;
  color: $ink-3;
  background: $paper;
  border: 2px solid $line;
  transition: all 0.22s cubic-bezier(0.34, 1.56, 0.64, 1);

  .rail-step--done & {
    color: #fff;
    background: $signal;
    border-color: $signal;
    box-shadow: 0 4px 12px -3px rgba(43, 43, 245, 0.45);
    animation: cm-check-pop 0.3s ease;
  }

  .rail-step:hover:not(.rail-step--done) & {
    border-color: $ink-3;
    color: $ink-2;
    transform: scale(1.08);
  }
}

.rail-step__body {
  min-width: 0;
  flex: 1;
  padding-top: 5px;
}

.rail-step__label {
  display: block;
  font-size: 1.2308em; // 1.0rem / 0.8125rem
  font-weight: 700;
  color: $ink-2;
  transition: color 0.2s ease;

  .rail-step--done & { color: $ink; }
}

.rail-step__value {
  display: block;
  font-family: $mono;
  font-size: 1.0000em; // 0.8125rem / 0.8125rem
  color: $ink-3;
  margin-top: 5px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;

  .rail-step--done & { color: $signal; font-weight: 700; }
}

.rail-step__missing {
  display: flex;
  align-items: flex-start;
  gap: 5px;
  margin-top: 6px;
  font-family: $sans;
  font-size: 0.9231em; // 0.75rem / 0.8125rem
  font-weight: 600;
  color: $amber;
  line-height: 1.35;
  white-space: normal;

  svg { flex-shrink: 0; margin-top: 1px; }
}

.rail-step__arrow {
  flex-shrink: 0;
  align-self: center;
  color: $ink-3;
  opacity: 0;
  transform: translateX(-4px);
  transition: all 0.2s ease;

  .rail-step--done & { color: $signal; }
}

// ---------------------------------------------------------------------------
// RIGHT WORKSPACE — all sections stacked
// ---------------------------------------------------------------------------
.work {
  display: flex;
  flex-direction: column;
  min-height: 0;
  background: $paper;
}

.work__scroll {
  flex: 1;
  min-height: 0;
  overflow-y: auto;
  padding: 32px 36px;
}

.work__inner {
  margin: 0 auto;
}

.section {
  background: $card;
  border: 1px solid $line;
  border-radius: 18px;
  box-shadow: $soft;
  padding: 28px 32px 32px;
  margin-bottom: 18px;
  animation: cm-fade-up 0.28s ease;
}
.section--last { margin-bottom: 0; }

.work__eyebrow {
  @extend %micro;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  color: $ink-3;
  margin-bottom: 8px;
}

.work__title {
  font-family: $display;
  font-size: 1.6923em; // 1.375rem / 0.8125rem
  font-weight: 800;
  letter-spacing: -0.02em;
  color: $ink;
  line-height: 1.2;
}

.work__desc {
  margin-top: 6px;
  margin-bottom: 26px;
  font-size: 1.1538em; // 0.9375rem / 0.8125rem
  color: $ink-2;
  line-height: 1.5;
}

// ---------------------------------------------------------------------------
// sticky footer (Cancel / Run Validation / Save)
// ---------------------------------------------------------------------------
.work__foot {
  flex-shrink: 0;
  position: sticky;
  bottom: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  padding: 16px 36px;
  background: $card;
  border-top: 1px solid $line;
  z-index: 5;
}

.work__foot-info {
  font-family: $mono;
  font-size: 0.9231em; // 0.75rem / 0.8125rem
  font-weight: 700;
  color: $ink-3;
}

.work__foot-actions {
  display: flex;
  align-items: center;
  gap: 10px;
}

// ---------------------------------------------------------------------------
// buttons
// ---------------------------------------------------------------------------
.btn {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  padding: 10px 18px;
  border-radius: 10px;
  border: 1px solid $line;
  background: $card;
  color: $ink-2;
  font-family: $sans;
  font-size: 1.0385em; // 0.84375rem / 0.8125rem
  font-weight: 650;
  cursor: pointer;
  transition: all 0.15s ease;
  white-space: nowrap;

  &:hover:not(:disabled) { border-color: $ink-3; color: $ink; }
  &:disabled { opacity: 0.45; cursor: not-allowed; }
}

.btn--sm { padding: 7px 12px; font-size: 0.9615em; border-radius: 8px; } // 0.78125rem / 0.8125rem

.btn--primary {
  border-color: $signal;
  background: $signal;
  color: #fff;
  &:hover:not(:disabled) { background: $signal-2; border-color: $signal-2; color: #fff; transform: translateY(-1px); box-shadow: $lift; }
}

.btn--ghost { background: transparent; border-color: transparent; &:hover:not(:disabled) { background: $paper; border-color: $line; } }

.btn--ok {
  border-color: $ok; background: $ok; color: #fff;
  &:hover:not(:disabled) { filter: brightness(0.95); color: #fff; transform: translateY(-1px); box-shadow: $lift; }
}

.btn-icon {
  display: inline-flex; align-items: center; justify-content: center;
  width: 30px; height: 30px; border-radius: 8px;
  border: 1px solid transparent; background: transparent; color: $ink-3; cursor: pointer;
  transition: all 0.15s ease;
  &:hover { background: $danger-wash; border-color: rgba($danger, 0.2); color: $danger; }
}

// ---------------------------------------------------------------------------
// forms
// ---------------------------------------------------------------------------
.field { margin-bottom: 20px; }

// constrains a field to its natural content width instead of stretching
// across the (now unconstrained) workspace width — used for compact
// single-value inputs/toggles like the Built-in Check params.
.field--fit {
  max-width: 360px;

  .input { width: 100%; }
}

// side-by-side fields (e.g. Metric Name / Description) to use the wider
// workspace now that .work__inner has no max-width cap
.field-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 16px;

  .field { margin-bottom: 20px; }
}

.field__label {
  display: block;
  @extend %micro;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  color: $ink-2;
  margin-bottom: 8px;
}
.field__hint { font-size: 0.9615em; color: $ink-3; margin-top: 6px; } // 0.78125rem / 0.8125rem

.input, .textarea {
  width: 100%;
  border: 1.5px solid $line;
  border-radius: 10px;
  padding: 11px 13px;
  font-size: 1.1538em; // 0.9375rem / 0.8125rem
  font-family: $sans;
  color: $ink;
  background: $card;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;

  &::placeholder { color: $ink-3; }
  &:focus { outline: none; border-color: $signal; box-shadow: 0 0 0 3px $wash; }
}
.textarea { resize: vertical; min-height: 92px; line-height: 1.55; }

// ---------------------------------------------------------------------------
// selectable option cards (eval type / metric type)
// ---------------------------------------------------------------------------
.opt-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 12px;
}
.opt-grid--3 { grid-template-columns: repeat(3, 1fr); }
.opt-grid--4 { grid-template-columns: repeat(4, 1fr); }

.opt {
  position: relative;
  text-align: left;
  border: 1.5px solid $line;
  border-radius: 16px;
  padding: 18px;
  cursor: pointer;
  background: $card;
  transition: border-color 0.16s ease, box-shadow 0.16s ease, transform 0.16s ease, background 0.16s ease;

  &:hover:not(&--disabled) { border-color: $ink-3; transform: translateY(-2px); box-shadow: $lift; }

  &--selected {
    border-color: $signal;
    background: $wash;
    box-shadow: 0 0 0 1px $signal inset;
  }
  &--disabled { opacity: 0.5; cursor: not-allowed; }
}

.opt__icon {
  width: 42px;
  height: 42px;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: $paper;
  border: 1px solid $line;
  color: $signal;
  margin-bottom: 12px;
  transition: all 0.16s ease;

  .opt--selected & { background: $signal; border-color: $signal; color: #fff; }
}

.opt__title {
  font-family: $display;
  font-weight: 700;
  font-size: 1.2308em; // 1.0rem / 0.8125rem
  color: $ink;
  margin-bottom: 4px;
}
.opt__desc { font-size: 1em; color: $ink-2; line-height: 1.45; } // 0.8125rem / 0.8125rem

.opt__check {
  position: absolute;
  top: 14px; right: 14px;
  width: 20px; height: 20px;
  border-radius: 50%;
  background: $signal;
  color: #fff;
  display: flex; align-items: center; justify-content: center;
  animation: cm-pop 0.22s ease;
}

// ---------------------------------------------------------------------------
// rule builder — each rule is its own card with a solid accent bar, a
// header row (index pill + remove), and a clean fields grid underneath.
// ---------------------------------------------------------------------------
.rules { display: flex; flex-direction: column; gap: 14px; }

.rule {
  position: relative;
  padding: 16px 18px 18px;
  border: 1.5px solid $line;
  border-radius: 16px;
  background: $card;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;

  &:hover { border-color: $ink-3; box-shadow: $soft; }
}

.rule__head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  margin-bottom: 14px;
  padding-bottom: 12px;
  border-bottom: 1px dashed $line;
}

.rule__index {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  font-family: $mono;
  font-size: 0.9231em; // 0.75rem / 0.8125rem
  font-weight: 800;
  letter-spacing: 0.06em;
  text-transform: uppercase;
  color: $signal;

  &::before {
    content: '';
    width: 7px;
    height: 7px;
    border-radius: 50%;
    background: $signal;
  }
}

.rule__grid {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr 1fr;
  gap: 12px;
  min-width: 0;
}

.rule__field {
  display: flex;
  flex-direction: column;
  gap: 7px;
  min-width: 0;
}

.rule__field-label {
  @extend %micro;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  color: $ink-3;
}

.gate {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 4px 0;

  &::before, &::after { content: ''; flex: 1; height: 1px; background: $line; }
}

.gate__toggle {
  display: inline-flex;
  padding: 2px;
  background: $card;
  border: 1px solid $line;
  border-radius: 8px;
  gap: 1px;
}

.gate__opt {
  padding: 4px 12px;
  border-radius: 6px;
  border: none;
  background: transparent;
  color: $ink-2;
  font-family: $mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  cursor: pointer;
  &.on { background: $signal; color: #fff; }
}

.add-rule { align-self: flex-start; margin-top: 14px; }

// ---------------------------------------------------------------------------
// boolean switch — used by Simple/Built-in Check params (case sensitive,
// check arguments) wherever a plain true/false toggle is needed.
// ---------------------------------------------------------------------------
.switch {
  position: relative;
  flex-shrink: 0;
  width: 40px;
  height: 24px;
  border-radius: 999px;
  border: 1.5px solid $line;
  background: $paper;
  cursor: pointer;
  transition: background 0.16s ease, border-color 0.16s ease;
  padding: 0;

  &--on {
    background: $signal;
    border-color: $signal;
  }
}

.switch__thumb {
  position: absolute;
  top: 2px;
  left: 2px;
  width: 17px;
  height: 17px;
  border-radius: 50%;
  background: #fff;
  box-shadow: $soft;
  transition: transform 0.16s ease;

  .switch--on & { transform: translateX(16px); }
}

.switch-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
}

.switch-row__label {
  font-size: 1em; // 0.8125rem / 0.8125rem
  font-weight: 650;
  color: $ink;
}
.switch-row__hint {
  font-size: 0.9231em; // 0.75rem / 0.8125rem
  color: $ink-2;
  margin-top: 2px;
}

.summary {
  margin-top: 18px;
  padding: 16px;
  border-radius: 12px;
  background: $paper;
  border: 1px solid $line;
}
.summary__label {
  @extend %micro;
  font-size: 0.6923em; // 0.5625rem / 0.8125rem
  color: $ink-3;
  margin-bottom: 8px;
}
.summary__code {
  font-family: $mono;
  font-size: 1.0385em; // 0.84375rem / 0.8125rem
  color: $ink;
  line-height: 1.7;
  word-break: break-word;
}
.summary__token { color: $signal; font-weight: 700; }
.summary__gate { color: $amber; font-weight: 700; padding: 0 4px; }


// ---------------------------------------------------------------------------
// prompt templates
// ---------------------------------------------------------------------------
.tpl-list {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  margin-bottom: 18px;
}

.tpl {
  display: flex;
  align-items: flex-start;
  gap: 12px;
  padding: 14px 16px;
  border: 1.5px solid $line;
  border-radius: 12px;
  cursor: pointer;
  background: $card;
  transition: border-color 0.15s ease, background 0.15s ease;
  // horizontal flow: each card takes a natural share of the row and wraps
  // to the next line once it no longer fits, instead of always stacking
  flex: 1 1 280px;
  max-width: 380px;

  &:hover { border-color: $ink-3; }
  &--selected { border-color: $signal; background: $wash; }
}

.tpl__radio {
  flex-shrink: 0;
  width: 18px; height: 18px;
  margin-top: 3px;
  border-radius: 50%;
  border: 1.5px solid $line-2;
  display: flex; align-items: center; justify-content: center;

  .tpl--selected & { border-color: $signal; }
  &::after { content: ''; width: 9px; height: 9px; border-radius: 50%; background: $signal; opacity: 0; transition: opacity 0.15s ease; }
  .tpl--selected &::after { opacity: 1; }
}

.tpl__body {
  display: flex;
  flex-direction: column;
  min-width: 0;
  flex: 1;
}
.tpl__label { display: block; font-weight: 700; font-size: 1.1538em; color: $ink; line-height: 1.3; } // 0.9375rem / 0.8125rem
.tpl__desc { display: block; font-size: 1em; color: $ink-2; margin-top: 4px; line-height: 1.45; } // 0.8125rem / 0.8125rem
.tpl__tags { display: flex; flex-wrap: wrap; gap: 5px; margin-top: 10px; }

.token {
  font-family: $mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  color: $signal;
  background: $wash;
  border: 1px solid rgba($signal, 0.16);
  border-radius: 999px;
  padding: 3px 9px;
}

// ---------------------------------------------------------------------------
// judge model list
// ---------------------------------------------------------------------------
.models { display: flex; flex-wrap: wrap; gap: 10px; }

.model {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 13px 15px;
  border: 1.5px solid $line;
  border-radius: 12px;
  cursor: pointer;
  background: $card;
  transition: border-color 0.15s ease, background 0.15s ease, opacity 0.15s ease;
  // horizontal flow, same as Judge Prompt template cards: wrap to the
  // next line once a card no longer fits in the row
  flex: 1 1 260px;

  &:hover:not(&--disabled) { border-color: $ink-3; }
  &--selected { border-color: $signal; background: $wash; }
  &--disabled { opacity: 0.5; cursor: not-allowed; }
}

.model__radio {
  flex-shrink: 0;
  width: 18px; height: 18px;
  border-radius: 50%;
  border: 1.5px solid $line-2;
  display: flex; align-items: center; justify-content: center;
  .model--selected & { border-color: $signal; }
  &::after { content: ''; width: 9px; height: 9px; border-radius: 50%; background: $signal; opacity: 0; transition: opacity 0.15s ease; }
  .model--selected &::after { opacity: 1; }
}

.model__body { display: flex; flex-direction: column; min-width: 0; flex: 1; }
.model__name { font-family: $display; font-weight: 700; font-size: 1.1538em; color: $ink; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; } // 0.9375rem / 0.8125rem
.model__meta { font-family: $mono; font-size: 0.8462em; color: $ink-3; margin-top: 2px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; } // 0.6875rem / 0.8125rem

.model__health {
  flex-shrink: 0;
  display: inline-flex; align-items: center; gap: 6px;
  font-family: $mono; font-size: 0.7692em; font-weight: 700; text-transform: uppercase; // 0.625rem / 0.8125rem
}
.health-dot { width: 7px; height: 7px; border-radius: 50%; flex-shrink: 0; }
.health--healthy { color: $ok; .health-dot { background: $ok; } }
.health--unhealthy { color: $danger; .health-dot { background: $danger; } }
.health--checking { color: $ink-3; .health-dot { background: $ink-3; animation: cm-spin 1s linear infinite; border-radius: 2px; } }

// ---------------------------------------------------------------------------
// code editor
// ---------------------------------------------------------------------------
.code {
  border: 1px solid $line;
  border-radius: 14px;
  overflow: hidden;
}
.code__bar {
  display: flex; align-items: center; justify-content: space-between;
  padding: 10px 14px;
  background: $ink-solid;
  color: rgba(255, 255, 255, 0.7);
}
.code__lang {
  @extend %micro;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  color: rgba(255, 255, 255, 0.55);
}
.code__area {
  width: 100%;
  min-height: 520px;
  border: none;
  resize: vertical;
  padding: 16px;
  font-family: $mono;
  font-size: 1.0000em; // 0.8125rem / 0.8125rem
  line-height: 1.65;
  color: $ink;
  background: $card;
  &:focus { outline: none; }
}

// ---------------------------------------------------------------------------
// threshold slider
// ---------------------------------------------------------------------------
.thr {
  padding: 14px 16px;
  border: 1px solid $line;
  border-radius: 12px;
  background: $card;
}
.thr__row {
  display: flex;
  align-items: baseline;
  justify-content: space-between;
  gap: 12px;
  margin-bottom: 10px;
}
.thr__value {
  font-family: $mono;
  font-size: 1.2308em; // 1rem / 0.8125rem
  font-weight: 700;
  color: $signal;
  line-height: 1;
  flex-shrink: 0;
}
.thr__cap { font-size: 0.9231em; color: $ink-3; } // 0.75rem / 0.8125rem
.thr__slider {
  -webkit-appearance: none;
  appearance: none;
  width: 100%;
  height: 5px;
  border-radius: 999px;
  background: $line;
  outline: none;
  cursor: pointer;

  &::-webkit-slider-thumb {
    -webkit-appearance: none;
    appearance: none;
    width: 17px; height: 17px;
    border-radius: 50%;
    background: $signal;
    border: 2.5px solid $card;
    box-shadow: 0 2px 6px rgba(43, 43, 245, 0.4);
    cursor: pointer;
  }
  &::-moz-range-thumb {
    width: 17px; height: 17px;
    border-radius: 50%;
    background: $signal;
    border: 2.5px solid $card;
    cursor: pointer;
  }
}
.thr__scale {
  display: flex;
  justify-content: space-between;
  margin-top: 6px;
  font-family: $mono;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  color: $ink-3;
}

// ---------------------------------------------------------------------------
// dataset + preview (side by side, card-style columns)
// ---------------------------------------------------------------------------
.data-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 20px;
  align-items: start;
  margin-bottom: 8px;
}

.data-col {
  min-width: 0;
  border: 1px solid $line;
  border-radius: 16px;
  background: $card;
  overflow: hidden;
  box-shadow: $soft;
}

.data-col__head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  padding: 12px 16px;
  background: $paper;
  border-bottom: 1px solid $line;
}

.data-col__head-title {
  @extend %micro;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  color: $ink-3;
  display: flex;
  align-items: center;
  gap: 6px;
}

.data-col__count {
  font-family: $mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  color: $signal;
  background: $wash;
  border: 1px solid rgba($signal, 0.18);
  border-radius: 999px;
  padding: 2px 9px;
}

.data-col__body {
  padding: 12px;
  max-height: 400px;
  overflow-y: auto;
}

// ---- dataset cards — grid of self-sizing tiles; as many fit per row as
// space allows, wrapping to the next row otherwise. Name and count are
// stacked (not squeezed onto one line), with a top icon chip and a
// corner check badge that pops in when selected. ----------------------
.ds-list {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(160px, 1fr));
  gap: 10px;
}

.ds {
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 10px;
  min-width: 0;
  padding: 14px 14px 13px;
  border: 1.5px solid $line;
  border-radius: 14px;
  background: $card;
  cursor: pointer;
  transition: border-color 0.16s ease, background 0.16s ease, transform 0.16s ease, box-shadow 0.16s ease;

  &:hover { border-color: $ink-3; transform: translateY(-2px); box-shadow: $soft; }

  &--selected {
    border-color: $signal;
    background: $wash;
    box-shadow: 0 0 0 1px $signal inset;
  }
}

.ds__icon {
  flex-shrink: 0;
  width: 30px;
  height: 30px;
  border-radius: 9px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: $paper;
  border: 1px solid $line;
  color: $signal;
  transition: all 0.16s ease;

  .ds--selected & { background: $signal; border-color: $signal; color: #fff; }
}

.ds__name {
  width: 100%;
  font-family: $display;
  font-weight: 700;
  font-size: 1.0769em; // 0.875rem / 0.8125rem
  color: $ink;
  line-height: 1.3;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.ds__count {
  align-self: flex-start;
  display: inline-flex;
  align-items: center;
  font-family: $mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  color: $ink-3;
  background: $paper;
  border: 1px solid $line;
  border-radius: 999px;
  padding: 3px 9px;
  white-space: nowrap;
  transition: all 0.16s ease;

  .ds--selected & { color: $signal; background: rgba(255, 255, 255, 0.6); border-color: rgba($signal, 0.3); }
}

.ds__check {
  position: absolute;
  top: 10px;
  right: 10px;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: $signal;
  color: #fff;
  display: flex;
  align-items: center;
  justify-content: center;
  opacity: 0;
  transform: scale(0.5);
  transition: all 0.18s cubic-bezier(0.34, 1.56, 0.64, 1);

  .ds--selected & { opacity: 1; transform: scale(1); }
}

.q-list { display: flex; flex-direction: column; gap: 8px; }

.q {
  display: flex; gap: 10px;
  padding: 12px 14px;
  border: 1px solid $line;
  border-radius: 12px;
  background: $card;
  cursor: pointer;
  transition: border-color 0.13s ease, background 0.13s ease;
  &:hover { border-color: $ink-3; background: $paper; }
  &--on { border-color: $signal; background: $wash; }
}
.q__check {
  flex-shrink: 0; width: 17px; height: 17px; margin-top: 2px;
  border-radius: 5px; border: 1.5px solid $line-2;
  display: flex; align-items: center; justify-content: center;
  color: #fff;
  transition: all 0.13s ease;
  .q--on & { background: $signal; border-color: $signal; }
}
.q__body { min-width: 0; }
.q__q { display: block; font-size: 1.0385em; color: $ink; font-weight: 600; margin-bottom: 3px; } // 0.84375rem / 0.8125rem
.q__a { display: block; font-size: 0.9615em; color: $ink-2; } // 0.78125rem / 0.8125rem
.q__a-label { font-family: $mono; font-size: 0.7692em; color: $ink-3; margin-right: 5px; } // 0.625rem / 0.8125rem

.link-btn {
  border: none; background: none; padding: 0;
  color: $signal; font-size: 0.8846em; font-weight: 650; cursor: pointer; // 0.71875rem / 0.8125rem
  &:hover { text-decoration: underline; }
}

// ---------------------------------------------------------------------------
// validate & save
// ---------------------------------------------------------------------------
.validate-section {
  margin-top: 32px;
  padding-top: 24px;
  border-top: 1px dashed $line;
}

.validate-section__label {
  @extend %micro;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  color: $ink-2;
  margin-bottom: 6px;
}

.validate-section__desc {
  font-size: 1.0385em; // 0.84375rem / 0.8125rem
  color: $ink-2;
  margin-bottom: 16px;
}

// ---------------------------------------------------------------------------
// validation results
// ---------------------------------------------------------------------------
.banner {
  display: flex; align-items: center; gap: 8px;
  padding: 12px 16px; border-radius: 12px;
  font-size: 1.0385em; font-weight: 600; // 0.84375rem / 0.8125rem
  margin-bottom: 18px;
}
.banner--ok { background: $ok-wash; color: $ok; border: 1px solid rgba($ok, 0.2); }
.banner--err { background: $danger-wash; color: $danger; border: 1px solid rgba($danger, 0.2); }
.banner--info { background: $wash; color: $signal; border: 1px solid rgba($signal, 0.18); }

.results {
  border: 1px solid $line;
  border-radius: 14px;
  overflow: hidden;
}
.results__row {
  display: flex; align-items: flex-start; gap: 14px;
  padding: 14px 16px;
  border-bottom: 1px solid $line-2;
  &:last-child { border-bottom: none; }
}
.results__score {
  flex-shrink: 0;
  font-family: $mono; font-weight: 700; font-size: 1.2308em; // 1.0rem / 0.8125rem
  width: 48px; text-align: center;
}
.results__score--pass { color: $ok; }
.results__score--fail { color: $danger; }
.results__body { min-width: 0; flex: 1; }
.results__io { font-size: 1em; color: $ink; } // 0.8125rem / 0.8125rem
.results__reason { font-size: 0.9615em; color: $ink-2; margin-top: 4px; font-style: italic; } // 0.78125rem / 0.8125rem
.results__pill {
  flex-shrink: 0;
  @extend %micro;
  font-size: 0.6923em; // 0.5625rem / 0.8125rem
  padding: 3px 9px; border-radius: 999px;
}
.results__pill--pass { color: $ok; background: $ok-wash; }
.results__pill--fail { color: $danger; background: $danger-wash; }
.results__summary {
  display: flex; gap: 20px;
  padding: 12px 16px;
  background: $paper;
  border-top: 1px solid $line;
  font-size: 1.0385em; color: $ink-2; // 0.84375rem / 0.8125rem
  strong { color: $ink; font-family: $mono; }
}

// ---------------------------------------------------------------------------
// misc states
// ---------------------------------------------------------------------------
.loading, .empty {
  display: flex; align-items: center; justify-content: center; gap: 8px;
  padding: 28px; text-align: center;
  color: $ink-3; font-size: 1.0385em; // 0.84375rem / 0.8125rem
  border: 1px dashed $line;
  border-radius: 12px;
}

// ---------------------------------------------------------------------------
// success modal
// ---------------------------------------------------------------------------
.overlay {
  position: fixed; inset: 0; z-index: 300;
  display: flex; align-items: center; justify-content: center;
  background: rgba(10, 12, 18, 0.55);
  padding: 20px;
}
.modal {
  width: 100%; max-width: 400px;
  background: $card;
  border-radius: 20px;
  box-shadow: $lift;
  padding: 32px 28px 24px;
  text-align: center;
  animation: cm-modal-in 0.24s ease;
}
.modal__icon {
  width: 56px; height: 56px; margin: 0 auto 16px;
  border-radius: 50%;
  background: $ok-wash; color: $ok;
  display: flex; align-items: center; justify-content: center;
  animation: cm-pop 0.3s ease;
}
.modal__title { font-family: $display; font-size: 1.5385em; font-weight: 800; color: $ink; margin-bottom: 6px; } // 1.25rem / 0.8125rem
.modal__text { font-size: 1.0769em; color: $ink-2; margin-bottom: 8px; } // 0.8125rem / 0.8125rem
.modal__id {
  display: inline-block;
  font-family: $mono; font-size: 0.9231em; font-weight: 700; // 0.75rem / 0.8125rem
  color: $signal; background: $wash;
  border-radius: 8px; padding: 4px 10px; margin-bottom: 22px;
}
.modal__actions { display: flex; gap: 10px; }
.modal__actions .btn { flex: 1; justify-content: center; }

// ---------------------------------------------------------------------------
// toast (save error)
// ---------------------------------------------------------------------------
.toast {
  position: fixed;
  left: 50%; bottom: 26px;
  transform: translateX(-50%);
  z-index: 320;
  display: flex; align-items: center; gap: 8px;
  padding: 12px 18px;
  border-radius: 12px;
  background: $ink-solid; color: #fff;
  font-size: 1.0385em; font-weight: 600; // 0.84375rem / 0.8125rem
  box-shadow: $lift;
  animation: cm-fade-up 0.2s ease;
}

// ---------------------------------------------------------------------------
// responsive
// ---------------------------------------------------------------------------
@media (max-width: 1080px) {
  .builder { grid-template-columns: 1fr; }
  .rail {
    border-right: none;
    border-bottom: 1px solid $line;
    max-height: none;
  }
  .rail__steps { flex-direction: row; overflow-x: auto; }
  .rail-step { flex-direction: column; align-items: flex-start; min-width: 130px; }
  .rail-step:not(:last-child)::after { display: none; }
  .opt-grid--4 { grid-template-columns: repeat(2, 1fr); }
  .field-row { grid-template-columns: 1fr; }
}

@media (max-width: 760px) {
  .page-header { padding: 16px 18px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .work__scroll { padding: 22px 18px; }
  .work__foot { padding: 14px 18px; }
  .opt-grid, .opt-grid--3, .opt-grid--4 { grid-template-columns: 1fr; }
  .data-row { grid-template-columns: 1fr; }
  .ds-list { grid-template-columns: repeat(auto-fill, minmax(130px, 1fr)); }
  .rule__grid { grid-template-columns: 1fr 1fr; }
  .tpl { flex-basis: 100%; max-width: none; }
  .model { flex-basis: 100%; }
}
