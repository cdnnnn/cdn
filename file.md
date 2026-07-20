// ═══════════════════════════════════════════════
// UploadInfer.module.scss
// LectureAI · Upload & Inference page shell + tab bar
// ═══════════════════════════════════════════════
@use '../../styles/mixins' as m;

.page {
  display: flex;
  flex-direction: column;
  height: 100%;
  overflow: hidden;
  background: var(--bg0);
}

// ── Header — title row and tab bar share one row, tabs pinned right
// (avoids the extra height a stacked layout would add) ──
.headerBar {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 14px;
  padding: 14px 22px 12px;
  background: var(--bg1);
  border-bottom: none;
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

.phTitleRow {
  display: flex;
  align-items: center;
  gap: 12px;
  min-width: 0;
  flex-shrink: 0;
}

.phTitle {
  font-size: 18px;
  font-weight: 600;
  color: var(--t0);
  letter-spacing: -0.3px;
  line-height: 1.2;
  font-family: var(--font-display);
  @include m.truncate;
}

.tourTriggerBtn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 4px 10px;
  border-radius: 99px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  font-family: var(--font-ui);
  font-size: 12px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.12s;
  white-space: nowrap;
  flex-shrink: 0;

  svg { width: 13px; height: 13px; }

  &:hover {
    background: var(--bg3);
    color: var(--t1);
    border-color: var(--bdr3);
  }
}

// ── Tab bar — flat, underline style (mirrors STT Pipeline's .sttTabbar),
// pinned to the right of the header row ──
.tabbar {
  display: flex;
  gap: 4px;
  flex-shrink: 0;
}

.tabBtn {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 2px;
  padding: 8px 16px;
  border: none;
  border-bottom: 2px solid transparent;
  background: transparent;
  cursor: pointer;
  transition: all 0.14s;

  &:hover { background: var(--bg2); }
}

.tabBtnActive {
  border-bottom-color: var(--blue);
  background: var(--bg2);
}

.tabLabel {
  font-size: 13px;
  font-weight: 600;
  color: var(--t0);
  font-family: var(--font-display);
}

// Always visible — this is what tells the user what they'll find on
// this tab, without needing to click it first or read a separate banner.
.tabDesc {
  font-size: 10.5px;
  color: var(--t2);
  @include m.mono;
  white-space: nowrap;
}

// ── Tab content ──────────────────────────────────
.upbody {
  flex: 1;
  min-height: 0;
  position: relative;
}

// Every panel is rendered inside a .tabPane at all times; only the active
// one is switched to `display: flex` (see UploadInfer.tsx). This keeps
// file lists, batch polling, and scroll position alive across tab
// switches instead of remounting on every click.
.tabPane {
  position: absolute;
  inset: 0;
  flex-direction: row;
  overflow: hidden;
}

@media (max-width: 860px) {
  .headerBar { flex-direction: column; align-items: stretch; gap: 10px; padding: 12px 16px 10px; }
  .tabbar { flex-wrap: wrap; }
  .tabPane { flex-direction: column; }
}
