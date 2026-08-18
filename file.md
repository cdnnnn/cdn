@use '../../styles/_variables' as *;

// ===========================================================================
// Model Catalog — matches the Run Console / Dashboard / Providers design
// system: ink/paper palette, ultramarine signal accent, mono instrument
// labels, hover-lift cards, mono numerals for data-dense cells.
//
// Neutral/status tokens ($ink, $paper, $card, $line, $signal, $wash, $ok,
// $ink-wash, etc.) come from the shared "ink" block in _variables.scss
// (theme-aware via _theme.scss custom properties) — only font aliases and
// shadow tokens specific to this file are declared locally below.
//
// Font scaling: `.model-catalog` sets a single base font-size. All
// descendant font-sizes are expressed in `em` (relative to that base), so
// bumping `.model-catalog`'s font-size (e.g. on wide screens) scales the
// whole component proportionally from one place — same convention as
// Sidebar and Providers.
// ===========================================================================

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

$soft: 0 1px 2px rgba(20, 22, 27, 0.05);
$lift: 0 14px 30px -14px rgba(20, 22, 27, 0.22);

// base font-size the model catalog's internal `em` scale is built on
$model-catalog-base-font: 0.8125rem;

%micro {
  font-family: $mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

.model-catalog {
  // master scale control — every em-based font-size below responds to this
  font-size: $model-catalog-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  // ---- header -------------------------------------------------------------
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
  }

  &__header-meta {
    flex-shrink: 0;
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
    margin-bottom: 3px;
  }

  // ---- toolbar --------------------------------------------------------------
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
    max-width: 340px;
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
      &:focus {
        outline: none;
        border-color: $signal;
        background: $card;
      }
    }
  }

  &__filter-group {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    padding: 4px;
    background: $paper;
    border: 1px solid $line;
    border-radius: 999px;
    flex-wrap: wrap;
  }

  &__toolbar-label {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 5px 10px 5px 11px;
    @extend %micro;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    color: $ink-3;
    white-space: nowrap;
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

  &__loading {
    display: flex;
    align-items: center;
    gap: 8px;
    color: $ink-2;
    font-size: 1em; // 0.8125rem / 0.8125rem (base)
    margin-bottom: 16px;
  }

  // ---- table shell ----------------------------------------------------------
  &__table-wrap {
    border: 1px solid $line;
    border-radius: 16px;
    background: $card;
    overflow: hidden;
  }

  &__table {
    width: 100%;
    border-collapse: collapse;
    font-size: 1.0385em; // 0.84375rem / 0.8125rem

    thead th {
      text-align: left;
      background: $paper;
      border-bottom: 1px solid $line;
      @extend %micro;
      font-size: 0.7692em; // 0.625rem / 0.8125rem
      color: $ink-3;
      padding: 12px 16px;
      white-space: nowrap;
    }

    tbody tr {
      border-bottom: 1px solid $line-2;
      transition: background 0.13s ease;

      &:last-child { border-bottom: 0; }
      &:hover { background: $paper; }
    }

    tbody td {
      padding: 13px 16px;
      color: $ink;
      vertical-align: middle;
    }
  }

  &__name-cell {
    font-family: $display;
    font-weight: 700;
    color: $ink;
  }

  &__provider-cell {
    color: $ink-2;
  }

  &__caps-cell {
    display: flex;
    flex-wrap: wrap;
    gap: 5px;
  }

  &__tag {
    font-family: $mono;
    font-size: 0.8077em; // 0.65625rem / 0.8125rem
    font-weight: 600;
    letter-spacing: 0.02em;
    color: $ink-2;
    background: $paper;
    border: 1px solid $line;
    border-radius: 6px;
    padding: 2px 7px;
    white-space: nowrap;
  }

  &__mono-cell {
    font-family: $mono;
    font-size: 1em; // 0.8125rem / 0.8125rem (base)
    color: $ink;
  }

  &__mono-cell--muted {
    color: $ink-2;
  }

  &__accuracy {
    font-family: $mono;
    font-weight: 700;
    font-size: 1em; // 0.8125rem / 0.8125rem (base)
    color: $ink;

    &--high { color: $ok; }
  }

  &__status {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 3px 10px 3px 8px;
    border-radius: 999px;
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;

    &::before { content: ''; width: 5px; height: 5px; border-radius: 50%; }

    &--active {
      color: $ok;
      background: $ok-wash;
      &::before { background: $ok; }
    }

    &--inactive {
      color: $ink-3;
      background: $ink-wash;
      &::before { background: $ink-3; }
    }
  }

  // --- sortable column headers ----------------------------------------------
  &__sortable-th {
    padding: 0 !important;
  }

  &__sort-btn {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    width: 100%;
    padding: 12px 16px;
    background: none;
    border: none;
    cursor: pointer;
    font: inherit;
    font-family: $mono;
    font-weight: 700;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    letter-spacing: 0.14em;
    text-transform: uppercase;
    color: $ink-3;
    text-align: left;
    transition: color 0.15s ease;

    &:hover {
      color: $ink;

      .model-catalog__sort-icon-idle { opacity: 0.7; }
    }

    &--active {
      color: $signal;
    }
  }

  &__sort-icon-idle {
    opacity: 0.28;
    transition: opacity 0.15s ease;
  }

  &__empty {
    text-align: center;
    padding: 44px 16px !important;
    color: $ink-3;
    font-size: 1.0385em; // 0.84375rem / 0.8125rem
  }

  // --- pagination bar ---------------------------------------------------------
  &__pagination {
    display: flex;
    align-items: center;
    justify-content: space-between;
    flex-wrap: wrap;
    gap: 12px;
    padding: 14px 20px;
    border-top: 1px solid $line;
    background: $paper;
  }

  &__pagination-info {
    display: flex;
    align-items: center;
    flex-wrap: wrap;
    gap: 18px;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    color: $ink-2;

    strong { color: $ink; font-weight: 700; }
  }

  &__page-size {
    display: flex;
    align-items: center;
    gap: 8px;

    label {
      font-size: 0.8846em; // 0.71875rem / 0.8125rem
      color: $ink-3;
      white-space: nowrap;
    }

    select {
      appearance: none;
      -webkit-appearance: none;
      font: inherit;
      font-size: 0.9615em; // 0.78125rem / 0.8125rem
      font-weight: 650;
      color: $ink;
      background: $card;
      border: 1px solid $line;
      border-radius: 8px;
      padding: 5px 26px 5px 10px;
      cursor: pointer;
      background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='10' height='6' viewBox='0 0 10 6' fill='none'%3E%3Cpath d='M1 1L5 5L9 1' stroke='%23565B66' stroke-width='1.5' stroke-linecap='round' stroke-linejoin='round'/%3E%3C/svg%3E");
      background-repeat: no-repeat;
      background-position: right 10px center;

      &:hover { border-color: $ink-3; }
      &:focus { outline: none; border-color: $signal; box-shadow: 0 0 0 3px $wash; }
    }
  }

  &__pager {
    display: flex;
    align-items: center;
    gap: 4px;
  }

  &__page-btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    min-width: 30px;
    height: 30px;
    padding: 0 6px;
    border-radius: 8px;
    border: 1px solid transparent;
    background: transparent;
    color: $ink-2;
    font-family: $mono;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: background 0.14s ease, color 0.14s ease;

    &:hover:not(:disabled) {
      background: $wash;
      color: $signal;
    }

    &:disabled {
      opacity: 0.35;
      cursor: not-allowed;
    }

    &--num {
      min-width: 30px;
    }

    &--active {
      background: $signal;
      color: #fff;

      &:hover:not(:disabled) {
        background: $signal;
        color: #fff;
      }
    }
  }

  &__page-dots {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    min-width: 20px;
    height: 30px;
    color: $ink-3;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
  }
}

@media (max-width: 768px) {
  .model-catalog__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .model-catalog__toolbar { padding: 12px 18px; }
}
