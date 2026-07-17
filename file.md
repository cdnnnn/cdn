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
  const [correction, setCorrection] = useState({ dx: 0, dy: 0 });
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

  // Correction pass — after the tooltip actually paints at its estimated
  // position, check its real size and nudge it back on-screen if the
  // estimate was off (e.g. a longer translated string than expected).
  useLayoutEffect(() => {
    setCorrection({ dx: 0, dy: 0 });
    const el = tooltipRef.current;
    if (!el || !rect) return;
    const r = el.getBoundingClientRect();
    const EDGE = 12;
    const vw = window.innerWidth, vh = window.innerHeight;
    let dx = 0, dy = 0;
    if (r.top < EDGE) dy = EDGE - r.top;
    else if (r.bottom > vh - EDGE) dy = vh - EDGE - r.bottom;
    if (r.left < EDGE) dx = EDGE - r.left;
    else if (r.right > vw - EDGE) dx = vw - EDGE - r.right;
    if (dx !== 0 || dy !== 0) setCorrection({ dx, dy });
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [rect, index]);

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

  // Tooltip placement — default under the target, flipped above if there
  // isn't room, then clamped so it never runs off-screen horizontally.
  const spotRect = rect ?? { top: -9999, left: -9999, width: 0, height: 0 };
  const vw = typeof window !== 'undefined' ? window.innerWidth : 1200;
  const vh = typeof window !== 'undefined' ? window.innerHeight : 800;
  const tooltipW = 320;
  const EDGE = 12;
  const estH = 190; // rough estimate used before the real measurement pass below
  const placement = step.placement ?? (spotRect.top + spotRect.height + estH + 24 > vh ? 'top' : 'bottom');

  let tTop = 0, tLeft = 0;
  if (placement === 'bottom') {
    tTop = spotRect.top + spotRect.height + PAD + 12;
    tTop = Math.min(tTop, vh - estH - EDGE);
    tLeft = spotRect.left + spotRect.width / 2 - tooltipW / 2;
  } else if (placement === 'top') {
    tTop = spotRect.top - PAD - 12;
    tTop = Math.max(tTop, estH + EDGE);
    tLeft = spotRect.left + spotRect.width / 2 - tooltipW / 2;
  } else if (placement === 'left') {
    tTop = spotRect.top + spotRect.height / 2;
    tTop = Math.min(Math.max(tTop, estH / 2 + EDGE), vh - estH / 2 - EDGE);
    tLeft = spotRect.left - PAD - 12 - tooltipW;
  } else {
    tTop = spotRect.top + spotRect.height / 2;
    tTop = Math.min(Math.max(tTop, estH / 2 + EDGE), vh - estH / 2 - EDGE);
    tLeft = spotRect.left + spotRect.width + PAD + 12;
  }
  tTop = Math.max(tTop, EDGE);
  tLeft = Math.min(Math.max(tLeft, EDGE), vw - tooltipW - EDGE);
  const usesTransformY = placement === 'top';

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
          top: placement === 'top' || placement === 'bottom' ? tTop + correction.dy : undefined,
          left: placement === 'left' || placement === 'right' ? undefined : tLeft + correction.dx,
          right: placement === 'left' ? vw - tLeft - tooltipW - correction.dx : undefined,
          transform: usesTransformY ? 'translateY(-100%)' : (placement === 'left' || placement === 'right') ? `translateY(-50%) translateY(${correction.dy}px)` : undefined,
          maxHeight: `calc(100vh - ${EDGE * 2}px)`,
          overflowY: 'auto',
          width: tooltipW,
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
