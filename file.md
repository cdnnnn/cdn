// ═══════════════════════════════════════════════
// TourGuide.module.scss
// Content Analytics · Guided tour overlay
// ═══════════════════════════════════════════════
@use '../../styles/mixins' as m;

.tourRoot {
  position: fixed;
  inset: 0;
  z-index: 1100;
  pointer-events: none;
}

// The "hole" is created by a huge box-shadow spread around a
// transparent rect exactly matching the target — everything outside
// that rect gets dimmed, the target itself stays fully visible.
.spotlight {
  position: fixed;
  border-radius: 10px;
  box-shadow: 0 0 0 9999px rgba(0, 0, 0, 0.6);
  outline: 2px solid var(--blue);
  outline-offset: 2px;
  transition: top 0.25s ease, left 0.25s ease, width 0.25s ease, height 0.25s ease;
  pointer-events: none;
}

// Fallback full dim if a step's target can't be found in the DOM
.dimFallback {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.6);
}

.tooltip {
  position: fixed;
  z-index: 1101;
  pointer-events: auto;
  background: var(--bg1);
  border: 1px solid var(--bdr2);
  border-radius: var(--rl);
  box-shadow: var(--shadow);
  padding: 16px;
  transition: top 0.25s ease, left 0.25s ease;
}

.tooltipStep {
  font-size: 10.5px;
  font-weight: 600;
  color: var(--t2);
  letter-spacing: 0.06em;
  text-transform: uppercase;
  @include m.mono;
  margin-bottom: 6px;
}

.tooltipTitle {
  font-size: 14.5px;
  font-weight: 600;
  color: var(--t0);
  margin-bottom: 6px;
}

.tooltipBody {
  font-size: 13px;
  line-height: 1.5;
  color: var(--t1);
  margin-bottom: 14px;
}

.tooltipFooter {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
}

.skipBtn {
  padding: 0;
  border: none;
  background: transparent;
  color: var(--t2);
  font-size: 12px;
  cursor: pointer;

  &:hover {
    color: var(--t1);
    text-decoration: underline;
  }
}

.navBtns {
  display: flex;
  align-items: center;
  gap: 8px;
}

.backBtn {
  height: 30px;
  padding: 0 12px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t1);
  font-size: 12.5px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.12s;

  &:hover {
    background: var(--bg3);
  }
}

.nextBtn {
  height: 30px;
  padding: 0 14px;
  border-radius: var(--r);
  border: 1px solid var(--blue);
  background: var(--blue-dim);
  color: var(--blue);
  font-size: 12.5px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.12s;

  &:hover {
    background: var(--blue);
    color: #fff;
  }
}

@media (max-width: 560px) {
  .tooltip {
    width: calc(100vw - 24px) !important;
    left: 12px !important;
    right: auto !important;
  }
}
