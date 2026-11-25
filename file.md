//logo.jsx
import "./logo.css";

/* SLM Studio logo, built in HTML/CSS.
   The mark is three stacked layers (raw documents -> model -> dataset) with a spark of generation above. */
export default function Logo({ size = 36, markOnly = false, animated = true }) {
  return (
    <span className={`slm-logo${animated ? " is-animated" : ""}`} style={{ "--s": `${size}px` }} role="img" aria-label="SLM Studio">
      <span className="slm-mark" aria-hidden="true">
        <span className="slm-halo" />
        <span className="slm-layer l3" />
        <span className="slm-layer l2" />
        <span className="slm-layer l1"><span className="slm-core" /></span>
        <span className="slm-spark" />
      </span>
      {!markOnly && <span className="slm-word" aria-hidden="true"><b>SLM</b><em>Studio</em></span>}
    </span>
  );
}










//logo.css
/* SLM Studio logo in HTML/CSS. --s is the logo height; --u is one unit of a 64-unit grid. */
.slm-logo { --u: calc(var(--s) / 64); display: inline-flex; align-items: center; gap: calc(var(--u) * 11); line-height: 1; text-decoration: none; user-select: none; }

/* ---- the mark: three stacked layers seen in isometric view */
.slm-mark { position: relative; flex: none; width: var(--s); height: var(--s); }
.slm-halo { position: absolute; inset: 8%; border-radius: 50%; background: radial-gradient(circle, rgba(244, 63, 94, .32), transparent 68%);
  filter: blur(calc(var(--u) * 4)); opacity: .7; }

.slm-layer { position: absolute; left: 50%; width: calc(var(--u) * 34); height: calc(var(--u) * 34); border-radius: calc(var(--u) * 7);
  transform: translate(-50%, -50%) scaleY(.56) rotate(45deg); transition: translate .45s cubic-bezier(.34, 1.45, .64, 1); }
/* bottom: the raw documents */
.slm-layer.l3 { top: calc(var(--u) * 45); background: linear-gradient(135deg, #F472B6, #DB2777);
  box-shadow: calc(var(--u) * 2.6) calc(var(--u) * 2.6) 0 #9D174D, calc(var(--u) * 5) calc(var(--u) * 5) calc(var(--u) * 8) rgba(157, 23, 77, .35); opacity: .62; }
/* middle: the model */
.slm-layer.l2 { top: calc(var(--u) * 33); background: linear-gradient(135deg, #FB7185, #E11D48);
  box-shadow: calc(var(--u) * 2.6) calc(var(--u) * 2.6) 0 #9F1239, calc(var(--u) * 4) calc(var(--u) * 4) calc(var(--u) * 6) rgba(159, 18, 57, .3); opacity: .86; }
/* top: the finished dataset, with a light sweep */
.slm-layer.l1 { top: calc(var(--u) * 21); overflow: hidden; background: linear-gradient(135deg, #FDBA74 0%, #F97316 45%, #F43F5E 100%);
  box-shadow: calc(var(--u) * 2.6) calc(var(--u) * 2.6) 0 #C2410C, inset calc(var(--u) * 1.2) calc(var(--u) * 1.2) 0 rgba(255, 255, 255, .45); }
.slm-layer.l1::after { content: ""; position: absolute; inset: -40%; background: linear-gradient(90deg, transparent 40%, rgba(255, 255, 255, .75) 50%, transparent 60%);
  translate: -100% 0; }
/* the glowing core on the top layer */
.slm-core { position: absolute; left: 50%; top: 50%; width: calc(var(--u) * 9); height: calc(var(--u) * 9); border-radius: calc(var(--u) * 2.4);
  translate: -50% -50%; background: #fff; box-shadow: 0 0 calc(var(--u) * 6) rgba(255, 255, 255, .95), 0 0 calc(var(--u) * 12) rgba(255, 237, 213, .8); }

/* the spark of generation, top right */
.slm-spark { position: absolute; right: calc(var(--u) * -2); top: calc(var(--u) * -1); width: calc(var(--u) * 15); height: calc(var(--u) * 15);
  background: linear-gradient(135deg, #FDBA74, #F43F5E);
  clip-path: polygon(50% 0, 59% 41%, 100% 50%, 59% 59%, 50% 100%, 41% 59%, 0 50%, 41% 41%); }

/* ---- the word mark */
.slm-word { display: inline-flex; align-items: baseline; gap: calc(var(--u) * 7); font-family: "Segoe UI Variable Display", "Segoe UI", system-ui, sans-serif;
  font-size: calc(var(--u) * 30); font-weight: 800; letter-spacing: calc(var(--u) * -.8); white-space: nowrap; }
.slm-word b { font-weight: 800; color: var(--ink, #111827); transition: color .35s; }
.slm-word em { font-style: normal; font-weight: 700; background: linear-gradient(90deg, #F97316, #EC4899, #F97316); background-size: 200% 100%;
  -webkit-background-clip: text; background-clip: text; color: transparent; }

/* ---- motion
   1. on load the layers drop into place one by one
   2. then the stack slowly breathes: layers part and settle, like data being processed
   3. a light sweep crosses the top layer, the core pulses, the spark twinkles */
.slm-logo.is-animated .slm-layer { animation: slm-drop .7s cubic-bezier(.34, 1.45, .64, 1) backwards, slm-breathe 5s ease-in-out 1.2s infinite; }
.slm-logo.is-animated .l3 { --b: calc(var(--u) * 2.2); animation-delay: 0s, 1.2s; }
.slm-logo.is-animated .l2 { --b: 0px; animation-delay: .12s, 1.2s; }
.slm-logo.is-animated .l1 { --b: calc(var(--u) * -2.2); animation-delay: .24s, 1.2s; }
.slm-logo.is-animated .slm-layer.l1::after { animation: slm-sweep 6s ease-in-out 1.6s infinite; }
.slm-logo.is-animated .slm-core { animation: slm-pulse 2.5s ease-in-out infinite; }
.slm-logo.is-animated .slm-spark { animation: slm-pop .5s cubic-bezier(.34, 1.6, .64, 1) .55s backwards, slm-twinkle 3.4s ease-in-out 1.1s infinite; }
.slm-logo.is-animated .slm-halo { animation: slm-halo 5s ease-in-out infinite; }
.slm-logo.is-animated .slm-word em { animation: slm-text 8s ease-in-out infinite; }

@keyframes slm-drop { from { translate: 0 calc(var(--u) * -14); opacity: 0; } }
@keyframes slm-breathe { 0%, 100% { translate: 0 0; } 50% { translate: 0 var(--b); } }
@keyframes slm-sweep { 0%, 55% { translate: -100% 0; } 85%, 100% { translate: 100% 0; } }
@keyframes slm-pulse { 0%, 100% { scale: 1; opacity: .95; } 50% { scale: .78; opacity: .75; } }
@keyframes slm-pop { from { scale: 0; rotate: -90deg; } }
@keyframes slm-twinkle { 0%, 100% { scale: 1; rotate: 0deg; } 40% { scale: .55; rotate: 0deg; } 55% { scale: 1.15; rotate: 90deg; } 70%, 99% { scale: 1; rotate: 90deg; } }
@keyframes slm-halo { 0%, 100% { opacity: .55; scale: .95; } 50% { opacity: .85; scale: 1.05; } }
@keyframes slm-text { 0%, 100% { background-position: 0% 0; } 50% { background-position: 100% 0; } }

/* hover: the stack fans open and the spark spins */
a:hover > .slm-logo .l1, .slm-logo:hover .l1 { translate: 0 calc(var(--u) * -4); }
a:hover > .slm-logo .l3, .slm-logo:hover .l3 { translate: 0 calc(var(--u) * 3.5); }
a:hover > .slm-logo .slm-layer, .slm-logo:hover .slm-layer { animation-play-state: paused; }
a:hover > .slm-logo .slm-spark, .slm-logo:hover .slm-spark { animation: slm-spin .8s cubic-bezier(.34, 1.4, .64, 1); }
@keyframes slm-spin { from { rotate: 0deg; scale: 1; } 50% { rotate: 180deg; scale: 1.3; } to { rotate: 360deg; scale: 1; } }

/* dark mode: brighter halo, lighter back layers so the stack reads on a dark background */
[data-theme="dark"] .slm-halo { background: radial-gradient(circle, rgba(244, 63, 94, .5), transparent 68%); }
[data-theme="dark"] .slm-layer.l3 { opacity: .8; }
[data-theme="dark"] .slm-layer.l2 { opacity: .95; }

@media (prefers-reduced-motion: reduce) { .slm-logo * { animation: none !important; } }

/* never inherit link underline */
.sb-brand, .sb-brand:hover, .slm-logo, .slm-logo * { text-decoration: none !important; }
