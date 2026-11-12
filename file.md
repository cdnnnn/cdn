@use '../../styles/_variables' as *;

// ===========================================================================
// SemcoEval — Run Console
// A precision "instrument panel" for assembling and launching an evaluation.
// Signature: a live Run Manifest (mono spec sheet) threaded by a signal rail.
// Header matches the History/Reports/Comparison/Sidebar design standard.
//
// All color tokens ($ink, $paper, $signal, $ok, $danger, $violet-ink,
// $ink-solid, etc.) come from the shared "ink" design system in
// ../../styles/_variables.scss (imported above) — this file no longer
// redeclares them locally. Neutrals resolve to theme CSS vars (see
// _theme.scss) for dark-mode support; $ink-solid/$ink-solid-hover are a
// FIXED near-black used only for "always dark" chips/buttons (option
// icons, the Continue button, the launch toast) — using themed $ink
// there would turn them near-white (and invisible) in dark mode. See the
// note above those two tokens in _theme.scss for why they're still CSS
// vars despite not actually varying by theme.
// ===========================================================================

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

$soft:  0 1px 2px rgba(20, 22, 27, 0.05);
$lift:  0 14px 30px -14px rgba(20, 22, 27, 0.22);
$ring:  0 0 0 3px rgba(43, 43, 245, 0.16);

// Base font-size every `em` font-size below is expressed relative to — same
// convention as Model Catalog / Sidebar. Bumping this (e.g. on wide
// screens, see the `@media (min-width: 1800px)` rules below) scales every
// descendant font-size proportionally from one place. This value has four
// independent application points since the component renders four separate
// DOM subtrees (.ev__header, .page → .ev, .ev-toast, .ev-preview-overlay)
// rather than one single wrapper — each sets `font-size: $ev-base-font;`
// itself so inheritance covers everything nested inside it.
$ev-base-font: 0.8125rem;

%micro {
  font-family: $mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

// ===========================================================================
// Header — matches History / Reports / Comparison / Sidebar header pattern
// ===========================================================================
.ev__header {
  // master scale control for this subtree — see $ev-base-font above
  font-size: $ev-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  flex-shrink: 0;
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
  gap: 1rem;
  padding: 24px 32px 20px;
  border-bottom: 1px solid $line;
  background: $card;

  h1 {
    font-family: $display;
    font-size: 1.8462em; // 1.5rem / 0.8125rem
    font-weight: 800;
    letter-spacing: -0.02em;
    color: $ink;
    line-height: 1.2;
  }
}

.ev__header-eyebrow {
  @extend %micro;
  display: flex;
  align-items: center;
  gap: 8px;
  color: $signal;
  margin-bottom: 6px;

  &::before {
    content: '';
    width: 16px;
    height: 2px;
    border-radius: 2px;
    background: $signal;
  }
}

.ev__header-sub {
  margin-top: 4px;
  font-size: 1.0385em; color: $ink-2; } // 0.84375rem / 0.8125rem

.ev__header-meta {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 3px;
}

.ev__header-status {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  padding: 7px 13px;
  border-radius: 999px;
  border: 1px solid $line;
  background: $paper;
  font-family: $mono;
  font-size: 0.8846em; // 0.71875rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.06em;
  text-transform: uppercase;
  color: $ink-2;
  white-space: nowrap;

  &::before {
    content: '';
    width: 6px;
    height: 6px;
    border-radius: 50%;
    background: $ink-3;
  }

  &[data-state='draft']::before { background: $signal; box-shadow: 0 0 0 3px $wash; }
  &[data-state='live'] { color: $signal; border-color: rgba($signal, 0.35); background: $wash; }
  &[data-state='live']::before { background: $signal; animation: ev-pulse 1.1s ease-in-out infinite; }
}

.ev__header-eta {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  font-size: 0.9231em; // 0.75rem / 0.8125rem
  font-weight: 600;
  color: $ink-3;
  white-space: nowrap;
}

.page {
  // master scale control for this subtree — cascades down through .ev and
  // every nested &__ selector below, since none of them reset font-size
  // themselves (they're all expressed in em relative to whatever ancestor
  // font-size is in effect). See $ev-base-font above.
  font-size: $ev-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  flex: 1;
  height: 100%;
  min-height: 0;
  padding: 22px 30px 26px;
  display: flex;
  flex-direction: column;
  background: $paper;
}

// ===========================================================================
// Root
// ===========================================================================
.ev {
  flex: 1;
  min-height: 0;
  display: flex;
  flex-direction: column;

  // ---- shell (manifest + stage) ------------------------------------------
  &__shell {
    flex: 1;
    min-height: 0;
    display: grid;
    grid-template-columns: 288px 1fr;
    gap: 16px;
  }

  // ========================================================================
  // SIGNATURE: Run Manifest
  // ========================================================================
  &__manifest {
    background: $card;
    border: 1px solid $line;
    border-radius: 16px;
    box-shadow: $soft;
    display: flex;
    flex-direction: column;
    min-height: 0;
    overflow: hidden;
  }

  &__manifest-head {
    flex-shrink: 0;
    padding: 18px 20px 16px;
    border-bottom: 1px solid $line-2;
  }

  &__manifest-eyebrow {
    @extend %micro;
    color: $ink-3;
    display: flex;
    align-items: center;
    justify-content: space-between;
  }

  &__manifest-pct {
    color: $signal;
    font-size: 0.9231em; letter-spacing: 0.06em; } // 0.75rem / 0.8125rem

  &__manifest-title {
    margin-top: 8px;
    font-family: $display;
    font-size: 1.2308em; // 1rem / 0.8125rem
    font-weight: 800;
    letter-spacing: -0.015em;
    color: $ink;
    line-height: 1.15;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;

    &[data-empty='true'] { color: $ink-3; font-style: normal; }
  }

  &__meter {
    margin-top: 12px;
    height: 4px;
    border-radius: 999px;
    background: $line;
    overflow: hidden;
  }

  &__meter-fill {
    height: 100%;
    border-radius: 999px;
    background: linear-gradient(90deg, $signal, $signal-2);
    transition: width 0.4s cubic-bezier(0.32, 0.72, 0, 1);
  }

  // ---- the spec list (each row = a step, with its live value) -------------
  &__spec {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding: 14px 12px 16px;
    display: flex;
    flex-direction: column;
    gap: 24px;
  }

  &__spec-row {
    position: relative;
    display: grid;
    grid-template-columns: 30px 1fr;
    align-items: start;
    gap: 12px;
    width: 100%;
    text-align: left;
    border: 0;
    background: transparent;
    padding: 12px 12px 12px 4px;
    border-radius: 12px;
    cursor: pointer;
    transition: background 0.15s ease;

    // Extends past this row's own bottom edge by exactly the flex `gap`
    // above (24px), so the line still reaches the next row's tick — the
    // larger the gap, the more this needs to stretch to stay unbroken.
    &::before {
      content: '';
      position: absolute;
      left: 18px;
      top: 38px;
      bottom: -24px;
      width: 2px;
      background: $line;
      transition: background 0.2s ease;
    }
    &:last-child::before { display: none; }

    &:disabled { cursor: default; }
    &:not(:disabled):hover { background: $paper; }
  }

  &__spec-tick {
    position: relative;
    z-index: 1;
    width: 28px;
    height: 28px;
    border-radius: 9px;
    display: grid;
    place-items: center;
    background: $card;
    border: 1.5px solid $line;
    color: $ink-3;
    font-family: $mono;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    font-weight: 700;
    transition: all 0.18s ease;
  }

  &__spec-body {
    min-width: 0;
    padding-top: 1px;
    display: flex;
    flex-direction: column;
    gap: 2px;
  }

  &__spec-label {
    @extend %micro;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    color: $ink-3;
    transition: color 0.18s ease;
  }

  &__spec-value {
    font-size: 1.1538em; // 0.9375rem / 0.8125rem
    font-weight: 600;
    color: $ink;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;

    &[data-empty='true'] {
      color: $ink-3;
      font-weight: 500;
      font-family: $mono;
    }
  }

  &__spec-row--done {
    &::before { background: $signal; }
    .ev__spec-tick { background: $signal; border-color: $signal; color: #fff; }
    .ev__spec-label { color: $ink-3; }
  }

  &__spec-row--active {
    background: $wash;
    .ev__spec-tick {
      background: $card;
      border-color: $signal;
      color: $signal;
      box-shadow: 0 0 0 4px rgba($signal, 0.14);
    }
    .ev__spec-label { color: $signal; }
    &:not(:disabled):hover { background: $wash; }
  }

  &__spec-row--todo { opacity: 0.9; }

  // ========================================================================
  // Stage (the working area for the current step)
  // ========================================================================
  &__stage {
    background: $card;
    border: 1px solid $line;
    border-radius: 16px;
    box-shadow: $soft;
    display: flex;
    flex-direction: column;
    min-height: 0;
    overflow: hidden;
  }

  &__stage-head {
    flex-shrink: 0;
    padding: 22px 28px 18px;
    border-bottom: 1px solid $line-2;
  }

  &__crumb {
    display: flex;
    align-items: center;
    gap: 9px;
    @extend %micro;
    color: $ink-3;

    b { color: $signal; font-weight: 700; }

    span:first-child {
      display: inline-flex;
      align-items: center;
      gap: 7px;
      color: $signal;
    }
  }

  &__crumb-sep {
    width: 3px;
    height: 3px;
    border-radius: 50%;
    background: $ink-3;
  }

  &__stage-title {
    margin-top: 12px;
    font-family: $display;
    font-size: 1.6923em; // 1.375rem / 0.8125rem
    font-weight: 800;
    letter-spacing: -0.025em;
    color: $ink;
    line-height: 1.1;
  }

  &__stage-sub {
    margin-top: 6px;
    font-size: 1.0385em; // 0.84375rem / 0.8125rem
    color: $ink-2;
    line-height: 1.5;
    max-width: 60ch;
  }

  &__stage-body {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding: 22px 28px 26px;
    display: flex;
    flex-direction: column;
  }

  &__anim {
    animation: ev-rise 0.34s cubic-bezier(0.22, 0.72, 0.16, 1) both;
  }

  // ---- footer nav ---------------------------------------------------------
  &__footer {
    flex-shrink: 0;
    padding: 16px 28px;
    border-top: 1px solid $line-2;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
  }

  &__hint {
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    color: $ink-3;
    display: flex;
    align-items: center;
    gap: 7px;
    min-width: 0;

    kbd {
      font-family: $mono;
      font-size: 0.8462em; // 0.6875rem / 0.8125rem
      color: $ink-2;
      background: $paper;
      border: 1px solid $line;
      border-bottom-width: 2px;
      border-radius: 5px;
      padding: 1px 6px;
    }
  }

  &__btn {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    font-family: $sans;
    font-size: 1.0385em; // 0.84375rem / 0.8125rem
    font-weight: 650;
    border-radius: 10px;
    padding: 10px 16px;
    cursor: pointer;
    border: 1px solid transparent;
    transition: background 0.16s ease, border-color 0.16s ease, color 0.16s ease, box-shadow 0.16s ease, transform 0.12s ease;

    &:disabled { cursor: not-allowed; opacity: 0.5; }

    &--ghost {
      background: transparent;
      border-color: $line;
      color: $ink-2;
      &:not(:disabled):hover { border-color: $ink-3; color: $ink; background: $paper; }
    }

    // fixed-dark chip — do NOT switch to $ink here, it would go near-white
    // (and invisible) in dark mode since $ink is theme-aware.
    &--primary {
      background: $ink-solid;
      color: #fff;
      box-shadow: $soft;
      &:not(:disabled):hover { background: $ink-solid-hover; transform: translateY(-1px); box-shadow: $lift; }
    }

    &--launch {
      background: $signal;
      color: #fff;
      box-shadow: 0 8px 20px -8px rgba($signal, 0.7);
      &:not(:disabled):hover { background: $signal-2; transform: translateY(-1px); }
    }
  }

  // ========================================================================
  // Shared field primitives
  // ========================================================================
  &__field {
    max-width: 620px;

    & + & { margin-top: 20px; }
  }

  &__label {
    display: flex;
    align-items: center;
    gap: 7px;
    @extend %micro;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    color: $ink-2;
    margin-bottom: 9px;

    .opt {
      font-family: $sans;
      letter-spacing: 0;
      text-transform: none;
      font-weight: 500;
      font-size: 0.9231em; color: $ink-3; } // 0.75rem / 0.8125rem
  }

  &__input {
    width: 100%;
    border: 1.5px solid $line;
    border-radius: 11px;
    padding: 12px 14px;
    font-size: 1.1538em; // 0.9375rem / 0.8125rem
    font-weight: 500;
    font-family: $sans;
    color: $ink;
    background: $card;
    transition: border-color 0.15s ease, box-shadow 0.15s ease;

    &::placeholder { color: $ink-3; font-weight: 400; }
    &:focus { outline: none; border-color: $signal; box-shadow: $ring; }
    &:disabled { background: $paper; color: $ink-2; }
  }

  &__input-wrap {
    position: relative;
    svg {
      position: absolute;
      top: 50%;
      left: 15px;
      transform: translateY(-50%);
      color: $ink-3;
      pointer-events: none;
    }
    input { padding-left: 42px; }
  }

  // ---- big "name your run" input -----------------------------------------
  &__name-input {
    width: 100%;
    border: 0;
    border-bottom: 2px solid $line;
    border-radius: 0;
    padding: 8px 2px 12px;
    background: transparent;
    font-family: $display;
    font-size: 1.6923em; // 1.375rem / 0.8125rem
    font-weight: 800;
    letter-spacing: -0.03em;
    color: $ink;
    transition: border-color 0.16s ease;

    &::placeholder { color: $ink-3; font-weight: 700; }
    &:focus { outline: none; border-color: $signal; }
  }

  &__name-caption {
    margin-top: 10px;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    color: $ink-3;
    display: flex;
    align-items: center;
    gap: 7px;
  }

  // ---- quick-start presets (mono chips) ----------------------------------
  &__quick {
    margin-top: 30px;
    max-width: 620px;
  }

  &__quick-head {
    @extend %micro;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    color: $ink-3;
    margin-bottom: 11px;
  }

  &__quick-row {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
  }

  &__preset {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    padding: 8px 12px 8px 11px;
    border: 1px solid $line;
    border-radius: 999px;
    background: $card;
    cursor: pointer;
    font-family: $mono;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 600;
    color: $ink-2;
    transition: all 0.15s ease;

    svg { color: $ink-3; transition: color 0.15s ease; }

    &:hover {
      border-color: $ink;
      color: $ink;
      transform: translateY(-1px);
      svg { color: $signal; }
    }

    &--on {
      border-color: $signal;
      background: $wash;
      color: $signal;
      svg { color: $signal; }
    }
  }

  // ---- tips note ----------------------------------------------------------
  &__note {
    margin-top: 28px;
    max-width: 620px;
    display: flex;
    gap: 12px;
    padding: 14px 16px;
    border: 1px solid $line;
    border-left: 2.5px solid $signal;
    border-radius: 12px;
    background: $card;
  }

  &__note-icon {
    flex-shrink: 0;
    color: $signal;
    margin-top: 1px;
  }

  &__note-title {
    font-size: 1em; // 0.8125rem / 0.8125rem (base)
    font-weight: 700;
    color: $ink;
    margin-bottom: 6px;
  }

  &__note-list {
    display: flex;
    flex-direction: column;
    gap: 5px;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    color: $ink-2;
    line-height: 1.5;

    li { display: flex; gap: 8px; }
    li::before {
      content: '—';
      color: $signal;
      flex-shrink: 0;
    }
  }

  // ========================================================================
  // Option rows (Type step) & framework
  // ========================================================================
  &__options {
    display: flex;
    gap: 10px;
  }

  &__option {
    position: relative;
    display: flex;
    align-items: center;
    gap: 15px;
    width: 100%;
    text-align: left;
    padding: 16px 52px 16px 16px;
    border: 1.5px solid $line;
    border-radius: 14px;
    background: $card;
    cursor: pointer;
    transition: border-color 0.18s ease, box-shadow 0.18s ease, transform 0.18s ease, background 0.18s ease;

    &:hover {
      border-color: $ink-3;
      box-shadow: $lift;
      transform: translateY(-2px);
    }

    &--on {
      border-color: $signal;
      background: $wash;
      &:hover { border-color: $signal; }
    }

    &--off {
      opacity: 0.55;
      cursor: not-allowed;
      &:hover { border-color: $line; box-shadow: none; transform: none; }
    }
  }

  // fixed-dark chip — icon glyph is always white-on-dark regardless of theme
  &__option-icon {
    flex-shrink: 0;
    width: 48px;
    height: 48px;
    border-radius: 13px;
    display: grid;
    place-items: center;
    background: $ink-solid;
    color: #fff;
    position: relative;
    overflow: hidden;

    &::after {
      content: '';
      position: absolute;
      inset: 0;
      background: linear-gradient(140deg, transparent 45%, rgba(255,255,255,0.16) 140%);
    }
    svg { position: relative; z-index: 1; }
  }
  &__option-icon--agent { background: $violet-ink; }
  &__option-icon--rag   { background: $sky-ink; }

  &__option-main {
    flex: 1;
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 4px;
  }

  &__option-name {
    display: flex;
    align-items: center;
    gap: 9px;
    font-family: $display;
    font-size: 1.3846em; // 1.125rem / 0.8125rem
    font-weight: 700;
    color: $ink;
  }

  &__badge {
    @extend %micro;
    font-size: 0.6923em; // 0.5625rem / 0.8125rem
    color: $ink-3;
    background: $paper;
    border: 1px solid $line;
    border-radius: 999px;
    padding: 2px 8px;
  }

  &__option-desc {
    font-size: 1.1538em; // 0.9375rem / 0.8125rem
    color: $ink-2;
    line-height: 1.5;
  }

  // selection marker (shared)
  &__mark {
    position: absolute;
    top: 50%;
    right: 16px;
    transform: translateY(-50%);
    width: 22px;
    height: 22px;
    border-radius: 50%;
    display: grid;
    place-items: center;
    background: $signal;
    color: #fff;
    box-shadow: 0 2px 6px rgba($signal, 0.4);
  }

  &__section {
    margin-top: 26px;
    padding-top: 22px;
    border-top: 1px solid $line-2;
    max-width: 720px;
  }

  &__section-hint {
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    color: $ink-3;
    margin: 4px 0 14px;
  }

  &__fw-grid {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 10px;
  }

  &__fw {
    position: relative;
    display: flex;
    align-items: center;
    gap: 12px;
    text-align: left;
    padding: 13px 42px 13px 13px;
    border: 1.5px solid $line;
    border-radius: 12px;
    background: $card;
    cursor: pointer;
    transition: border-color 0.16s ease, box-shadow 0.16s ease, transform 0.16s ease, background 0.16s ease;

    &:hover { border-color: $ink-3; transform: translateY(-2px); box-shadow: $lift; }
    &--on { border-color: $signal; background: $wash; }
  }

  &__fw-icon {
    flex-shrink: 0;
    width: 36px;
    height: 36px;
    border-radius: 10px;
    display: grid;
    place-items: center;
    background: $wash;
    color: $signal;
  }

  &__fw-name { font-family: $display; font-size: 1.1538em; font-weight: 700; color: $ink; } // 0.9375rem / 0.8125rem
  &__fw-desc { font-size: 1em; color: $ink-2; margin-top: 2px; line-height: 1.4; } // 0.8125rem / 0.8125rem (base)

  // ========================================================================
  // Per-step toolbar: search + refresh
  // Used on the Providers, Models, Test Suite (browse tab), and Metrics
  // steps. Search filters the already-fetched list client-side; refresh
  // re-dispatches that step's existing fetch thunk.
  // ========================================================================
  &__step-toolbar {
    display: flex;
    align-items: center;
    gap: 10px;
    margin-bottom: 14px;
  }

  &__toolbar-search {
    flex: 1;
    min-width: 0;
    max-width: 320px;
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 8px 12px;
    border: 1.5px solid $line;
    border-radius: 10px;
    background: $card;
    color: $ink-3;
    transition: border-color 0.15s ease, box-shadow 0.15s ease, color 0.15s ease;

    &:focus-within {
      border-color: $signal;
      box-shadow: $ring;
      color: $signal;
    }

    svg { flex-shrink: 0; }

    input {
      flex: 1;
      min-width: 0;
      border: 0;
      background: transparent;
      font-family: $sans;
      font-size: 1em; // 0.8125rem / 0.8125rem (base)
      font-weight: 500;
      color: $ink;
      outline: none;

      &::placeholder { color: $ink-3; font-weight: 400; }
    }
  }

  &__toolbar-refresh {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 34px;
    height: 34px;
    border: 1.5px solid $line;
    border-radius: 10px;
    background: $card;
    color: $ink-2;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover:not(:disabled) { border-color: $signal; color: $signal; background: $wash; }
    &:disabled { cursor: default; color: $signal; }
  }

  // ---- Test Suite: All / Custom / Deepeval filter (Change-1) -------------
  &__filter-row {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    margin-bottom: 14px;
  }

  &__filter-chip {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    padding: 7px 8px 7px 13px;
    border: 1.5px solid $line;
    border-radius: 999px;
    background: $card;
    color: $ink-2;
    font-family: $sans;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $ink-3; color: $ink; }

    &--on {
      border-color: $signal;
      background: $wash;
      color: $signal;
    }
  }

  &__filter-chip-count {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    min-width: 20px;
    height: 20px;
    padding: 0 6px;
    border-radius: 999px;
    background: $paper;
    color: $ink-3;
    font-family: $mono;
    font-size: 0.8077em; font-weight: 700; } // 0.65625rem / 0.8125rem
  &__filter-chip--on &__filter-chip-count {
    background: $signal;
    color: #fff;
  }

  // ========================================================================
  // Skeleton loaders (Change-3) — shown in place of a step's grid/chips
  // while its "refresh" request is in flight. Shapes loosely echo each
  // step's real card so the layout doesn't jump when data arrives.
  // ========================================================================
  &__skel-block {
    display: block;
    border-radius: 6px;
    background: linear-gradient(90deg, $line 25%, $line-2 37%, $line 63%);
    background-size: 400% 100%;
    animation: ev-shimmer 1.4s ease infinite;

    &--icon { width: 40px; height: 40px; border-radius: 11px; flex-shrink: 0; }
    &--icon-sm { width: 36px; height: 36px; border-radius: 10px; flex-shrink: 0; }
    &--line { height: 11px; border-radius: 5px; }
    &--pill { width: 72px; height: 16px; border-radius: 999px; margin-top: 2px; }
    &--tag { width: 54px; height: 18px; border-radius: 6px; }
    &--stat { width: 58px; height: 26px; border-radius: 7px; }
    &--chip { height: 34px; border-radius: 999px; }
  }

  &__skel-pcard {
    display: flex;
    align-items: flex-start;
    gap: 13px;
    padding: 15px 42px 15px 15px;
    border: 1.5px solid $line;
    border-radius: 14px;
    background: $card;
  }
  &__skel-lines {
    flex: 1;
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  &__skel-mcard {
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 15px 16px;
    border: 1.5px solid $line;
    border-radius: 14px;
    background: $card;
  }
  &__skel-caps { display: flex; gap: 6px; }
  &__skel-stats {
    display: flex;
    gap: 10px;
    padding-top: 10px;
    border-top: 1px solid $line-2;
  }

  &__skel-dcard {
    display: flex;
    flex-direction: column;
    gap: 12px;
    padding: 15px 16px;
    border: 1.5px solid $line;
    border-radius: 14px;
    background: $card;
  }
  &__skel-dcard-top {
    display: flex;
    align-items: center;
    gap: 11px;
  }

  // ========================================================================
  // Card grids (providers / models / datasets)
  // ========================================================================
  &__scroll {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    margin: 0 -6px;
    padding: 4px 6px;
  }

  // ---- Models step (Step 3): search toolbar + Retries & timeout panel
  // stay fixed; only the card grid below them scrolls. -------------------
  &__models-panel {
    flex: 1;
    min-height: 0;
    display: flex;
    flex-direction: column;
  }

  &__models-scroll {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    margin: 8px -6px 0;
    padding: 4px 6px;
  }

  // ---- RAG Models step: one column per model category, single-select
  // within each column (see toggleModelForCategory). --------------------
  &__rag-columns {
    display: flex;
    align-items: stretch;
    gap: 14px;
    height: 100%;
    min-height: 0;
    overflow-x: auto;
  }

  &__rag-col {
    // No fixed width — each column grows to share the available space
    // equally; min-width just keeps a column from getting crushed if
    // there are many categories (overflow-x on &__rag-columns takes over
    // once they can't all fit at that minimum).
    flex: 1 1 0;
    min-width: 240px;
    min-height: 0;
    display: flex;
    flex-direction: column;
    border: 1px solid $line;
    border-radius: 14px;
    background: $paper;
    overflow: hidden;
  }

  &__rag-col-head {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
    padding: 12px 14px;
    border-bottom: 1px solid $line;
    background: $card;
  }

  // Per-category search box (Embedding/Reranker/LLM each get their own,
  // independent of the others and of the Model/Agent grid's shared
  // search).
  &__rag-col-search {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 7px;
    margin: 10px 12px 0;
    padding: 6px 10px;
    border: 1px solid $line;
    border-radius: 9px;
    background: $card;

    svg { flex-shrink: 0; color: $ink-3; }

    input {
      flex: 1;
      min-width: 0;
      border: 0;
      background: transparent;
      font-family: $sans;
      font-size: 0.8846em; // 0.71875rem / 0.8125rem
      color: $ink;

      &:focus { outline: none; }
      &::placeholder { color: $ink-3; }
    }
  }

  &__rag-col-title {
    min-width: 0;
    font-family: $display;
    font-size: 1.0em;
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $ink;
    overflow: hidden;
    white-space: nowrap;
    text-overflow: ellipsis;
  }

  &__rag-col-status {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 4px;
    padding: 3px 8px;
    border-radius: 999px;
    background: $wash;
    color: $ink-3;
    font-family: $sans;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.03em;
  }
  &__rag-col-status--on { background: rgba($ok, 0.14); color: $ok; }

  // Own scrollbar per category — independent of the other columns and of
  // the page's own scroll, so a long list in one category never pushes
  // another category's cards out of view.
  &__rag-col-list {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 12px;

    // Cards inside a RAG column stack in a single column instead of
    // wrapping into a multi-column grid like &__grid does.
    .ev__mcard { width: 100%; }
  }

  &__grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(258px, 1fr));
    gap: 12px;

    @media (min-width: 1800px) {
      grid-template-columns: repeat(auto-fill, minmax(310px, 1fr));
    }
  }

  &__grid--wide {
    grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  }

  // ---- provider card ------------------------------------------------------
  &__pcard {
    position: relative;
    display: flex;
    align-items: flex-start;
    gap: 13px;
    text-align: left;
    padding: 15px 42px 15px 15px;
    border: 1.5px solid $line;
    border-radius: 14px;
    background: $card;
    cursor: pointer;
    transition: border-color 0.16s ease, box-shadow 0.16s ease, transform 0.16s ease, background 0.16s ease;

    &:hover { border-color: $ink-3; box-shadow: $lift; transform: translateY(-2px); }
    &--on { border-color: $signal; background: $wash; &:hover { border-color: $signal; } }
  }

  &__pcard-icon {
    flex-shrink: 0;
    width: 40px;
    height: 40px;
    border-radius: 11px;
    display: grid;
    place-items: center;
    background: $paper;
    border: 1px solid $line;
    color: $ink;
    font-family: $display;
    font-weight: 800;
    font-size: 1.2308em; transition: all 0.16s ease; } // 1rem / 0.8125rem
  &__pcard--on &__pcard-icon { background: $signal; border-color: $signal; color: #fff; }

  &__pcard-body { min-width: 0; display: flex; flex-direction: column; gap: 3px; }
  &__pcard-name { font-family: $display; font-size: 1.3846em; font-weight: 700; color: $ink; } // 1.125rem / 0.8125rem
  &__pcard-meta { font-size: 1.1538em; color: $ink-3; } // 0.9375rem / 0.8125rem

  &__pill {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    margin-top: 4px;
    width: fit-content;
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ok;
    background: $ok-wash;
    border-radius: 999px;
    padding: 3px 8px 3px 6px;

    &::before { content: ''; width: 5px; height: 5px; border-radius: 50%; background: $ok; }
  }

  // ---- model card ---------------------------------------------------------
  &__mcard {
    position: relative;
    display: flex;
    flex-direction: column;
    gap: 10px;
    text-align: left;
    padding: 15px 16px;
    border: 1.5px solid $line;
    border-radius: 14px;
    background: $card;
    cursor: pointer;
    transition: border-color 0.16s ease, box-shadow 0.16s ease, transform 0.16s ease, background 0.16s ease;

    &:hover { border-color: $ink-3; box-shadow: $lift; transform: translateY(-2px); }
    &--on { border-color: $signal; background: $wash; &:hover { border-color: $signal; } }
  }

  // locked = provider chosen but health not yet confirmed successful;
  // dims the interaction affordance so it doesn't read as clickable-to-select.
  &__mcard--locked {
    cursor: default;
    &:hover { border-color: $line; box-shadow: none; transform: none; }
  }

  // checking = a health check (auto or manual) is currently in flight for
  // this card — a soft pulsing border + sheen so it reads as "in progress"
  // at a glance, on top of the "Checking…" badge text.
  &__mcard--checking {
    position: relative;
    overflow: hidden;
    border-color: rgba($signal, 0.35);
    animation: ev-mcard-pulse 1.6s ease-in-out infinite;

    &::after {
      content: '';
      position: absolute;
      inset: 0;
      background: linear-gradient(100deg, transparent 30%, rgba($signal, 0.08) 50%, transparent 70%);
      background-size: 200% 100%;
      animation: ev-mcard-sheen 1.6s ease-in-out infinite;
      pointer-events: none;
    }
  }

  &__mcard-top {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 10px;
  }

  &__mcard-name { font-family: $display; font-size: 1.1154em; font-weight: 700; color: $ink; line-height: 1.25; } // 0.90625rem / 0.8125rem
  &__mcard-provider { font-size: 0.8846em; color: $ink-3; } // 0.71875rem / 0.8125rem

  // ---- provider row + manual health check (Step 3) -------------------------
  &__mcard-provider-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
    margin-top: 2px;
  }

  &__mcard-hint {
    margin-top: 10px;
    padding-top: 10px;
    border-top: 1px dashed $line-2;
    font-size: 0.8846em; // 0.71875rem / 0.8125rem
    color: $ink-3;
    line-height: 1.4;
  }

  &__health-badge {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 4px;
    padding: 3px 8px;
    border-radius: 999px;
    border: 1px solid transparent;
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.02em;
    white-space: nowrap;

    &--success {
      color: $ok;
      background: $ok-wash;
    }

    &--failed {
      color: $danger;
      background: $danger-wash;
      cursor: pointer;
      border: 0;
      &:hover { background: rgba($danger, 0.16); }
    }

    &--loading {
      color: $ink-3;
      background: $paper;
    }
  }

  &__health-check-btn {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 4px;
    padding: 3px 9px;
    border-radius: 999px;
    border: 1px solid $signal;
    background: $card;
    color: $signal;
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.02em;
    white-space: nowrap;
    cursor: pointer;
    transition: background 0.15s ease, color 0.15s ease;

    &:hover { background: $wash; }
  }

  &__mcard-mark {
    flex-shrink: 0;
    width: 20px;
    height: 20px;
    border-radius: 50%;
    display: grid;
    place-items: center;
    background: $signal;
    color: #fff;
  }

  &__caps { display: flex; flex-wrap: wrap; gap: 5px; }
  &__cap {
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 600;
    letter-spacing: 0.02em;
    color: $ink-2;
    background: $paper;
    border: 1px solid $line;
    border-radius: 6px;
    padding: 2px 7px;
  }

  &__mcard-stats {
    display: flex;
    flex-wrap: wrap;
    gap: 14px;
    padding-top: 10px;
    border-top: 1px solid $line-2;
  }

  &__stat { display: flex; flex-direction: column; gap: 1px; }
  &__stat-k { @extend %micro; font-size: 0.6923em; color: $ink-3; } // 0.5625rem / 0.8125rem
  &__stat-v { font-family: $mono; font-size: 0.9615em; font-weight: 700; color: $ink; letter-spacing: -0.01em; } // 0.78125rem / 0.8125rem

  // Per-model retry/timeout — shown on a model card once it's selected.
  // Wrapped so clicks on the inputs (stopPropagation'd in the component)
  // don't also toggle the card's selection.
  &__mcard-retry {
    display: flex;
    flex-direction: column;
    gap: 8px;
    margin-top: 10px;
    padding-top: 10px;
    border-top: 1px solid $line-2;
    cursor: default;
  }
  &__mcard-retry-fields {
    display: flex;
    gap: 10px;
  }
  &__mcard-retry-hint {
    display: flex;
    align-items: flex-start;
    gap: 5px;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    color: $ink-3;
    line-height: 1.3;

    svg { flex-shrink: 0; margin-top: 2px; }
  }
  &__mcard-retry-field {
    flex: 1;
    display: flex;
    flex-direction: column;
    gap: 4px;

    label {
      @extend %micro;
      font-size: 0.6923em; // 0.5625rem / 0.8125rem
      color: $ink-3;
    }

    input {
      width: 100%;
      border: 1.5px solid $line;
      border-radius: 8px;
      padding: 5px 8px;
      font-family: $mono;
      font-size: 0.8846em; // 0.71875rem / 0.8125rem
      font-weight: 700;
      color: $ink;
      background: $card;
      transition: border-color 0.15s ease, box-shadow 0.15s ease;

      &:focus { outline: none; border-color: $signal; box-shadow: $ring; }
    }
  }

  // ---- Retry/timeout mode: "Apply to all" vs "Individually" (Models step) -
  &__retry-mode {
    margin-top: 14px;
    margin-bottom: 4px;
    padding: 12px 14px;
    border: 1px solid $line;
    border-radius: 12px;
    background: $paper;
  }

  &__retry-mode-head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    flex-wrap: wrap;
    gap: 10px;
  }

  &__retry-mode-all {
    display: flex;
    align-items: flex-end;
    flex-wrap: wrap;
    gap: 12px;
    margin-top: 12px;

    .ev__mcard-retry-field { flex: 0 0 130px; }
  }

  &__retry-mode-note {
    display: flex;
    align-items: flex-start;
    gap: 5px;
    flex: 1 1 100%;
    margin-top: 10px;
    font-size: 0.8846em; // 0.71875rem / 0.8125rem
    color: $ink-3;
    line-height: 1.4;

    svg { flex-shrink: 0; margin-top: 2px; }
  }

  // ---- Retest on Wrong body (card wrapper is &__msec; header lives there
  // too, with the Yes/No radio row as a head-action). The two controls
  // (Rounds / Verify Metric) each get their own bordered, tinted panel so
  // they read as clearly separate settings rather than two fields
  // floating in the same open space. -------------------------------------
  &__retest-fields {
    display: flex;
    flex-wrap: wrap;
    align-items: stretch;
    gap: 16px;
  }

  &__retest-panel {
    flex: 0 0 200px;
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 14px 16px;
    border: 1px solid $line;
    border-radius: 12px;
    background: $card;
    // Thin colored top edge distinguishes the two panels from each other
    // at a glance (rounds vs. verify metric), on top of the shared card
    // border/background separation.
    border-top: 3px solid $signal;

    input[type='number'] {
      width: 100%;
      border: 1.5px solid $line;
      border-radius: 8px;
      padding: 7px 10px;
      font-family: $mono;
      font-size: 1.0385em; // 0.84375rem / 0.8125rem
      font-weight: 700;
      color: $ink;
      background: $paper;
      transition: border-color 0.15s ease, box-shadow 0.15s ease;

      &:focus { outline: none; border-color: $signal; box-shadow: $ring; }
    }
  }

  &__retest-panel--verify {
    flex: 1 1 280px;
  }

  &__retest-panel-label {
    display: flex;
    align-items: center;
    gap: 6px;
    @extend %micro;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    color: $ink-2;

    svg { flex-shrink: 0; color: $signal; }
  }

  &__retest-panel-hint {
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    color: $ink-3;
    line-height: 1.4;
  }

  // ========================================================================
  // Test-suite step: tabs, dataset grid, subgroup rail, upload
  // ========================================================================
  &__tabs {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    padding: 4px;
    border: 1px solid $line;
    border-radius: 12px;
    background: $paper;
    margin-bottom: 18px;
  }

  &__tab {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    padding: 8px 16px;
    border: 0;
    border-radius: 9px;
    background: transparent;
    color: $ink-2;
    font-family: $sans;
    font-size: 1em; // 0.8125rem / 0.8125rem (base)
    font-weight: 650;
    cursor: pointer;
    transition: all 0.16s ease;

    &:hover { color: $ink; }
    &--on { background: $card; color: $signal; box-shadow: $soft; }
  }

  &__suite {
    flex: 1;
    min-height: 0;
    display: grid;
    grid-template-columns: minmax(0, 1fr) 390px;
    grid-template-rows: minmax(0, 1fr);
    gap: 20px;
  }

  // "N suites selected" counter, pushed to the right of the use-case chips.
  &__suite-selected {
    margin-left: auto;
    align-self: center;
    font-family: $mono;
    font-size: 0.9615em;
    color: $ink-2;
    b { color: $signal; font-weight: 700; }
  }

  // Harness suite card: description + "Scored by" line under the tags.
  &__dcard-desc {
    padding-top: 10px;
    border-top: 1px solid $line-2;
    font-size: 0.9231em;
    color: $ink-2;
    line-height: 1.45;

    p + p { margin-top: 5px; }
    p span {
      @extend %micro;
      font-size: 0.7692em;
      color: $ink-3;
      margin-right: 6px;
    }
  }

  &__suite-scroll {
    min-width: 0;
    min-height: 0;
    overflow-y: auto;
    margin: 0 -6px;
    padding: 2px 6px 6px;
  }

  &__dgrid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
    gap: 12px;

    @media (min-width: 1800px) {
      grid-template-columns: repeat(auto-fill, minmax(310px, 1fr));
    }
  }

  &__dcard {
    position: relative;
    display: flex;
    flex-direction: column;
    gap: 11px;
    text-align: left;
    padding: 15px 16px;
    border: 1.5px solid $line;
    border-radius: 14px;
    background: $card;
    cursor: pointer;
    transition: border-color 0.16s ease, box-shadow 0.16s ease, transform 0.16s ease, background 0.16s ease;

    &:hover { border-color: $ink-3; box-shadow: $lift; transform: translateY(-2px); }
    &--on { border-color: $signal; background: $wash; &:hover { border-color: $signal; } }

    // A dataset with 0 questions can still be picked (so the info icon
    // and warning banner are reachable), but reads visibly muted/flagged
    // so it doesn't look identical to a usable suite at a glance.
    &--empty {
      border-color: rgba($danger, 0.3);

      &:hover { border-color: rgba($danger, 0.5); }
    }
  }

  &__dcard-top { display: flex; align-items: center; justify-content: space-between; gap: 10px; }
  &__dcard-id { display: flex; align-items: center; gap: 11px; min-width: 0; }

  &__dcard-icon {
    flex-shrink: 0;
    width: 36px;
    height: 36px;
    border-radius: 10px;
    display: grid;
    place-items: center;
    background: $paper;
    border: 1px solid $line;
    color: $ink;
    transition: all 0.16s ease;
  }
  &__dcard--on &__dcard-icon { background: $signal; border-color: $signal; color: #fff; }

  &__dcard-name { font-family: $display; font-size: 1.0769em; font-weight: 700; color: $ink; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; } // 0.875rem / 0.8125rem

  &__dcard-warn-icon {
    flex-shrink: 0;
    display: inline-flex;
    color: $danger;
  }

  &__dcard-tags { display: flex; flex-wrap: wrap; align-items: center; gap: 6px; }

  &__tag {
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 600;
    letter-spacing: 0.02em;
    color: $ink-2;
    background: $paper;
    border: 1px solid $line;
    border-radius: 6px;
    padding: 2px 7px;
  }
  &__tag--custom { color: $signal; background: $wash; border-color: rgba($signal, 0.25); }
  &__tag--deepeval { color: $ok; background: $ok-wash; border-color: rgba($ok, 0.25); }
  &__tag--count { border: 0; background: transparent; color: $ink-3; padding-left: 0; }
  &__tag--count-empty { color: $danger; font-weight: 700; }

  // Full-width banner shown below the grid+subgroup-rail layout whenever
  // the currently selected dataset has 0 questions — explains why
  // Continue is disabled without the user having to guess.
  &__dataset-empty-warning {
    display: flex;
    align-items: flex-start;
    gap: 9px;
    margin-top: 14px;
    padding: 11px 14px;
    border: 1px solid rgba($danger, 0.3);
    border-radius: 12px;
    background: $danger-wash;
    color: $danger;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    line-height: 1.5;

    svg { flex-shrink: 0; margin-top: 2px; }
    strong { font-weight: 700; }
  }

  // ---- dcard header actions: preview button + selection mark (Change-2) --
  &__dcard-actions {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 8px;
  }

  &__dcard-preview-btn {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 5px 10px;
    border: 1px solid $line;
    border-radius: 999px;
    background: $paper;
    color: $ink-2;
    font-family: $sans;
    font-size: 0.8846em; // 0.71875rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $signal; color: $signal; background: $wash; }
  }

  // ---- subgroup rail ------------------------------------------------------
  &__rail {
    flex-shrink: 0;
    display: flex;
    flex-direction: column;
    border: 1px solid $line;
    border-radius: 14px;
    background: $paper;
    min-height: 0;
    overflow: hidden;
  }

  &__rail-head { flex-shrink: 0; padding: 15px 16px 13px; border-bottom: 1px solid $line; }
  &__rail-head-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
  }
  &__rail-title {
    display: flex;
    align-items: center;
    gap: 7px;
    font-family: $display;
    font-size: 1em; // 0.8125rem / 0.8125rem (base)
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $ink;
    svg { color: $signal; }
  }
  &__rail-actions {
    flex-shrink: 0;
    display: flex;
    gap: 10px;
  }
  &__rail-sub { margin-top: 4px; font-size: 0.8846em; color: $ink-3; line-height: 1.45; } // 0.71875rem / 0.8125rem

  &__rail-scroll {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding: 10px;
    display: flex;
    flex-direction: column;
    gap: 6px;
  }

  &__rail-empty {
    margin: 10px;
    padding: 18px 12px;
    text-align: center;
    border: 1px dashed $line;
    border-radius: 10px;
    font-size: 0.9231em; color: $ink-3; } // 0.75rem / 0.8125rem

  &__check-row {
    display: flex;
    align-items: center;
    gap: 10px;
    width: 100%;
    text-align: left;
    padding: 9px 11px;
    border: 1px solid $line;
    border-radius: 10px;
    background: $card;
    cursor: pointer;
    transition: border-color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $ink-3; }
    &--on { border-color: $signal; background: $wash; }
  }

  &__check {
    flex-shrink: 0;
    width: 17px;
    height: 17px;
    border-radius: 5px;
    border: 1.5px solid $ink-3;
    background: $card;
    display: grid;
    place-items: center;
    color: transparent;
    transition: all 0.14s ease;

    &--on { background: $signal; border-color: $signal; color: #fff; }
  }

  &__check-label {
    font-size: 1em; // 0.8125rem / 0.8125rem (base)
    font-weight: 600;
    color: $ink;

    // Extra boost beyond the automatic base-font scale-up (see
    // $ev-base-font), since at this width the subgroup cards still read
    // smaller than the surrounding provider/model/dataset cards otherwise.
    @media (min-width: 1800px) {
      font-size: 1.1538em; // 0.9375rem / 0.8125rem
    }
  }

  // ---- upload panel -------------------------------------------------------
  &__upload {
    border: 1.5px solid $line;
    border-radius: 14px;
    background: $paper;
    padding: 20px;
    max-width: 560px;
  }

  &__drop {
    display: flex;
    align-items: center;
    gap: 11px;
    width: 100%;
    border: 1.5px dashed $ink-3;
    border-radius: 12px;
    padding: 16px;
    background: $card;
    color: $ink-3;
    font-size: 1.0385em; // 0.84375rem / 0.8125rem
    font-weight: 500;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $signal; color: $signal; background: $wash; }

    svg { flex-shrink: 0; }
  }
  &__drop-file { color: $ink; font-weight: 600; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
  &__drop--has { border-style: solid; border-color: $signal; color: $ink; }

  &__upload-actions {
    display: flex;
    justify-content: flex-end;
    gap: 10px;
    margin-top: 20px;
  }

  // ========================================================================
  // Metrics step: chips + judge rail
  // ========================================================================
  &__metrics {
    flex: 1;
    min-height: 0;
    display: grid;
    grid-template-columns: 1fr 300px;
    // Without an explicit row height, the implicit single row sizes to
    // its tallest column's content (auto/max-content) instead of being
    // constrained to the space actually available — which is what was
    // letting &__metrics-main's content grow past the bottom of the
    // stage with no scrollbar appearing anywhere. minmax(0, 1fr) forces
    // the row to the container's real height so &__metrics-main's own
    // overflow-y: auto (below) can actually kick in.
    grid-template-rows: minmax(0, 1fr);
    gap: 16px;
  }

  &__metrics-main {
    min-width: 0;
    min-height: 0;
    display: flex;
    flex-direction: column;
    gap: 16px;
    // Independent scrollbar for the cards column — without this, overflow
    // bubbles up to &__stage-body and scrolls the whole step (cards +
    // Judge Model rail) as one unit. With it, only this column scrolls;
    // &__judge keeps its own separate internal scroll region
    // (&__judge-scroll) and its head/footer stay pinned in place.
    overflow-y: auto;
    padding-right: 4px;
    margin-right: -4px;
  }

  // ---- Metrics step: one card per section -----------------------------
  // Same visual language as &__judge (border/radius/paper background,
  // header+body split) so the whole step reads as a consistent stack of
  // cards instead of one long undifferentiated column.
  &__msec {
    border: 1px solid $line;
    border-radius: 14px;
    background: $paper;
  }

  &__msec-head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
    padding: 14px 16px 12px;
    border-bottom: 1px solid $line;
  }

  &__msec-head-text { min-width: 0; }

  &__msec-title {
    display: flex;
    align-items: center;
    gap: 7px;
    font-family: $display;
    font-size: 1.0769em; // 0.875rem / 0.8125rem
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $ink;

    svg { flex-shrink: 0; color: $signal; }
  }

  &__msec-sub { margin-top: 3px; font-size: 0.9231em; color: $ink-3; line-height: 1.4; } // 0.75rem / 0.8125rem

  &__msec-head-action { flex-shrink: 0; }

  &__msec-body { padding: 16px; }

  // ---- Run Samples + Instruction side-by-side (top of Metrics step) -------
  &__samples-instruction-row {
    display: grid;
    grid-template-columns: 1fr 1fr;
    align-items: start;
    gap: 16px;
  }

  // Run Samples card body wraps the standalone runSamplesControl field
  // directly — .ev__field--samples below sets its width; no wrapper class
  // needed now that it isn't laid out side-by-side with a note anymore.
  &__field--samples { max-width: 260px; }
  // Top K (RAG-only, Test Suite step) — a simple number field, doesn't
  // need the wide layout the samples control does.
  // Top K (RAG-only, Test Suite step) — the field row is wide enough for
  // the label's long hint text to stay on one line; the actual number
  // input is kept narrow separately via &__input--topk below.
  &__field--topk {
    max-width: 480px;
    margin-bottom: 20px;

    .opt {
      white-space: nowrap;
      font-size: 0.85em;
    }
  }
  &__input--topk { max-width: 140px; }

  // Shown above the metrics chips when the catalog has entries but nothing
  // is selected yet — mirrors .ev__judge-required's styling/tone.
  &__metrics-required {
    margin-bottom: 12px;
    padding: 9px 11px;
    border: 1px dashed rgba($danger, 0.35);
    border-radius: 10px;
    background: $danger-wash;
    color: $danger;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    line-height: 1.4;
  }

  // ---- Run samples: Custom / Full radio row -------------------------------
  &__radio-row {
    display: inline-flex;
    align-items: center;
    gap: 16px;
  }

  &__radio-opt {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    border: 0;
    background: transparent;
    padding: 0;
    cursor: pointer;
    font-family: $sans;
    font-size: 1em; // 0.8125rem / 0.8125rem (base)
    font-weight: 600;
    color: $ink-2;
    transition: color 0.15s ease;

    &:hover { color: $ink; }
    &--on { color: $ink; }
  }

  &__radio-full-note {
    margin-top: 10px;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    color: $ink-3;
    line-height: 1.4;
  }

  &__metrics-count {
    font-family: $mono;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    color: $ink-2;
    b { color: $signal; font-weight: 700; }
  }

  &__metrics-actions { display: flex; gap: 14px; }

  &__link {
    font-family: $sans;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 600;
    color: $signal;
    background: none;
    border: 0;
    padding: 0;
    cursor: pointer;
    &:hover { text-decoration: underline; }
  }

  &__chips {
    flex: 1;
    margin: 0 -6px;
    padding: 4px 6px;
    align-content: flex-start;
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
  }

  &__chip {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    padding: 9px 14px;
    border: 1.5px solid $line;
    border-radius: 999px;
    background: $card;
    color: $ink-2;
    font-size: 0.95em;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.15s ease;
    height: fit-content;

    &:hover { border-color: $ink-3; color: $ink; }

    &--on {
      border-color: $signal;
      background: $signal;
      color: #fff;
    }

    // Custom metrics (from GET /metrics's `custom` array) get a dashed
    // border at rest so they read as a distinct category from the plain
    // built-in metric chips above, even before the section header makes
    // that explicit.
    &--custom {
      border-style: dashed;

      &.ev__chip--on { border-style: solid; }
    }
  }

  &__chip-tick {
    display: grid;
    place-items: center;
    width: 14px;
    height: 14px;
  }

  // Small gavel glyph on a custom metric chip that needs a judge model to
  // grade it — purely informational, doesn't affect the always-shown/
  // mandatory Judge Model rail itself.
  &__chip-judge-icon {
    flex-shrink: 0;
    opacity: 0.6;
  }
  &__chip--on &__chip-judge-icon { opacity: 0.85; }

  // ---- instruction (Metrics-step card; see &__msec for the header/body
  // wrapper) — generate button sits in the card head as a head-action,
  // textarea in the card body. ------------------------------------------
  &__instruction-generate-btn {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 6px 12px;
    border: 1px solid $signal;
    border-radius: 999px;
    background: $card;
    color: $signal;
    font-family: $sans;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: background 0.15s ease, color 0.15s ease;

    &:hover:not(:disabled) { background: $wash; }
    &:disabled { cursor: not-allowed; opacity: 0.5; }
  }

  &__instruction-textarea {
    width: 100%;
    min-height: 110px;
    border: 1.5px solid $line;
    border-radius: 11px;
    padding: 11px 13px;
    font-family: $sans;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 500;
    line-height: 1.55;
    color: $ink;
    background: $card;
    resize: vertical;
    transition: border-color 0.15s ease, box-shadow 0.15s ease;

    &::placeholder { color: $ink-3; font-weight: 400; }
    &:focus { outline: none; border-color: $signal; box-shadow: $ring; }
  }

  // ---- judge rail ---------------------------------------------------------
  &__judge {
    flex-shrink: 0;
    display: flex;
    flex-direction: column;
    border: 1px solid $line;
    border-radius: 14px;
    background: $paper;
    min-height: 0;
    overflow: hidden;
  }

  &__judge-head { flex-shrink: 0; padding: 15px 16px 13px; border-bottom: 1px solid $line; }

  &__judge-head-row {
    position: relative;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__judge-title {
    display: flex;
    align-items: center;
    gap: 7px;
    font-family: $display;
    font-size: 1.0769em; // 0.875rem / 0.8125rem
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $ink;
    svg { color: $signal; }
  }
  &__judge-sub { margin-top: 4px; font-size: 0.9615em; color: $ink-3; line-height: 1.45; } // 0.78125rem / 0.8125rem

  // Small toggle button + popover replacing the old always-open info card
  // — same explanatory copy, but collapsed by default so it doesn't eat
  // into the rail's limited vertical space.
  &__judge-info-btn {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 24px;
    height: 24px;
    border: 1px solid $line;
    border-radius: 999px;
    background: $paper;
    color: $ink-3;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $signal; color: $signal; }
  }
  &__judge-info-btn--on { border-color: $signal; color: $signal; background: $wash; }

  &__judge-info-popover {
    position: absolute;
    top: calc(100% + 8px);
    right: 0;
    z-index: 5;
    width: 240px;
    padding: 10px 11px;
    border: 1px solid rgba($signal, 0.2);
    border-radius: 10px;
    background: $wash;
    color: $ink-2;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    line-height: 1.45;
    box-shadow: 0 12px 28px -12px rgba(0, 0, 0, 0.25);
    animation: ev-fade-in 0.15s ease both;
  }

  &__judge-search {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 8px;
    margin: 12px 12px 0;
    padding: 8px 11px;
    border: 1px solid $line;
    border-radius: 10px;
    background: $card;

    svg { flex-shrink: 0; color: $ink-3; }

    input {
      flex: 1;
      min-width: 0;
      border: 0;
      background: transparent;
      font-family: $sans;
      font-size: 0.9231em; // 0.75rem / 0.8125rem
      color: $ink;

      &:focus { outline: none; }
      &::placeholder { color: $ink-3; }
    }
  }

  // Shown at the bottom of the judge rail whenever no judge model has been
  // picked yet — mandatory for every evaluation type now, not gated on
  // LLM_Judge or the agent framework anymore.
  &__judge-required {
    flex-shrink: 0;
    margin: 0 10px 10px;
    padding: 9px 11px;
    border: 1px dashed rgba($danger, 0.35);
    border-radius: 10px;
    background: $danger-wash;
    color: $danger;
    font-size: 0.8846em; line-height: 1.4; } // 0.71875rem / 0.8125rem

  &__judge-scroll {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding: 10px;
    display: flex;
    flex-direction: column;
    gap: 6px;
  }

  &__judge-empty {
    margin: 10px;
    padding: 18px 12px;
    text-align: center;
    border: 1px dashed $line;
    border-radius: 10px;
    font-size: 0.9231em; color: $ink-3; } // 0.75rem / 0.8125rem

  &__judge-row {
    display: flex;
    align-items: center;
    gap: 10px;
    width: 100%;
    text-align: left;
    padding: 10px 11px;
    border: 1px solid $line;
    border-radius: 10px;
    background: $card;
    cursor: pointer;
    transition: border-color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $ink-3; }
    &--on { border-color: $signal; background: $wash; }
  }

  &__radio {
    flex-shrink: 0;
    width: 15px;
    height: 15px;
    border-radius: 50%;
    border: 1.5px solid $ink-3;
    background: $card;
    transition: border-width 0.14s ease, border-color 0.14s ease;
    &--on { border-color: $signal; border-width: 5px; }
  }

  &__judge-name {
    font-size: 1.0769em; // 0.875rem / 0.8125rem
    font-weight: 600;
    color: $ink;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;

    // Same extra wide-screen boost as the subgroup cards (.ev__check-label)
    // above — otherwise the judge model list reads smaller than the rest
    // of the metrics step at this width.
    @media (min-width: 1800px) {
      font-size: 1.2308em; // 1rem / 0.8125rem
    }
  }
  &__judge-meta {
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    color: $ink-3;

    @media (min-width: 1800px) {
      font-size: 1.0769em; // 0.875rem / 0.8125rem
    }
  }

  // Input/output price per 1M tokens — same data + formatting (formatPrice)
  // as the Models step's card stats, just laid out as a single line on the
  // right of the row instead of a labeled stat block.
  &__judge-price {
    flex-shrink: 0;
    font-family: $mono;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 700;
    color: $ink-2;

    @media (min-width: 1800px) {
      font-size: 1.0385em; // 0.84375rem / 0.8125rem
    }
  }

  // ========================================================================
  // Review step
  // ========================================================================
  &__summary {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 12px;
  }

  &__summary-cell {
    padding: 16px;
    border: 1px solid $line;
    border-radius: 14px;
    background: $paper;
  }

  &__summary-k {
    display: flex;
    align-items: center;
    gap: 6px;
    @extend %micro;
    font-size: 0.6923em; // 0.5625rem / 0.8125rem
    color: $ink-3;
    margin-bottom: 8px;
    svg { color: $signal; }
  }

  &__summary-v { font-family: $mono; font-size: 1.8462em; font-weight: 700; color: $ink; letter-spacing: -0.02em; line-height: 1; } // 1.5rem / 0.8125rem
  &__summary-v--muted { color: $ink-3; }

  &__block { margin-top: 26px; }

  &__block-title {
    display: flex;
    align-items: center;
    gap: 7px;
    @extend %micro;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    color: $ink-2;
    margin-bottom: 12px;
    svg { color: $signal; }
    b { color: $ink-3; font-weight: 700; }
  }

  &__rows {
    border: 1px solid $line;
    border-radius: 12px;
    overflow: hidden;
  }

  &__row {
    display: flex;
    align-items: baseline;
    justify-content: space-between;
    gap: 16px;
    padding: 12px 15px;
    border-bottom: 1px solid $line-2;
    font-size: 1.0385em; &:last-child { border-bottom: 0; } // 0.84375rem / 0.8125rem

    span:first-child { @extend %micro; font-size: 0.7692em; color: $ink-3; } // 0.625rem / 0.8125rem
    span:last-child { color: $ink; font-weight: 600; text-align: right; }
  }

  &__review-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
    gap: 10px;
  }

  &__review-card {
    display: flex;
    align-items: center;
    gap: 11px;
    padding: 12px 14px;
    border: 1px solid $line;
    border-radius: 12px;
    background: $paper;
  }
  &__review-card-icon {
    flex-shrink: 0;
    width: 32px;
    height: 32px;
    border-radius: 9px;
    display: grid;
    place-items: center;
    background: $card;
    border: 1px solid $line;
    color: $signal;
  }
  &__review-card-name { font-family: $display; font-size: 1em; font-weight: 700; color: $ink; } // 0.8125rem / 0.8125rem (base)
  &__review-card-sub { font-size: 0.8846em; color: $ink-3; margin-top: 1px; } // 0.71875rem / 0.8125rem

  &__metric-tags { display: flex; flex-wrap: wrap; gap: 7px; }
  &__metric-tag {
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 600;
    color: $signal;
    background: $wash;
    border: 1px solid rgba($signal, 0.2);
    border-radius: 8px;
    padding: 5px 11px;
  }

  &__empty {
    padding: 20px;
    text-align: center;
    border: 1px dashed $line;
    border-radius: 12px;
    background: $paper;
    color: $ink-3;
    font-size: 1.0385em; } // 0.84375rem / 0.8125rem

  &__error {
    margin-top: 18px;
    display: flex;
    align-items: center;
    gap: 9px;
    font-size: 1em; // 0.8125rem / 0.8125rem (base)
    font-weight: 500;
    color: $danger;
    background: $danger-wash;
    border: 1px solid rgba($danger, 0.2);
    border-radius: 10px;
    padding: 11px 14px;
  }

  &__spin { animation: ev-spin 0.8s linear infinite; }

  // ========================================================================
  // REDESIGN LAYER
  // Everything below restyles the wizard to the reference screens: numbered
  // stepper, radio/checkbox cards, table + side panel, sectioned form rows,
  // review banner. Only existing tokens are used ($signal, $wash, $ink*,
  // $paper, $card, $line*, $ok, $danger, $sans/$display/$mono). Rules here
  // sit last in the block so they win over the earlier ones they refine.
  // ========================================================================

  // ---- sidebar stepper ----------------------------------------------------
  // Quiet timeline: completed steps recede (soft tick, their value still
  // readable), the current step is the only filled element, upcoming steps
  // are muted. The connector is a short segment between ticks rather than
  // one continuous rule, coloured only where the path has been walked.
  &__manifest-head { padding: 18px 20px 16px; }
  &__manifest-title { margin-top: 8px; }
  &__manifest-done { margin-top: 8px; font-size: 1.0385em; color: $ink-3; }

  &__segs { display: flex; gap: 4px; margin-top: 14px; }
  &__seg {
    flex: 1;
    height: 4px;
    border-radius: 999px;
    background: $line;
    transition: background 0.2s ease;

    &--done { background: $signal; }
    &--active { background: rgba($signal, 0.4); }
  }

  // shell: tighter gutter; the sidebar (aside) and stage (section) keep
  // their original 16px radius (set on &__manifest / &__stage above)
  &__shell { gap: 6px; }

  &__spec { gap: 4px; padding: 12px 10px 14px; }

  &__spec-row {
    grid-template-columns: 28px minmax(0, 1fr) auto;
    align-items: center;
    gap: 12px;
    padding: 10px 12px 10px 10px;
    border-radius: 10px;

    // segment from this tick to the next one, with a 3px breath at each end
    // (rows share one height, so the percentages line up)
    &::before {
      left: 23px;
      top: calc(50% + 17px);
      bottom: calc(-50% + 13px);
      width: 2px;
      border-radius: 2px;
      background: $line;
    }
  }

  &__spec-tick {
    width: 28px;
    height: 28px;
    border-radius: 50%;
    border: 1.5px solid $line;
    background: $card;
    color: $ink-3;
    font-size: 1em;
  }

  &__spec-label {
    font-family: $sans;
    font-size: 1.0769em;
    font-weight: 600;
    letter-spacing: 0;
    text-transform: none;
    color: $ink-3;
  }

  &__spec-value {
    font-size: 1.2692em;
    font-weight: 650;

    &[data-empty='true'] { font-family: $sans; font-weight: 500; color: $ink-3; }
  }

  &__spec-row--done {
    &::before { background: rgba($signal, 0.45); }
    .ev__spec-tick { background: $wash; border-color: transparent; color: $signal; }
    .ev__spec-label { color: $ink-3; }
    &:not(:disabled):hover { background: $paper; }
  }

  &__spec-row--active {
    background: $wash;
    box-shadow: none;
    .ev__spec-tick {
      background: $signal;
      border-color: $signal;
      color: #fff;
      box-shadow: 0 0 0 4px rgba($signal, 0.16);
    }
    .ev__spec-label { color: $signal; }
    &:not(:disabled):hover { background: $wash; }
  }

  &__spec-row--todo { opacity: 1; }

  // "Edit" pill, revealed on hover / keyboard focus of a completed step
  &__spec-edit {
    align-self: center;
    padding: 3px 9px;
    border: 1px solid rgba($signal, 0.25);
    border-radius: 999px;
    background: $card;
    font-family: $sans;
    font-size: 0.7692em;
    font-weight: 650;
    color: $signal;
    opacity: 0;
    transition: opacity 0.15s ease;
  }
  &__spec-row:hover &__spec-edit,
  &__spec-row:focus-visible &__spec-edit { opacity: 1; }

  &__manifest-foot {
    flex-shrink: 0;
    padding: 12px 20px;
    border-top: 1px solid $line-2;
    font-size: 0.9615em;
    color: $ink-3;
    line-height: 1.4;
  }

  // ---- stage head / footer ------------------------------------------------
  &__stage-step {
    font-family: $sans;
    font-size: 0.9231em;
    font-weight: 700;
    color: $signal;
  }
  &__stage-title { margin-top: 4px; }

  &__blocker { font-size: 1em; color: $ink-2; }

  // ---- question label -----------------------------------------------------
  &__q {
    font-family: $sans;
    font-size: 1.0769em;
    font-weight: 700;
    color: $ink;
    margin-bottom: 10px;

    span { font-weight: 500; color: $ink-2; }
  }

  // ---- radio / checkbox cards --------------------------------------------
  &__choice-row { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 12px; }
  &__choice-col { display: flex; flex-direction: column; gap: 10px; }
  &__choice-grid { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 10px; }

  &__choice {
    display: flex;
    flex-direction: column;
    gap: 8px;
    width: 100%;
    text-align: left;
    padding: 16px;
    border: 1.5px solid $line;
    border-radius: 14px;
    background: $card;
    font-family: $sans;
    cursor: pointer;
    transition: border-color 0.16s ease, background 0.16s ease;

    &:hover { border-color: $ink-3; }
    &:disabled { opacity: 0.55; cursor: not-allowed; }

    &--on {
      border-color: $signal;
      background: $wash;
      &:hover { border-color: $signal; }
      .ev__pilltag { background: $card; }
    }

    &--row {
      flex-direction: row;
      align-items: flex-start;
      gap: 12px;
      padding: 13px 16px;

      > .ev__radio, > .ev__check { margin-top: 3px; }
    }
  }

  &__choice-head { display: flex; align-items: center; gap: 10px; }
  &__choice-body { flex: 1; min-width: 0; display: flex; flex-direction: column; gap: 3px; }
  &__choice-name {
    font-family: $display;
    font-size: 1.1538em;
    font-weight: 700;
    color: $ink;
  }
  &__choice-desc { font-size: 1em; color: $ink-2; line-height: 1.45; }
  &__choice-tags { display: flex; flex-wrap: wrap; gap: 6px; margin-top: 6px; }
  &__choice-price {
    flex-shrink: 0;
    align-self: center;
    font-family: $mono;
    font-size: 0.9231em;
    font-weight: 600;
    color: $ink-2;
    white-space: nowrap;
  }

  &__pilltag {
    font-family: $sans;
    font-size: 0.8462em;
    font-weight: 650;
    color: $signal;
    background: $wash;
    border-radius: 6px;
    padding: 2px 8px;
    white-space: nowrap;

    &--muted { color: $ink-2; background: $paper; }
  }

  // "Because you chose …" follow-up block
  &__followup {
    margin-top: 20px;
    padding: 18px 20px 20px;
    border-radius: 14px;
    background: $paper;

    .ev__q { margin-top: 16px; }
  }
  &__followup-eyebrow { font-size: 0.9231em; font-weight: 650; color: $ink-2; }

  // checkbox used as a real <button> (models table, "save traces")
  &__check {
    padding: 0;
    &:disabled { opacity: 0.45; cursor: not-allowed; }
  }
  button#{&}__check { cursor: pointer; }

  // ---- main + side panel split (Models step) ------------------------------
  &__split {
    flex: 1;
    min-height: 0;
    display: grid;
    grid-template-columns: minmax(0, 1fr) 390px;
    grid-template-rows: minmax(0, 1fr);
    gap: 20px;
  }
  &__split-main { min-width: 0; min-height: 0; display: flex; flex-direction: column; }
  // model cards inside the split: slightly narrower minimum so two or more
  // fit beside the 390px panel, and every card is the same size — equal
  // row heights, cards stretched to fill, stats pinned to the bottom edge
  &__split &__models-scroll { margin-top: 0; }
  &__split &__grid--wide {
    grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
    grid-auto-rows: 1fr;
    align-items: stretch;
  }
  &__split &__mcard { height: 100%; }
  &__split &__mcard-stats { margin-top: auto; }

  &__step-toolbar { flex-wrap: wrap; }
  &__viewchips { flex: 0 1 auto; display: flex; flex-wrap: wrap; gap: 8px; }
  &__split &__toolbar-search { flex: 1 1 200px; }
  &__split &__toolbar-refresh { margin-left: auto; }

  // Side panel: square corners, bordered on all sides, runs the full height
  // of the step.
  &__panel {
    align-self: stretch;
    min-height: 0;
    overflow-y: auto;
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 18px;
    border: 1px solid $line;
    border-radius: 0;
    background: $paper;
  }
  // Pinned variant (Models step): the top block stays put, the rest scrolls.
  &__panel--pinned { overflow: hidden; padding: 0; gap: 0; }
  &__panel-fixed {
    flex-shrink: 0;
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 18px;
    border-bottom: 1px solid $line;
    background: $paper;
  }
  &__panel-scroll {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 18px;
  }
  // ---- per-model item ("In this run") ------------------------------------
  // Header: health dot, model name + provider, remove. In "Per model" mode a
  // tinted body underneath holds that model's two steppers, so each model
  // reads as one self-contained unit.
  &__pm {
    border: 1px solid $line;
    border-radius: 10px;
    background: $card;
    overflow: hidden;
  }
  &__pm-head { display: flex; align-items: center; gap: 10px; padding: 10px 12px; }
  &__pm-dot {
    flex-shrink: 0;
    width: 8px;
    height: 8px;
    border-radius: 50%;
    background: $ink-3;

    &--ok { background: $ok; }
    &--fail { background: $danger; }
  }
  &__pm-name {
    flex: 1;
    min-width: 0;
    font-size: 1em;
    font-weight: 700;
    color: $ink;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;

    small { display: block; font-size: 0.8846em; font-weight: 500; color: $ink-2; overflow: hidden; text-overflow: ellipsis; }
  }
  &__pm-body { padding: 10px 12px 12px; border-top: 1px solid $line-2; background: $paper; }

  // ---- stepper fields (retries / timeout) --------------------------------
  &__stepfields { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 10px; }
  &__stepfield { min-width: 0; display: flex; flex-direction: column; gap: 6px; }
  &__stepfield-label {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    font-size: 0.8846em;
    font-weight: 650;
    color: $ink-2;

    svg { color: $signal; }
  }

  &__stepper {
    display: flex;
    align-items: stretch;
    height: 34px;
    border: 1.5px solid $line;
    border-radius: 9px;
    background: $card;
    overflow: hidden;
    transition: border-color 0.15s ease, box-shadow 0.15s ease;

    &:focus-within { border-color: $signal; box-shadow: $ring; }

    button {
      flex: 0 0 30px;
      display: grid;
      place-items: center;
      border: 0;
      background: $paper;
      color: $ink-2;
      cursor: pointer;
      transition: background 0.12s ease, color 0.12s ease;

      &:hover:not(:disabled) { background: $wash; color: $signal; }
      &:disabled { opacity: 0.35; cursor: not-allowed; }
    }

    input {
      flex: 1;
      min-width: 0;
      width: 100%;
      border: 0;
      border-left: 1px solid $line-2;
      border-right: 1px solid $line-2;
      background: transparent;
      text-align: center;
      font-family: $mono;
      font-size: 1em;
      font-weight: 700;
      color: $ink;
      -moz-appearance: textfield;
      appearance: textfield;

      &:focus { outline: none; }
      &::-webkit-outer-spin-button,
      &::-webkit-inner-spin-button { -webkit-appearance: none; margin: 0; }
    }
  }

  &__panel-title { font-family: $display; font-size: 1.0769em; font-weight: 800; color: $ink; }
  &__panel-sub { margin-top: -6px; font-size: 0.9231em; color: $ink-2; }
  &__panel-list { display: flex; flex-direction: column; gap: 8px; }
  &__panel-item {
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 10px 12px;
    border: 1px solid $line;
    border-radius: 10px;
    background: $card;
    font-size: 1em;
    font-weight: 600;
    color: $ink;

    small { display: block; font-size: 0.9231em; font-weight: 500; color: $ink-2; }

    &--off { background: transparent; border-style: dashed; font-weight: 500; color: $ink-2; }
  }
  &__panel-item-row { display: flex; align-items: center; justify-content: space-between; gap: 10px; }
  &__panel-cov { display: flex; align-items: center; gap: 10px; }
  &__panel-x {
    flex-shrink: 0;
    display: inline-flex;
    padding: 2px;
    border: 0;
    background: transparent;
    color: $ink-2;
    cursor: pointer;
    &:hover { color: $ink; }
  }
  &__panel-divider { height: 1px; margin: 6px 0; background: $line; }
  &__panel-stats {
    display: flex;
    flex-direction: column;
    gap: 6px;
    margin-top: 4px;
    padding-top: 12px;
    border-top: 1px solid $line;

    div { display: flex; justify-content: space-between; gap: 12px; font-size: 1em; color: $ink-2; }
    b { font-family: $mono; font-weight: 700; color: $ink; }
  }
  &__panel-note { font-size: 0.9231em; color: $ink-2; line-height: 1.45; }

  &__dot {
    flex-shrink: 0;
    width: 16px;
    height: 16px;
    border-radius: 50%;
    border: 1.5px solid $ink-3;
    display: grid;
    place-items: center;
    color: #fff;

    &--on { background: $signal; border-color: $signal; }
  }

  // ---- Type step: "About" panel -------------------------------------------
  // Sits under the three type cards and describes the selected one: what it
  // measures, what it's good for, and how a run of that type works.
  &__about {
    // the stage body is a flex column; without this the panel (overflow:
    // hidden) would be squeezed to nothing when the step's content is tall
    flex-shrink: 0;
    margin-top: 16px;
    border: 1px solid $line;
    border-radius: 14px;
    background: $paper;
    overflow: hidden;
  }
  &__about-head {
    display: flex;
    align-items: center;
    gap: 14px;
    padding: 16px 20px;
    border-bottom: 1px solid $line;
    background: $card;

    .ev__option-icon { width: 40px; height: 40px; border-radius: 11px; }
    h3 { font-family: $display; font-size: 1.2308em; font-weight: 800; letter-spacing: -0.01em; color: $ink; }
    p { margin-top: 2px; font-size: 1.0385em; color: $ink-2; }
  }
  &__about-cols { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); }
  &__about-col {
    padding: 18px 20px 20px;
    & + & { border-left: 1px solid $line; }
  }
  &__about-k { @extend %micro; font-size: 0.7692em; color: $ink-3; margin-bottom: 12px; }

  &__about-list,
  &__about-flow {
    display: flex;
    flex-direction: column;
    gap: 10px;
    list-style: none;
    font-size: 1.0385em;
    color: $ink;
    line-height: 1.45;

    li { display: grid; grid-template-columns: 20px minmax(0, 1fr); align-items: start; gap: 8px; }
    svg { margin-top: 3px; color: $signal; }
  }
  &__about-flow span {
    width: 20px;
    height: 20px;
    border-radius: 50%;
    display: grid;
    place-items: center;
    background: $signal;
    color: #fff;
    font-family: $mono;
    font-size: 0.7692em;
    font-weight: 700;
  }
  // below the agent options it needs a little more room above
  &__section + &__about { margin-top: 26px; margin-bottom: 22px; }

  &__about-empty {
    flex-shrink: 0;
    margin-top: 16px;
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 16px 18px;
    border: 1px dashed $line;
    border-radius: 12px;
    background: $paper;
    font-size: 1em;
    color: $ink-2;

    svg { flex-shrink: 0; color: $signal; }
  }

  // ---- Name step ----------------------------------------------------------
  // Two columns: the form on the left, a live "Preview in History" card on
  // the right.
  &__namegrid {
    display: grid;
    grid-template-columns: minmax(0, 620px) minmax(260px, 340px);
    justify-content: space-between;
    align-items: start;
    gap: 40px;

    .ev__quick { margin-top: 24px; }
    .ev__quick + .ev__quick { margin-top: 18px; }
  }
  &__namegrid-main { min-width: 0; }

  &__name-wrap {
    position: relative;

    .ev__name-input { padding-right: 34px; }
  }
  &__name-clear {
    position: absolute;
    right: 2px;
    top: 50%;
    transform: translateY(-60%);
    width: 26px;
    height: 26px;
    display: grid;
    place-items: center;
    border: 0;
    border-radius: 50%;
    background: $paper;
    color: $ink-2;
    cursor: pointer;
    transition: background 0.12s ease, color 0.12s ease;

    &:hover { background: $line; color: $ink; }
  }
  &__name-caption--ok { color: $ok; font-weight: 600; }

  &__hprev {
    position: sticky;
    top: 0;
    padding: 18px;
    border: 1px solid $line;
    border-radius: 14px;
    background: $paper;
  }
  &__hprev-head { @extend %micro; font-size: 0.7692em; color: $ink-3; margin-bottom: 12px; }

  &__hprev-card {
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 14px;
    border: 1.5px solid $line;
    border-radius: 12px;
    background: $card;
    box-shadow: $soft;
    transition: border-color 0.2s ease, box-shadow 0.2s ease;

    &--live { border-color: $signal; box-shadow: 0 10px 22px -14px rgba($signal, 0.6); }
  }
  &__hprev-top { display: flex; align-items: center; gap: 10px; }
  &__hprev-icon {
    flex-shrink: 0;
    width: 32px;
    height: 32px;
    border-radius: 9px;
    display: grid;
    place-items: center;
    background: $wash;
    color: $signal;
  }
  &__hprev-name {
    min-width: 0;
    font-family: $display;
    font-size: 1.1538em;
    font-weight: 700;
    color: $ink;
    overflow-wrap: anywhere;

    &[data-empty='true'] { color: $ink-3; font-weight: 600; }
  }
  &__hprev-badges { display: flex; flex-wrap: wrap; align-items: center; gap: 6px; }
  &__hprev-status {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 2px 8px;
    border-radius: 999px;
    background: $wash;
    font-family: $mono;
    font-size: 0.7692em;
    font-weight: 700;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $signal;

    &::before { content: ''; width: 5px; height: 5px; border-radius: 50%; background: $signal; }
  }
  &__hprev-meta {
    display: flex;
    justify-content: space-between;
    gap: 10px;
    padding-top: 10px;
    border-top: 1px solid $line-2;
    font-size: 0.9231em;
    color: $ink-2;
  }
  &__hprev-note { margin-top: 12px; font-size: 0.9231em; color: $ink-3; line-height: 1.45; }

  // ---- list header (Providers / Models / Test Suite) ---------------------
  // Title + live count on the left, search and refresh on the right, filter
  // chips on a second row, and a rule underneath separating it from the list.
  &__lhead {
    flex-shrink: 0;
    display: flex;
    flex-direction: column;
    gap: 12px;
    margin-bottom: 16px;
    padding-bottom: 14px;
    border-bottom: 1px solid $line;

    // inside a scrolling list (Test Suite) the header stays in view
    &--sticky {
      position: sticky;
      top: -2px;
      z-index: 2;
      padding-top: 2px;
      background: $card;
      // soft edge so the rule still reads when cards scroll underneath
      box-shadow: 0 8px 10px -10px rgba(20, 22, 27, 0.28);
    }


  }

  &__lhead-row { display: flex; align-items: center; gap: 16px; }

  &__lhead-title {
    flex: 1;
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 2px;

    h3 { font-family: $display; font-size: 1.1538em; font-weight: 800; letter-spacing: -0.01em; color: $ink; }
    span { font-size: 0.9231em; color: $ink-2; }
    b { font-weight: 700; color: $signal; }
  }

  // ---- search bar ---------------------------------------------------------
  // A compact "command bar" on the right of the list title: gradient icon
  // tile, field, live result count, "/" hint or clear button, then refresh
  // behind a divider. Resting
  // it's a white bar with a soft shadow; on focus the border becomes a
  // signal gradient, the bar lifts and a glow spreads under it.
  &__sbar {
    position: relative;
    display: flex;
    align-items: center;
    flex: 0 1 340px;
    min-width: 220px;
    gap: 9px;
    height: 38px;
    padding: 0 5px 0 5px;
    border: 1.5px solid transparent;
    border-radius: 11px;
    // two layers: the fill (padding-box) and the border colour (border-box),
    // so the border can be a gradient on focus
    background:
      linear-gradient($card, $card) padding-box,
      linear-gradient($line, $line) border-box;
    box-shadow: 0 1px 2px rgba(20, 22, 27, 0.05), 0 6px 16px -10px rgba(20, 22, 27, 0.18);
    cursor: text;
    transition: box-shadow 0.2s ease, transform 0.2s ease;

    &:hover {
      background:
        linear-gradient($card, $card) padding-box,
        linear-gradient($ink-3, $ink-3) border-box;
    }

    &:focus-within,
    &--active {
      background:
        linear-gradient($card, $card) padding-box,
        linear-gradient(120deg, $signal, $signal-2 55%, rgba($signal, 0.35)) border-box;
    }

    &:focus-within {
      transform: translateY(-1px);
      box-shadow: 0 0 0 3px rgba($signal, 0.12), 0 10px 20px -12px rgba($signal, 0.55);
    }

    input {
      flex: 1;
      min-width: 0;
      height: 100%;
      border: 0;
      background: transparent;
      font-family: $sans;
      font-size: 1em;
      font-weight: 500;
      color: $ink;
      outline: none;

      &::placeholder { color: $ink-3; font-weight: 400; }
    }
  }

  &__sbar-icon {
    flex-shrink: 0;
    width: 26px;
    height: 26px;
    border-radius: 8px;
    display: grid;
    place-items: center;
    background: linear-gradient(140deg, $signal, $signal-2);
    color: #fff;
    box-shadow: 0 4px 8px -4px rgba($signal, 0.7);
    transition: transform 0.2s ease;
  }
  &__sbar:focus-within &__sbar-icon { transform: scale(1.06) rotate(-6deg); }

  // live result count, shown while there's a query
  &__sbar-count {
    flex-shrink: 0;
    padding: 2px 8px;
    border-radius: 999px;
    background: $wash;
    font-family: $sans;
    font-size: 0.8077em;
    font-weight: 700;
    color: $signal;
    white-space: nowrap;
    animation: ev-rise 0.2s ease both;

    &--none { background: $danger-wash; color: $danger; }
  }

  &__sbar-kbd {
    flex-shrink: 0;
    min-width: 20px;
    padding: 1px 5px;
    border: 1px solid $line;
    border-bottom-width: 2px;
    border-radius: 6px;
    background: $paper;
    text-align: center;
    font-family: $mono;
    font-size: 0.7692em;
    font-weight: 700;
    color: $ink-3;
  }
  &__sbar:focus-within &__sbar-kbd { display: none; }

  &__sbar-clear {
    flex-shrink: 0;
    width: 20px;
    height: 20px;
    display: grid;
    place-items: center;
    border: 0;
    border-radius: 50%;
    background: $line;
    color: $ink-2;
    cursor: pointer;
    transition: background 0.12s ease, color 0.12s ease;

    &:hover { background: $ink-2; color: #fff; }
  }

  &__sbar-sep { flex-shrink: 0; width: 1px; height: 18px; background: $line; }

  &__sbar-refresh {
    flex-shrink: 0;
    width: 28px;
    height: 28px;
    display: grid;
    place-items: center;
    border: 0;
    border-radius: 8px;
    background: transparent;
    color: $ink-2;
    cursor: pointer;
    transition: background 0.15s ease, color 0.15s ease;

    &:hover:not(:disabled) { background: $wash; color: $signal; }
    &:disabled { cursor: default; color: $signal; }
  }

  &__lhead-filters { display: flex; flex-wrap: wrap; align-items: center; gap: 8px; }

  // ---- provider cards (Providers step) -----------------------------------
  // Same card as the Type step's options (&__option: icon tile on the left,
  // name + description, selection mark on the right), with the provider's
  // initials in the tile, a "Connected" status beside the name and the model
  // count as a last line.
  &__grid--providers {
    grid-template-columns: repeat(auto-fill, minmax(340px, 1fr));
    grid-auto-rows: 1fr;
    gap: 10px;
  }

  &__option--provider {
    height: 100%;

    .ev__option-icon {
      font-family: $display;
      font-size: 1.3846em;
      font-weight: 800;
      letter-spacing: -0.02em;
    }
    .ev__option-name { flex-wrap: wrap; gap: 4px 10px; }
    .ev__option-desc {
      display: -webkit-box;
      -webkit-box-orient: vertical;
      -webkit-line-clamp: 3;
      line-clamp: 3;
      overflow: hidden;
    }
  }
  &__option--provider.ev__option--on .ev__option-icon { background: $signal; }

  &__pv-status {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-family: $sans;
    font-size: 0.6944em; // back to ~0.96em of the base inside the larger name
    font-weight: 650;
    color: $ok;

    &::before { content: ''; width: 6px; height: 6px; border-radius: 50%; background: $ok; }
  }

  &__option-meta {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    margin-top: 4px;
    font-size: 1em;
    color: $ink-2;

    svg { color: $ink-3; }
    b { font-family: $mono; font-weight: 700; color: $ink; }
  }

  // ---- segmented control --------------------------------------------------
  &__segctl {
    display: inline-flex;
    gap: 2px;
    padding: 3px;
    border: 1px solid $line;
    border-radius: 10px;
    background: $paper;

    &--full { display: flex; background: $card; .ev__segbtn { flex: 1; } }
  }
  &__segbtn {
    border: 0;
    border-radius: 8px;
    padding: 8px 14px;
    background: transparent;
    font-family: $sans;
    font-size: 1em;
    font-weight: 650;
    color: $ink-2;
    cursor: pointer;
    transition: background 0.15s ease, color 0.15s ease;

    &:hover { color: $ink; }
    &--on { background: $signal; color: #fff; &:hover { color: #fff; } }
  }

  // ---- data table ---------------------------------------------------------
  &__table-wrap {
    flex: 0 1 auto;
    min-height: 0;
    overflow: auto;
    border: 1px solid $line;
    border-radius: 0;

    &--static { flex: none; overflow: visible; }
  }

  &__table {
    width: 100%;
    border-collapse: collapse;
    font-size: 1em;

    th {
      position: sticky;
      top: 0;
      z-index: 1;
      padding: 10px 14px;
      background: $paper;
      border-bottom: 1px solid $line;
      text-align: left;
      font-family: $sans;
      font-size: 0.9231em;
      font-weight: 650;
      color: $ink-2;
      white-space: nowrap;
    }

    td {
      padding: 11px 14px;
      border-bottom: 1px solid $line-2;
      color: $ink;
      vertical-align: middle;
    }
    tbody tr:last-child td { border-bottom: 0; }
    tbody tr { cursor: pointer; transition: background 0.12s ease; }
    tbody tr:hover td { background: $paper; }

    th.ev__table-num,
    td.ev__table-num { text-align: right; white-space: nowrap; }
    td.ev__table-num { font-family: $mono; font-size: 0.9615em; }

    tbody tr.ev__table-row--on td { background: $wash; }
    tbody tr.ev__table-row--locked { cursor: default; }

    // Health check in flight: a soft signal sheen sweeps across the whole
    // row and a bar pulses on its left edge. The sheen lives on the <tr>
    // (cells go transparent) so it travels as one band, not per cell.
    tbody tr.ev__table-row--checking {
      background-image: linear-gradient(100deg, transparent 30%, rgba($signal, 0.1) 50%, transparent 70%);
      background-size: 200% 100%;
      background-repeat: no-repeat;
      animation: ev-mcard-sheen 1.6s ease-in-out infinite;

      td { background: transparent; }
      td:first-child { animation: ev-row-bar 1.6s ease-in-out infinite; }
      &:hover td { background: transparent; }
    }
    tbody tr.ev__table-row--checking.ev__table-row--on { background-color: $wash; }

    &--static {
      th { position: static; }
      tbody tr { cursor: default; }
      tbody tr:hover td { background: transparent; }
      td.ev__table-num { font-family: $sans; font-size: 1em; }
    }
  }

  &__table-model {
    display: flex;
    align-items: center;
    gap: 12px;

    b { display: block; font-family: $display; font-weight: 700; color: $ink; }
    small { display: block; font-size: 0.9231em; color: $ink-2; }
  }
  &__table-health { display: flex; align-items: center; gap: 8px; color: $ink-2; white-space: nowrap; }

  // ---- suite rows (Test Suite step) --------------------------------------
  // Suite cards: as many per row as fit, all the same height. Each card
  // shows everything the dataset carries — name, tags, description, how it's
  // scored (harness), its subgroups, and question / subgroup counts.
  &__slist {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
    grid-auto-rows: 1fr;
    gap: 12px;
  }

  &__srow {
    display: flex;
    flex-direction: column;
    gap: 10px;
    height: 100%;
    min-height: 236px;
    padding: 16px;
    border: 1.5px solid $line;
    border-radius: 14px;
    background: $card;
    cursor: pointer;
    transition: border-color 0.16s ease, box-shadow 0.16s ease, transform 0.16s ease, background 0.16s ease;

    &:hover { border-color: $ink-3; box-shadow: $lift; transform: translateY(-2px); }

    &--on {
      border-color: $signal;
      background: $wash;
      &:hover { border-color: $signal; }
      .ev__pilltag { background: $card; }
      .ev__srow-icon { background: $signal; border-color: $signal; color: #fff; }
      .ev__srow-side { border-top-color: rgba($signal, 0.18); }
    }
    &--empty { border-color: rgba($danger, 0.3); }
  }

  &__srow-head { display: flex; align-items: center; gap: 10px; }
  &__srow-icon {
    flex-shrink: 0;
    width: 32px;
    height: 32px;
    border-radius: 9px;
    display: grid;
    place-items: center;
    border: 1px solid $line;
    background: $paper;
    color: $ink;
    transition: background 0.16s ease, border-color 0.16s ease, color 0.16s ease;
  }
  &__srow-name {
    min-width: 0;
    font-family: $display;
    font-size: 1.2308em;
    font-weight: 700;
    color: $ink;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__srow-tags { display: flex; flex-wrap: wrap; gap: 6px; }
  &__srow-tags .ev__pilltag { display: inline-flex; align-items: center; gap: 4px; }

  &__srow-desc {
    font-size: 1.0385em;
    color: $ink-2;
    line-height: 1.5;
    display: -webkit-box;
    -webkit-box-orient: vertical;
    -webkit-line-clamp: 3;
    line-clamp: 3;
    overflow: hidden;

    &--none { color: $ink-3; font-style: italic; }
  }
  &__srow-scoring {
    font-size: 0.9615em;
    color: $ink-2;
    line-height: 1.45;
    b { font-weight: 700; color: $ink; }
  }

  &__srow-subs { display: flex; flex-direction: column; gap: 6px; }
  &__srow-k { @extend %micro; font-size: 0.6923em; color: $ink-3; }
  &__srow-chips { display: flex; flex-wrap: wrap; gap: 5px; }

  // Model cards (Models step) use this same card at the same size (they
  // inherit &__srow's minimum height; the grid below matches the suite
  // grid's column width). Locked = can't be selected until its health check
  // passes.
  &__srow--model { gap: 12px; }
  &__srow--locked {
    cursor: default;
    &:hover { border-color: $line; box-shadow: none; transform: none; }
  }
  // health badge sits at the end of the tags row
  &__srow-health { margin-left: auto; display: inline-flex; }
  &__srow-none { font-size: 0.9231em; color: $ink-3; }

  // footer: stats on the left, Preview pushed to the right
  &__srow-side {
    margin-top: auto;
    display: flex;
    align-items: flex-end;
    gap: 18px;
    padding-top: 12px;
    border-top: 1px solid $line-2;

    .ev__stat-v { font-size: 1.1538em; }
  }
  &__srow-side .ev__stat-v.ev__srow-count--empty { color: $danger; }
  &__srow-preview {
    margin-left: auto;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 6px 12px;
    border: 1px solid $line;
    border-radius: 999px;
    background: $card;
    font-family: $sans;
    font-size: 0.9231em;
    font-weight: 650;
    color: $ink-2;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $signal; color: $signal; background: $wash; }
  }

  // the Subgroups rail matches the side panel: square, bordered on all sides
  &__rail { border: 1px solid $line; border-radius: 0; }

  // ---- sectioned form rows (Scoring step) --------------------------------
  &__form { display: flex; flex-direction: column; }

  // Scoring step: the form is the main column of an &__split (judge panel
  // on the right), so it scrolls on its own.
  &__form--scroll { min-width: 0; min-height: 0; overflow-y: auto; padding-right: 8px; }
  &__split &__frow { grid-template-columns: 180px minmax(0, 1fr); gap: 22px; }
  &__form &__toolbar-refresh { margin-left: 0; }
  &__panel &__toolbar-search { flex: none; max-width: none; }
  &__panel-req { font-size: 0.9231em; font-weight: 700; color: $danger; }

  &__frow {
    display: grid;
    grid-template-columns: 200px minmax(0, 1fr);
    gap: 28px;
    padding: 22px 0;
    border-bottom: 1px solid $line-2;

    &:first-child { padding-top: 0; }
    &:last-child { border-bottom: 0; padding-bottom: 0; }
  }

  &__frow-side {
    h3 { font-family: $display; font-size: 1.1538em; font-weight: 800; color: $ink; }
    p { margin-top: 4px; font-size: 1em; color: $ink-2; line-height: 1.45; }
    .ev__metrics-actions { margin-top: 10px; }
    .ev__frow-req { margin-top: 8px; font-weight: 700; color: $danger; }
    .ev__link:disabled { opacity: 0.5; cursor: not-allowed; text-decoration: none; }
  }

  &__frow-main { min-width: 0; display: flex; flex-direction: column; gap: 12px; }

  &__inline { display: flex; flex-wrap: wrap; align-items: center; gap: 12px; }
  &__inline-note { font-size: 1em; color: $ink-2; line-height: 1.45; }

  &__num {
    width: 88px;
    border: 1.5px solid $line;
    border-radius: 10px;
    padding: 9px 12px;
    font-family: $sans;
    font-size: 1.0769em;
    font-weight: 650;
    color: $ink;
    background: $card;
    transition: border-color 0.15s ease, box-shadow 0.15s ease;

    &:focus { outline: none; border-color: $signal; box-shadow: $ring; }
  }
  &__numfield {
    flex: 1;
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 6px;

    label { font-size: 0.9231em; font-weight: 650; color: $ink; }
    .ev__num { width: 100%; }
  }
  &__inline &__numfield { flex: 0 0 120px; }

  &__formula {
    padding: 10px 12px;
    border-radius: 10px;
    background: $wash;
    font-size: 1em;
    color: $ink;
    b { font-weight: 800; }
  }

  &__sublabel {
    display: flex;
    align-items: center;
    gap: 14px;
    font-size: 0.9231em;
    font-weight: 650;
    color: $ink-2;
  }

  &__chips { flex: none; margin: 0; padding: 0; }

  &__jgrid { display: grid; grid-template-columns: repeat(auto-fill, minmax(270px, 1fr)); gap: 8px; }

  &__checkline {
    display: inline-flex;
    align-items: center;
    gap: 9px;
    padding: 0 0 10px;
    border: 0;
    background: transparent;
    font-family: $sans;
    font-size: 1em;
    font-weight: 650;
    color: $ink;
    cursor: pointer;
  }

  // ---- review -------------------------------------------------------------
  &__banner {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 24px;
    padding: 18px 20px;
    border-radius: 14px;
    background: $wash;
  }
  &__banner-title { font-family: $display; font-size: 1.3846em; font-weight: 800; color: $ink; overflow-wrap: anywhere; }
  &__banner-sub { margin-top: 4px; font-size: 1em; color: $ink-2; }
  &__banner-stats {
    flex-shrink: 0;
    display: flex;
    gap: 28px;

    div { display: flex; flex-direction: column; gap: 2px; }
    span { font-size: 0.9231em; color: $ink-2; }
    b { font-family: $display; font-size: 1.6923em; font-weight: 800; line-height: 1.1; color: $ink; }
  }

  &__sechead {
    display: flex;
    align-items: baseline;
    justify-content: space-between;
    gap: 16px;
    margin: 24px 0 10px;

    h3 { font-family: $display; font-size: 1.1538em; font-weight: 800; color: $ink; }
    span { font-size: 1em; color: $ink-2; text-align: right; }
  }
  &__secnote { margin-top: 10px; font-size: 1em; color: $ink-2; }

  &__settings { border: 1px solid $line; border-radius: 12px; overflow: hidden; }
  &__settings-row {
    display: grid;
    grid-template-columns: 140px minmax(0, 1fr) auto;
    align-items: baseline;
    gap: 16px;
    padding: 14px 16px;
    border-bottom: 1px solid $line-2;

    &:last-child { border-bottom: 0; }
  }
  &__settings-k { font-size: 1em; color: $ink-2; }
  &__settings-v { font-size: 1em; font-weight: 700; color: $ink; line-height: 1.45; overflow-wrap: anywhere; }
}

// ===========================================================================
// Dataset preview slider (Change-2) — right-to-left panel opened from a
// Test Suite card's "Preview" button. Kept as top-level (non-&__) classes
// since it's an overlay outside the .ev tree, same pattern as .ev-toast.
// ===========================================================================
.ev-preview-overlay {
  // master scale control for this subtree — cascades to .ev-preview-panel
  // and its descendants below. See $ev-base-font above.
  font-size: $ev-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  position: fixed;
  inset: 0;
  z-index: 70;
  background: rgba(10, 11, 15, 0.38);
  display: flex;
  justify-content: flex-end;
  animation: ev-fade-in 0.2s ease both;
}

.ev-preview-panel {
  width: min(440px, 100vw);
  height: 100%;
  background: $card;
  border-left: 1px solid $line;
  box-shadow: -20px 0 40px -16px rgba(0, 0, 0, 0.28);
  display: flex;
  flex-direction: column;
  min-height: 0;
  animation: ev-slide-in 0.28s cubic-bezier(0.22, 0.72, 0.16, 1) both;
}

.ev-preview-head {
  flex-shrink: 0;
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 12px;
  padding: 20px 20px 16px;
  border-bottom: 1px solid $line-2;
}

.ev-preview-head-main { min-width: 0; }

.ev-preview-eyebrow {
  @extend %micro;
  font-size: 0.7692em; color: $signal; } // 0.625rem / 0.8125rem

.ev-preview-title {
  margin-top: 6px;
  font-family: $display;
  font-size: 1.3077em; // 1.0625rem / 0.8125rem
  font-weight: 800;
  letter-spacing: -0.015em;
  color: $ink;
  line-height: 1.25;
  overflow-wrap: anywhere;
}

.ev-preview-close {
  flex-shrink: 0;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 30px;
  height: 30px;
  border: 1px solid $line;
  border-radius: 9px;
  background: $paper;
  color: $ink-2;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

  &:hover { border-color: $ink-3; color: $ink; }
}

// ---- image lightbox (MinIO object-preview style) — a centered modal for
// viewing a question's input.images[], distinct from the slide-in
// .ev-preview-* panel it's launched from. ------------------------------
.ev-imgview-overlay {
  font-size: $ev-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  position: fixed;
  inset: 0;
  z-index: 80;
  background: rgba(10, 11, 15, 0.62);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 32px;
  animation: ev-fade-in 0.18s ease both;
}

.ev-imgview-panel {
  width: min(720px, 100%);
  max-height: 100%;
  background: $ink-solid;
  border-radius: 16px;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  box-shadow: 0 30px 60px -20px rgba(0, 0, 0, 0.55);
  animation: ev-toast-in 0.22s cubic-bezier(0.22, 0.72, 0.16, 1) both;
}

.ev-imgview-head {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 12px 12px 16px;
  background: rgba(255, 255, 255, 0.04);
  border-bottom: 1px solid rgba(255, 255, 255, 0.08);
}

.ev-imgview-path {
  flex: 1;
  min-width: 0;
  display: flex;
  align-items: center;
  gap: 7px;
  font-family: $mono;
  font-size: 0.9231em; // 0.75rem / 0.8125rem
  font-weight: 600;
  color: #fff;
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;

  svg { flex-shrink: 0; color: rgba(255, 255, 255, 0.55); }
}

.ev-imgview-count {
  flex-shrink: 0;
  font-family: $mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  color: rgba(255, 255, 255, 0.55);
}

.ev-imgview-close {
  flex-shrink: 0;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  border: 1px solid rgba(255, 255, 255, 0.16);
  border-radius: 8px;
  background: transparent;
  color: rgba(255, 255, 255, 0.75);
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

  &:hover { border-color: rgba(255, 255, 255, 0.4); color: #fff; background: rgba(255, 255, 255, 0.08); }
}

.ev-imgview-body {
  position: relative;
  min-height: 260px;
  max-height: calc(100vh - 180px);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 16px;
  overflow: auto;
}

.ev-imgview-img {
  display: block;
  max-width: 100%;
  max-height: calc(100vh - 220px);
  border-radius: 6px;
  object-fit: contain;
}

.ev-imgview-status {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
  color: rgba(255, 255, 255, 0.7);
  font-size: 0.9231em; // 0.75rem / 0.8125rem

  .ev__error { color: $danger; }
}

.ev-imgview-nav {
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  z-index: 1;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 36px;
  height: 36px;
  border: 1px solid rgba(255, 255, 255, 0.16);
  border-radius: 999px;
  background: rgba(0, 0, 0, 0.35);
  color: #fff;
  cursor: pointer;
  transition: border-color 0.15s ease, background 0.15s ease, opacity 0.15s ease;

  &:hover:not(:disabled) { border-color: rgba(255, 255, 255, 0.4); background: rgba(0, 0, 0, 0.55); }
  &:disabled { opacity: 0.3; cursor: not-allowed; }
}
.ev-imgview-nav--prev { left: 12px; }
.ev-imgview-nav--next { right: 12px; }

.ev-preview-controls {
  flex-shrink: 0;
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
  gap: 14px;
  padding: 16px 20px;
  border-bottom: 1px solid $line-2;
  background: $paper;
}

.ev-preview-limit { min-width: 0; }

.ev-preview-limit-label {
  @extend %micro;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  color: $ink-2;
  display: block;
  margin-bottom: 8px;
}

.ev-preview-limit-range {
  font-family: $sans;
  letter-spacing: 0;
  text-transform: none;
  font-weight: 500;
  color: $ink-3;
}

.ev-preview-limit-controls {
  display: inline-flex;
  align-items: center;
  gap: 6px;
}

.ev-preview-stepper-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  border: 1.5px solid $line;
  border-radius: 8px;
  background: $card;
  color: $ink-2;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease;

  &:hover:not(:disabled) { border-color: $signal; color: $signal; }
  &:disabled { cursor: not-allowed; opacity: 0.4; }
}

.ev-preview-body {
  flex: 1;
  min-height: 0;
  overflow-y: auto;
  padding: 18px 20px 24px;
}

.ev-preview-skel-list,
.ev-preview-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.ev-preview-skel-card {
  display: flex;
  flex-direction: column;
  gap: 9px;
  padding: 14px 15px;
  border: 1px solid $line;
  border-radius: 12px;
  background: $paper;
}

.ev-preview-q {
  padding: 14px 15px;
  border: 1px solid $line;
  border-radius: 12px;
  background: $paper;
}

.ev-preview-q-head {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
}

.ev-preview-q-index {
  font-family: $mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  color: $signal;
  background: $wash;
  border-radius: 6px;
  padding: 2px 7px;
}

.ev-preview-q-cat {
  font-family: $mono;
  font-size: 0.8077em; // 0.65625rem / 0.8125rem
  font-weight: 600;
  color: $ink-3;
  background: $card;
  border: 1px solid $line;
  border-radius: 6px;
  padding: 2px 7px;
}

.ev-preview-q-prompt {
  font-size: 1.0385em; // 0.84375rem / 0.8125rem
  color: $ink;
  line-height: 1.5;
}

.ev-preview-q-source {
  margin-top: 4px;
  font-size: 0.9231em; // 0.75rem / 0.8125rem
  color: $ink-2;
  line-height: 1.4;

  span {
    @extend %micro;
    font-size: 0.7308em; // 0.59375rem / 0.8125rem
    color: $ink-3;
    margin-right: 6px;
  }
}

// Thumbnail-style trigger buttons for a question's input.images[] — click
// opens the MinIO image lightbox (.ev-imgview-* below) for that image.
.ev-preview-q-images {
  margin-top: 9px;
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.ev-preview-q-image-btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  max-width: 220px;
  padding: 5px 10px;
  border: 1px solid $line;
  border-radius: 8px;
  background: $card;
  color: $ink-2;
  font-family: $sans;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 600;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

  svg { flex-shrink: 0; color: $signal; }

  // Truncate a long filename rather than wrapping/overflowing the pill.
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;

  &:hover { border-color: $signal; color: $signal; background: $wash; }
}

.ev-preview-q-choices {
  margin-top: 9px;
  display: flex;
  flex-direction: column;
  gap: 5px;

  li {
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    color: $ink-2;
    padding: 6px 9px;
    border: 1px solid $line;
    border-radius: 8px;
    background: $card;
  }
}

.ev-preview-q-answer {
  margin-top: 10px;
  padding-top: 10px;
  border-top: 1px dashed $line-2;
  font-size: 1em; // 0.8125rem / 0.8125rem (base)
  font-weight: 600;
  color: $ok;

  span {
    @extend %micro;
    font-size: 0.7308em; // 0.59375rem / 0.8125rem
    color: $ink-3;
    margin-right: 7px;
  }

  // A second .ev-preview-q-answer block (e.g. Expected followed by
  // Reference) shouldn't repeat the divider directly under the first one.
  & + & {
    margin-top: 6px;
    padding-top: 0;
    border-top: 0;
  }
}

.ev-preview-q-meta {
  margin-top: 10px;
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.ev-preview-q-meta-item {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  color: $ink-2;
  background: $paper;
  border: 1px solid $line;
  border-radius: 6px;
  padding: 3px 8px;

  b {
    @extend %micro;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    color: $ink-3;
    font-weight: 700;
  }
}

@keyframes ev-fade-in {
  from { opacity: 0; }
  to { opacity: 1; }
}
@keyframes ev-slide-in {
  from { transform: translateX(100%); }
  to { transform: translateX(0); }
}

// ---- toast (fixed-dark chip, same reasoning as .ev__btn--primary) ---------
.ev-toast {
  // master scale control for this subtree — see $ev-base-font above
  font-size: $ev-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  position: fixed;
  right: 24px;
  bottom: 24px;
  z-index: 60;
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 14px 18px 14px 14px;
  background: $ink-solid;
  color: #fff;
  border-radius: 14px;
  box-shadow: 0 20px 40px -16px rgba(0, 0, 0, 0.5);
  animation: ev-toast-in 0.32s cubic-bezier(0.22, 0.72, 0.16, 1) both;

  &__icon {
    width: 34px;
    height: 34px;
    border-radius: 9px;
    display: grid;
    place-items: center;
    background: rgba($ok, 0.2);
    color: $ink-solid-ok;
  }
  &__title { font-family: $display; font-weight: 700; font-size: 1.0385em; } // 0.84375rem / 0.8125rem
  &__sub { font-size: 0.9231em; color: rgba(255, 255, 255, 0.6); margin-top: 1px; } // 0.75rem / 0.8125rem
}

// ---- keyframes ------------------------------------------------------------
@keyframes ev-spin { to { transform: rotate(360deg); } }
@keyframes ev-shimmer {
  0% { background-position: 100% 50%; }
  100% { background-position: 0 50%; }
}
@keyframes ev-pulse {
  0%, 100% { box-shadow: 0 0 0 0 rgba(43, 43, 245, 0.5); }
  50% { box-shadow: 0 0 0 4px rgba(43, 43, 245, 0); }
}
@keyframes ev-mcard-pulse {
  0%, 100% { box-shadow: 0 0 0 0 rgba(43, 43, 245, 0.16); }
  50% { box-shadow: 0 0 0 5px rgba(43, 43, 245, 0); }
}
@keyframes ev-row-bar {
  0%, 100% { box-shadow: inset 3px 0 0 rgba(43, 43, 245, 0.25); }
  50% { box-shadow: inset 3px 0 0 rgba(43, 43, 245, 0.9); }
}
@keyframes ev-mcard-sheen {
  0% { background-position: 140% 0; }
  100% { background-position: -40% 0; }
}
@keyframes ev-rise {
  from { opacity: 0; transform: translateY(8px); }
  to { opacity: 1; transform: none; }
}
@keyframes ev-toast-in {
  from { opacity: 0; transform: translateY(12px) scale(0.98); }
  to { opacity: 1; transform: none; }
}

// ---- responsive -----------------------------------------------------------
@media (max-width: 1040px) {
  .ev__shell { grid-template-columns: 1fr; }
  .ev__manifest { display: none; }
  .ev__suite, .ev__metrics { grid-template-columns: 1fr; }
  .ev__samples-instruction-row { grid-template-columns: 1fr; }
  .ev__split, .ev__frow, .ev__choice-row, .ev__namegrid { grid-template-columns: 1fr; }
  .ev__about-cols { grid-template-columns: 1fr; }
  .ev__about-col + .ev__about-col { border-left: 0; border-top: 1px solid $line; }
  .ev__hprev { position: static; }
  .ev__split { grid-template-rows: none; }
  .ev__frow { gap: 12px; }
  .ev__rail, .ev__judge { max-height: 15rem; }
}

@media (max-width: 640px) {
  .ev__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .page { padding: 16px 14px 22px; }
  .ev__stage-head { padding: 18px 18px 15px; }
  .ev__stage-body { padding: 18px; }
  .ev__footer { padding: 14px 18px; }
  .ev__fw-grid, .ev__choice-grid { grid-template-columns: 1fr; }
  .ev__blocker { display: none; }
  .ev__banner { flex-direction: column; align-items: flex-start; }
  .ev__settings-row { grid-template-columns: minmax(0, 1fr) auto; }
  .ev__settings-k { grid-column: 1 / -1; }
  .ev__summary { grid-template-columns: 1fr; }
  .ev__hint { display: none; }
  .ev__name-input { font-size: 1.6923em; } // 1.375rem / 0.8125rem
  .ev__step-toolbar { flex-direction: column; align-items: stretch; }
  .ev__toolbar-search { max-width: none; }
  .ev-preview-panel { width: 100vw; }
  .ev-preview-controls { flex-direction: column; align-items: stretch; }
}

@media (prefers-reduced-motion: reduce) {
  .ev-preview-overlay, .ev-preview-panel { animation: none !important; }
  .ev__skel-block { animation: none !important; }
}

@media (prefers-reduced-motion: reduce) {
  .ev *, .ev-toast { animation: none !important; transition: none !important; }
}
