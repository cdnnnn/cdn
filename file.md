// ═══════════════════════════════════════════════
// pages/UploadInfer/TourGuide.tsx
// Content Analytics · Lightweight guided product tour
// No external dependency — spotlight overlay + positioned tooltip,
// driven by `data-tour="<id>"` attributes placed on real UI elements.
// ═══════════════════════════════════════════════
import React, { useEffect, useLayoutEffect, useRef, useState } from 'react';
import styles from './TourGuide.module.scss';

export interface TourStep {
  target: string;              // matches an element's data-tour="<target>"
  title: string;
  content: string;
  placement?: 'top' | 'bottom' | 'left' | 'right';
  // Fired right before this step tries to measure its target — use this
  // to switch tabs / open panels so the target actually exists in the DOM.
  onEnter?: () => void;
}

interface TourGuideProps {
  steps: TourStep[];
  active: boolean;
  onFinish: () => void;
}

interface Rect { top: number; left: number; width: number; height: number; }

const PAD = 6; // spotlight padding around the target's real bounding box

const TourGuide: React.FC<TourGuideProps> = ({ steps, active, onFinish }) => {
  const [index, setIndex] = useState(0);
  const [rect, setRect] = useState<Rect | null>(null);
  const [ready, setReady] = useState(false);
  // Real tooltip size, measured after each render — starts with a rough
  // estimate for the very first paint, then locks to the actual size.
  const [size, setSize] = useState({ w: 320, h: 190 });
  const tooltipRef = useRef<HTMLDivElement>(null);

  const step = steps[index];
  const isLast = index === steps.length - 1;

  useEffect(() => {
    if (active) { setIndex(0); setReady(false); }
  }, [active]);

  // Measure the current step's target — runs the step's onEnter first
  // (e.g. switch tabs), then waits a tick for that to paint, then locates
  // and scrolls to the element before reading its rect.
  useLayoutEffect(() => {
    if (!active || !step) return;
    setReady(false);
    step.onEnter?.();

    let cancelled = false;
    const measure = () => {
      if (cancelled) return;
      const el = document.querySelector<HTMLElement>(`[data-tour="${step.target}"]`);
      if (!el) { setRect(null); setReady(true); return; }
      el.scrollIntoView({ block: 'nearest', inline: 'nearest' });
      requestAnimationFrame(() => {
        if (cancelled) return;
        const r = el.getBoundingClientRect();
        setRect({ top: r.top, left: r.left, width: r.width, height: r.height });
        setReady(true);
      });
    };
    // Two frames: one for onEnter's state update (e.g. tab switch) to
    // render, one more for layout to settle before measuring.
    const raf1 = requestAnimationFrame(() => requestAnimationFrame(measure));

    const onResize = () => measure();
    window.addEventListener('resize', onResize);
    window.addEventListener('scroll', onResize, true);
    return () => {
      cancelled = true;
      cancelAnimationFrame(raf1);
      window.removeEventListener('resize', onResize);
      window.removeEventListener('scroll', onResize, true);
    };
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [active, index]);

  // Lock in the tooltip's real rendered size — runs after every render
  // (cheap: only updates state, and thus re-renders, on an actual change).
  useLayoutEffect(() => {
    const el = tooltipRef.current;
    if (!el) return;
    const w = el.offsetWidth, h = el.offsetHeight;
    if (w && h && (w !== size.w || h !== size.h)) setSize({ w, h });
  });

  useEffect(() => {
    if (!active) return;
    const onKey = (e: KeyboardEvent) => {
      if (e.key === 'Escape') onFinish();
      if (e.key === 'ArrowRight') go(1);
      if (e.key === 'ArrowLeft') go(-1);
    };
    window.addEventListener('keydown', onKey);
    return () => window.removeEventListener('keydown', onKey);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [active, index]);

  if (!active || !step) return null;

  const go = (delta: number) => {
    const next = index + delta;
    if (next < 0) return;
    if (next >= steps.length) { onFinish(); return; }
    setIndex(next);
  };

  // ── Placement: pick whichever side actually has room, treating the
  // step's preferred side as a preference rather than a guarantee. This
  // is what stops a "top" or "right" placement from clipping off-screen
  // when its target sits near a viewport edge. ──
  const spotRect = rect ?? { top: -9999, left: -9999, width: 0, height: 0 };
  const vw = typeof window !== 'undefined' ? window.innerWidth : 1200;
  const vh = typeof window !== 'undefined' ? window.innerHeight : 800;
  const EDGE = 12;
  const GAP = PAD + 12;

  const spaceBelow = vh - (spotRect.top + spotRect.height);
  const spaceAbove = spotRect.top;
  const spaceRight = vw - (spotRect.left + spotRect.width);
  const spaceLeft = spotRect.left;
  const fits = {
    bottom: spaceBelow >= size.h + GAP + EDGE,
    top: spaceAbove >= size.h + GAP + EDGE,
    right: spaceRight >= size.w + GAP + EDGE,
    left: spaceLeft >= size.w + GAP + EDGE,
  };
  const preferred = step.placement;
  const placement: 'top' | 'bottom' | 'left' | 'right' =
    (preferred && fits[preferred]) ? preferred
      : fits.bottom ? 'bottom'
        : fits.top ? 'top'
          : fits.right ? 'right'
            : fits.left ? 'left'
              : 'bottom'; // nothing fits cleanly — fall back and let clamping do its best

  let top = 0, left = 0;
  if (placement === 'bottom') {
    top = spotRect.top + spotRect.height + GAP;
    left = spotRect.left + spotRect.width / 2 - size.w / 2;
  } else if (placement === 'top') {
    top = spotRect.top - GAP - size.h;
    left = spotRect.left + spotRect.width / 2 - size.w / 2;
  } else if (placement === 'left') {
    top = spotRect.top + spotRect.height / 2 - size.h / 2;
    left = spotRect.left - GAP - size.w;
  } else {
    top = spotRect.top + spotRect.height / 2 - size.h / 2;
    left = spotRect.left + spotRect.width + GAP;
  }
  // Final safety clamp regardless of placement — guarantees the tooltip
  // is always fully on-screen even in edge cases nothing above caught.
  top = Math.min(Math.max(top, EDGE), vh - size.h - EDGE);
  left = Math.min(Math.max(left, EDGE), vw - size.w - EDGE);

  return (
    <div className={styles.tourRoot} aria-live="polite">
      {/* Spotlight cutout — a box-shadow "hole" over the real target */}
      {rect && (
        <div
          className={styles.spotlight}
          style={{
            top: rect.top - PAD, left: rect.left - PAD,
            width: rect.width + PAD * 2, height: rect.height + PAD * 2,
          }}
        />
      )}
      {!rect && ready && <div className={styles.dimFallback} />}

      <div
        ref={tooltipRef}
        className={`${styles.tooltip} ${styles[`place_${placement}`]}`}
        style={{
          top, left,
          maxHeight: `calc(100vh - ${EDGE * 2}px)`,
          overflowY: 'auto',
          width: 320,
        }}
      >
        <div className={styles.tooltipStep}>{index + 1} / {steps.length}</div>
        <div className={styles.tooltipTitle}>{step.title}</div>
        <div className={styles.tooltipBody}>{step.content}</div>
        <div className={styles.tooltipFooter}>
          <button className={styles.skipBtn} onClick={onFinish}>Skip tour</button>
          <div className={styles.navBtns}>
            {index > 0 && (
              <button className={styles.backBtn} onClick={() => go(-1)}>Back</button>
            )}
            <button className={styles.nextBtn} onClick={() => go(1)}>
              {isLast ? 'Done' : 'Next'}
            </button>
          </div>
        </div>
      </div>
    </div>
  );
};

export default TourGuide;
