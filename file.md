// ═══════════════════════════════════════════════
// FilePanel.module.scss
// Content Analytics · Upload panel — two sections
// ═══════════════════════════════════════════════
@use '../../styles/mixins' as m;

// ── Outer panel shell ─────────────────────────
.panel {
  width: 100%;
  flex: 1;
  border-right: none;
  display: flex;
  flex-direction: row;
  align-items: stretch;
  overflow: hidden;
  background: var(--bg0);
  position: relative;

  &::after {
    content: '';
    position: absolute;
    top: 0;
    right: 0;
    bottom: 0;
    width: 1px;
    background: linear-gradient(180deg,
        rgba(139, 92, 246, 0.0) 0%,
        rgba(139, 92, 246, 0.6) 20%,
        rgba(56, 196, 186, 0.7) 55%,
        rgba(240, 160, 48, 0.6) 85%,
        rgba(240, 160, 48, 0.0) 100%);
    pointer-events: none;
    z-index: 1;
  }

  // On very large screens, everything (text, padding, borders, icons)
  // was tuned for ~1440-1900px and starts to look small. `zoom` scales
  // the whole subtree proportionally and reflows layout correctly
  // (unlike transform: scale, which doesn't affect layout/hit-testing).
  @media (min-width: 1900px) {
    zoom: 1.25;
  }
}

// ══════════════════════════════════════
// SECTION 1 — Step header + upload zone
// ══════════════════════════════════════
.step1 {
  width: 400px;
  flex-shrink: 0;
  border-bottom: none;
  border-right: 1px solid var(--bdr);
  background: var(--bg0);
  display: flex;
  flex-direction: column;
  overflow: hidden;
  position: relative;

  @media (max-width: 1499px) {
    width: 350px;
  }
}

// ── The upload zone itself, as a distinct card sitting in the sidebar ──
.step1Card {
  flex: 1;
  min-height: 0;
  display: flex;
  flex-direction: column;
  background: var(--bg1);
  overflow: hidden;
  box-shadow: var(--shadow-sm);
}

// Reopen affordance shown in the file-list header once the upload
// column has been collapsed to width: 0 (and its own header bar with it).
.reopenUploadBtn {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  margin-right: 8px;
  padding: 3px 9px;
  border-radius: 99px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  font-size: 12px;
  @include m.mono;
  cursor: pointer;
  transition: all 0.12s;

  svg { width: 10px; height: 10px; }

  &:hover {
    background: var(--bg3);
    color: var(--t1);
    border-color: var(--bdr3);
  }
}

// ── Step-1 header bar (original design) ──────
.step1Bar {
  padding: 10px 14px;
  display: flex;
  align-items: center;
  gap: 9px;
  border-bottom: none;
  background: var(--bg1);
  flex-shrink: 0;
  white-space: nowrap;
  position: relative;

  &::after {
    content: '';
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    height: 1px;
    background: linear-gradient(90deg,
        rgba(139, 92, 246, 0.0) 0%,
        rgba(139, 92, 246, 0.6) 20%,
        rgba(56, 196, 186, 0.7) 50%,
        rgba(240, 160, 48, 0.6) 80%,
        rgba(240, 160, 48, 0.0) 100%);
    pointer-events: none;
  }
}

.slbl {
  font-size: 12px;
  font-weight: 600;
  color: var(--t2);
  text-transform: uppercase;
  letter-spacing: 0.1em;
  @include m.mono;
}

// Collapse toggle button — matches original
.collapseBtn {
  display: flex;
  align-items: center;
  gap: 5px;
  margin-left: auto;
  padding: 3px 8px;
  border-radius: 99px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  font-size: 12px;
  @include m.mono;
  cursor: pointer;
  transition: all 0.12s;
  user-select: none;
  flex-shrink: 0;

  svg {
    width: 10px;
    height: 10px;
  }

  &:hover {
    background: var(--bg3);
    color: var(--t1);
    border-color: var(--bdr3);
  }
}

// ── Collapsible content area — fills the rest of the column, scrolls
// internally so a long browsed/uploading list doesn't blow out the height ──
.step1Content {
  flex: 1;
  min-height: 0;
  overflow-y: auto;
  @include m.scrollbar;
}

// ── Drop zone (original style) ────────────────
.dropzone {
  border: 1.5px dashed var(--bdr2);
  border-radius: var(--rxl);
  padding: 24px 20px;
  margin: 12px 14px;
  text-align: center;
  background: var(--bg1);
  cursor: pointer;
  transition: all 0.18s;
  position: relative;
  overflow: hidden;

  &::before {
    content: '';
    position: absolute;
    inset: 0;
    background: radial-gradient(ellipse at 50% 0%, rgba(139, 92, 246, 0.04), transparent 65%);
    pointer-events: none;
  }

  &:hover,
  &.dragOver {
    border-color: var(--blue);
    background: var(--bg2);
  }

  &.dragOver {
    background: var(--blue-dim);
  }
}

.dzIc {
  width: 38px;
  height: 38px;
  border-radius: 10px;
  background: var(--blue-dim);
  border: 1px solid var(--blue-bdr);
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 0 auto 11px;

  svg {
    width: 18px;
    height: 18px;
  }
}

.dzTitle {
  font-size: 14px;
  font-weight: 500;
  color: var(--t0);
  margin-bottom: 4px;
}

.dzSub {
  font-size: 13px;
  color: var(--t2);
  @include m.mono;
}

.chips {
  display: flex;
  justify-content: center;
  gap: 6px;
  flex-wrap: wrap;
  margin-top: 10px;
}

.chip {
  font-size: 12px;
  padding: 2px 8px;
  border-radius: 99px;
  background: var(--bg3);
  border: 1px solid var(--bdr);
  color: var(--t2);
  @include m.mono;
}

.dzActions {
  display: flex;
  gap: 8px;
  justify-content: center;
  margin-top: 11px;
  position: relative;
  z-index: 1;
}

// ── Preview / uploading ───────────────────────
.previewWrap {
  display: flex;
  flex-direction: column;
  animation: fadeSlide 0.18s ease;
}

.previewList {
  overflow-y: auto;
  padding: 8px 10px;
  display: flex;
  flex-direction: column;
  gap: 5px;
  @include m.scrollbar;
}

// Browsed file card
.fileCard {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 11px;
  background: var(--bg2);
  border: 1px solid var(--bdr);
  border-radius: var(--rxl);
  transition: border-color 0.15s, background 0.15s, box-shadow 0.15s;
  position: relative;
  overflow: hidden;

  // Subtle shimmer background
  &::before {
    content: '';
    position: absolute;
    inset: 0;
    background: radial-gradient(ellipse at 0% 50%, rgba(139, 92, 246, 0.04), transparent 70%);
    pointer-events: none;
  }

  // Left accent bar
  &::after {
    content: '';
    position: absolute;
    left: 0;
    top: 8px;
    bottom: 8px;
    width: 3px;
    border-radius: 0 3px 3px 0;
    background: var(--bdr2);
    transition: background 0.2s;
  }

  &.uploading {
    border-color: rgba(139, 92, 246, 0.35);
    background: rgba(139, 92, 246, 0.05);
    box-shadow: 0 0 0 1px rgba(139, 92, 246, 0.12) inset;

    &::after {
      background: linear-gradient(180deg, var(--blue), #a78bfa);
    }

    &::before {
      background: radial-gradient(ellipse at 0% 50%, rgba(139, 92, 246, 0.08), transparent 70%);
    }
  }

  &.success {
    border-color: var(--green-bdr);
    background: var(--green-dim);

    &::after {
      background: var(--green);
    }

    &::before {
      background: radial-gradient(ellipse at 0% 50%, rgba(52, 211, 153, 0.08), transparent 70%);
    }
  }

  &.failed {
    border-color: var(--red-bdr);
    background: var(--red-dim);

    &::after {
      background: var(--red);
    }

    &::before {
      background: radial-gradient(ellipse at 0% 50%, rgba(239, 68, 68, 0.08), transparent 70%);
    }
  }
}

// Extension badge (shared between browsed + uploaded cards)
.extBadge {
  width: 34px;
  height: 34px;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 10px;
  font-weight: 800;
  @include m.mono;
  flex-shrink: 0;
  letter-spacing: 0.03em;
  position: relative;
  z-index: 1;

  &.vtt {
    background: linear-gradient(135deg, rgba(91, 164, 239, 0.18) 0%, rgba(139, 92, 246, 0.14) 100%);
    color: var(--blue);
    border: 1px solid rgba(91, 164, 239, 0.3);
    box-shadow: 0 1px 4px rgba(91, 164, 239, 0.12);
  }

  &.srt {
    background: linear-gradient(135deg, rgba(52, 211, 153, 0.18) 0%, rgba(56, 196, 186, 0.14) 100%);
    color: var(--green);
    border: 1px solid rgba(52, 211, 153, 0.3);
    box-shadow: 0 1px 4px rgba(52, 211, 153, 0.12);
  }
}

.fileInfo {
  flex: 1;
  min-width: 0;
}

.fileNameRow {
  display: flex;
  align-items: center;
  gap: 5px;
  min-width: 0;
}

.fileName {
  font-size: 13px;
  font-weight: 500;
  color: var(--t0);
  @include m.truncate;
  margin-bottom: 1px;
  flex: 1;
  min-width: 0;
}

.fileId {
  font-size: 10px;
  color: var(--t2);
  @include m.mono;
  flex-shrink: 0;
  opacity: 0.65;
}

// ── File ID badge (shown before ext badge in card) ──
.fileIdBadge {
  font-size: 10px;
  font-weight: 700;
  font-family: var(--font-mono);
  color: #a78bfa;
  background: rgba(167, 139, 250, 0.12);
  border: 1px solid rgba(167, 139, 250, 0.3);
  border-radius: 4px;
  padding: 1px 5px;
  flex-shrink: 0;
  white-space: nowrap;
  letter-spacing: 0.02em;
}

.fileMeta {
  font-size: 12px;
  color: var(--t2);
  @include m.mono;
  display: flex;
  align-items: center;
  gap: 6px;
  margin-top: 2px;
  flex-wrap: wrap;
}

.fileSizeChip {
  font-size: 11px;
  padding: 1px 6px;
  border-radius: 99px;
  background: var(--bg3);
  border: 1px solid var(--bdr);
  color: var(--t2);
}

.fileStatusText {
  font-size: 11px;
  color: var(--t2);
  opacity: 0.7;
}

.fileStatusTextSuccess {
  font-size: 11px;
  color: var(--green);
  font-weight: 600;
}

.fileError {
  color: var(--red);
}

// Status icon (upload progress)
.statusIc {
  width: 18px;
  height: 18px;
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;

  svg {
    width: 15px;
    height: 15px;
  }

  &.pending {
    color: var(--t2);
  }

  &.uploading {
    color: var(--blue);
    animation: spin 0.9s linear infinite;
  }

  &.success {
    color: var(--green);
  }

  &.failed {
    color: var(--red);
  }
}

.removeBtn {
  width: 22px;
  height: 22px;
  border-radius: 5px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  padding: 0;
  transition: all 0.12s;

  svg {
    width: 10px;
    height: 10px;
  }

  &:hover {
    background: var(--red-dim);
    border-color: var(--red-bdr);
    color: var(--red);
  }
}

// Action bar (Upload / Cancel)
.step1Footer {
  flex-shrink: 0;
  display: flex;
  gap: 7px;
  padding: 12px 14px;
  border-top: 1px solid var(--bdr);
  background: var(--bg1);
}

.step1FooterBtn {
  flex: 1;
}

// Upload progress bar
.uploadSummary {
  width: 100%;
}

.uploadProgressBar {
  height: 3px;
  background: var(--bg3);
  border-radius: 99px;
  overflow: hidden;
  margin-bottom: 6px;
}

.uploadProgressFill {
  height: 100%;
  background: linear-gradient(90deg, var(--blue), #a78bfa);
  border-radius: 99px;
  transition: width 0.4s ease;
}

.uploadProgressLabel {
  font-size: 12px;
  color: var(--t2);
  @include m.mono;
}

.failCount {
  color: var(--red);
}

// ══════════════════════════════════════
// SECTION 2 — Uploaded files list
// ══════════════════════════════════════
.section2 {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  background: var(--bg0);
}

// Section 2 header — stacked: title row + action row/grid below
.section2Header {
  display: flex;
  flex-direction: column;
  width: 100%;
  box-sizing: border-box;
  padding: 0;
  border-bottom: none;
  background: var(--bg1);
  flex-shrink: 0;
  position: relative;
}

// Title row — checkbox + title + optional search icon
.section2TitleRow {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 14px 8px;
  width: 100%;
  box-sizing: border-box;
}

.section2Title {
  font-size: 12px;
  font-weight: 600;
  color: var(--t2);
  text-transform: uppercase;
  letter-spacing: 0.1em;
  @include m.mono;
  display: flex;
  align-items: center;
  gap: 7px;
}

.filesCount {
  font-size: 10px;
  font-weight: 700;
  color: var(--blue);
  background: var(--blue-dim);
  border: 1px solid var(--blue-bdr);
  padding: 1px 6px;
  border-radius: 99px;
}

// Cross-page selection total — shown next to filesCount while in
// select/delete/export mode so it's clear selections persist across pages.
.selectedTotalHint {
  font-size: 10px;
  font-weight: 700;
  color: var(--green);
  background: var(--green-dim);
  border: 1px solid var(--green-bdr);
  padding: 1px 6px;
  border-radius: 99px;
  margin-left: 4px;
}



// ── Delete icon button (used inside mode-bar) ─────────────────
.deleteIconBtn {
  width: 28px;
  height: 28px;
  border-radius: 7px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  flex-shrink: 0;
  transition: all 0.15s;

  svg {
    width: 14px;
    height: 14px;
  }

  &:hover:not(:disabled) {
    background: rgba(239, 68, 68, 0.1);
    border-color: rgba(239, 68, 68, 0.35);
    color: #ef4444;
  }

  &:disabled {
    opacity: 0.3;
    cursor: default;
  }
}

// Confirm-delete state — always red, solid
.deleteIconBtnConfirm {
  border-color: rgba(239, 68, 68, 0.45);
  background: rgba(239, 68, 68, 0.12);
  color: #ef4444;

  &:hover:not(:disabled) {
    background: rgba(239, 68, 68, 0.22);
    border-color: rgba(239, 68, 68, 0.7);
    box-shadow: 0 0 8px rgba(239, 68, 68, 0.25);
  }
}

.deleteIconBtnDisabled {
  opacity: 0.4 !important;
  cursor: default !important;
}

// ── Export icon button (used inside mode-bar) ─────────────────
.exportIconBtn {
  width: 28px;
  height: 28px;
  border-radius: 7px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  flex-shrink: 0;
  transition: all 0.15s;

  svg {
    width: 14px;
    height: 14px;
  }

  &:hover:not(:disabled) {
    background: rgba(78, 200, 122, 0.10);
    border-color: rgba(78, 200, 122, 0.35);
    color: var(--green);
  }

  &:disabled {
    opacity: 0.3;
    cursor: default;
  }
}

// Confirm-export state — always green, solid
.exportIconBtnConfirm {
  border-color: rgba(78, 200, 122, 0.45);
  background: rgba(78, 200, 122, 0.10);
  color: var(--green);

  &:hover:not(:disabled) {
    background: rgba(78, 200, 122, 0.20);
    border-color: rgba(78, 200, 122, 0.70);
    box-shadow: 0 0 8px rgba(78, 200, 122, 0.22);
  }
}

.exportIconBtnDisabled {
  opacity: 0.4 !important;
  cursor: default !important;
}

.miniSpinner {
  width: 12px;
  height: 12px;
  border: 1.5px solid rgba(239, 68, 68, 0.3);
  border-top-color: #ef4444;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
  flex-shrink: 0;
}

.miniSpinnerGreen {
  width: 12px;
  height: 12px;
  border: 1.5px solid rgba(78, 200, 122, 0.3);
  border-top-color: #4ec87a;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
  flex-shrink: 0;
}


// ── Cancel icon button (used inside mode-bar) ─────────────────
.cancelIconBtn {
  width: 28px;
  height: 28px;
  border-radius: 7px;
  border: 1px solid rgba(239, 68, 68, 0.35);
  background: rgba(239, 68, 68, 0.08);
  color: #ef4444;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  flex-shrink: 0;
  transition: all 0.15s;

  svg {
    width: 13px;
    height: 13px;
  }

  &:hover {
    background: rgba(239, 68, 68, 0.16);
    border-color: rgba(239, 68, 68, 0.6);
    box-shadow: 0 0 6px rgba(239, 68, 68, 0.2);
  }
}

// Cards get pointer cursor only when in selection mode
.uploadedCardWrapSelectable {
  cursor: pointer;
}

// Date range filter — sits above section2Header
.filterSortRow {
  display: flex;
  flex-direction: column;
  gap: 10px;
  padding: 10px 12px;
  background: var(--bg1);
  flex-shrink: 0;
  position: relative;

  &::after {
    content: '';
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    height: 1px;
    background: linear-gradient(90deg,
        rgba(139, 92, 246, 0.0) 0%,
        rgba(139, 92, 246, 0.6) 20%,
        rgba(56, 196, 186, 0.7) 50%,
        rgba(240, 160, 48, 0.6) 80%,
        rgba(240, 160, 48, 0.0) 100%);
    pointer-events: none;
  }
}

.sortHeader {
  margin-left: auto;
}

// Row 2 — date filter + status filter on the left, actions trigger on the right
.filterBarRow {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 10px;
  width: 100%;
}

// Groups the status filter and the actions trigger together, pushed to
// the right edge of the row, with a visible gap between the two.
.filterBarRight {
  display: flex;
  align-items: center;
  gap: 14px;
  margin-left: auto;
  flex-shrink: 0;
}

// Small caption shown inside the date/status/actions pills so their
// purpose is legible at a glance, not just implied by an icon.
.filterBarLabel {
  font-size: 9.5px;
  font-weight: 600;
  color: var(--t2);
  text-transform: uppercase;
  letter-spacing: 0.06em;
  white-space: nowrap;
  @include m.mono;
}

// ── Actions — collapsed into a single trigger button + popover ──
.actionsPopoverWrap {
  position: relative;
  flex-shrink: 0;
}

.actionsTrigger {
  display: flex;
  align-items: center;
  gap: 6px;
  height: 32px;
  padding: 0 10px;
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  background: var(--bg2);
  color: var(--t1);
  cursor: pointer;
  transition: all 0.12s;

  svg {
    width: 13px;
    height: 13px;
    flex-shrink: 0;
  }

  &:hover {
    background: var(--bg3);
    border-color: var(--bdr3);
  }
}

.actionsTriggerOpen {
  background: var(--blue-dim);
  border-color: var(--blue-bdr);
  color: var(--blue);
}

.actionsTriggerChevron {
  transition: transform 0.15s;

  .actionsTriggerOpen & {
    transform: rotate(180deg);
  }
}

.actionsPopover {
  position: absolute;
  top: calc(100% + 6px);
  right: 0;
  z-index: 20;
  display: flex;
  flex-direction: column;
  gap: 4px;
  padding: 8px;
  min-width: 168px;
  border: 1px solid var(--bdr2);
  border-radius: var(--rl);
  background: var(--bg1);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.18);
  animation: fadeSlide 0.14s ease;
}

.actionsPopover .actionTile {
  width: 100%;
  justify-content: flex-start;
}

.actionTile {
  display: flex;
  flex-direction: row;
  align-items: center;
  justify-content: center;
  width: 100%;
  min-width: 0;
  height: 28px;
  gap: 5px;
  padding: 0 9px;
  border-radius: var(--r);
  border: 1px solid transparent;
  background: var(--bg3);
  color: var(--t2);
  font-size: 12px;
  font-weight: 500;
  font-family: var(--font-ui);
  letter-spacing: 0.01em;
  cursor: pointer;
  transition: all 0.13s;
  user-select: none;

  svg {
    width: 12px;
    height: 12px;
    flex-shrink: 0;
  }

  &:active:not(:disabled) {
    transform: scale(0.97);
  }

  &:disabled {
    opacity: 0.3;
    cursor: not-allowed;
  }
}

.actionTileLabel {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  min-width: 0;
}

.actionTileDictionary {
  background: var(--violet-dim);
  color: var(--violet);
  border-color: var(--violet-dim);

  &:hover:not(:disabled) { background: var(--violet-dim); border-color: var(--violet); }
}

.actionTileTemplate {
  background: var(--teal-dim);
  color: var(--teal);
  border-color: var(--teal-bdr);

  &:hover:not(:disabled) { background: var(--teal-dim); border-color: var(--teal); }
}

.actionTileSearch {
  background: var(--amber-dim);
  color: var(--amber);
  border-color: var(--amber-bdr);

  &:hover:not(:disabled) { background: var(--amber-dim); border-color: var(--amber); }
}

.actionTileExport {
  background: var(--green-dim);
  color: var(--green);
  border-color: var(--green-bdr);

  &:hover:not(:disabled) { background: var(--green-dim); border-color: var(--green); }
}

.actionTileDelete {
  background: var(--red-dim);
  color: var(--red);
  border-color: var(--red-bdr);

  &:hover:not(:disabled) { background: var(--red-dim); border-color: var(--red); }
}

.actionTileActive {
  outline: 2px solid var(--blue-bdr);
  outline-offset: -1px;
}

// Single-line, icon-led date filter — same 32px height as the rest of the row
.dateFilter {
  display: flex;
  align-items: center;
  height: 32px;
  gap: 6px;
  flex-shrink: 0;
}

.dateIcon {
  width: 14px;
  height: 14px;
  color: var(--t2);
  flex-shrink: 0;
}

.dateField {
  display: flex;
  flex-direction: column;
  gap: 3px;
  width: 118px;
  flex-shrink: 0;
}

.dateLabel {
  font-size: 10px;
  font-weight: 600;
  color: var(--t2);
  text-transform: uppercase;
  letter-spacing: 0.1em;
  @include m.mono;
}

.dateInput {
  width: 115px;
  height: 32px;
  padding: 0 8px;
  background: var(--bg2);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 12.5px;
  outline: none;
  appearance: none;
  transition: border-color 0.12s;
  cursor: pointer;

  &:focus {
    border-color: var(--blue);
    box-shadow: 0 0 0 2px var(--blue-dim);
  }

  &::-webkit-calendar-picker-indicator {
    opacity: 0.7;
    cursor: pointer;
    filter: var(--date-icon-filter);
  }
}

.dateSep {
  font-size: 12px;
  color: var(--t2);
  flex-shrink: 0;
}

// ── Status filter ──────────────────────────────
.statusFilterWrap {
  display: flex;
  align-items: center;
  height: 32px;
  gap: 6px;
  flex-shrink: 0;
}

.statusFilterSelect {
  height: 32px;
  padding: 0 8px;
  background: var(--bg2);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 12.5px;
  outline: none;
  cursor: pointer;
  transition: border-color 0.12s;

  &:focus { border-color: var(--blue); }
  &:disabled { opacity: 0.5; cursor: default; }
}

.applyBtn {
  height: 32px;
  padding: 0 12px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t1);
  font-family: var(--font-ui);
  font-size: 12.5px;
  font-weight: 500;
  cursor: pointer;
  flex-shrink: 0;
  transition: all 0.12s;

  &:hover:not(:disabled) {
    background: var(--bg3);
    border-color: var(--bdr3);
  }

  &:disabled {
    opacity: 0.5;
    cursor: default;
  }
}

// Uploaded file cards
.uploadedBody {
  flex: 1;
  overflow-y: auto;
  padding: 10px;
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
  grid-auto-rows: 240px;
  align-content: start;
  gap: 12px;
  @include m.scrollbar;
}

// listState/errorState (loading, error, empty) span every column so they
// don't get squeezed into a single grid cell
.listState {
  grid-column: 1 / -1;
}

// ── Square file cards ─────────────────────────
.fcard {
  position: relative;
  height: 100%;
  min-width: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: flex-start;
  padding: 50px 16px 40px;
  border-radius: var(--rl);
  border: 1px solid var(--bdr3);
  background: var(--bg1);
  cursor: default;
  text-align: center;
  overflow: hidden;
  transition: all 0.12s;
  user-select: none;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.06);

  // Thin accent strip along the top edge — colored by file type (vtt/srt),
  // gives the card a bit more identity than a flat, uniform border.
  &::before {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    height: 2px;
    background: var(--bdr3);
  }

  &:has(.vtt)::before { background: linear-gradient(90deg, var(--blue), rgba(91, 164, 239, 0.35)); }
  &:has(.srt)::before { background: linear-gradient(90deg, var(--green), rgba(78, 200, 122, 0.35)); }
}

.fcardStatic {
  cursor: default;
}

.fcard:not(.fcardStatic) {
  cursor: pointer;

  &:hover {
    background: var(--bg2);
    border-color: var(--bdr4, var(--bdr3));
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
    transform: translateY(-1px);
  }
}

.fcardActive,
.fcardActiveView {
  background: rgba(91, 164, 239, 0.06);
  border-color: var(--blue-bdr);
}

.fcardActiveDelete {
  background: var(--red-dim);
  border-color: var(--red-bdr);
}

.fcardActiveExport {
  background: var(--green-dim);
  border-color: var(--green-bdr);
}

.fcardCheck {
  position: absolute;
  top: 10px;
  left: 10px;
  z-index: 1;
}

// Dictionary / prompt-template association icons — mirrors the checkbox's
// corner, opposite side, and only renders when at least one is linked.
.fcardLinks {
  position: absolute;
  top: 10px;
  right: 10px;
  z-index: 1;
  display: flex;
  gap: 4px;
}

.fcardLinkIcon {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 22px;
  height: 22px;
  border-radius: 6px;

  svg { width: 12px; height: 12px; }
}

.fcardLinkIconDict {
  background: var(--amber-dim);
  color: var(--amber);
}

.fcardLinkIconTemplate {
  background: var(--violet-dim);
  color: var(--violet);
}

.fcardIcon {
  position: absolute;
  top: 10px;
  left: 10px;
  z-index: 1;
  width: 36px;
  height: 36px;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 10.5px;
  font-weight: 700;
  transition: top 0.12s;

  &.vtt { background: var(--blue-dim); color: var(--blue); }
  &.srt { background: var(--green-dim); color: var(--green); }
}

// When the selection checkbox is also occupying the top-left corner,
// nudge the ext badge down below it instead of overlapping.
.fcardIconShifted {
  top: 42px;
}

.fcardName {
  width: 100%;
  max-width: 100%;
  min-height: 38px;
  margin-bottom: 8px;
  font-size: 13.5px;
  font-weight: 500;
  color: var(--t0);
  line-height: 1.35;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
  word-break: break-word;
  flex-shrink: 0;
}

.fcardMeta {
  font-size: 10.5px;
  color: var(--t2);
  flex-shrink: 0;
  margin-bottom: 14px;
  @include m.mono;
}

// Compact toolbar of "which content prompts are customized" buttons
// (Summary, Keywords, Questions, short Answer, True/false). Bordered as a
// group + labeled, so it reads as an interactive control rather than a
// row of static status dots.
.fcardPrompts {
  display: flex;
  flex-direction: column;
  gap: 4px;
  flex-shrink: 0;
  margin-top: auto;
  margin-bottom: 10px;
  cursor: default;
}

.fcardPromptsLabel {
  font-size: 9px;
  font-weight: 600;
  color: var(--t3, var(--t2));
  text-transform: uppercase;
  letter-spacing: 0.08em;
}

.fcardPromptsRow {
  display: flex;
  align-items: center;
  gap: 4px;
  padding: 4px;
  border: 1px solid var(--bdr2);
  border-radius: 8px;
  background: var(--bg2);
  width: fit-content;
}

.fcardPromptDot {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 25px;
  height: 25px;
  border-radius: 6px;
  border: 1px solid transparent;
  background: var(--bg1);
  color: var(--t2);
  cursor: pointer;
  transition: all 0.12s;

  svg { width: 13px; height: 13px; }

  &:hover {
    background: var(--bg3);
    border-color: var(--bdr3);
    color: var(--t0);
    transform: translateY(-1px);
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.12);
  }
}

.fcardPromptDotSet {
  background: var(--blue-dim);
  color: var(--blue);
  border-color: var(--blue-bdr);

  &:hover {
    background: var(--blue-dim);
    color: var(--blue);
    border-color: var(--blue);
    opacity: 0.9;
  }
}

.fcardBadgeDot {
  width: 5px;
  height: 5px;
  border-radius: 50%;
  background: currentColor;
  animation: fcardPulse 1.4s ease-in-out infinite;
}

@keyframes fcardPulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.35; }
}

.fcardBadge {
  position: absolute;
  bottom: 12px;
  left: 50%;
  transform: translateX(-50%);
  font-size: 10px;
  padding: 3px 8px;
  max-width: calc(100% - 16px);
  overflow: hidden;
  text-overflow: ellipsis;
}

// ── Uploaded card wrap — carries the static green gradient border ──
// ── Uploaded file card — mirrors HistoryPanel .hitm ──────────────
.hitm {
  position: relative;
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 9px;
  border-radius: var(--r);
  border: 1px solid transparent;
  transition: all 0.12s;
  margin-bottom: 3px;
  user-select: none;

  &:hover {
    background: var(--bg2);
    border-color: var(--bdr);
  }

  &.active {
    background: rgba(91, 164, 239, 0.06);
    border-color: var(--blue-bdr);

    &::before {
      content: '';
      position: absolute;
      left: 0;
      top: 6px;
      bottom: 6px;
      width: 3px;
      border-radius: 0 3px 3px 0;
      background: linear-gradient(180deg, var(--blue), #a78bfa);
    }
  }

  // Normal-mode file view highlight — distinct purple/amber accent
  &.activeView {
    background: rgba(167, 139, 250, 0.06);
    border-color: rgba(167, 139, 250, 0.3);

    &::before {
      content: '';
      position: absolute;
      left: 0;
      top: 6px;
      bottom: 6px;
      width: 3px;
      border-radius: 0 3px 3px 0;
      background: linear-gradient(180deg, #a78bfa, var(--amber));
    }
  }

  // Delete mode selected — red accent
  &.activeDelete {
    background: rgba(239, 68, 68, 0.05);
    border-color: rgba(239, 68, 68, 0.3);

    &::before {
      content: '';
      position: absolute;
      left: 0;
      top: 6px;
      bottom: 6px;
      width: 3px;
      border-radius: 0 3px 3px 0;
      background: linear-gradient(180deg, #ef4444, #f97316);
    }
  }

  // Export mode selected — green accent
  &.activeExport {
    background: rgba(78, 200, 122, 0.05);
    border-color: rgba(78, 200, 122, 0.28);

    &::before {
      content: '';
      position: absolute;
      left: 0;
      top: 6px;
      bottom: 6px;
      width: 3px;
      border-radius: 0 3px 3px 0;
      background: linear-gradient(180deg, #4ec87a, #38c4ba);
    }
  }
}

.hitmSelectable {
  cursor: pointer;
}

// ── Ext icon ──
.ficon {
  width: 26px;
  height: 26px;
  border-radius: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 10px;
  font-weight: 700;
  @include m.mono;
  flex-shrink: 0;

  &.vtt {
    background: var(--blue-dim);
    color: var(--blue);
  }

  &.srt {
    background: var(--green-dim);
    color: var(--green);
  }
}

// ── File info ──
.hi {
  flex: 1;
  min-width: 0;
}

.hn {
  font-size: 13px;
  font-weight: 500;
  color: var(--t0);
  @include m.truncate;
}

.hm {
  font-size: 12px;
  color: var(--t2);
  margin-top: 2px;
  @include m.mono;
}

// Empty / loading / error state
.listState {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 10px;
  padding: 32px 16px;
  font-size: 13px;
  color: var(--t2);
  @include m.mono;
  text-align: center;

  svg {
    width: 18px;
    height: 18px;
    opacity: 0.5;
  }
}

.errorState {
  color: var(--red);

  svg {
    opacity: 0.7;
  }
}

.spinner {
  width: 18px;
  height: 18px;
  border: 2px solid var(--bdr2);
  border-top-color: var(--blue);
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

// ── Search icon button ──────────────────────────
.searchIconBtn {
  width: 28px;
  height: 28px;
  border-radius: 7px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  flex-shrink: 0;
  transition: all 0.15s;

  svg {
    width: 13px;
    height: 13px;
  }

  &:hover {
    background: var(--bg3);
    border-color: var(--bdr3);
    color: var(--t0);
  }
}

.searchIconBtnActive {
  border-color: var(--blue-bdr);
  background: var(--blue-dim);
  color: var(--blue);

  &:hover {
    background: var(--blue-dim);
    border-color: var(--blue);
    color: var(--blue);
  }
}

// ── Search bar (slides in below sortHeader) ──────
.searchBar {
  display: grid;
  grid-template-rows: 0fr;
  opacity: 0;
  transition:
    grid-template-rows 0.22s cubic-bezier(0.4, 0, 0.2, 1),
    opacity 0.18s ease;
  overflow: hidden;
  padding: 0 10px;
}

.searchBarOpen {
  grid-template-rows: 1fr;
  opacity: 1;
  padding: 6px 10px 4px;
}

// Flex row: input box + close button — right-aligned, capped width so
// it doesn't stretch across the whole panel.
.searchBarRow {
  min-height: 0;
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 6px;
}

.searchInner {
  flex: 0 1 320px;
  min-width: 0;
  min-height: 0;
  display: flex;
  align-items: center;
  gap: 6px;
  background: var(--bg2);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  padding: 0 8px;
  transition: border-color 0.15s, box-shadow 0.15s;

  &:focus-within {
    border-color: var(--blue);
    box-shadow: 0 0 0 2px var(--blue-dim);
  }
}

.searchBarIcon {
  width: 12px;
  height: 12px;
  flex-shrink: 0;
  color: var(--t2);
  opacity: 0.6;
}

.searchInput {
  flex: 1;
  min-width: 0;
  padding: 6px 0;
  background: transparent;
  border: none;
  outline: none;
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 13px;

  &::placeholder {
    color: var(--t2);
    opacity: 0.55;
  }
}

.searchCloseBtn {
  width: 20px;
  height: 20px;
  border-radius: 5px;
  border: 1px solid rgba(239, 68, 68, 0.3);
  background: rgba(239, 68, 68, 0.08);
  color: #ef4444;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  flex-shrink: 0;
  transition: all 0.12s;

  svg {
    width: 8px;
    height: 8px;
  }

  &:hover {
    background: rgba(239, 68, 68, 0.18);
    border-color: rgba(239, 68, 68, 0.6);
    box-shadow: 0 0 6px rgba(239, 68, 68, 0.2);
  }
}

.searchCount {
  font-size: 11px;
  color: var(--t2);
  font-family: var(--font-mono);
  white-space: nowrap;
  flex-shrink: 0;
}


.badge {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-size: 12px;
  padding: 2px 8px;
  border-radius: 99px;
  font-weight: 500;
  border: 1px solid transparent;
  white-space: nowrap;
  @include m.mono;
}

.bReady {
  background: var(--green-dim);
  color: var(--green);
  border-color: var(--green-bdr);
}

// Inferenced — green (file has completed inference)
.bInferred {
  background: var(--green-dim);
  color: var(--green);
  border-color: var(--green-bdr);
}

// Not Inferenced — amber (file uploaded but not yet processed)
.bNotInferred {
  background: rgba(240, 160, 48, 0.1);
  color: var(--amber);
  border-color: rgba(240, 160, 48, 0.3);
}

// Running — blue, with the pulsing dot from fcardBadgeDot
.bRunning {
  background: var(--blue-dim);
  color: var(--blue);
  border-color: var(--blue-bdr);
}

// Queued — amber, waiting its turn in the batch
.bQueued {
  background: rgba(240, 160, 48, 0.1);
  color: var(--amber);
  border-color: rgba(240, 160, 48, 0.3);
}

// Error — red, inference failed for this file
.bError {
  background: var(--red-dim);
  color: var(--red);
  border-color: var(--red-bdr);
}

// Waiting — neutral gray, distinct from "not inferenced" (amber)
.bWaiting {
  background: var(--bg2);
  color: var(--t2);
  border-color: var(--bdr2);
}

.bSelected {
  background: var(--blue-dim);
  color: var(--blue);
  border-color: var(--blue-bdr);
  font-weight: 600;
}

.bDelete {
  background: rgba(239, 68, 68, 0.1);
  color: #ef4444;
  border-color: rgba(239, 68, 68, 0.3);
  font-weight: 600;
}

.bExport {
  background: var(--green-dim);
  color: var(--green);
  border-color: var(--green-bdr);
  font-weight: 600;
  gap: 4px;
}

.bInfo {
  background: var(--blue-dim);
  color: var(--blue);
  border-color: var(--blue-bdr);
}

// ── Buttons ───────────────────────────────────
.btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 5px;
  padding: 6px 13px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t1);
  font-family: var(--font-ui);
  font-size: 13px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.12s;
  white-space: nowrap;
  user-select: none;

  svg {
    width: 11px;
    height: 11px;
  }

  &:hover {
    background: var(--bg3);
    color: var(--t0);
    border-color: var(--bdr3);
  }

  &:disabled {
    opacity: 0.45;
    cursor: default;
  }
}

.btnP {
  background: var(--blue);
  color: #fff;
  border-color: var(--blue);
  font-weight: 600;

  &:hover {
    background: #a78bfa;
    border-color: #a78bfa;
    color: #fff;
  }
}

.btnSm {
  padding: 4px 10px;
  font-size: 13px;
}

.btnFull {
  flex: 1;
}

.btnDanger {
  color: var(--red);
  border-color: var(--red-bdr);

  &:hover {
    background: var(--red-dim);
    border-color: var(--red);
  }

  &:disabled {
    color: var(--t2);
    border-color: var(--bdr2);
    background: transparent;
    opacity: 0.4;
    cursor: not-allowed;
    pointer-events: none;

    &:hover {
      background: transparent;
      border-color: var(--bdr2);
    }
  }
}

// ── Animations ───────────────────────────────
@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

@keyframes fadeSlide {
  from {
    opacity: 0;
    transform: translateY(4px);
  }

  to {
    opacity: 1;
    transform: translateY(0);
  }
}

// ── Checkbox component ──────────────────────────
.cb {
  width: 14px;
  height: 14px;
  border: 1.5px solid var(--cb-bdr);
  border-radius: 4px;
  flex-shrink: 0;
  cursor: pointer;
  transition: all 0.12s;
  position: relative;
  outline: none;

  &:hover:not(.cbDisabled) {
    border-color: var(--blue);
  }

  &:focus-visible {
    box-shadow: 0 0 0 2px var(--blue-dim);
  }

  &.cbChecked {
    background: var(--blue);
    border-color: var(--blue);

    &::after {
      content: '';
      position: absolute;
      left: 2px;
      top: 5px;
      width: 7px;
      height: 4px;
      border-left: 1.5px solid #fff;
      border-bottom: 1.5px solid #fff;
      transform: rotate(-45deg) translate(0, -1px);
    }
  }

  &.cbIndet {
    background: var(--blue);
    border-color: var(--blue);

    &::after {
      content: '';
      position: absolute;
      left: 2px;
      top: 5px;
      width: 8px;
      height: 1.5px;
      background: #fff;
    }
  }

  &.cbDisabled {
    opacity: 0.35;
    cursor: not-allowed;
    pointer-events: none;
  }
}

// Grouped action buttons — used inside mode-bar (title row)
.headerActions {
  display: flex;
  align-items: center;
  gap: 4px;
  margin-left: auto;
  flex-shrink: 0;
}

// ── Mode action row: search LEFT, confirm+cancel RIGHT ────────
.modeActionsRow {
  display: flex;
  align-items: center;
  box-sizing: border-box;
  width: 100%;
  padding: 0 10px 8px;
  min-width: 0;
}

.modeActionsRight {
  display: flex;
  align-items: center;
  gap: 4px;
  margin-left: auto;
  flex-shrink: 0;
}

.modeActionsLeft {
  display: flex;
  align-items: center;
  gap: 4px;
  min-width: 0;
  flex-wrap: wrap;
}

// Base tile — icon + label side by side
.modeTile {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 4px 9px;
  border-radius: 6px;
  border: 1px solid var(--bdr);
  background: var(--bg3);
  color: var(--t2);
  font-size: 11px;
  font-weight: 500;
  font-family: var(--font-ui);
  cursor: pointer;
  transition: all 0.13s;
  user-select: none;
  white-space: nowrap;

  svg {
    width: 12px;
    height: 12px;
    flex-shrink: 0;
  }

  &:active:not(:disabled) {
    transform: scale(0.97);
  }

  &:disabled {
    opacity: 0.3;
    cursor: not-allowed;
  }
}

// Search tile — amber tint (left side)
.modeTileSearch {
  color: var(--amber);
  border-color: rgba(240, 160, 48, 0.25);
  background: rgba(240, 160, 48, 0.08);

  &:hover {
    background: rgba(240, 160, 48, 0.16);
    border-color: rgba(240, 160, 48, 0.45);
  }
}

.modeTileSearchActive {
  background: rgba(240, 160, 48, 0.15);
  border-color: rgba(240, 160, 48, 0.45);
}

// Confirm export — green
.modeTileConfirmExport {
  color: var(--green);
  border-color: var(--green-bdr);
  background: var(--green-dim);

  &:hover:not(:disabled) {
    background: var(--green);
    border-color: var(--green);
    color: #fff;
  }
}

// Confirm delete — red
.modeTileConfirmDelete {
  color: #ef4444;
  border-color: rgba(239, 68, 68, 0.25);
  background: rgba(239, 68, 68, 0.08);

  &:hover:not(:disabled) {
    background: rgba(239, 68, 68, 0.18);
    border-color: rgba(239, 68, 68, 0.5);
  }
}

// Cancel
.modeTileCancel {
  color: var(--t2);
  border-color: var(--bdr2);
  background: transparent;

  &:hover {
    background: var(--bg2);
    border-color: var(--bdr3);
    color: var(--t0);
  }
}

// Disabled state
.modeTileDisabled {
  opacity: 0.35 !important;
  cursor: not-allowed !important;
}

// "Export all files" trigger — mirrors modeTileDeleteAll but green/non-destructive
.modeTileExportAll {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 4px 9px;
  border-radius: 6px;
  border: 1px solid var(--green-bdr);
  background: var(--green-dim);
  color: var(--green);
  font-size: 11px;
  font-weight: 600;
  font-family: var(--font-ui);
  cursor: pointer;
  transition: all 0.13s;
  user-select: none;
  white-space: nowrap;

  svg {
    width: 12px;
    height: 12px;
    flex-shrink: 0;
  }

  &:hover:not(:disabled) {
    background: var(--green);
    border-color: var(--green);
    color: #fff;
  }

  &:disabled {
    opacity: 0.35;
    cursor: not-allowed;
  }
}

.modeActionsError {
  font-size: 11px;
  color: #ef4444;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 160px;
}

// "Delete all files" trigger — deliberately louder/more solid than the
// per-row delete tile since it's an account-wide destructive action.
.modeTileDeleteAll {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 4px 9px;
  border-radius: 6px;
  border: 1px solid rgba(239, 68, 68, 0.5);
  background: rgba(239, 68, 68, 0.14);
  color: #ef4444;
  font-size: 11px;
  font-weight: 600;
  font-family: var(--font-ui);
  cursor: pointer;
  transition: all 0.13s;
  user-select: none;
  white-space: nowrap;

  svg {
    width: 12px;
    height: 12px;
    flex-shrink: 0;
  }

  &:hover:not(:disabled) {
    background: #ef4444;
    border-color: #ef4444;
    color: #fff;
  }

  &:disabled {
    opacity: 0.3;
    cursor: not-allowed;
  }
}

// ── Delete-ALL confirmation (danger zone) ───────────────────────────
.dangerOverlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.55);
  backdrop-filter: blur(2px);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 200;
  padding: 16px;
}

.dangerModal {
  width: 100%;
  max-width: 360px;
  background: var(--bg1);
  border: 1px solid rgba(239, 68, 68, 0.4);
  border-radius: 10px;
  padding: 20px;
  box-shadow: 0 12px 40px rgba(0, 0, 0, 0.4);
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  gap: 4px;
}

.dangerIcon {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background: rgba(239, 68, 68, 0.14);
  color: #ef4444;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 6px;

  svg {
    width: 22px;
    height: 22px;
  }
}

.dangerTitle {
  font-size: 15px;
  font-weight: 700;
  color: var(--t0);
  font-family: var(--font-ui);
}

.dangerBody {
  font-size: 13px;
  color: var(--t2);
  line-height: 1.5;
  margin-bottom: 8px;
}

.dangerLabel {
  align-self: flex-start;
  font-size: 12px;
  color: var(--t1);
  margin-top: 4px;
}

.dangerInput {
  width: 100%;
  padding: 7px 10px;
  background: var(--bg0);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 13px;
  outline: none;
  margin-top: 4px;
  margin-bottom: 6px;
  text-align: center;
  transition: border-color 0.12s;

  &:focus {
    border-color: #ef4444;
    box-shadow: 0 0 0 2px rgba(239, 68, 68, 0.15);
  }

  &:disabled {
    opacity: 0.6;
  }
}

.dangerError {
  font-size: 12px;
  color: #ef4444;
  margin-bottom: 6px;
}

.dangerActions {
  display: flex;
  gap: 8px;
  width: 100%;
  margin-top: 6px;
}

// .uploadedCardSel replaced by .hitm.active

// Uploaded card disabled (batch running)
.uploadedCardDisabled {
  cursor: default !important;
  opacity: 0.65;
  pointer-events: none;
}

// ── Sort header ─────────────────────────────────────
.sortHeader {
  display: flex;
  align-items: center;
  height: 32px;
  background: var(--bg2);
  border: 1px solid var(--bdr);
  border-radius: var(--r);
  padding: 0 6px 0 10px;
  gap: 6px;
  flex-shrink: 0;
  overflow: hidden;
}

.sortHeaderLabel {
  display: flex;
  align-items: center;
  gap: 5px;
  font-size: 10px;
  font-weight: 600;
  color: var(--t2);
  font-family: var(--font-mono);
  text-transform: uppercase;
  letter-spacing: 0.06em;
  white-space: nowrap;
  flex-shrink: 0;

  svg {
    width: 11px;
    height: 11px;
    opacity: 0.6;
  }
}

.sortCols {
  display: flex;
  align-items: center;
  gap: 2px;
}

.sortCol {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 4px;
  height: 24px;
  padding: 0 8px;
  background: transparent;
  border: none;
  border-radius: 5px;
  color: var(--t2);
  font-size: 12px;
  font-family: var(--font-mono);
  cursor: pointer;
  transition: all 0.12s;
  white-space: nowrap;

  svg {
    width: 8px;
    height: 10px;
    flex-shrink: 0;
    transition: transform 0.2s;
  }

  &:hover {
    background: var(--bg3);
    color: var(--t1);
  }
}

.sortColActive {
  background: var(--blue-dim);
  color: var(--blue);
  font-weight: 600;

  &:hover {
    background: var(--blue-dim);
  }
}

.sortInactive {
  opacity: 0.25;
}

.sortAsc {
  transform: rotate(180deg);
}

// arrow up
.sortDesc {
  transform: rotate(0deg);
}

// arrow down (default path direction)
// ── Small screen overrides (height < 1000px) ─────────────────
@media (max-height: 999px) {

  // Step-1 bar — tighter
  .step1Bar {
    padding: 7px 14px;
  }

  // Dropzone — much more compact
  .dropzone {
    padding: 10px 16px;
    margin: 6px 10px;
  }

  .dzIc {
    width: 28px;
    height: 28px;
    margin-bottom: 6px;

    svg {
      width: 14px;
      height: 14px;
    }
  }

  .dzTitle {
    font-size: 12px;
    margin-bottom: 2px;
  }

  .dzSub {
    font-size: 11px;
  }

  .dzActions {
    margin-top: 7px;
    gap: 6px;
  }

  // Section2 title row
  .section2TitleRow {
    padding: 7px 14px 6px;
  }

  // Mode action row
  .modeActionsRow {
    padding: 0 8px 6px;
  }

  // Date filter — tighter
  .dateFilter {
    padding: 5px 10px 5px;
    gap: 5px;
  }

  .dateInput {
    padding: 3px 6px;
    font-size: 12px;
  }

  // Sort header — tighter
  .sortHeader {
    margin-bottom: 4px;
  }

  .sortCol {
    padding: 4px 4px;
    font-size: 11px;
  }

  .sortHeaderLabel {
    padding: 4px 0;
    padding-right: 6px;
    font-size: 9px;
  }

  // File list — tighter items, ensure it can grow
  .uploadedBody {
    padding: 4px 8px;
    gap: 2px;
  }

  .hitm {
    padding: 5px 8px;
    margin-bottom: 1px;
  }

  .ficon {
    width: 22px;
    height: 22px;
    font-size: 9px;
  }

  .hn {
    font-size: 12px;
  }

  .hm {
    font-size: 11px;
    margin-top: 1px;
  }

  .badge {
    font-size: 10px;
    padding: 1px 6px;
  }
}

// ── Pagination footer (bottom of the uploaded-files list) ──────────
.paginationBar {
  display: flex;
  flex-direction: column;
  gap: 6px;
  padding: 8px 10px;
  border-top: 1px solid var(--bdr2);
  background: var(--bg1);
  flex-shrink: 0;
}

.paginationTopRow {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  flex-wrap: nowrap;
  min-width: 0;
}

.pageSizeGroup {
  display: flex;
  align-items: center;
  gap: 6px;
  flex-shrink: 0;
}

.pageSizeLabel {
  font-size: 11px;
  color: var(--t2);
  white-space: nowrap;
}

.pageSizeSelect {
  padding: 3px 6px;
  background: var(--bg0);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 12px;
  outline: none;
  cursor: pointer;
  transition: border-color 0.12s;

  &:focus {
    border-color: var(--blue);
    box-shadow: 0 0 0 2px var(--blue-dim);
  }
}

// Page-number row sits to the right of the page-size selector. It never
// wraps to its own line — if there isn't room for every number button,
// this row scrolls horizontally within itself instead.
.pageNav {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 2px;
  flex-wrap: nowrap;
  flex-shrink: 1;
  min-width: 0;
  overflow-x: auto;
  scrollbar-width: none;
  -ms-overflow-style: none;

  &::-webkit-scrollbar {
    display: none;
  }
}

.pageNavBtn,
.pageNumBtn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 24px;
  height: 24px;
  padding: 0 6px;
  border-radius: var(--r);
  border: 1px solid transparent;
  background: transparent;
  color: var(--t1);
  font-family: var(--font-ui);
  font-size: 12px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.12s;
  user-select: none;
  flex-shrink: 0;

  svg {
    width: 11px;
    height: 11px;
  }

  &:hover:not(:disabled) {
    background: var(--bg3);
    color: var(--t0);
    border-color: var(--bdr3);
  }

  &:disabled {
    opacity: 0.35;
    cursor: default;
  }
}

.pageNumBtnActive {
  background: var(--blue-dim);
  border-color: var(--blue-bdr);
  color: var(--blue);
  font-weight: 700;

  &:hover:not(:disabled) {
    background: var(--blue-dim);
    color: var(--blue);
  }
}

.pageEllipsis {
  color: var(--t2);
  font-size: 12px;
  padding: 0 2px;
  user-select: none;
  flex-shrink: 0;
}

.pageInfo {
  font-size: 11px;
  color: var(--t2);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  text-align: right;
  @include m.mono;
}

@media (min-width: 1920px) {
  .section1Title {
    font-size: 14px;
  }

  .section2Title {
    font-size: 14px;
  }

  .uploadHint {
    font-size: 13px;
  }

  .uploadZoneText {
    font-size: 14px;
  }

  .sortHeaderLabel {
    font-size: 12px;
  }

  .sortBtn {
    font-size: 13px;
  }

  .fileName {
    font-size: 14px;
  }

  .fileMeta {
    font-size: 12px;
  }

  .fileDate {
    font-size: 12px;
  }

  .badge {
    font-size: 13px;
  }

  .btn {
    font-size: 13px;
  }

  .listState {
    font-size: 14px;
  }

  .searchInput {
    font-size: 14px;
  }

  .searchCount {
    font-size: 12px;
  }

  .dateLabel {
    font-size: 12px;
  }

  .dateInput {
    font-size: 13px;
  }

  .emptyState {
    font-size: 14px;
  }
}

// ── Per-file prompt viewer popup ─────────────────
.promptViewerModal {
  width: 100%;
  max-width: 480px;
  max-height: 70vh;
  display: flex;
  flex-direction: column;
  background: var(--bg1);
  border: 1px solid var(--bdr2);
  border-radius: var(--rl);
  box-shadow: var(--shadow);
  overflow: hidden;
}

.promptViewerHead {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 10px;
  padding: 14px 16px;
  border-bottom: 1px solid var(--bdr);
  flex-shrink: 0;
}

.promptViewerTitle {
  font-size: 13.5px;
  font-weight: 600;
  color: var(--t0);
}

.promptViewerFile {
  margin-top: 2px;
  font-size: 12px;
  color: var(--t2);
  @include m.truncate;
}

.promptViewerClose {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 26px;
  height: 26px;
  flex-shrink: 0;
  border-radius: 6px;
  border: none;
  background: transparent;
  color: var(--t2);
  cursor: pointer;

  svg { width: 13px; height: 13px; }

  &:hover {
    background: var(--bg3);
    color: var(--t0);
  }
}

.promptViewerBody {
  padding: 16px;
  overflow-y: auto;
  font-size: 13px;
  line-height: 1.6;
  color: var(--t1);
  white-space: pre-wrap;
  word-break: break-word;
  @include m.scrollbar;
}

.promptViewerEmpty {
  color: var(--t2);
  font-style: italic;
}
















// ═══════════════════════════════════════════════
// FileSidebar.module.scss
// Content Analytics · Shared sidebar for the Infer + Workspace tabs
// ═══════════════════════════════════════════════
@use '../../styles/mixins' as m;

.sidebar {
  width: 400px;
  flex-shrink: 0;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  background: var(--bg0);
  border-right: 1px solid var(--bdr);

  :global(html.light) & {
    background: #fff;
  }

  @media (max-width: 1499px) {
    width: 350px;
  }

  // On very large screens, everything (text, padding, borders, icons)
  // was tuned for ~1440-1900px and starts to look small. `zoom` scales
  // the whole subtree proportionally and reflows layout correctly
  // (unlike transform: scale, which doesn't affect layout/hit-testing).
  @media (min-width: 1900px) {
    zoom: 1.25;
  }
}

// ── Date range filter ─────────────────────────
.dateFilter {
  display: flex;
  align-items: flex-end;
  gap: 6px;
  padding: 12px 14px;
  flex-shrink: 0;
  border-bottom: 1px solid var(--bdr);
}

.dateField {
  display: flex;
  flex-direction: column;
  gap: 3px;
  flex: 1;
  min-width: 0;
}

.dateLabel {
  font-size: 10px;
  font-weight: 600;
  color: var(--t2);
  text-transform: uppercase;
  letter-spacing: 0.1em;
  @include m.mono;
}

.dateInput {
  width: 100%;
  padding: 4px 7px;
  background: var(--bg1);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 13px;
  outline: none;
  appearance: none;
  transition: border-color 0.12s;
  cursor: pointer;

  &:focus {
    border-color: var(--blue);
    box-shadow: 0 0 0 2px var(--blue-dim);
  }

  &::-webkit-calendar-picker-indicator {
    opacity: 0.7;
    cursor: pointer;
    filter: var(--date-icon-filter);
  }
}

.dateSep {
  font-size: 13px;
  color: var(--t2);
  padding-bottom: 4px;
  flex-shrink: 0;
}

// ── Status filter ──────────────────────────────
// Status filter + search box share one row to avoid stacking two
// nearly-empty lines on top of each other.
.filterSearchRow {
  display: flex;
  align-items: center;
  gap: 8px;
  margin: 10px 14px 10px;
  flex-shrink: 0;
}

.statusFilterWrap {
  display: flex;
  align-items: center;
  gap: 6px;
  flex: 0 0 auto;
  width: 132px;
  flex-shrink: 0;
}

.statusFilterIcon {
  width: 14px;
  height: 14px;
  color: var(--t2);
  flex-shrink: 0;
}

.statusFilterSelect {
  flex: 1;
  height: 30px;
  padding: 0 8px;
  background: var(--bg1);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 12.5px;
  outline: none;
  cursor: pointer;
  transition: border-color 0.12s;

  &:focus { border-color: var(--blue); }
  &:disabled { opacity: 0.5; cursor: default; }
}

.applyBtn {
  align-self: flex-end;
  padding: 4px 12px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t1);
  font-family: var(--font-ui);
  font-size: 13px;
  font-weight: 500;
  cursor: pointer;
  flex-shrink: 0;
  transition: all 0.12s;

  &:hover:not(:disabled) {
    background: var(--bg3);
    border-color: var(--bdr3);
  }

  &:disabled {
    opacity: 0.6;
    cursor: default;
  }
}

.miniSpinner {
  display: inline-block;
  width: 12px;
  height: 12px;
  border: 2px solid var(--bdr3);
  border-top-color: var(--t1);
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
}

.headSpinner {
  display: inline-block;
  width: 11px;
  height: 11px;
  border: 2px solid var(--bdr3);
  border-top-color: var(--blue);
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
  flex-shrink: 0;
}

.rowsSpinner {
  display: inline-block;
  width: 18px;
  height: 18px;
  border: 2.5px solid var(--bdr3);
  border-top-color: var(--blue);
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
  margin-bottom: 8px;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

// ── List header ────────────────────────────────
.head {
  @include m.flex-center;
  gap: 8px;
  padding: 12px 14px 8px;
  flex-shrink: 0;
}

.headTitle {
  font-size: 12px;
  font-weight: 600;
  color: var(--t1);
}

.headCount {
  font-size: 11px;
  font-weight: 600;
  color: var(--t2);
  background: var(--bg2);
  border-radius: 99px;
  padding: 1px 7px;
}

.headSelected {
  margin-left: auto;
  font-size: 11px;
  color: var(--blue);
}

// ── Search ─────────────────────────────────────
.searchWrap {
  position: relative;
  flex: 1;
  min-width: 0;
}

.searchIcon {
  position: absolute;
  left: 9px;
  top: 50%;
  transform: translateY(-50%);
  width: 13px;
  height: 13px;
  color: var(--t2);
  pointer-events: none;
}

.searchInput {
  width: 100%;
  height: 30px;
  padding: 0 10px 0 28px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: var(--bg1);
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 12px;

  &::placeholder { color: var(--t2); }
  &:focus { outline: none; border-color: var(--bdr3); }
}

// ── Rows ───────────────────────────────────────
.rowsWrap {
  position: relative;
  flex: 1;
  min-height: 0;
  display: flex;
  flex-direction: column;
}

.rows {
  flex: 1;
  overflow-y: auto;
  padding: 0 8px 12px;
  @include m.scrollbar;
}

.rowsLoadingOverlay {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--bg1);
  background: color-mix(in srgb, var(--bg0) 70%, transparent);
  cursor: wait;
  z-index: 2;
}

.empty {
  padding: 20px 12px;
  text-align: center;
  font-size: 12px;
  color: var(--t2);
}

.emptyLoading {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 24px 12px;
  text-align: center;
  font-size: 12px;
  color: var(--t2);
}

.row {
  display: flex;
  align-items: center;
  gap: 9px;
  padding: 8px 9px;
  border-radius: var(--r);
  color: var(--t1);
  cursor: pointer;
  @include m.theme-transition;

  &:hover { background: var(--bg2); }
}

.rowDisabled {
  opacity: 0.5;
  cursor: default;
  pointer-events: none;
}

// ── File row — mirrors FilePanel's pre-card .hitm design ──────
.hitm {
  position: relative;
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 9px;
  border-radius: var(--r);
  border: 1px solid transparent;
  transition: all 0.12s;
  margin-bottom: 3px;
  user-select: none;

  &:hover {
    background: var(--bg2);
    border-color: var(--bdr);
  }

  &.active {
    background: rgba(91, 164, 239, 0.06);
    border-color: var(--blue-bdr);

    &::before {
      content: '';
      position: absolute;
      left: 0;
      top: 6px;
      bottom: 6px;
      width: 3px;
      border-radius: 0 3px 3px 0;
      background: linear-gradient(180deg, var(--blue), #a78bfa);
    }
  }

  &.activeView {
    background: rgba(167, 139, 250, 0.06);
    border-color: rgba(167, 139, 250, 0.3);

    &::before {
      content: '';
      position: absolute;
      left: 0;
      top: 6px;
      bottom: 6px;
      width: 3px;
      border-radius: 0 3px 3px 0;
      background: linear-gradient(180deg, #a78bfa, var(--amber));
    }
  }
}

.hitmSelectable {
  cursor: pointer;
}

.ficon {
  width: 26px;
  height: 26px;
  border-radius: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 10px;
  font-weight: 700;
  @include m.mono;
  flex-shrink: 0;

  &.vtt { background: var(--blue-dim); color: var(--blue); }
  &.srt { background: var(--green-dim); color: var(--green); }
}

.hi {
  flex: 1;
  min-width: 0;
}

.hn {
  font-size: 13px;
  font-weight: 500;
  color: var(--t0);
  @include m.truncate;
}

.hm {
  font-size: 12px;
  color: var(--t2);
  margin-top: 2px;
  @include m.mono;
}

.badge {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-size: 11px;
  padding: 2px 7px;
  border-radius: 99px;
  font-weight: 500;
  border: 1px solid transparent;
  white-space: nowrap;
  flex-shrink: 0;
  @include m.mono;
}

.bSelected {
  background: var(--blue-dim);
  color: var(--blue);
  border-color: var(--blue-bdr);
}

.bInferred {
  background: var(--green-dim);
  color: var(--green);
  border-color: var(--green-bdr);
}

.bNotInferred {
  background: rgba(240, 160, 48, 0.1);
  color: var(--amber);
  border-color: rgba(240, 160, 48, 0.3);
}

.bRunning {
  background: var(--blue-dim);
  color: var(--blue);
  border-color: var(--blue-bdr);
}

.bQueued {
  background: rgba(240, 160, 48, 0.1);
  color: var(--amber);
  border-color: rgba(240, 160, 48, 0.3);
}

// Waiting — neutral gray, distinct from "not inferenced" (amber)
.bWaiting {
  background: var(--bg2);
  color: var(--t2);
  border-color: var(--bdr2);
}

.badgeDot {
  width: 5px;
  height: 5px;
  border-radius: 50%;
  background: currentColor;
  animation: fsBadgePulse 1.4s ease-in-out infinite;
}

@keyframes fsBadgePulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.35; }
}

// ── Checkbox (mirrors FilePanel's) ─────────────
.cb {
  width: 15px;
  height: 15px;
  flex-shrink: 0;
  border-radius: 4px;
  border: 1px solid var(--cb-bdr);
  background: transparent;
  cursor: pointer;
  position: relative;
  transition: all 0.12s;

  &:hover { border-color: var(--blue); }
}

.cbChecked {
  background: var(--blue);
  border-color: var(--blue);

  &::after {
    content: '';
    position: absolute;
    left: 4px;
    top: 1px;
    width: 4px;
    height: 8px;
    border: solid white;
    border-width: 0 2px 2px 0;
    transform: rotate(45deg);
  }
}

.cbIndet {
  background: var(--blue);
  border-color: var(--blue);

  &::after {
    content: '';
    position: absolute;
    left: 3px;
    top: 6px;
    width: 7px;
    height: 2px;
    background: white;
  }
}

.cbDisabled {
  opacity: 0.4;
  pointer-events: none;
}

// ── Pagination footer — matches the Upload tab's layout ──
.paginationBar {
  display: flex;
  flex-direction: column;
  gap: 6px;
  padding: 8px 12px;
  border-top: 1px solid var(--bdr);
  flex-shrink: 0;
}

.paginationTopRow {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  flex-wrap: nowrap;
  min-width: 0;
}

.pageSizeGroup {
  display: flex;
  align-items: center;
  gap: 6px;
  flex-shrink: 0;
}

.pageSizeLabel {
  font-size: 11px;
  color: var(--t2);
  white-space: nowrap;
}

.pageSizeSelect {
  padding: 3px 6px;
  background: var(--bg1);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  color: var(--t1);
  font-family: var(--font-ui);
  font-size: 12px;
  outline: none;
  cursor: pointer;
  transition: border-color 0.12s;

  &:focus { border-color: var(--blue); }
}

.pageNav {
  display: flex;
  align-items: center;
  gap: 4px;
}

.pageNavBtn {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 24px;
  height: 24px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: var(--bg1);
  color: var(--t2);
  cursor: pointer;
  transition: all 0.12s;
  flex-shrink: 0;

  svg { width: 13px; height: 13px; }

  &:hover:not(:disabled) {
    background: var(--bg2);
    color: var(--t0);
  }

  &:disabled {
    opacity: 0.35;
    cursor: default;
  }
}

.pageNumBtn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 24px;
  height: 24px;
  padding: 0 6px;
  border-radius: var(--r);
  border: 1px solid transparent;
  background: transparent;
  color: var(--t1);
  font-family: var(--font-ui);
  font-size: 12px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.12s;
  user-select: none;
  flex-shrink: 0;

  &:hover:not(:disabled) {
    background: var(--bg3);
    color: var(--t0);
    border-color: var(--bdr3);
  }

  &:disabled {
    opacity: 0.35;
    cursor: default;
  }
}

.pageNumBtnActive {
  background: var(--blue-dim);
  border-color: var(--blue-bdr);
  color: var(--blue);
  font-weight: 700;

  &:hover:not(:disabled) {
    background: var(--blue-dim);
    color: var(--blue);
  }
}

.pageEllipsis {
  color: var(--t2);
  font-size: 12px;
  padding: 0 2px;
  user-select: none;
  flex-shrink: 0;
}

.pageInfo {
  font-size: 11px;
  color: var(--t2);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  text-align: right;
  @include m.mono;
}

.pageIndicator {
  font-size: 12px;
  color: var(--t2);
  @include m.mono;
  min-width: 44px;
  text-align: center;
}

// ── Sort header — same as the Upload tab's sort chip ──
.sortHeader {
  display: flex;
  align-items: center;
  height: 32px;
  background: var(--bg2);
  border: 1px solid var(--bdr);
  border-radius: var(--r);
  padding: 0 6px 0 10px;
  gap: 6px;
  margin: 0 14px 10px;
  flex-shrink: 0;
  overflow: hidden;
}

.sortHeaderLabel {
  display: flex;
  align-items: center;
  gap: 5px;
  font-size: 10px;
  font-weight: 600;
  color: var(--t2);
  font-family: var(--font-mono);
  text-transform: uppercase;
  letter-spacing: 0.06em;
  white-space: nowrap;
  flex-shrink: 0;

  svg {
    width: 11px;
    height: 11px;
    opacity: 0.6;
  }
}

.sortCols {
  display: flex;
  align-items: center;
  gap: 2px;
  flex: 1;
}

.sortCol {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 4px;
  height: 24px;
  padding: 0 6px;
  background: transparent;
  border: none;
  border-radius: 5px;
  color: var(--t2);
  font-size: 11.5px;
  font-family: var(--font-mono);
  cursor: pointer;
  transition: all 0.12s;
  white-space: nowrap;

  svg {
    width: 8px;
    height: 10px;
    flex-shrink: 0;
    transition: transform 0.2s;
  }

  &:hover:not(:disabled) {
    background: var(--bg3);
    color: var(--t1);
  }

  &:disabled {
    opacity: 0.5;
    cursor: default;
  }
}

.sortColActive {
  background: var(--blue-dim);
  color: var(--blue);
  font-weight: 600;

  &:hover:not(:disabled) {
    background: var(--blue-dim);
  }
}

.sortInactive {
  opacity: 0.25;
}

.sortAsc {
  transform: rotate(180deg);
}
















// ═══════════════════════════════════════════════
// InferencePanel.module.scss
// ═══════════════════════════════════════════════
@use '../../styles/mixins' as m;

// ── Panel shell ──
.infpanel {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  position: relative;

  // On very large screens, everything (text, padding, borders, icons)
  // was tuned for ~1440-1900px and starts to look small. `zoom` scales
  // the whole subtree proportionally and reflows layout correctly
  // (unlike transform: scale, which doesn't affect layout/hit-testing).
  @media (min-width: 1900px) {
    zoom: 1.25;
  }
}

// ── Inference Running label in step header ──
.inferenceRunningLabel {
  background: linear-gradient(90deg, var(--blue), #a78bfa, var(--amber));
  background-size: 200% auto;
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  animation: labelShimmer 2.5s linear infinite;
  font-weight: 700;
}

@keyframes labelShimmer {
  0% {
    background-position: 0% center;
  }

  100% {
    background-position: 200% center;
  }
}

// ── Panel header ──
.infpanelHead {
  padding: 13px 18px 10px;
  border-bottom: none;
  background: var(--bg1);
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  position: relative;

  &::after {
    content: '';
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    height: 1px;
    background: linear-gradient(90deg,
        rgba(139, 92, 246, 0.0) 0%,
        rgba(139, 92, 246, 0.6) 20%,
        rgba(56, 196, 186, 0.7) 50%,
        rgba(240, 160, 48, 0.6) 80%,
        rgba(240, 160, 48, 0.0) 100%);
    pointer-events: none;
  }
}

.slbl {
  font-size: 12px;
  font-weight: 600;
  color: var(--t2);
  text-transform: uppercase;
  letter-spacing: 0.1em;
  @include m.mono;
  margin-bottom: 2px;
}

.selSummary {
  font-size: 13px;
  color: var(--t1);
}

.headActions {
  display: flex;
  align-items: center;
  gap: 8px;
}

.closeStepBtn {
  width: 28px;
  height: 28px;
  border-radius: 7px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  flex-shrink: 0;
  transition: all 0.14s;

  svg {
    width: 13px;
    height: 13px;
  }

  &:hover {
    background: rgba(239, 68, 68, 0.08);
    border-color: rgba(239, 68, 68, 0.3);
    color: #ef4444;
  }
}

// ══════════════════════════════════════
// RUN INFERENCE BUTTON
// ══════════════════════════════════════

.runBtn {
  display: inline-flex;
  align-items: center;
  gap: 10px;
  padding: 8px 16px 8px 10px;
  border-radius: 10px;
  border: 1px solid transparent;
  background: var(--blue);
  color: #ffffff;
  font-family: var(--font-ui);
  cursor: pointer;
  transition: background 0.15s, box-shadow 0.15s, opacity 0.15s, border-color 0.15s;
  white-space: nowrap;
  user-select: none;
  flex-shrink: 0;

  &:hover:not(:disabled) {
    background: #6bb3f5;
  }

  &:active:not(:disabled) {
    background: #4a90d9;
  }
}

// Icon chip inside the button
.runBtnIconWrap {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 30px;
  height: 30px;
  border-radius: 7px;
  background: rgba(255, 255, 255, 0.18);
  flex-shrink: 0;
  transition: background 0.14s;

  svg {
    display: block;
  }

  .runBtn:hover:not(:disabled) & {
    background: rgba(255, 255, 255, 0.25);
  }
}

// Two-line text stack
.runBtnText {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  line-height: 1.2;
  gap: 2px;
}

.runBtnTitle {
  font-size: 14px;
  font-weight: 700;
  letter-spacing: -0.1px;
}

.runBtnSub {
  font-size: 11px;
  font-weight: 500;
  opacity: 0.82;
  font-family: var(--font-mono);
}

// Ready state — glowing outline pulse to draw the eye
.runBtnReady {
  box-shadow:
    0 0 0 1px rgba(91, 164, 239, 0.7),
    0 2px 12px rgba(91, 164, 239, 0.35);
  animation: runReadyPulse 2.4s ease-in-out infinite;
}

@keyframes runReadyPulse {

  0%,
  100% {
    box-shadow:
      0 0 0 1px rgba(91, 164, 239, 0.7),
      0 2px 12px rgba(91, 164, 239, 0.30);
  }

  50% {
    box-shadow:
      0 0 0 2px rgba(91, 164, 239, 0.55),
      0 4px 20px rgba(91, 164, 239, 0.50);
  }
}

// Disabled state — muted, clearly not actionable, sub-label explains why
.runBtnDisabled {
  opacity: 0.48;
  cursor: not-allowed;
  pointer-events: none;
  box-shadow: none;
}

// Large screen tweaks
@media (min-width: 1920px) {
  .runBtnTitle {
    font-size: 15px;
  }

  .runBtnSub {
    font-size: 12px;
  }

  .runBtnIconWrap {
    width: 32px;
    height: 32px;
  }

  .runBtn {
    padding: 9px 18px 9px 11px;
    gap: 11px;
    border-radius: 11px;
  }
}

// ── Main body ──
.infpanelBody {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  padding: 14px 18px 0;
}

// Configuration mode — body itself scrolls, batch section is absent
.infpanelBodyConfig {
  overflow-y: auto;
  @include m.scrollbar;
}

// Running mode — no scroll on body; each batch column scrolls independently
.infpanelBodyRunning {
  overflow: hidden;
}

// ── Selection banner ──
.selBanner {
  background: var(--bg2);
  border: 1px solid var(--blue-bdr);
  border-radius: var(--rl);
  padding: 9px 13px;
  margin-bottom: 13px;
  display: flex;
  align-items: center;
  gap: 9px;
}

.selCt {
  font-size: 13px;
  color: var(--blue);
  font-weight: 600;
  @include m.mono;
  flex-shrink: 0;
}

.selNm {
  font-size: 12px;
  color: var(--t2);
  @include m.mono;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  flex: 1;
}

// ── Collapsible settings wrap ──
.infSettingsWrap {
  padding-bottom: 16px;
  flex-shrink: 0;
}

.settingsGrid {
  display: grid;
  grid-template-columns: 1fr;
  gap: 11px;
  margin-bottom: 12px;
}

// 2 fields per row inside the merged Generate content card
.fieldsGrid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 4px 20px;
  align-items: start;
}

.fieldCell {
  min-width: 0;
}

// Model dropdown spans both columns
.modelFieldTop {
  width: 260px;
  max-width: 100%;
  padding-bottom: 14px;
  margin-bottom: 14px;
  border-bottom: 1px solid var(--bdr);
}

.promptGrid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 11px;
}

.promptStack {
  display: flex;
  flex-direction: column;
  gap: 11px;
}

.noPromptHint {
  font-size: 13px;
  color: var(--t2);
  font-family: var(--font-mono);
  text-align: center;
  padding: 10px 0 2px;
}

// ── Card ──
.card {
  background: var(--bg1);
  border: 1px solid var(--bdr);
  border-radius: var(--rl);
  padding: 14px 16px;
}

.cardT {
  font-size: 10px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  color: var(--t2);
  margin-bottom: 12px;
  @include m.mono;
}

.promptMode {
  font-size: 12px;
  margin-left: 8px;
  font-weight: 400;
  text-transform: none;
  letter-spacing: 0;
  @include m.mono;
}

// ── Form ──
.fg {
  margin-bottom: 11px;
}

// Indents a prompt block under the checkbox it belongs to (same 20px
// indent convention as .nestedField, used for the timestamp interval).
.promptInline {
  padding-left: 20px;
  margin-bottom: 4px;
  cursor: default;
}

.mb0 {
  margin-bottom: 0 !important;
}

.mb8 {
  margin-bottom: 8px;
}

.mb12 {
  margin-bottom: 12px !important;
}

.flRow {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-bottom: 4px;
}

.fl {
  display: block;
  font-size: 13px;
  color: var(--t1);
  margin-bottom: 0;
  font-weight: 500;
}

.fc {
  width: 100%;
  padding: 7px 11px;
  background: var(--bg0);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 13px;
  transition: border-color 0.12s;
  outline: none;
  appearance: none;

  &:focus {
    border-color: var(--blue);
    box-shadow: 0 0 0 3px rgba(91, 164, 239, 0.07);
  }

  &::placeholder {
    color: var(--t2);
  }
}

select.fc {
  cursor: pointer;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='10' height='7'%3E%3Cpath d='M1 1l4 4 4-4' stroke='%234e5668' stroke-width='1.5' fill='none' stroke-linecap='round'/%3E%3C/svg%3E");
  background-repeat: no-repeat;
  background-position: right 10px center;
  padding-right: 26px;
}

textarea.fc {
  resize: vertical;
  min-height: 56px;
  line-height: 1.6;
}

.optTag {
  font-size: 12px;
  color: var(--t2);
  margin-left: 4px;
  font-weight: 400;
  @include m.mono;
}

// ── Checkboxes ──
.cr {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 3px 0;
  cursor: pointer;
  user-select: none;

  label {
    font-size: 13px;
    color: var(--t1);
    cursor: pointer;
  }
}

.crDisabled {
  cursor: default;
  opacity: 0.45;

  label {
    cursor: default;
  }

  .cb {
    cursor: default;
  }
}

.cb {
  width: 14px;
  height: 14px;
  border: 1.5px solid var(--bdr2);
  border-radius: 4px;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  transition: all 0.12s;
}

.ck {
  .cb {
    background: var(--blue);
    border-color: var(--blue);

    &::after {
      content: '';
      width: 7px;
      height: 4px;
      border-left: 1.5px solid #0b0d10;
      border-bottom: 1.5px solid #0b0d10;
      transform: rotate(-45deg) translate(0, -1px);
      display: block;
    }
  }

  label {
    color: var(--t0);
  }
}

// Nested option, e.g. "Keyword Insights" under Keywords — indented, only
// rendered while its parent checkbox is on.
.crNested {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 3px 0 3px 20px;
  cursor: pointer;
  user-select: none;
  position: relative;

  &::before {
    content: '';
    position: absolute;
    left: 6px;
    top: 0;
    bottom: 50%;
    width: 8px;
    border-left: 1.5px solid var(--bdr2);
    border-bottom: 1.5px solid var(--bdr2);
    border-radius: 0 0 0 4px;
  }

  label {
    font-size: 12.5px;
    color: var(--t1);
    cursor: pointer;
  }

  &.ck label {
    color: var(--t0);
  }
}

// Nested field, e.g. the Timestamp Summary Interval dropdown — indented to
// visually sit under its parent checkbox.
.nestedField {
  padding-left: 20px;
  margin-bottom: 8px;

  label {
    display: block;
    font-size: 11.5px;
    color: var(--t2);
    margin-bottom: 4px;
  }

  select {
    max-width: 200px;
  }
}

// ── Range sliders ──
.rangeInput {
  -webkit-appearance: none;
  width: 100%;
  height: 3px;
  background: var(--bg4);
  border-radius: 99px;
  outline: none;
  cursor: pointer;
  display: block;
  margin-top: 4px;

  &::-webkit-slider-thumb {
    -webkit-appearance: none;
    width: 14px;
    height: 14px;
    background: var(--blue);
    border-radius: 50%;
    border: 2px solid var(--bg0);
    cursor: pointer;
    box-shadow: 0 0 0 2px rgba(91, 164, 239, 0.25);
  }
}

.slim {
  display: flex;
  justify-content: space-between;
  font-size: 12px;
  color: var(--t2);
  margin-top: 3px;
  @include m.mono;
}

.sv {
  @include m.mono;
  font-size: 13px;
  color: var(--blue);
}

.qTypeLabel {
  font-size: 13px;
  color: var(--t1);
  font-weight: 500;
  margin-bottom: 5px;
}

.dimLabel {
  color: var(--t2) !important;
}

.feasibilityTag {
  font-size: 10px;
  background: var(--amber-dim);
  color: var(--amber);
  padding: 1px 5px;
  border-radius: 99px;
  margin-left: 3px;
}

// ── Divider ──
.divider {
  border: none;
  height: 1px;
  margin: 12px 0;
  background: linear-gradient(90deg,
      rgba(139, 92, 246, 0.0) 0%,
      rgba(139, 92, 246, 0.35) 25%,
      rgba(56, 196, 186, 0.4) 50%,
      rgba(240, 160, 48, 0.35) 75%,
      rgba(240, 160, 48, 0.0) 100%);
}

// ── Callout ──
.callout {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  padding: 9px 12px;
  border-radius: var(--r);
  font-size: 13px;
  margin-bottom: 10px;
}

.cInfo {
  background: var(--blue-dim);
  border: 1px solid var(--blue-bdr);
  color: var(--blue);
}

// ── Generic utility buttons (used for save/cancel etc.) ──
.btn {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 6px 13px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t1);
  font-family: var(--font-ui);
  font-size: 13px;
  cursor: pointer;
  transition: all 0.12s;
  white-space: nowrap;
  user-select: none;

  &:hover {
    background: var(--bg3);
    color: var(--t0);
    border-color: var(--bdr3);
  }
}

.btnP {
  background: var(--blue);
  color: #ffffff;
  border-color: var(--blue);
  font-weight: 600;

  &:hover {
    background: #a78bfa;
    border-color: #a78bfa;
    color: #ffffff;
  }

  &:disabled {
    opacity: 0.4;
    cursor: not-allowed;
    pointer-events: none;
  }
}

// ── Batch running pill in header ─────────────────
.batchRunPill {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  font-size: 12px;
  color: var(--amber);
  background: var(--amber-dim);
  border: 1px solid var(--amber-bdr);
  padding: 1px 8px;
  border-radius: 99px;
  margin-left: 10px;
  font-family: var(--font-mono);
  vertical-align: middle;
}

.batchRunDot {
  width: 5px;
  height: 5px;
  border-radius: 50%;
  background: var(--amber);
  animation: breathe 1.2s ease-in-out infinite;
}

@keyframes breathe {

  0%,
  100% {
    opacity: 1;
  }

  50% {
    opacity: 0.35;
  }
}

// ── Model loading indicator ───────────────────────
.loadingDot {
  font-size: 13px;
  color: var(--t2);
  font-family: var(--font-mono);
}

// ══════════════════════════════════════
// 3-COLUMN BATCH STATUS
// ══════════════════════════════════════
.batchSection {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  margin-top: 16px;
  animation: fadeIn 0.2s ease;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(4px);
  }

  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.batchSectionTitle {
  font-size: 12px;
  font-weight: 600;
  color: var(--t2);
  text-transform: uppercase;
  letter-spacing: 0.1em;
  font-family: var(--font-mono);
  margin-bottom: 10px;
  display: flex;
  align-items: center;
  gap: 10px;
  flex-shrink: 0;
  flex-wrap: wrap;
}

.liveLabel {
  flex-shrink: 0;
}

// ── Live status pill ─────────────────────────────
.liveStatus {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  font-size: 12px;
  font-weight: 500;
  color: var(--green);
  background: var(--green-dim);
  border: 1px solid var(--green-bdr);
  padding: 2px 9px 2px 7px;
  border-radius: 99px;
  font-family: var(--font-mono);
  text-transform: none;
  letter-spacing: 0;
  white-space: nowrap;
}

.liveDot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: var(--green);
  flex-shrink: 0;
  animation: livePulse 1.4s ease-in-out infinite;
}

@keyframes livePulse {

  0%,
  100% {
    opacity: 1;
    transform: scale(1);
  }

  50% {
    opacity: 0.4;
    transform: scale(0.75);
  }
}

.liveSep {
  color: var(--green);
  opacity: 0.4;
  font-size: 12px;
}

.liveCountdown {
  color: var(--green);
  font-weight: 700;
  font-variant-numeric: tabular-nums;
  min-width: 2ch;
  display: inline-block;
  text-align: right;
}

.liveFiles {
  color: var(--green);
  opacity: 0.85;
}

.batchColumns {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 10px;
  flex: 1;
  overflow: hidden;
  min-height: 0;
}

.batchCol {
  display: flex;
  flex-direction: column;
  gap: 6px;
  min-width: 0;
  min-height: 0;
  overflow: hidden;
}

// Column headers
.batchColHead {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 12px;
  font-weight: 700;
  padding: 6px 10px;
  border-radius: 8px;
  font-family: var(--font-mono);
  text-transform: uppercase;
  letter-spacing: 0.06em;
  flex-shrink: 0;

  svg {
    width: 12px;
    height: 12px;
    flex-shrink: 0;
  }

  &.headQueued {
    background: rgba(240, 160, 48, 0.08);
    color: var(--amber);
    border: 1px solid var(--amber-bdr);

    svg {
      stroke: var(--amber);
    }
  }

  &.headRunning {
    background: var(--blue-dim);
    color: var(--blue);
    border: 1px solid var(--blue-bdr);

    svg {
      stroke: var(--blue);
    }
  }

  &.headCompleted {
    background: var(--green-dim);
    color: var(--green);
    border: 1px solid var(--green-bdr);

    svg {
      stroke: var(--green);
    }
  }
}

.batchColCount {
  margin-left: auto;
  font-size: 13px;
  font-weight: 700;
  min-width: 16px;
  text-align: center;
}

.batchColBody {
  display: flex;
  flex-direction: column;
  gap: 5px;
  flex: 1;
  overflow-y: auto;
  padding-bottom: 14px;
  @include m.scrollbar;
}

.batchEmpty {
  text-align: center;
  font-size: 13px;
  color: var(--t2);
  font-family: var(--font-mono);
  padding: 12px 0;
}

// ── Status card wrapper (handles enter/exit animation) ──
.statusCardWrap {
  position: relative;
  border-radius: 10px;
}

// ── Queued: slow breathing amber dashed-style border ──
.queuedWrap {
  padding: 2px;
  background: conic-gradient(from 0deg,
      rgba(240, 160, 48, 0.7) 0deg,
      rgba(240, 160, 48, 0.15) 60deg,
      rgba(240, 160, 48, 0.05) 120deg,
      rgba(240, 160, 48, 0.15) 180deg,
      rgba(240, 160, 48, 0.05) 240deg,
      rgba(240, 160, 48, 0.15) 300deg,
      rgba(240, 160, 48, 0.7) 360deg);
  animation: queuedBreath 2.8s ease-in-out infinite;
}

@keyframes queuedBreath {

  0%,
  100% {
    opacity: 0.5;
  }

  50% {
    opacity: 1;
  }
}

// ── Completed: static clean green gradient border, no animation ──
.completedWrap {
  padding: 2px;
  background: linear-gradient(135deg,
      rgba(74, 222, 128, 0.9) 0%,
      rgba(74, 222, 128, 0.3) 40%,
      rgba(74, 222, 128, 0.6) 70%,
      rgba(74, 222, 128, 0.9) 100%);
}

// ── Animated gradient border for running cards ──
.runningWrap {
  border-radius: 10px;
  padding: 2px;
  background: conic-gradient(from var(--border-angle, 0deg),
      transparent 0deg,
      transparent 30deg,
      #5ba4ef 50deg,
      #a78bfa 65deg,
      #f0a030 80deg,
      transparent 100deg,
      transparent 360deg);
  animation: borderBeamSpin 2.4s linear infinite;
}

@property --border-angle {
  syntax: '<angle>';
  initial-value: 0deg;
  inherits: false;
}

@keyframes borderBeamSpin {
  to {
    --border-angle: 360deg;
  }
}

// Status cards
.statusCard {
  display: flex;
  align-items: flex-start;
  gap: 7px;
  padding: 8px 10px;
  border-radius: 8px;
  border: 1px solid;
  min-width: 0;
  position: relative;

  &.queued {
    background: var(--bg1);
    border-color: transparent;
    border-radius: 8px;
  }

  &.running {
    background: var(--bg1);
    border-color: transparent;
    border-radius: 8px;
  }

  &.completed {
    background: var(--bg1);
    border-color: transparent;
    border-radius: 8px;
  }
}

.statusExt {
  width: 26px;
  height: 26px;
  border-radius: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 10px;
  font-weight: 800;
  font-family: var(--font-mono);
  flex-shrink: 0;

  &.vtt {
    background: var(--blue-dim);
    color: var(--blue);
  }

  &.srt {
    background: var(--green-dim);
    color: var(--green);
  }
}

// ── Completed check icon ──
.completedCheck {
  width: 26px;
  height: 26px;
  border-radius: 50%;
  background: var(--green-dim);
  border: 1.5px solid var(--green-bdr);
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  color: var(--green);

  svg {
    width: 13px;
    height: 13px;
  }
}

// ── Queued "waiting" meta line ──
.queuedMeta {
  display: flex;
  align-items: center;
  gap: 5px;
  font-size: 10px;
  color: var(--amber);
  font-family: var(--font-mono);
  opacity: 0.8;
}

.queuedDot {
  width: 5px;
  height: 5px;
  border-radius: 50%;
  background: var(--amber);
  flex-shrink: 0;
  animation: queuedDotPulse 1.6s ease-in-out infinite;
}

@keyframes queuedDotPulse {

  0%,
  100% {
    opacity: 0.3;
    transform: scale(0.8);
  }

  50% {
    opacity: 1;
    transform: scale(1.1);
  }
}

// ── Completed date meta ──
.completedMeta {
  font-size: 10px;
  color: var(--green);
  font-family: var(--font-mono);
  opacity: 0.75;
}

.statusInfo {
  flex: 1;
  min-width: 0;
}

.statusNameRow {
  display: flex;
  align-items: center;
  gap: 5px;
  min-width: 0;
  margin-bottom: 3px;
  overflow: hidden;
}

.statusName {
  font-size: 13px;
  font-weight: 500;
  color: var(--t0);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  flex: 1;
  min-width: 0;
}

.statusIdBadge {
  font-size: 10px;
  font-weight: 700;
  font-family: var(--font-mono);
  border-radius: 4px;
  padding: 1px 5px;
  flex-shrink: 0;
  white-space: nowrap;
  letter-spacing: 0.02em;
  color: var(--blue);
  background: var(--blue-dim);
  border: 1px solid var(--blue-bdr);
}

.statusIdBadge_queued {
  color: var(--amber);
  background: rgba(240, 160, 48, 0.12);
  border-color: rgba(240, 160, 48, 0.3);
}

.statusIdBadge_running {
  color: var(--blue);
  background: var(--blue-dim);
  border-color: var(--blue-bdr);
}

.statusIdBadge_completed {
  color: var(--green);
  background: var(--green-dim);
  border-color: var(--green-bdr);
}

.statusDate {
  font-size: 10px;
  color: var(--t2);
  font-family: var(--font-mono);
}

.statusProgress {
  display: flex;
  align-items: center;
  gap: 5px;
}

.statusBar {
  flex: 1;
  height: 3px;
  background: var(--bg4);
  border-radius: 99px;
  overflow: hidden;
}

.statusFill {
  height: 100%;
  background: var(--blue);
  border-radius: 99px;
  transition: width 0.4s ease;
}

.statusPct {
  font-size: 10px;
  color: var(--blue);
  font-family: var(--font-mono);
  flex-shrink: 0;
}

// ── Model row (label + refresh) ──────────────────
.modelLabelRow {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 4px;

  .fl {
    margin-bottom: 0;
  }
}

.refreshBtn {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 2px 8px;
  border-radius: 99px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  font-family: var(--font-ui);
  font-size: 12px;
  cursor: pointer;
  transition: all 0.12s;
  flex-shrink: 0;

  svg {
    width: 10px;
    height: 10px;
    flex-shrink: 0;
    transition: transform 0.12s;
  }

  &:hover:not(:disabled) {
    background: var(--bg3);
    color: var(--t1);
    border-color: var(--bdr3);
  }

  &:disabled {
    opacity: 0.45;
    cursor: not-allowed;
  }
}

.spinning {
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  from {
    transform: rotate(0deg);
  }

  to {
    transform: rotate(360deg);
  }
}

// ── Model status messages ─────────────────────────
.modelWarn {
  display: flex;
  align-items: flex-start;
  gap: 6px;
  margin-top: 7px;
  padding: 7px 10px;
  border-radius: var(--r);
  background: var(--amber-dim);
  border: 1px solid var(--amber-bdr);
  color: var(--amber);
  font-size: 13px;
  line-height: 1.5;

  svg {
    width: 12px;
    height: 12px;
    flex-shrink: 0;
    margin-top: 1px;
    stroke: var(--amber);
  }
}

.modelOk {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-top: 6px;
  font-size: 12px;
  color: var(--t2);
  font-family: var(--font-mono);

  svg {
    width: 11px;
    height: 11px;
    flex-shrink: 0;
    stroke: var(--green);
  }
}

// ══════════════════════════════════════
// SUBMITTING OVERLAY
// ══════════════════════════════════════
.submittingOverlay {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  background: rgba(var(--bg0-rgb, 11, 13, 16), 0.82);
  backdrop-filter: blur(4px);
  z-index: 20;
  animation: fadeIn 0.18s ease;
}

.submittingCard {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
  padding: 28px 32px;
  background: var(--bg1);
  border: 1px solid var(--bdr2);
  border-radius: var(--rxl, 12px);
  max-width: 320px;
  width: 90%;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.28);
  text-align: center;
}

.submittingSpinner {
  width: 36px;
  height: 36px;
  border: 3px solid var(--bdr2);
  border-top-color: var(--blue);
  border-right-color: rgba(139, 92, 246, 0.5);
  border-radius: 50%;
  animation: spin 0.75s linear infinite;
  flex-shrink: 0;
}

.submittingTitle {
  font-size: 14px;
  font-weight: 600;
  color: var(--t0);
  letter-spacing: -0.2px;
}

.submittingDesc {
  font-size: 13px;
  color: var(--t2);
  font-family: var(--font-mono);
  line-height: 1.65;
}

.submittingFiles {
  display: flex;
  flex-direction: column;
  gap: 5px;
  width: 100%;
  margin-top: 2px;
}

.submittingFile {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 13px;
  color: var(--t1);
  font-family: var(--font-mono);
  background: var(--bg2);
  border: 1px solid var(--bdr);
  border-radius: var(--r);
  padding: 5px 10px;
  text-align: left;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.submittingDot {
  width: 5px;
  height: 5px;
  border-radius: 50%;
  background: var(--blue);
  flex-shrink: 0;
  animation: breathe 1.2s ease-in-out infinite;
}

.submittingMore {
  font-size: 12px;
  color: var(--t2);
  font-family: var(--font-mono);
  text-align: center;
  padding: 2px 0;
}

// ── Stop button on queued file cards ─────────────────────
.stopBtn {
  width: 24px;
  height: 24px;
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 6px;
  border: 1px solid rgba(239, 68, 68, 0.35);
  background: rgba(239, 68, 68, 0.08);
  color: #ef4444;
  cursor: pointer;
  padding: 0;
  transition: all 0.15s;
  margin-left: auto;

  svg {
    width: 9px;
    height: 9px;
  }

  &:hover:not(:disabled) {
    background: rgba(239, 68, 68, 0.18);
    border-color: rgba(239, 68, 68, 0.6);
    box-shadow: 0 0 6px rgba(239, 68, 68, 0.2);
  }

  &:disabled {
    opacity: 0.5;
    cursor: default;
  }
}

.stopSpinner {
  width: 10px;
  height: 10px;
  border: 1.5px solid rgba(239, 68, 68, 0.3);
  border-top-color: #ef4444;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
  flex-shrink: 0;
}

// ── Large screen overrides (> 1900px) ────────────────────
@media (min-width: 1920px) {
  .slbl {
    font-size: 13px;
  }

  .selSummary {
    font-size: 14px;
  }

  .selCt {
    font-size: 14px;
  }

  .selNm {
    font-size: 13px;
  }

  .cardT {
    font-size: 11px;
  }

  .fl {
    font-size: 14px;
  }

  .fc {
    font-size: 14px;
  }

  .optTag {
    font-size: 13px;
  }

  .cr label {
    font-size: 14px;
  }

  .slim {
    font-size: 13px;
  }

  .sv {
    font-size: 14px;
  }

  .qTypeLabel {
    font-size: 14px;
  }

  .noPromptHint {
    font-size: 14px;
  }

  .btn {
    font-size: 14px;
  }

  .batchSectionTitle {
    font-size: 13px;
  }

  .batchColHead {
    font-size: 13px;
  }

  .batchColCount {
    font-size: 14px;
  }

  .batchEmpty {
    font-size: 14px;
  }

  .statusName {
    font-size: 14px;
  }

  .statusIdBadge {
    font-size: 11px;
  }

  .statusDate {
    font-size: 11px;
  }

  .statusPct {
    font-size: 11px;
  }

  .queuedMeta {
    font-size: 11px;
  }

  .completedMeta {
    font-size: 11px;
  }

  .loadingDot {
    font-size: 14px;
  }

  .refreshBtn {
    font-size: 13px;
  }

  .modelWarn {
    font-size: 14px;
  }

  .modelOk {
    font-size: 13px;
  }

  .submittingTitle {
    font-size: 15px;
  }

  .submittingDesc {
    font-size: 14px;
  }

  .submittingFile {
    font-size: 14px;
  }

  .submittingMore {
    font-size: 13px;
  }

  .liveStatus {
    font-size: 13px;
  }
}

// ── Per-file prompt preview ───────────────────
.perFileBtn {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  margin-left: 8px;
  padding: 2px 8px 2px 6px;
  border-radius: 5px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  font-size: 11px;
  font-family: var(--font-ui);
  cursor: pointer;
  vertical-align: middle;
  transition: all 0.14s;

  svg:first-child {
    width: 12px;
    height: 12px;
    flex-shrink: 0;
  }
}

.perFileBtn:hover {
  background: var(--bg3);
  border-color: var(--bdr3);
  color: var(--t0);
}

.perFileBtnActive {
  background: var(--blue-dim);
  border-color: var(--blue-bdr);
  color: var(--blue);
}

.perFileBtnActive:hover {
  background: var(--blue-dim);
  border-color: var(--blue);
  color: var(--blue);
}

.perFileChevron {
  width: 10px;
  height: 10px;
  flex-shrink: 0;
  transition: transform 0.18s ease;
}

.perFileChevronOpen {
  transform: rotate(180deg);
}

// Animated expand container
.perFilePanel {
  display: grid;
  grid-template-rows: 0fr;
  opacity: 0;
  transition:
    grid-template-rows 0.22s cubic-bezier(0.4, 0, 0.2, 1),
    opacity 0.18s ease,
    margin 0.2s ease;
  margin-bottom: 0;
}

.perFilePanelOpen {
  grid-template-rows: 1fr;
  opacity: 1;
  margin-bottom: 6px;
}

.perFilePanelInner {
  min-height: 0;
  overflow: hidden;
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  background: var(--bg1);
  max-height: 200px;
  overflow-y: auto;
}

.perFileRow {
  padding: 8px 10px;
  border-bottom: 1px solid var(--bdr1);
  display: flex;
  flex-direction: column;
  gap: 3px;

  &:last-child {
    border-bottom: none;
  }
}

.perFileName {
  font-size: 11px;
  font-family: var(--font-mono);
  color: var(--t2);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.perFilePrompt {
  font-size: 12px;
  color: var(--t0);
  line-height: 1.5;
  white-space: pre-wrap;
  word-break: break-word;
}

.perFileEmpty {
  font-size: 12px;
  color: var(--t2);
  font-style: italic;
  opacity: 0.6;
}

@media (min-width: 1920px) {
  .perFileBtn {
    font-size: 12px;
  }

  .perFileName {
    font-size: 12px;
  }

  .perFilePrompt {
    font-size: 13px;
  }

  .perFileEmpty {
    font-size: 13px;
  }
}

// ── Prompt save / cancel bar ──────────────────
.promptActions {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 8px;
  padding: 10px 12px 12px;
  border-top: 1px solid var(--bdr1);
  margin-top: 4px;
}

.promptSaveError {
  flex: 1;
  font-size: 12px;
  color: var(--red, #ef4444);
  margin-right: 4px;
}

.btnSm {
  height: 30px;
  padding: 0 14px;
  font-size: 12px;
  border-radius: 6px;
  border: 1px solid transparent;
  font-family: var(--font-ui);
  font-weight: 500;
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  gap: 6px;
  transition: all 0.14s;

  &:disabled {
    opacity: 0.4;
    cursor: not-allowed;
  }
}

.btnGhost {
  background: transparent;
  border-color: var(--bdr2);
  color: var(--t1);

  &:hover:not(:disabled) {
    background: var(--bg3);
    border-color: var(--bdr3);
  }
}

.btnPrimary {
  background: var(--blue);
  border-color: var(--blue);
  color: #fff;

  &:hover:not(:disabled) {
    opacity: 0.88;
  }
}

.btnSpinner {
  width: 11px;
  height: 11px;
  border: 1.5px solid rgba(255, 255, 255, 0.35);
  border-top-color: #fff;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
  flex-shrink: 0;
}

@media (min-width: 1920px) {
  .btnSm {
    height: 34px;
    font-size: 13px;
    padding: 0 16px;
  }

  .promptSaveError {
    font-size: 13px;
  }
}

// ─────────────────────────────────────────────
// Minimize / minimized rail
// ─────────────────────────────────────────────

.minimizeStepBtn {
  width: 28px;
  height: 28px;
  border-radius: 7px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  flex-shrink: 0;
  transition: all 0.14s;

  svg {
    width: 13px;
    height: 13px;
  }

  &:hover {
    background: var(--bg3);
    border-color: var(--bdr3);
    color: var(--t0);
  }
}

.infpanelMinimized {
  flex: 1;
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.minRail {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: flex-start;
  gap: 14px;
  padding: 14px 0;
  width: 100%;
  background: var(--bg2);
  border: 1px solid var(--bdr);
  border-radius: var(--rl);
  color: var(--t1);
  cursor: pointer;
  transition: background 0.14s, border-color 0.14s, color 0.14s;

  &:hover {
    background: var(--bg3);
    border-color: var(--bdr2);
    color: var(--t0);
  }
}

.minRailIcon {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  border-radius: 7px;
  border: 1px solid var(--bdr2);
  color: var(--t2);
  flex-shrink: 0;

  svg {
    width: 14px;
    height: 14px;
  }

  .minRail:hover & {
    color: var(--t0);
    border-color: var(--bdr3);
  }
}

.minRailLabel {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  writing-mode: vertical-rl;
  transform: rotate(180deg);
  font-size: 12px;
  font-weight: 500;
  letter-spacing: 0.04em;
  color: var(--t1);
  white-space: nowrap;
  user-select: none;
}

.minRailDot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: var(--amber);
  animation: minRailPulse 1.2s infinite;
}

@keyframes minRailPulse {

  0%,
  100% {
    opacity: 1;
  }

  50% {
    opacity: 0.4;
  }
}
