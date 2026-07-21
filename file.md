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
// ── Header — title and tab bar share one row ──────
.headerBar {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 20px;
  padding: 16px 24px;
  background: var(--bg1);
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
  flex-shrink: 0;
}
.phTitle {
  flex-shrink: 0;
  font-size: 18px;
  font-weight: 600;
  color: var(--t0);
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

// ── Tab bar — plain segmented buttons, sized to feel substantial without
// being oversized. ──
.tabbar {
  display: flex;
  align-items: stretch;
  gap: 8px;
  flex-shrink: 0;
}
.tabBtn {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  justify-content: center;
  gap: 3px;
  min-width: 0;
  height: 54px;
  padding: 0 20px;
  border-radius: var(--rl);
  border: 1px solid var(--bdr2);
  background: var(--bg1);
  color: var(--t1);
  cursor: pointer;
  text-align: left;
  font-family: var(--font-ui);
  // Note: intentionally not using the shared theme-transition mixin here
  // — it's tuned for light/dark theme swaps and was causing a visible
  // background flash/crossfade on ordinary tab clicks. This explicit,
  // fast transition covers the click-driven active/inactive state change.
  transition: background 0.14s ease, border-color 0.14s ease, color 0.14s ease, box-shadow 0.14s ease;

  &:hover {
    border-color: var(--bdr3);
    background: var(--bg2);
    color: var(--t0);
  }

  &:active {
    transform: scale(0.98);
  }
}
.tabBtnActive {
  border-color: var(--blue);
  background: var(--blue);
  color: #fff;
  box-shadow: 0 3px 10px var(--blue-dim);

  &:hover {
    border-color: var(--blue);
    background: var(--blue);
    color: #fff;
  }
}
.tabLabel {
  font-size: 14px;
  font-weight: 600;
  letter-spacing: 0.01em;
  white-space: nowrap;
}
// Always visible — this is what tells the user what they'll find on this
// tab, without needing to click it first or read a separate banner.
.tabDesc {
  font-size: 10.5px;
  color: var(--t2);
  opacity: 0.8;
  white-space: nowrap;
}
.tabBtnActive .tabDesc {
  color: rgba(255, 255, 255, 0.85);
  opacity: 1;
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
  .headerBar { flex-direction: column; align-items: stretch; gap: 12px; padding: 14px 16px; }
  .tabbar { flex-wrap: wrap; }
  .tabBtn { flex: 1; width: auto; min-width: 130px; }
  .tabPane { flex-direction: column; }
}
