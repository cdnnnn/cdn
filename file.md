//Sidebar.jsx
// Sidebar: the logo, the two pages and the signed-in user. It can be collapsed to icons only; the choice is remembered.
import { useEffect, useRef, useState } from "react";
import { Icon, useToast } from "./ui.jsx";
import Logo from "./Logo.jsx";
import "./sidebar.css";

const KEY = "slm-sidebar";
// a dummy user until sign-in exists
const USER = { name: "Prakash", role: "AI Department", initials: "P" };

export default function Sidebar({ route, pages }) {
  const [collapsed, setCollapsed] = useState(() => localStorage.getItem(KEY) === "collapsed");
  useEffect(() => { localStorage.setItem(KEY, collapsed ? "collapsed" : "open"); }, [collapsed]);
  // keyboard shortcut: Ctrl/Cmd + B toggles the sidebar
  useEffect(() => {
    const k = (e) => { if ((e.ctrlKey || e.metaKey) && e.key.toLowerCase() === "b") { e.preventDefault(); setCollapsed((c) => !c); } };
    window.addEventListener("keydown", k);
    return () => window.removeEventListener("keydown", k);
  }, []);

  return (
    <aside className={`sidebar${collapsed ? " is-collapsed" : ""}`}>
      <div className="sb-top">
        <a className="sb-brand" href="#/" title="Back to the home page"><Logo size={42} /></a>
      </div>
      <button type="button" className="sb-toggle" onClick={() => setCollapsed((c) => !c)}
        aria-label={collapsed ? "Expand sidebar" : "Collapse sidebar"} aria-expanded={!collapsed} data-tip={collapsed ? "Expand (Ctrl+B)" : "Collapse (Ctrl+B)"}>
        <Icon name="chevron" size={14} stroke={2.2} />
      </button>

      <nav className="sb-nav" style={{ "--i": Math.max(0, Object.keys(pages).indexOf(route)) }}>
        <span className="sb-ind" aria-hidden="true" />
        {Object.entries(pages).map(([k, p]) => (
          <a key={k} className={route === k ? "on" : ""} href={`#/${k}`} data-tip={p.nav} aria-label={p.nav}>
            <span className="sb-nav-ico"><Icon name={p.icon} size={17} /></span><span className="sb-label">{p.nav}</span>
          </a>
        ))}
      </nav>

      <UserCard collapsed={collapsed} />
    </aside>
  );
}

function UserCard({ collapsed }) {
  const toast = useToast();
  const [open, setOpen] = useState(false);
  const ref = useRef(null);
  useEffect(() => {
    if (!open) return undefined;
    const h = (e) => { if (!ref.current?.contains(e.target)) setOpen(false); };
    const k = (e) => { if (e.key === "Escape") setOpen(false); };
    document.addEventListener("mousedown", h); document.addEventListener("keydown", k);
    return () => { document.removeEventListener("mousedown", h); document.removeEventListener("keydown", k); };
  }, [open]);
  const demo = (what) => { setOpen(false); toast(`${what} is not available in this demo yet`); };
  return (
    <div className="sb-user" ref={ref}>
      {open && (
        <div className="sb-user-menu" role="menu">
          <div className="sb-user-menu-h"><span className="sb-avatar sm">{USER.initials}</span><div><b>{USER.name}</b><small>{USER.role}</small></div></div>
          <button type="button" role="menuitem" onClick={() => demo("Your profile")}><Icon name="eye" size={15} />Your profile</button>
          <button type="button" role="menuitem" onClick={() => demo("Settings")}><Icon name="gear" size={15} />Settings</button>
          <div className="sb-user-sep" />
          <button type="button" role="menuitem" className="danger" onClick={() => demo("Sign out")}><Icon name="back" size={15} />Sign out</button>
        </div>
      )}
      <button type="button" className={`sb-user-btn${open ? " on" : ""}`} onClick={() => setOpen((o) => !o)} aria-haspopup="menu" aria-expanded={open}
        data-tip={collapsed && !open ? USER.name : undefined}>
        <span className="sb-avatar">{USER.initials}<i className="sb-online" aria-label="Online" /></span>
        <span className="sb-user-t sb-label"><b>{USER.name}</b><small>{USER.role}</small></span>
        <span className="sb-user-chev sb-label"><Icon name="chevron" size={14} /></span>
      </button>
    </div>
  );
}
































//sidebar.css
/* Sidebar extras: collapse to icons only, tooltips while collapsed, and the user card at the bottom.
   Selectors start with "aside.sidebar" so they win over the base sidebar styles whatever order the CSS loads in. */

/* ---- width: the app grid animates between the two widths */
div.app { transition: grid-template-columns .38s cubic-bezier(.22, 1, .36, 1); }
div.app:has(> aside.sidebar.is-collapsed) { grid-template-columns: 78px minmax(0, 1fr); }
aside.sidebar { overflow: visible; position: relative; }

/* labels and the word mark fold away smoothly */
aside.sidebar .sb-label { display: inline-block; max-width: 180px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
  transition: max-width .38s cubic-bezier(.22, 1, .36, 1), opacity .2s ease .08s, transform .3s ease; }
aside.sidebar .slm-word { max-width: 200px; overflow: hidden; transition: max-width .38s cubic-bezier(.22, 1, .36, 1), opacity .2s ease .08s; }
aside.sidebar .slm-logo { transition: gap .38s cubic-bezier(.22, 1, .36, 1); }
aside.sidebar .sb-top { transition: padding .38s cubic-bezier(.22, 1, .36, 1), margin .38s; }
aside.sidebar.is-collapsed .sb-label { max-width: 0; opacity: 0; transform: translateX(-6px); transition: max-width .38s cubic-bezier(.22, 1, .36, 1), opacity .12s ease, transform .2s ease; }
aside.sidebar.is-collapsed .slm-word { max-width: 0; opacity: 0; transition: max-width .38s cubic-bezier(.22, 1, .36, 1), opacity .12s ease; }
aside.sidebar.is-collapsed .slm-logo { gap: 0; }
aside.sidebar.is-collapsed .sb-top { padding-left: 0; padding-right: 0; margin: 0; justify-content: center; }
aside.sidebar.is-collapsed .sb-nav a { gap: 0; padding: 0 9px; }

/* ---- the collapse button: a small round handle on the sidebar's edge */
aside.sidebar .sb-toggle { position: absolute; top: 34px; right: -13px; z-index: 5; width: 26px; height: 26px; border-radius: 50%; display: grid; place-items: center;
  border: 1px solid var(--line); background: var(--surface); color: var(--muted); cursor: pointer; padding: 0;
  box-shadow: 0 1px 2px rgba(16, 24, 40, .08), 0 4px 10px -4px rgba(16, 24, 40, .18);
  transition: color .15s, border-color .15s, box-shadow .2s, transform .2s, opacity .2s; }
aside.sidebar .sb-toggle svg { transform: rotate(180deg); transition: transform .38s cubic-bezier(.22, 1, .36, 1); }
aside.sidebar.is-collapsed .sb-toggle svg { transform: rotate(0deg); }
aside.sidebar .sb-toggle:hover { color: var(--accent-600); border-color: var(--accent-200); transform: scale(1.08);
  box-shadow: 0 0 0 4px rgba(var(--accent-rgb), .1), 0 4px 12px -4px rgba(var(--accent-rgb), .45); }
aside.sidebar .sb-toggle:focus-visible { outline: none; box-shadow: var(--ring); }

/* ---- tooltips: nav items and the user show their name while collapsed; the handle always explains itself */
aside.sidebar [data-tip]:not(.sb-toggle) { position: relative; }
aside.sidebar.is-collapsed [data-tip]::after, aside.sidebar .sb-toggle[data-tip]::after {
  content: attr(data-tip); position: absolute; left: calc(100% + 14px); top: 50%; z-index: 30; pointer-events: none; white-space: nowrap;
  padding: 6px 10px; border-radius: 8px; font-size: 12.5px; font-weight: 600; color: var(--dark-btn-ink); background: var(--dark-btn-b);
  box-shadow: 0 8px 20px -6px rgba(16, 24, 40, .35); opacity: 0; transform: translate(-4px, -50%); transition: opacity .15s, transform .2s; }
aside.sidebar.is-collapsed [data-tip]:hover::after, aside.sidebar.is-collapsed [data-tip]:focus-visible::after,
aside.sidebar .sb-toggle[data-tip]:hover::after { opacity: 1; transform: translate(0, -50%); transition-delay: .12s; }
aside.sidebar .sb-toggle[data-tip]::after { left: calc(100% + 10px); }

/* ---- user card at the bottom */
.sb-user { position: relative; margin-top: auto; padding-top: 12px; border-top: 1px solid var(--line-2); }
.sb-user-btn { display: flex; align-items: center; gap: 10px; width: 100%; padding: 8px; border-radius: 12px; border: 1px solid transparent; background: none;
  cursor: pointer; text-align: left; font: inherit; color: var(--ink); transition: background .15s, border-color .15s, padding .38s cubic-bezier(.22, 1, .36, 1), gap .38s; }
.sb-user-btn:hover, .sb-user-btn.on { background: var(--surface); border-color: var(--line); box-shadow: var(--shadow-xs); }
aside.sidebar.is-collapsed .sb-user-btn { gap: 0; padding: 8px 7px; }
.sb-avatar { position: relative; width: 34px; height: 34px; border-radius: 50%; flex: none; display: grid; place-items: center; font-size: 14px; font-weight: 700; color: #fff;
  background: var(--grad); box-shadow: 0 0 0 2px var(--surface), 0 0 0 3px rgba(var(--accent-rgb), .25), 0 4px 10px -4px rgba(var(--accent-rgb), .6); }
.sb-avatar.sm { width: 30px; height: 30px; font-size: 13px; box-shadow: none; }
.sb-online { position: absolute; right: -1px; bottom: -1px; width: 10px; height: 10px; border-radius: 50%; background: var(--green); border: 2px solid var(--surface); }
aside.sidebar .sb-user-t { display: grid; gap: 1px; flex: 1; min-width: 0; }
.sb-user-t b { font-size: 14px; font-weight: 600; overflow: hidden; text-overflow: ellipsis; }
.sb-user-t small { font-size: 12px; color: var(--muted); overflow: hidden; text-overflow: ellipsis; }
.sb-user-chev { color: var(--faint); display: grid; place-items: center; }
.sb-user-chev svg { transform: rotate(-90deg); transition: transform .25s; }
.sb-user-btn.on .sb-user-chev svg { transform: rotate(90deg); }

.sb-user-menu { position: absolute; left: 0; bottom: calc(100% + 8px); width: 232px; z-index: 40; padding: 6px; border-radius: 14px; background: var(--surface);
  border: 1px solid var(--line); box-shadow: var(--shadow-lg); display: grid; gap: 1px; transform-origin: bottom left; animation: sb-pop .22s cubic-bezier(.22, 1, .36, 1); }
@keyframes sb-pop { from { opacity: 0; transform: translateY(6px) scale(.97); } }
.sb-user-menu-h { display: flex; align-items: center; gap: 10px; padding: 8px 8px 10px; margin-bottom: 4px; border-bottom: 1px solid var(--line-2); }
.sb-user-menu-h b { display: block; font-size: 14px; }
.sb-user-menu-h small { font-size: 12px; color: var(--muted); }
.sb-user-menu button { display: flex; align-items: center; gap: 10px; padding: 8px 10px; border: 0; border-radius: 8px; background: none; cursor: pointer;
  font: inherit; font-size: 14px; color: var(--ink-2); text-align: left; transition: background .12s, color .12s; }
.sb-user-menu button svg { color: var(--faint); }
.sb-user-menu button:hover { background: var(--surface-3); color: var(--ink); }
.sb-user-menu button.danger:hover { background: var(--red-50); color: var(--red); } .sb-user-menu button.danger:hover svg { color: var(--red); }
.sb-user-sep { height: 1px; background: var(--line-2); margin: 4px 6px; }

@media (prefers-reduced-motion: reduce) { div.app, aside.sidebar * { transition: none !important; } }
