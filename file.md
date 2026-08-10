//Reports.module.scss
@use '../../styles/_variables' as *;

.reports {
  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding: 24px 32px 18px;
    margin-bottom: 24px;
    border-bottom: 1px solid $border-light;

    h1 {
      font-family: $font-display;
      font-size: 1.5rem;
      font-weight: 800;
      letter-spacing: -0.02em;
      color: $text-primary;
      line-height: 1.2;
    }
  }

  &__header-eyebrow {
    display: flex;
    align-items: center;
    gap: 8px;
    font-family: $font-mono;
    font-size: 0.6875rem;
    font-weight: 700;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    color: $indigo;
    margin-bottom: 6px;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $indigo;
    }
  }

  &__header-sub {
    margin-top: 4px;
    font-size: 0.875rem;
    color: $text-secondary;
  }

  &__header-meta {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-secondary;
    background: $surface-alt;
    border: 1px solid $border-light;
    border-radius: 999px;
    padding: 7px 13px;
    white-space: nowrap;
    margin-bottom: 3px;
  }
}

// Same fixed-shell / independent-panel-scroll pattern as History.module.scss
// (Reports is structurally identical: a list + detail panel) — pg-body
// itself doesn't scroll here, .layout fills it, and each panel scrolls on
// its own via flex:1/min-height:0 rather than a fixed max-height.
.pg-body-fixed {
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

// Border/radius/shadow still come from the shared .split-shell /
// .split-shell__sidebar / .split-shell__main classes in global.scss — only
// the background is overridden here per-page, plus width/padding/layout.
.sidebar {
  width: 380px; padding: 18px;
  background: #F1F4F9;
}
.empty { padding: 24px; text-align: center; color: $text-secondary; font-size: 13px; display: flex; align-items: center; gap: 8px; justify-content: center; flex-shrink: 0; }

.rows { flex: 1; min-height: 0; overflow-y: auto; display: flex; flex-direction: column; gap: 10px; }

.row {
  border: 1px solid $border; border-radius: 14px; padding: 14px 14px 14px 16px; cursor: pointer; transition: all .15s;
  background: #FFF;
  position: relative;
}
.row:hover { border-color: $indigo-light; box-shadow: $shadow-2; }
.row.selected {
  border-color: transparent;
  background:
    linear-gradient(135deg, rgba(20, 40, 160, 0.07), rgba(20, 40, 160, 0.02)) padding-box,
    #FFF padding-box;
  background-color: #EEF1FC;
  box-shadow: $shadow-2, inset 3px 0 0 $indigo;
}

.row__top { display: flex; align-items: center; gap: 10px; margin-bottom: 8px; }
.row__icon { width: 30px; height: 30px; border-radius: 9px; background: $indigo-pale; color: $indigo; display: flex; align-items: center; justify-content: center; flex-shrink: 0; }
.row__name { font-weight: 700; font-size: 14px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.row__badges { display: flex; align-items: center; gap: 6px; margin-bottom: 6px; flex-wrap: wrap; }
.row__meta { font-size: 11px; color: $text-muted; margin-bottom: 8px; }
.row__stats { display: flex; gap: 12px; font-size: 11px; color: $text-secondary; flex-wrap: wrap; }

.detail { padding: 24px; min-height: 0; overflow-y: auto; background: #FFF; }
.detail-empty { padding: 80px 24px; text-align: center; color: $text-secondary; font-size: 14px; }

.detail-hdr { display: flex; justify-content: space-between; align-items: flex-start; margin-bottom: 24px; flex-wrap: wrap; gap: 16px; }
.detail-hdr__badges { display: flex; gap: 8px; margin-bottom: 10px; }
.detail-hdr__name { font-size: 22px; font-weight: 700; letter-spacing: -.3px; }
.detail-hdr__date { font-size: 12px; color: $text-muted; margin-top: 4px; }
.detail-hdr__actions { display: flex; gap: 8px; flex-wrap: wrap; }

.summary-cards { display: grid; grid-template-columns: repeat(3,1fr); gap: 14px; margin-bottom: 24px; }
.summary-card { display: flex; align-items: center; gap: 12px; padding: 16px; background: $surface-alt; border-radius: 14px; }
.summary-card__label { font-size: 11px; color: $text-muted; font-weight: 700; text-transform: uppercase; letter-spacing: .5px; }
.summary-card__val { font-size: 13px; font-weight: 700; margin-top: 2px; }

.summary-text { padding: 16px; background: $surface-alt; border-radius: 14px; font-size: 13px; color: $text-secondary; margin-bottom: 24px; }

.status-message { padding: 40px; text-align: center; background: $surface-alt; border-radius: 14px; color: $text-secondary; font-size: 14px; }

.download-row { display: flex; gap: 8px; flex-wrap: wrap; margin-top: 4px; }

@media (max-width: 900px) {
  :global(.split-shell--fill) { flex-direction: column; }
  .sidebar { width: 100%; border-right: none; border-bottom: 1px solid $border; }
  .summary-cards { grid-template-columns: 1fr; }
  // Same mobile fallback as History — independent-panel scrolling needs a
  // definite grid row height, which is lost once stacked to one column.
  .pg-body-fixed { overflow-y: auto; }
  .sidebar, .detail { overflow-y: visible; min-height: 0; }
  .rows { overflow-y: visible; }
}













//History.module.scss
@use '../../styles/_variables' as *;

.history {
  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding: 24px 32px 18px;
    margin-bottom: 24px;
    border-bottom: 1px solid $border-light;

    h1 {
      font-family: $font-display;
      font-size: 1.5rem;
      font-weight: 800;
      letter-spacing: -0.02em;
      color: $text-primary;
      line-height: 1.2;
    }
  }

  &__header-eyebrow {
    display: flex;
    align-items: center;
    gap: 8px;
    font-family: $font-mono;
    font-size: 0.6875rem;
    font-weight: 700;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    color: $indigo;
    margin-bottom: 6px;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $indigo;
    }
  }

  &__header-sub {
    margin-top: 4px;
    font-size: 0.875rem;
    color: $text-secondary;
  }

  &__header-meta {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-secondary;
    background: $surface-alt;
    border: 1px solid $border-light;
    border-radius: 999px;
    padding: 7px 13px;
    white-space: nowrap;
    margin-bottom: 3px;
  }
}

@property --angle {
  syntax: '<angle>';
  initial-value: 0deg;
  inherits: false;
}
@keyframes rotate-angle {
  to { --angle: 360deg; }
}
@keyframes live-dot-pulse {
  0%, 100% { opacity: 1; transform: scale(1); }
  50% { opacity: .5; transform: scale(1.3); }
}

// Fixed-shell override: History's list + detail panels need to scroll
// independently of each other, so pg-body itself must not scroll — this
// makes it a plain flex:1/min-height:0 pass-through instead.
.pg-body-fixed {
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

// Border/radius/shadow still come from the shared .split-shell /
// .split-shell__sidebar / .split-shell__main classes in global.scss — only
// the background is overridden here per-page, plus width/padding/layout.
.sidebar {
  width: 380px; padding: 18px;
  background: #F1F4F9;
}
.filters { flex-shrink: 0; }
.empty { padding: 24px; text-align: center; color: $text-secondary; font-size: 13px; display: flex; align-items: center; gap: 8px; justify-content: center; }

.rows { flex: 1; min-height: 0; overflow-y: auto; display: flex; flex-direction: column; gap: 10px; margin-top: 14px; }

.row {
  border: 1px solid $border; border-radius: 14px; padding: 14px 14px 14px 16px; cursor: pointer; transition: all .15s;
  background: #FFF;
  position: relative;
}
.row:hover { border-color: $indigo-light; box-shadow: $shadow-2; }
.row.selected {
  border-color: transparent;
  background-color: #EEF1FC;
  box-shadow: $shadow-2, inset 3px 0 0 $indigo;
}

// Running-state animation: a thin light continuously traveling around the
// card border (spec §2.2), built with a rotating conic-gradient angle.
// The gradient's inner fill (first layer) still needs to be an opaque
// color, so it picks up whichever background the row currently has instead
// of a hardcoded #fff — otherwise selected+running rows would flash white.
.row--running {
  --angle: 0deg;
  border: 1px solid transparent;
  background:
    linear-gradient(#fff, #fff) padding-box,
    conic-gradient(from var(--angle), $border 0%, $indigo 8%, $border 16%) border-box;
  animation: rotate-angle 2.4s linear infinite;
}
.row--running.selected {
  background:
    linear-gradient(#EEF1FC, #EEF1FC) padding-box,
    conic-gradient(from var(--angle), $border 0%, $indigo 8%, $border 16%) border-box;
  box-shadow: $shadow-2;
}

.row__top { display: flex; align-items: center; gap: 10px; margin-bottom: 8px; }
.row__icon { width: 30px; height: 30px; border-radius: 9px; background: $indigo-pale; color: $indigo; display: flex; align-items: center; justify-content: center; flex-shrink: 0; }
.row__name { font-weight: 700; font-size: 14px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.row__badges { display: flex; align-items: center; gap: 6px; margin-bottom: 6px; flex-wrap: wrap; }
.row__meta { font-size: 11px; color: $text-muted; margin-bottom: 8px; }
.row__stats { display: flex; gap: 12px; font-size: 11px; color: $text-secondary; margin-bottom: 10px; flex-wrap: wrap; }
.row__actions { display: flex; gap: 6px; }

.live-dot {
  width: 6px; height: 6px; border-radius: 50%; background: currentColor; display: inline-block;
  animation: live-dot-pulse 1.4s ease-in-out infinite;
}

.detail { padding: 24px; min-height: 0; overflow-y: auto; background: #FFF; }
.detail-empty { padding: 80px 24px; text-align: center; color: $text-secondary; font-size: 14px; }

.detail-hdr { display: flex; justify-content: space-between; align-items: flex-start; margin-bottom: 24px; flex-wrap: wrap; gap: 16px; }
.detail-hdr__badges { display: flex; gap: 8px; margin-bottom: 10px; }
.detail-hdr__name { font-size: 22px; font-weight: 700; letter-spacing: -.3px; }
.detail-hdr__date { font-size: 12px; color: $text-muted; margin-top: 4px; }
.detail-hdr__actions { display: flex; gap: 8px; flex-wrap: wrap; }

.summary-cards { display: grid; grid-template-columns: repeat(3,1fr); gap: 14px; margin-bottom: 24px; }
.summary-card { display: flex; align-items: center; gap: 12px; padding: 16px; background: $surface-alt; border-radius: 14px; }
.summary-card__label { font-size: 11px; color: $text-muted; font-weight: 700; text-transform: uppercase; letter-spacing: .5px; }
.summary-card__val { font-size: 13px; font-weight: 700; margin-top: 2px; }

.status-message { padding: 40px; text-align: center; background: $surface-alt; border-radius: 14px; color: $text-secondary; font-size: 14px; }

@media (max-width: 900px) {
  :global(.split-shell--fill) { flex-direction: column; }
  .sidebar { width: 100%; border-right: none; border-bottom: 1px solid $border; }
  .summary-cards { grid-template-columns: 1fr; }
  // Independent-panel scrolling relies on the grid stretching each panel to
  // a definite row height; once stacked to one column that no longer holds,
  // so fall back to one normal scrolling column instead of risking clipped
  // content under pg-body-fixed's overflow:hidden.
  .pg-body-fixed { overflow-y: auto; }
  .sidebar, .detail { overflow-y: visible; min-height: 0; }
  .rows { overflow-y: visible; }
}
