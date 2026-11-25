//src/step-motion.css
/* Moving between steps (Generate and Results).
   The new step glides in from the side you moved towards: a soft blur clears, it settles from a
   slightly smaller scale, and its parts follow in a short cascade. Back plays the mirror image.
   With reduced motion it becomes a quick, plain fade. */
:root { --step-ease: cubic-bezier(.22, 1, .36, 1); --step-dx: 26px; }

@keyframes step-in-fwd  { from { opacity: 0; transform: translateX(var(--step-dx)) scale(.985); filter: blur(6px); } to { opacity: 1; transform: none; filter: none; } }
@keyframes step-in-back { from { opacity: 0; transform: translateX(calc(var(--step-dx) * -1)) scale(.985); filter: blur(6px); } to { opacity: 1; transform: none; filter: none; } }
@keyframes step-part-fwd  { from { opacity: 0; transform: translate(14px, 8px); filter: blur(3px); } to { opacity: 1; transform: none; filter: none; } }
@keyframes step-part-back { from { opacity: 0; transform: translate(-14px, 8px); filter: blur(3px); } to { opacity: 1; transform: none; filter: none; } }
@keyframes step-ico { 0% { opacity: 0; transform: scale(.7) rotate(-8deg); } 60% { opacity: 1; transform: scale(1.06) rotate(2deg); } 100% { transform: none; } }
@keyframes step-sheen-fwd  { 0% { opacity: 0; transform: scaleX(0); transform-origin: left; } 30% { opacity: 1; } 100% { opacity: 0; transform: scaleX(1); transform-origin: left; } }
@keyframes step-sheen-back { 0% { opacity: 0; transform: scaleX(0); transform-origin: right; } 30% { opacity: 1; } 100% { opacity: 0; transform: scaleX(1); transform-origin: right; } }
@keyframes step-ring { 0% { box-shadow: 0 0 0 0 color-mix(in srgb, var(--accent) 45%, transparent); } 100% { box-shadow: 0 0 0 9px color-mix(in srgb, var(--accent) 0%, transparent); } }

/* header: the icon settles, then eyebrow / title / lead follow in turn */
.gen .gen-hd.anim-fwd, .gen .gen-hd.anim-back { position: relative; animation: none; }
.gen .gen-hd.anim-fwd > *, .gen .gen-hd.anim-back > * { animation: none; }
.gen .gen-hd.anim-fwd > .gen-hd-ico, .gen .gen-hd.anim-back > .gen-hd-ico { animation: step-ico .6s var(--step-ease) both; }
.gen .gen-hd.anim-fwd > div > *  { animation: step-in-fwd  .6s var(--step-ease) both; }
.gen .gen-hd.anim-back > div > * { animation: step-in-back .6s var(--step-ease) both; }
.gen .gen-hd > div > :nth-child(1) { animation-delay: .04s; }
.gen .gen-hd > div > :nth-child(2) { animation-delay: .09s; }
.gen .gen-hd > div > :nth-child(3) { animation-delay: .14s; }
/* a thin coral sheen runs along the header's lower edge, in the direction you moved */
.gen .gen-hd.anim-fwd::after, .gen .gen-hd.anim-back::after { content: ""; position: absolute; left: 0; right: 0; bottom: -1px; height: 2px;
  background: var(--grad); pointer-events: none; opacity: 0; }
.gen .gen-hd.anim-fwd::after  { animation: step-sheen-fwd  .9s var(--step-ease) .05s both; }
.gen .gen-hd.anim-back::after { animation: step-sheen-back .9s var(--step-ease) .05s both; }

/* body: each section follows the header in a soft cascade */
.gen .gen-bd.anim-fwd > *  { animation: step-part-fwd  .6s var(--step-ease) both; }
.gen .gen-bd.anim-back > * { animation: step-part-back .6s var(--step-ease) both; }
.gen .gen-bd.anim-fwd > :nth-child(1), .gen .gen-bd.anim-back > :nth-child(1) { animation-delay: .12s; }
.gen .gen-bd.anim-fwd > :nth-child(2), .gen .gen-bd.anim-back > :nth-child(2) { animation-delay: .19s; }
.gen .gen-bd.anim-fwd > :nth-child(3), .gen .gen-bd.anim-back > :nth-child(3) { animation-delay: .26s; }
.gen .gen-bd.anim-fwd > :nth-child(4), .gen .gen-bd.anim-back > :nth-child(4) { animation-delay: .33s; }
.gen .gen-bd.anim-fwd > :nth-child(n+5), .gen .gen-bd.anim-back > :nth-child(n+5) { animation-delay: .38s; }

/* stepper: the card and the progress line glide on the same curve; the new step's number gives one soft ring */
.gen .stp-ind { transition: transform .55s var(--step-ease), width .55s var(--step-ease), height .55s var(--step-ease); }
.gen .stp-progress i { position: relative; transition: width .6s var(--step-ease); }
.gen .stp-progress i::after { content: ""; position: absolute; right: 0; top: 0; bottom: 0; width: 28px; border-radius: inherit; pointer-events: none;
  background: linear-gradient(90deg, transparent, rgb(255 255 255 / .6)); box-shadow: 0 0 10px 1px color-mix(in srgb, var(--accent) 45%, transparent); }
.gen .stp.on .stp-n { animation: step-ring .7s ease-out .15s 1 both; }

@media (prefers-reduced-motion: reduce) {
  .gen .gen-hd > *, .gen .gen-hd > div > *, .gen .gen-bd > * { animation: fade-in .2s ease both !important; animation-delay: 0s !important; transform: none !important; filter: none !important; }
  .gen .gen-hd::after, .gen .stp.on .stp-n { animation: none !important; }
}


















//src/stepper.js
// Measures the active step of a stepper so the soft card behind it and the coral progress line
// always line up with the real button, on any screen size, zoom level, sidebar state or font.
import { useLayoutEffect, useRef, useState } from "react";
import "./step-motion.css";

const same = (a, b) => a === b || (a && b && a.left === b.left && a.top === b.top && a.width === b.width && a.height === b.height && a.end === b.end);

export function useStepper() {
  const ref = useRef(null);
  const [ind, setInd] = useState(null);
  const placeRef = useRef(() => {});
  placeRef.current = () => {
    const nav = ref.current;
    const el = nav?.querySelector(".stp.on");
    if (!nav || !el) { setInd(null); return; }
    // positions relative to the stepper itself, so they hold under CSS zoom too
    const nb = nav.getBoundingClientRect(), eb = el.getBoundingClientRect();
    const k = nav.offsetWidth ? nb.width / nav.offsetWidth : 1; // rendered px per CSS px
    const left = (eb.left - nb.left) / k - nav.clientLeft, top = (eb.top - nb.top) / k - nav.clientTop;
    const width = eb.width / k, height = eb.height / k;
    const n = { left: Math.round(left * 10) / 10, top: Math.round(top * 10) / 10, width: Math.round(width * 10) / 10, height: Math.round(height * 10) / 10,
      end: Math.round((left + width) * 10) / 10 };
    setInd((c) => (same(c, n) ? c : n));
  };
  const roRef = useRef(null), seen = useRef(null);
  // re-measure after every render (step changed, labels changed), and watch the stepper once it is on screen
  useLayoutEffect(() => {
    placeRef.current();
    const nav = ref.current, ro = roRef.current;
    if (ro && nav !== seen.current) {
      ro.disconnect(); seen.current = nav;
      if (nav) { ro.observe(nav); nav.querySelectorAll(".stp").forEach((x) => ro.observe(x)); }
    }
  });
  // ... and whenever the layout moves: window resize, stepper or step size changes, fonts finishing loading
  useLayoutEffect(() => {
    const place = () => placeRef.current();
    let raf = 0;
    const soon = () => { cancelAnimationFrame(raf); raf = requestAnimationFrame(place); };
    window.addEventListener("resize", soon);
    const ro = typeof ResizeObserver === "function" ? new ResizeObserver(soon) : null;
    roRef.current = ro; seen.current = null;
    if (ro && ref.current) { seen.current = ref.current; ro.observe(ref.current); ref.current.querySelectorAll(".stp").forEach((x) => ro.observe(x)); }
    document.fonts?.ready?.then(soon);
    document.fonts?.addEventListener?.("loadingdone", soon);
    // the sidebar animates its width; measure again once that settles
    const onEnd = (e) => { if (e.propertyName === "grid-template-columns" || e.propertyName === "width") soon(); };
    document.addEventListener("transitionend", onEnd);
    return () => {
      cancelAnimationFrame(raf); window.removeEventListener("resize", soon); ro?.disconnect(); roRef.current = null;
      document.fonts?.removeEventListener?.("loadingdone", soon); document.removeEventListener("transitionend", onEnd);
    };
  }, []);
  return [ref, ind];
}
