@use '../../styles/_variables' as *;

// ===========================================================================
// Comparison — mirrors History/Reports/Providers/Model Catalog design
// system: ink/paper palette, ultramarine signal accent, mono instrument
// labels, hover-lift, mono numerals.
//
// Font scaling: `.comparison` sets a single base font-size. All descendant
// font-sizes are expressed in `em` (relative to that base), so bumping
// `.comparison`'s font-size (e.g. on wide screens) scales the whole
// component proportionally from one place — same convention as
// Model Catalog / Sidebar / Providers. IMPORTANT: for this to take effect,
// the `.comparison` class must be applied on an actual DOM ancestor of the
// component's markup (see Comparison.tsx's root element).
//
// Theming: all neutral/status tokens below resolve to CSS custom
// properties defined in _theme.scss (via _variables.scss), so this file
// is fully dark-mode aware. Only the flat brand accents ($signal, $ok,
// $amber, $danger) stay constant across themes, per the ink design
// system convention — see _theme.scss's header comment.
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

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

// Previously hardcoded as rgba(20, 22, 27, ...), which doesn't adapt in
// dark mode. Now sourced from _theme.scss so these shadows stay correct
// across themes — see --ink-shadow-soft / --ink-shadow-lift there.
$soft: var(--ink-shadow-soft);
$lift: var(--ink-shadow-lift);

$comparison-base-font: 0.8125rem;

%micro {
  font-family: $mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

.comparison {
  // master scale control — every em-based font-size below responds to this
  font-size: $comparison-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  &__header {
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

  &__controls {
    display: flex;
    align-items: center;
    gap: 10px;
    flex-wrap: wrap;
    margin-top: 20px;
    margin-bottom: 20px;
  }

  &__label {
    @extend %micro;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    color: $ink-3;
  }

  &__grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 16px;
    margin-bottom: 20px;
  }

  &__panel-title {
    font-family: $display;
    font-size: 1.0769em; // 0.875rem / 0.8125rem
    font-weight: 700;
    color: $ink;
  }

  &__panel-sub {
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    color: $ink-3;
    margin-top: 2px;
    margin-bottom: 16px;
  }

  &__legend {
    display: flex;
    gap: 14px;
    justify-content: center;
    margin-top: 12px;
    font-family: $mono;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    font-weight: 700;
    color: $ink-2;
  }

  &__dot {
    display: inline-block;
    width: 8px;
    height: 8px;
    border-radius: 50%;
    margin-right: 5px;
  }

  &__scores {
    display: flex;
    gap: 32px;
    flex-wrap: wrap;
    justify-content: center;
  }

  &__score-item {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 8px;
  }

  &__hint {
    margin-top: 8px;
    font-family: $mono;
    font-size: 0.8846em; // 0.71875rem / 0.8125rem
    color: $ink-3;

    &--error { color: $danger; }
  }
}

.score-item__name { font-family: $display; font-weight: 700; font-size: 1.0769em; color: $ink; text-align: center; } // 0.875rem / 0.8125rem
.score-item__meta { font-family: $mono; font-size: 0.9231em; color: $ink-3; } // 0.75rem / 0.8125rem

// ---- shared card/panel (replaces global .card) -----------------------------
.panel {
  padding: 20px 24px;
  background: $card;
  border: 1px solid $line;
  border-radius: 16px;
  box-shadow: $soft;

  &--flush { padding: 0; }
}

.panel-title--spaced { margin-bottom: 20px; }

.table-title {
  font-family: $display;
  font-size: 1.0769em; // 0.875rem / 0.8125rem
  font-weight: 700;
  color: $ink;
  padding: 20px 24px;
  border-bottom: 1px solid $line;
}

.radar-wrap {
  display: flex;
  justify-content: center;
  padding: 8px 0;
}

.loading-wrap { margin-top: 20px; }

// ---- model chips (comparing summary) ---------------------------------------
.model-chip {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  font-family: $mono;
  font-size: 0.8846em; // 0.71875rem / 0.8125rem
  font-weight: 700;
  border: 1px solid;
  border-radius: 999px;
  padding: 5px 10px;

  &__dot {
    width: 7px;
    height: 7px;
    border-radius: 50%;
    display: inline-block;
  }
}

// ---- model select grid ------------------------------------------------------
.model-select-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(160px, 1fr));
  gap: 8px;
  margin-top: 14px;
  margin-bottom: 16px;
}

.model-select-item {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 12px;
  border: 1.5px solid $line;
  border-radius: 10px;
  background: $paper;
  font-size: 1em; // 0.8125rem / 0.8125rem (base)
  font-weight: 650;
  color: $ink;
  cursor: pointer;
  text-align: left;
  transition: border-color 0.15s ease, background 0.15s ease, box-shadow 0.15s ease;

  &:hover { border-color: $ink-3; }

  &.active {
    background: $card;
    box-shadow: $soft;
  }

  &__check {
    width: 16px;
    height: 16px;
    border-radius: 4px;
    border: 1.5px solid $line;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-shrink: 0;
    color: inherit;
  }
}

.compare-btn {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 9px 16px;
  border-radius: 10px;
  border: 1px solid $signal;
  background: $signal;
  color: #fff;
  font-family: $mono;
  font-size: 0.9231em; // 0.75rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.03em;
  cursor: pointer;
  transition: background 0.15s ease, border-color 0.15s ease, opacity 0.15s ease;

  &:hover:not(:disabled) { background: $signal-2; border-color: $signal-2; }

  &:disabled {
    opacity: 0.4;
    cursor: not-allowed;
  }
}

.empty {
  padding: 24px;
  text-align: center;
  color: $ink-3;
  font-size: 1em; // 0.8125rem / 0.8125rem (base)
}

// ---------------------------------------------------------------------------
// Empty state shown before a benchmark is selected.
// ---------------------------------------------------------------------------
.empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  padding: 56px 32px;
  border: 1px dashed $line;
  border-radius: 16px;
  background: $paper;

  &__icon {
    width: 56px;
    height: 56px;
    border-radius: 16px;
    background: $wash;
    color: $signal;
    display: flex;
    align-items: center;
    justify-content: center;
    margin-bottom: 18px;
  }

  h3 {
    font-family: $display;
    font-size: 1.2308em; // 1rem / 0.8125rem
    font-weight: 700;
    color: $ink;
    margin-bottom: 8px;
  }

  p {
    max-width: 420px;
    font-size: 1em; // 0.8125rem / 0.8125rem (base)
    line-height: 1.6;
    color: $ink-2;
    margin-bottom: 24px;
  }

  &__stats {
    display: flex;
    gap: 12px;
    flex-wrap: wrap;
    justify-content: center;
  }

  &__stat {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    color: $ink-2;
    background: $card;
    border: 1px solid $line;
    border-radius: 999px;
    padding: 8px 14px;

    svg { color: $signal; flex-shrink: 0; }

    strong {
      color: $ink;
      font-weight: 700;
    }
  }
}

// ---- results table (shared visual grammar with History/Reports) -----------
.results-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 1.0385em; // 0.84375rem / 0.8125rem

  thead th {
    text-align: left;
    background: $paper;
    border-bottom: 1px solid $line;
    @extend %micro;
    font-size: 0.6923em; // 0.5625rem / 0.8125rem
    color: $ink-3;
    padding: 11px 14px;
    white-space: nowrap;
  }

  tbody tr {
    border-bottom: 1px solid $line-2;
    transition: background 0.13s ease;

    &:last-child { border-bottom: 0; }
    &:hover { background: $paper; }
  }

  tbody td {
    padding: 12px 14px;
    color: $ink;
  }
}

.cell-model { font-family: $display; font-weight: 700; }
.cell-provider { color: $ink-2; }
.cell-num { font-family: $mono; font-size: 1em; font-weight: 700; color: $ink; } // 0.8125rem / 0.8125rem
.cell-num--muted { font-weight: 500; color: $ink-2; }
.cell-pass { font-family: $mono; font-size: 1em; font-weight: 700; color: $ok; } // 0.8125rem / 0.8125rem
.cell-fail { font-family: $mono; font-size: 1em; font-weight: 700; color: $danger; } // 0.8125rem / 0.8125rem

// ---------------------------------------------------------------------------
// Skeleton loader
// ---------------------------------------------------------------------------
@keyframes comparison-skeleton-pulse {
  0%, 100% { opacity: 0.55; }
  50% { opacity: 1; }
}

.skeletonLine,
.skeletonCircle {
  background: $line-2;
  border-radius: 8px;
  animation: comparison-skeleton-pulse 1.3s ease-in-out infinite;
}

.skeletonLine--title { width: 40%; height: 16px; margin-bottom: 20px; }
.skeletonLine--row { width: 100%; height: 32px; margin-bottom: 8px; }

.skeletonCircle {
  width: 100px;
  height: 100px;
  border-radius: 50%;
  margin: 20px auto 0;
}

@media (max-width: 900px) {
  .comparison__grid {
    grid-template-columns: 1fr;
  }
}

@media (max-width: 640px) {
  .comparison__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
}
