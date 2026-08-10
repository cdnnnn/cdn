@use './_variables' as *;

* { margin: 0; padding: 0; box-sizing: border-box; }

body {
  font-family: $font-body;
  background: $bg;
  color: $text-primary;
  -webkit-font-smoothing: antialiased;
}

h1, h2, h3, h4, h5 { font-family: $font-display; }

.page-enter { animation: pageIn .35s ease both; }
@keyframes pageIn { from { opacity: 0; transform: translateY(12px); } to { opacity: 1; transform: translateY(0); } }
@keyframes spin { to { transform: rotate(360deg); } }
@keyframes pulse { 0%, 100% { opacity: 1; } 50% { opacity: .4; } }
@keyframes float { 0%, 100% { transform: translateY(0); } 50% { transform: translateY(-8px); } }
@keyframes dotPulse { 0%, 100% { opacity: .15; } 50% { opacity: .35; } }
@keyframes toastIn { from { opacity: 0; transform: translateY(20px) scale(.95); } to { opacity: 1; transform: translateY(0) scale(1); } }

// ---- Shared primitives reused across feature components ----
.btn {
  display: inline-flex; align-items: center; gap: 6px;
  padding: 10px 20px; border-radius: 12px; font-size: 14px; font-weight: 600;
  cursor: pointer; transition: all .2s; border: none; font-family: $font-display;
}
.btn-ind { background: $grad-primary; color: #fff; box-shadow: 0 2px 8px rgba(20, 40, 160, .2); }
.btn-ind:hover { box-shadow: 0 4px 16px rgba(20, 40, 160, .3); transform: translateY(-1px); }
.btn-ghost { background: $surface; color: $text-secondary; border: 1px solid $border; }
.btn-ghost:hover { border-color: $indigo; color: $indigo; background: $indigo-pale; }
.btn-sm { padding: 7px 14px; font-size: 13px; border-radius: 10px; }
.btn-danger { background: $red-pale; color: $red; border: 1px solid transparent; }
.btn-danger:hover { background: #FEE2E2; }
.btn:disabled { opacity: .45; cursor: not-allowed; }

.badge {
  display: inline-flex; align-items: center; gap: 6px;
  padding: 5px 12px; border-radius: 100px; font-size: 12px; font-weight: 700; letter-spacing: .2px;
}
.badge-green { background: $emerald-pale; color: $emerald-dark; }
.badge-gray { background: $surface-alt; color: $text-muted; }
.badge-blue { background: $indigo-pale; color: $indigo; }
.badge-amber { background: $amber-pale; color: $amber-dark; }
.badge-run { background: $indigo-pale; color: $indigo; }

.tag { display: inline-block; padding: 3px 9px; border-radius: 7px; font-size: 11px; font-weight: 700; margin-right: 4px; margin-bottom: 4px; }
.tag-ind { background: $indigo-pale; color: $indigo; }
.tag-amb { background: $amber-pale; color: $amber-dark; }
.tag-em { background: $emerald-pale; color: $emerald-dark; }

.search-box {
  display: flex; align-items: center; gap: 8px;
  background: $surface; border: 1px solid $border; border-radius: 12px;
  padding: 10px 16px; min-width: 300px; transition: all .2s;
}
.search-box:focus-within { border-color: $indigo; box-shadow: 0 0 0 4px rgba(20, 40, 160, .08); }
.search-box input { border: none; outline: none; font-size: 14px; flex: 1; color: $text-primary; font-family: $font-body; background: transparent; }
.search-box input::placeholder { color: $text-muted; }

.pills { display: flex; gap: 6px; flex-wrap: wrap; }
.pill {
  padding: 7px 16px; border-radius: 100px; font-size: 13px; font-weight: 600;
  border: 1px solid $border; background: $surface; color: $text-secondary; cursor: pointer; transition: all .2s;
}
.pill:hover { border-color: $indigo; color: $indigo; }
.pill.on { background: $grad-primary; color: #fff; border-color: transparent; box-shadow: 0 2px 8px rgba(20, 40, 160, .25); }

.toolbar { display: flex; align-items: center; justify-content: space-between; margin-bottom: 24px; flex-wrap: wrap; gap: 12px; }

.cards-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(330px, 1fr)); gap: 16px; }
.card {
  background: $surface; border: 1px solid $border; border-radius: 16px; padding: 24px;
  transition: all .25s; position: relative; overflow: hidden;
}
.card:hover { border-color: rgba(20, 40, 160, .2); box-shadow: $shadow-3; transform: translateY(-2px); }
.card-hdr { display: flex; justify-content: space-between; align-items: flex-start; margin-bottom: 14px; }
.card-icon { width: 44px; height: 44px; border-radius: 12px; display: flex; align-items: center; justify-content: center; flex-shrink: 0; }
.card-title { font-family: $font-display; font-size: 16px; font-weight: 700; }
.card-desc { font-size: 13px; color: $text-secondary; line-height: 1.6; margin-bottom: 16px; }
.card-foot { display: flex; justify-content: space-between; align-items: center; margin-top: 16px; padding-top: 16px; border-top: 1px solid $border-light; }

.tw { background: $surface; border: 1px solid $border; border-radius: 16px; overflow: hidden; }
.tbl { width: 100%; border-collapse: collapse; }
.tbl th {
  text-align: left; padding: 14px 20px; font-size: 11px; font-weight: 700; color: $text-muted;
  text-transform: uppercase; letter-spacing: 1px; background: $surface-alt; border-bottom: 1px solid $border;
  font-family: $font-display;
}
.tbl td { padding: 14px 20px; font-size: 14px; border-bottom: 1px solid $border-light; }
.tbl tr:last-child td { border-bottom: none; }
.tbl tr { transition: background .15s; }
.tbl tr:hover td { background: $surface-hover; }
.tbl .winner td { background: $amber-pale; }
.tbl .winner td:first-child { box-shadow: inset 3px 0 0 $amber; }

.fg { margin-bottom: 22px; }
.fl { display: block; font-size: 13px; font-weight: 700; margin-bottom: 8px; }
.fl .opt { color: $text-muted; font-weight: 400; font-size: 12px; }
.fi {
  width: 100%; padding: 12px 16px; border: 1px solid $border; border-radius: 12px;
  font-size: 14px; font-family: $font-body; color: $text-primary; transition: all .2s; background: $surface;
}
.fi:focus { outline: none; border-color: $indigo; box-shadow: 0 0 0 4px rgba(20, 40, 160, .08); }
.fi::placeholder { color: $text-muted; }

.sel-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(200px, 1fr)); gap: 10px; }
.sel-opt {
  display: flex; align-items: center; gap: 12px; padding: 16px;
  border: 1.5px solid $border; border-radius: 14px; cursor: pointer; transition: all .2s; background: $surface;
}
.sel-opt:hover { border-color: $indigo-light; background: rgba(238, 242, 255, .4); }
.sel-opt.on { border-color: $indigo; background: $indigo-pale; }
.sel-chk {
  width: 22px; height: 22px; border: 2px solid $border; border-radius: 7px;
  display: flex; align-items: center; justify-content: center; flex-shrink: 0; transition: all .2s;
}
.sel-opt.on .sel-chk { background: $grad-primary; border-color: $indigo; color: #fff; box-shadow: 0 2px 4px rgba(20, 40, 160, .25); }
.sel-txt { font-size: 14px; font-weight: 600; }
.sel-sub { font-size: 12px; color: $text-secondary; margin-top: 2px; }

.toast {
  position: fixed; bottom: 32px; right: 32px; background: $surface; border: 1px solid $border;
  border-radius: 16px; padding: 18px 24px; display: flex; align-items: center; gap: 12px;
  box-shadow: $shadow-4; z-index: 999; animation: toastIn .4s ease both;
}

// Shared by Dashboard and Comparison — kept global since both need it.
.radar-wrap { display: flex; justify-content: center; align-items: center; padding: 20px; }

// ─────────────────────────────────────────────────────────────────────────
// Shared "sidebar + main content, single seamless container" layout, used
// by New Evaluation (stepper + wizard), History (list + detail), and
// Reports (list + detail) — one bordered/shadowed box with an internal
// divider between the two sides, no gap between them. Each usage sets its
// own sidebar width via inline style; add `.split-shell--fill` when the
// container needs to stretch to fill a flex parent (History/Reports, whose
// panels scroll independently) rather than size to its own content
// (New Evaluation, whose wizard body scrolls internally instead).
//
// Sidebar gets a light tinted background ($surface-alt-equivalent #F1F4F9)
// to visually separate it from the white main/card content next to it.
// ─────────────────────────────────────────────────────────────────────────
.split-shell {
  display: flex;
  background: $surface;
  border: 1px solid $border;
  border-radius: 20px;
  overflow: hidden;
  box-shadow: $shadow-2;
}
.split-shell--fill { flex: 1; min-height: 0; }
.split-shell__sidebar {
  flex-shrink: 0;
  border-right: 1px solid $border;
  display: flex;
  flex-direction: column;
  min-height: 0;
  background: #F1F4F9;
}
.split-shell__main {
  flex: 1;
  min-width: 0;
  min-height: 0;
  background: $surface;
}

// ─────────────────────────────────────────────────────────────────────────
// Fixed-header page shell. Every /app page's root uses `pg-shell`; the
// header (`pg-hdr`) and, if present, a filters/toolbar row (`pg-toolbar`)
// stay pinned while only `pg-body` scrolls beneath them.
//
// `pg-shell`'s height is derived from the parent flex chain (AppShell's
// __main -> __content, both flex:1/min-height:0 — see AppShell.module.scss)
// rather than grown from content, then extended via calc to reclaim
// __content's $footer-height bottom padding (reserved for the fixed
// Footer), leaving a small $page-bottom-reclaim gap above it.
// ─────────────────────────────────────────────────────────────────────────
.pg-shell {
  display: flex;
  flex-direction: column;
  min-height: 0;
  height: calc(100% + #{$footer-height} - #{$page-bottom-reclaim});
  margin-bottom: calc(-#{$footer-height} + #{$page-bottom-reclaim});
}

.pg-hdr { flex-shrink: 0; padding: 32px 40px 0; }
.pg-hdr h1 { font-size: 28px; font-weight: 700; letter-spacing: -.5px; }
.pg-hdr p { color: $text-secondary; font-size: 14px; margin-top: 4px; }

// Wraps a page's search/filter row (`.toolbar`) so it stays fixed directly
// below the header, above the scrolling body.
.pg-toolbar { flex-shrink: 0; padding: 20px 40px 0; }

// The ONLY element that scrolls. When it directly follows `.pg-toolbar`,
// that block already provides the header-to-content gap (plus `.toolbar`'s
// own margin-bottom), so drop pg-body's own top padding to avoid doubling up.
.pg-body { flex: 1; min-height: 0; overflow-y: auto; padding: 24px 40px 40px; }
.pg-toolbar + .pg-body { padding-top: 0; }

@media (max-width: 768px) {
  .cards-grid { grid-template-columns: 1fr; }
  .pg-hdr, .pg-toolbar, .pg-body { padding-left: 20px; padding-right: 20px; }
}















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
}
.filters { flex-shrink: 0; }
.empty { padding: 24px; text-align: center; color: $text-secondary; font-size: 13px; display: flex; align-items: center; gap: 8px; justify-content: center; }

.rows { flex: 1; min-height: 0; overflow-y: auto; display: flex; flex-direction: column; gap: 10px; margin-top: 14px; }

.row {
  border: 1px solid $border; border-radius: 14px; padding: 14px; cursor: pointer; transition: all .15s;
  background: #FFF;
  position: relative;
}
.row:hover { border-color: $indigo-light; box-shadow: $shadow-2; }
.row.selected {
  border-color: $indigo;
  background-color: #EEF1FC;
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

.detail { padding: 24px; min-height: 0; overflow-y: auto; }
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
}
.empty { padding: 24px; text-align: center; color: $text-secondary; font-size: 13px; display: flex; align-items: center; gap: 8px; justify-content: center; flex-shrink: 0; }

.rows { flex: 1; min-height: 0; overflow-y: auto; display: flex; flex-direction: column; gap: 10px; }

.row {
  border: 1px solid $border; border-radius: 14px; padding: 14px; cursor: pointer; transition: all .15s;
  background: #FFF;
  position: relative;
}
.row:hover { border-color: $indigo-light; box-shadow: $shadow-2; }
.row.selected {
  border-color: $indigo;
  background-color: #EEF1FC;
}

.row__top { display: flex; align-items: center; gap: 10px; margin-bottom: 8px; }
.row__icon { width: 30px; height: 30px; border-radius: 9px; background: $indigo-pale; color: $indigo; display: flex; align-items: center; justify-content: center; flex-shrink: 0; }
.row__name { font-weight: 700; font-size: 14px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.row__badges { display: flex; align-items: center; gap: 6px; margin-bottom: 6px; flex-wrap: wrap; }
.row__meta { font-size: 11px; color: $text-muted; margin-bottom: 8px; }
.row__stats { display: flex; gap: 12px; font-size: 11px; color: $text-secondary; flex-wrap: wrap; }

.detail { padding: 24px; min-height: 0; overflow-y: auto; }
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
