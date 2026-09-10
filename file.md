//Ticketboard.module.scss
@use '../../styles/_variables' as *;

// ===========================================================================
// Ticket board — same ink/paper design system as Providers/Dashboard:
// theme-aware neutrals from _variables, flat accent constants, hover-lift
// cards, mono-ish instrument labels.
//
// Header/toolbar structure and font-scaling convention are copied 1:1 from
// Providers.module.scss: `.ticket-board` sets one base font-size that every
// descendant `em` value is relative to, bumped to 1rem at wide (>1800px)
// viewports so the whole page reads larger on big monitors without any
// individual rule changing.
// ===========================================================================

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;
$radius:  12px;

@keyframes modalIn {
  from { transform: translateY(8px) scale(0.98); opacity: 0; }
  to { transform: translateY(0) scale(1); opacity: 1; }
}

$soft: 0 1px 2px rgba(20, 22, 27, 0.05);
$lift: 0 14px 30px -14px rgba(20, 22, 27, 0.22);

// base font-size the board's internal `em` scale is built on — same value
// Providers uses, so the two pages feel identical in density.
$board-base-font: 0.8125rem;

%micro {
  font-family: $mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

.ticket-board {
  // master scale control — every em-based font-size below responds to this
  font-size: $board-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  display: flex;
  flex-direction: column;
  min-height: 0;
  flex: 1;
  color: $ink;
}

// ---- header -----------------------------------------------------------
.ticket-board__header {
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

.ticket-board__header-eyebrow {
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

.ticket-board__header-sub {
  margin-top: 4px;
  font-size: 1.0385em; // 0.84375rem / 0.8125rem
  color: $ink-2;
}

.ticket-board__header-meta {
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

// ---- toolbar ------------------------------------------------------------
.ticket-board__toolbar {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 14px;
  padding: 14px 32px;
  background: $card;
  border-bottom: 1px solid $line;
  margin-bottom: 20px;
  flex-wrap: wrap;
}

.ticket-board__search {
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

.ticket-board__toolbar-right {
  display: flex;
  align-items: center;
  gap: 14px;
  flex-wrap: wrap;
}

.ticket-board__filter-group {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 4px;
  background: $paper;
  border: 1px solid $line;
  border-radius: 999px;
}

.ticket-board__toolbar-label {
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

.ticket-board__filter-pill {
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

.ticket-board__toolbar-divider {
  flex-shrink: 0;
  width: 1px;
  height: 26px;
  background: $line;
}

.ticket-board__add-btn {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  padding: 9px 15px;
  border: 1px solid $signal;
  border-radius: 10px;
  background: $signal;
  color: #fff;
  font-family: $sans;
  font-size: 1em; // 0.8125rem / 0.8125rem (base)
  font-weight: 650;
  cursor: pointer;
  box-shadow: $soft;
  transition: background 0.16s ease, border-color 0.16s ease, transform 0.16s ease, box-shadow 0.16s ease;

  &:hover { background: $signal-2; border-color: $signal-2; transform: translateY(-1px); box-shadow: $lift; }
}

// ---- columns --------------------------------------------------------------
.ticket-board__columns {
  padding: 0 32px 28px;
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 1em;
  align-items: start;
  flex: 1;
  min-height: 0;
  overflow-y: auto;
}
.ticket-board__column {
  background: $paper;
  border: 1px solid $line-2;
  border-radius: $radius;
  padding: 0.75em;
  display: flex;
  flex-direction: column;
  gap: 0.6em;
  min-height: 8em;
  transition: background 0.15s, border-color 0.15s, box-shadow 0.15s;
}
.ticket-board__column--over {
  border-color: $signal;
  border-style: dashed;
  background: $wash;
  box-shadow: inset 0 0 0 1px $signal;
}
.ticket-board__column--locked {
  border-color: $danger;
  background: $danger-wash;
  box-shadow: inset 0 0 0 1px $danger;
  cursor: not-allowed;
}
.ticket-board__column-head {
  display: flex;
  align-items: center;
  gap: 0.5em;
  padding: 0.1em 0.25em;
}
.ticket-board__column-dot {
  width: 0.6em;
  height: 0.6em;
  border-radius: 50%;
  flex: none;
}
.ticket-board__column-title {
  font-weight: 600;
  font-size: 0.92em;
  letter-spacing: 0.01em;
}
.ticket-board__column-count {
  margin-left: auto;
  min-width: 1.6em;
  height: 1.6em;
  padding: 0 0.4em;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border-radius: 6px;
  background: $ink-wash;
  color: $ink-2;
  font-size: 0.78em;
  font-weight: 600;
  font-family: $mono;
}
.ticket-board__column-lock {
  color: $ink-3;
}
.ticket-board__column-body {
  display: flex;
  flex-direction: column;
  gap: 0.6em;
  min-height: 2em;
}
.ticket-board__column-empty {
  padding: 1.5em 0.5em;
  text-align: center;
  color: $ink-3;
  font-size: 0.82em;
  border: 1px dashed $line;
  border-radius: 8px;
}

// ---- card -----------------------------------------------------------------
.ticket-card {
  --priority-accent: #{$ink-3};
  // Slightly below page body size — dense enough for a kanban card without
  // reading oversized next to the column chrome around it.
  font-size: 0.92em;
  position: relative;
  background: $card;
  border: 1px solid $line;
  border-left: 3px solid var(--priority-accent);
  border-radius: 10px;
  padding: 0.75em 0.8em;
  display: flex;
  flex-direction: column;
  gap: 0.55em;
  cursor: grab;
  box-shadow: $shadow-2;
  transition: transform 0.12s, box-shadow 0.12s, border-color 0.12s;
  &:hover {
    transform: translateY(-1px);
    box-shadow: $shadow-3;
  }
  &:active {
    cursor: grabbing;
  }
}
.ticket-card--moving {
  opacity: 0.6;
  cursor: default;
}
.ticket-card--discarded {
  opacity: 0.72;
  .ticket-card__title {
    text-decoration: line-through;
    color: $ink-2;
  }
}
.ticket-card__top {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.5em;
}
.ticket-card__key {
  font-family: $mono;
  font-size: 0.85em;
  font-weight: 700;
  letter-spacing: 0.04em;
  color: $ink-3;
}
.ticket-card__top-right {
  display: flex;
  align-items: center;
  gap: 0.4em;
}
.ticket-card__priority {
  --priority-accent: #{$ink-3};
  font-size: 0.8em;
  font-weight: 700;
  letter-spacing: 0.03em;
  text-transform: uppercase;
  padding: 0.25em 0.5em;
  border-radius: 5px;
  color: var(--priority-accent);
  background: color-mix(in srgb, var(--priority-accent) 12%, transparent);
}
.ticket-card__spin {
  animation: spin 1.5s linear infinite;
  color: $signal;
}
.ticket-card__title {
  margin: 0;
  font-size: 1.03em;
  font-weight: 600;
  line-height: 1.35;
  color: $ink;
}
.ticket-card__labels {
  display: flex;
  flex-wrap: wrap;
  gap: 0.35em;
}
.ticket-card__label {
  font-size: 0.82em;
  padding: 0.2em 0.5em;
  border-radius: 5px;
  background: $ink-wash;
  color: $ink-2;
}
.ticket-card__foot {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.5em;
  margin-top: 0.1em;
}
.ticket-card__resolution {
  font-size: 0.78em;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.03em;
  padding: 0.2em 0.5em;
  border-radius: 5px;
}
.ticket-card__resolution--done {
  color: $ok;
  background: $ok-wash;
}
.ticket-card__resolution--discarded {
  color: $rose-ink;
  background: $rose-ink-wash;
}
.ticket-card__avatars {
  display: flex;
  align-items: center;
  margin-left: auto;
}
.ticket-card__avatar {
  width: 1.7em;
  height: 1.7em;
  border-radius: 50%;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  font-size: 0.72em;
  font-weight: 700;
  color: #fff;
  border: 2px solid $card;
  & + & {
    margin-left: -0.5em;
  }
}
.ticket-card__avatar--owner {
  box-shadow: 0 0 0 1px $line;
}

// ---- card menu ------------------------------------------------------------
.ticket-card__menu-wrap {
  position: relative;
}
.ticket-card__menu-btn {
  border: 0;
  background: transparent;
  color: $ink-3;
  padding: 0.15em;
  border-radius: 5px;
  cursor: pointer;
  display: inline-flex;
  &:hover {
    background: $ink-wash;
    color: $ink;
  }
}
.ticket-card__menu {
  position: absolute;
  right: 0;
  top: 1.7em;
  z-index: 20;
  min-width: 12em;
  background: $card;
  border: 1px solid $line;
  border-radius: 9px;
  box-shadow: $shadow-3;
  padding: 0.35em;
  display: flex;
  flex-direction: column;
  gap: 0.1em;
  animation: drawerIn 0.12s ease both;
}
.ticket-card__menu-label {
  font-size: 0.66em;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: $ink-3;
  padding: 0.35em 0.55em 0.15em;
}
.ticket-card__menu-item {
  display: flex;
  align-items: center;
  gap: 0.5em;
  width: 100%;
  border: 0;
  background: transparent;
  color: $ink;
  font-size: 0.82em;
  text-align: left;
  padding: 0.5em 0.55em;
  border-radius: 6px;
  cursor: pointer;
  &:hover:not(:disabled) {
    background: $wash;
  }
  &:disabled {
    color: $ink-3;
    cursor: not-allowed;
  }
}
.ticket-card__menu-item--danger:not(:disabled) {
  color: $danger;
  &:hover {
    background: $danger-wash;
  }
}
.ticket-card__menu-dot {
  width: 0.55em;
  height: 0.55em;
  border-radius: 50%;
  flex: none;
}
.ticket-card__menu-lead {
  flex: none;
}
.ticket-card__menu-check {
  margin-left: auto;
  color: $signal;
}
.ticket-card__menu-sep {
  height: 1px;
  background: $line-2;
  margin: 0.2em 0.3em;
}

// ---- loading --------------------------------------------------------------
.ticket-board__loading {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.6em;
  padding: 4em;
  color: $ink-3;
}
.ticket-board__spin {
  animation: spin 1.5s linear infinite;
  color: $signal;
}

// ===========================================================================
// Detail modal — centered dialog (not a bottom sheet), sized to fit the
// two-pane main/rail layout below.
// ===========================================================================
$detail-width: 980px;
$detail-max-height: 700px;

.ticket-detail__overlay {
  position: fixed;
  inset: 0;
  bottom: $footer-height;
  background: rgba(17, 24, 39, 0.5);
  z-index: 100;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 28px;
}
.ticket-detail {
  position: relative;
  width: min(#{$detail-width}, 100%);
  max-height: min(#{$detail-max-height}, 100%);
  background: $surface;
  border: 1px solid $line;
  border-radius: 14px;
  box-shadow: $shadow-4;
  z-index: 101;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  // Own base size, slightly larger than the page base at very wide
  // viewports — a focused modal reads better a touch bigger than the
  // dense board sitting behind it.
  font-size: 0.8125rem;
  @media (min-width: 1800px) {
    font-size: 1.0625rem;
  }
  animation: modalIn 0.16s ease both;
}
.ticket-detail__header {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.5em;
  padding: 1em 1.25em;
  border-bottom: 1px solid $line;
}
.ticket-detail__key {
  font-family: $mono;
  font-size: 0.8em;
  font-weight: 700;
  letter-spacing: 0.04em;
  color: $ink-3;
  margin-right: 0.6em;
}
.ticket-detail__header-actions {
  display: flex;
  align-items: center;
  gap: 0.2em;
}

// Two-pane body: scrollable content on the left, a narrow fixed-width meta
// rail on the right holding requester/assignee/status/actions — so those
// no longer stretch to the panel's full width.
.ticket-detail__body {
  flex: 1;
  min-height: 0;
  display: flex;
}
.ticket-detail__main {
  flex: 1;
  min-width: 0;
  overflow-y: auto;
  padding: 1.1em 1.25em;
  display: flex;
  flex-direction: column;
  gap: 1.1em;
}
.ticket-detail__rail {
  flex: none;
  width: 230px;
  border-left: 1px solid $line;
  background: $paper;
  overflow-y: auto;
  padding: 1.1em;
  display: flex;
  flex-direction: column;
  gap: 1.2em;
}

.ticket-detail__title {
  margin: 0;
  font-size: 1.15em;
  font-weight: 700;
  line-height: 1.35;
}
.ticket-detail__desc {
  margin: 0;
  color: $ink-2;
  font-size: 0.9em;
  line-height: 1.55;
  white-space: pre-wrap;
}
.ticket-detail__desc--empty {
  margin: 0;
  color: $ink-3;
  font-size: 0.88em;
  font-style: italic;
}

// ---- rail: compact people chips (auto-width, not stretched) --------------
.ticket-detail__people {
  display: flex;
  flex-direction: column;
  gap: 0.8em;
}
.ticket-detail__person {
  display: flex;
  flex-direction: column;
  gap: 0.35em;
}
.ticket-detail__person-label {
  font-size: 0.68em;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: $ink-3;
}
.ticket-detail__person-val {
  display: inline-flex;
  align-items: center;
  gap: 0.45em;
  width: fit-content;
  max-width: 100%;
  font-size: 0.86em;
  font-weight: 500;
  padding: 0.3em 0.55em 0.3em 0.3em;
  border-radius: 999px;
  background: $card;
  border: 1px solid $line;
}
.ticket-detail__you {
  font-size: 0.72em;
  font-weight: 700;
  color: $signal;
  background: $wash;
  padding: 0.1em 0.4em;
  border-radius: 4px;
}
.ticket-detail__muted {
  color: $ink-3;
}
.ticket-detail__section-label {
  display: flex;
  align-items: center;
  gap: 0.4em;
  font-size: 0.68em;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: $ink-3;
}

// ---- rail: status — compact auto-width pills, not a full-width grid ------
.ticket-detail__stepper {
  display: flex;
  flex-wrap: wrap;
  gap: 0.4em;
}
.ticket-detail__step {
  --step-accent: #{$signal};
  flex: none;
  border: 1px solid $line;
  background: $card;
  color: $ink-2;
  font-size: 0.78em;
  font-weight: 600;
  padding: 0.45em 0.65em;
  border-radius: 999px;
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 0.3em;
  transition: border-color 0.12s, background 0.12s, color 0.12s;
  &:hover:not(:disabled) {
    border-color: var(--step-accent);
    color: $ink;
  }
  &:disabled {
    cursor: default;
  }
}
.ticket-detail__step--current {
  border-color: var(--step-accent);
  background: color-mix(in srgb, var(--step-accent) 12%, transparent);
  color: var(--step-accent);
}

// ---- rail: terminal actions — compact auto-width buttons, side by side ---
.ticket-detail__terminal {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5em;
}
.ticket-detail__done-btn,
.ticket-detail__discard-btn {
  flex: none;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 0.4em;
  padding: 0.5em 0.8em;
  border-radius: 999px;
  font-size: 0.8em;
  font-weight: 600;
  cursor: pointer;
  border: 1px solid transparent;
  transition: filter 0.12s, opacity 0.12s;
  &:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
  &:not(:disabled):hover {
    filter: brightness(0.96);
  }
}
.ticket-detail__done-btn {
  background: $ok;
  color: #fff;
}
.ticket-detail__discard-btn {
  background: $card;
  border-color: $danger;
  color: $danger;
}
.ticket-detail__gate-note {
  display: flex;
  align-items: flex-start;
  gap: 0.4em;
  margin: 0;
  font-size: 0.75em;
  line-height: 1.4;
  color: $ink-3;
}

// ---- main: attachments -----------------------------------------------------
.ticket-detail__attachments {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(72px, 1fr));
  gap: 0.5em;
}
.ticket-detail__attachment {
  position: relative;
  aspect-ratio: 1;
  border-radius: 8px;
  overflow: hidden;
  border: 1px solid $line;
  background: $paper;
  display: block;

  img {
    width: 100%;
    height: 100%;
    object-fit: cover;
  }
}

// ---- main: comments ---------------------------------------------------------
.ticket-detail__comments {
  display: flex;
  flex-direction: column;
  gap: 0.9em;
}
.ticket-detail__comment {
  display: flex;
  gap: 0.6em;
}
.ticket-detail__comment-body {
  flex: 1;
  min-width: 0;
  background: $paper;
  border: 1px solid $line-2;
  border-radius: 10px;
  padding: 0.6em 0.75em;
}
.ticket-detail__comment-head {
  display: flex;
  align-items: baseline;
  gap: 0.5em;
  margin-bottom: 0.2em;
}
.ticket-detail__comment-author {
  font-size: 0.85em;
  font-weight: 700;
  color: $ink;
}
.ticket-detail__comment-time {
  font-size: 0.72em;
  color: $ink-3;
}
.ticket-detail__comment-text {
  margin: 0;
  font-size: 0.86em;
  line-height: 1.5;
  color: $ink-2;
  white-space: pre-wrap;
}
.ticket-detail__comment-empty {
  font-size: 0.85em;
  color: $ink-3;
  font-style: italic;
}
.ticket-detail__comment-form {
  display: flex;
  gap: 0.6em;
  align-items: flex-start;
}
.ticket-detail__comment-input {
  flex: 1;
  min-height: 2.6em;
  max-height: 8em;
  resize: vertical;
  border: 1px solid $line;
  border-radius: 10px;
  padding: 0.55em 0.7em;
  font-family: $sans;
  font-size: 0.86em;
  color: $ink;
  background: $card;
  transition: border-color 0.15s, box-shadow 0.15s;
  &::placeholder { color: $ink-3; }
  &:focus {
    outline: none;
    border-color: $signal;
    box-shadow: 0 0 0 3px $wash;
  }
}
.ticket-detail__comment-send {
  flex: none;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 2.6em;
  height: 2.6em;
  border-radius: 10px;
  border: 1px solid $signal;
  background: $signal;
  color: #fff;
  cursor: pointer;
  transition: background 0.15s;
  &:hover:not(:disabled) { background: $signal-2; }
  &:disabled { opacity: 0.5; cursor: not-allowed; }
}

@media (max-width: 768px) {
  .ticket-board__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .ticket-board__toolbar { padding: 12px 18px; }
  .ticket-board__columns { padding: 0 18px 20px; grid-template-columns: 1fr; }
  .ticket-detail__overlay { padding: 0; }
  .ticket-detail { width: 100%; max-height: 100%; border-radius: 0; border: 0; }
  .ticket-detail__body { flex-direction: column; overflow-y: auto; }
  .ticket-detail__rail { width: auto; border-left: 0; border-top: 1px solid $line; }
}






















//Createticketdrawer.module.scss
@use '../../styles/_variables' as *;

// Centered modal — same content/layout as before, but presented as a
// classic centered dialog instead of a bottom sheet.

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

$modal-width: 780px;
$modal-max-height: 660px;

@keyframes modalIn {
  from { transform: translateY(8px) scale(0.98); opacity: 0; }
  to { transform: translateY(0) scale(1); opacity: 1; }
}

.sheet__overlay {
  position: fixed;
  inset: 0;
  bottom: $footer-height;
  background: rgba(17, 24, 39, 0.5);
  z-index: 100;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 28px;
}
.sheet {
  position: relative;
  width: min(#{$modal-width}, 100%);
  max-height: min(#{$modal-max-height}, 100%);
  background: $surface;
  border: 1px solid $line;
  border-radius: 14px;
  box-shadow: $shadow-4;
  z-index: 101;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  // base size the modal's own em scale runs on — a little larger than the
  // page base at very wide viewports, since a focused modal reads better
  // slightly bigger than the dense board behind it.
  font-size: 0.8125rem;
  @media (min-width: 1800px) {
    font-size: 1.0625rem;
  }
  animation: modalIn 0.16s ease both;
}

// ---- header -----------------------------------------------------------
.sheet__header {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1.1em 1.25em;
  border-bottom: 1px solid $line;
}
.sheet__header-text {
  display: flex;
  flex-direction: column;
  gap: 0.15em;
}
.sheet__eyebrow {
  font-family: $mono;
  font-size: 0.68em;
  font-weight: 700;
  letter-spacing: 0.1em;
  text-transform: uppercase;
  color: $signal;
}
.sheet__title {
  font-size: 1.2em;
  font-weight: 700;
  color: $ink;
}

// ---- two-column body ----------------------------------------------------
.sheet__body {
  flex: 1;
  min-height: 0;
  display: flex;
}
.sheet__main {
  flex: 1;
  min-width: 0;
  overflow-y: auto;
  padding: 1.25em;
  display: flex;
  flex-direction: column;
  gap: 1.1em;
}
.sheet__rail {
  flex: none;
  width: 260px;
  border-left: 1px solid $line;
  background: $paper;
  overflow-y: auto;
  padding: 1.25em;
  display: flex;
  flex-direction: column;
  gap: 1.1em;
}

.sheet__field {
  display: flex;
  flex-direction: column;
  gap: 0.4em;
}
.sheet__label {
  font-size: 0.78em;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  color: $ink-2;
}
.sheet__req {
  color: $danger;
}
.sheet__input,
.sheet__textarea {
  width: 100%;
  border: 1px solid $line;
  border-radius: 8px;
  background: $card;
  color: $ink;
  font-size: 0.92em;
  font-family: inherit;
  padding: 0.65em 0.7em;
  outline: 0;
  transition: border-color 0.15s, box-shadow 0.15s;
  &:focus {
    border-color: $signal;
    box-shadow: 0 0 0 3px $wash;
  }
  &::placeholder {
    color: $ink-3;
  }
}
.sheet__input--lg {
  font-size: 1.15em;
  font-weight: 600;
  padding: 0.6em 0.7em;
}
.sheet__textarea {
  resize: vertical;
  line-height: 1.55;
  flex: 1;
}
.sheet__input--error {
  border-color: $danger;
  &:focus {
    box-shadow: 0 0 0 3px $danger-wash;
  }
}
.sheet__error {
  font-size: 0.78em;
  color: $danger;
}
.sheet__meta-note {
  margin-top: auto;
  display: flex;
  align-items: center;
  gap: 0.4em;
  font-size: 0.76em;
  color: $ink-3;
  padding-top: 0.8em;
  border-top: 1px dashed $line;
}

// ---- attachments ----------------------------------------------------------
.sheet__attach-zone {
  display: flex;
  flex-wrap: wrap;
  gap: 0.6em;
}
.sheet__attach-thumb {
  position: relative;
  width: 84px;
  height: 84px;
  border-radius: 8px;
  overflow: hidden;
  border: 1px solid $line;
  background: $paper;

  img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    display: block;
  }
}
.sheet__attach-remove {
  position: absolute;
  top: 4px;
  right: 4px;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  border: 0;
  background: rgba(17, 24, 39, 0.65);
  color: #fff;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  &:hover {
    background: $danger;
  }
}
.sheet__attach-meta {
  position: absolute;
  left: 0;
  right: 0;
  bottom: 0;
  padding: 0.15em 0.35em;
  font-size: 0.6em;
  color: #fff;
  background: rgba(17, 24, 39, 0.55);
  text-align: center;
}
.sheet__attach-add {
  width: 84px;
  height: 84px;
  border-radius: 8px;
  border: 1.5px dashed $line;
  background: $paper;
  color: $ink-3;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 0.3em;
  font-size: 0.72em;
  font-weight: 600;
  cursor: pointer;
  transition: border-color 0.15s, color 0.15s, background 0.15s;
  &:hover {
    border-color: $signal;
    color: $signal;
    background: $wash;
  }
}

// ---- chip input -----------------------------------------------------------
.sheet__chip-input {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 0.4em;
  border: 1px solid $line;
  border-radius: 8px;
  background: $card;
  padding: 0.45em 0.5em;
  min-height: 2.6em;
  &:focus-within {
    border-color: $signal;
    box-shadow: 0 0 0 3px $wash;
  }
}
.sheet__chip {
  display: inline-flex;
  align-items: center;
  gap: 0.3em;
  font-size: 0.8em;
  padding: 0.25em 0.3em 0.25em 0.55em;
  border-radius: 6px;
  background: $ink-wash;
  color: $ink-2;
  button {
    border: 0;
    background: transparent;
    color: $ink-3;
    display: inline-flex;
    cursor: pointer;
    padding: 0.1em;
    border-radius: 4px;
    &:hover {
      color: $danger;
      background: $danger-wash;
    }
  }
}
.sheet__chip-field {
  flex: 1;
  min-width: 6em;
  border: 0;
  outline: 0;
  background: transparent;
  color: $ink;
  font-size: 0.9em;
  &::placeholder {
    color: $ink-3;
  }
}

// ---- footer -----------------------------------------------------------
.sheet__footer {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 0.6em;
  padding: 1em 1.25em;
  border-top: 1px solid $line;
  background: $surface;
}
.sheet__spin {
  animation: spin 1.5s linear infinite;
}

@media (max-width: 768px) {
  .sheet__overlay { padding: 0; }
  .sheet { width: 100%; max-height: 100%; border-radius: 0; border: 0; }
  .sheet__body { flex-direction: column; overflow-y: auto; }
  .sheet__rail { width: auto; border-left: 0; border-top: 1px solid $line; }
}
