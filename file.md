//Logo.jsx
// Application logo, loaded from public/assets/.
//
// To use your own logo, replace public/assets/logo.svg with your file.
// If your file is not an SVG (for example logo.png), put it in public/assets/ and change LOGO_FILE below.
// The landing page and the sidebar both use this component.
const LOGO_FILE = "logo.svg";
const LOGO_URL = `${import.meta.env.BASE_URL}assets/${LOGO_FILE}`;

export default function Logo({ size = 36, withName = true, sub = "Data generation" }) {
  return (
    <span className="logo">
      <img src={LOGO_URL} alt="Logo" width={size} height={size} className="logo-img" />
      {withName && <span className="logo-text"><b>SLM-Finetuning</b>{sub && <small>{sub}</small>}</span>}
    </span>
  );
}












//styles.css
/* SLM-Finetuning · data generation — clean light theme */

/* ================================================================== tokens */
:root {
  /* neutrals */
  --bg: #f6f7f9;
  --surface: #ffffff;
  --surface-2: #fafbfc;
  --surface-3: #f3f4f6;
  --line: #e6e8ec;
  --line-2: #eff1f4;
  --line-strong: #d5d9e0;
  --ink: #111827;
  --ink-2: #374151;
  --muted: #6b7280;
  --faint: #9ca3af;

  /* accent (indigo) */
  --accent: #4f46e5;
  --accent-600: #4338ca;
  --accent-400: #818cf8;
  --accent-200: #c7d2fe;
  --accent-100: #e0e7ff;
  --accent-50: #eef2ff;

  /* supporting tones */
  --violet: #7c3aed;
  --violet-50: #f5f3ff;
  --green: #10b981;
  --green-600: #059669;
  --green-50: #ecfdf5;
  --amber: #f59e0b;
  --orange: #ea580c;
  --orange-50: #fff7ed;
  --red: #dc2626;
  --red-50: #fef2f2;

  /* shape and depth */
  --radius-sm: 8px;
  --radius: 12px;
  --radius-lg: 16px;
  --shadow-xs: 0 1px 2px rgba(16, 24, 40, .04);
  --shadow: 0 1px 2px rgba(16, 24, 40, .04), 0 1px 3px rgba(16, 24, 40, .04);
  --shadow-md: 0 4px 12px -2px rgba(16, 24, 40, .08), 0 2px 4px -2px rgba(16, 24, 40, .04);
  --shadow-lg: 0 24px 48px -12px rgba(16, 24, 40, .18);
  --ring: 0 0 0 3px rgba(79, 70, 229, .16);

  --font: "Inter", "Segoe UI Variable Text", "Segoe UI", system-ui, -apple-system, "Helvetica Neue", Arial, sans-serif;
  --mono: "JetBrains Mono", "Cascadia Mono", "SFMono-Regular", Consolas, ui-monospace, monospace;

  font-family: var(--font);
  color: var(--ink);
  background: var(--bg);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  font-feature-settings: "cv11", "ss01";
}
* { box-sizing: border-box; }
html, body, #root { margin: 0; height: 100%; }
body { font-size: 15px; line-height: 1.5; background: var(--bg); }
button, input, select, textarea { font: inherit; color: inherit; }
a { color: var(--accent); }
::selection { background: var(--accent-100); }
:focus-visible { outline: 2px solid var(--accent-400); outline-offset: 2px; }
::-webkit-scrollbar { width: 10px; height: 10px; }
::-webkit-scrollbar-thumb { background: #d9dce2; border-radius: 99px; border: 3px solid transparent; background-clip: content-box; }
::-webkit-scrollbar-thumb:hover { background-color: #c3c8d0; }
::-webkit-scrollbar-track { background: transparent; }

.grow { flex: 1; }
.muted { color: var(--muted); }
.small { font-size: 13px; }
.mono { font-family: var(--mono); font-size: 13.5px; }
.ellipsis { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; min-width: 0; }
.ico { flex: none; display: block; }
.err-t { color: var(--red); }
.ok-t { color: var(--green-600); display: inline-flex; gap: 4px; align-items: center; }

/* ================================================================== shell */
.app { display: grid; grid-template-columns: 256px minmax(0, 1fr); height: 100vh; }

/* ---- sidebar */
.sidebar {
  background: var(--surface); border-right: 1px solid var(--line);
  display: flex; flex-direction: column; gap: 20px; padding: 18px 14px 14px; overflow: auto; min-height: 0;
}
.brand { display: flex; gap: 11px; align-items: center; padding: 2px 6px 4px; text-decoration: none; color: inherit; border-radius: 10px; }
.brand-mark { width: 34px; height: 34px; border-radius: 10px; display: grid; place-items: center; color: #fff; flex: none;
  background: linear-gradient(135deg, #6366f1 0%, #4f46e5 50%, #7c3aed 100%);
  box-shadow: 0 4px 10px -2px rgba(79, 70, 229, .45), inset 0 1px 0 rgba(255, 255, 255, .25); }
.brand b { display: block; font-size: 15.5px; font-weight: 650; letter-spacing: -.1px; color: var(--ink); line-height: 1.25; }
.brand span { font-size: 13px; color: var(--muted); }

.side-label { padding: 0 10px 6px; font-size: 12px; font-weight: 600; letter-spacing: .5px; text-transform: uppercase; color: var(--faint); }
.nav { display: grid; gap: 2px; }
.nav a, .nav button { display: flex; align-items: center; gap: 10px; padding: 8px 10px; border-radius: var(--radius-sm); color: var(--ink-2);
  text-decoration: none; background: none; border: 0; cursor: pointer; text-align: left; font-weight: 500; font-size: 14.5px;
  transition: background .12s, color .12s; }
.nav a .ico, .nav button .ico { color: var(--faint); transition: color .12s; }
.nav a:hover, .nav button:hover:not(:disabled) { background: var(--surface-3); color: var(--ink); }
.nav a:hover .ico, .nav button:hover:not(:disabled) .ico { color: var(--ink-2); }
.nav a.on { background: var(--accent-50); color: var(--accent-600); font-weight: 600; }
.nav a.on .ico { color: var(--accent); }
.nav button:disabled { opacity: .45; cursor: default; }
.nav-n { margin-left: auto; font-size: 12px; font-weight: 600; background: var(--surface-3); color: var(--ink-2); padding: 1px 8px; border-radius: 99px; }
.nav a.on .nav-n { background: #fff; color: var(--accent-600); box-shadow: 0 0 0 1px var(--accent-100); }

.side-sec { display: flex; flex-direction: column; min-height: 0; flex: 1; }
.side-h { display: flex; align-items: center; justify-content: space-between; padding: 0 6px 8px 10px; font-size: 12px; font-weight: 600;
  letter-spacing: .5px; text-transform: uppercase; color: var(--faint); }
.side-new { display: inline-flex; align-items: center; gap: 4px; background: var(--surface); border: 1px solid var(--line); color: var(--ink-2);
  border-radius: 7px; padding: 3px 8px; font-size: 13px; cursor: pointer; text-transform: none; letter-spacing: 0; font-weight: 600;
  box-shadow: var(--shadow-xs); transition: border-color .12s, background .12s; }
.side-new:hover { border-color: var(--line-strong); background: var(--surface-2); }
.side-new.icon { padding: 4px; border-color: transparent; box-shadow: none; background: none; color: var(--muted); }
.side-new.icon:hover { background: var(--surface-3); color: var(--ink); }
.joblist { display: grid; gap: 2px; overflow: auto; min-height: 0; align-content: start; }
.jobitem { display: flex; gap: 10px; align-items: center; padding: 7px 10px; border-radius: var(--radius-sm); background: none; border: 1px solid transparent;
  cursor: pointer; color: var(--ink-2); text-align: left; min-width: 0; transition: background .12s; }
.jobitem:hover { background: var(--surface-3); }
.jobitem.on { background: var(--surface); border-color: var(--line); box-shadow: var(--shadow); color: var(--ink); }
.ji-t { display: grid; min-width: 0; flex: 1; }
.ji-t b { font-weight: 600; font-size: 14px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.ji-t small { font-size: 12.5px; color: var(--faint); font-family: var(--mono); overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.side-empty { font-size: 13.5px; color: var(--muted); padding: 10px; border: 1px dashed var(--line); border-radius: var(--radius-sm); }
.side-ep { background: var(--surface-2); border: 1px solid var(--line); border-radius: var(--radius); padding: 10px 4px 8px; }
.side-ep .side-h { padding-bottom: 4px; }
.ep-line { display: flex; align-items: center; gap: 9px; padding: 4px 10px; font-size: 14px; color: var(--ink-2); font-weight: 500; }
.ep-line small { margin-left: auto; color: var(--muted); font-size: 12.5px; font-variant-numeric: tabular-nums; font-weight: 400; }

.dot { width: 8px; height: 8px; border-radius: 50%; background: #d1d5db; flex: none; }
.dot.ok { background: var(--green); box-shadow: 0 0 0 3px rgba(16, 185, 129, .16); }
.dot.bad { background: #f87171; box-shadow: 0 0 0 3px rgba(248, 113, 113, .16); }
.dot.run { background: var(--accent); animation: pulse 1.3s infinite; }
.dot.muted { background: #e5e7eb; }
@keyframes pulse { 0% { box-shadow: 0 0 0 0 rgba(79, 70, 229, .35); } 70% { box-shadow: 0 0 0 6px rgba(79, 70, 229, 0); } 100% { box-shadow: 0 0 0 0 rgba(79, 70, 229, 0); } }

/* ---- main column */
.main { display: flex; flex-direction: column; min-width: 0; height: 100vh; }
.topbar { display: flex; align-items: center; gap: 10px; padding: 14px 28px; background: rgba(255, 255, 255, .85);
  backdrop-filter: saturate(1.4) blur(8px); border-bottom: 1px solid var(--line); min-height: 68px; }
.tb-title { min-width: 0; }
.crumbs { font-size: 13px; color: var(--muted); display: flex; align-items: center; gap: 6px; }
.crumbs .sep { color: var(--line-strong); }
.crumbs .mono { font-size: 12.5px; color: var(--ink-2); background: var(--surface-3); padding: 0 6px; border-radius: 5px; }
.tb-title h1 { margin: 2px 0 0; font-size: 20.5px; font-weight: 650; letter-spacing: -.3px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.pill { display: inline-flex; align-items: center; gap: 6px; font-size: 13px; font-weight: 600; padding: 4px 10px; border-radius: 99px;
  background: var(--surface); color: var(--ink-2); border: 1px solid var(--line); white-space: nowrap; }
.pill::before { content: ""; width: 6px; height: 6px; border-radius: 50%; background: var(--faint); }
.pill.mode::before { display: none; }
.pill.ok { background: var(--green-50); color: var(--green-600); border-color: #c8eedd; }
.pill.ok::before { background: var(--green); }
.pill.bad { background: var(--red-50); color: var(--red); border-color: #fbd5d5; }
.pill.bad::before { background: var(--red); }
.pill.run { background: var(--accent-50); color: var(--accent-600); border-color: var(--accent-100); }
.pill.run::before { background: var(--accent); animation: pulse 1.3s infinite; }
.tabs-top { display: flex; gap: 2px; background: var(--surface-3); padding: 3px; border-radius: 10px; margin-left: 8px; border: 1px solid var(--line-2); }
.tabs-top a { display: inline-flex; align-items: center; gap: 6px; padding: 6px 14px; border-radius: 7px; text-decoration: none; color: var(--muted);
  font-weight: 600; font-size: 14px; transition: color .12s, background .12s; }
.tabs-top a:hover { color: var(--ink); }
.tabs-top a.on { background: var(--surface); color: var(--ink); box-shadow: 0 1px 2px rgba(16, 24, 40, .08), 0 0 0 1px rgba(16, 24, 40, .03); }
.tabs-top a.on .ico { color: var(--accent); }
.content { flex: 1; min-height: 0; overflow: auto; padding: 22px 28px 26px; }
.content.fit { overflow: hidden; }

/* ================================================================== cards and controls */
.card { background: var(--surface); border: 1px solid var(--line); border-radius: var(--radius-lg); box-shadow: var(--shadow);
  display: flex; flex-direction: column; min-height: 0; min-width: 0; }
.card-hd { display: flex; align-items: center; gap: 12px; padding: 12px 18px; border-bottom: 1px solid var(--line-2); min-height: 60px; }
.card-t { display: grid; min-width: 0; }
.card-t h2 { margin: 0; font-size: 15.5px; font-weight: 650; letter-spacing: -.1px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.card-t small { color: var(--muted); font-size: 13.5px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.card-ico { width: 34px; height: 34px; border-radius: 10px; display: grid; place-items: center; background: var(--accent-50); color: var(--accent);
  flex: none; box-shadow: inset 0 0 0 1px var(--accent-100); }
.card-ico.indigo { background: var(--violet-50); color: var(--violet); box-shadow: inset 0 0 0 1px #e9e3fd; }
.card-ico.green { background: var(--green-50); color: var(--green-600); box-shadow: inset 0 0 0 1px #d1f2e4; }
.card-ico.sm { width: 28px; height: 28px; border-radius: 8px; }
.step-n { font-size: 11.5px; font-weight: 700; color: var(--accent); letter-spacing: .5px; text-transform: uppercase; line-height: 1.3; }
.card-bd { padding: 16px 18px 18px; overflow: auto; overflow-x: hidden; flex: 1; min-height: 0; display: grid; gap: 16px; align-content: start; }
.card-bd.tight { gap: 10px; }

.btn { display: inline-flex; align-items: center; justify-content: center; gap: 7px; height: 36px; padding: 0 14px; border-radius: 9px;
  border: 1px solid var(--line); background: var(--surface); color: var(--ink-2); font-weight: 600; font-size: 14px; cursor: pointer;
  text-decoration: none; white-space: nowrap; box-shadow: var(--shadow-xs);
  transition: background .12s, border-color .12s, box-shadow .12s, color .12s, transform .06s; }
.btn:hover:not(:disabled) { border-color: var(--line-strong); background: var(--surface-2); color: var(--ink); }
.btn:active:not(:disabled) { transform: translateY(.5px); }
.btn.primary { background: var(--accent); border-color: var(--accent); color: #fff;
  box-shadow: 0 1px 2px rgba(16, 24, 40, .08), 0 2px 8px -2px rgba(79, 70, 229, .5), inset 0 1px 0 rgba(255, 255, 255, .14); }
.btn.primary:hover:not(:disabled) { background: var(--accent-600); border-color: var(--accent-600); color: #fff; }
.btn.soft { background: var(--accent-50); border-color: var(--accent-100); color: var(--accent-600); box-shadow: none; }
.btn.soft:hover:not(:disabled) { background: var(--accent-100); border-color: var(--accent-200); color: var(--accent-600); }
.btn.ghost { background: transparent; border-color: transparent; color: var(--ink-2); box-shadow: none; }
.btn.ghost:hover:not(:disabled) { background: var(--surface-3); border-color: transparent; }
.btn.danger { color: var(--red); }
.btn.danger:hover:not(:disabled) { background: var(--red-50); color: var(--red); }
.btn.small { height: 30px; padding: 0 10px; font-size: 13.5px; border-radius: 8px; gap: 5px; }
.btn.wide { width: 100%; }
.btn:disabled { opacity: .5; cursor: not-allowed; box-shadow: none; }
.icon-btn { display: grid; place-items: center; width: 30px; height: 30px; border-radius: 8px; border: 0; background: none; color: var(--muted);
  cursor: pointer; transition: background .12s, color .12s; }
.icon-btn:hover { background: var(--surface-3); color: var(--ink); }
.link { background: none; border: 0; padding: 0; color: var(--accent); font-weight: 600; cursor: pointer; display: inline-flex; align-items: center; gap: 4px; font-size: 13.5px; }
.link:hover { color: var(--accent-600); text-decoration: underline; text-underline-offset: 2px; }
.vsep { width: 1px; height: 24px; background: var(--line); margin: 0 4px; }
.btn-group { display: inline-flex; border: 1px solid var(--line); border-radius: 9px; background: var(--surface); box-shadow: var(--shadow-xs); overflow: hidden; }
.btn-group .btn { border: 0; border-radius: 0; box-shadow: none; height: 34px; }
.btn-group .btn + .btn { border-left: 1px solid var(--line-2); }

.field { display: grid; gap: 6px; min-width: 0; align-content: start; }
.field-l { font-size: 13.5px; font-weight: 600; color: var(--ink-2); }
.field-h { font-size: 13px; color: var(--muted); overflow-wrap: anywhere; line-height: 1.4; }
.span2 { grid-column: span 2; }
input:not([type=checkbox]):not([type=file]), select, textarea {
  width: 100%; height: 36px; padding: 0 11px; border: 1px solid var(--line-strong); border-radius: 9px; background: var(--surface); outline: none; min-width: 0;
  box-shadow: var(--shadow-xs); transition: border-color .12s, box-shadow .12s; }
textarea { height: auto; padding: 9px 11px; resize: vertical; line-height: 1.5; }
input::placeholder, textarea::placeholder { color: var(--faint); }
input:hover:not(:focus):not(:disabled), select:hover:not(:focus):not(:disabled), textarea:hover:not(:focus):not(:disabled) { border-color: #c3c8d0; }
input:focus, select:focus, textarea:focus { border-color: var(--accent-400); box-shadow: var(--ring); }
select { appearance: none; padding-right: 32px; cursor: pointer;
  background: var(--surface) url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='%236b7280' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'%3E%3Cpath d='m7 10 5 5 5-5'/%3E%3C/svg%3E") no-repeat right 9px center; }
input.mini { width: 64px !important; height: 30px !important; padding: 0 8px !important; text-align: center; }
select.compact { width: auto; height: 34px; font-size: 14px; }
.grid2 { display: grid; grid-template-columns: minmax(0, 1fr) minmax(0, 1fr); gap: 14px; }
.grid3 { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 16px; }

.switch { width: 38px; height: 22px; border-radius: 99px; border: 0; background: #d9dde3; position: relative; cursor: pointer; flex: none;
  transition: background .18s; padding: 0; box-shadow: inset 0 1px 2px rgba(16, 24, 40, .08); }
.switch span { position: absolute; top: 3px; left: 3px; width: 16px; height: 16px; border-radius: 50%; background: #fff;
  box-shadow: 0 1px 3px rgba(16, 24, 40, .2); transition: left .18s cubic-bezier(.3, .7, .4, 1); }
.switch:hover:not(:disabled) { background: #cbd0d8; }
.switch.on, .switch.on:hover:not(:disabled) { background: var(--accent); }
.switch.on span { left: 19px; }
.switch:disabled { opacity: .5; cursor: not-allowed; }

.seg { display: inline-flex; background: var(--surface-3); border-radius: 9px; padding: 3px; gap: 2px; width: 100%; border: 1px solid var(--line-2); }
.seg button { flex: 1; border: 0; background: none; height: 28px; padding: 0 10px; border-radius: 7px; font-weight: 600; font-size: 13.5px; color: var(--muted);
  cursor: pointer; white-space: nowrap; transition: color .12s, background .12s; }
.seg button:hover { color: var(--ink); }
.seg button.on { background: var(--surface); color: var(--accent-600); box-shadow: 0 1px 2px rgba(16, 24, 40, .1), 0 0 0 1px rgba(16, 24, 40, .03); }
.seg.small { width: auto; margin-bottom: 6px; }
.seg.small button { height: 24px; padding: 0 10px; font-size: 13px; }

.chips { display: flex; flex-wrap: wrap; gap: 6px; }
.chip { display: inline-flex; align-items: center; gap: 5px; height: 28px; padding: 0 11px; border-radius: 99px; border: 1px solid var(--line);
  font-size: 13.5px; font-weight: 500; color: var(--ink-2); background: var(--surface); cursor: pointer; user-select: none; transition: all .12s; }
.chip:hover { border-color: var(--line-strong); background: var(--surface-2); }
.chip.on { background: var(--accent-50); border-color: var(--accent-200); color: var(--accent-600); font-weight: 600; }
.chip.static { cursor: default; font-size: 12.5px; height: 22px; padding: 0 9px; background: var(--surface-3); border-color: transparent; color: var(--ink-2); }
.chip-x { border: 0; background: none; display: grid; place-items: center; padding: 0 0 0 2px; color: inherit; cursor: pointer; opacity: .6; }
.chip-x:hover { opacity: 1; }
.chip-note { font-weight: 400; }
.tag { display: inline-flex; align-items: center; gap: 4px; font-size: 12px; font-weight: 600; padding: 2px 8px; border-radius: 6px;
  background: var(--surface-3); color: var(--ink-2); white-space: nowrap; line-height: 1.5; }
.tag.ok { background: var(--green-50); color: var(--green-600); }
.tag.bad { background: var(--red-50); color: var(--red); }
.tag.run { background: var(--accent-50); color: var(--accent-600); }
.tag.muted { background: var(--surface-3); color: var(--faint); }

/* ================================================================== generate */
.generate { height: 100%; display: grid; grid-template-rows: minmax(0, 1fr) auto; gap: 18px; }
.cols { border: 0; margin: 0; padding: 0; min-width: 0; min-height: 0; display: grid; gap: 18px;
  grid-template-columns: minmax(0, .95fr) minmax(0, .9fr) minmax(0, 1.2fr); grid-template-rows: minmax(0, 1fr); }
.cols:disabled .card-bd { opacity: .55; pointer-events: none; }

.drop { display: flex; gap: 14px; align-items: center; padding: 18px; border: 1.5px dashed var(--line-strong); border-radius: var(--radius);
  background: var(--surface-2); cursor: pointer; transition: border-color .15s, background .15s; }
.drop:hover, .drop.drag { border-color: var(--accent-400); background: var(--accent-50); }
.drop-ico { width: 42px; height: 42px; border-radius: 11px; display: grid; place-items: center; background: var(--surface); color: var(--accent);
  box-shadow: var(--shadow-md); border: 1px solid var(--line); flex: none; transition: transform .15s; }
.drop:hover .drop-ico, .drop.drag .drop-ico { transform: translateY(-1px); }
.drop b { color: var(--ink); font-weight: 600; } .drop u { color: var(--accent); text-decoration: none; font-weight: 600; }
.drop small { display: block; color: var(--muted); font-size: 13px; margin-top: 2px; }
.files { display: grid; gap: 6px; max-height: 132px; overflow: auto; }
.file { display: flex; gap: 10px; align-items: center; padding: 8px 10px; border: 1px solid var(--line); border-radius: 10px; background: var(--surface);
  text-align: left; min-width: 0; overflow: hidden; transition: border-color .12s, box-shadow .12s, background .12s; }
.file.pick { cursor: pointer; width: 100%; }
.file.pick:hover { border-color: var(--line-strong); background: var(--surface-2); }
.file.on, .file.on:hover { border-color: var(--accent-400); background: var(--surface); box-shadow: var(--ring); }
.file-n { display: grid; flex: 1; min-width: 0; font-weight: 600; font-size: 14px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.file-n small { font-weight: 400; color: var(--muted); font-size: 12.5px; overflow: hidden; text-overflow: ellipsis; }
.ext { font-size: 10.5px; font-weight: 700; letter-spacing: .3px; width: 36px; text-align: center; padding: 5px 0; border-radius: 7px; background: var(--surface-3);
  color: var(--ink-2); flex: none; overflow: hidden; }
.ext.pdf { background: #fef2f2; color: #dc2626; } .ext.docx { background: #eff6ff; color: #2563eb; }
.ext.html { background: var(--orange-50); color: var(--orange); } .ext.md, .ext.txt { background: var(--green-50); color: var(--green-600); }

.models { border: 1px solid var(--line); border-radius: var(--radius); padding: 12px 14px; display: grid; gap: 8px; background: var(--surface-2); }
.models-h { display: flex; align-items: center; justify-content: space-between; font-size: 13.5px; font-weight: 600; color: var(--ink-2); }
.models-h span { display: inline-flex; gap: 7px; align-items: center; }
.models-h .ico { color: var(--muted); }
.model-row { display: grid; grid-template-columns: 10px 58px minmax(0, 1fr) auto; gap: 9px; align-items: center; font-size: 14px;
  padding: 7px 10px; background: var(--surface); border: 1px solid var(--line-2); border-radius: 9px; }
.model-row b { font-weight: 600; }
.model-row .mono { color: var(--muted); font-size: 13px; }
.models-f { display: flex; align-items: center; gap: 8px; justify-content: space-between; padding-top: 2px; }

.feature { display: grid; grid-template-columns: 38px minmax(0, 1fr) auto; gap: 12px; align-items: center; padding: 12px 14px;
  border: 1px solid var(--line); border-radius: var(--radius); background: var(--surface-2); transition: background .15s, border-color .15s; }
.feature.on { background: linear-gradient(135deg, var(--accent-50), #f8f7ff 60%, var(--surface)); border-color: var(--accent-100); }
.feature-ico { width: 38px; height: 38px; border-radius: 10px; display: grid; place-items: center; background: var(--surface); color: var(--muted);
  border: 1px solid var(--line); transition: all .15s; }
.feature.on .feature-ico { background: var(--accent); color: #fff; border-color: var(--accent); box-shadow: 0 4px 10px -3px rgba(79, 70, 229, .5); }
.feature b { display: block; font-size: 14.5px; font-weight: 600; } .feature small { color: var(--muted); font-size: 13px; display: block; line-height: 1.4; }

.split3 { display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); gap: 8px; }
.inset { position: relative; }
.inset small { position: absolute; top: 6px; left: 11px; font-size: 11.5px; color: var(--muted); font-weight: 600; pointer-events: none; letter-spacing: .2px; }
.inset input { height: 46px !important; padding: 18px 11px 4px !important; font-weight: 600; font-variant-numeric: tabular-nums; }

.tile { border: 1px solid var(--line); border-radius: var(--radius); padding: 10px 12px; display: grid; gap: 9px; background: var(--surface);
  transition: border-color .15s, box-shadow .15s, background .15s; }
.tile:hover { border-color: var(--line-strong); }
.tile.on { border-color: var(--accent-200); background: linear-gradient(180deg, #fbfbff, var(--surface)); box-shadow: 0 0 0 3px rgba(79, 70, 229, .06); }
.tile-top { display: grid; grid-template-columns: 32px minmax(0, 1fr) auto; gap: 12px; align-items: center; }
.tile-ico { width: 32px; height: 32px; border-radius: 9px; display: grid; place-items: center; background: var(--surface-3); color: var(--muted); transition: all .15s; }
.tile.on .tile-ico { background: var(--accent); color: #fff; box-shadow: 0 4px 10px -3px rgba(79, 70, 229, .5); }
.tile-t b { display: block; font-size: 14.5px; font-weight: 600; }
.tile-t small { display: block; color: var(--muted); font-size: 13px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.tile-bd { display: grid; gap: 9px; padding: 10px 0 2px; border-top: 1px dashed var(--line); }
.tile-bd .chips { gap: 5px; }
.tile-bd .chip { height: 26px; padding: 0 10px; font-size: 13px; }
.inline { display: flex; align-items: center; gap: 8px; font-size: 13.5px; color: var(--ink-2); flex-wrap: wrap; }
.addtopic { display: flex; gap: 6px; flex: 1; min-width: 180px; margin-left: 8px; }
.addtopic input { height: 30px !important; padding: 0 10px !important; font-size: 13.5px; }

.mixbox { border: 1px solid var(--line); border-radius: var(--radius); padding: 12px 14px; display: grid; gap: 9px; background: var(--surface-2); }
.mixbox-h { display: flex; justify-content: space-between; align-items: center; font-size: 13.5px; }
.mixbox-h b { font-weight: 600; }
.mix { display: grid; gap: 8px; }
.mixbar { display: flex; height: 8px; border-radius: 99px; overflow: hidden; background: var(--line-2); gap: 2px; }
.mixbar i { display: block; height: 100%; transition: width .3s; border-radius: 99px; }
.mixleg { display: flex; flex-wrap: wrap; gap: 4px 14px; font-size: 13px; color: var(--muted); }
.mixleg b { color: var(--ink); font-weight: 600; font-variant-numeric: tabular-nums; }
.mixleg span::before { content: ""; display: inline-block; width: 8px; height: 8px; border-radius: 3px; margin-right: 6px; background: var(--c); }

.warn { display: flex; gap: 10px; align-items: flex-start; font-size: 13.5px; color: #92400e; background: #fffbeb;
  border: 1px solid #fde68a; border-radius: var(--radius); padding: 10px 12px; line-height: 1.45; }
.warn .ico { margin-top: 1px; color: var(--amber); }
.warn b { color: #78350f; font-weight: 600; }
.warn.block { margin-bottom: 18px; padding: 12px 16px; }
.warn ul { margin: 4px 0 0; padding-left: 18px; }
.small-warn { margin-top: 10px; font-size: 13px; }
.info { display: flex; gap: 9px; align-items: center; font-size: 14px; color: var(--accent-600); background: var(--accent-50);
  border: 1px solid var(--accent-100); border-radius: var(--radius); padding: 10px 14px; margin-bottom: 16px; }
.info.small { font-size: 13px; margin: 0; padding: 8px 12px; }
.error { font-size: 13.5px; background: var(--red-50); color: #b91c1c; border: 1px solid #fecaca; border-radius: 10px; padding: 9px 12px; white-space: pre-wrap; }

/* ---- run panel (docked at the bottom of Generate) */
.runpanel { padding: 14px 18px 16px; gap: 12px; box-shadow: var(--shadow-md); }
.run-top { display: flex; align-items: center; gap: 8px; }
.run-dot { width: 36px; height: 36px; border-radius: 10px; flex: none; margin-right: 4px; display: grid; place-items: center;
  background: var(--surface-3); color: var(--faint); position: relative; }
.run-dot::after { content: ""; width: 10px; height: 10px; border-radius: 50%; background: currentColor; }
.run-dot.run { background: var(--accent-50); color: var(--accent); }
.run-dot.run::after { animation: pulse 1.3s infinite; }
.run-dot.ok { background: var(--green-50); color: var(--green); }
.run-dot.bad { background: var(--red-50); color: var(--red); }
.run-t { display: grid; min-width: 0; }
.run-t b { font-size: 15px; font-weight: 650; } .run-t span { color: var(--muted); font-size: 13.5px; font-variant-numeric: tabular-nums; }
.track { display: grid; gap: 6px; }
.step { display: grid; gap: 6px; min-width: 0; }
.step > b { display: block; height: 5px; border-radius: 99px; background: var(--line-2); overflow: hidden; }
.step > b > i { display: block; height: 100%; width: 0; background: var(--accent); border-radius: 99px; transition: width .3s; }
.step.now > b > i { background: linear-gradient(90deg, var(--accent-400), var(--accent)); }
.step.done > b > i { width: 100%; background: var(--green); }
.step.fail > b > i { width: 100%; background: var(--red); }
.step.skip > b { background: repeating-linear-gradient(45deg, var(--line-2) 0 4px, var(--surface) 4px 8px); }
.step > span { display: flex; align-items: center; gap: 3px; font-size: 12px; font-weight: 500; color: var(--faint); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.step.done > span { color: var(--green-600); } .step.now > span { color: var(--accent); font-weight: 600; }
.step.fail > span { color: var(--red); font-weight: 600; } .step.skip > span { color: #c4c9d1; }

/* ================================================================== modal / drawer */
.overlay { position: fixed; inset: 0; background: rgba(17, 24, 39, .32); backdrop-filter: blur(3px); z-index: 50; display: grid; place-items: center;
  padding: 24px; animation: fade .15s ease-out; }
@keyframes fade { from { opacity: 0; } }
.modal { background: var(--surface); border-radius: var(--radius-lg); box-shadow: var(--shadow-lg); border: 1px solid var(--line);
  width: min(640px, 100%); max-height: calc(100vh - 48px); display: flex; flex-direction: column; animation: pop .18s cubic-bezier(.2, .8, .3, 1); }
@keyframes pop { from { transform: translateY(8px) scale(.985); opacity: 0; } }
.modal.wide { width: min(1020px, 100%); }
.modal-hd { display: flex; gap: 12px; align-items: flex-start; justify-content: space-between; padding: 18px 22px; border-bottom: 1px solid var(--line-2); }
.modal-hd h2 { margin: 0; font-size: 18.5px; font-weight: 650; letter-spacing: -.2px; }
.modal-hd p { margin: 3px 0 0; color: var(--muted); font-size: 14px; max-width: 760px; }
.modal-bd { padding: 20px 22px; overflow: auto; min-height: 0; }
.modal-ft { display: flex; gap: 8px; align-items: center; justify-content: flex-end; padding: 14px 22px; border-top: 1px solid var(--line-2);
  flex-wrap: wrap; background: var(--surface-2); border-radius: 0 0 var(--radius-lg) var(--radius-lg); }
.adv { display: grid; grid-template-columns: 220px minmax(0, 1fr); gap: 24px; min-height: 400px; }
.adv-nav { display: grid; gap: 2px; align-content: start; padding-right: 16px; border-right: 1px solid var(--line-2); }
.adv-nav button { display: flex; gap: 10px; align-items: center; padding: 9px 11px; border-radius: var(--radius-sm); border: 0; background: none; cursor: pointer;
  text-align: left; color: var(--ink-2); font-weight: 500; font-size: 14px; transition: background .12s, color .12s; }
.adv-nav button .ico { color: var(--faint); }
.adv-nav button:hover { background: var(--surface-3); }
.adv-nav button.on { background: var(--accent-50); color: var(--accent-600); font-weight: 600; }
.adv-nav button.on .ico { color: var(--accent); }
.adv-bd { display: grid; gap: 16px; align-content: start; }
.lead { margin: 0; color: var(--muted); font-size: 14px; line-height: 1.55; }
.test-row { display: flex; align-items: center; justify-content: space-between; gap: 12px; padding: 12px 14px; background: var(--surface-2);
  border: 1px solid var(--line); border-radius: var(--radius); }
.rules { display: grid; gap: 12px; }
.rule { border: 1px solid var(--line); border-radius: var(--radius); padding: 14px 16px; display: grid; gap: 12px; background: var(--surface); box-shadow: var(--shadow-xs); transition: opacity .15s; }
.rule.off { opacity: .55; background: var(--surface-2); }
.rule-top { display: flex; gap: 12px; align-items: center; }
.rule-name { max-width: 300px; font-weight: 600; }

.drawer-wrap { place-items: stretch end; padding: 0; }
.drawer { width: min(1180px, 94vw); height: 100vh; background: var(--bg); display: flex; flex-direction: column; box-shadow: var(--shadow-lg);
  border-left: 1px solid var(--line); animation: slide .22s cubic-bezier(.2, .8, .3, 1); }
@keyframes slide { from { transform: translateX(32px); opacity: .5; } }
.drawer-hd { display: flex; align-items: center; gap: 16px; padding: 14px 22px; background: var(--surface); border-bottom: 1px solid var(--line); min-height: 64px; }
.drawer-hd h2 { margin: 0; font-size: 17.5px; font-weight: 650; letter-spacing: -.2px; }
.drawer-bd { padding: 22px; overflow: auto; flex: 1; min-height: 0; }

.toasts { position: fixed; right: 22px; bottom: 22px; display: grid; gap: 8px; z-index: 80; }
.toast { display: flex; gap: 10px; align-items: center; background: var(--surface); color: var(--ink); padding: 11px 14px 11px 12px; border-radius: var(--radius);
  font-size: 14px; font-weight: 500; box-shadow: var(--shadow-lg); border: 1px solid var(--line); max-width: 440px; animation: slide .2s ease-out; }
.toast .ico { width: 22px; height: 22px; padding: 4px; border-radius: 50%; background: var(--green-50); color: var(--green-600); }
.toast.error .ico { background: var(--red-50); color: var(--red); }

/* ================================================================== results */
.results { display: grid; gap: 0; }
.stats { display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); gap: 16px; margin-bottom: 18px; }
.stat { background: var(--surface); border: 1px solid var(--line); border-radius: var(--radius-lg); box-shadow: var(--shadow); padding: 16px 18px;
  display: flex; flex-direction: column; gap: 12px; min-width: 0; }
.stat-ico { width: 34px; height: 34px; border-radius: 10px; display: grid; place-items: center; flex: none; }
.stat-ico.blue { background: var(--accent-50); color: var(--accent); } .stat-ico.green { background: var(--green-50); color: var(--green-600); }
.stat-ico.violet { background: var(--violet-50); color: var(--violet); } .stat-ico.orange { background: var(--orange-50); color: var(--orange); }
.stat-b { display: grid; gap: 2px; min-width: 0; flex: 1; align-content: start; }
.stat-b > b { font-size: 28px; font-weight: 700; letter-spacing: -.6px; line-height: 1.15; font-variant-numeric: tabular-nums; }
.stat-b > span { font-size: 14px; color: var(--ink-2); font-weight: 500; }
.stat-b > small { font-size: 13px; color: var(--muted); }
.stat-b .mix { margin-top: 8px; }

.acceptbar { flex-direction: row; align-items: center; gap: 10px; padding: 14px 18px; margin-bottom: 18px; flex-wrap: wrap;
  background: linear-gradient(100deg, var(--surface) 40%, var(--accent-50)); border-color: var(--accent-100); }
.accept-t { display: grid; gap: 2px; min-width: 0; }
.accept-t b { font-size: 16px; font-weight: 650; }
.res-grid { display: grid; grid-template-columns: minmax(0, 1.65fr) minmax(320px, 1fr); gap: 18px; align-items: start; }
.side-col { display: grid; gap: 16px; }
.panel .card-hd { padding: 12px 16px; min-height: 52px; }
.panel .card-t h2 { font-size: 14.5px; }
.panel .card-bd { padding: 14px 16px 16px; }

.search { display: flex; align-items: center; gap: 7px; border: 1px solid var(--line-strong); border-radius: 9px; padding: 0 10px; color: var(--faint);
  background: var(--surface); box-shadow: var(--shadow-xs); height: 34px; transition: border-color .12s, box-shadow .12s; }
.search:focus-within { border-color: var(--accent-400); box-shadow: var(--ring); }
.search input { border: 0 !important; box-shadow: none !important; padding: 0 !important; height: 32px !important; width: 180px !important; background: none; }

.pilltabs { display: flex; gap: 2px; flex-wrap: nowrap; overflow-x: auto; max-width: 100%; scrollbar-width: none; background: var(--surface-3); padding: 3px; border-radius: 10px; border: 1px solid var(--line-2); width: fit-content; }
.pilltabs button { display: inline-flex; align-items: center; gap: 6px; border: 0; background: none; color: var(--muted); height: 28px; padding: 0 10px; flex: none; white-space: nowrap;
  border-radius: 7px; font-weight: 600; font-size: 13.5px; cursor: pointer; transition: color .12s, background .12s; }
.pilltabs button:hover { color: var(--ink); }
.pilltabs button span { font-size: 12px; color: var(--faint); font-weight: 600; background: var(--surface); padding: 0 6px; border-radius: 99px;
  min-width: 20px; text-align: center; font-variant-numeric: tabular-nums; border: 1px solid var(--line-2); }
.pilltabs button.on { background: var(--surface); color: var(--ink); box-shadow: 0 1px 2px rgba(16, 24, 40, .1), 0 0 0 1px rgba(16, 24, 40, .03); }
.pilltabs button.on span { background: var(--accent-50); color: var(--accent-600); border-color: var(--accent-100); }
.inline-tabs { margin-left: 4px; }

.pair { border: 1px solid var(--line); border-radius: var(--radius); padding: 14px 16px; display: grid; gap: 10px; background: var(--surface);
  transition: box-shadow .15s, border-color .15s; }
.pair:hover { border-color: var(--line-strong); box-shadow: var(--shadow-md); }
.pair.rejected { opacity: .6; background: var(--surface-2); }
.pair-meta { display: flex; gap: 8px; align-items: center; font-size: 13px; color: var(--muted); min-width: 0; }
.pair-meta > .mono:first-child { font-size: 12.5px; color: var(--ink-2); background: var(--surface-3); padding: 1px 6px; border-radius: 5px; }
.tag.t-from_images { background: var(--violet-50); color: var(--violet); }
.tag.t-decline { background: var(--orange-50); color: var(--orange); }
.tag.t-general { background: var(--green-50); color: var(--green-600); }
.state { display: inline-flex; align-items: center; gap: 5px; font-size: 12.5px; font-weight: 600; padding: 2px 9px; border-radius: 99px;
  background: var(--surface-3); color: var(--muted); white-space: nowrap; }
.state::before { content: ""; width: 6px; height: 6px; border-radius: 50%; background: currentColor; opacity: .8; }
.state.ok { background: var(--green-50); color: var(--green-600); } .state.bad { background: var(--red-50); color: var(--red); }

.msg { display: grid; grid-template-columns: 76px minmax(0, 1fr); gap: 12px; align-items: start; }
.msg > span { font-size: 11.5px; font-weight: 700; letter-spacing: .5px; text-transform: uppercase; color: var(--faint); padding-top: 9px; }
.msg p { margin: 0; white-space: pre-wrap; padding: 8px 12px; border-radius: 4px 12px 12px 12px; line-height: 1.55; }
.msg.u p { background: var(--surface-3); color: var(--ink); font-weight: 500; }
.msg.a > span { color: var(--accent); }
.msg.a p { background: var(--accent-50); color: #1e1b4b; box-shadow: inset 0 0 0 1px var(--accent-100); }
.msg.a p.muted { background: var(--surface-2); color: var(--muted); box-shadow: inset 0 0 0 1px var(--line-2); }
.msg.t > span { color: var(--violet); }
.msg.t p { background: var(--violet-50); font-size: 13.5px; color: #4c1d95; box-shadow: inset 0 0 0 1px #ede9fe; }
.msg textarea { margin: 0; }
.pair-foot { display: flex; gap: 6px; align-items: center; flex-wrap: wrap; padding-top: 10px; border-top: 1px solid var(--line-2); }
.score { font-size: 12px; font-weight: 600; padding: 2px 8px; border-radius: 6px; font-variant-numeric: tabular-nums; }
.score.shi { background: var(--green-50); color: var(--green-600); } .score.smid { background: #fffbeb; color: #b45309; } .score.slo { background: var(--red-50); color: var(--red); }
.ctx { font-size: 12.5px; color: var(--muted); display: inline-flex; gap: 4px; align-items: center; flex-wrap: wrap; }
.cite { font-size: 12px; padding: 1px 7px; border-radius: 5px; background: var(--surface-3); color: var(--ink-2); font-family: var(--mono); }
.cite.img { background: var(--violet-50); color: var(--violet); }
.why { font-size: 13px; color: var(--muted); font-style: italic; padding: 6px 10px; border-left: 2px solid var(--line-strong); }
.pair-act { display: flex; gap: 6px; justify-content: flex-end; }
.pager { display: flex; align-items: center; gap: 8px; padding-top: 4px; }

.bars { display: grid; gap: 9px; }
.bar-row { display: grid; grid-template-columns: minmax(0, 150px) minmax(0, 1fr) 46px; gap: 12px; align-items: center; font-size: 13.5px; }
.bar-l { color: var(--ink-2); overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.bar-t { height: 8px; background: var(--line-2); border-radius: 99px; overflow: hidden; }
.bar-t i { display: block; height: 100%; background: var(--accent); border-radius: 99px; transition: width .4s; }
.bar-v { font-weight: 600; color: var(--ink); text-align: right; font-variant-numeric: tabular-nums; }
.scorebars { display: grid; gap: 9px; }
.scorebars .bar-row { grid-template-columns: 100px minmax(0, 1fr); }
.stack { display: flex; height: 18px; border-radius: 6px; overflow: hidden; background: var(--line-2); gap: 1px; }
.stack i { display: grid; place-items: center; font-style: normal; font-size: 11px; font-weight: 600; }
.score-leg { display: flex; gap: 12px; font-size: 12px; color: var(--muted); padding: 4px 0 0 112px; }
.score-leg span::before { content: ""; display: inline-block; width: 9px; height: 9px; border-radius: 3px; margin-right: 5px; background: var(--c); vertical-align: -1px; }

.tbl { width: 100%; border-collapse: separate; border-spacing: 0; font-size: 13.5px; }
.tbl th { text-align: left; font-size: 12px; color: var(--muted); font-weight: 600; text-transform: uppercase; letter-spacing: .4px;
  padding: 8px 10px; border-bottom: 1px solid var(--line); background: var(--surface-2); }
.tbl th:first-child { border-top-left-radius: 8px; } .tbl th:last-child { border-top-right-radius: 8px; }
.tbl td { padding: 8px 10px; border-bottom: 1px solid var(--line-2); vertical-align: top; font-variant-numeric: tabular-nums; }
.tbl tbody tr:hover td { background: var(--surface-2); }
.tbl tbody tr:last-child td { border-bottom: 0; }
.tbl td:first-child { max-width: 220px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.tbl tr.bad td { color: var(--red); font-weight: 600; }

.empty { display: grid; justify-items: center; gap: 6px; padding: 72px 20px; color: var(--muted); text-align: center; }
.empty b { color: var(--ink); font-size: 16px; font-weight: 600; }
.empty-ico { width: 56px; height: 56px; border-radius: 16px; display: grid; place-items: center; background: var(--surface); color: var(--accent);
  margin-bottom: 6px; border: 1px solid var(--line); box-shadow: var(--shadow-md); }

/* ================================================================== source data */
.src-grid { display: grid; grid-template-columns: 340px minmax(0, 1fr); gap: 18px; align-items: start; }
.src-col { display: grid; gap: 18px; min-width: 0; }
.src-col .card-bd { gap: 8px; }
.sec-row { display: flex; justify-content: space-between; gap: 10px; width: 100%; padding: 7px 10px; border: 0; border-radius: 7px; background: none;
  cursor: pointer; text-align: left; font-size: 14px; color: var(--ink-2); transition: background .12s, color .12s; }
.sec-row:hover { color: var(--accent-600); background: var(--accent-50); }
div.md { background: var(--surface-2); border: 1px solid var(--line); border-radius: var(--radius); padding: 14px 16px; font: 13.5px/1.65 var(--mono);
  white-space: pre-wrap; max-height: 460px; overflow: auto; color: var(--ink-2); }
.md-h { color: var(--accent-600); font-weight: 700; }
.md-img { display: block; background: var(--violet-50); color: #5b21b6; border-radius: 8px; padding: 8px 10px; margin: 4px 0; border: 1px solid #ede9fe; }
.imgcard { display: grid; grid-template-columns: 160px minmax(0, 1fr); gap: 16px; padding: 12px; border: 1px solid var(--line); border-radius: var(--radius); background: var(--surface); }
.thumb { border: 1px solid var(--line); border-radius: 9px; background: var(--surface-2); height: 112px; overflow: hidden; }
.thumb img { width: 100%; height: 100%; object-fit: contain; display: block; }
.img-b { display: grid; gap: 6px; min-width: 0; font-size: 13.5px; }
.img-h { display: flex; gap: 8px; align-items: center; flex-wrap: wrap; }
.img-b p { margin: 0; color: var(--ink-2); }
.img-b p small { display: block; font-size: 11.5px; font-weight: 600; text-transform: uppercase; color: var(--faint); letter-spacing: .5px; margin-bottom: 1px; }
.kb { display: grid; gap: 18px; }
.overview { margin: 0; font-size: 15px; color: var(--ink-2); line-height: 1.65; }
.doccards { display: grid; grid-template-columns: repeat(auto-fill, minmax(260px, 1fr)); gap: 14px; }
.doccard { background: var(--surface); border: 1px solid var(--line); border-radius: var(--radius); padding: 14px 16px; display: grid; gap: 8px;
  font-size: 13.5px; box-shadow: var(--shadow); transition: box-shadow .15s, border-color .15s; }
.doccard:hover { border-color: var(--line-strong); box-shadow: var(--shadow-md); }
.doccard > b { font-size: 14.5px; font-weight: 600; }
.doccard p { margin: 0; color: var(--ink-2); line-height: 1.55; }
.kb-grid { display: grid; grid-template-columns: minmax(0, 1fr) minmax(0, 1.2fr); gap: 18px; align-items: start; }
.cov-doc { display: grid; gap: 8px; padding-bottom: 12px; border-bottom: 1px solid var(--line-2); }
.cov-doc:last-child { border-bottom: 0; padding-bottom: 0; }
.cov-doc > b { font-size: 14px; font-weight: 600; }

/* ================================================================== smaller screens */
@media (max-width: 1240px) {
  .cols { grid-template-columns: minmax(0, 1fr) minmax(0, 1fr); grid-template-rows: auto; }
  .cols > .card:last-child { grid-column: 1 / -1; }
  .content.fit { overflow: auto; }
  .generate { height: auto; }
  .card-bd { overflow: visible; }
  .stats { grid-template-columns: repeat(2, minmax(0, 1fr)); }
  .res-grid, .src-grid, .kb-grid { grid-template-columns: minmax(0, 1fr); }
}
@media (max-width: 900px) {
  .app { grid-template-columns: minmax(0, 1fr); }
  .sidebar { display: none; }
  .cols { grid-template-columns: minmax(0, 1fr); }
  .run-top { flex-wrap: wrap; }
  .adv { grid-template-columns: minmax(0, 1fr); }
  .adv-nav { border-right: 0; padding-right: 0; }
  .grid3 { grid-template-columns: minmax(0, 1fr) minmax(0, 1fr); }
  .topbar, .content { padding-left: 16px; padding-right: 16px; }
}
@media (max-height: 760px) and (min-width: 1241px) {
  .content.fit { overflow: auto; }
  .generate { height: auto; }
  .card-bd { overflow: visible; }
}
























//Landing.css
/* Landing page and logo placeholder. Uses the tokens from styles.css. */

/* ---- logo (the file is public/assets/logo.svg) */
.logo { display: inline-flex; align-items: center; gap: 11px; min-width: 0; color: inherit; text-decoration: none; }
.logo-img { display: block; flex: none; border-radius: 8px; object-fit: contain; }
.logo-text { display: grid; min-width: 0; line-height: 1.25; }
.logo-text b { font-size: 16px; font-weight: 650; letter-spacing: -.2px; color: var(--ink); }
.logo-text small { font-size: 13px; color: var(--muted); }

/* ---- page */
.landing { height: 100vh; overflow: auto; background: var(--surface); scroll-behavior: smooth; }
.lp-wrap { max-width: 1160px; margin: 0 auto; padding: 0 32px; }
.lp-eyebrow { display: inline-flex; align-items: center; gap: 6px; font-size: 13.5px; font-weight: 600; color: var(--accent-600); }
.btn.lg { height: 44px; padding: 0 20px; font-size: 15px; border-radius: 10px; }

/* nav */
.lp-nav { position: sticky; top: 0; z-index: 10; background: rgba(255, 255, 255, .85); backdrop-filter: saturate(1.4) blur(10px); border-bottom: 1px solid var(--line-2); }
.lp-nav-in { display: flex; align-items: center; gap: 28px; height: 68px; }
.lp-links { display: flex; gap: 4px; margin-left: auto; }
.lp-links a { padding: 7px 12px; border-radius: 8px; color: var(--ink-2); text-decoration: none; font-weight: 500; font-size: 15px; }
.lp-links a:hover { background: var(--surface-3); color: var(--ink); }

/* hero */
.lp-hero { position: relative; overflow: hidden; padding: 84px 0 96px;
  background: radial-gradient(900px 420px at 85% 10%, rgba(129, 140, 248, .16), transparent 70%),
              radial-gradient(700px 380px at 0% 100%, rgba(16, 185, 129, .08), transparent 70%), var(--surface); }
.lp-hero::before { content: ""; position: absolute; inset: 0; pointer-events: none; opacity: .5;
  background-image: linear-gradient(var(--line-2) 1px, transparent 1px), linear-gradient(90deg, var(--line-2) 1px, transparent 1px);
  background-size: 44px 44px; mask-image: radial-gradient(ellipse at 50% 0%, #000 30%, transparent 75%); }
.lp-hero-in { position: relative; display: grid; grid-template-columns: minmax(0, 1.05fr) minmax(0, 1fr); gap: 56px; align-items: center; }
.lp-hero-text .lp-eyebrow { padding: 5px 12px; border-radius: 99px; background: var(--accent-50); border: 1px solid var(--accent-100); }
.lp-hero h1 { margin: 20px 0 16px; font-size: 54px; line-height: 1.08; letter-spacing: -1.4px; font-weight: 700; }
.lp-hero h1 em { font-style: normal; background: linear-gradient(90deg, var(--accent), var(--violet)); -webkit-background-clip: text; background-clip: text; color: transparent; }
.lp-hero p { margin: 0; font-size: 18.5px; line-height: 1.6; color: var(--muted); max-width: 520px; }
.lp-cta { display: flex; gap: 10px; margin-top: 32px; flex-wrap: wrap; }

/* hero preview */
.lp-preview { position: relative; padding: 10px 28px 36px 0; }
.pv-card { background: var(--surface); border: 1px solid var(--line); border-radius: 18px; box-shadow: 0 30px 60px -20px rgba(16, 24, 40, .22), 0 0 0 6px rgba(255, 255, 255, .6);
  padding: 16px 18px 18px; display: grid; gap: 14px; }
.pv-head { display: flex; align-items: center; gap: 12px; padding-bottom: 12px; border-bottom: 1px solid var(--line-2); }
.pv-dots { display: inline-flex; gap: 5px; } .pv-dots i { width: 9px; height: 9px; border-radius: 50%; background: var(--line); }
.pv-files { display: flex; gap: 8px; flex-wrap: wrap; }
.pv-file { display: inline-flex; align-items: center; gap: 8px; padding: 6px 10px 6px 6px; border: 1px solid var(--line); border-radius: 10px; font-size: 13.5px; font-weight: 500; background: var(--surface-2); }
.pv-arrow { display: grid; place-items: center; color: var(--accent-400); transform: rotate(90deg); height: 16px; }
.pv-qa { display: grid; gap: 8px; }
.pv-q, .pv-a { padding: 10px 12px; border-radius: 4px 12px 12px 12px; font-size: 14px; line-height: 1.5; }
.pv-q small, .pv-a small { display: block; font-size: 11px; font-weight: 700; letter-spacing: .5px; text-transform: uppercase; margin-bottom: 2px; }
.pv-q { background: var(--surface-3); font-weight: 500; } .pv-q small { color: var(--faint); }
.pv-a { background: var(--accent-50); color: #1e1b4b; box-shadow: inset 0 0 0 1px var(--accent-100); } .pv-a small { color: var(--accent); }
.pv-scores { display: flex; gap: 6px; flex-wrap: wrap; }
.pv-stat { position: absolute; right: 0; bottom: 0; display: flex; gap: 10px; align-items: center; background: var(--surface); border: 1px solid var(--line);
  border-radius: 14px; padding: 10px 16px 10px 10px; box-shadow: var(--shadow-lg); }
.pv-stat b { display: block; font-size: 19.5px; letter-spacing: -.3px; } .pv-stat small { color: var(--muted); font-size: 13px; }

/* sections */
.lp-ico { width: 38px; height: 38px; border-radius: 11px; display: grid; place-items: center; background: var(--accent-50); color: var(--accent);
  box-shadow: inset 0 0 0 1px var(--accent-100); flex: none; }
.lp-ico.green { background: var(--green-50); color: var(--green-600); box-shadow: inset 0 0 0 1px #d1f2e4; }
.lp-sec { padding: 88px 0; }
.lp-sec.alt { background: var(--bg); border-top: 1px solid var(--line-2); border-bottom: 1px solid var(--line-2); }
.lp-sec-h { text-align: center; margin-bottom: 44px; }
.lp-sec-h h2 { margin: 8px 0 0; font-size: 34.5px; letter-spacing: -.8px; font-weight: 700; }
.lp-sec h3 { margin: 0 0 6px; font-size: 17.5px; font-weight: 650; letter-spacing: -.2px; }
.lp-sec p { margin: 0; color: var(--muted); font-size: 15px; line-height: 1.6; }

.lp-steps { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 20px; }
.lp-step { padding: 24px; border: 1px solid var(--line); border-radius: 16px; background: var(--surface); box-shadow: var(--shadow); }
.lp-step-top { display: flex; align-items: center; justify-content: space-between; margin-bottom: 20px; }
.lp-step-n { font-size: 14px; font-weight: 700; color: var(--faint); width: 30px; height: 30px; border-radius: 50%; display: grid; place-items: center; border: 1px solid var(--line); }

.lp-features { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 16px; }
.lp-feature { display: flex; gap: 14px; padding: 22px; border-radius: 16px; background: var(--surface); border: 1px solid var(--line); transition: box-shadow .15s, border-color .15s; }
.lp-feature:hover { border-color: var(--line-strong); box-shadow: var(--shadow-md); }

.lp-band { display: flex; align-items: center; justify-content: space-between; gap: 24px; padding: 40px 44px; border-radius: 20px;
  background: linear-gradient(120deg, var(--accent-50), #f5f3ff 60%, var(--surface)); border: 1px solid var(--accent-100); }
.lp-band h2 { margin: 0 0 6px; font-size: 28px; letter-spacing: -.6px; font-weight: 700; }

.lp-foot { border-top: 1px solid var(--line-2); padding: 26px 0; }
.lp-foot-in { display: flex; align-items: center; justify-content: space-between; gap: 16px; }

@media (max-width: 960px) {
  .lp-hero-in { grid-template-columns: minmax(0, 1fr); }
  .lp-hero h1 { font-size: 41px; }
  .lp-steps, .lp-features { grid-template-columns: minmax(0, 1fr); }
  .lp-band { flex-direction: column; align-items: flex-start; }
  .lp-links { display: none; }
  .lp-nav-in .btn { margin-left: auto; }
}
