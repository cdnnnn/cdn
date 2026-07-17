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

// ── Page header ──────────────────────────────────
.ph {
  flex-shrink: 0;
  padding: 18px 24px 0;
}

.phRow {
  @include m.flex-between;
}

.phTitle {
  font-size: 18px;
  font-weight: 600;
  color: var(--t0);
}

.phSub {
  margin-top: 2px;
  font-size: 13px;
  color: var(--t2);
}

// ── Tab bar — three connected step buttons, every tab always clickable ──
.tabbarWrap {
  flex-shrink: 0;
  padding: 20px 24px;
  display: flex;
  justify-content: center;
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

.tabbar {
  position: relative;
  display: flex;
  justify-content: space-between;
  width: 100%;
  max-width: 620px;
}

// Connecting line — sits behind the buttons; only visible in the gaps
// between them since each button has an opaque background.
.tabbarLine {
  position: absolute;
  top: 50%;
  left: 0;
  right: 0;
  height: 2px;
  background: var(--bdr2);
  transform: translateY(-50%);
  z-index: 0;
}

.tabBtn {
  position: relative;
  z-index: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 5px;
  width: 180px;
  height: 58px;
  padding: 0 14px;
  border-radius: 12px;
  border: 1px solid var(--bdr2);
  background: var(--bg1);
  cursor: pointer;
  @include m.theme-transition;

  &:hover {
    border-color: var(--bdr3);
    background: var(--bg2);
  }
}

.tabBtnActive {
  border-color: var(--blue);
  background-color: var(--bg1);
  background-image: linear-gradient(var(--blue-dim), var(--blue-dim));
  box-shadow: 0 0 0 3px var(--blue-dim);

  &:hover {
    border-color: var(--blue);
    background-color: var(--bg1);
    background-image: linear-gradient(var(--blue-dim), var(--blue-dim));
  }
}

.tabLabel {
  font-family: var(--font-ui);
  font-size: 13.5px;
  font-weight: 500;
  color: var(--t1);
  letter-spacing: 0.01em;
}

.tabBtnActive .tabLabel {
  font-weight: 600;
  color: var(--t0);
}

.tabBadge {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-size: 10px;
  font-weight: 600;
  color: var(--t2);
  background: var(--bg2);
  border-radius: 99px;
  padding: 1px 8px;
  line-height: 1.5;
}

.tabBtnActive .tabBadge {
  color: var(--blue);
  background: rgba(0, 0, 0, 0.12);
}

.tabHint {
  font-size: 10px;
  color: var(--t2);
}

.tabLiveDot {
  width: 4px;
  height: 4px;
  border-radius: 50%;
  background: currentColor;
  animation: tabLivePulse 1.4s ease-in-out infinite;
}

@keyframes tabLivePulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.35; }
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
  .ph { padding: 14px 16px 0; }
  .tabbarWrap { padding: 14px 16px; }
  .tabbar { flex-wrap: wrap; gap: 8px; max-width: none; }
  .tabbarLine { display: none; }
  .tabBtn { width: calc(50% - 4px); }

  .tabPane { flex-direction: column; }
}
