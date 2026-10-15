@use 'sass:color';
@use '../../styles/_variables' as *;

// ===========================================================================
// New Evaluation — wizard console, restyled to the Datasets (Test Suite
// Library) design language:
//   • flat full-bleed master/detail body: left rail (the Run Manifest) +
//     right detail pane (the current step), no floating outer cards
//   • bordered rail cards with a 4px colored left edge, 13px radius, 1px lift
//   • pill buttons / pill filter groups, mono micro-labels, hero accent bar
//   • stats grid (1px hairlines), section heads with count pills
//   • same modal / drawer / lightbox treatment as Datasets
//
// Color tokens come from the shared ink system in _variables.scss — nothing
// is redeclared here, so dark mode keeps working. Every class name used by
// NewEvaluation.tsx is preserved; this file is a drop-in replacement.
// ===========================================================================

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

$soft: 0 1px 2px rgba(20, 22, 27, 0.05);
$lift: 0 14px 30px -14px rgba(20, 22, 27, 0.22);
$ring: 0 0 0 3px rgba(43, 43, 245, 0.16);

// Base font-size every `em` below is relative to. Four DOM subtrees render
// independently (.ev__header, .page, .ev-toast, .ev-preview-overlay /
// .ev-imgview-overlay), so each sets it itself.
$ev-base-font: 0.8125rem;

%micro {
  font-family: $mono;
  font-size: 0.8462em;
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

@mixin scale-root {
  font-size: $ev-base-font;
  @media (min-width: 1800px) { font-size: 1rem; }
}

// Selectable card — the Datasets "row" look: bordered, 4px left edge,
// 13px radius, subtle lift on hover.
@mixin sel-card {
  background: $card;
  border: 1px solid $line;
  border-left-width: 4px;
  border-radius: 13px;
  cursor: pointer;
  text-align: left;
  transition: border-color 0.14s ease, box-shadow 0.14s ease, transform 0.14s ease, background 0.14s ease;

  &:hover {
    border-color: $ink-3;
    box-shadow: $soft;
    transform: translateY(-1px);
  }
}
@mixin sel-card-on {
  border-color: $signal;
  background: $wash;
  box-shadow: $soft;
  &:hover { border-color: $signal; }
}

// Pill group container (Datasets filters / eval-pills).
@mixin pill-group {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 4px;
  background: $paper;
  border: 1px solid $line;
  border-radius: 999px;
  flex-wrap: wrap;
}

// ===========================================================================
// Header — identical to Datasets header
// ===========================================================================
.ev__header {
  @include scale-root;
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
    font-size: 1.8462em;
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
  font-size: 1.0385em;
  color: $ink-2;
  max-width: 52ch;
}

.ev__header-meta {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  gap: 10px;
  flex-wrap: wrap;
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
  font-size: 0.8846em;
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

// Same pill treatment as Datasets' header-count
.ev__header-eta {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  padding: 7px 13px;
  border-radius: 999px;
  border: 1px solid $line;
  background: $paper;
  font-family: $mono;
  font-size: 0.8846em;
  font-weight: 700;
  letter-spacing: 0.06em;
  text-transform: uppercase;
  color: $ink-2;
  white-space: nowrap;
}

.page {
  @include scale-root;
  flex: 1;
  height: 100%;
  min-height: 0;
  padding: 0;
  display: flex;
  flex-direction: column;
  background: $paper;
}

// ===========================================================================
// Root + master/detail shell
// ===========================================================================
.ev {
  flex: 1;
  min-height: 0;
  display: flex;
  flex-direction: column;

  &__shell {
    flex: 1;
    min-height: 0;
    display: grid;
    grid-template-columns: 288px minmax(0, 1fr);
    gap: 0;
    padding-left: 30px;
  }

  // ========================================================================
  // Run Manifest — original steps design: floating card, a connected
  // vertical signal rail through the step ticks, live value per step.
  // ========================================================================
  &__manifest {
    margin: 22px 26px 26px 0;
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

  &__manifest-pct { color: $signal; font-size: 0.9231em; letter-spacing: 0.06em; }

  &__manifest-title {
    margin-top: 8px;
    font-family: $display;
    font-size: 1.2308em;
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

    // Extends past this row's own bottom edge by exactly the flex gap
    // above (24px), so the line still reaches the next row's tick.
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
    font-size: 0.8462em;
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
    font-size: 0.9231em;
    color: $ink-3;
    transition: color 0.18s ease;
  }

  &__spec-value {
    font-size: 1.1538em;
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
  // DETAIL — the current step (Datasets detail pane)
  // ========================================================================
  &__stage {
    min-width: 0;
    min-height: 0;
    display: flex;
    flex-direction: column;
    background: $paper;
  }

  // hero: coloured accent bar down the left edge, display title, sub copy
  &__stage-head {
    position: relative;
    flex-shrink: 0;
    padding: 26px 30px 20px 52px;
    border-bottom: 1px solid $line-2;

    &::before {
      content: '';
      position: absolute;
      left: 30px;
      top: 30px;
      bottom: 24px;
      width: 4px;
      border-radius: 3px;
      background: $signal;
    }
  }

  &__crumb {
    @extend %micro;
    display: flex;
    align-items: center;
    gap: 9px;
    font-size: 0.8462em;
    letter-spacing: 0.1em;
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
    margin-top: 8px;
    font-family: $display;
    font-size: 1.8462em;
    font-weight: 700;
    letter-spacing: -0.02em;
    color: $ink;
    line-height: 1.15;
  }

  &__stage-sub {
    margin-top: 10px;
    font-size: 1.1538em;
    color: $ink-2;
    line-height: 1.6;
    max-width: 64ch;
  }

  &__stage-body {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding: 26px 30px 40px;
    display: flex;
    flex-direction: column;
  }

  &__anim { animation: ev-rise 0.22s ease both; }

  &__footer {
    flex-shrink: 0;
    padding: 14px 30px;
    border-top: 1px solid $line;
    background: $card;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
  }

  &__hint {
    font-size: 0.9231em;
    color: $ink-3;
    display: flex;
    align-items: center;
    gap: 7px;
    min-width: 0;

    kbd {
      font-family: $mono;
      font-size: 0.8462em;
      color: $ink-2;
      background: $paper;
      border: 1px solid $line;
      border-bottom-width: 2px;
      border-radius: 5px;
      padding: 1px 6px;
    }
  }

  // ---- buttons: pills, like Datasets refresh / upload buttons -------------
  &__btn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    white-space: nowrap;
    font-family: $sans;
    font-size: 0.9615em;
    font-weight: 650;
    border-radius: 999px;
    padding: 8px 16px;
    cursor: pointer;
    border: 1px solid transparent;
    transition: background 0.15s ease, border-color 0.15s ease, color 0.15s ease, transform 0.15s ease, box-shadow 0.15s ease;

    &:disabled { cursor: not-allowed; opacity: 0.5; }

    &--ghost {
      background: $card;
      border-color: $line;
      color: $ink-2;
      &:not(:disabled):hover { border-color: $ink-3; color: $ink; background: $paper; }
    }

    &--primary {
      background: $signal;
      color: #fff;
      &:not(:disabled):hover { background: $signal-2; }
    }

    &--launch {
      background: $signal;
      color: #fff;
      box-shadow: 0 4px 12px -6px rgba(43, 43, 245, 0.6);
      &:not(:disabled):hover { background: $signal-2; transform: translateY(-1px); }
    }
  }

  // ========================================================================
  // Field primitives (Datasets modal fields)
  // ========================================================================
  &__field {
    max-width: 620px;
    & + & { margin-top: 20px; }
  }

  &__label {
    display: flex;
    align-items: center;
    gap: 7px;
    font-family: $mono;
    font-size: 0.7692em;
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ink-3;
    margin-bottom: 8px;

    .opt {
      font-family: $sans;
      letter-spacing: 0;
      text-transform: none;
      font-weight: 500;
      font-size: 1.1em;
      color: $ink-3;
    }
  }

  &__input {
    width: 100%;
    border: 1.5px solid $line;
    border-radius: 10px;
    padding: 9px 12px;
    font-size: 1.0385em;
    font-family: $sans;
    color: $ink;
    background: $card;
    transition: border-color 0.15s ease, background 0.15s ease;

    &::placeholder { color: $ink-3; }
    &:focus { outline: none; border-color: $signal; }
    &:disabled { background: $paper; color: $ink-2; opacity: 0.8; cursor: not-allowed; }
  }

  // Run name — original underline-style field
  &__name-input {
    width: 100%;
    border: 0;
    border-bottom: 2px solid $line;
    border-radius: 0;
    padding: 8px 2px 12px;
    background: transparent;
    font-family: $display;
    font-size: 1.6923em;
    font-weight: 800;
    letter-spacing: -0.03em;
    color: $ink;
    transition: border-color 0.16s ease;

    &::placeholder { color: $ink-3; font-weight: 700; }
    &:focus { outline: none; border-color: $signal; }
  }

  &__field:has(> .ev__name-input) > .ev__label {
    font-size: 0.8462em;
    letter-spacing: 0.14em;
    color: $ink-2;
    margin-bottom: 9px;
  }

  &__name-caption {
    margin-top: 10px;
    font-size: 0.9615em;
    color: $ink-3;
    display: flex;
    align-items: center;
    gap: 7px;
  }

  // ---- quick-start presets (original mono chips) --------------------------
  &__quick {
    margin-top: 30px;
    max-width: 620px;
  }

  &__quick-head {
    @extend %micro;
    font-size: 0.7692em;
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
    font-size: 0.9231em;
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

  // ---- "What makes a good name" tips note (original) ----------------------
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
    font-size: 1em;
    font-weight: 700;
    color: $ink;
    margin-bottom: 6px;
  }

  &__note-list {
    display: flex;
    flex-direction: column;
    gap: 5px;
    font-size: 0.9615em;
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
  // Type step: option rows + framework cards
  // ========================================================================
  &__options { display: flex; flex-direction: column; gap: 10px; max-width: 720px; }

  &__option {
    @include sel-card;
    position: relative;
    display: flex;
    align-items: center;
    gap: 15px;
    width: 100%;
    padding: 16px 52px 16px 16px;

    &--on { @include sel-card-on; }
    &--off { opacity: 0.55; cursor: not-allowed; &:hover { border-color: $line; box-shadow: none; transform: none; } }
  }

  &__option-icon {
    flex-shrink: 0;
    width: 44px;
    height: 44px;
    border-radius: 12px;
    display: grid;
    place-items: center;
    background: $ink-solid;
    color: #fff;
  }
  &__option-icon--agent { background: $violet-ink; }
  &__option-icon--rag   { background: $sky-ink; }

  &__option-main { flex: 1; min-width: 0; display: flex; flex-direction: column; gap: 4px; }

  &__option-name {
    display: flex;
    align-items: center;
    gap: 9px;
    font-family: $display;
    font-size: 1.2308em;
    font-weight: 650;
    color: $ink;
  }

  &__badge {
    font-family: $mono;
    font-size: 0.7692em;
    font-weight: 700;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $ink-3;
    background: $paper;
    border: 1px solid $line;
    border-radius: 999px;
    padding: 1px 7px;
  }

  &__option-desc { font-size: 1.0769em; color: $ink-2; line-height: 1.5; }

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
  }

  &__section {
    margin-top: 26px;
    padding-top: 22px;
    border-top: 1px solid $line-2;
    max-width: 720px;
  }

  &__section-hint { font-size: 0.9615em; color: $ink-3; margin: 4px 0 14px; }

  &__fw-grid { display: grid; grid-template-columns: repeat(2, 1fr); gap: 10px; }

  &__fw {
    @include sel-card;
    position: relative;
    display: flex;
    align-items: center;
    gap: 12px;
    padding: 13px 42px 13px 13px;

    &--on { @include sel-card-on; }
  }

  &__fw-icon {
    flex-shrink: 0;
    width: 34px;
    height: 34px;
    border-radius: 10px;
    display: grid;
    place-items: center;
    background: $wash;
    color: $signal;
  }

  &__fw-name { font-family: $display; font-size: 1.1538em; font-weight: 650; color: $ink; }
  &__fw-desc { font-size: 1em; color: $ink-2; margin-top: 2px; line-height: 1.4; }

  // ========================================================================
  // Per-step toolbar (Datasets search + refresh)
  // ========================================================================
  &__step-toolbar { display: flex; align-items: center; gap: 10px; margin-bottom: 14px; }

  &__toolbar-search {
    position: relative;
    flex: 1;
    min-width: 200px;
    max-width: 360px;
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 9px 12px;
    border: 1.5px solid $line;
    border-radius: 10px;
    background: $card;
    color: $ink-3;
    transition: border-color 0.15s ease, background 0.15s ease;

    &:focus-within { border-color: $signal; color: $signal; }

    svg { flex-shrink: 0; }

    input {
      flex: 1;
      min-width: 0;
      border: 0;
      background: transparent;
      font-family: $sans;
      font-size: 1.0385em;
      color: $ink;
      outline: none;
      &::placeholder { color: $ink-3; }
    }
  }

  &__toolbar-refresh {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 34px;
    height: 34px;
    border: 1px solid $line;
    border-radius: 999px;
    background: $card;
    color: $ink-2;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover:not(:disabled) { border-color: $ink-3; color: $ink; background: $paper; }
    &:disabled { cursor: default; color: $signal; }
  }

  // All / Custom / Deepeval — Datasets filter pill group
  &__filter-row { @include pill-group; margin-bottom: 14px; }

  &__filter-chip {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    padding: 6px 8px 6px 13px;
    border: 0;
    border-radius: 999px;
    background: transparent;
    color: $ink-2;
    font-family: $sans;
    font-size: 0.9615em;
    font-weight: 650;
    cursor: pointer;
    transition: all 0.15s ease;

    &:hover { color: $ink; }

    &--on { background: $card; color: $signal; box-shadow: $soft; }
  }

  &__filter-chip-count {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    min-width: 20px;
    height: 18px;
    padding: 0 6px;
    border-radius: 999px;
    background: $paper;
    border: 1px solid $line;
    color: $ink-3;
    font-family: $mono;
    font-size: 0.8077em;
    font-weight: 700;
  }
  &__filter-chip--on &__filter-chip-count { background: $signal; border-color: $signal; color: #fff; }

  // ========================================================================
  // Skeletons — Datasets shimmer
  // ========================================================================
  &__skel-block {
    display: block;
    border-radius: 6px;
    background: linear-gradient(90deg, $line-2 25%, $paper 50%, $line-2 75%);
    background-size: 200% 100%;
    animation: ev-shimmer 1.2s ease-in-out infinite;

    &--icon { width: 40px; height: 40px; border-radius: 11px; flex-shrink: 0; }
    &--icon-sm { width: 36px; height: 36px; border-radius: 10px; flex-shrink: 0; }
    &--line { height: 11px; border-radius: 6px; }
    &--pill { width: 72px; height: 16px; border-radius: 999px; margin-top: 2px; }
    &--tag { width: 54px; height: 18px; border-radius: 6px; }
    &--stat { width: 58px; height: 26px; border-radius: 7px; }
    &--chip { height: 30px; border-radius: 8px; }
  }

  &__skel-pcard,
  &__skel-mcard,
  &__skel-dcard {
    border: 1px solid $line;
    border-left-width: 4px;
    border-radius: 13px;
    background: $card;
  }
  &__skel-pcard { display: flex; align-items: flex-start; gap: 13px; padding: 15px 42px 15px 15px; }
  &__skel-lines { flex: 1; min-width: 0; display: flex; flex-direction: column; gap: 8px; }
  &__skel-mcard { display: flex; flex-direction: column; gap: 10px; padding: 15px 16px; }
  &__skel-caps { display: flex; gap: 6px; }
  &__skel-stats { display: flex; gap: 10px; padding-top: 10px; border-top: 1px solid $line-2; }
  &__skel-dcard { display: flex; flex-direction: column; gap: 12px; padding: 15px 16px; }
  &__skel-dcard-top { display: flex; align-items: center; gap: 11px; }

  // ========================================================================
  // Card grids
  // ========================================================================
  &__scroll { flex: 1; min-height: 0; overflow-y: auto; margin: 0 -6px; padding: 4px 6px; }

  &__models-panel { flex: 1; min-height: 0; display: flex; flex-direction: column; }
  &__models-scroll { flex: 1; min-height: 0; overflow-y: auto; margin: 8px -6px 0; padding: 4px 6px; }

  &__rag-columns {
    display: flex;
    align-items: stretch;
    gap: 14px;
    height: 100%;
    min-height: 0;
    overflow-x: auto;
  }

  &__rag-col {
    flex: 1 1 0;
    min-width: 240px;
    min-height: 0;
    display: flex;
    flex-direction: column;
    border: 1px solid $line;
    border-radius: 14px;
    background: $card;
    overflow: hidden;
  }

  &__rag-col-head {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
    padding: 12px 14px;
    border-bottom: 1px solid $line-2;
  }

  &__rag-col-search {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 7px;
    margin: 10px 12px 0;
    padding: 6px 10px;
    border: 1.5px solid $line;
    border-radius: 10px;
    background: $paper;
    transition: border-color 0.15s ease, background 0.15s ease;

    &:focus-within { border-color: $signal; background: $card; }
    svg { flex-shrink: 0; color: $ink-3; }

    input {
      flex: 1;
      min-width: 0;
      border: 0;
      background: transparent;
      font-family: $sans;
      font-size: 0.9231em;
      color: $ink;
      &:focus { outline: none; }
      &::placeholder { color: $ink-3; }
    }
  }

  &__rag-col-title {
    min-width: 0;
    font-family: $display;
    font-size: 1.1538em;
    font-weight: 700;
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
    padding: 1px 8px;
    border-radius: 999px;
    border: 1px solid $line;
    background: $paper;
    color: $ink-3;
    font-family: $mono;
    font-size: 0.7692em;
    font-weight: 700;
    letter-spacing: 0.04em;
    text-transform: uppercase;
  }
  &__rag-col-status--on { background: $ok-wash; border-color: transparent; color: $ok; }

  &__rag-col-list {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 12px;

    .ev__mcard { width: 100%; }
  }

  &__grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(258px, 1fr));
    gap: 12px;
    align-items: start;

    @media (min-width: 1800px) { grid-template-columns: repeat(auto-fill, minmax(310px, 1fr)); }
  }
  &__grid--wide { grid-template-columns: repeat(auto-fill, minmax(300px, 1fr)); }

  // ---- provider card ------------------------------------------------------
  &__pcard {
    @include sel-card;
    position: relative;
    display: flex;
    align-items: flex-start;
    gap: 13px;
    padding: 15px 42px 15px 15px;

    &--on { @include sel-card-on; }
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
    font-size: 1.1538em;
    transition: all 0.16s ease;
  }
  &__pcard--on &__pcard-icon { background: $signal; border-color: $signal; color: #fff; }

  &__pcard-body { min-width: 0; display: flex; flex-direction: column; gap: 3px; }
  &__pcard-name { font-family: $display; font-size: 1.2308em; font-weight: 650; color: $ink; }
  &__pcard-meta { font-family: $mono; font-size: 1em; font-weight: 700; color: $ink-3; }

  &__pill {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    margin-top: 4px;
    width: fit-content;
    font-family: $mono;
    font-size: 0.7692em;
    font-weight: 700;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $ok;
    background: $ok-wash;
    border-radius: 999px;
    padding: 1px 8px 1px 6px;

    &::before { content: ''; width: 5px; height: 5px; border-radius: 50%; background: $ok; }
  }

  // ---- model card ---------------------------------------------------------
  &__mcard {
    @include sel-card;
    position: relative;
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 14px 16px 14px 17px;

    &--on { @include sel-card-on; }
  }

  &__mcard--locked {
    cursor: default;
    &:hover { border-color: $line; box-shadow: none; transform: none; }
  }

  &__mcard--checking {
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

  &__mcard-top { display: flex; align-items: baseline; justify-content: space-between; gap: 10px; }

  &__mcard-name { font-family: $display; font-size: 1.2308em; font-weight: 650; color: $ink; line-height: 1.3; }
  &__mcard-provider {
    font-family: $mono;
    font-size: 0.9231em;
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ink-3;
  }

  &__mcard-provider-row { display: flex; align-items: center; justify-content: space-between; gap: 8px; }

  &__mcard-hint {
    margin-top: 2px;
    padding-top: 10px;
    border-top: 1px dashed $line-2;
    font-size: 0.8846em;
    color: $ink-3;
    line-height: 1.4;
  }

  &__health-badge {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 4px;
    padding: 1px 8px 1px 6px;
    border-radius: 999px;
    border: 0;
    font-family: $mono;
    font-size: 0.7692em;
    font-weight: 700;
    letter-spacing: 0.02em;
    white-space: nowrap;

    &--success { color: $ok; background: $ok-wash; }
    &--failed { color: $danger; background: $danger-wash; cursor: pointer; &:hover { background: rgba($danger, 0.16); } }
    &--loading { color: $ink-3; background: $paper; }
  }

  &__health-check-btn {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 4px;
    padding: 1px 9px;
    border-radius: 999px;
    border: 1px solid $signal;
    background: $card;
    color: $signal;
    font-family: $mono;
    font-size: 0.7692em;
    font-weight: 700;
    letter-spacing: 0.02em;
    white-space: nowrap;
    cursor: pointer;
    transition: background 0.15s ease;

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
    font-size: 0.7692em;
    font-weight: 700;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $ink-3;
    background: $paper;
    border: 1px solid $line;
    border-radius: 999px;
    padding: 1px 7px;
  }

  &__mcard-stats { display: flex; flex-wrap: wrap; gap: 14px; padding-top: 10px; border-top: 1px solid $line-2; }

  &__stat { display: flex; flex-direction: column; gap: 3px; }
  &__stat-k {
    font-family: $mono;
    font-size: 0.7692em;
    font-weight: 700;
    letter-spacing: 0.1em;
    text-transform: uppercase;
    color: $ink-3;
  }
  &__stat-v { font-family: $mono; font-size: 1.0769em; font-weight: 700; color: $ink; }

  &__mcard-retry {
    display: flex;
    flex-direction: column;
    gap: 8px;
    margin-top: 2px;
    padding-top: 10px;
    border-top: 1px solid $line-2;
    cursor: default;
  }
  &__mcard-retry-fields { display: flex; gap: 10px; }
  &__mcard-retry-hint {
    display: flex;
    align-items: flex-start;
    gap: 5px;
    font-size: 0.7692em;
    color: $ink-3;
    line-height: 1.3;
    svg { flex-shrink: 0; margin-top: 2px; }
  }
  &__mcard-retry-field {
    flex: 1;
    display: flex;
    flex-direction: column;
    gap: 5px;

    label {
      font-family: $mono;
      font-size: 0.7692em;
      font-weight: 700;
      letter-spacing: 0.06em;
      text-transform: uppercase;
      color: $ink-3;
    }

    input {
      width: 100%;
      border: 1.5px solid $line;
      border-radius: 10px;
      padding: 6px 10px;
      font-family: $mono;
      font-size: 0.9615em;
      font-weight: 700;
      color: $ink;
      background: $paper;
      transition: border-color 0.15s ease, background 0.15s ease;

      &:focus { outline: none; border-color: $signal; background: $card; }
    }
  }

  // Retries & timeout panel — bordered card, like a Datasets section
  &__retry-mode {
    margin-top: 2px;
    margin-bottom: 4px;
    padding: 14px 16px;
    border: 1px solid $line;
    border-radius: 14px;
    background: $card;
  }
  &__retry-mode-head { display: flex; align-items: center; justify-content: space-between; flex-wrap: wrap; gap: 10px; .ev__label { margin-bottom: 0; } }
  &__retry-mode-all { display: flex; align-items: flex-end; flex-wrap: wrap; gap: 12px; margin-top: 12px; .ev__mcard-retry-field { flex: 0 0 130px; } }
  &__retry-mode-note {
    display: flex;
    align-items: flex-start;
    gap: 5px;
    flex: 1 1 100%;
    margin-top: 10px;
    font-size: 0.8846em;
    color: $ink-3;
    line-height: 1.4;
    svg { flex-shrink: 0; margin-top: 2px; }
  }

  // ---- Retest on Wrong body ----------------------------------------------
  &__retest-fields { display: flex; flex-wrap: wrap; align-items: stretch; gap: 16px; }

  &__retest-panel {
    flex: 0 0 200px;
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 14px 16px;
    border: 1px solid $line;
    border-left: 4px solid $signal;
    border-radius: 13px;
    background: $paper;

    input[type='number'] {
      width: 100%;
      border: 1.5px solid $line;
      border-radius: 10px;
      padding: 8px 12px;
      font-family: $mono;
      font-size: 1.0385em;
      font-weight: 700;
      color: $ink;
      background: $card;
      transition: border-color 0.15s ease;

      &:focus { outline: none; border-color: $signal; }
    }
  }
  &__retest-panel--verify { flex: 1 1 280px; }

  &__retest-panel-label {
    display: flex;
    align-items: center;
    gap: 6px;
    font-family: $mono;
    font-size: 0.7692em;
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ink-3;
    svg { flex-shrink: 0; color: $signal; }
  }
  &__retest-panel-hint { font-size: 0.8462em; color: $ink-3; line-height: 1.4; }

  // ========================================================================
  // Test Suite step: tabs (pill group), dataset cards, subgroup rail, upload
  // ========================================================================
  &__tabs { @include pill-group; margin-bottom: 16px; }

  &__tab {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    padding: 6px 16px;
    border: 0;
    border-radius: 999px;
    background: transparent;
    color: $ink-2;
    font-family: $sans;
    font-size: 0.9615em;
    font-weight: 650;
    cursor: pointer;
    transition: all 0.15s ease;

    &:hover { color: $ink; }
    &--on { background: $card; color: $signal; box-shadow: $soft; }
  }

  // Split layout — same separation as the Datasets RAG "Upload & Processing"
  // panel: the right column runs to the stage's edge with its own
  // border-left, tinted background and independent scroll.
  &__suite {
    flex: 1;
    min-height: 0;
    display: grid;
    grid-template-columns: minmax(0, 1fr) minmax(300px, 380px);
    grid-template-rows: minmax(0, 1fr);
    gap: 0;
    margin: 0 -30px -40px 0; // bleed through the stage-body padding (right/bottom)
  }
  // keeps the 0-question warning from sliding under the bled-out grid
  &__suite + &__dataset-empty-warning { margin-top: 54px; }

  &__suite-scroll { min-width: 0; min-height: 0; overflow-y: auto; margin: 0 0 0 -6px; padding: 2px 30px 6px 6px; }

  &__dgrid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
    gap: 12px;

    @media (min-width: 1800px) { grid-template-columns: repeat(auto-fill, minmax(310px, 1fr)); }
  }

  &__dcard {
    @include sel-card;
    position: relative;
    display: flex;
    flex-direction: column;
    gap: 11px;
    padding: 14px 16px 14px 17px;

    &--on { @include sel-card-on; }
    &--empty { border-left-color: $danger; }
  }

  &__dcard-top { display: flex; align-items: center; justify-content: space-between; gap: 10px; }
  &__dcard-id { display: flex; align-items: center; gap: 11px; min-width: 0; }

  &__dcard-icon {
    flex-shrink: 0;
    width: 34px;
    height: 34px;
    border-radius: 10px;
    display: grid;
    place-items: center;
    background: $paper;
    border: 1px solid $line;
    color: $ink;
    transition: all 0.16s ease;
  }
  &__dcard--on &__dcard-icon { background: $signal; border-color: $signal; color: #fff; }

  &__dcard-name {
    font-family: $display;
    font-size: 1.2308em;
    font-weight: 650;
    color: $ink;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__dcard-warn-icon { flex-shrink: 0; display: inline-flex; color: $danger; }

  &__dcard-tags { display: flex; flex-wrap: wrap; align-items: center; gap: 6px; }

  &__tag {
    font-family: $mono;
    font-size: 0.7692em;
    font-weight: 700;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $ink-3;
    background: $paper;
    border: 1px solid $line;
    border-radius: 999px;
    padding: 1px 7px;
  }
  &__tag--custom { color: $signal; background: $wash; border-color: rgba($signal, 0.22); }
  &__tag--deepeval { color: $ok; background: $ok-wash; border-color: transparent; }
  &__tag--count { border: 0; background: transparent; color: $ink-3; padding-left: 0; }
  &__tag--count-empty { color: $danger; }

  &__dataset-empty-warning {
    display: flex;
    align-items: flex-start;
    gap: 9px;
    margin-top: 14px;
    padding: 11px 14px;
    border-radius: 10px;
    background: $danger-wash;
    color: $danger;
    font-size: 0.9231em;
    font-weight: 600;
    line-height: 1.5;

    svg { flex-shrink: 0; margin-top: 2px; }
    strong { font-weight: 700; }
  }

  &__dcard-actions { flex-shrink: 0; display: flex; align-items: center; gap: 8px; }

  // Datasets' preview button (solid pill)
  &__dcard-preview-btn {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 5px 12px;
    border-radius: 999px;
    border: 1px solid transparent;
    background: $signal;
    color: #fff;
    font-family: $sans;
    font-size: 0.9231em;
    font-weight: 650;
    cursor: pointer;
    transition: background 0.15s ease, transform 0.15s ease;

    &:hover { background: $signal-2; transform: translateY(-1px); }
  }

  // ---- subgroup rail (Datasets rail look) ---------------------------------
  &__rail {
    display: flex;
    flex-direction: column;
    min-height: 0;
    overflow: hidden;
    border-left: 1px solid $line;
    background: $card;
    padding: 26px 24px 40px;
  }

  &__rail-head { flex-shrink: 0; padding: 0 0 14px; border-bottom: 1px solid $line-2; }
  &__rail-head-row { display: flex; align-items: center; justify-content: space-between; gap: 10px; }
  &__rail-title {
    display: flex;
    align-items: center;
    gap: 7px;
    font-family: $mono;
    font-size: 0.9231em;
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ink-3;
    svg { color: $signal; }
  }
  &__rail-actions { flex-shrink: 0; display: flex; gap: 10px; }
  &__rail-sub { margin-top: 6px; font-size: 0.8846em; color: $ink-3; line-height: 1.45; }

  &__rail-scroll {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding: 14px 4px 4px;
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  &__rail-empty {
    margin: 10px;
    padding: 18px 12px;
    text-align: center;
    border: 1px dashed $line;
    border-radius: 12px;
    background: $paper;
    font-size: 0.9231em;
    color: $ink-3;
  }

  // subgroup checkbox rows — same bordered card + left edge as list rows
  &__check-row {
    @include sel-card;
    display: flex;
    align-items: center;
    gap: 10px;
    width: 100%;
    padding: 10px 12px 10px 13px;
    border-radius: 11px;

    &--on { @include sel-card-on; }
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
    font-family: $mono;
    font-size: 0.9231em;
    font-weight: 700;
    color: $ink;

    @media (min-width: 1800px) { font-size: 1.0769em; }
  }

  // ---- upload panel (Datasets modal body feel) -----------------------------
  &__upload {
    border: 1px solid $line;
    border-radius: 14px;
    background: $card;
    padding: 20px;
    max-width: 560px;
  }

  &__drop {
    display: flex;
    align-items: center;
    gap: 11px;
    width: 100%;
    border: 1.5px dashed $line;
    border-radius: 12px;
    padding: 20px 16px;
    background: $paper;
    color: $ink-2;
    font-size: 0.9615em;
    font-weight: 650;
    cursor: pointer;
    transition: border-color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $signal; background: $wash; }
    svg { flex-shrink: 0; color: $ink-3; }
  }
  &__drop-file { color: $ink; font-weight: 650; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; display: inline-flex; align-items: center; gap: 8px; }
  &__drop--has { border-style: solid; border-color: $signal; color: $ink; }

  &__upload-actions { display: flex; justify-content: flex-end; gap: 10px; margin-top: 20px; }

  // ========================================================================
  // Metrics step
  // ========================================================================
  // Split layout — same separation as the Datasets RAG panel (see &__suite).
  &__metrics {
    flex: 1;
    min-height: 0;
    display: grid;
    grid-template-columns: minmax(0, 1fr) minmax(300px, 380px);
    grid-template-rows: minmax(0, 1fr);
    gap: 0;
    margin: -26px -30px -40px 0; // run to the stage's top/right/bottom edges
  }

  &__metrics-main {
    min-width: 0;
    min-height: 0;
    display: flex;
    flex-direction: column;
    gap: 16px;
    overflow-y: auto;
    padding: 26px 30px 40px 0;
  }

  // section card — Datasets stats/section container
  &__msec { border: 1px solid $line; border-radius: 14px; background: $card; }

  &__msec-head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
    padding: 14px 16px 12px;
    border-bottom: 1px solid $line-2;
  }

  &__msec-head-text { min-width: 0; }

  &__msec-title {
    display: flex;
    align-items: center;
    gap: 8px;
    font-family: $display;
    font-size: 1.1538em;
    font-weight: 700;
    color: $ink;

    svg { flex-shrink: 0; color: $signal; }
  }

  &__msec-sub { margin-top: 4px; font-size: 0.9231em; color: $ink-3; line-height: 1.4; }
  &__msec-head-action { flex-shrink: 0; }
  &__msec-body { padding: 16px; }

  &__samples-instruction-row { display: grid; grid-template-columns: 1fr 1fr; align-items: start; gap: 16px; }

  &__field--samples { max-width: 260px; }
  &__field--topk {
    max-width: 480px;
    margin-bottom: 20px;
    .opt { white-space: nowrap; font-size: 0.85em; }
  }
  &__input--topk { max-width: 140px; }

  &__metrics-required {
    margin-bottom: 12px;
    padding: 10px 12px;
    border-radius: 10px;
    background: $danger-wash;
    color: $danger;
    font-size: 0.9231em;
    font-weight: 600;
    line-height: 1.4;
  }

  // Custom / Full, Yes / No, Apply to all / Individually — radio pills
  &__radio-row { @include pill-group; flex-wrap: nowrap; }

  &__radio-opt {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    border: 0;
    border-radius: 999px;
    background: transparent;
    padding: 6px 14px;
    cursor: pointer;
    font-family: $sans;
    font-size: 0.9615em;
    font-weight: 650;
    color: $ink-2;
    white-space: nowrap;
    transition: all 0.15s ease;

    &:hover { color: $ink; }
    &--on { background: $card; color: $signal; box-shadow: $soft; }
  }

  &__radio-full-note { margin-top: 10px; font-size: 0.9231em; color: $ink-3; line-height: 1.4; }

  &__metrics-count {
    font-family: $mono;
    font-size: 0.8462em;
    font-weight: 700;
    color: $ink-3;
    background: $paper;
    border: 1px solid $line;
    border-radius: 99px;
    padding: 2px 8px;
    b { color: $signal; font-weight: 700; }
  }

  &__metrics-actions { display: flex; gap: 14px; }

  &__link {
    font-family: $sans;
    font-size: 0.9231em;
    font-weight: 700;
    color: $signal;
    background: none;
    border: 0;
    padding: 0;
    cursor: pointer;
    &:hover { text-decoration: underline; }
  }

  &__chips {
    flex: 1;
    align-content: flex-start;
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
  }

  // metric chips — Datasets "cap" chips (mono, 8px radius)
  &__chip {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 6px 12px;
    border: 1px solid $line;
    border-radius: 8px;
    background: $paper;
    color: $ink-2;
    font-family: $mono;
    font-size: 0.8846em;
    font-weight: 700;
    cursor: pointer;
    height: fit-content;
    transition: transform 0.13s ease, box-shadow 0.13s ease, border-color 0.13s ease, color 0.13s ease;

    &:hover { border-color: $ink-3; color: $ink; transform: translateY(-1px); }

    &--on {
      border-color: $signal;
      background: $signal;
      color: #fff;
      box-shadow: 0 0 0 2px $card, 0 0 0 4px $signal, 0 4px 10px rgba(20, 22, 27, 0.18);
      &:hover { color: #fff; border-color: $signal; }
    }

    &--custom { border-style: dashed; &.ev__chip--on { border-style: solid; } }
  }

  &__chip-tick { display: grid; place-items: center; width: 12px; height: 12px; }
  &__chip-judge-icon { flex-shrink: 0; opacity: 0.6; }
  &__chip--on &__chip-judge-icon { opacity: 0.85; }

  &__instruction-generate-btn {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 6px 13px;
    border: 1px solid $line;
    border-radius: 999px;
    background: $card;
    color: $signal;
    font-family: $sans;
    font-size: 0.9231em;
    font-weight: 650;
    cursor: pointer;
    transition: border-color 0.15s ease, background 0.15s ease;

    &:hover:not(:disabled) { border-color: $signal; background: $wash; }
    &:disabled { cursor: not-allowed; opacity: 0.5; }
  }

  &__instruction-textarea {
    width: 100%;
    min-height: 110px;
    border: 1.5px solid $line;
    border-radius: 10px;
    padding: 10px 12px;
    font-family: $sans;
    font-size: 1em;
    line-height: 1.55;
    color: $ink;
    background: $paper;
    resize: vertical;
    transition: border-color 0.15s ease, background 0.15s ease;

    &::placeholder { color: $ink-3; }
    &:focus { outline: none; border-color: $signal; background: $card; }
  }

  // ---- judge rail ---------------------------------------------------------
  &__judge {
    display: flex;
    flex-direction: column;
    min-height: 0;
    overflow: hidden;
    border-left: 1px solid $line;
    background: $card;
    padding: 26px 24px 40px;
  }

  &__judge-head { flex-shrink: 0; padding: 0 0 14px; border-bottom: 1px solid $line-2; }
  &__judge-head-row { position: relative; display: flex; align-items: center; justify-content: space-between; gap: 8px; }

  &__judge-title {
    display: flex;
    align-items: center;
    gap: 7px;
    font-family: $mono;
    font-size: 0.9231em;
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ink-3;
    svg { color: $signal; }
  }
  &__judge-sub { margin-top: 6px; font-size: 0.9231em; color: $ink-3; line-height: 1.45; }

  &__judge-info-btn {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 26px;
    height: 26px;
    border: 1px solid $line;
    border-radius: 8px;
    background: $paper;
    color: $ink-2;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $ink-3; color: $ink; }
  }
  &__judge-info-btn--on { border-color: $signal; color: $signal; background: $wash; }

  &__judge-info-popover {
    position: absolute;
    top: calc(100% + 8px);
    right: 0;
    z-index: 5;
    width: 240px;
    padding: 10px 12px;
    border: 1px solid $line;
    border-radius: 12px;
    background: $card;
    color: $ink-2;
    font-size: 0.9231em;
    line-height: 1.45;
    box-shadow: 0 24px 60px -20px rgba(20, 22, 27, 0.4);
    animation: ev-fade-in 0.15s ease both;
  }

  &__judge-search {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 8px;
    margin: 14px 0 0;
    padding: 8px 11px;
    border: 1.5px solid $line;
    border-radius: 10px;
    background: $paper;
    transition: border-color 0.15s ease, background 0.15s ease;

    &:focus-within { border-color: $signal; background: $card; }
    svg { flex-shrink: 0; color: $ink-3; }

    input {
      flex: 1;
      min-width: 0;
      border: 0;
      background: transparent;
      font-family: $sans;
      font-size: 1em;
      color: $ink;
      &:focus { outline: none; }
      &::placeholder { color: $ink-3; }
    }
  }

  &__judge-required {
    flex-shrink: 0;
    margin: 12px 0 0;
    padding: 9px 11px;
    border-radius: 10px;
    background: $danger-wash;
    color: $danger;
    font-size: 0.8846em;
    font-weight: 600;
    line-height: 1.4;
  }

  &__judge-scroll { flex: 1; min-height: 0; overflow-y: auto; padding: 12px 4px 4px; display: flex; flex-direction: column; gap: 10px; }

  &__judge-empty {
    margin: 10px;
    padding: 18px 12px;
    text-align: center;
    border: 1px dashed $line;
    border-radius: 12px;
    background: $paper;
    font-size: 0.9231em;
    color: $ink-3;
  }

  &__judge-row {
    @include sel-card;
    display: flex;
    align-items: center;
    gap: 10px;
    width: 100%;
    padding: 10px 12px 10px 13px;
    border-radius: 11px;

    &--on { @include sel-card-on; }
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
    font-family: $display;
    font-size: 1.0769em;
    font-weight: 650;
    color: $ink;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;

    @media (min-width: 1800px) { font-size: 1.2308em; }
  }
  &__judge-meta {
    font-family: $mono;
    font-size: 0.8462em;
    font-weight: 700;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $ink-3;

    @media (min-width: 1800px) { font-size: 0.9231em; }
  }
  &__judge-price {
    flex-shrink: 0;
    font-family: $mono;
    font-size: 0.9231em;
    font-weight: 700;
    color: $ink-3;

    @media (min-width: 1800px) { font-size: 1.0385em; }
  }

  // ========================================================================
  // Review step — Datasets stats grid + section heads
  // ========================================================================
  &__summary {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 1px;
    background: $line;
    border: 1px solid $line;
    border-radius: 14px;
    overflow: hidden;
  }

  &__summary-cell { background: $card; padding: 15px 16px; display: flex; flex-direction: column; gap: 6px; }

  &__summary-k {
    order: 2;
    display: flex;
    align-items: center;
    gap: 6px;
    font-family: $mono;
    font-size: 0.7692em;
    font-weight: 700;
    letter-spacing: 0.1em;
    text-transform: uppercase;
    color: $ink-3;
    svg { color: $signal; }
  }

  &__summary-v {
    order: 1;
    font-family: $display;
    font-size: 1.6923em;
    font-weight: 700;
    color: $ink;
    letter-spacing: -0.02em;
    line-height: 1;
  }
  &__summary-v--muted { color: $ink-3; }

  &__block { margin-top: 26px; }

  // section head: display title + mono count pill (Datasets section-head h3)
  &__block-title {
    display: flex;
    align-items: center;
    gap: 8px;
    font-family: $display;
    font-size: 1.1538em;
    font-weight: 700;
    color: $ink;
    margin-bottom: 12px;
    svg { color: $signal; }

    b {
      font-family: $mono;
      font-style: normal;
      font-size: 0.7333em;
      font-weight: 700;
      color: $ink-3;
      background: $paper;
      border: 1px solid $line;
      border-radius: 99px;
      padding: 2px 8px;
    }
  }

  &__rows { border: 1px solid $line; border-radius: 14px; background: $card; overflow: hidden; }

  &__row {
    display: flex;
    align-items: baseline;
    justify-content: space-between;
    gap: 16px;
    padding: 12px 16px;
    border-bottom: 1px solid $line-2;
    font-size: 1.0385em;
    &:last-child { border-bottom: 0; }

    span:first-child {
      font-family: $mono;
      font-size: 0.7692em;
      font-weight: 700;
      letter-spacing: 0.1em;
      text-transform: uppercase;
      color: $ink-3;
    }
    span:last-child { color: $ink; font-weight: 650; text-align: right; }
  }

  &__review-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(220px, 1fr)); gap: 10px; }

  &__review-card {
    display: flex;
    align-items: center;
    gap: 11px;
    padding: 12px 14px 12px 15px;
    border: 1px solid $line;
    border-left-width: 4px;
    border-left-color: $signal;
    border-radius: 13px;
    background: $card;
  }
  &__review-card-icon {
    flex-shrink: 0;
    width: 32px;
    height: 32px;
    border-radius: 9px;
    display: grid;
    place-items: center;
    background: $paper;
    border: 1px solid $line;
    color: $signal;
  }
  &__review-card-name { font-family: $display; font-size: 1.0769em; font-weight: 650; color: $ink; }
  &__review-card-sub {
    font-family: $mono;
    font-size: 0.8462em;
    font-weight: 700;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $ink-3;
    margin-top: 1px;
  }

  &__metric-tags { display: flex; flex-wrap: wrap; gap: 8px; }
  &__metric-tag {
    font-family: $mono;
    font-size: 0.8846em;
    font-weight: 700;
    color: #fff;
    background: $signal;
    border: 1px solid $signal;
    border-radius: 8px;
    padding: 6px 12px;
  }

  // dashed empty state (Datasets empty/single)
  &__empty {
    grid-column: 1 / -1;
    padding: 22px 20px;
    text-align: center;
    border: 1px dashed $line;
    border-radius: 12px;
    background: $card;
    color: $ink-3;
    font-size: 1.0385em;
    line-height: 1.5;
  }

  // inline error — Datasets modal-error
  &__error {
    margin-top: 16px;
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 10px 12px;
    border-radius: 10px;
    background: $danger-wash;
    color: $danger;
    font-size: 0.9231em;
    font-weight: 600;
    line-height: 1.5;
  }

  &__spin { animation: ev-spin 0.8s linear infinite; }
}

// ===========================================================================
// Preview drawer — Datasets slide-over (480px, right edge)
// ===========================================================================
.ev-preview-overlay {
  @include scale-root;
  position: fixed;
  inset: 0;
  z-index: 70;
  background: rgba(20, 22, 27, 0.32); // deliberately fixed — scrim stays dark in both themes
  display: flex;
  justify-content: flex-end;
  animation: ev-fade-in 0.22s ease both;
}

.ev-preview-panel {
  width: 480px;
  max-width: 92vw;
  height: 100%;
  background: $card;
  border-left: 1px solid $line;
  box-shadow: -18px 0 40px -20px rgba(20, 22, 27, 0.35);
  display: flex;
  flex-direction: column;
  min-height: 0;
  animation: ev-slide-in 0.26s cubic-bezier(0.22, 1, 0.36, 1) both;
}

.ev-preview-head {
  flex-shrink: 0;
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 12px;
  padding: 20px 20px 16px;
  border-bottom: 1px solid $line;
}

.ev-preview-head-main { min-width: 0; }

.ev-preview-eyebrow { @extend %micro; color: $signal; margin-bottom: 6px; }

.ev-preview-title {
  font-family: $display;
  font-size: 1.3846em;
  font-weight: 800;
  letter-spacing: -0.01em;
  color: $ink;
  line-height: 1.25;
  overflow-wrap: anywhere;
}

.ev-preview-close {
  flex-shrink: 0;
  width: 30px;
  height: 30px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 8px;
  border: 1px solid $line;
  background: $paper;
  color: $ink-2;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease;

  &:hover { border-color: $ink-3; color: $ink; }
}

// pagination bar (Datasets puts it at the foot; here it stays under the header)
.ev-preview-controls {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  flex-wrap: wrap;
  padding: 12px 20px;
  border-bottom: 1px solid $line;
  background: $paper;
}

.ev-preview-limit { min-width: 0; display: flex; align-items: center; gap: 10px; flex-wrap: wrap; }

.ev-preview-limit-label {
  font-family: $mono;
  font-size: 0.7692em;
  font-weight: 700;
  color: $ink;
  display: block;
}

.ev-preview-limit-range { font-weight: 500; color: $ink-3; }

.ev-preview-limit-controls { display: inline-flex; align-items: center; gap: 6px; }

.ev-preview-stepper-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 26px;
  height: 26px;
  border-radius: 8px;
  border: 1px solid $line;
  background: $card;
  color: $ink-2;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

  &:hover:not(:disabled) { border-color: $signal; color: $signal; background: $wash; }
  &:disabled { opacity: 0.4; cursor: not-allowed; }
}

.ev-preview-body {
  flex: 1;
  min-height: 0;
  overflow-y: auto;
  padding: 16px 20px 24px;
}

.ev-preview-skel-list,
.ev-preview-list { display: flex; flex-direction: column; gap: 12px; }

.ev-preview-skel-card {
  display: flex;
  flex-direction: column;
  gap: 9px;
  padding: 14px 16px;
  border: 1px solid $line;
  border-radius: 12px;
  background: $paper;
}

.ev-preview-q {
  border: 1px solid $line;
  border-radius: 12px;
  padding: 14px 16px;
  background: $paper;
}

.ev-preview-q-head { display: flex; align-items: center; flex-wrap: wrap; gap: 6px; margin-bottom: 10px; }

.ev-preview-q-index {
  margin-right: auto;
  font-family: $mono;
  font-weight: 700;
  font-size: 0.9231em;
  color: $ink-3;
  background: transparent;
  padding: 0;
}

.ev-preview-q-cat {
  font-family: $mono;
  font-size: 0.7692em;
  font-weight: 700;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  color: $signal;
  background: $wash;
  border-radius: 6px;
  padding: 3px 8px;
  white-space: nowrap;
}

// question / expected text — Datasets field-text box
.ev-preview-q-prompt,
.ev-preview-q-answer {
  font-family: $mono;
  font-size: 0.9231em;
  font-weight: 500;
  line-height: 1.5;
  color: $ink;
  white-space: pre-wrap;
  word-break: break-word;
  background: $card;
  border: 1px solid $line-2;
  border-radius: 8px;
  padding: 8px 10px;
}

.ev-preview-q-answer {
  margin-top: 10px;

  span {
    @extend %micro;
    font-size: 0.6923em;
    color: $ink-3;
    margin-right: 7px;
  }

  & + & { margin-top: 6px; }
}

.ev-preview-q-source {
  margin-top: 8px;
  font-size: 0.9231em;
  color: $ink-2;
  line-height: 1.4;

  span { @extend %micro; font-size: 0.6923em; color: $ink-3; margin-right: 6px; }
}

.ev-preview-q-images { margin-top: 10px; display: flex; flex-wrap: wrap; gap: 6px; }

.ev-preview-q-image-btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  max-width: 100%;
  padding: 6px 11px;
  border: 1px solid $line;
  border-radius: 999px;
  background: $card;
  color: $ink-2;
  font-family: $sans;
  font-size: 0.8462em;
  font-weight: 600;
  cursor: pointer;
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
  transition: border-color 0.14s ease, color 0.14s ease, background 0.14s ease;

  svg { flex-shrink: 0; color: $signal; }
  &:hover { border-color: $signal; color: $signal; background: $wash; }
}

.ev-preview-q-choices {
  margin-top: 10px;
  display: flex;
  flex-direction: column;
  gap: 5px;

  li {
    font-family: $mono;
    font-size: 0.9231em;
    color: $ink-2;
    padding: 6px 10px;
    border: 1px solid $line-2;
    border-radius: 8px;
    background: $card;
  }
}

// metadata — dashed divider + mono label/value (Datasets preview-meta)
.ev-preview-q-meta {
  margin-top: 10px;
  padding-top: 10px;
  border-top: 1px dashed $line-2;
  display: flex;
  flex-wrap: wrap;
  gap: 14px;
}

.ev-preview-q-meta-item {
  display: flex;
  flex-direction: column;
  gap: 3px;
  min-width: 0;
  font-family: $mono;
  font-size: 0.8462em;
  font-weight: 600;
  color: $ink-2;
  line-height: 1.5;
  overflow-wrap: anywhere;

  b {
    @extend %micro;
    font-size: 0.6154em;
    color: $ink-3;
  }
}

// ===========================================================================
// Image lightbox — Datasets "always dark" viewer stage
// ===========================================================================
.ev-imgview-overlay {
  @include scale-root;
  position: fixed;
  inset: 0;
  z-index: 80;
  background: rgba(10, 11, 14, 0.82);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 32px;
  animation: ev-fade-in 0.15s ease both;
}

.ev-imgview-panel {
  width: min(880px, 100%);
  max-height: 88vh;
  display: flex;
  flex-direction: column;
  background: #14161B;
  border-radius: 14px;
  overflow: hidden;
  box-shadow: 0 30px 70px -20px rgba(0, 0, 0, 0.6);
  animation: ev-toast-in 0.18s cubic-bezier(0.22, 1, 0.36, 1) both;
}

.ev-imgview-head {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 14px 12px 16px;
  background: rgba(255, 255, 255, 0.04);
  border-bottom: 1px solid rgba(255, 255, 255, 0.08);
}

.ev-imgview-path {
  flex: 1;
  min-width: 0;
  display: flex;
  align-items: center;
  gap: 8px;
  font-family: $mono;
  font-size: 0.8462em;
  font-weight: 600;
  color: rgba(255, 255, 255, 0.92);
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;

  svg { flex-shrink: 0; color: rgba(255, 255, 255, 0.55); }
}

.ev-imgview-count { flex-shrink: 0; font-family: $mono; font-size: 0.8462em; color: rgba(255, 255, 255, 0.55); }

.ev-imgview-close {
  flex-shrink: 0;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 30px;
  height: 30px;
  border-radius: 8px;
  border: 1px solid transparent;
  background: transparent;
  color: rgba(255, 255, 255, 0.7);
  cursor: pointer;
  transition: background 0.15s ease, color 0.15s ease;

  &:hover { background: rgba(255, 255, 255, 0.1); color: #fff; }
}

.ev-imgview-body {
  position: relative;
  flex: 1;
  min-height: 320px;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 24px;
  overflow: auto;
  // faint checker so transparent PNGs read against the dark stage
  background-image:
    linear-gradient(45deg, rgba(255, 255, 255, 0.03) 25%, transparent 25%),
    linear-gradient(-45deg, rgba(255, 255, 255, 0.03) 25%, transparent 25%),
    linear-gradient(45deg, transparent 75%, rgba(255, 255, 255, 0.03) 75%),
    linear-gradient(-45deg, transparent 75%, rgba(255, 255, 255, 0.03) 75%);
  background-size: 20px 20px;
  background-position: 0 0, 0 10px, 10px -10px, -10px 0;
}

.ev-imgview-img {
  display: block;
  max-width: 100%;
  max-height: 72vh;
  object-fit: contain;
  border-radius: 4px;
  box-shadow: 0 12px 30px -10px rgba(0, 0, 0, 0.5);
}

.ev-imgview-status {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
  color: rgba(255, 255, 255, 0.55);
  font-size: 0.9231em;
  text-align: center;
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
  background: rgba(255, 255, 255, 0.06);
  color: #fff;
  cursor: pointer;
  transition: background 0.15s ease, opacity 0.15s ease;

  &:hover:not(:disabled) { background: rgba(255, 255, 255, 0.14); }
  &:disabled { opacity: 0.3; cursor: not-allowed; }
}
.ev-imgview-nav--prev { left: 12px; }
.ev-imgview-nav--next { right: 12px; }

// ---- toast (fixed-dark chip) -----------------------------------------------
.ev-toast {
  @include scale-root;
  position: fixed;
  right: 24px;
  bottom: 24px;
  z-index: 90;
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 14px 18px 14px 14px;
  background: $ink-solid;
  color: #fff;
  border-radius: 14px;
  box-shadow: 0 24px 60px -20px rgba(20, 22, 27, 0.4);
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
  &__title { font-family: $display; font-weight: 700; font-size: 1.0385em; }
  &__sub { font-size: 0.9231em; color: rgba(255, 255, 255, 0.6); margin-top: 1px; }
}

// ---- keyframes --------------------------------------------------------------
@keyframes ev-spin { to { transform: rotate(360deg); } }
@keyframes ev-shimmer {
  from { background-position: 200% 0; }
  to { background-position: -200% 0; }
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
  from { opacity: 0; transform: translateY(6px); }
  to { opacity: 1; transform: none; }
}
@keyframes ev-fade-in { from { opacity: 0; } to { opacity: 1; } }
@keyframes ev-slide-in { from { transform: translateX(100%); } to { transform: translateX(0); } }
@keyframes ev-toast-in {
  from { opacity: 0; transform: translateY(8px) scale(0.98); }
  to { opacity: 1; transform: none; }
}

// ---- responsive (mirrors the Datasets breakpoints) --------------------------
// Not enough width for two usable columns — stack them (same as the
// Datasets split panel) and let the whole step scroll as one.
@media (max-width: 1180px) {
  .ev__suite { grid-template-columns: 1fr; grid-template-rows: none; margin: 0; }
  .ev__suite-scroll { padding-right: 6px; }
  .ev__suite + .ev__dataset-empty-warning { margin-top: 14px; }
  .ev__metrics { display: flex; flex-direction: column; gap: 0; margin: 0; }
  .ev__metrics > * { flex-shrink: 0; }
  .ev__metrics-main { overflow: visible; padding: 0 0 20px; }
  .ev__rail, .ev__judge { border-left: 0; border-top: 1px solid $line; padding: 20px 0 0; margin-top: 16px; }
  .ev__rail { max-height: 22rem; }
  .ev__judge { max-height: 26rem; }
}

@media (max-width: 1040px) {
  .ev__samples-instruction-row { grid-template-columns: 1fr; }
}

@media (max-width: 820px) {
  .ev__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  // rail stacks above the detail pane, like Datasets
  .ev__shell { grid-template-columns: 1fr; grid-template-rows: minmax(160px, 34vh) minmax(0, 1fr); padding-left: 0; }
  .ev__manifest { margin: 14px 14px 0; }
  .ev__stage-head { padding: 20px 18px 16px 36px; }
  .ev__stage-head::before { left: 18px; top: 24px; bottom: 20px; }
  .ev__stage-body { padding: 20px 18px 32px; }
  .ev__footer { padding: 14px 18px; }
  .ev__fw-grid { grid-template-columns: 1fr; }
  .ev__summary { grid-template-columns: repeat(2, 1fr); }
  .ev__hint { display: none; }
  .ev__step-toolbar { flex-direction: column; align-items: stretch; }
  .ev__toolbar-search { max-width: none; }
  .ev-preview-panel { width: 100%; max-width: 100vw; }
  .ev-imgview-overlay { padding: 12px; }
  .ev-imgview-panel { max-height: 92vh; }
  .ev-imgview-img { max-height: 60vh; }
}

@media (prefers-reduced-motion: reduce) {
  .ev-preview-overlay, .ev-preview-panel, .ev-imgview-overlay, .ev-imgview-panel { animation: none !important; }
  .ev__skel-block { animation: none !important; }
  .ev *, .ev-toast { animation: none !important; transition: none !important; }
}
