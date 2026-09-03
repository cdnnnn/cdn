@use '../../styles/_variables' as *;

// ===========================================================================
// History — matches the Run Console / Dashboard / Providers / Model Catalog /
// Datasets design system: ink/paper palette, ultramarine signal accent, mono
// instrument labels, hover-lift, mono numerals. Master–detail split shell is
// self-contained here (no dependency on global .split-shell*).
// ===========================================================================

// $ink, $ink-2, $ink-3, $paper, $card, $line, $line-2, $signal, $signal-2,
// $wash, $ok, $ok-wash, $danger, $danger-wash, $ink-wash all come from
// _variables.scss (imported above via `@use ... as *`) — they already
// resolve to theme-aware CSS custom properties, so History picks up dark
// mode automatically without redeclaring anything here.
//
// $amber / $amber-wash aren't part of the shared "ink" names (the shared
// tokens use $amber-ink / $amber-ink-wash to avoid clashing with the
// brand-palette $amber further up _variables.scss), so alias them locally
// rather than renaming every usage in this file.
$amber:      $amber-ink;
$amber-wash: $amber-ink-wash;

// $soft/$lift now point at the shared theme-aware shadow tokens instead of
// a hardcoded ink-black rgba — so card shadows get properly darker in
// dark mode instead of staying a flat, barely-visible tint.
$soft: $shadow-2;
$lift: $shadow-3;

// Base font-size the whole History component's internal `em` scale is built
// on. All descendant font-sizes in this file are expressed in `em` relative
// to this, so bumping it (e.g. on wide screens below) scales the whole
// component proportionally from one place — same convention as Model
// Catalog / Sidebar / Providers.
$history-base-font: 0.8125rem;

%micro {
  font-family: $font-mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

.history {
  // master scale control — every em-based font-size below responds to this
  font-size: $history-base-font;

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
    margin-bottom: 0;
    border-bottom: 1px solid $line;
    background: $card;

    h1 {
      font-family: $font-display;
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
    font-family: $font-mono;
    font-size: 0.8846em; // 0.71875rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ink-2;
    white-space: nowrap;
    margin-bottom: 3px;
  }
}

@property --angle {
  syntax: '<angle>';
  initial-value: 0deg;
  inherits: false;
}
@keyframes history-rotate-angle {
  to { --angle: 360deg; }
}
@keyframes history-live-pulse {
  0%, 100% { opacity: 1; transform: scale(1); }
  50% { opacity: 0.5; transform: scale(1.3); }
}
@keyframes history-spin { to { transform: rotate(360deg); } }
@keyframes history-row-delete-pulse {
  0%, 100% { box-shadow: 0 0 0 0 rgba($danger, 0.18); }
  50% { box-shadow: 0 0 0 5px rgba($danger, 0.08); }
}
@keyframes history-row-delete-sheen {
  0% { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}

// Fixed-shell override: list + detail scroll independently, so pg-body
// itself must not scroll — plain flex:1/min-height:0 pass-through.
.pg-body-fixed {
  overflow: hidden;
  display: flex;
  flex-direction: column;
  padding: 20px 32px 24px;
}

// ---- self-contained split shell -------------------------------------------
.shell {
  flex: 1;
  min-height: 0;
  display: flex;
  gap: 16px;
}

.sidebar {
  flex-shrink: 0;
  width: 380px;
  display: flex;
  flex-direction: column;
  min-height: 0;
  padding: 16px;
  background: $card;
  border: 1px solid $line;
  border-radius: 16px;
  box-shadow: $soft;
}

.detail {
  flex: 1;
  min-width: 0;
  min-height: 0;
  overflow-y: auto;
  padding: 24px;
  background: $card;
  border: 1px solid $line;
  border-radius: 16px;
  box-shadow: $soft;
}

.filters { flex-shrink: 0; }

// ---- filter toolbar --------------------------------------------------------
.filter-toolbar {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 6px 8px;
  margin-bottom: 8px;
  background: $paper;
  border: 1px solid $line;
  border-radius: 12px;
}

.filter-toolbar__label {
  flex-shrink: 0;
  @extend %micro;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  color: $ink-3;
  padding-left: 4px;
}

.filter-toolbar__divider {
  flex-shrink: 0;
  width: 1px;
  height: 16px;
  background: $line;
}

.filter-toolbar__btn {
  position: relative;
  flex-shrink: 0;
  width: 30px;
  height: 30px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 8px;
  border: 1px solid transparent;
  background: transparent;
  color: $ink-2;
  cursor: pointer;
  transition: background 0.15s ease, color 0.15s ease, border-color 0.15s ease, box-shadow 0.15s ease;

  &:hover { background: $card; color: $signal; box-shadow: $soft; }

  &.on {
    border-color: $signal;
    background: $card;
    color: $signal;
    box-shadow: 0 0 0 3px $wash;
  }
}

.filter-toolbar__dot {
  position: absolute;
  top: -2px;
  right: -2px;
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: $signal;
  border: 1.5px solid $paper;
}

.filter-toolbar__summary {
  display: flex;
  align-items: center;
  gap: 6px;
  flex-wrap: wrap;
  min-width: 0;
  overflow: hidden;
}

.filter-chip {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  font-family: $font-mono;
  font-size: 0.8077em; // 0.65625rem / 0.8125rem
  font-weight: 700;
  color: $signal;
  background: $wash;
  border: 1px solid rgba($signal, 0.18);
  border-radius: 999px;
  padding: 4px 8px;
  white-space: nowrap;
  max-width: 140px;

  span { overflow: hidden; text-overflow: ellipsis; }

  svg {
    cursor: pointer;
    flex-shrink: 0;
    opacity: 0.6;
    transition: opacity 0.15s ease;
    &:hover { opacity: 1; }
  }
}

// ---- collapsible filter panel ----------------------------------------------
.filter-panel {
  display: grid;
  grid-template-rows: 0fr;
  opacity: 0;
  transition: grid-template-rows 0.18s ease, opacity 0.15s ease, margin-bottom 0.18s ease;

  > * {
    overflow: hidden;
    min-height: 0;
    background: $paper;
    border: 1px solid $line;
    border-radius: 12px;
    padding: 10px;
  }

  &--open {
    grid-template-rows: 1fr;
    opacity: 1;
    margin-bottom: 8px;
  }
}

// ---- filter panel inner controls (self-contained search + pills) -----------
.panel-search {
  position: relative;

  svg {
    position: absolute;
    top: 50%;
    left: 12px;
    transform: translateY(-50%);
    color: $ink-3;
    pointer-events: none;
  }

  input {
    width: 100%;
    border: 1.5px solid $line;
    border-radius: 9px;
    padding: 8px 11px 8px 36px;
    font-size: 1em; // 0.8125rem / 0.8125rem
    font-family: $font-body;
    color: $ink;
    background: $card;

    &::placeholder { color: $ink-3; }
    &:focus { outline: none; border-color: $signal; box-shadow: 0 0 0 3px $wash; }
  }
}

.panel-pills {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.panel-pill {
  padding: 6px 12px;
  border: 1px solid $line;
  border-radius: 999px;
  background: $card;
  color: $ink-2;
  font-size: 0.9231em; // 0.75rem / 0.8125rem
  font-weight: 650;
  cursor: pointer;
  transition: all 0.14s ease;

  &:hover { border-color: $ink-3; color: $ink; }

  &.on {
    border-color: $signal;
    background: $signal;
    color: #fff;
  }
}

.empty {
  padding: 24px;
  text-align: center;
  color: $ink-3;
  font-size: 1em; // 0.8125rem / 0.8125rem
  display: flex;
  align-items: center;
  gap: 8px;
  justify-content: center;
}

// ---- true empty state (no evaluations at all yet) --------------------
.sidebar-empty {
  flex: 1;
  min-height: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  text-align: center;
  gap: 4px;
  padding: 40px 24px;
}
.sidebar-empty__icon {
  width: 52px;
  height: 52px;
  border-radius: 16px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: $wash;
  color: $signal;
  margin-bottom: 14px;
}
.sidebar-empty__title {
  font-family: $font-display;
  font-size: 1.1538em; // 0.9375rem / 0.8125rem
  font-weight: 800;
  letter-spacing: -0.01em;
  color: $ink;
}
.sidebar-empty__sub {
  max-width: 240px;
  margin-top: 6px;
  font-size: 0.9615em; // 0.78125rem / 0.8125rem
  line-height: 1.55;
  color: $ink-3;
}
.sidebar-empty__cta {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  margin-top: 18px;
  padding: 9px 16px;
  border-radius: 10px;
  border: 1px solid transparent;
  background: $signal;
  color: #fff;
  font-family: $font-mono;
  font-size: 0.8846em; // 0.71875rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.03em;
  cursor: pointer;
  transition: background 0.15s ease, transform 0.15s ease;

  &:hover { background: $signal-2; transform: translateY(-1px); }
}

// ---- evaluation list rows --------------------------------------------------
.rows {
  flex: 1;
  min-height: 0;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin-top: 14px;
  padding-right: 2px;
}

// ---- pagination bar (bottom of sidebar) ------------------------------------
.pagination {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  flex-wrap: wrap;
  margin-top: 12px;
  padding-top: 12px;
  border-top: 1px solid $line;
}

.pagination__info {
  font-family: $font-mono;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  color: $ink-3;
  white-space: nowrap;
}

.pagination__controls {
  display: flex;
  align-items: center;
  gap: 6px;
}

.pagination__size-select {
  appearance: none;
  -webkit-appearance: none;
  font: inherit;
  font-family: $font-mono;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  font-weight: 650;
  color: $ink-2;
  background: $paper;
  border: 1px solid $line;
  border-radius: 8px;
  padding: 5px 22px 5px 9px;
  cursor: pointer;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='10' height='6' viewBox='0 0 10 6' fill='none'%3E%3Cpath d='M1 1L5 5L9 1' stroke='%238A909B' stroke-width='1.5' stroke-linecap='round' stroke-linejoin='round'/%3E%3C/svg%3E");
  background-repeat: no-repeat;
  background-position: right 8px center;

  &:hover { border-color: $ink-3; }
  &:focus { outline: none; border-color: $signal; box-shadow: 0 0 0 3px $wash; }
}

.pagination__nav-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 26px;
  height: 26px;
  border-radius: 8px;
  border: 1px solid $line;
  background: $paper;
  color: $ink-2;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

  &:hover:not(:disabled) { border-color: $signal; color: $signal; background: $wash; }

  &:disabled {
    opacity: 0.4;
    cursor: not-allowed;
  }
}

.pagination__page-label {
  min-width: 44px;
  text-align: center;
  font-family: $font-mono;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  font-weight: 700;
  color: $ink;
}

.row {
  position: relative;
  border: 1.5px solid $line;
  border-radius: 14px;
  padding: 14px;
  cursor: pointer;
  background: $card;
  overflow: hidden;
  // A concrete max-height (comfortably above real card content, ~130-160px)
  // is required for the collapse transition in .row--exiting below to
  // animate at all — CSS can't transition to/from `auto`.
  max-height: 260px;
  transition: border-color 0.15s ease, box-shadow 0.15s ease, transform 0.15s ease, background 0.15s ease,
    max-height 0.32s ease, opacity 0.28s ease, padding 0.32s ease, margin 0.32s ease;
}
.row:hover { border-color: $ink-3; box-shadow: $soft; transform: translateY(-1px); }
.row.selected { border-color: $signal; background: $wash; box-shadow: 0 0 0 1px $signal inset; }

// Running-state: a multi-color light traveling around the border via a
// rotating conic angle.
.row--running {
  --angle: 0deg;
  border: 1.5px solid transparent;
  background:
    linear-gradient($card, $card) padding-box,
    conic-gradient(
      from var(--angle),
      $line 0%,
      $signal 4%,
      $violet-ink 8%,
      $rose-ink 12%,
      $line 18%
    ) border-box;
  animation: history-rotate-angle 2.4s linear infinite;
}
.row--running.selected {
  background:
    linear-gradient($wash, $wash) padding-box,
    conic-gradient(
      from var(--angle),
      $line 0%,
      $signal 4%,
      $violet-ink 8%,
      $rose-ink 12%,
      $line 18%
    ) border-box;
}

// Running/deleting cards aren't clickable — the Stop/Delete confirm flow is
// the only interactive element on them, so the card itself shouldn't hint
// at being clickable.
.row--unselectable {
  cursor: default;

  &:hover { transform: none; }
}

// Delete in flight: a red pulsing border + light sheen sweep, same pattern
// as the running-state animation above but in the danger color, so it
// visually reads as "something destructive is happening" rather than
// "in progress" — while the DELETE request is out.
.row--deleting {
  position: relative;
  border-color: rgba($danger, 0.35);
  animation: history-row-delete-pulse 1.2s ease-in-out infinite;

  &::after {
    content: '';
    position: absolute;
    inset: 0;
    background: linear-gradient(100deg, transparent 30%, rgba($danger, 0.1) 50%, transparent 70%);
    background-size: 200% 100%;
    animation: history-row-delete-sheen 1.2s ease-in-out infinite;
    pointer-events: none;
  }

  // Dim everything except the delete button itself (which shows its own
  // spinner) so attention stays on the destructive action in progress.
  > *:not(.row__badges) {
    opacity: 0.55;
    transition: opacity 0.2s ease;
  }
}

// Delete succeeded: collapse the card away instead of having it vanish
// the instant Redux removes it from the list — opacity + scale + the
// max-height/padding collapse declared on .row above animate together.
.row--exiting {
  opacity: 0;
  transform: scale(0.97);
  max-height: 0;
  padding-top: 0;
  padding-bottom: 0;
  border-width: 0;
  pointer-events: none;
}

.row__stop-btn {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  margin-left: auto; // pushes it to the far right of the badges row
  padding: 3px 9px 3px 7px;
  border-radius: 999px;
  border: 1px solid rgba($danger, 0.3);
  background: $card;
  color: $danger;
  font-family: $font-mono;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.02em;
  white-space: nowrap;
  cursor: pointer;
  transition: border-color 0.15s ease, background 0.15s ease, color 0.15s ease, transform 0.15s ease, box-shadow 0.15s ease;

  &:hover:not(:disabled) {
    background: $danger;
    border-color: $danger;
    color: #fff;
    box-shadow: 0 0 0 3px rgba($danger, 0.12);
  }

  &:active:not(:disabled) { transform: scale(0.96); }

  &:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }
}

// Icon-only, ghost by default — deliberately quieter than the Stop pill
// since delete is available on every non-running card and shouldn't compete
// visually with the status badge next to it. Only turns red on hover/focus.
.row__delete-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  margin-left: auto; // pushes it to the far right of the badges row
  width: 22px;
  height: 22px;
  border-radius: 7px;
  border: 1px solid transparent;
  background: transparent;
  color: $ink-3;
  cursor: pointer;
  transition: border-color 0.15s ease, background 0.15s ease, color 0.15s ease, transform 0.15s ease;

  &:hover:not(:disabled) {
    border-color: rgba($danger, 0.3);
    background: $danger-wash;
    color: $danger;
  }

  &:active:not(:disabled) { transform: scale(0.94); }

  &:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
}


.row__top { display: flex; align-items: center; gap: 10px; margin-bottom: 8px; }
.row__icon {
  width: 30px;
  height: 30px;
  border-radius: 9px;
  background: $wash;
  color: $signal;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}
.row__name {
  font-family: $font-display;
  font-weight: 700;
  font-size: 1.0769em; // 0.875rem / 0.8125rem
  color: $ink;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.row__badges { display: flex; align-items: center; gap: 6px; margin-bottom: 6px; flex-wrap: wrap; }
.row__meta { font-family: $font-mono; font-size: 0.8462em; /* 0.6875rem / 0.8125rem */ color: $ink-3; margin-bottom: 8px; }
.row__stats {
  display: flex;
  gap: 12px;
  font-family: $font-mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  color: $ink-2;
  flex-wrap: wrap;
}

// ---- type tag + status badge (shared visual grammar) -----------------------
.type-tag {
  display: inline-flex;
  align-items: center;
  font-family: $font-mono;
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

.status {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 3px 10px 3px 9px;
  border-radius: 999px;
  font-family: $font-mono;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.06em;
  text-transform: uppercase;

  &::before { content: ''; width: 5px; height: 5px; border-radius: 50%; }

  &--completed { color: $ok; background: $ok-wash; &::before { background: $ok; } }
  &--running   { color: $signal; background: $wash; &::before { display: none; } }
  &--pending   { color: $amber; background: $amber-wash; &::before { background: $amber; } }
  &--failed,
  &--canceled  { color: $ink-3; background: $ink-wash; &::before { background: $ink-3; } }
}

.live-dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: currentColor;
  display: inline-block;
  animation: history-live-pulse 1.4s ease-in-out infinite;
}

// ---- detail panel ----------------------------------------------------------
.detail-empty {
  padding: 80px 24px;
  text-align: center;
  color: $ink-3;
  font-size: 1.0769em; // 0.875rem / 0.8125rem
}

.detail-hdr {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 24px;
  flex-wrap: wrap;
  gap: 16px;
}
.detail-hdr__badges { display: flex; gap: 8px; margin-bottom: 12px; }
.detail-hdr__name {
  font-family: $font-display;
  font-size: 1.6923em; // 1.375rem / 0.8125rem
  font-weight: 800;
  letter-spacing: -0.02em;
  color: $ink;
}
.detail-hdr__date { font-family: $font-mono; font-size: 0.8846em; /* 0.71875rem / 0.8125rem */ color: $ink-3; margin-top: 6px; }
.detail-hdr__actions {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

.dl-btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 7px 12px;
  border-radius: 9px;
  border: 1px solid $line;
  background: $paper;
  color: $ink-2;
  font-family: $font-mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.03em;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

  &:hover:not(:disabled) { border-color: $signal; color: $signal; background: $wash; }

  &:disabled {
    opacity: 0.45;
    cursor: not-allowed;
  }
}

.summary-cards {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 12px;
  margin-bottom: 24px;
}
.summary-card {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 15px 16px;
  background: $paper;
  border: 1px solid $line;
  border-radius: 14px;
}
.summary-card__icon {
  flex-shrink: 0;
  width: 34px;
  height: 34px;
  border-radius: 10px;
  display: grid;
  place-items: center;
  background: $card;
  border: 1px solid $line;

  &--win { color: $amber; }
  &--info { color: $signal; }
  &--status { color: $ok; }
}
.summary-card__label {
  @extend %micro;
  font-size: 0.6923em; // 0.5625rem / 0.8125rem
  color: $ink-3;
}
.summary-card__val {
  font-size: 1em; // 0.8125rem / 0.8125rem
  font-weight: 700;
  color: $ink;
  margin-top: 3px;
}

// ---- run configuration panel (metrics_config — dynamic, may be partial) ---
.config-panel {
  margin-bottom: 16px;
  padding: 14px 16px;
  background: $paper;
  border: 1px solid $line;
  border-radius: 12px;
}
.config-panel__title {
  @extend %micro;
  font-size: 0.6923em; // 0.5625rem / 0.8125rem
  color: $signal;
  margin-bottom: 10px;
}
.config-panel__grid {
  display: flex;
  flex-wrap: wrap;
  gap: 18px;
}
.config-panel__item {
  display: flex;
  flex-direction: column;
  gap: 3px;
  min-width: 0;
}
.config-panel__label {
  @extend %micro;
  font-size: 0.6923em; // 0.5625rem / 0.8125rem
  color: $ink-3;
}
.config-panel__val {
  font-family: $font-mono;
  font-size: 0.9231em; // 0.75rem / 0.8125rem
  font-weight: 600;
  color: $ink;
}
.config-panel__models {
  margin-top: 12px;
  padding-top: 12px;
  border-top: 1px solid $line-2;
}
.config-panel__model-list {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-top: 8px;
}
.config-panel__model-row {
  display: inline-flex;
  align-items: center;
  gap: 10px;
  padding: 6px 12px;
  background: $card;
  border: 1px solid $line-2;
  border-radius: 999px;
}
.config-panel__model-name {
  font-family: $font-display;
  font-weight: 700;
  font-size: 0.9231em; // 0.75rem / 0.8125rem
  color: $ink;
  white-space: nowrap;
}
.config-panel__model-vals {
  display: flex;
  align-items: center;
  gap: 10px;
  padding-left: 10px;
  border-left: 1px solid $line-2;
  font-family: $font-mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  color: $ink-2;
  white-space: nowrap;
}

// ---- results metadata strip (benchmark / dataset / started / metrics) ------
.meta-strip {
  display: flex;
  flex-wrap: wrap;
  gap: 20px;
  padding: 12px 16px;
  margin-bottom: 16px;
  background: $paper;
  border: 1px solid $line;
  border-radius: 12px;
}
.meta-strip__item {
  display: flex;
  flex-direction: column;
  gap: 3px;
  min-width: 0;
}
.meta-strip__label {
  @extend %micro;
  font-size: 0.6923em; // 0.5625rem / 0.8125rem
  color: $ink-3;
}
.meta-strip__val {
  font-family: $font-mono;
  font-size: 0.9231em; // 0.75rem / 0.8125rem
  font-weight: 600;
  color: $ink;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.meta-strip__chips {
  display: flex;
  flex-wrap: wrap;
  gap: 5px;
}

// ---- test-details launcher button (in results table) ------------------
.cell-details { width: 32px; text-align: center; }
.details-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 26px;
  height: 26px;
  border-radius: 7px;
  border: 1px solid $line;
  background: $paper;
  color: $ink-2;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

  &:hover { border-color: $signal; color: $signal; background: $wash; }
}

// ---- test-details slide-over drawer ------------------------------------
.drawer-overlay {
  position: fixed;
  inset: 0;
  // Deliberately hardcoded, not themed: an overlay scrim must stay dark
  // in both themes (same "always dark" reasoning as $ink-solid in
  // _theme.scss) — using the themed $ink here would go near-white, and
  // invisible, in dark mode.
  background: rgba(20, 22, 27, 0.32);
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.22s ease;
  z-index: 40;

  &--open { opacity: 1; pointer-events: auto; }
}

.drawer {
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
  z-index: 41;
  display: flex;
  flex-direction: column;
  min-height: 0;

  &--open { transform: translateX(0); }
}

.drawer__header {
  flex-shrink: 0;
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 12px;
  padding: 20px 20px 16px;
  border-bottom: 1px solid $line;
}
.drawer__eyebrow {
  @extend %micro;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  color: $signal;
  margin-bottom: 6px;
}
.drawer__title {
  font-family: $font-display;
  font-size: 1.3846em; // 1.125rem / 0.8125rem
  font-weight: 800;
  letter-spacing: -0.01em;
  color: $ink;
}
.drawer__sub {
  margin-top: 5px;
  font-family: $font-mono;
  font-size: 0.8846em; // 0.71875rem / 0.8125rem
  color: $ink-3;
}
.drawer__close {
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

// ---- tab switcher (test details vs metric scores) --------------------
.drawer__tabs {
  flex-shrink: 0;
  display: flex;
  gap: 6px;
  padding: 10px 20px 0;
  border-bottom: 1px solid $line;
  background: $card;
}
.drawer__tab {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 8px 12px;
  border: 1px solid transparent;
  border-bottom: 2px solid transparent;
  border-radius: 8px 8px 0 0;
  background: transparent;
  color: $ink-3;
  font-family: $font-mono;
  font-size: 0.8846em; // 0.71875rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.02em;
  cursor: pointer;
  transition: color 0.15s ease, border-color 0.15s ease;
  margin-bottom: -1px;

  &:hover:not(:disabled) { color: $ink-2; }

  &.on {
    color: $signal;
    border-bottom-color: $signal;
  }

  &:disabled {
    opacity: 0.35;
    cursor: not-allowed;
  }
}

.drawer__stats {
  flex-shrink: 0;
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 14px;
  padding: 12px 20px;
  border-bottom: 1px solid $line;
  background: $paper;
  font-family: $font-mono;
  font-size: 0.8846em; // 0.71875rem / 0.8125rem
  font-weight: 700;
  color: $ink-2;
}
.drawer__stat {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  text-transform: capitalize;
}
.drawer__stat-icon--pass { color: $ok; }
.drawer__stat-icon--fail { color: $danger; }

.drawer__body {
  flex: 1;
  min-height: 0;
  overflow-y: auto;
  padding: 16px 20px 24px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

// ---- metric-score bars (Answer Relevancy, Toxicity, Bias, ...) -------
.metric-card {
  border: 1px solid $line;
  border-radius: 12px;
  padding: 13px 16px;
  background: $paper;
}
.metric-card__hdr {
  display: flex;
  align-items: baseline;
  justify-content: space-between;
  gap: 10px;
  margin-bottom: 9px;
}
.metric-card__label {
  font-family: $font-display;
  font-weight: 700;
  font-size: 1em; // 0.8125rem / 0.8125rem
  color: $ink;
  text-transform: capitalize;
}
.metric-card__value {
  font-family: $font-mono;
  font-size: 1em; // 0.8125rem / 0.8125rem
  font-weight: 800;
  flex-shrink: 0;

  &--good { color: $ok; }
  &--mid  { color: $amber; }
  &--low  { color: $danger; }
}
.metric-card__track {
  height: 7px;
  border-radius: 999px;
  background: $line-2;
  overflow: hidden;
}
.metric-card__fill {
  height: 100%;
  border-radius: 999px;
  transition: width 0.3s ease;

  &--good { background: $ok; }
  &--mid  { background: $amber; }
  &--low  { background: $danger; }
}

.detail-card {
  border: 1px solid $line;
  border-left: 3px solid $line;
  border-radius: 12px;
  padding: 14px 16px;
  background: $paper;

  &--pass { border-left-color: $ok; }
  &--fail { border-left-color: $danger; background: rgba($danger, 0.03); }
}
.detail-card__hdr {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  margin-bottom: 10px;
}
.detail-card__task {
  font-family: $font-display;
  font-weight: 700;
  font-size: 1em; // 0.8125rem / 0.8125rem
  color: $ink;
}
.detail-card__badge {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 2px 8px;
  border-radius: 999px;
  font-family: $font-mono;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  flex-shrink: 0;

  &--pass { color: $ok; background: $ok-wash; }
  &--fail { color: $danger; background: $danger-wash; }
}
.detail-card__row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
  margin-top: 10px;
}
.detail-card__field {
  min-width: 0;
}
.detail-card__label {
  @extend %micro;
  font-size: 0.6923em; // 0.5625rem / 0.8125rem
  color: $ink-3;
  display: block;
  margin-bottom: 4px;
}
.detail-card__text {
  font-family: $font-mono;
  font-size: 0.9231em; // 0.75rem / 0.8125rem
  line-height: 1.5;
  color: $ink;
  white-space: pre-wrap;
  word-break: break-word;
  background: $card;
  border: 1px solid $line-2;
  border-radius: 8px;
  padding: 8px 10px;
}
.detail-card__text--fail { color: $danger; border-color: rgba($danger, 0.25); }

@media (max-width: 640px) {
  .drawer { width: 100%; max-width: 100vw; }
  .detail-card__row { grid-template-columns: 1fr; }
}

// ---- results table ---------------------------------------------------------
.results {
  border: 1px solid $line;
  border-radius: 14px;
  overflow: hidden;
}

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

  tbody tr.winner {
    background: rgba($amber, 0.05);
    &:hover { background: rgba($amber, 0.09); }
  }

  tbody td {
    padding: 12px 14px;
    color: $ink;
  }
}

.cell-rank { font-family: $font-mono; font-weight: 700; color: $ink; }
.cell-model { font-family: $font-display; font-weight: 700; color: $ink; }
.cell-provider { color: $ink-2; }
.cell-num { font-family: $font-mono; font-size: 1em; /* 0.8125rem / 0.8125rem */ font-weight: 700; color: $ink; }
.cell-num--muted { font-weight: 500; color: $ink-2; }
.cell-pass { font-family: $font-mono; font-size: 1em; /* 0.8125rem / 0.8125rem */ font-weight: 700; color: $ok; }
.cell-fail { font-family: $font-mono; font-size: 1em; /* 0.8125rem / 0.8125rem */ font-weight: 700; color: $danger; }

.status-message {
  padding: 40px;
  text-align: center;
  background: $paper;
  border: 1px dashed $line;
  border-radius: 14px;
  color: $ink-2;
  font-size: 1.0769em; // 0.875rem / 0.8125rem
}

.spin { animation: history-spin 0.8s linear infinite; }

// ---- stop-evaluation confirm dialog ------------------------------------
.confirm-overlay {
  position: fixed;
  inset: 0;
  // Deliberately hardcoded, not themed: same "always dark" scrim reasoning
  // as the drawer overlay above.
  background: rgba(20, 22, 27, 0.4);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
  z-index: 50;
}

.confirm-dialog {
  width: 100%;
  max-width: 400px;
  background: $card;
  border: 1px solid $line;
  border-radius: 16px;
  box-shadow: $lift;
  padding: 24px;
}

.confirm-dialog__icon {
  width: 40px;
  height: 40px;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: $danger-wash;
  color: $danger;
  margin-bottom: 14px;
}

.confirm-dialog__title {
  font-family: $font-display;
  font-size: 1.1538em; // 0.9375rem / 0.8125rem
  font-weight: 800;
  letter-spacing: -0.01em;
  color: $ink;
}

.confirm-dialog__body {
  margin-top: 8px;
  font-size: 0.9615em; // 0.78125rem / 0.8125rem
  line-height: 1.55;
  color: $ink-2;

  strong { color: $ink; }
}

.confirm-dialog__actions {
  display: flex;
  justify-content: flex-end;
  gap: 8px;
  margin-top: 22px;
}

.confirm-dialog__btn--ghost {
  padding: 9px 15px;
  border-radius: 9px;
  border: 1px solid $line;
  background: $paper;
  color: $ink-2;
  font-size: 0.9231em; // 0.75rem / 0.8125rem
  font-weight: 650;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease;

  &:hover { border-color: $ink-3; color: $ink; }
}

.confirm-dialog__btn--danger {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 9px 15px;
  border-radius: 9px;
  border: 1px solid transparent;
  background: $danger;
  color: #fff;
  font-size: 0.9231em; // 0.75rem / 0.8125rem
  font-weight: 700;
  cursor: pointer;
  transition: background 0.15s ease, transform 0.15s ease;

  &:hover { background: darken($danger, 8%); transform: translateY(-1px); }
}

@media (max-width: 900px) {
  .shell { flex-direction: column; }
  .sidebar { width: 100%; }
  .summary-cards { grid-template-columns: 1fr; }
  // Once stacked, fall back to one normal scrolling column.
  .pg-body-fixed { overflow-y: auto; }
  .sidebar, .detail { overflow-y: visible; min-height: 0; }
  .rows { overflow-y: visible; }
}

@media (max-width: 640px) {
  .history__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .pg-body-fixed { padding: 16px 18px 22px; }
}
