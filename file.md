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
  }

  &.success {
    border-color: var(--green-bdr);
    background: var(--green-dim);

    &::after {
      background: var(--green);
    }
  }

  &.failed {
    border-color: var(--red-bdr);
    background: var(--red-dim);

    &::after {
      background: var(--red);
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
// pages/UploadInfer/InferencePanel.tsx
// Content Analytics · Inference configuration + batch status
// ═══════════════════════════════════════════════
import React, { useEffect, useRef, useCallback, useState } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import {
  inferenceStatusSuccess, updateRunningProgress,
  updateSummaryPrompt, updateKeywordPrompt, updateQuestionPrompt,
  updateShortAnswerPrompt, updateTrueFalsePrompt, updateSettings,
  modelsLoading, modelsSuccess, modelsFailure, setSelectedModel,
  updateFilePrompts,
  type ServerFile, type ServerFilesData, type TimeInterval,
} from '../../store/uploadSlice';
import api from '../../services/api';
import styles from './InferencePanel.module.scss';
import { addToast } from '../../store/toastSlice';

// ── Batch status columns ──────────────────────────
const StatusCard: React.FC<{ file: ServerFile; variant: 'queued' | 'running' | 'completed'; onStop?: (id: number) => void; stopping?: boolean }> = ({ file, variant, onStop, stopping }) => {
  const { t } = useTranslation();
  const ext = file.original_name.toLowerCase().endsWith('.srt') ? 'srt' : 'vtt';
  return (
    <div className={`${styles.statusCardWrap} ${styles[variant + 'Wrap']}`}>
      <div className={`${styles.statusCard} ${styles[variant]}`}>

        {/* Left icon — ext badge for queued/running, green check for completed */}
        {variant === 'completed' ? (
          <div className={styles.completedCheck}>
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
              <path d="M3 8l3.5 3.5L13 5" />
            </svg>
          </div>
        ) : (
          <div className={`${styles.statusExt} ${styles[ext]}`}>{ext.toUpperCase()}</div>
        )}

        <div className={styles.statusInfo}>
          <div className={styles.statusNameRow}>
            <span className={`${styles.statusIdBadge} ${styles['statusIdBadge_' + variant]}`}>#{file.id}</span>
            <div className={styles.statusName}>{file.original_name}</div>
          </div>

          {variant === 'running' && typeof file.progress === 'number' && (
            <div className={styles.statusProgress}>
              <div className={styles.statusBar}>
                <div className={styles.statusFill} style={{ width: `${file.progress}%` }} />
              </div>
              <span className={styles.statusPct}>{file.progress}%</span>
            </div>
          )}

          {variant === 'queued' && (
            <div className={styles.queuedMeta}>
              <span className={styles.queuedDot} />
              {t('uploadInfer.inferencePanel.waitingQueue')}
            </div>
          )}

          {variant === 'completed' && (
            <div className={styles.completedMeta}>{file.inserted_at}</div>
          )}

          {variant === 'running' && typeof file.progress !== 'number' && (
            <div className={styles.statusDate}>{file.inserted_at}</div>
          )}
        </div>

        {/* Stop button — queued files only */}
        {variant === 'queued' && onStop && (
          <button
            className={styles.stopBtn}
            onClick={() => onStop(file.id)}
            disabled={stopping}
            title={t('uploadInfer.inferencePanel.stopFile')}
          >
            {stopping ? (
              <span className={styles.stopSpinner} />
            ) : (
              <svg viewBox="0 0 16 16" fill="currentColor" stroke="none">
                <rect x="4" y="4" width="8" height="8" rx="1.5" />
              </svg>
            )}
          </button>
        )}

      </div>
    </div>
  );
};

// ── InferencePanel ───────────────────────────────
interface InferencePanelProps {
  onClose?: () => void;
  minimized?: boolean;
  onToggleMinimize?: () => void;
}

const InferencePanel: React.FC<InferencePanelProps> = ({ onClose, minimized = false, onToggleMinimize }) => {
  const { t } = useTranslation();
  const dispatch = useAppDispatch();
  const {
    settings,
    selectedServerIds, isBatchRunning,
    selectFiles,
    models, modelsLoading: mlLoading, selectedModel,
    dateFrom, dateTo,
  } = useAppSelector(s => s.upload);

  const [running, setRunning] = React.useState(false);
  const [stoppingIds, setStoppingIds] = React.useState<Set<number>>(new Set());

  // ── Prompt save state ──────────────────────────
  const [promptSaving, setPromptSaving] = useState(false);
  const [promptSaveError, setPromptSaveError] = useState<string | null>(null);
  const promptSnapshot = useRef({ summary: '', keyword: '', question: '', shortAnswer: '', trueFalse: '' });
  const promptsDirty =
    settings.summaryPromptOverride !== promptSnapshot.current.summary ||
    settings.keywordPromptOverride !== promptSnapshot.current.keyword ||
    settings.questionPromptOverride !== promptSnapshot.current.question ||
    settings.shortAnswerPromptOverride !== promptSnapshot.current.shortAnswer ||
    settings.trueFalsePromptOverride !== promptSnapshot.current.trueFalse;

  const emptyBatch: ServerFilesData = { queued: [], running: [], completed: [], pending: [] };
  const [batchData, setBatchData] = React.useState<ServerFilesData>(emptyBatch);
  const [countdown, setCountdown] = React.useState(10);
  const countdownRef = useRef<ReturnType<typeof setInterval> | null>(null);
  const n = selectedServerIds.length;
  // When nothing is selected, every generation checkbox should read as
  // unchecked and be non-interactive — there's nothing to configure yet.
  // Settings themselves aren't reset (so a prior selection is restored if
  // the user re-selects the same files), only the display/interaction is.
  const noFilesSelected = n === 0;
  const isChecked = (v: boolean) => v && !noFilesSelected;
  const canRunInference = n > 0 && !isBatchRunning && !!selectedModel;
  const pollingRef = useRef<ReturnType<typeof setInterval> | null>(null);

  // ── Fetch models ────────────────────────────
  const fetchModels = useCallback(async () => {
    dispatch(modelsLoading());
    try {
      const res = await api.get('/get_models');
      const result = (res.data as any)?.result ?? [];
      dispatch(modelsSuccess(result));
    } catch {
      dispatch(modelsFailure());
    }
  }, [dispatch]); // eslint-disable-line

  useEffect(() => { fetchModels(); }, []); // eslint-disable-line

  // ── Polling helper ───────────────────────────
  const fetchFilesForPolling = useCallback(async () => {
    try {
      const statusRes = await api.post('/files/by-progress/', { start_date: dateFrom, end_date: dateTo });
      const d = (statusRes.data as any)?.data;
      const data: ServerFilesData = {
        queued: d?.queued ?? [],
        completed: d?.completed ?? [],
        pending: d?.pending ?? [],
        running: d?.running ?? [],
      };
      dispatch(inferenceStatusSuccess(data));
      setBatchData(data);

      const stillRunning = data.running.length > 0 || data.queued.length > 0;

      if (data.running.length > 0) {
        const runningIds = data.running.map(f => f.id);
        try {
          const progressRes = await api.post('/files/progress/', { file_ids: runningIds });
          const progressMap = (progressRes.data as any)?.result ?? {};
          dispatch(updateRunningProgress(progressMap));
          setBatchData(prev => ({
            ...prev,
            running: prev.running.map(f => {
              const raw = progressMap[String(f.id)];
              if (raw === undefined) return f;
              const pct = typeof raw === 'string' ? parseFloat(raw) : raw;
              return { ...f, progress: isNaN(pct) ? f.progress : Math.min(100, Math.max(0, pct)) };
            }),
          }));
        } catch {
          // progress fetch failing is non-critical — silently ignore
        }
      }

      if (!stillRunning) setBatchData(emptyBatch);
      return stillRunning;
    } catch {
      return false;
    }
  }, [dispatch, dateFrom, dateTo]);

  const stopCountdown = useCallback(() => {
    if (countdownRef.current) {
      clearInterval(countdownRef.current);
      countdownRef.current = null;
    }
  }, []);

  const startCountdown = useCallback(() => {
    stopCountdown();
    setCountdown(10);
    countdownRef.current = setInterval(() => {
      setCountdown(prev => (prev <= 1 ? 10 : prev - 1));
    }, 1000);
  }, [stopCountdown]);

  const stopPolling = useCallback(() => {
    if (pollingRef.current) {
      clearInterval(pollingRef.current);
      pollingRef.current = null;
    }
    stopCountdown();
    setCountdown(10);
  }, [stopCountdown]);

  const startPolling = useCallback(() => {
    stopPolling();
    startCountdown();
    pollingRef.current = setInterval(async () => {
      const stillRunning = await fetchFilesForPolling();
      if (!stillRunning) stopPolling();
      else startCountdown();
    }, 10000);
  }, [fetchFilesForPolling, stopPolling, startCountdown]);

  useEffect(() => {
    if (isBatchRunning) {
      fetchFilesForPolling().then(stillRunning => {
        if (stillRunning) startPolling();
        else stopPolling();
      });
    } else {
      stopPolling();
    }
    return stopPolling;
  }, [isBatchRunning]); // eslint-disable-line

  // ── One-time bootstrap check on mount ──
  // /files/by-date/ is now paginated and no longer tells us whether a batch
  // is running (a running/queued file may simply be on another page), so we
  // can't rely on its response to seed isBatchRunning like before. Ask
  // /files/by-progress/ directly once on mount to catch a batch that was
  // already running before this page loaded (e.g. after a refresh).
  useEffect(() => {
    fetchFilesForPolling().then(stillRunning => { if (stillRunning) startPolling(); });
  }, []); // eslint-disable-line

  // ── Stop a queued file ───────────────────────
  const handleStop = useCallback(async (fileId: number) => {
    setStoppingIds(prev => new Set(prev).add(fileId));
    try {
      await api.patch('/files/stop', { fileID: [fileId] });
    } catch (err: any) {
      if (err?.response?.status === 403) {
        dispatch(addToast(t('uploadInfer.inferencePanel.stopAlready'), 'error'));
      } else {
        console.error('Stop file failed:', err);
      }
    } finally {
      await fetchFilesForPolling();
      setStoppingIds(prev => { const s = new Set(prev); s.delete(fileId); return s; });
    }
  }, [fetchFilesForPolling]);

  // ── Run inference ────────────────────────────
  const handleRun = async () => {
    if (!canRunInference) return;
    setRunning(true);
    try {
      await api.post('/batch_process', {
        all: false, // always false for now — file_ids-driven runs only
        file_ids: selectedServerIds,
        start_date: dateFrom,
        end_date: dateTo,
        model_name: selectedModel,
        summary_prompt: settings.summaryPromptOverride,
        faq_prompt: settings.questionPromptOverride,
        keywords_prompt: settings.keywordPromptOverride,
        short_answers_prompt: settings.shortAnswerPromptOverride,
        true_false_prompt: settings.trueFalsePromptOverride,
        generate_summary: settings.generateSummary,
        generate_keywords: settings.generateKeywords,
        generate_faq: settings.generateQuestions,
        generate_short_answer: settings.generateShortAnswer,
        generate_true_false: settings.generateTrueFalse,
        // Only meaningful — and only ever sent true — when Keywords itself is on.
        generate_keyword_insights: settings.generateKeywords && settings.generateKeywordInsights,
        timestamped_summary: settings.timestampedSummary,
        time_interval: settings.timeInterval,
      });
      const stillRunning = await fetchFilesForPolling();
      if (stillRunning) startPolling();
      dispatch(addToast(t('uploadInfer.inferencePanel.inferenceStarted'), 'success'));
    } catch (err) {
      console.error('Batch process failed:', err);
    } finally {
      setRunning(false);
    }
  };

  // ── Run button sub-label ─────────────────────
  const runBtnSubLabel = (() => {
    if (n === 0) return t('uploadInfer.inferencePanel.selectFilesFirst');
    if (!selectedModel) return t('uploadInfer.inferencePanel.noModelSelected');
    return `${n} file${n !== 1 ? 's' : ''} ready`;
  })();

  // ── Auto-fill prompts when exactly 1 file is selected ────────────
  // Keyed off the *actual selected file id* (or a stable "multi"/"none"
  // marker), not the selection count and not the selectedServerIds array
  // reference. Redux can hand us a brand-new array reference for the same
  // underlying selection on unrelated updates — keying off the array (or
  // count alone) made this effect refire and stomp on prompt edits the
  // user was still mid-typing, even though the selection hadn't changed.
  const singleSelectedId = n === 1 ? selectedServerIds[0] : null;
  const selectionKey = n === 1 ? `one:${singleSelectedId}` : n === 0 ? 'none' : 'multi';
  const prevSelectionKey = useRef<string | null>(null);
  useEffect(() => {
    if (selectionKey === prevSelectionKey.current) return;
    prevSelectionKey.current = selectionKey;

    if (n === 1) {
      const file = selectFiles.find(f => f.id === singleSelectedId);
      if (file) {
        const s = file.summary_prompt ?? '';
        const k = file.keywords_prompt ?? '';
        const q = file.faq_prompt ?? '';
        const sa = file.short_answer_prompt ?? '';
        const tf = file.true_false_prompt ?? '';
        dispatch(updateSummaryPrompt(s));
        dispatch(updateKeywordPrompt(k));
        dispatch(updateQuestionPrompt(q));
        dispatch(updateShortAnswerPrompt(sa));
        dispatch(updateTrueFalsePrompt(tf));
        promptSnapshot.current = { summary: s, keyword: k, question: q, shortAnswer: sa, trueFalse: tf };
      }
    } else {
      dispatch(updateSummaryPrompt(''));
      dispatch(updateKeywordPrompt(''));
      dispatch(updateQuestionPrompt(''));
      dispatch(updateShortAnswerPrompt(''));
      dispatch(updateTrueFalsePrompt(''));
      promptSnapshot.current = { summary: '', keyword: '', question: '', shortAnswer: '', trueFalse: '' };
    }
    setPromptSaveError(null);
  }, [selectionKey]); // eslint-disable-line

  // ── Per-file prompt preview expand/collapse ──
  const [summaryExpanded, setSummaryExpanded] = useState(false);
  const [keywordExpanded, setKeywordExpanded] = useState(false);
  const [questionExpanded, setQuestionExpanded] = useState(false);
  const [shortAnswerExpanded, setShortAnswerExpanded] = useState(false);
  const [trueFalseExpanded, setTrueFalseExpanded] = useState(false);

  const prevSelectedCount = useRef(selectedServerIds.length);
  useEffect(() => {
    if (selectedServerIds.length !== prevSelectedCount.current) {
      prevSelectedCount.current = selectedServerIds.length;
      setSummaryExpanded(false);
      setKeywordExpanded(false);
      setQuestionExpanded(false);
      setShortAnswerExpanded(false);
      setTrueFalseExpanded(false);
    }
  });

  const selectedFiles = selectFiles.filter(f => selectedServerIds.includes(f.id));

  const handlePromptCancel = () => {
    dispatch(updateSummaryPrompt(promptSnapshot.current.summary));
    dispatch(updateKeywordPrompt(promptSnapshot.current.keyword));
    dispatch(updateQuestionPrompt(promptSnapshot.current.question));
    dispatch(updateShortAnswerPrompt(promptSnapshot.current.shortAnswer));
    dispatch(updateTrueFalsePrompt(promptSnapshot.current.trueFalse));
    setPromptSaveError(null);
  };

  const handlePromptSave = async () => {
    setPromptSaving(true);
    setPromptSaveError(null);
    try {
      const payload: Record<string, unknown> = {
        file_ids: selectedServerIds,
      };
      if (settings.generateSummary) payload.summary_prompt = settings.summaryPromptOverride;
      if (settings.generateKeywords) payload.keywords_prompt = settings.keywordPromptOverride;
      if (settings.generateQuestions) payload.faq_prompt = settings.questionPromptOverride;
      if (settings.generateShortAnswer) payload.short_answers_prompt = settings.shortAnswerPromptOverride;
      if (settings.generateTrueFalse) payload.true_false_prompt = settings.trueFalsePromptOverride;

      await api.post('/prompt_update', payload);

      dispatch(updateFilePrompts({
        fileIds: selectedServerIds,
        ...(settings.generateSummary && { summaryPrompt: settings.summaryPromptOverride }),
        ...(settings.generateKeywords && { keywordsPrompt: settings.keywordPromptOverride }),
        ...(settings.generateQuestions && { faqPrompt: settings.questionPromptOverride }),
        ...(settings.generateShortAnswer && { shortAnswerPrompt: settings.shortAnswerPromptOverride }),
        ...(settings.generateTrueFalse && { trueFalsePrompt: settings.trueFalsePromptOverride }),
      }));

      promptSnapshot.current = {
        summary: settings.summaryPromptOverride,
        keyword: settings.keywordPromptOverride,
        question: settings.questionPromptOverride,
        shortAnswer: settings.shortAnswerPromptOverride,
        trueFalse: settings.trueFalsePromptOverride,
      };
    } catch {
      setPromptSaveError(t('uploadInfer.inferencePanel.promptSaveFail'));
    } finally {
      setPromptSaving(false);
    }
  };

  return (
    <div className={`${styles.infpanel} ${minimized ? styles.infpanelMinimized : ''}`}>

      {/* ── Minimized rail ── */}
      {minimized ? (
        <button
          type="button"
          className={styles.minRail}
          onClick={onToggleMinimize}
          title={t('uploadInfer.inferencePanel.expandStep2')}
          aria-label={t('uploadInfer.inferencePanel.expandStep2')}
        >
          <span className={styles.minRailIcon}>
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
              strokeWidth="1.6" strokeLinecap="round" strokeLinejoin="round">
              <path d="M6 4l4 4-4 4" />
            </svg>
          </span>
          <span className={styles.minRailLabel}>
            {t('uploadInfer.inferencePanel.step2Label').replace('Configuration', '').replace('2 —', '2 —')}
            {isBatchRunning && <span className={styles.minRailDot} />}
          </span>
        </button>
      ) : (
        <>

          {/* ── Header ── */}
          <div className={styles.infpanelHead}>
            <div>
              <div className={styles.slbl}>
                {isBatchRunning ? (
                  <span className={styles.inferenceRunningLabel}>{t('uploadInfer.inferencePanel.step2Running')}</span>
                ) : t('uploadInfer.inferencePanel.step2Label')}
              </div>
              <div className={styles.selSummary}>
                {t('uploadInfer.inferencePanel.filesSelected', { count: n })}
                {isBatchRunning && <span className={styles.batchRunPill}><span className={styles.batchRunDot} />{t('uploadInfer.inferencePanel.batchRunPill')}</span>}
              </div>
            </div>
            <div className={styles.headActions}>
              {!isBatchRunning && (
                <>
                  {/* ── Run Inference button — large, self-describing ── */}
                  <button
                    className={`${styles.runBtn} ${canRunInference ? styles.runBtnReady : styles.runBtnDisabled}`}
                    onClick={handleRun}
                    disabled={!canRunInference}
                    data-tour="infer-run"
                    aria-label={`Run inference on ${n} file${n !== 1 ? 's' : ''}`}
                  >
                    <span className={styles.runBtnIconWrap}>
                      <svg width="15" height="15" viewBox="0 0 16 16" aria-hidden="true">
                        <path d="M5 3l8 5-8 5V3z" fill="currentColor" />
                      </svg>
                    </span>
                    <span className={styles.runBtnText}>
                      <span className={styles.runBtnTitle}>Run inference</span>
                      <span className={styles.runBtnSub}>{runBtnSubLabel}</span>
                    </span>
                  </button>

                  {onToggleMinimize && (
                    <button
                      className={styles.minimizeStepBtn}
                      onClick={onToggleMinimize}
                      title={t('uploadInfer.inferencePanel.minimizeStep2')}
                      aria-label={t('uploadInfer.inferencePanel.minimizeStep2')}
                    >
                      <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                        strokeWidth="1.6" strokeLinecap="round">
                        <path d="M3 8h10" />
                      </svg>
                    </button>
                  )}
                  {onClose && (
                    <button
                      className={styles.closeStepBtn}
                      onClick={onClose}
                      title={t('uploadInfer.inferencePanel.closeStep2')}
                    >
                      <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                        <path d="M4 4l8 8M12 4l-8 8" />
                      </svg>
                    </button>
                  )}
                </>
              )}
              {isBatchRunning && onToggleMinimize && (
                <button
                  className={styles.minimizeStepBtn}
                  onClick={onToggleMinimize}
                  title={t('uploadInfer.inferencePanel.minimizeStep2')}
                  aria-label={t('uploadInfer.inferencePanel.minimizeStep2')}
                >
                  <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                    strokeWidth="1.6" strokeLinecap="round">
                    <path d="M3 8h10" />
                  </svg>
                </button>
              )}
            </div>
          </div>


          {/* ── Main body ── */}
          <div className={`${styles.infpanelBody} ${isBatchRunning ? styles.infpanelBodyRunning : styles.infpanelBodyConfig}`}>

            {/* ── Submitting overlay — shown while awaiting /batch_process ── */}
            {running && (
              <div className={styles.submittingOverlay}>
                <div className={styles.submittingCard}>
                  <div className={styles.submittingSpinner} />
                  <div className={styles.submittingTitle}>{t('uploadInfer.inferencePanel.submitting')}</div>
                  <div className={styles.submittingDesc}>
                    {t('uploadInfer.inferencePanel.submittingDesc', { count: n }).split('\n').map((line, i) => <React.Fragment key={i}>{line}{i === 0 && <br />}</React.Fragment>)}
                  </div>
                  <div className={styles.submittingFiles}>
                    {selectedServerIds.slice(0, 5).map((id, i) => {
                      const f = selectFiles.find(sf => sf.id === id);
                      return f ? (
                        <div key={id} className={styles.submittingFile}>
                          <span className={styles.submittingDot} style={{ animationDelay: `${i * 0.15}s` }} />
                          {f.original_name}
                        </div>
                      ) : null;
                    })}
                    {selectedServerIds.length > 5 && (
                      <div className={styles.submittingMore}>+{selectedServerIds.length - 5} more</div>
                    )}
                  </div>
                </div>
              </div>
            )}

            {/* Selection banner — hidden while batch running or submitting */}
            {!isBatchRunning && !running && <div className={styles.selBanner}>
              <svg width="12" height="12" viewBox="0 0 16 16" fill="none" stroke="var(--blue)" strokeWidth="1.5" strokeLinecap="round">
                <path d="M2 8h12M8 3l5 5-5 5" />
              </svg>
              <span className={styles.selCt}>{n} file{n !== 1 ? 's' : ''}</span>
              <span className={styles.selNm}>
                {n === 0 ? t('uploadInfer.inferencePanel.selBannerEmpty') : `${n} file${n !== 1 ? 's' : ''} selected for inference`}
              </span>
            </div>}

            {/* ── Settings — hidden while batch running or submitting ── */}
            {!isBatchRunning && !running && <div className={styles.infSettingsWrap} data-tour="infer-settings">
              <div className={styles.settingsGrid}>

                {/* Generate content card */}
                <div className={styles.card}>
                  <div className={styles.cardT}>{t('uploadInfer.inferencePanel.generateContent')}</div>

                  {/* Model dropdown — moved to the top; deliberately narrow rather than spanning the row */}
                  <div className={styles.modelFieldTop} data-tour="infer-model">
                    <div className={styles.fg}>
                      <div className={styles.modelLabelRow}>
                        <label className={styles.fl}>
                          {t('uploadInfer.inferencePanel.modelLabel')} {mlLoading && <span className={styles.loadingDot}>…</span>}
                        </label>
                        <button
                          className={styles.refreshBtn}
                          onClick={fetchModels}
                          disabled={mlLoading}
                          title={t('uploadInfer.inferencePanel.refreshModels')}
                        >
                          <svg
                            viewBox="0 0 16 16" fill="none" stroke="currentColor"
                            strokeWidth="1.6" strokeLinecap="round" strokeLinejoin="round"
                            className={mlLoading ? styles.spinning : undefined}
                          >
                            <path d="M13.5 8A5.5 5.5 0 1 1 10 3.07" />
                            <path d="M10 2v3h3" />
                          </svg>
                          Refresh
                        </button>
                      </div>
                      <select className={styles.fc} value={selectedModel}
                        onChange={e => dispatch(setSelectedModel(e.target.value))}
                        disabled={mlLoading || models.length === 0}>
                        {models.length === 0
                          ? <option value="">{t('uploadInfer.inferencePanel.noModelsOption')}</option>
                          : models.map(m => <option key={m} value={m}>{m}</option>)
                        }
                      </select>
                      {!mlLoading && models.length === 0 && (
                        <div className={styles.modelWarn}>
                          <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                            <path d="M8 2L14 13H2L8 2z" /><path d="M8 7v3M8 11.5v.1" />
                          </svg>
                          {t('uploadInfer.inferencePanel.noModels')}
                        </div>
                      )}
                      {!mlLoading && models.length > 0 && (
                        <div className={styles.modelOk}>
                          <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                            <circle cx="8" cy="8" r="5.5" /><path d="M5.5 8l2 2 3-3" />
                          </svg>
                          {t('uploadInfer.inferencePanel.modelsAvailable', { count: models.length })}
                        </div>
                      )}
                    </div>
                  </div>

                  <div className={styles.genContentCols}>

                  {/* ── Left column: all checkboxes ── */}
                  <div className={styles.checkboxCol}>

                    <div data-tour="infer-check-summary"
                      className={`${styles.cr} ${isChecked(settings.generateSummary) ? styles.ck : ''} ${styles.mb8} ${noFilesSelected ? styles.crDisabled : ''}`}
                      onClick={() => { if (noFilesSelected) return; dispatch(updateSettings({ generateSummary: !settings.generateSummary })); }}
                    >
                      <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateSummary')}</label>
                    </div>

                    <div data-tour="infer-check-keywords"
                      className={`${styles.cr} ${isChecked(settings.generateKeywords) ? styles.ck : ''} ${styles.mb8} ${noFilesSelected ? styles.crDisabled : ''}`}
                      onClick={() => {
                        if (noFilesSelected) return;
                        dispatch(updateSettings({
                          generateKeywords: !settings.generateKeywords,
                          // Force the nested toggle off when the parent turns off,
                          // so a stale "on" state can never leak into the request.
                          ...(settings.generateKeywords ? { generateKeywordInsights: false } : {}),
                        }));
                      }}
                    >
                      <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateKeywords')}</label>
                    </div>
                    {isChecked(settings.generateKeywords) && (
                      <div
                        className={`${styles.crNested} ${isChecked(settings.generateKeywordInsights) ? styles.ck : ''} ${styles.mb8}`}
                        onClick={(e) => { e.stopPropagation(); if (noFilesSelected) return; dispatch(updateSettings({ generateKeywordInsights: !settings.generateKeywordInsights })); }}
                      >
                        <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateKeywordInsights')}</label>
                      </div>
                    )}

                    <div data-tour="infer-check-questions"
                      className={`${styles.cr} ${isChecked(settings.generateQuestions) ? styles.ck : ''} ${styles.mb8} ${noFilesSelected ? styles.crDisabled : ''}`}
                      onClick={() => { if (noFilesSelected) return; dispatch(updateSettings({ generateQuestions: !settings.generateQuestions })); }}
                    >
                      <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateQuestions')}</label>
                    </div>

                    <div data-tour="infer-check-shortanswer"
                      className={`${styles.cr} ${isChecked(settings.generateShortAnswer) ? styles.ck : ''} ${styles.mb8} ${noFilesSelected ? styles.crDisabled : ''}`}
                      onClick={() => { if (noFilesSelected) return; dispatch(updateSettings({ generateShortAnswer: !settings.generateShortAnswer })); }}
                    >
                      <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateShortAnswer')}</label>
                    </div>

                    <div data-tour="infer-check-truefalse"
                      className={`${styles.cr} ${isChecked(settings.generateTrueFalse) ? styles.ck : ''} ${styles.mb8} ${noFilesSelected ? styles.crDisabled : ''}`}
                      onClick={() => { if (noFilesSelected) return; dispatch(updateSettings({ generateTrueFalse: !settings.generateTrueFalse })); }}
                    >
                      <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateTrueFalse')}</label>
                    </div>

                    <div data-tour="infer-check-timestamped"
                      className={`${styles.cr} ${isChecked(settings.timestampedSummary) ? styles.ck : ''} ${styles.mb8} ${noFilesSelected ? styles.crDisabled : ''}`}
                      onClick={() => { if (noFilesSelected) return; dispatch(updateSettings({ timestampedSummary: !settings.timestampedSummary })); }}
                    >
                      <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.timestampedSummary')}</label>
                    </div>
                    {isChecked(settings.timestampedSummary) && (
                      <div className={styles.nestedField} onClick={(e) => e.stopPropagation()}>
                        <label>{t('uploadInfer.inferencePanel.timeInterval')}</label>
                        <select
                          className={styles.fc}
                          value={settings.timeInterval}
                          onChange={e => dispatch(updateSettings({ timeInterval: Number(e.target.value) as TimeInterval }))}
                        >
                          {([5, 10, 15, 20, 30, 45, 60] as const).map(min => (
                            <option key={min} value={min}>{t('uploadInfer.inferencePanel.minutesOption', { count: min })}</option>
                          ))}
                        </select>
                      </div>
                    )}

                  </div>

                  {/* ── Right column: prompt panel for whichever checkboxes are on ── */}
                  <div className={styles.promptCol}>

                    {!isChecked(settings.generateSummary) && !isChecked(settings.generateKeywords)
                      && !isChecked(settings.generateQuestions) && !isChecked(settings.generateShortAnswer)
                      && !isChecked(settings.generateTrueFalse) && (
                      <div className={styles.noPromptHint}>
                        {noFilesSelected
                          ? t('uploadInfer.inferencePanel.selBannerEmpty')
                          : t('uploadInfer.inferencePanel.noPromptHint', 'Check an option on the left to set its prompt')}
                      </div>
                    )}

                    {isChecked(settings.generateSummary) && (
                      <div className={styles.promptCard}>
                        <div className={styles.fg}>
                          <div className={styles.flRow}>
                            <label className={styles.fl}>
                              {t('uploadInfer.inferencePanel.summaryPrompt')} <span className={styles.optTag}>{t('uploadInfer.inferencePanel.optional')}</span>
                            </label>
                            {n > 1 && (
                              <button
                                className={`${styles.perFileBtn} ${summaryExpanded ? styles.perFileBtnActive : ''}`}
                                onClick={(e) => { e.stopPropagation(); setSummaryExpanded(v => !v); }}
                                title={summaryExpanded ? 'Collapse per-file prompts' : 'View per-file prompts'}
                              >
                                <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                  <rect x="1.5" y="2" width="11" height="10" rx="2" />
                                  <path d="M4 5h6M4 7.5h4" />
                                </svg>
                                {summaryExpanded ? 'Hide' : t('uploadInfer.inferencePanel.perFile')}
                                <svg className={`${styles.perFileChevron} ${summaryExpanded ? styles.perFileChevronOpen : ''}`} viewBox="0 0 10 10" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                                  <path d="M2 3.5l3 3 3-3" />
                                </svg>
                              </button>
                            )}
                          </div>
                          <div className={`${styles.perFilePanel} ${summaryExpanded ? styles.perFilePanelOpen : ''}`}>
                            <div className={styles.perFilePanelInner}>
                              {selectedFiles.map(f => (
                                <div key={f.id} className={styles.perFileRow}>
                                  <div className={styles.perFileName}>{f.original_name}</div>
                                  <div className={styles.perFilePrompt}>{f.summary_prompt || <span className={styles.perFileEmpty}>{t('uploadInfer.inferencePanel.noPromptSet')}</span>}</div>
                                </div>
                              ))}
                            </div>
                          </div>
                          <textarea className={styles.fc} rows={3}
                            value={settings.summaryPromptOverride}
                            placeholder={n === 1 ? t('uploadInfer.inferencePanel.autofillPlaceholder') : t('uploadInfer.inferencePanel.manualPlaceholder')}
                            onChange={e => dispatch(updateSummaryPrompt(e.target.value))} />
                        </div>
                      </div>
                    )}

                    {isChecked(settings.generateKeywords) && (
                      <div className={styles.promptCard}>
                        <div className={styles.fg}>
                          <div className={styles.flRow}>
                            <label className={styles.fl}>
                              {t('uploadInfer.inferencePanel.keywordPrompt')} <span className={styles.optTag}>{t('uploadInfer.inferencePanel.optional')}</span>
                            </label>
                            {n > 1 && (
                              <button
                                className={`${styles.perFileBtn} ${keywordExpanded ? styles.perFileBtnActive : ''}`}
                                onClick={(e) => { e.stopPropagation(); setKeywordExpanded(v => !v); }}
                                title={keywordExpanded ? 'Collapse per-file prompts' : 'View per-file prompts'}
                              >
                                <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                  <rect x="1.5" y="2" width="11" height="10" rx="2" />
                                  <path d="M4 5h6M4 7.5h4" />
                                </svg>
                                {keywordExpanded ? t('uploadInfer.inferencePanel.hide') : t('uploadInfer.inferencePanel.perFile')}
                                <svg className={`${styles.perFileChevron} ${keywordExpanded ? styles.perFileChevronOpen : ''}`} viewBox="0 0 10 10" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                                  <path d="M2 3.5l3 3 3-3" />
                                </svg>
                              </button>
                            )}
                          </div>
                          <div className={`${styles.perFilePanel} ${keywordExpanded ? styles.perFilePanelOpen : ''}`}>
                            <div className={styles.perFilePanelInner}>
                              {selectedFiles.map(f => (
                                <div key={f.id} className={styles.perFileRow}>
                                  <div className={styles.perFileName}>{f.original_name}</div>
                                  <div className={styles.perFilePrompt}>{f.keywords_prompt || <span className={styles.perFileEmpty}>No prompt set</span>}</div>
                                </div>
                              ))}
                            </div>
                          </div>
                          <textarea className={styles.fc} rows={3}
                            value={settings.keywordPromptOverride}
                            placeholder={n === 1 ? 'Autofilled from file — edit to override…' : 'Leave blank to use per-file prompts…'}
                            onChange={e => dispatch(updateKeywordPrompt(e.target.value))} />
                        </div>
                      </div>
                    )}

                    {isChecked(settings.generateQuestions) && (
                      <div className={styles.promptCard}>
                        <div className={styles.fg}>
                          <div className={styles.flRow}>
                            <label className={styles.fl}>
                              {t('uploadInfer.inferencePanel.questionPrompt')} <span className={styles.optTag}>{t('uploadInfer.inferencePanel.optional')}</span>
                            </label>
                            {n > 1 && (
                              <button
                                className={`${styles.perFileBtn} ${questionExpanded ? styles.perFileBtnActive : ''}`}
                                onClick={(e) => { e.stopPropagation(); setQuestionExpanded(v => !v); }}
                                title={questionExpanded ? 'Collapse per-file prompts' : 'View per-file prompts'}
                              >
                                <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                  <rect x="1.5" y="2" width="11" height="10" rx="2" />
                                  <path d="M4 5h6M4 7.5h4" />
                                </svg>
                                {questionExpanded ? t('uploadInfer.inferencePanel.hide') : t('uploadInfer.inferencePanel.perFile')}
                                <svg className={`${styles.perFileChevron} ${questionExpanded ? styles.perFileChevronOpen : ''}`} viewBox="0 0 10 10" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                                  <path d="M2 3.5l3 3 3-3" />
                                </svg>
                              </button>
                            )}
                          </div>
                          <div className={`${styles.perFilePanel} ${questionExpanded ? styles.perFilePanelOpen : ''}`}>
                            <div className={styles.perFilePanelInner}>
                              {selectedFiles.map(f => (
                                <div key={f.id} className={styles.perFileRow}>
                                  <div className={styles.perFileName}>{f.original_name}</div>
                                  <div className={styles.perFilePrompt}>{f.faq_prompt || <span className={styles.perFileEmpty}>No prompt set</span>}</div>
                                </div>
                              ))}
                            </div>
                          </div>
                          <textarea className={styles.fc} rows={3}
                            value={settings.questionPromptOverride}
                            placeholder={n === 1 ? 'Autofilled from file — edit to override…' : 'Leave blank to use per-file prompts…'}
                            onChange={e => dispatch(updateQuestionPrompt(e.target.value))} />
                        </div>
                      </div>
                    )}

                    {isChecked(settings.generateShortAnswer) && (
                      <div className={styles.promptCard}>
                        <div className={styles.fg}>
                          <div className={styles.flRow}>
                            <label className={styles.fl}>
                              {t('uploadInfer.inferencePanel.shortAnswerPrompt')} <span className={styles.optTag}>{t('uploadInfer.inferencePanel.optional')}</span>
                            </label>
                            {n > 1 && (
                              <button
                                className={`${styles.perFileBtn} ${shortAnswerExpanded ? styles.perFileBtnActive : ''}`}
                                onClick={(e) => { e.stopPropagation(); setShortAnswerExpanded(v => !v); }}
                                title={shortAnswerExpanded ? t('uploadInfer.inferencePanel.hide') : t('uploadInfer.inferencePanel.viewPerFile')}
                              >
                                <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                  <rect x="1.5" y="2" width="11" height="10" rx="2" />
                                  <path d="M4 5h6M4 7.5h4" />
                                </svg>
                                {shortAnswerExpanded ? t('uploadInfer.inferencePanel.hide') : t('uploadInfer.inferencePanel.perFile')}
                                <svg className={`${styles.perFileChevron} ${shortAnswerExpanded ? styles.perFileChevronOpen : ''}`} viewBox="0 0 10 10" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                                  <path d="M2 3.5l3 3 3-3" />
                                </svg>
                              </button>
                            )}
                          </div>
                          <div className={`${styles.perFilePanel} ${shortAnswerExpanded ? styles.perFilePanelOpen : ''}`}>
                            <div className={styles.perFilePanelInner}>
                              {selectedFiles.map(f => (
                                <div key={f.id} className={styles.perFileRow}>
                                  <div className={styles.perFileName}>{f.original_name}</div>
                                  <div className={styles.perFilePrompt}>{f.short_answer_prompt || <span className={styles.perFileEmpty}>{t('uploadInfer.inferencePanel.noPromptSet')}</span>}</div>
                                </div>
                              ))}
                            </div>
                          </div>
                          <textarea className={styles.fc} rows={3}
                            value={settings.shortAnswerPromptOverride}
                            placeholder={n === 1 ? t('uploadInfer.inferencePanel.autofillPlaceholder') : t('uploadInfer.inferencePanel.manualPlaceholder')}
                            onChange={e => dispatch(updateShortAnswerPrompt(e.target.value))} />
                        </div>
                      </div>
                    )}

                    {isChecked(settings.generateTrueFalse) && (
                      <div className={styles.promptCard}>
                        <div className={styles.fg}>
                          <div className={styles.flRow}>
                            <label className={styles.fl}>
                              {t('uploadInfer.inferencePanel.trueFalsePrompt')} <span className={styles.optTag}>{t('uploadInfer.inferencePanel.optional')}</span>
                            </label>
                            {n > 1 && (
                              <button
                                className={`${styles.perFileBtn} ${trueFalseExpanded ? styles.perFileBtnActive : ''}`}
                                onClick={(e) => { e.stopPropagation(); setTrueFalseExpanded(v => !v); }}
                                title={trueFalseExpanded ? t('uploadInfer.inferencePanel.hide') : t('uploadInfer.inferencePanel.viewPerFile')}
                              >
                                <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                  <rect x="1.5" y="2" width="11" height="10" rx="2" />
                                  <path d="M4 5h6M4 7.5h4" />
                                </svg>
                                {trueFalseExpanded ? t('uploadInfer.inferencePanel.hide') : t('uploadInfer.inferencePanel.perFile')}
                                <svg className={`${styles.perFileChevron} ${trueFalseExpanded ? styles.perFileChevronOpen : ''}`} viewBox="0 0 10 10" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                                  <path d="M2 3.5l3 3 3-3" />
                                </svg>
                              </button>
                            )}
                          </div>
                          <div className={`${styles.perFilePanel} ${trueFalseExpanded ? styles.perFilePanelOpen : ''}`}>
                            <div className={styles.perFilePanelInner}>
                              {selectedFiles.map(f => (
                                <div key={f.id} className={styles.perFileRow}>
                                  <div className={styles.perFileName}>{f.original_name}</div>
                                  <div className={styles.perFilePrompt}>{f.true_false_prompt || <span className={styles.perFileEmpty}>{t('uploadInfer.inferencePanel.noPromptSet')}</span>}</div>
                                </div>
                              ))}
                            </div>
                          </div>
                          <textarea className={styles.fc} rows={3}
                            value={settings.trueFalsePromptOverride}
                            placeholder={n === 1 ? t('uploadInfer.inferencePanel.autofillPlaceholder') : t('uploadInfer.inferencePanel.manualPlaceholder')}
                            onChange={e => dispatch(updateTrueFalsePrompt(e.target.value))} />
                        </div>
                      </div>
                    )}

                  </div>

                  </div>

                  {/* Save / Cancel bar — persists any edited prompt overrides above */}
                  {n > 0 && (settings.generateSummary || settings.generateKeywords || settings.generateQuestions
                    || settings.generateShortAnswer || settings.generateTrueFalse) && (
                    <div className={styles.promptActions}>
                      {promptSaveError && (
                        <span className={styles.promptSaveError}>{promptSaveError}</span>
                      )}
                      <button
                        className={`${styles.btn} ${styles.btnSm} ${styles.btnGhost}`}
                        onClick={handlePromptCancel}
                        disabled={!promptsDirty || promptSaving}
                      >
                        Cancel
                      </button>
                      <button
                        className={`${styles.btn} ${styles.btnSm} ${styles.btnPrimary}`}
                        onClick={handlePromptSave}
                        disabled={!promptsDirty || promptSaving}
                      >
                        {promptSaving ? (
                          <><span className={styles.btnSpinner} />{t('uploadInfer.inferencePanel.saving')}</>
                        ) : t('uploadInfer.inferencePanel.savePrompts')}
                      </button>
                    </div>
                  )}
                </div>

              </div>
            </div>}

            {/* ══ 3-Column batch status ══ */}
            {isBatchRunning && (
              <div className={styles.batchSection}>
                <div className={styles.batchSectionTitle}>
                  <span className={styles.liveLabel}>{t('uploadInfer.inferencePanel.inferenceStatus')}</span>
                  {isBatchRunning && (
                    <span className={styles.liveStatus}>
                      <span className={styles.liveDot} />
                      Live
                      <span className={styles.liveSep}>·</span>
                      {t('uploadInfer.inferencePanel.refreshingIn', { sec: countdown })}
                      {batchData.running.length > 0 && (
                        <><span className={styles.liveSep}>·</span>
                          <span className={styles.liveFiles}>
                            {t('uploadInfer.inferencePanel.filesRunning', { count: batchData.running.length })}
                          </span></>
                      )}
                    </span>
                  )}
                </div>
                <div className={styles.batchColumns}>

                  {/* Queued */}
                  <div className={styles.batchCol}>
                    <div className={`${styles.batchColHead} ${styles.headQueued}`}>
                      <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round">
                        <circle cx="7" cy="7" r="5" /><path d="M7 4v3.5l2 1.5" />
                      </svg>
                      Queued
                      <span className={styles.batchColCount}>{batchData.queued.length}</span>
                    </div>
                    <div className={styles.batchColBody}>
                      {batchData.queued.length === 0
                        ? <div className={styles.batchEmpty}>—</div>
                        : batchData.queued.map(f => (
                          <StatusCard
                            key={f.id}
                            file={f}
                            variant="queued"
                            onStop={handleStop}
                            stopping={stoppingIds.has(f.id)}
                          />
                        ))
                      }
                    </div>
                  </div>

                  {/* Running */}
                  <div className={styles.batchCol}>
                    <div className={`${styles.batchColHead} ${styles.headRunning}`}>
                      <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round">
                        <circle cx="7" cy="7" r="5" /><path d="M7 4v3M7 9.5v.2" />
                      </svg>
                      Running
                      <span className={styles.batchColCount}>{batchData.running.length}</span>
                    </div>
                    <div className={styles.batchColBody}>
                      {batchData.running.length === 0
                        ? <div className={styles.batchEmpty}>—</div>
                        : batchData.running.map(f => <StatusCard key={f.id} file={f} variant="running" />)
                      }
                    </div>
                  </div>

                  {/* Completed */}
                  <div className={styles.batchCol}>
                    <div className={`${styles.batchColHead} ${styles.headCompleted}`}>
                      <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                        <circle cx="7" cy="7" r="5" /><path d="M4.5 7l2 2 3-3" />
                      </svg>
                      Completed
                      <span className={styles.batchColCount}>{batchData.completed.length}</span>
                    </div>
                    <div className={styles.batchColBody}>
                      {batchData.completed.length === 0
                        ? <div className={styles.batchEmpty}>—</div>
                        : batchData.completed.map(f => <StatusCard key={f.id} file={f} variant="completed" />)
                      }
                    </div>
                  </div>

                </div>
              </div>
            )}

          </div>
        </>
      )}
    </div>
  );
};

export default InferencePanel;


















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

// Two columns inside the merged Generate content card:
// left = all option checkboxes stacked; right = the prompt panel(s) for
// whichever checkboxes are currently checked.
.genContentCols {
  display: grid;
  grid-template-columns: minmax(180px, 240px) 1fr;
  gap: 4px 24px;
  align-items: start;
}

.checkboxCol {
  min-width: 0;
  display: flex;
  flex-direction: column;
  border-right: 1px solid var(--bdr);
  padding-right: 20px;
}

.promptCol {
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 14px;
}

@media (max-width: 860px) {
  .genContentCols {
    grid-template-columns: 1fr;
  }

  .checkboxCol {
    border-right: none;
    border-bottom: 1px solid var(--bdr);
    padding-right: 0;
    padding-bottom: 12px;
    margin-bottom: 4px;
  }
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
// Kept for any remaining inline usages.
.promptInline {
  padding-left: 20px;
  margin-bottom: 4px;
  cursor: default;
}

// Prompt panel rendered in the right-hand column, one per checked option.
.promptCard {
  padding: 10px 12px;
  background: var(--bg0);
  border: 1px solid var(--bdr);
  border-radius: var(--r);
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



















// ═══════════════════════════════════════════════
// store/uploadSlice.ts
// Content Analytics · Upload & Inference state
// ═══════════════════════════════════════════════
import { createSlice, type PayloadAction } from '@reduxjs/toolkit';

// ── Server file (from /files/by-date/) ──────────
// "tbd" = not yet inferenced, "completed" = inferenced.
export type FileStatus = 'waiting' | 'running' | 'completed' | 'error' | 'tbd' | 'queued';

export interface ServerFile {
  id: number;
  original_name: string;
  inserted_at: string;
  summary_prompt: string;
  keywords_prompt: string;
  faq_prompt: string;
  short_answer_prompt: string;
  true_false_prompt: string;
  progress: string | number | null;
  dictionary_id?: number | null;
  prompt_template_id?: number | null;
  status: FileStatus;
}

// ── /files/by-date/ pagination params & sort keys ──
export type FilesSortBy = 'id' | 'original_name' | 'inserted_at' | 'status';
export type SortOrder = 'asc' | 'desc';

export interface ServerFilesData {
  queued: ServerFile[];
  completed: ServerFile[];
  pending: ServerFile[];
  running: ServerFile[];
}

export interface UploadedFile {
  id: number;
  name: string;
  size: string;
  type: 'vtt' | 'srt';
  status: 'ready' | 'running' | 'done' | 'failed';
  summaryPrompt: string;
  questionPrompt: string;
}

export interface BatchFile {
  id: number;
  name: string;
  size: string;
  type: 'vtt' | 'srt';
  progress?: number;
}

export interface BatchGroup {
  id: string;
  status: 'running' | 'queued' | 'done' | 'pending';
  files: BatchFile[];
  collapsed: boolean;
}

export type TimeInterval = 5 | 10 | 15 | 20 | 30 | 45 | 60;

export interface InferenceSettings {
  generateSummary: boolean;
  generateKeywords: boolean;
  generateQuestions: boolean;
  summaryStyle: string;
  timestampInterval: string;
  languageOutput: string;
  mcq: boolean;
  trueFalse: boolean;
  shortAnswer: boolean;
  questionCount: number;
  keywordCount: number;
  summaryPromptOverride: string;
  keywordPromptOverride: string;
  questionPromptOverride: string;

  // ── New independent content types ──
  generateShortAnswer: boolean;
  generateTrueFalse: boolean;
  // Only meaningful (and only sent as true) when generateKeywords is also true.
  generateKeywordInsights: boolean;
  timestampedSummary: boolean;
  timeInterval: TimeInterval;
  shortAnswerPromptOverride: string;
  trueFalsePromptOverride: string;
}

interface UploadState {
  files: UploadedFile[];
  selectedIds: number[];
  settingsCollapsed: boolean;
  batchVisible: boolean;
  batchGroups: BatchGroup[];
  settings: InferenceSettings;
  uploadZoneCollapsed: boolean;

  // ── Server files (from /files/by-date/) ──
  // NOTE: serverFilesData is now populated exclusively by inferenceStatusSuccess
  // (the /files/by-progress/ polling endpoint), which still returns the old
  // bucketed shape. /files/by-date/ is paginated and returns a flat, per-file
  // `status` — so it no longer feeds serverFilesData.
  serverFilesData: ServerFilesData;          // raw split by status (from /files/by-progress/)
  serverFiles: ServerFile[];             // current page of /files/by-date/ results (Upload tab only)
  serverFilesLoading: boolean;
  serverFilesError: string | null;
  // Separate from serverFiles above — that one belongs to the Upload tab
  // and gets cleared whenever it goes inactive. FileSidebar (Infer tab,
  // mode="select") publishes its own fetched page here instead, so
  // InferencePanel has a live source of full file objects (prompt
  // defaults included) for whichever file(s) are currently selected.
  selectFiles: ServerFile[];
  dateFrom: string;
  dateTo: string;
  statusFilter: FileStatus | '';

  // ── /files/by-date/ pagination, sort & search ──
  filesPage: number;
  filesPageSize: number;
  filesTotal: number;
  filesTotalPages: number;
  filesSortBy: FilesSortBy;
  filesSortOrder: SortOrder;
  filesSearch: string;

  // ── Selection on uploaded files ──
  selectedServerIds: number[];

  // ── Batch running state ──
  isBatchRunning: boolean;                  // true while queued/running not empty
  lastBatchFinishedAt: number | null;       // timestamp (Date.now()) set when batch transitions running→done
  lastFileCompletedAt: number | null;       // timestamp (Date.now()) set whenever any individual file finishes inferring

  // ── Models ──
  models: string[];
  modelsLoading: boolean;
  selectedModel: string;
}

function toDateStr(d: Date) { return d.toISOString().slice(0, 10); }
function defaultFrom() { const d = new Date(); d.setMonth(d.getMonth() - 1); return toDateStr(d); }
function defaultTo() { return toDateStr(new Date()); }

const emptyData: ServerFilesData = { queued: [], completed: [], pending: [], running: [] };

const initialState: UploadState = {
  files: [],
  selectedIds: [],
  uploadZoneCollapsed: false,
  settingsCollapsed: false,
  batchVisible: false,
  batchGroups: [],
  settings: {
    generateSummary: true, generateKeywords: true, generateQuestions: true,
    summaryStyle: 'Table of Contents', timestampInterval: '5 minute segments',
    languageOutput: 'Korean + English',
    mcq: true, trueFalse: true, shortAnswer: false,
    questionCount: 10, keywordCount: 15,
    summaryPromptOverride: '', keywordPromptOverride: '', questionPromptOverride: '',

    generateShortAnswer: false, generateTrueFalse: false,
    generateKeywordInsights: false,
    timestampedSummary: false, timeInterval: 5,
    shortAnswerPromptOverride: '', trueFalsePromptOverride: '',
  },
  serverFilesData: emptyData,
  serverFiles: [],
  selectFiles: [],
  serverFilesLoading: false,
  serverFilesError: null,
  dateFrom: defaultFrom(),
  dateTo: defaultTo(),
  statusFilter: '',

  filesPage: 1,
  filesPageSize: 50,
  filesTotal: 0,
  filesTotalPages: 1,
  filesSortBy: 'id',
  filesSortOrder: 'desc',
  filesSearch: '',

  selectedServerIds: [],
  isBatchRunning: false,
  lastBatchFinishedAt: null,
  lastFileCompletedAt: null,
  models: [],
  modelsLoading: false,
  selectedModel: '',
};

const uploadSlice = createSlice({
  name: 'upload',
  initialState,
  reducers: {
    // ── Inference legacy ──
    toggleFileSelection(state, action: PayloadAction<number>) {
      const idx = state.selectedIds.indexOf(action.payload);
      if (idx > -1) state.selectedIds.splice(idx, 1);
      else state.selectedIds.push(action.payload);
    },
    toggleSelectAll(state) {
      const allIds = state.files.map(f => f.id);
      state.selectedIds = state.selectedIds.length === allIds.length ? [] : allIds;
    },
    toggleUploadZone(state) {
      state.uploadZoneCollapsed = !state.uploadZoneCollapsed;
    },
    toggleBatchGroup(state, action: PayloadAction<string>) {
      const g = state.batchGroups.find(g => g.id === action.payload);
      if (g) g.collapsed = !g.collapsed;
    },
    runInference(state) {
      state.batchVisible = true;
    },
    updateSummaryPrompt(state, action: PayloadAction<string>) {
      state.settings.summaryPromptOverride = action.payload;
    },
    updateKeywordPrompt(state, action: PayloadAction<string>) {
      state.settings.keywordPromptOverride = action.payload;
    },
    updateQuestionPrompt(state, action: PayloadAction<string>) {
      state.settings.questionPromptOverride = action.payload;
    },
    updateShortAnswerPrompt(state, action: PayloadAction<string>) {
      state.settings.shortAnswerPromptOverride = action.payload;
    },
    updateTrueFalsePrompt(state, action: PayloadAction<string>) {
      state.settings.trueFalsePromptOverride = action.payload;
    },
    updateSettings(state, action: PayloadAction<Partial<InferenceSettings>>) {
      state.settings = { ...state.settings, ...action.payload };
    },

    // ── Date range ──
    setDateFrom(state, action: PayloadAction<string>) { state.dateFrom = action.payload; },
    setDateTo(state, action: PayloadAction<string>) { state.dateTo = action.payload; },
    setStatusFilter(state, action: PayloadAction<FileStatus | ''>) { state.statusFilter = action.payload; },

    // ── Patch progress % onto running files from /files/progress/ response ──
    updateRunningProgress(state, action: PayloadAction<Record<string, number | string>>) {
      const progressMap = action.payload;
      state.serverFilesData.running = state.serverFilesData.running.map(f => {
        const raw = progressMap[String(f.id)];
        if (raw === undefined) return f;
        const pct = typeof raw === 'string' ? parseFloat(raw) : raw;
        return { ...f, progress: isNaN(pct) ? f.progress : Math.min(100, Math.max(0, pct)) };
      });
    },

    // ── Inference-only status update (does NOT touch the FilePanel file list) ──
    inferenceStatusSuccess(state, action: PayloadAction<ServerFilesData>) {
      const d = action.payload;
      // Merge any newly-completed files from the polling response into the
      // existing completed bucket. We keep the historical /by-date/ entries
      // and append/replace any IDs that just finished in the active batch,
      // so FilePanel badges flip to "Inferenced" the moment Step-2 moves a
      // file into its Completed column — without needing another API call.
      const incomingCompleted = d.completed ?? [];
      const incomingCompletedIds = new Set(incomingCompleted.map(f => f.id));
      const priorCompletedIds = new Set(state.serverFilesData.completed.map(f => f.id));
      const mergedCompleted = [
        ...state.serverFilesData.completed.filter(f => !incomingCompletedIds.has(f.id)),
        ...incomingCompleted,
      ];
      // A file just finished inferring individually (as opposed to the whole
      // batch finishing) whenever an id shows up in `completed` that wasn't
      // there on the previous poll. Stamp a timestamp so FileSidebar can
      // re-fetch /files/by-date/ right away instead of waiting for the
      // entire batch to drain.
      const hasNewlyCompleted = incomingCompleted.some(f => !priorCompletedIds.has(f.id));
      if (hasNewlyCompleted) {
        state.lastFileCompletedAt = Date.now();
      }
      state.serverFilesData = {
        queued: d.queued ?? [],
        running: d.running ?? [],
        pending: d.pending ?? [],
        completed: mergedCompleted,
      };
      const running = (d.queued?.length ?? 0) > 0 || (d.running?.length ?? 0) > 0;
      // Clear selections the moment the batch becomes active so highlighted
      // cards stop showing the blue selected state after a successful submit.
      if (running && !state.isBatchRunning) {
        state.selectedServerIds = [];
      }
      // Auto-collapse settings when a batch starts running. (Previously derived
      // from /files/by-date/, which is now paginated and can't reliably tell us this.)
      if (running) state.settingsCollapsed = true;
      // When batch transitions running → done, stamp a timestamp so FilePanel
      // knows to re-fetch /by-date/ and refresh the completed list.
      if (!running && state.isBatchRunning) {
        state.lastBatchFinishedAt = Date.now();
      }
      state.isBatchRunning = running;
    },

    // ── Select-mode files (Infer tab's FileSidebar, mode="select") ──
    // Deliberately minimal — just the current page of full ServerFile
    // objects, so InferencePanel can look up prompt defaults for whatever
    // is in selectedServerIds. No pagination/sort/search mirrored here;
    // FileSidebar keeps that locally, this is purely a lookup source.
    selectFilesSuccess(state, action: PayloadAction<ServerFile[]>) {
      state.selectFiles = action.payload;
    },

    // ── Server files (paginated /files/by-date/) ──
    serverFilesLoading(state) {
      state.serverFilesLoading = true;
      state.serverFilesError = null;
    },
    serverFilesSuccess(state, action: PayloadAction<{
      files: ServerFile[];
      total: number;
      page: number;
      pageSize: number;
      totalPages: number;
      sortBy: FilesSortBy;
      sortOrder: SortOrder;
      search: string;
    }>) {
      const { files, total, page, pageSize, totalPages, sortBy, sortOrder, search } = action.payload;
      state.serverFiles = files;
      state.serverFilesLoading = false;
      state.serverFilesError = null;

      state.filesTotal = total;
      state.filesPage = page;
      state.filesPageSize = pageSize;
      state.filesTotalPages = totalPages;
      state.filesSortBy = sortBy;
      state.filesSortOrder = sortOrder;
      state.filesSearch = search;

      // NOTE: isBatchRunning is no longer derived here — /files/by-date/ is
      // paginated so a running/queued file may simply be on another page.
      // isBatchRunning is set by inferenceStatusSuccess (/files/by-progress/),
      // which always reflects the full, unpaginated set of active files.

      // NOTE: selections are intentionally NOT pruned against this page's ids —
      // a file not present here may just be on a different page, not deleted.
      // Deletion flows already clear selectedServerIds explicitly on success.
    },
    serverFilesFailure(state, action: PayloadAction<string>) {
      state.serverFilesLoading = false;
      state.serverFilesError = action.payload;
    },

    // ── Pagination / sort / search (kept in sync for UI display; the actual
    //     fetch is triggered imperatively by the component with these values) ──
    setFilesPage(state, action: PayloadAction<number>) {
      state.filesPage = action.payload;
    },
    setFilesPageSize(state, action: PayloadAction<number>) {
      state.filesPageSize = action.payload;
      state.filesPage = 1;
    },
    setFilesSort(state, action: PayloadAction<{ sortBy: FilesSortBy; sortOrder: SortOrder }>) {
      state.filesSortBy = action.payload.sortBy;
      state.filesSortOrder = action.payload.sortOrder;
      state.filesPage = 1;
    },
    setFilesSearch(state, action: PayloadAction<string>) {
      state.filesSearch = action.payload;
      state.filesPage = 1;
    },

    // ── Server file selection ──
    toggleServerFileSelection(state, action: PayloadAction<number>) {
      if (state.isBatchRunning) return;  // no selection while batch running
      const idx = state.selectedServerIds.indexOf(action.payload);
      if (idx > -1) state.selectedServerIds.splice(idx, 1);
      else state.selectedServerIds.push(action.payload);
    },
    toggleSelectAllServerFiles(state, action: PayloadAction<number[]>) {
      if (state.isBatchRunning) return;
      const pageIds = action.payload;
      if (pageIds.length === 0) return;
      const selectedSet = new Set(state.selectedServerIds);
      const allPageSelected = pageIds.every(id => selectedSet.has(id));
      if (allPageSelected) {
        // Uncheck — remove only this page's ids, leave other pages' selections alone.
        const pageIdSet = new Set(pageIds);
        state.selectedServerIds = state.selectedServerIds.filter(id => !pageIdSet.has(id));
      } else {
        // Check — add this page's ids on top of whatever's already selected elsewhere.
        pageIds.forEach(id => { if (!selectedSet.has(id)) state.selectedServerIds.push(id); });
      }
    },
    clearServerSelection(state) {
      state.selectedServerIds = [];
    },

    // ── Patch prompt fields on saved files ──
    updateFilePrompts(state, action: PayloadAction<{
      fileIds: number[];
      summaryPrompt?: string;
      keywordsPrompt?: string;
      faqPrompt?: string;
      shortAnswerPrompt?: string;
      trueFalsePrompt?: string;
    }>) {
      const { fileIds, summaryPrompt, keywordsPrompt, faqPrompt, shortAnswerPrompt, trueFalsePrompt } = action.payload;
      const idSet = new Set(fileIds);
      state.serverFiles = state.serverFiles.map(f => {
        if (!idSet.has(f.id)) return f;
        return {
          ...f,
          ...(summaryPrompt !== undefined && { summary_prompt: summaryPrompt }),
          ...(keywordsPrompt !== undefined && { keywords_prompt: keywordsPrompt }),
          ...(faqPrompt !== undefined && { faq_prompt: faqPrompt }),
          ...(shortAnswerPrompt !== undefined && { short_answer_prompt: shortAnswerPrompt }),
          ...(trueFalsePrompt !== undefined && { true_false_prompt: trueFalsePrompt }),
        };
      });
      // Also patch inside serverFilesData buckets so the per-file preview reflects the change
      const patchBucket = (bucket: ServerFile[]) => bucket.map(f => {
        if (!idSet.has(f.id)) return f;
        return {
          ...f,
          ...(summaryPrompt !== undefined && { summary_prompt: summaryPrompt }),
          ...(keywordsPrompt !== undefined && { keywords_prompt: keywordsPrompt }),
          ...(faqPrompt !== undefined && { faq_prompt: faqPrompt }),
          ...(shortAnswerPrompt !== undefined && { short_answer_prompt: shortAnswerPrompt }),
          ...(trueFalsePrompt !== undefined && { true_false_prompt: trueFalsePrompt }),
        };
      });
      state.serverFilesData.completed = patchBucket(state.serverFilesData.completed);
      state.serverFilesData.queued = patchBucket(state.serverFilesData.queued);
      state.serverFilesData.running = patchBucket(state.serverFilesData.running);
      state.serverFilesData.pending = patchBucket(state.serverFilesData.pending);
    },


    // ── Patch dictionary_id on saved files (after associate/disassociate) ──
    patchFileDictionaries(state, action: PayloadAction<Record<number, number | null>>) {
      const map = action.payload;
      const apply = (f: ServerFile): ServerFile =>
        map[f.id] !== undefined ? { ...f, dictionary_id: map[f.id] } : f;
      state.serverFiles = state.serverFiles.map(apply);
      state.serverFilesData.completed = state.serverFilesData.completed.map(apply);
      state.serverFilesData.queued = state.serverFilesData.queued.map(apply);
      state.serverFilesData.running = state.serverFilesData.running.map(apply);
      state.serverFilesData.pending = state.serverFilesData.pending.map(apply);
    },

    // ── Patch prompt_template_id on saved files (after template association) ──
    patchFilePromptTemplate(state, action: PayloadAction<Record<number, number | null>>) {
      const map = action.payload;
      const apply = (f: ServerFile): ServerFile =>
        map[f.id] !== undefined ? { ...f, prompt_template_id: map[f.id] } : f;
      state.serverFiles = state.serverFiles.map(apply);
      state.serverFilesData.completed = state.serverFilesData.completed.map(apply);
      state.serverFilesData.queued = state.serverFilesData.queued.map(apply);
      state.serverFilesData.running = state.serverFilesData.running.map(apply);
      state.serverFilesData.pending = state.serverFilesData.pending.map(apply);
    },

    // ── Models ──
    modelsLoading(state) {
      state.modelsLoading = true;
    },
    modelsSuccess(state, action: PayloadAction<string[]>) {
      state.models = action.payload;
      state.modelsLoading = false;
      if (action.payload.length > 0 && !state.selectedModel) {
        state.selectedModel = action.payload[0];
      }
    },
    modelsFailure(state) {
      state.modelsLoading = false;
    },
    setSelectedModel(state, action: PayloadAction<string>) {
      state.selectedModel = action.payload;
    },
  },
});

export const {
  toggleFileSelection, toggleSelectAll,
  toggleUploadZone, toggleBatchGroup,
  runInference, updateSummaryPrompt, updateKeywordPrompt, updateQuestionPrompt,
  updateShortAnswerPrompt, updateTrueFalsePrompt, updateSettings,
  setDateFrom, setDateTo, setStatusFilter,
  inferenceStatusSuccess,
  updateRunningProgress,
  serverFilesLoading, serverFilesSuccess, serverFilesFailure,
  selectFilesSuccess,
  setFilesPage, setFilesPageSize, setFilesSort, setFilesSearch,
  toggleServerFileSelection, toggleSelectAllServerFiles, clearServerSelection,
  updateFilePrompts,
  patchFileDictionaries,
  patchFilePromptTemplate,
  modelsLoading, modelsSuccess, modelsFailure, setSelectedModel,
} = uploadSlice.actions;

export default uploadSlice.reducer;
















// ═══════════════════════════════════════════════
// pages/UploadInfer/FileSidebar.tsx
// Content Analytics · Shared 400px file sidebar
// Used by both the Infer tab (mode="select") and the Workspace tab
// (mode="view"). Deliberately does NOT read the Upload tab's shared
// `serverFiles` — it keeps its own local list, date filter, sort,
// search, and pagination, and fetches independently every time its
// tab becomes active. The only thing still shared with the rest of
// the app is `selectedServerIds`, since InferencePanel needs that to
// run a batch.
// ═══════════════════════════════════════════════
import React, { useCallback, useEffect, useRef, useState } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import { toggleServerFileSelection, toggleSelectAllServerFiles, selectFilesSuccess, ServerFile, FilesSortBy, SortOrder, FileStatus } from '../../store/uploadSlice';
import api from '../../services/api';
import styles from './FileSidebar.module.scss';

const getExt = (n: string): 'vtt' | 'srt' => n.toLowerCase().endsWith('.srt') ? 'srt' : 'vtt';

function toDateStr(d: Date) { return d.toISOString().slice(0, 10); }
function defaultFrom() { const d = new Date(); d.setMonth(d.getMonth() - 1); return toDateStr(d); }
function defaultTo() { return toDateStr(new Date()); }

const PAGE_WINDOW = 4;
const getPageNumbers = (current: number, total: number): number[] => {
  if (total <= PAGE_WINDOW) return Array.from({ length: total }, (_, i) => i + 1);
  const start = Math.max(1, Math.min(current - 1, total - PAGE_WINDOW + 1));
  return Array.from({ length: PAGE_WINDOW }, (_, i) => start + i);
};

type SortKey = 'id' | 'name' | 'date' | 'status';
const SORT_KEY_TO_API: Record<SortKey, FilesSortBy> = {
  id: 'id', name: 'original_name', date: 'inserted_at', status: 'status',
};
const API_TO_SORT_KEY: Record<FilesSortBy, SortKey> = {
  id: 'id', original_name: 'name', inserted_at: 'date', status: 'status',
};

// ── Status → badge label/color — same 5-state mapping as the Upload tab ──
const STATUS_META: Record<FileStatus, { labelKey: string; cls: string }> = {
  waiting: { labelKey: 'uploadInfer.filePanel.statusWaiting', cls: 'bWaiting' },
  running: { labelKey: 'uploadInfer.filePanel.statusRunning', cls: 'bRunning' },
  completed: { labelKey: 'uploadInfer.filePanel.inferenced', cls: 'bInferred' },
  error: { labelKey: 'uploadInfer.filePanel.notInferenced', cls: 'bNotInferred' },
  tbd: { labelKey: 'uploadInfer.filePanel.notInferenced', cls: 'bNotInferred' },
  queued: { labelKey: 'uploadInfer.filePanel.statusQueued', cls: 'bQueued' },
};

const STATUS_FILTER_OPTIONS: { value: FileStatus | ''; labelKey: string }[] = [
  { value: '', labelKey: 'uploadInfer.filePanel.statusAll' },
  { value: 'waiting', labelKey: 'uploadInfer.filePanel.statusWaiting' },
  { value: 'queued', labelKey: 'uploadInfer.filePanel.statusQueued' },
  { value: 'running', labelKey: 'uploadInfer.filePanel.statusRunning' },
  { value: 'completed', labelKey: 'uploadInfer.filePanel.inferenced' },
  { value: 'error', labelKey: 'uploadInfer.filePanel.statusError' },
  { value: 'tbd', labelKey: 'uploadInfer.filePanel.statusTbd' },
];

interface CbProps { checked: boolean; indeterminate?: boolean; onChange: () => void; disabled?: boolean; }
const Checkbox: React.FC<CbProps> = ({ checked, indeterminate, onChange, disabled }) => (
  <div
    className={`${styles.cb} ${checked ? styles.cbChecked : ''} ${indeterminate ? styles.cbIndet : ''} ${disabled ? styles.cbDisabled : ''}`}
    onClick={e => { e.stopPropagation(); if (!disabled) onChange(); }}
    role="checkbox" aria-checked={indeterminate ? 'mixed' : checked} tabIndex={0}
    onKeyDown={e => { if (e.key === ' ' || e.key === 'Enter') { e.preventDefault(); if (!disabled) onChange(); } }}
  />
);

interface Props {
  // "select" — checkbox multi-select for building the inference batch (Infer tab)
  // "view"   — click a file to load its results (Workspace tab)
  mode: 'select' | 'view';
  // Whether this sidebar's tab is the one currently showing. A fresh
  // fetch fires each time this flips true — i.e. every time the user
  // navigates to the Infer or Result tab, not just on first mount.
  active: boolean;
  activeFileId?: number | null;
  onFileClick?: (fileId: number, fileName: string) => void;
}

const PAGE_SIZES = [25, 50, 75, 100];

const FileSidebar: React.FC<Props> = ({ mode, active, activeFileId = null, onFileClick }) => {
  const { t } = useTranslation();
  const dispatch = useAppDispatch();
  const selectedServerIds = useAppSelector(s => s.upload.selectedServerIds);
  const isBatchRunning = useAppSelector(s => s.upload.isBatchRunning);
  const lastFileCompletedAt = useAppSelector(s => s.upload.lastFileCompletedAt);

  // ── Independent local file list ──
  const [files, setFiles] = useState<ServerFile[]>([]);
  const [total, setTotal] = useState(0);
  const [totalPages, setTotalPages] = useState(1);
  const [page, setPage] = useState(1);
  const [pageSize, setPageSize] = useState(50);
  const [sortBy, setSortBy] = useState<FilesSortBy>('id');
  const [sortOrder, setSortOrder] = useState<SortOrder>('desc');
  const [search, setSearch] = useState('');
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const [dateFrom, setDateFrom] = useState(defaultFrom());
  const [dateTo, setDateTo] = useState(defaultTo());
  const [applying, setApplying] = useState(false);
  const [statusFilter, setStatusFilter] = useState<FileStatus | ''>('');

  const fetchFiles = useCallback(async (opts?: {
    page?: number; pageSize?: number; from?: string; to?: string;
    sortBy?: FilesSortBy; sortOrder?: SortOrder; search?: string;
    statusFilter?: FileStatus | '';
  }) => {
    const p = opts?.page ?? page;
    const ps = opts?.pageSize ?? pageSize;
    const from = opts?.from ?? dateFrom;
    const to = opts?.to ?? dateTo;
    const sBy = opts?.sortBy ?? sortBy;
    const sOrd = opts?.sortOrder ?? sortOrder;
    const q = opts?.search ?? search;
    const statusF = opts?.statusFilter ?? statusFilter;
    setLoading(true);
    setError(null);
    try {
      const res = await api.post('/files/by-date/', {
        start_date: from,
        end_date: to,
        page: p,
        page_size: ps,
        sort_by: sBy,
        sort_order: sOrd,
        ...(q ? { search: q } : {}),
        ...(statusF ? { status_filter: statusF } : {}),
      });
      const d = (res.data as any)?.data ?? {};
      setFiles(d.data ?? []);
      setTotal(d.total ?? 0);
      setPage(d.page ?? p);
      setPageSize(d.page_size ?? ps);
      setTotalPages(d.total_pages ?? 1);
      setSortBy(sBy);
      setSortOrder(sOrd);
      setStatusFilter(statusF);
      // select mode = Infer tab: this is the only live source of full file
      // objects (prompt defaults included) while that tab is active, since
      // the Upload tab's shared `serverFiles` gets cleared as soon as it's
      // not the active tab. view mode (Workspace tab) doesn't publish —
      // InferencePanel only needs this for its own selection.
      if (mode === 'select') dispatch(selectFilesSuccess(d.data ?? []));
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to load files.');
    } finally {
      setLoading(false);
    }
  }, [page, pageSize, dateFrom, dateTo, sortBy, sortOrder, search, statusFilter, mode, dispatch]);

  const handleApply = useCallback(async () => {
    setApplying(true);
    await fetchFiles({ page: 1 });
    setApplying(false);
  }, [fetchFiles]);

  const handleStatusFilterChange = (value: FileStatus | '') => {
    fetchFiles({ page: 1, statusFilter: value });
  };

  const goToPage = (p: number) => {
    if (p < 1 || p > totalPages || p === page || loading) return;
    fetchFiles({ page: p });
  };

  const handlePageSizeChange = (size: number) => {
    fetchFiles({ page: 1, pageSize: size });
  };

  const handleSort = useCallback((key: SortKey) => {
    const apiKey = SORT_KEY_TO_API[key];
    const dir: SortOrder = sortBy === apiKey ? (sortOrder === 'asc' ? 'desc' : 'asc') : 'asc';
    fetchFiles({ sortBy: apiKey, sortOrder: dir, page: 1 });
  }, [sortBy, sortOrder, fetchFiles]);
  const sort = { key: API_TO_SORT_KEY[sortBy], dir: sortOrder };

  // ── Search — server-side, debounced (matches the Upload tab's search) ──
  const [searchQuery, setSearchQuery] = useState('');
  const skipNextSearchEffect = useRef(true);
  useEffect(() => {
    if (skipNextSearchEffect.current) { skipNextSearchEffect.current = false; return; }
    const q = searchQuery.trim();
    const handle = setTimeout(() => {
      setSearch(q);
      fetchFiles({ search: q, page: 1 });
    }, 400);
    return () => clearTimeout(handle);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [searchQuery]);

  // Own API call, fired every time this tab becomes active — entirely
  // separate from whatever the Upload tab is doing. When the tab goes
  // inactive (navigated away from), clear the list and reset search/
  // sort/filter/page back to defaults so a return visit shows a clean
  // loading state and a genuinely fresh fetch, instead of the previous
  // tab's stale results flashing before the new data arrives.
  useEffect(() => {
    if (active) {
      fetchFiles({ page: 1 });
    } else {
      skipNextSearchEffect.current = true; // don't let the reset below trigger a second debounced fetch
      setFiles([]);
      if (mode === 'select') dispatch(selectFilesSuccess([]));
      setTotal(0);
      setTotalPages(1);
      setPage(1);
      setSortBy('id');
      setSortOrder('desc');
      setSearch('');
      setSearchQuery('');
      setStatusFilter('');
      setError(null);
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [active]);

  // Re-fetch /files/by-date/ whenever an individual file finishes inferring
  // (InferencePanel stamps this the moment a file lands in the completed
  // bucket), so this sidebar's statuses stay current without waiting for
  // the whole batch to drain. Only relevant while the tab is actually
  // active — skip the initial mount (timestamp starts as null).
  useEffect(() => {
    if (active && lastFileCompletedAt !== null) {
      fetchFiles();
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [lastFileCompletedAt]);

  const pageSelectedCount = files.filter(f => selectedServerIds.includes(f.id)).length;
  const allSelected = files.length > 0 && pageSelectedCount === files.length;
  const someSelected = pageSelectedCount > 0 && !allSelected;

  const handleRowClick = (fileId: number, fileName: string) => {
    if (mode === 'select') {
      if (isBatchRunning) return;
      dispatch(toggleServerFileSelection(fileId));
    } else {
      onFileClick?.(fileId, fileName);
    }
  };

  return (
    <div className={styles.sidebar} data-tour={mode === 'select' ? 'infer-sidebar' : 'results-sidebar'}>
      <div className={styles.dateFilter} data-tour={mode === 'select' ? 'infer-date-filter' : 'results-date-filter'}>
        <div className={styles.dateField}>
          <label className={styles.dateLabel}>{t('uploadInfer.filePanel.dateFrom')}</label>
          <input
            type="date" className={styles.dateInput} value={dateFrom} max={dateTo}
            disabled={applying}
            onChange={e => setDateFrom(e.target.value)}
          />
        </div>
        <div className={styles.dateSep}>—</div>
        <div className={styles.dateField}>
          <label className={styles.dateLabel}>{t('uploadInfer.filePanel.dateTo')}</label>
          <input
            type="date" className={styles.dateInput} value={dateTo} min={dateFrom}
            disabled={applying}
            onChange={e => setDateTo(e.target.value)}
          />
        </div>
        <button className={styles.applyBtn} onClick={handleApply} disabled={applying || loading}>
          {applying ? <span className={styles.miniSpinner} /> : t('uploadInfer.filePanel.applyDate')}
        </button>
      </div>

      <div className={styles.filterSearchRow}>
        <div className={styles.statusFilterWrap} data-tour={mode === 'select' ? 'infer-status-filter' : 'results-status-filter'}>
          <svg className={styles.statusFilterIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
            <path d="M2 4h12M4.5 8h7M7 12h2" />
          </svg>
          <select
            className={styles.statusFilterSelect}
            value={statusFilter}
            disabled={loading}
            onChange={e => handleStatusFilterChange(e.target.value as FileStatus | '')}
          >
            {STATUS_FILTER_OPTIONS.map(opt => (
              <option key={opt.value || 'all'} value={opt.value}>{t(opt.labelKey)}</option>
            ))}
          </select>
        </div>

        <div className={styles.searchWrap} data-tour={mode === 'select' ? 'infer-search' : 'results-search'}>
          <svg className={styles.searchIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
            <circle cx="6.5" cy="6.5" r="4" /><path d="M11 11l2.5 2.5" />
          </svg>
          <input
            type="text"
            className={styles.searchInput}
            placeholder={t('uploadInfer.workspace.searchFiles')}
            value={searchQuery}
            onChange={e => setSearchQuery(e.target.value)}
          />
        </div>
      </div>

      <div className={styles.head}>
        {mode === 'select' && (
          <Checkbox
            checked={allSelected}
            indeterminate={someSelected}
            onChange={() => dispatch(toggleSelectAllServerFiles(files.map(f => f.id)))}
            disabled={files.length === 0 || isBatchRunning}
          />
        )}
        <span className={styles.headTitle}>{t('uploadInfer.workspace.filesTitle')}</span>
        {total > 0 && <span className={styles.headCount}>{total}</span>}
        {loading && <span className={styles.headSpinner} aria-label="Loading" />}
        {mode === 'select' && selectedServerIds.length > 0 && (
          <span className={styles.headSelected}>
            {t('uploadInfer.filePanel.selectedTotal', { count: selectedServerIds.length, total })}
          </span>
        )}
      </div>

      {/* Sort header — same sort keys as the Upload tab */}
      <div className={styles.sortHeader} data-tour={mode === 'select' ? 'infer-sort' : 'results-sort'}>
        <span className={styles.sortHeaderLabel}>
          <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
            <path d="M2 4h10M4 7h6M6 10h2" />
          </svg>
          {t('uploadInfer.filePanel.sortBy')}
        </span>
        <div className={styles.sortCols}>
          {([
            { key: 'id' as SortKey, label: t('uploadInfer.filePanel.sortId') },
            { key: 'name' as SortKey, label: t('uploadInfer.filePanel.sortName') },
            { key: 'date' as SortKey, label: t('uploadInfer.filePanel.sortDate') },
            { key: 'status' as SortKey, label: t('uploadInfer.filePanel.sortStatus') },
          ]).map(({ key, label }) => (
            <button
              key={key}
              className={`${styles.sortCol} ${sort.key === key ? styles.sortColActive : ''}`}
              onClick={() => handleSort(key)}
              disabled={loading}
            >
              {label}
              <svg viewBox="0 0 10 12" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round"
                className={sort.key === key ? (sort.dir === 'asc' ? styles.sortAsc : styles.sortDesc) : styles.sortInactive}>
                <path d="M5 1v10M2 8l3 3 3-3" />
              </svg>
            </button>
          ))}
        </div>
      </div>

      <div className={styles.rowsWrap}>
        <div className={styles.rows}>
        {error && (
          <div className={styles.empty}>{error}</div>
        )}
        {!error && loading && files.length === 0 && (
          <div className={styles.emptyLoading}>
            <span className={styles.rowsSpinner} />
            {t('uploadInfer.workspace.loadingFiles')}
          </div>
        )}
        {!error && !loading && files.length === 0 && (
          <div className={styles.empty}>
            {search ? t('uploadInfer.workspace.noFilesMatch') : t('uploadInfer.workspace.noFilesYet')}
          </div>
        )}
        {files.map(f => {
          const ext = getExt(f.original_name);
          const isChecked = mode === 'select' && selectedServerIds.includes(f.id);
          const isActiveView = mode === 'view' && f.id === activeFileId;
          const disabled = mode === 'select' && isBatchRunning;
          const statusMeta = STATUS_META[f.status] ?? STATUS_META.tbd;
          return (
            <div
              key={f.id}
              role="button"
              tabIndex={0}
              className={`${styles.hitm} ${styles.hitmSelectable}
                ${isChecked ? styles.active : ''}
                ${isActiveView ? styles.activeView : ''}
                ${disabled ? styles.rowDisabled : ''}`}
              onClick={() => !disabled && handleRowClick(f.id, f.original_name)}
              onKeyDown={e => { if (!disabled && (e.key === 'Enter' || e.key === ' ')) { e.preventDefault(); handleRowClick(f.id, f.original_name); } }}
            >
              {mode === 'select' && (
                <Checkbox checked={isChecked} onChange={() => handleRowClick(f.id, f.original_name)} disabled={isBatchRunning} />
              )}
              <div className={`${styles.ficon} ${styles[ext]}`}>{ext.toUpperCase()}</div>
              <div className={styles.hi}>
                <div className={styles.hn}>{f.original_name}</div>
                <div className={styles.hm}>{f.inserted_at} · #{f.id}</div>
              </div>
              <span className={`${styles.badge} ${isChecked ? styles.bSelected : styles[statusMeta.cls]}`}>
                {isChecked ? (
                  <svg viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" width="10" height="10">
                    <path d="M2 6l3 3 5-5" />
                  </svg>
                ) : (
                  <>
                    {f.status === 'completed' && (
                      <svg viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" width="9" height="9">
                        <path d="M2 6l3 3 5-5" />
                      </svg>
                    )}
                    {(f.status === 'running' || f.status === 'queued') && (
                      <span className={styles.badgeDot} />
                    )}
                    {t(statusMeta.labelKey)}
                  </>
                )}
              </span>
            </div>
          );
        })}
        </div>

        {/* Blocks interaction with the current list while a fetch is in
            flight (pagination, sort, search, filter) — only shown when
            there's already content on screen; the empty-state spinner
            above covers the first load. */}
        {loading && files.length > 0 && (
          <div className={styles.rowsLoadingOverlay}>
            <span className={styles.rowsSpinner} />
          </div>
        )}
      </div>

      {/* Pagination footer — same page-number layout as the Upload tab */}
      {!loading && !error && total > 0 && (
        <div className={styles.paginationBar}>
          <div className={styles.paginationTopRow}>
            <div className={styles.pageSizeGroup}>
              <span className={styles.pageSizeLabel}>{t('uploadInfer.filePanel.perPage')}</span>
              <select
                className={styles.pageSizeSelect}
                value={pageSize}
                onChange={e => handlePageSizeChange(Number(e.target.value))}
              >
                {PAGE_SIZES.map(size => (
                  <option key={size} value={size}>{size}</option>
                ))}
              </select>
            </div>

            <div className={styles.pageNav}>
              <button
                className={styles.pageNavBtn}
                onClick={() => goToPage(page - 1)}
                disabled={page <= 1}
                aria-label="Previous page"
              >
                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                  <path d="M10 3L6 8l4 5" />
                </svg>
              </button>

              {(() => {
                const pageWindow = getPageNumbers(page, totalPages);
                const showFirst = pageWindow[0] > 1;
                const showLast = pageWindow[pageWindow.length - 1] < totalPages;
                return (
                  <>
                    {showFirst && (
                      <>
                        <button className={styles.pageNumBtn} onClick={() => goToPage(1)}>1</button>
                        {pageWindow[0] > 2 && <span className={styles.pageEllipsis}>…</span>}
                      </>
                    )}
                    {pageWindow.map(p => (
                      <button
                        key={p}
                        className={`${styles.pageNumBtn} ${p === page ? styles.pageNumBtnActive : ''}`}
                        onClick={() => goToPage(p)}
                      >
                        {p}
                      </button>
                    ))}
                    {showLast && (
                      <>
                        {pageWindow[pageWindow.length - 1] < totalPages - 1 && <span className={styles.pageEllipsis}>…</span>}
                        <button className={styles.pageNumBtn} onClick={() => goToPage(totalPages)}>{totalPages}</button>
                      </>
                    )}
                  </>
                );
              })()}

              <button
                className={styles.pageNavBtn}
                onClick={() => goToPage(page + 1)}
                disabled={page >= totalPages}
                aria-label="Next page"
              >
                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                  <path d="M6 3l4 5-4 5" />
                </svg>
              </button>
            </div>
          </div>

          <div className={styles.pageInfo}>
            {t('uploadInfer.filePanel.pageInfo', { page, totalPages, total })}
          </div>
        </div>
      )}
    </div>
  );
};

export default FileSidebar;
