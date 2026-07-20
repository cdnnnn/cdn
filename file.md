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

// ── Header — title row on top, tab bar below (mirrors STT Pipeline) ──
.headerBar {
  flex-shrink: 0;
  padding: 16px 24px 0;
  background: var(--bg1);
  border-bottom: 1px solid var(--bdr);
}

.phTitleRow {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 14px;
}

.phTitle {
  flex-shrink: 0;
  font-size: 18px;
  font-weight: 600;
  color: var(--t0);
  font-family: var(--font-display);
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

  svg { width: 13px; height: 13px; }

  &:hover {
    background: var(--bg3);
    color: var(--t1);
    border-color: var(--bdr3);
  }
}

// ── Tab bar — flat, underline style (mirrors STT Pipeline's .sttTabbar) ──
.tabbar {
  display: flex;
  gap: 4px;
}

.tabBtn {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 2px;
  padding: 10px 18px;
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
  .headerBar { padding: 14px 16px 0; }
  .phTitleRow { flex-wrap: wrap; gap: 8px; margin-bottom: 10px; }
  .tabbar { flex-wrap: wrap; }
  .tabPane { flex-direction: column; }
}
