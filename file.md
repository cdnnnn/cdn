//Datasets.module.scss
@use 'sass:color';
@use '../../styles/_variables' as *;

// ===========================================================================
// Datasets (Test Suite Library) — master/detail library browser.
// Header + toolbar are unchanged from the original. The body below replaces
// the card grid + modal with a scrollable list rail and a rich detail pane.
// Uses the app's existing font variables ($font-mono / $font-body /
// $font-display) — no font-family is introduced or overridden here.
// ===========================================================================

// ---------------------------------------------------------------------------
// Color tokens: intentionally NOT redeclared here. $ink, $ink-2, $ink-3,
// $paper, $card, $line, $line-2, $signal, $signal-2, $wash, $ok, $danger
// already come from the shared "ink" system in _variables.scss via the
// `@use` above, where the neutrals resolve to the themed CSS custom
// properties in _theme.scss and the accents are flat constants shared
// across light/dark. Redeclaring them locally as flat hex (as this file
// previously did) would silently break dark-mode theming for this
// component while looking identical in light mode. If a color this
// component needs doesn't exist in _variables.scss yet, add it there
// (and to _theme.scss if it should be theme-aware) rather than
// hardcoding it here — that's the single source of truth for the ink
// system.
// ---------------------------------------------------------------------------

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

$soft: 0 1px 2px rgba(20, 22, 27, 0.05);

// Base font-size the whole Datasets component's internal `em` scale is
// built on. All descendant font-sizes in this file are expressed in `em`
// relative to this, so bumping it (e.g. on wide screens below) scales the
// whole component proportionally from one place — same convention as
// Model Catalog / Sidebar / Providers / History.
$datasets-base-font: 0.8125rem;

%micro {
  font-family: $mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

.datasets {
  // master scale control — every em-based font-size below responds to this
  font-size: $datasets-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  // ---- header (unchanged) ---------------------------------------------------
  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding: 24px 32px 20px;
    margin-bottom: 20px;
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

  &__header-eyebrow {
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

  &__header-sub {
    margin-top: 4px;
    font-size: 1.0385em; // 0.84375rem / 0.8125rem
    color: $ink-2;
    max-width: 52ch;
  }

  &__header-meta {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 10px;
    flex-wrap: wrap;
    margin-bottom: 3px;
  }

  &__header-count {
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
  }

  &__refresh-btn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 8px 13px;
    border: 1px solid $line;
    border-radius: 999px;
    background: $card;
    color: $ink-2;
    font-family: $sans;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $ink-3; color: $ink; background: $paper; }
  }

  &__upload-btn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 8px 14px;
    border: 1px solid transparent;
    border-radius: 999px;
    background: $signal;
    color: #fff;
    font-family: $sans;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: background 0.15s ease, transform 0.15s ease;

    &:hover { background: $signal-2; }
  }

  // ---- toolbar (unchanged) --------------------------------------------------
  &__toolbar {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 14px;
    padding: 14px 32px;
    background: $card;
    border-bottom: 1px solid $line;
    flex-wrap: wrap;
  }

  &__search {
    position: relative;
    flex: 1;
    max-width: 360px;
    min-width: 200px;

    svg {
      position: absolute;
      top: 50%;
      left: 13px;
      transform: translateY(-50%);
      color: $ink-3;
      pointer-events: none;
    }

    input {
      width: 100%;
      border: 1.5px solid $line;
      border-radius: 10px;
      padding: 9px 12px 9px 38px;
      font-size: 1.0385em; // 0.84375rem / 0.8125rem
      font-family: $sans;
      color: $ink;
      background: $paper;
      transition: border-color 0.15s ease, background 0.15s ease;

      &::placeholder { color: $ink-3; }
      &:focus { outline: none; border-color: $signal; background: $card; }
    }
  }

  &__toolbar-label {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 5px 8px 5px 9px;
    color: $ink-3;
  }

  &__filters {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    padding: 4px;
    background: $paper;
    border: 1px solid $line;
    border-radius: 999px;
    flex-wrap: wrap;
  }

  &__filter-pill {
    padding: 6px 13px;
    border: 0;
    border-radius: 999px;
    background: transparent;
    color: $ink-2;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: all 0.15s ease;

    &:hover { color: $ink; }

    &--on {
      background: $card;
      color: $signal;
      box-shadow: $soft;
    }
  }

  // ---- capability facet bar -------------------------------------------------
  &__facets {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 8px;
    flex-wrap: wrap;
    padding: 10px 32px;
    background: $wash;
    border-bottom: 1px solid $line;
    color: $signal-2;
  }

  &__facets-lead {
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 650;
    color: $ink-2;
  }

  &__facet {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 4px 6px 4px 10px;
    border: 0;
    border-radius: 999px;
    font-family: $mono;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    font-weight: 700;
    cursor: pointer;
    transition: filter 0.12s ease;

    &:hover { filter: brightness(0.96); }
  }

  &__facets-clear {
    margin-left: 2px;
    border: 0;
    background: none;
    color: $signal;
    font-family: $sans;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 700;
    cursor: pointer;

    &:hover { text-decoration: underline; }
  }

  // ---- body split -----------------------------------------------------------
  // NOTE: relies on the page shell (.pg-shell) being a flex column that gives
  // this element the remaining height. If your shell differs, set a height /
  // min-height on &__body instead of flex:1.
  &__body {
    flex: 1;
    min-height: 0;
    display: grid;
    grid-template-columns: minmax(320px, 380px) 1fr;
  }

  // ---- list rail ------------------------------------------------------------
  &__rail {
    display: flex;
    flex-direction: column;
    min-height: 0;
    border-right: 1px solid $line;
    background: $card;
  }

  &__rail-head {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 14px 20px;
    border-bottom: 1px solid $line-2;
    font-family: $mono;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ink-3;
  }

  &__rail-hint {
    display: inline-flex;
    align-items: center;
    gap: 5px;
  }

  &__rail-scroll {
    flex: 1;
    overflow-y: auto;
    padding: 12px;
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  &__row {
    text-align: left;
    width: 100%;
    cursor: pointer;
    background: $card;
    border: 1px solid $line;
    border-left-width: 4px;          // colored per type via inline borderLeftColor
    border-radius: 13px;
    padding: 14px 16px 14px 17px;
    display: flex;
    flex-direction: column;
    gap: 10px;
    font-family: $sans;
    transition: border-color 0.14s ease, box-shadow 0.14s ease, transform 0.14s ease, background 0.14s ease;

    &:hover {
      border-color: $ink-3;
      box-shadow: $soft;
      transform: translateY(-1px);
    }

    &--on {
      border-color: $signal;
      background: $wash;
      box-shadow: $soft;
    }
  }

  &__row-top {
    display: flex;
    align-items: baseline;
    justify-content: space-between;
    gap: 10px;
  }

  &__row-name {
    font-family: $display;
    font-size: 1.2308em; // 1rem / 0.8125rem
    font-weight: 650;
    color: $ink;
    line-height: 1.3;
  }

  &__row-count {
    font-family: $mono;
    font-size: 1.0769em; // 0.875rem / 0.8125rem
    font-weight: 700;
    color: $ink-3;
  }

  &__row-foot {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__row-type {
    font-family: $mono;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
  }

  &__row-dots {
    display: inline-flex;
    gap: 5px;
    flex-shrink: 0;

    i { width: 7px; height: 7px; border-radius: 99px; display: block; }
  }

  &__empty-rail {
    margin: auto;
    text-align: center;
    color: $ink-3;
    padding: 40px 16px;

    svg { margin-bottom: 8px; }
    p { font-size: 1em; /* 0.8125rem / 0.8125rem (base) */ line-height: 1.5; margin: 0; }
  }

  // ---- loading skeleton -----------------------------------------------------
  &__skel-row {
    padding: 15px 17px;
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  &__skel {
    display: block;
    height: 11px;
    border-radius: 6px;
    background: linear-gradient(90deg, $line-2 25%, $paper 50%, $line-2 75%);
    background-size: 200% 100%;
    animation: datasets-shimmer 1.2s ease-in-out infinite;
  }

  // ---- detail ---------------------------------------------------------------
  &__detail {
    min-height: 0;
    display: flex;
    flex-direction: column;
  }

  &__detail-scroll {
    flex: 1;
    overflow-y: auto;
    padding: 26px 30px 40px;
    animation: datasets-detail-in 0.22s ease;
  }

  &__detail-empty {
    margin: auto;
    text-align: center;
    color: $ink-3;
    max-width: 280px;

    svg { margin-bottom: 10px; }
    p { font-size: 1.0769em; /* 0.875rem / 0.8125rem */ line-height: 1.5; }
  }

  &__hero {
    position: relative;
    padding-left: 18px;
    margin-bottom: 22px;
  }

  &__hero-bar {
    position: absolute;
    left: 0;
    top: 4px;
    bottom: 4px;
    width: 4px;
    border-radius: 3px;
  }

  &__hero-top {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
  }

  &__hero-type {
    font-family: $mono;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.1em;
    text-transform: uppercase;
  }

  &__hero {
    h2 {
      font-family: $display;
      font-size: 1.8462em; // 1.5rem / 0.8125rem
      font-weight: 700;
      letter-spacing: -0.02em;
      margin: 5px 0 0;
      line-height: 1.15;
      color: $ink;
    }
  }

  &__hero-desc {
    margin: 14px 0 0;
    font-size: 1.1538em; // 0.9375rem / 0.8125rem
    line-height: 1.6;
    color: $ink-2;
  }

  &__source-badge {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    font-family: $mono;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $ink-2;
    padding: 5px 10px;
    border: 1px solid $line;
    border-radius: 999px;
    background: $paper;
    white-space: nowrap;
  }

  &__hero-actions {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 8px;
  }

  &__preview-btn {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 7px 14px;
    border-radius: 999px;
    border: 1px solid transparent;
    background: $signal;
    color: #fff;
    font-family: $sans;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: background 0.15s ease, transform 0.15s ease, box-shadow 0.15s ease;
    box-shadow: 0 4px 12px -6px rgba(43, 43, 245, 0.6);

    &:hover { background: $signal-2; transform: translateY(-1px); }
  }

  &__delete-btn {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 30px;
    height: 30px;
    border-radius: 999px;
    border: 1px solid $line;
    background: $paper;
    color: $ink-3;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover { border-color: rgba($danger, 0.3); color: $danger; background: $danger-wash; }
  }

  &__stats {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 1px;
    background: $line;
    border: 1px solid $line;
    border-radius: 14px;
    overflow: hidden;
    margin-bottom: 26px;
  }

  &__stat {
    background: $card;
    padding: 15px 16px;
    display: flex;
    flex-direction: column;
    gap: 3px;
  }

  &__stat-val {
    font-family: $display;
    font-size: 1.6923em; // 1.375rem / 0.8125rem
    font-weight: 700;
    color: $ink;
    letter-spacing: -0.02em;
    line-height: 1;

    &--mono { font-family: $mono; font-size: 1.1538em; /* 0.9375rem / 0.8125rem */ font-weight: 700; }
  }

  &__stat-label {
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.1em;
    text-transform: uppercase;
    color: $ink-3;
  }

  &__section {
    margin-bottom: 26px;
  }

  &__section-head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
    margin-bottom: 12px;

    h3 {
      font-family: $display;
      font-size: 1.1538em; // 0.9375rem / 0.8125rem
      font-weight: 700;
      color: $ink;
      margin: 0;
      display: flex;
      align-items: center;
      gap: 8px;

      em {
        font-family: $mono;
        font-style: normal;
        font-size: 0.8462em; // 0.6875rem / 0.8125rem
        font-weight: 700;
        color: $ink-3;
        background: $paper;
        border: 1px solid $line;
        border-radius: 99px;
        padding: 2px 8px;
      }
    }
  }

  &__section-hint {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    font-size: 0.8846em; // 0.71875rem / 0.8125rem
    color: $ink-3;
  }

  &__caps {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
  }

  &__cap {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 6px 12px;
    border: 1px solid;
    border-radius: 8px;
    font-family: $mono;
    font-size: 0.8846em; // 0.71875rem / 0.8125rem
    font-weight: 700;
    cursor: pointer;
    transition: transform 0.13s ease;

    &:hover { transform: translateY(-1px); }

    &--active { box-shadow: $soft; }
  }

  &__single {
    display: flex;
    gap: 11px;
    align-items: flex-start;
    padding: 14px 16px;
    border: 1px dashed $line;
    border-radius: 12px;
    background: $paper;
    color: $ink-3;

    strong { display: block; font-size: 1em; /* 0.8125rem / 0.8125rem (base) */ color: $ink; font-weight: 650; }
    span { display: block; font-size: 0.9615em; /* 0.78125rem / 0.8125rem */ color: $ink-2; margin-top: 2px; }
  }

  // ---- state banner (error) -------------------------------------------------
  &__state {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 12px;
    padding: 48px 24px;
    margin: 24px 32px;
    border: 1px dashed $line;
    border-radius: 16px;
    background: $paper;
    color: $ink-2;
    font-size: 1.0769em; // 0.875rem / 0.8125rem
    text-align: center;

    svg { color: $ink-3; }
  }

  &__state--error svg { color: $danger; }

  // ---- upload dataset modal --------------------------------------------------
  &__modal-overlay {
    position: fixed;
    inset: 0;
    z-index: 60;
    background: rgba(20, 22, 27, 0.45);
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 24px;
    animation: datasets-fade-in 0.15s ease;
  }

  &__modal {
    width: min(460px, 100%);
    max-height: 88vh;
    display: flex;
    flex-direction: column;
    background: $card;
    border: 1px solid $line;
    border-radius: 18px;
    box-shadow: 0 24px 60px -20px rgba(20, 22, 27, 0.4);
    overflow: hidden;
    animation: datasets-modal-in 0.18s cubic-bezier(0.22, 1, 0.36, 1);
  }

  &__modal-hdr {
    flex-shrink: 0;
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
    padding: 20px 20px 16px;
    border-bottom: 1px solid $line;

    h3 {
      font-family: $display;
      font-size: 1.3846em; // 1.125rem / 0.8125rem
      font-weight: 700;
      color: $ink;
      margin: 3px 0 0;
    }
  }

  &__modal-eyebrow {
    @extend %micro;
    color: $signal;

    &--danger { color: $danger; }
  }

  &__modal-close {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 28px;
    height: 28px;
    border-radius: 8px;
    border: 1px solid $line;
    background: $paper;
    color: $ink-2;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease;

    &:hover:not(:disabled) { border-color: $ink-3; color: $ink; }
    &:disabled { opacity: 0.4; cursor: not-allowed; }
  }

  &__modal-body {
    flex: 1;
    overflow-y: auto;
    padding: 18px 20px 4px;
    display: flex;
    flex-direction: column;
    gap: 16px;
  }

  &__field {
    display: flex;
    flex-direction: column;
    gap: 7px;
  }

  &__field-label {
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ink-3;
  }

  &__field-input,
  &__field-textarea {
    width: 100%;
    border: 1.5px solid $line;
    border-radius: 10px;
    padding: 9px 12px;
    font-size: 1em; /* 0.8125rem / 0.8125rem (base) */
    font-family: $sans;
    color: $ink;
    background: $paper;
    transition: border-color 0.15s ease, background 0.15s ease;
    resize: none;

    &::placeholder { color: $ink-3; }
    &:focus { outline: none; border-color: $signal; background: $card; }
    &:disabled { opacity: 0.6; cursor: not-allowed; }
  }

  &__field-note {
    font-size: 0.8846em; // 0.71875rem / 0.8125rem
    color: $ink-3;
    line-height: 1.5;
  }

  &__eval-pills {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    padding: 4px;
    background: $paper;
    border: 1px solid $line;
    border-radius: 999px;
    width: fit-content;
  }

  &__eval-pill {
    padding: 6px 16px;
    border: 0;
    border-radius: 999px;
    background: transparent;
    color: $ink-2;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: all 0.15s ease;

    &:hover:not(:disabled) { color: $ink; }
    &:disabled { cursor: not-allowed; opacity: 0.6; }

    &--on {
      background: $card;
      color: $signal;
      box-shadow: $soft;
    }
  }

  &__dropzone {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 4px;
    width: 100%;
    padding: 20px 16px;
    border: 1.5px dashed $line;
    border-radius: 12px;
    background: $paper;
    color: $ink-2;
    cursor: pointer;
    text-align: center;
    transition: border-color 0.15s ease, background 0.15s ease;

    &:hover:not(:disabled) { border-color: $signal; background: $wash; }
    &:disabled { opacity: 0.6; cursor: not-allowed; }

    svg { color: $ink-3; margin-bottom: 2px; }
  }

  &__dropzone-text {
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 650;
    color: $ink;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    max-width: 100%;
  }

  &__dropzone-hint {
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    color: $ink-3;
  }

  &__file-input {
    display: none;
  }

  &__modal-error {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 10px 12px;
    border-radius: 10px;
    background: $danger-wash;
    color: $danger;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 600;

    svg { flex-shrink: 0; }
  }

  &__modal-success {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 10px 12px;
    border-radius: 10px;
    background: $ok-wash;
    color: $ok;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 600;

    svg { flex-shrink: 0; }
  }

  &__modal-foot {
    flex-shrink: 0;
    display: flex;
    justify-content: flex-end;
    gap: 10px;
    padding: 16px 20px 20px;
  }

  &__modal-cancel {
    padding: 9px 16px;
    border: 1px solid $line;
    border-radius: 10px;
    background: $card;
    color: $ink-2;
    font-family: $sans;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover:not(:disabled) { border-color: $ink-3; color: $ink; background: $paper; }
    &:disabled { opacity: 0.5; cursor: not-allowed; }
  }

  &__modal-submit {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    padding: 9px 18px;
    border: 1px solid transparent;
    border-radius: 10px;
    background: $signal;
    color: #fff;
    font-family: $sans;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: background 0.15s ease;

    &:hover:not(:disabled) { background: $signal-2; }
    &:disabled { opacity: 0.6; cursor: not-allowed; }
  }

  &__modal-danger {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    padding: 9px 18px;
    border: 1px solid transparent;
    border-radius: 10px;
    background: $danger;
    color: #fff;
    font-family: $sans;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: background 0.15s ease;

    &:hover:not(:disabled) { background: color.scale($danger, $lightness: -8%); }
    &:disabled { opacity: 0.6; cursor: not-allowed; }
  }

  &__delete-copy {
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    line-height: 1.6;
    color: $ink-2;

    strong { color: $ink; font-weight: 700; }
  }

  &__spin {
    animation: datasets-spin 0.8s linear infinite;
  }

  // ---- question preview slide-over drawer ------------------------------------
  // Same pattern as History's .drawer-overlay/.drawer: a fade-only scrim
  // plus a fixed panel pinned to the right edge that translates in on open.
  &__preview-overlay {
    position: fixed;
    inset: 0;
    // Deliberately hardcoded, not themed — an overlay scrim must stay dark
    // in both themes; the themed $ink would go near-white in dark mode.
    background: rgba(20, 22, 27, 0.32);
    opacity: 0;
    pointer-events: none;
    transition: opacity 0.22s ease;
    z-index: 55;

    &--open { opacity: 1; pointer-events: auto; }
  }

  &__preview-drawer {
    position: fixed;
    top: 0;
    right: 0;
    height: 100%;
    width: 480px;
    max-width: 92vw;
    background: $card;
    border-left: 1px solid $line;
    box-shadow: -18px 0 40px -20px rgba(20, 22, 27, 0.35);
    transform: translateX(100%);
    transition: transform 0.26s cubic-bezier(0.22, 1, 0.36, 1);
    z-index: 56;
    display: flex;
    flex-direction: column;
    min-height: 0;

    &--open { transform: translateX(0); }
  }

  &__preview-header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
    padding: 20px 20px 16px;
    border-bottom: 1px solid $line;
  }

  &__preview-eyebrow {
    @extend %micro;
    color: $signal;
    margin-bottom: 6px;
  }

  &__preview-title {
    font-family: $display;
    font-size: 1.3846em; // 1.125rem / 0.8125rem
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $ink;
  }

  &__preview-sub {
    margin-top: 5px;
    font-family: $mono;
    font-size: 0.8846em; // 0.71875rem / 0.8125rem
    color: $ink-3;
  }

  &__preview-close {
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
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $ink-3; color: $ink; }
  }

  &__preview-body {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding: 16px 20px 24px;
    display: flex;
    flex-direction: column;
    gap: 12px;
  }

  &__preview-loading {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
    padding: 40px 16px;
    color: $ink-3;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    text-align: center;
  }

  &__preview-card {
    border: 1px solid $line;
    border-radius: 12px;
    padding: 14px 16px;
    background: $paper;
  }

  &__preview-card-hdr {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
    margin-bottom: 10px;
  }

  &__preview-card-num {
    font-family: $mono;
    font-weight: 700;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    color: $ink-3;
  }

  &__preview-card-cat {
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $signal;
    background: $wash;
    border-radius: 6px;
    padding: 3px 8px;
    white-space: nowrap;
  }

  &__preview-field {
    min-width: 0;

    & + & { margin-top: 10px; }
  }

  &__preview-field-label {
    @extend %micro;
    font-size: 0.6923em; // 0.5625rem / 0.8125rem
    color: $ink-3;
    display: block;
    margin-bottom: 4px;
  }

  &__preview-field-text {
    font-family: $mono;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    line-height: 1.5;
    color: $ink;
    white-space: pre-wrap;
    word-break: break-word;
    background: $card;
    border: 1px solid $line-2;
    border-radius: 8px;
    padding: 8px 10px;

    &--empty {
      color: $ink-3;
      font-style: italic;
      font-family: $sans;
    }
  }

  // ---- preview pagination bar -------------------------------------------------
  &__preview-pagination {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
    flex-wrap: wrap;
    padding: 12px 20px;
    border-top: 1px solid $line;
    background: $paper;
  }

  &__preview-pagination-info {
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    color: $ink-3;
    white-space: nowrap;
  }

  &__preview-pagination-controls {
    display: flex;
    align-items: center;
    gap: 6px;
  }

  &__preview-size-select {
    appearance: none;
    -webkit-appearance: none;
    font: inherit;
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 650;
    color: $ink-2;
    background: $card;
    border: 1px solid $line;
    border-radius: 8px;
    padding: 5px 22px 5px 9px;
    cursor: pointer;
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='10' height='6' viewBox='0 0 10 6' fill='none'%3E%3Cpath d='M1 1L5 5L9 1' stroke='%238A909B' stroke-width='1.5' stroke-linecap='round' stroke-linejoin='round'/%3E%3C/svg%3E");
    background-repeat: no-repeat;
    background-position: right 8px center;

    &:hover { border-color: $ink-3; }
    &:focus { outline: none; border-color: $signal; box-shadow: 0 0 0 3px $wash; }
    &:disabled { opacity: 0.5; cursor: not-allowed; }
  }

  &__preview-nav-btn {
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

  &__preview-page-label {
    min-width: 44px;
    text-align: center;
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 700;
    color: $ink;
  }
}

@keyframes datasets-shimmer {
  from { background-position: 200% 0; }
  to { background-position: -200% 0; }
}

@keyframes datasets-detail-in {
  from { opacity: 0; transform: translateY(6px); }
  to { opacity: 1; transform: none; }
}

@keyframes datasets-fade-in {
  from { opacity: 0; }
  to { opacity: 1; }
}

@keyframes datasets-modal-in {
  from { opacity: 0; transform: translateY(8px) scale(0.98); }
  to { opacity: 1; transform: none; }
}

@keyframes datasets-spin {
  to { transform: rotate(360deg); }
}

@media (max-width: 820px) {
  .datasets__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .datasets__toolbar { padding: 12px 18px; }
  .datasets__facets { padding: 10px 18px; }
  .datasets__body { grid-template-columns: 1fr; grid-template-rows: minmax(180px, 34vh) 1fr; }
  .datasets__rail { border-right: 0; border-bottom: 1px solid $line; }
  .datasets__detail-scroll { padding: 20px 18px 32px; }
  .datasets__stats { grid-template-columns: repeat(2, 1fr); }
  .datasets__hero h2 { font-size: 1.5385em; /* 1.25rem / 0.8125rem */ }
  .datasets__modal { width: 100%; max-width: 100%; border-radius: 16px; }
  .datasets__modal-overlay { padding: 14px; }
  .datasets__preview-drawer { width: 100%; max-width: 100vw; }
}

@media (prefers-reduced-motion: reduce) {
  .datasets__skel,
  .datasets__detail-scroll,
  .datasets__modal-overlay,
  .datasets__modal,
  .datasets__spin { animation: none; }
  .datasets__row,
  .datasets__cap,
  .datasets__upload-btn,
  .datasets__dropzone,
  .datasets__eval-pill,
  .datasets__modal-submit,
  .datasets__modal-danger,
  .datasets__delete-btn,
  .datasets__preview-btn,
  .datasets__preview-drawer,
  .datasets__preview-overlay,
  .datasets__preview-nav-btn,
  .datasets__modal-cancel { transition: none; }
}
