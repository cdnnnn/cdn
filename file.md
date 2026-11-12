@use '../../styles/_variables' as *;

// ===========================================================================
// HarnessScorecard — detail view for a Document-agent harness run, shown in
// History's detail panel. Same "ink" tokens and em-based type scale as the
// New run wizard (NewEvaluation.module.scss); nothing is redeclared here.
// ===========================================================================

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

.hs {
  font-size: 0.8125rem; // base every em below is relative to
  display: flex;
  flex-direction: column;
  gap: 26px;
  margin-top: 20px;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  &__state {
    display: flex;
    align-items: center;
    gap: 8px;
    margin-top: 20px;
    padding: 18px;
    border: 1px dashed $line;
    border-radius: 12px;
    background: $paper;
    color: $ink-2;
    font-size: 0.875rem;
  }

  &__spin { animation: hs-spin 0.8s linear infinite; }

  // ---- section ------------------------------------------------------------
  &__sec { display: flex; flex-direction: column; gap: 12px; }

  &__sec-head {
    display: flex;
    align-items: baseline;
    justify-content: space-between;
    flex-wrap: wrap;
    gap: 6px 16px;

    h3 { font-family: $display; font-size: 1.1538em; font-weight: 800; letter-spacing: -0.01em; color: $ink; }
    span { font-size: 0.9615em; color: $ink-2; }
  }

  // ---- agent cards --------------------------------------------------------
  &__agents { display: grid; gap: 12px; }

  &__agent {
    padding: 16px 18px;
    border: 1px solid $line;
    border-radius: 12px;
    background: $card;

    &--best { border-color: $signal; background: $wash; }
  }

  &__agent-top { display: flex; align-items: center; justify-content: space-between; gap: 10px; }
  &__agent-name { font-family: $display; font-size: 1.1538em; font-weight: 700; color: $ink; }

  &__badge {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 3px 9px;
    border-radius: 999px;
    background: $signal;
    color: #fff;
    font-size: 0.8462em;
    font-weight: 650;
    white-space: nowrap;
  }

  &__agent-rate {
    margin-top: 10px;
    font-family: $display;
    font-size: 2.4615em;
    font-weight: 800;
    letter-spacing: -0.03em;
    line-height: 1;
    color: $ink;
  }
  &__agent-label { margin-top: 6px; font-size: 0.9615em; color: $ink-2; }

  &__bar { margin-top: 12px; height: 6px; border-radius: 999px; background: $line; overflow: hidden; }
  &__bar-fill { height: 100%; border-radius: 999px; background: $signal; }

  // ---- tables -------------------------------------------------------------
  &__table-wrap { overflow-x: auto; border: 1px solid $line; border-radius: 12px; }

  &__table {
    width: 100%;
    border-collapse: collapse;
    font-size: 1em;

    th {
      padding: 10px 14px;
      background: $paper;
      border-bottom: 1px solid $line;
      text-align: left;
      font-family: $sans;
      font-size: 0.9231em;
      font-weight: 650;
      color: $ink-2;
      white-space: nowrap;
      vertical-align: bottom;

      small { display: block; font-weight: 500; color: $ink-3; }
    }

    td {
      padding: 11px 14px;
      border-bottom: 1px solid $line-2;
      color: $ink;
      vertical-align: middle;

      b { display: block; font-weight: 700; }
      small { display: block; font-size: 0.9231em; color: $ink-2; }
    }
    tbody tr:last-child td { border-bottom: 0; }

    th.hs__num,
    td.hs__num { text-align: right; white-space: nowrap; }
    td.hs__num { font-family: $mono; font-size: 0.9615em; }
    td.hs__num--best { font-weight: 700; color: $signal; }
    td.hs__num--strong { font-weight: 700; }
    td.hs__cell-agent { vertical-align: top; border-right: 1px solid $line-2; }
    td.hs__none { text-align: center; color: $ink-2; }

    // clickable rows (trace store)
    &--rows tbody tr { cursor: pointer; transition: background 0.12s ease; }
    &--rows tbody tr:hover td { background: $paper; }
    &--rows tbody tr.hs__row--open td { background: $wash; border-bottom-color: transparent; }
    &--rows tbody tr.hs__row--trace { cursor: default; }
    &--rows tbody tr.hs__row--trace td,
    &--rows tbody tr.hs__row--trace:hover td { background: $wash; padding-top: 0; }
  }

  // success-rate pill, three tiers (legend under the table)
  &__rate {
    display: inline-block;
    min-width: 46px;
    padding: 3px 8px;
    border-radius: 6px;
    text-align: center;
    font-family: $mono;
    font-size: 0.9615em;
    font-weight: 700;

    &--high { color: $ok; background: $ok-wash; }
    &--mid { color: $ink-2; background: $paper; }
    &--low { color: $danger; background: $danger-wash; }
  }

  &__legend { display: flex; flex-wrap: wrap; gap: 8px; }

  // ---- error bars ---------------------------------------------------------
  &__errbar { display: flex; align-items: center; gap: 10px; }
  &__errbar-track { flex: 1; min-width: 60px; height: 8px; border-radius: 999px; background: $line-2; overflow: hidden; }
  &__errbar-fill { display: block; height: 100%; border-radius: 999px; background: $danger; }
  &__errbar-n { min-width: 22px; text-align: right; font-family: $mono; font-size: 0.9615em; font-weight: 700; color: $ink; }

  // ---- trace store --------------------------------------------------------
  &__filters { display: flex; flex-wrap: wrap; align-items: center; gap: 8px; }
  &__filters-sep { width: 1px; height: 18px; background: $line; }

  &__chip {
    padding: 6px 12px;
    border: 1.5px solid $line;
    border-radius: 999px;
    background: $card;
    font-family: $sans;
    font-size: 0.9615em;
    font-weight: 650;
    color: $ink-2;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $ink-3; color: $ink; }
    &--on { border-color: $signal; background: $wash; color: $signal; }
  }

  &__toggle {
    width: 34px;

    button {
      display: inline-flex;
      padding: 2px;
      border: 0;
      background: transparent;
      color: $ink-2;
      cursor: pointer;
    }
  }

  &__result {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 3px 9px;
    border-radius: 999px;
    font-size: 0.9231em;
    font-weight: 650;
    white-space: nowrap;

    &--pass { color: $ok; background: $ok-wash; }
    &--fail { color: $danger; background: $danger-wash; }
  }

  &__steps {
    display: flex;
    flex-direction: column;
    gap: 6px;
    padding: 4px 0 12px;
    list-style: none;

    li {
      display: grid;
      grid-template-columns: 24px minmax(120px, 190px) minmax(0, 1fr) auto;
      align-items: center;
      gap: 10px;
      padding: 7px 10px;
      border: 1px solid $line;
      border-radius: 8px;
      background: $card;
      font-size: 0.9615em;
      color: $ink;
    }

    code { font-family: $mono; font-weight: 700; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
  }
  &__step-n { font-family: $mono; font-size: 0.9231em; color: $ink-3; }
  &__step-sum { color: $ink-2; }
  &__step-ms { font-family: $mono; font-size: 0.9231em; color: $ink-3; }
  &__steps li.hs__step--fail { border-color: rgba($danger, 0.35); }
  &__step--fail &__step-sum { color: $danger; font-weight: 650; }
}

@keyframes hs-spin { to { transform: rotate(360deg); } }

@media (max-width: 900px) {
  .hs__agents { grid-template-columns: 1fr !important; }
}

@media (prefers-reduced-motion: reduce) {
  .hs * { animation: none !important; transition: none !important; }
}
