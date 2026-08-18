@use '../../styles/_variables' as *;

// ===========================================================================
// SemcoEval — Run Console
// A precision "instrument panel" for assembling and launching an evaluation.
// Signature: a live Run Manifest (mono spec sheet) threaded by a signal rail.
// Header matches the History/Reports/Comparison/Sidebar design standard.
//
// Neutrals resolve to theme CSS vars (see _theme.scss) for dark-mode support.
// $solid is a FIXED near-black used only for "always dark" chips/buttons
// (option icons, the Continue button, the launch toast) — using themed
// $ink there would turn them near-white (and invisible) in dark mode.
// ===========================================================================

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
$ok:       #0FA968;
$ok-wash:  var(--ok-wash);
$amber:    #E08600;
$danger:   #DC2626;
$danger-wash: var(--danger-wash);

// fixed, non-themed — always dark, regardless of light/dark mode
$solid:       #14161B;
$solid-hover: #000000;

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
      background: $solid;
      color: #fff;
      box-shadow: $soft;
      &:not(:disabled):hover { background: $solid-hover; transform: translateY(-1px); box-shadow: $lift; }
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
    font-size: 2.1538em; // 1.75rem / 0.8125rem
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
    flex-direction: column;
    gap: 10px;
    max-width: 720px;
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
    background: $solid;
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
  &__option-icon--agent { background: #6D28D9; }
  &__option-icon--rag   { background: #0369A1; }

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
    font-size: 1.2308em; // 1rem / 0.8125rem
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
    font-size: 1.0769em; // 0.875rem / 0.8125rem
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

  &__fw-name { font-family: $display; font-size: 1.0385em; font-weight: 700; color: $ink; } // 0.84375rem / 0.8125rem
  &__fw-desc { font-size: 0.9231em; color: $ink-2; margin-top: 2px; line-height: 1.4; } // 0.75rem / 0.8125rem

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

  &__grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(258px, 1fr));
    gap: 12px;
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
  &__pcard-name { font-family: $display; font-size: 1.2308em; font-weight: 700; color: $ink; } // 1rem / 0.8125rem
  &__pcard-meta { font-size: 1.0769em; color: $ink-3; } // 0.875rem / 0.8125rem

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
    grid-template-columns: 1fr 300px;
    gap: 16px;
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

  &__check-label { font-size: 1em; font-weight: 600; color: $ink; } // 0.8125rem / 0.8125rem (base)

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
    gap: 16px;
  }

  // No judge rail rendered (LLM_Judge metric not selected) — metrics chips
  // take the full stage width instead of leaving an empty 300px column.
  &__metrics--single {
    grid-template-columns: 1fr;
  }

  &__metrics-main { min-width: 0; min-height: 0; display: flex; flex-direction: column; }

  &__samples {
    display: flex;
    align-items: flex-start;
    flex-wrap: wrap;
    gap: 16px;
    margin-bottom: 18px;
  }
  &__samples .ev__field { max-width: 150px; margin: 0; }
  // The Custom/Full run-samples control needs more room than the old plain
  // number input did — override the tight 150px cap from the rule above.
  &__field--samples { max-width: 260px; }
  &__samples-note { font-size: 0.9231em; color: $ink-3; padding-top: 34px; line-height: 1.4; max-width: 240px; } // 0.75rem / 0.8125rem

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

  &__metrics-bar {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
    margin-bottom: 12px;
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
    min-height: 0;
    overflow-y: auto;
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
    font-size: 1em; // 0.8125rem / 0.8125rem (base)
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
  }

  &__chip-tick {
    display: grid;
    place-items: center;
    width: 14px;
    height: 14px;
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

  // Shown at the bottom of the judge rail while it's mandatory (LLM_Judge
  // selected) but no judge model has been picked yet.
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

  &__judge-name { font-size: 1.0769em; font-weight: 600; color: $ink; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; } // 0.875rem / 0.8125rem
  &__judge-meta { font-size: 0.9231em; color: $ink-3; } // 0.75rem / 0.8125rem

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
  background: $solid;
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
    background: rgba(15, 169, 104, 0.2);
    color: #34D399;
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
  .ev__rail, .ev__judge { max-height: 15rem; }
}

@media (max-width: 640px) {
  .ev__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .page { padding: 16px 14px 22px; }
  .ev__stage-head { padding: 18px 18px 15px; }
  .ev__stage-body { padding: 18px; }
  .ev__footer { padding: 14px 18px; }
  .ev__fw-grid { grid-template-columns: 1fr; }
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
