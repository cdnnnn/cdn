//jobs.js
// Live list of every job, shared by the sidebar badge, the Jobs page and the "job finished" toasts.
// One poller for the whole app: every 3 s while a job is queued or running, every 15 s otherwise.
import { useCallback, useEffect, useSyncExternalStore } from "react";
import { api, post } from "../api.js";
import { STEP_TITLES } from "../model.js";
import { useBus } from "../bus.js";
import { useToast } from "../ui.jsx";
import "./jobs.css";

export const isActive = (s) => s === "queued" || s === "running";

let state = { jobs: null, details: {}, error: null };
const subs = new Set();
const setState = (p) => { state = { ...state, ...p }; subs.forEach((f) => f()); };
const subscribe = (f) => { subs.add(f); return () => subs.delete(f); };
export const useJobs = () => useSyncExternalStore(subscribe, () => state);

let timer = null;
let running = false;
let onFinish = null;

export async function refreshJobs() {
  if (running) return;
  running = true;
  clearTimeout(timer);
  try {
    const list = await api("/jobs");
    const prev = state.jobs;
    const details = { ...state.details };
    // step-by-step progress of the jobs that are still going, and a last look at the ones that just ended
    const ended = prev ? list.filter((j) => isActive(prev.find((x) => x.id === j.id)?.status) && !isActive(j.status)) : [];
    await Promise.all([...list.filter((j) => isActive(j.status)), ...ended].map(async (j) => {
      try { details[j.id] = await api(`/jobs/${encodeURIComponent(j.id)}`); } catch { /* keep the last one */ }
    }));
    setState({ jobs: list, details, error: null });
    ended.forEach((j) => onFinish?.(j, details[j.id]));
  } catch (e) {
    setState({ jobs: state.jobs || [], error: e.message });
  } finally {
    running = false;
    const busy = !state.error && (state.jobs || []).some((j) => isActive(j.status));
    timer = setTimeout(refreshJobs, busy ? 3000 : 15000);
  }
}

// mounted once in the app shell: starts the poller and shows a toast when a job ends, whatever page is open
export function JobWatcher() {
  const toast = useToast();
  useEffect(() => {
    onFinish = (j, d) => {
      const name = j.name || j.id;
      if (j.status === "done") {
        const n = d?.counts?.pairs_kept;
        toast(`"${name}" finished${n != null ? `: ${Number(n).toLocaleString()} rows kept` : ""}. Open Jobs or Results to see it.`);
      } else if (j.status === "failed") toast(`"${name}" failed${j.error ? `: ${j.error}` : ""}`, "error");
      else if (j.status === "cancelled") toast(`"${name}" was cancelled`);
    };
    refreshJobs();
    return () => { onFinish = null; clearTimeout(timer); };
  }, [toast]);
  // the Generate page says when it starts, cancels or retries a job: look again right away
  useBus("jobs-changed", useCallback(() => { setTimeout(refreshJobs, 400); }, []));
  return null;
}

// ---- small helpers for the page
export const stepTitleOf = (key) => STEP_TITLES.find(([k]) => k === key)?.[1] || key || "";

// how far a job is: { done, total, pct, label }
export function progressOf(j, d) {
  const steps = d?.steps?.length ? d.steps : null;
  const total = steps ? steps.length : STEP_TITLES.length;
  if (j.status === "done") return { done: total, total, pct: 100, label: "All steps done" };
  let done;
  if (steps) done = steps.filter((s) => s.status === "done" || s.status === "skipped").length;
  else done = Math.max(0, STEP_TITLES.findIndex(([k]) => k === j.current_step));
  const pct = Math.round((100 * done) / total);
  if (j.status === "queued") return { done, total, pct, label: "Waiting to start" };
  const cur = stepTitleOf(j.current_step);
  if (j.status === "running") return { done, total, pct, label: `Step ${Math.min(done + 1, total)} of ${total}${cur ? ` · ${cur}` : ""}` };
  return { done, total, pct, label: `Stopped at step ${Math.min(done + 1, total)} of ${total}${cur ? ` · ${cur}` : ""}` };
}

export const cancelJob = (id) => post(`/jobs/${encodeURIComponent(id)}/cancel`);
export const retryJob = (id) => post(`/jobs/${encodeURIComponent(id)}/retry`);


















//JobsPage.jsx
// Jobs page: every dataset job with its live status, progress, start time and duration, and what you can do with it.
import { useEffect, useState } from "react";
import { Icon, useToast } from "../ui.jsx";
import { emit } from "../bus.js";
import { cancelJob, isActive, progressOf, refreshJobs, retryJob, useJobs } from "./jobs.js";

const STATUS = {
  running: { label: "Running", cls: "run", icon: "refresh" },
  queued: { label: "Queued", cls: "wait", icon: "clock" },
  done: { label: "Finished", cls: "ok", icon: "check" },
  failed: { label: "Failed", cls: "bad", icon: "alert" },
  cancelled: { label: "Cancelled", cls: "off", icon: "stop" },
};
const FILTERS = [
  ["all", "All", () => true],
  ["active", "In progress", (j) => isActive(j.status)],
  ["done", "Finished", (j) => j.status === "done"],
  ["failed", "Failed", (j) => j.status === "failed"],
  ["cancelled", "Cancelled", (j) => j.status === "cancelled"],
];

const toDate = (s) => (s ? new Date(/[zZ]|[+-]\d\d:?\d\d$/.test(s) ? s : `${s}Z`) : null);
function when(s) {
  const d = toDate(s);
  if (!d || isNaN(d)) return "";
  const today = new Date(); const y = new Date(); y.setDate(today.getDate() - 1);
  const t = d.toLocaleTimeString([], { hour: "2-digit", minute: "2-digit" });
  if (d.toDateString() === today.toDateString()) return `Today, ${t}`;
  if (d.toDateString() === y.toDateString()) return `Yesterday, ${t}`;
  return `${d.toLocaleDateString([], { day: "numeric", month: "short" })}, ${t}`;
}
function took(ms) {
  if (!(ms >= 0)) return "";
  const s = Math.round(ms / 1000), h = Math.floor(s / 3600), m = Math.floor((s % 3600) / 60);
  return h ? `${h} h ${m} min` : m ? `${m} min ${s % 60} s` : `${s} s`;
}

const openOn = (route, id) => {
  const u = new URL(location.href);
  u.searchParams.set("job", id);
  if (route === "results") localStorage.setItem("slm-results-job", id);
  u.hash = `#/${route}`;
  location.href = u.toString();
};

export default function JobsPage() {
  const { jobs, details, error } = useJobs();
  const [filter, setFilter] = useState("all");
  const [q, setQ] = useState("");
  const [, tick] = useState(0);
  const anyActive = (jobs || []).some((j) => isActive(j.status));
  // running durations count up every second
  useEffect(() => { if (!anyActive) return undefined; const t = setInterval(() => tick((x) => x + 1), 1000); return () => clearInterval(t); }, [anyActive]);
  useEffect(() => { refreshJobs(); }, []);

  const list = jobs || [];
  const count = (k) => list.filter(FILTERS.find((f) => f[0] === k)[2]).length;
  const shown = list.filter(FILTERS.find((f) => f[0] === filter)[2])
    .filter((j) => !q.trim() || `${j.name} ${j.id}`.toLowerCase().includes(q.trim().toLowerCase()));

  return (
    <div className="jobs">
      <div className="jb-stats">
        {[["active", "In progress", "refresh", "run"], ["done", "Finished", "check", "ok"], ["failed", "Failed", "alert", "bad"], ["cancelled", "Cancelled", "stop", "off"]].map(([k, l, ic, cls]) => (
          <button type="button" key={k} className={`jb-stat ${cls}${filter === k ? " on" : ""}`} onClick={() => setFilter(filter === k ? "all" : k)}>
            <span className="jb-stat-ico"><Icon name={ic} size={16} /></span>
            <span><b className="num">{jobs ? count(k) : "–"}</b><small>{l}</small></span>
          </button>
        ))}
      </div>

      <section className="card jb-card">
        <div className="jb-bar">
          <div className="pilltabs" role="tablist" aria-label="Filter jobs by status">
            {FILTERS.map(([k, l]) => (
              <button type="button" role="tab" key={k} aria-selected={filter === k} className={filter === k ? "on" : ""} onClick={() => setFilter(k)}>
                {l}<span className="num">{jobs ? count(k) : 0}</span>
              </button>
            ))}
          </div>
          <div className="grow" />
          <label className="search"><Icon name="search" size={15} /><input value={q} onChange={(e) => setQ(e.target.value)} placeholder="Search by name or id" /></label>
          <button type="button" className="btn ghost small" onClick={() => refreshJobs()} title="Refresh now"><Icon name="refresh" size={14} />Refresh</button>
        </div>

        {anyActive && (
          <p className="jb-note"><Icon name="clock" size={14} />The list updates by itself every few seconds. The server runs one job at a time, so start the next one when this one has finished.</p>
        )}
        {error && <p className="jb-note bad"><Icon name="alert" size={14} />Could not reach the API: {error}. Trying again shortly.</p>}

        {!jobs ? <div className="jb-empty"><span className="spin" />Loading jobs…</div>
          : shown.length === 0 ? (
            <div className="jb-empty">
              <span className="jb-empty-ico"><Icon name="layers" size={22} /></span>
              <b>{list.length ? "No jobs match" : "No jobs yet"}</b>
              <small>{list.length ? "Try another filter or search." : "Jobs you start on Generate Dataset show up here."}</small>
              {!list.length && <a className="btn primary" href="#/generate"><Icon name="plus" size={14} />New dataset</a>}
            </div>
          ) : (
            <ul className="jb-list">
              {shown.map((j) => <JobRow key={j.id} j={j} d={details[j.id]} />)}
            </ul>
          )}
      </section>
    </div>
  );
}

function JobRow({ j, d }) {
  const toast = useToast();
  const [busy, setBusy] = useState(false);
  const st = STATUS[j.status] || STATUS.queued;
  const p = progressOf(j, d);
  const start = toDate(j.created), end = isActive(j.status) ? new Date() : toDate(j.updated);
  const dur = start && end ? took(end - start) : "";
  const kept = d?.counts?.pairs_kept;
  const act = async (fn, okText) => {
    setBusy(true);
    try { await fn(); toast(okText); emit("jobs-changed"); await refreshJobs(); } catch (e) { toast(e.message, "error"); } finally { setBusy(false); }
  };
  return (
    <li className={`jb-row ${st.cls}`}>
      <span className="jb-ico"><Icon name={st.icon} size={16} className={j.status === "running" ? "spin-slow" : ""} /></span>
      <div className="jb-main">
        <div className="jb-top">
          <b className="jb-name" title={j.name}>{j.name || j.id}</b>
          <span className="jb-id num">{j.id}</span>
          {j.mode && <span className="jb-mode">{j.mode === "reasoning" ? "Reasoning" : "Chat"}</span>}
          <span className={`jb-status ${st.cls}`}><i />{st.label}</span>
        </div>
        <div className="jb-meta">
          <span><Icon name="clock" size={13} />Started {when(j.created) || "–"}</span>
          {dur && <span>{isActive(j.status) ? (j.status === "queued" ? `Waiting ${dur}` : `Running for ${dur}`) : `Took ${dur}`}</span>}
          {j.status === "done" && kept != null && <span><b className="num">{Number(kept).toLocaleString()}</b> rows kept</span>}
        </div>
        <div className="jb-prog">
          <div className="jb-track" role="progressbar" aria-valuenow={p.pct} aria-valuemin={0} aria-valuemax={100} aria-label={`${j.name} progress`}>
            <i style={{ width: `${Math.max(p.pct, j.status === "running" ? 3 : 0)}%` }} />
          </div>
          <span className="jb-pct num">{p.pct}%</span>
          <span className="jb-prog-t">{p.label}</span>
        </div>
        {j.status === "failed" && j.error && <p className="jb-err" title={j.error}>{j.error}</p>}
      </div>
      <div className="jb-acts">
        {j.status === "done" && <button type="button" className="btn small primary" onClick={() => openOn("results", j.id)}><Icon name="results" size={13} />Open results</button>}
        {isActive(j.status) && <button type="button" className="btn small" onClick={() => openOn("generate", j.id)}><Icon name="eye" size={13} />Follow</button>}
        {isActive(j.status) && <button type="button" className="btn small ghost danger" disabled={busy} onClick={() => act(() => cancelJob(j.id), "Cancel requested: the job stops after the current item")}><Icon name="stop" size={13} />Cancel</button>}
        {(j.status === "failed" || j.status === "cancelled") && <button type="button" className="btn small" disabled={busy} onClick={() => act(() => retryJob(j.id), "Job re-queued: finished steps are skipped")}><Icon name="repeat" size={13} />Retry</button>}
        {!isActive(j.status) && <button type="button" className="btn small ghost" onClick={() => openOn("generate", j.id)} title="See the steps and settings of this job"><Icon name="list" size={13} />Details</button>}
      </div>
    </li>
  );
}




















//jobs.css
/* Jobs page: status tiles, filter bar and the live list of jobs. Colours come from the theme tokens,
   so every palette and light/dark mode work; the "in progress" colour is the palette accent. */
.jobs { display: grid; gap: 18px; max-width: 1360px; margin: 0 auto; }

/* ---- status tiles (also filters) */
.jb-stats { display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); gap: 12px; }
.jb-stat { --c: var(--accent-600); --bg: var(--accent-50); display: flex; align-items: center; gap: 12px; padding: 14px 16px; border-radius: 14px;
  border: 1px solid var(--line); background: var(--surface); box-shadow: var(--shadow-xs); cursor: pointer; text-align: left; font: inherit; color: var(--ink);
  transition: border-color .15s, box-shadow .2s, transform .2s; }
.jb-stat:hover { transform: translateY(-1px); border-color: var(--line-strong); }
.jb-stat.on { border-color: var(--c); box-shadow: 0 0 0 1px var(--c), var(--shadow-xs); }
.jb-stat.ok { --c: var(--green-600); --bg: var(--green-50); }
.jb-stat.bad { --c: var(--red); --bg: var(--red-50); }
.jb-stat.off { --c: var(--muted); --bg: var(--surface-3); }
.jb-stat-ico { width: 36px; height: 36px; border-radius: 10px; display: grid; place-items: center; background: var(--bg); color: var(--c); flex: none; }
.jb-stat b { display: block; font-size: 22px; font-weight: 700; line-height: 1.1; }
.jb-stat small { font-size: 12.5px; color: var(--muted); font-weight: 550; }

/* ---- list card */
.jb-card { padding: 0; overflow: hidden; }
.jb-bar { display: flex; align-items: center; gap: 10px; padding: 14px 16px; border-bottom: 1px solid var(--line-2); flex-wrap: wrap; }
.jb-bar .search input { width: 200px !important; }
.jb-note { display: flex; align-items: center; gap: 8px; margin: 0; padding: 10px 16px; font-size: 13px; color: var(--muted); background: var(--surface-2); border-bottom: 1px solid var(--line-2); }
.jb-note.bad { color: var(--red); background: var(--red-50); }

.jb-list { list-style: none; margin: 0; padding: 0; }
.jb-row { --c: var(--accent); display: grid; grid-template-columns: auto minmax(0, 1fr) auto; gap: 14px; align-items: center; padding: 16px; border-bottom: 1px solid var(--line-2);
  position: relative; animation: jb-in .35s cubic-bezier(.22, 1, .36, 1) both; }
.jb-row:last-child { border-bottom: 0; }
.jb-row::before { content: ""; position: absolute; left: 0; top: 12px; bottom: 12px; width: 3px; border-radius: 0 3px 3px 0; background: var(--c); opacity: .9; }
.jb-row:hover { background: var(--surface-2); }
.jb-row.ok { --c: var(--green); }
.jb-row.bad { --c: var(--red); }
.jb-row.off, .jb-row.wait { --c: var(--line-strong); }
@keyframes jb-in { from { opacity: 0; transform: translateY(4px); } }

.jb-ico { width: 36px; height: 36px; border-radius: 50%; display: grid; place-items: center; background: var(--accent-50); color: var(--accent-600); }
.jb-row.ok .jb-ico { background: var(--green-50); color: var(--green-600); }
.jb-row.bad .jb-ico { background: var(--red-50); color: var(--red); }
.jb-row.off .jb-ico, .jb-row.wait .jb-ico { background: var(--surface-3); color: var(--muted); }
.spin-slow { animation: jb-spin 1.6s linear infinite; }
@keyframes jb-spin { to { transform: rotate(360deg); } }

.jb-main { display: grid; gap: 6px; min-width: 0; }
.jb-top { display: flex; align-items: center; gap: 8px; min-width: 0; flex-wrap: wrap; }
.jb-name { font-size: 15px; font-weight: 650; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; max-width: 360px; }
.jb-id { font-family: var(--mono); font-size: 12px; color: var(--muted); }
.jb-mode { font-size: 11.5px; font-weight: 600; padding: 1px 8px; border-radius: 99px; border: 1px solid var(--line); color: var(--muted); }
.jb-status { --c: var(--accent-600); --bg: var(--accent-50); display: inline-flex; align-items: center; gap: 6px; margin-left: auto; font-size: 12.5px; font-weight: 650;
  padding: 2px 10px; border-radius: 99px; color: var(--c); background: var(--bg); }
.jb-status i { width: 7px; height: 7px; border-radius: 50%; background: currentColor; }
.jb-status.run i { animation: pulse 1.6s infinite; }
.jb-status.ok { --c: var(--green-600); --bg: var(--green-50); }
.jb-status.bad { --c: var(--red); --bg: var(--red-50); }
.jb-status.off, .jb-status.wait { --c: var(--muted); --bg: var(--surface-3); }

.jb-meta { display: flex; flex-wrap: wrap; gap: 4px 16px; font-size: 13px; color: var(--muted); }
.jb-meta span { display: inline-flex; align-items: center; gap: 5px; }
.jb-meta b { color: var(--ink-2); }
.jb-prog { display: grid; grid-template-columns: minmax(120px, 360px) 36px minmax(0, 1fr); align-items: center; gap: 10px; }
.jb-track { height: 6px; border-radius: 99px; background: var(--line-2); overflow: hidden; }
.jb-track i { display: block; height: 100%; border-radius: 99px; background: var(--grad); transition: width .6s cubic-bezier(.22, 1, .36, 1); }
.jb-row.run .jb-track i { background-size: 200% 100%; animation: jb-flow 2.2s linear infinite; }
.jb-row.ok .jb-track i { background: var(--green); }
.jb-row.bad .jb-track i { background: var(--red); }
.jb-row.off .jb-track i, .jb-row.wait .jb-track i { background: var(--line-strong); }
@keyframes jb-flow { to { background-position: -200% 0; } }
.jb-prog-t { font-size: 12.5px; color: var(--ink-2); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.jb-pct { font-size: 12.5px; font-weight: 650; color: var(--ink-2); text-align: right; }
.jb-err { margin: 0; font-size: 12.5px; color: var(--red); display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; overflow: hidden; }
.jb-acts { display: flex; gap: 6px; flex-wrap: wrap; justify-content: flex-end; }

.jb-empty { display: grid; justify-items: center; gap: 6px; padding: 48px 20px; text-align: center; color: var(--muted); }
.jb-empty b { color: var(--ink); font-size: 15px; }
.jb-empty .btn { margin-top: 8px; }
.jb-empty-ico { width: 46px; height: 46px; border-radius: 14px; display: grid; place-items: center; background: var(--accent-50); color: var(--accent-600); }
.jb-empty .spin { width: 18px; height: 18px; border-radius: 50%; border: 2px solid var(--line); border-top-color: var(--accent); animation: jb-spin .8s linear infinite; }

@media (max-width: 1100px) {
  .jb-stats { grid-template-columns: repeat(2, minmax(0, 1fr)); }
  .jb-row { grid-template-columns: auto minmax(0, 1fr); }
  .jb-acts { grid-column: 2; justify-content: flex-start; }
}
@media (prefers-reduced-motion: reduce) { .spin-slow, .jb-row.run .jb-track i, .jb-row { animation: none; } }

/* ---- sidebar: running-jobs badge on the Jobs item */
.sb-badge { margin-left: auto; min-width: 20px; height: 20px; padding: 0 6px; border-radius: 99px; display: inline-grid; place-items: center; font-size: 11.5px; font-weight: 700;
  color: #fff; background: var(--grad); box-shadow: 0 2px 6px -2px rgba(var(--accent-rgb), .6); animation: sb-badge-in .3s cubic-bezier(.22, 1, .36, 1); }
@keyframes sb-badge-in { from { transform: scale(.4); opacity: 0; } }
aside.sidebar.is-collapsed .sb-badge { position: absolute; top: 5px; left: 34px; min-width: 16px; height: 16px; padding: 0 4px; font-size: 10px; margin: 0;
  box-shadow: 0 0 0 2px var(--surface); }















  


//App.jsx
// Routing: the landing page at #/ (the first screen), and the app shell with sidebar and top bar at #/generate, #/jobs and #/results.
// Features are added one piece at a time; see README.md for the build order.
import { useCallback, useEffect, useState } from "react";
import { Icon } from "./ui.jsx";
import Landing from "./Landing.jsx";
import Sidebar from "./Sidebar.jsx";
import { emit, lastOf, useBus } from "./bus.js";
import GeneratePage, { STEPS } from "./generate/GeneratePage.jsx";
import "./shell.css";
import "./rich.css";
import "./theme-dark.css";
import "./fields.css";
import ThemeToggle from "./ThemeToggle.jsx";
import { CustomizeButton } from "./Customize.jsx";
import ResultsPage from "./results/ResultsPage.jsx";
import JobsPage from "./jobs/JobsPage.jsx";
import { JobWatcher } from "./jobs/jobs.js";

const PAGES = {
  generate: { label: "Generate", nav: "Generate Dataset", icon: "spark", title: "Generate Dataset",
    sub: "Turn your documents into grounded question-and-answer pairs for fine-tuning." },
  jobs: { label: "Jobs", nav: "Jobs", icon: "list", title: "Jobs",
    sub: "Every dataset job and how it is doing. The list updates by itself." },
  results: { label: "Results", nav: "Results", icon: "results", title: "Results",
    sub: "Review the generated pairs, their scores, and download the dataset." },
};
// "#how" and "#features" are anchors on the landing page, so anything that is not an app page shows the landing page.
const routeFromHash = () => { const r = (location.hash || "").replace(/^#\/?/, ""); return PAGES[r] ? r : "home"; };

export default function App() {
  const [route, setRoute] = useState(routeFromHash);
  useEffect(() => {
    const h = () => setRoute(routeFromHash());
    window.addEventListener("hashchange", h);
    return () => window.removeEventListener("hashchange", h);
  }, []);
  if (route === "home") return <Landing />;
  const page = PAGES[route];

  return (
    <div className="app">
      <JobWatcher />
      <Sidebar route={route} pages={PAGES} />

      <main className="main">
        <PageHeader route={route} page={page} />
        <div className="content" key={route}>
          {route === "generate" ? <GeneratePage openResults={() => { location.hash = "#/results"; }} /> : route === "jobs" ? <JobsPage /> : <ResultsPage />}
        </div>
      </main>
    </div>
  );
}

// Page header: breadcrumb, title and one line of context on the left; the state of the current job and actions on the right.
function PageHeader({ route, page }) {
  const [job, setJob] = useState(() => lastOf("job") || null);
  useBus("job", useCallback((j) => setJob(j), []));
  const [rjob, setRjob] = useState(() => lastOf("results-job") || null);
  useBus("results-job", useCallback((j) => setRjob(j), []));
  const onGen = route === "generate";
  const open = onGen && job?.id;
  return (
    <header className="ph">
      <div className="ph-main">
        <span className="ph-ico"><Icon name={page.icon} size={20} /></span>
        <div className="ph-t">
          <div className="ph-crumbs">
            <a href="#/">SLM Studio</a><Icon name="chevron" size={12} />
            <a href={`#/${route}`}>{page.nav}</a>
            {onGen && <><Icon name="chevron" size={12} />{open ? <span className="num">{job.id}</span> : <span>New job</span>}</>}
            {route === "results" && rjob?.id && <><Icon name="chevron" size={12} /><span className="num">{rjob.name || rjob.id}</span></>}
          </div>
          <h1>{page.title}</h1>
          <p>{open ? <>Following <b>{job.name || job.id}</b>. You can leave this page; the job keeps running on the server.</> : page.sub}</p>
        </div>
      </div>
      <div className="ph-side">
        {onGen && !open && job && (
          <span className="ph-state">
            <span className="ph-state-l">Draft</span>
            <span className="ph-state-v">Step <b className="num">{(job.step ?? 0) + 1}</b> of {STEPS.length}</span>
            <span className={`ph-state-dot ${job.ready ? "ok" : ""}`} title={job.ready ? "Ready to start" : "Something still needs your input"} />
          </span>
        )}
        {open && <button type="button" className="btn" onClick={() => emit("new-job")}><Icon name="plus" size={14} />New job</button>}
        {onGen && !open && <a className="btn ghost" href="#/results"><Icon name="results" size={14} />Results</a>}
        {!onGen && <a className="btn" href="#/generate"><Icon name="plus" size={14} />New dataset</a>}
        <span className="ph-sep" aria-hidden="true" />
        <CustomizeButton />
        <ThemeToggle />
      </div>
    </header>
  );
}














//Sidebar.jsx
// Sidebar: the logo, the pages (Jobs shows how many jobs are in progress) and the signed-in user. It can be collapsed to icons only; the choice is remembered.
import { useEffect, useRef, useState } from "react";
import { openCustomize } from "./Customize.jsx";
import { Icon, useToast } from "./ui.jsx";
import Logo from "./Logo.jsx";
import { isActive, useJobs } from "./jobs/jobs.js";
import "./sidebar.css";

const KEY = "slm-sidebar";
// a dummy user until sign-in exists
const USER = { name: "Prakash", role: "AI Department", initials: "P" };

export default function Sidebar({ route, pages }) {
  const { jobs } = useJobs();
  const active = (jobs || []).filter((j) => isActive(j.status)).length;
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
          <a key={k} className={route === k ? "on" : ""} href={`#/${k}`} data-tip={k === "jobs" && active ? `${p.nav} · ${active} in progress` : p.nav}
            aria-label={k === "jobs" && active ? `${p.nav}, ${active} in progress` : p.nav}>
            <span className="sb-nav-ico"><Icon name={p.icon} size={17} /></span><span className="sb-label">{p.nav}</span>
            {k === "jobs" && active > 0 && <span className="sb-badge num" title={`${active} job${active > 1 ? "s" : ""} in progress`}>{active}</span>}
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
          <button type="button" role="menuitem" onClick={() => { setOpen(false); openCustomize(); }}><Icon name="palette" size={15} />Customize</button>
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

















//ResultsPage.jsx
// Results page as a guided, step-by-step flow (like Generate): pick a dataset, see the overview,
// review the pairs, look at the insights if you like, then accept and download.
import { useCallback, useEffect, useLayoutEffect, useRef, useState } from "react";
import { useStepper } from "../stepper.js";
import { api, DEMO, fileUrl, post } from "./source.js";
import { emit } from "../bus.js";
import { fmtN } from "../model.js";
import { Empty, Icon, useBusy, useToast } from "../ui.jsx";
import { Bars, Funnel, ImagePath, ScoreStacks, SplitTable, StackBar, TypeDonut, useCount } from "./Insights.jsx";
import Pair from "./Pair.jsx";
import SourceDrawer from "./SourceDrawer.jsx";
import { isActive, useJobs } from "../jobs/jobs.js";
import "./results.css";

const STEPS = [
  { key: "pick", title: "Choose dataset", icon: "database", lead: "Pick the generated dataset you want to look at. Only finished jobs have results." },
  { key: "overview", title: "Overview", icon: "chart", lead: "A quick look at what was generated, and anything to check before training." },
  { key: "review", title: "Review pairs", icon: "list", lead: "Read the question-and-answer pairs. Approve the good ones, reject the bad ones, or edit them. You can skip this step." },
  { key: "insights", title: "Insights", icon: "layers", lead: "Optional. Charts that show where the questions came from, how they scored, and why some were removed." },
  { key: "finish", title: "Accept & download", icon: "download", lead: "Write the final dataset with your reviews applied, and download the files for training." },
];
const SHOW = [["kept", "Kept"], ["review", "To review"], ["dropped", "Dropped"], ["all", "All"]];
const MORE = [["duplicate", "Duplicates"], ["image", "From images"]];
const TYPES = [["", "All types"], ["factual", "Factual"], ["explanation", "Explanation"], ["how-to", "How-to"], ["from_images", "From images"],
  ["comparison", "Comparison"], ["decline", "Decline"], ["general", "General knowledge"]];
const LIMIT = 10;
const KEY = "slm-results-job";
const STATUS = { done: ["Finished", "ok"], running: ["Still running", "run"], queued: ["Queued", "run"], failed: ["Failed", "bad"], cancelled: ["Cancelled", "muted"] };
const firstJob = () => new URL(location.href).searchParams.get("job") || localStorage.getItem(KEY);
const when = (iso) => { try { return new Date(iso).toLocaleString(undefined, { day: "numeric", month: "short", hour: "2-digit", minute: "2-digit" }); } catch { return ""; } };

export default function ResultsPage() {
  const toast = useToast();
  const [jobs, setJobs] = useState(null);
  // the app-wide job poller: keeps the list and the picked job's status current while this page is open
  const { jobs: liveJobs } = useJobs();
  useEffect(() => { if (!DEMO && liveJobs && jobs) setJobs(liveJobs); }, [liveJobs]); // eslint-disable-line react-hooks/exhaustive-deps
  const [jobId, setJobId] = useState(firstJob);
  // flips when the picked job finishes, so its results load without a page refresh
  const pickedActive = !DEMO && isActive(liveJobs?.find((j) => j.id === jobId)?.status);
  const [job, setJob] = useState(null);
  const [report, setReport] = useState(null);
  const [problem, setProblem] = useState(null); // { kind: "running" | "failed", text }
  const [step, setStep] = useState(0);
  const [counts, setCounts] = useState(null);
  const [accepted, setAccepted] = useState(null);
  const [drawer, setDrawer] = useState(null);

  // direction of travel, so each step slides in from the right side
  const prev = useRef(0);
  const dir = step >= prev.current ? "fwd" : "back";
  useEffect(() => { prev.current = step; }, [step]);

  useEffect(() => {
    api("/jobs").then((list) => {
      setJobs(list);
      setJobId((id) => (id && list.some((j) => j.id === id) ? id : (list.find((j) => j.status === "done") || list[0])?.id || null));
    }).catch((e) => { setJobs([]); toast(`API not reachable: ${e.message}`, "error"); });
  }, [toast]);

  useEffect(() => {
    setJob(null); setReport(null); setProblem(null); setCounts(null); setAccepted(null);
    if (!jobId) return;
    localStorage.setItem(KEY, jobId);
    // the job first: the API answers 409 "not ready" for a running job and for a failed one alike, so its status tells them apart
    let live = true;
    api(`/jobs/${jobId}`).then((j) => { if (live) setJob(j); return j; }).catch(() => null)
      .then((j) => api(`/jobs/${jobId}/report`).then((r) => { if (live) setReport(r); }).catch((e) => {
        if (!live) return;
        const st = j?.status;
        const running = st ? st === "queued" || st === "running" : e.status === 409;
        setProblem(running ? { kind: "running", text: "This job is still running. Its results appear here as soon as it finishes." }
          : { kind: "failed", text: st === "cancelled" ? "This job was cancelled before it produced a dataset." : st === "failed" ? "This job failed before it produced a dataset." : e.message });
      }));
    return () => { live = false; };
  }, [jobId, pickedActive]); // eslint-disable-line react-hooks/exhaustive-deps
  // counts of the review tabs, loaded once so the steps can show progress
  useEffect(() => {
    if (!report) return;
    api(`/jobs/${jobId}/rows?status=kept&limit=1`).then((x) => setCounts((c) => c || x.counts)).catch(() => {});
  }, [report, jobId]);
  useEffect(() => { emit("results-job", jobId ? { id: jobId, name: job?.name || job?.request?.name } : null); }, [jobId, job]);

  // the soft card behind the active step and the progress line are measured from the real step buttons
  const [stepperRef, ind] = useStepper();

  const openGenerate = (id) => { const u = new URL(location.href); id ? u.searchParams.set("job", id) : u.searchParams.delete("job"); u.hash = "#/generate"; location.href = u.toString(); };
  const onCounts = useCallback((c) => setCounts(c), []);
  const ready = !!report;
  const cur = STEPS[step];
  const reviewed = counts ? (counts.kept ?? 0) - (counts.review ?? 0) : 0;
  const notes = {
    pick: jobId ? { ok: ready, note: job?.name || jobs?.find((j) => j.id === jobId)?.name || jobId } : { note: "Pick a dataset" },
    overview: ready ? { ok: step > 1, note: `${fmtN(report.kept)} rows` } : { note: "—" },
    review: ready ? { ok: step > 2, note: counts ? `${fmtN(reviewed)} of ${fmtN(counts.kept)} reviewed` : "Optional" } : { note: "—" },
    insights: ready ? { ok: step > 3, note: "Optional" } : { note: "—" },
    finish: ready ? { ok: !!accepted, note: accepted ? "Accepted" : "Not yet" } : { note: "—" },
  };

  if (jobs && jobs.length === 0) {
    return <section className="card rs-empty"><Empty icon="results" title="No datasets yet">Generate your first dataset and its results show up here.
      <a className="btn primary" href="#/generate"><Icon name="plus" size={14} />New dataset</a></Empty></section>;
  }

  return (
    <div className="gen rs">
      <nav className="stepper" aria-label="Results steps" ref={stepperRef} style={{ "--p": step / (STEPS.length - 1) }}>
        <span className="stp-progress" aria-hidden="true"><i style={ind ? { width: ind.end } : undefined} /></span>
        {ind && <span className="stp-ind" aria-hidden="true" style={{ transform: `translate(${ind.left}px, ${ind.top}px)`, width: ind.width, height: ind.height }} />}
        {STEPS.map((s, i) => {
          const st = notes[s.key];
          return (
            <button type="button" key={s.key} className={`stp${i === step ? " on" : ""}${st.ok ? " ok" : ""}`} disabled={i > 0 && !ready} onClick={() => setStep(i)}>
              <span className="stp-n">{st.ok && i !== step ? <Icon name="check" size={14} stroke={2.6} className="pop" /> : <span className="pop">{i + 1}</span>}</span>
              <span className="stp-t"><b>{s.title}</b><small>{st.note}</small></span>
            </button>
          );
        })}
      </nav>

      <section className="card gen-main rs-main">
        <header className={`gen-hd anim-${dir}`} key={`hd${step}`}>
          <span className="gen-hd-ico"><Icon name={cur.icon} size={20} /></span>
          <div className="gen-hd-t">
            <div className="gen-hd-meta">
              <span className="step-n">Step {step + 1} of {STEPS.length}</span>
              <span className="step-dots" aria-hidden="true">{STEPS.map((s2, i) => <i key={s2.key} className={i < step ? "done" : i === step ? "on" : ""} />)}</span>
              {DEMO && <span className="rs-demo" title="Dummy data: set USE_DEMO_RESULTS to false in src/environment.js to use the real API">Demo data</span>}
            </div>
            <h2>{cur.title}</h2><p>{cur.lead}</p>
          </div>
        </header>

        <div className={`gen-bd anim-${dir}`} key={`bd${step}`}>
          {cur.key === "pick" && <PickStep jobs={jobs} jobId={jobId} onPick={setJobId} problem={problem} job={job} openGenerate={openGenerate} />}
          {cur.key === "overview" && ready && <OverviewStep r={report} onSource={() => setDrawer("documents")} />}
          {cur.key === "review" && ready && <Review jobId={jobId} job={job} onCounts={onCounts} />}
          {cur.key === "insights" && ready && <Insights r={report} />}
          {cur.key === "finish" && ready && <FinishStep jobId={jobId} job={job} r={report} counts={counts} accepted={accepted} setAccepted={setAccepted} goReview={() => setStep(2)} />}
        </div>

        <footer className="gen-ft">
          <button type="button" className="btn ghost" disabled={step === 0} onClick={() => setStep(step - 1)}><Icon name="back" size={15} />Back</button>
          <div className="grow" />
          {step === 2 && <button type="button" className="btn ghost" onClick={() => setStep(4)}>Skip to accept</button>}
          {step < STEPS.length - 1
            ? <button type="button" className="btn primary next" disabled={!ready} onClick={() => setStep(step + 1)}>Next: {STEPS[step + 1].title}<Icon name="arrow" size={15} /></button>
            : <a className="btn ghost" href="#/generate"><Icon name="plus" size={14} />Generate another dataset</a>}
        </footer>
      </section>
      <SourceDrawer jobId={jobId} job={job} tab={drawer} setTab={setDrawer} onClose={() => setDrawer(null)} />
    </div>
  );
}

// ------------------------------------------------------------------ step 1: choose a dataset
function PickStep({ jobs, jobId, onPick, problem, job, openGenerate }) {
  if (!jobs) return <div className="rs-skel rows"><i /><i /></div>;
  return (
    <div className="rs-pickstep">
      <div className="rs-jobs" role="radiogroup" aria-label="Datasets">
        {jobs.map((j) => {
          const [l, tone] = STATUS[j.status] || [j.status, "muted"];
          return (
            <button type="button" role="radio" aria-checked={j.id === jobId} key={j.id} className={`rs-job${j.id === jobId ? " on" : ""}`} onClick={() => onPick(j.id)}>
              <span className="rs-radio" aria-hidden="true" />
              <span className="rs-job-t"><b>{j.name || j.id}</b><small className="mono">{j.id}{j.created ? ` · ${when(j.created)}` : ""}</small></span>
              <span className={`rs-status ${tone}`}><span className={`rs-dot ${tone}`} />{l}</span>
            </button>
          );
        })}
      </div>
      {problem && (
        <div className={`rs-callout ${problem.kind}`}>
          <Icon name={problem.kind === "running" ? "clock" : "alert"} size={16} />
          <div><b>{problem.kind === "running" ? "Not finished yet" : "This job has no results"}</b><p>{job?.error || problem.text}</p></div>
          <button type="button" className="btn small" onClick={() => openGenerate(jobId)}>{problem.kind === "running" ? "Follow the progress" : "Open it on Generate"}</button>
        </div>
      )}
    </div>
  );
}

// ------------------------------------------------------------------ step 2: the key numbers and what to check
function OverviewStep({ r, onSource }) {
  const f = r.funnel || {}, sp = r.splits || {};
  const removed = Object.values(r.drop_reasons || {}).reduce((a, b) => a + b, 0);
  const kept = useCount(r.kept);
  const warns = r.warnings || [];
  return (
    <>
      <section className="rs-sum">
        <div><span>Rows in the dataset</span><b className="num">{fmtN(Math.round(kept))}</b><small>from {fmtN(f.questions_generated)} questions</small></div>
        <div><span>Average grounded score</span><b className="num">{r.avg_grounded ?? "—"}<em>/ 5</em></b><small>how well answers match the source</small></div>
        <div><span>Train · validation · test</span><b className="num">{fmtN(sp.train || 0)}<em> · {fmtN(sp.val || 0)} · {fmtN(sp.test || 0)}</em></b><small>split by {r.split_by || "section"}</small></div>
        <div><span>Removed</span><b className="num">{fmtN(removed)}</b><small>duplicates and low scores</small></div>
      </section>
      {warns.length > 0 ? (
        <div className="rs-callout warn"><Icon name="alert" size={16} />
          <div><b>Check before training</b><ul>{warns.map((w) => <li key={w}>{w}</li>)}</ul></div></div>
      ) : (
        <div className="rs-callout ok"><Icon name="check" size={16} /><div><b>Nothing to worry about</b><p>No warnings for this dataset.</p></div></div>
      )}
      <div className="rs-hint">
        <Icon name="book" size={16} />
        <span>Want to see what the questions were made from? Open the parsed documents, images and knowledge base.</span>
        <button type="button" className="btn small" onClick={onSource}>Open source data</button>
      </div>
    </>
  );
}

// every kept row, fetched in pages: the API returns at most 200 rows per call
async function allRows(jobId, status = "kept") {
  const PAGE = 200, out = [];
  for (let offset = 0; ; offset += PAGE) {
    const x = await api(`/jobs/${jobId}/rows?status=${status}&offset=${offset}&limit=${PAGE}`);
    out.push(...(x.rows || []));
    if (!x.rows?.length || out.length >= (x.total ?? 0)) return out;
  }
}

// ------------------------------------------------------------------ step 5: accept and download
function FinishStep({ jobId, job, r, counts, accepted, setAccepted, goReview }) {
  const toast = useToast();
  const [busy, run] = useBusy();
  const [all, setAll] = useState(null);
  useEffect(() => { let live = true; allRows(jobId).then((x) => live && setAll(x)).catch(() => {}); return () => { live = false; }; }, [jobId]);
  const n = (f) => (all ? all.filter(f).length : 0);
  const approved = n((x) => x.review?.action === "approve"), rejected = n((x) => x.review?.action === "reject"), edited = n((x) => x.review?.edited);
  const open = all ? all.length - approved - rejected : counts?.review ?? 0;
  const final = (r.kept ?? 0) - rejected;
  const accept = () => run(async () => {
    try { setAccepted(await post(`/jobs/${jobId}/accept`)); toast("Dataset written"); } catch (e) { toast(e.message, "error"); }
  });
  const files = [["train.jsonl", "Train set", "for training the model"], ["val.jsonl", "Validation set", "for checking during training"], ["test.jsonl", "Test set", "for the final evaluation"],
    ["quality_report.json", "Quality report", "all the numbers from this page"], ...(accepted ? [["all.jsonl", "Everything", "all rows in one file"]] : [])];
  return (
    <>
      <div className="rs-fin">
        <div className="rs-fin-n"><b className="num">{fmtN(final)}</b><span>rows will be written</span></div>
        <ul className="rs-fin-l">
          <li><i className="ok" /><b className="num">{fmtN(approved)}</b> approved{edited ? ` (${fmtN(edited)} edited)` : ""}</li>
          <li><i className="bad" /><b className="num">{fmtN(rejected)}</b> rejected, left out</li>
          <li><i /><b className="num">{fmtN(open)}</b> not reviewed, kept as they are</li>
        </ul>
        {open > 0 && <button type="button" className="btn small ghost" onClick={goReview}><Icon name="back" size={13} />Back to review</button>}
      </div>

      <div className={`rs-accept${accepted ? " done" : ""}`}>
        <span className="rs-num">1</span>
        <div className="grow"><b>{accepted ? "Dataset written" : "Accept the dataset"}</b>
          <p>{accepted ? <>Wrote {fmtN(accepted.rows)} rows to <span className="mono">{accepted.path}</span></> : "Saves the final files on the server, with your reviews applied."}</p></div>
        <button type="button" className={`btn ${accepted ? "" : "primary"}`} disabled={busy || job?.status !== "done"} onClick={accept}>
          <Icon name={accepted ? "refresh" : "check"} size={15} />{busy ? "Writing…" : accepted ? "Accept again" : "Accept dataset"}</button>
      </div>

      <div className="rs-dl">
        <div className="rs-dl-h"><span className="rs-num">2</span><div><b>Download the files</b><p>Use these to fine-tune your model.</p></div></div>
        <div className="rs-files">
          {files.map(([fn, l, d]) => (
            <a key={fn} className="rs-file" href={fileUrl(jobId, fn)} download={fn}>
              <span className="rs-file-ico"><Icon name="doc" size={16} /></span>
              <span className="rs-file-t"><b>{l}</b><small>{d}</small></span>
              <span className="mono rs-file-n">{fn}</span>
              <Icon name="download" size={15} className="rs-file-dl" />
            </a>
          ))}
        </div>
      </div>
    </>
  );
}

function Insights({ r }) {
  const f = r.funnel || {}, im = r.image_funnel || {};
  const mix = r.mix || { documents: r.kept - (r.paraphrases || 0) - (r.declines || 0), paraphrases: r.paraphrases || 0, decline: r.declines || 0, general: 0 };
  return (
    <div className="rs-ins">
      <Panel title="Where the questions went" d={0}>
        <Funnel steps={[["Generated", f.questions_generated], ["After dedupe", f.after_question_dedupe], ["Answered", f.answered], ["Passed the judge", f.passed_judge], ["In the dataset", f.kept, true]]} />
      </Panel>
      <Panel title="Judge scores of kept pairs" d={1}><ScoreStacks scores={r.scores_of_kept || {}} /></Panel>
      <Panel title="What the dataset is made of" d={2}>
        <StackBar parts={[["From documents", mix.documents, "var(--mix-1)"], ["Paraphrases", mix.paraphrases, "var(--mix-2)"], ["Declines", mix.decline, "var(--mix-3)"], ["General", mix.general, "var(--mix-4)"]]} />
      </Panel>
      <Panel title="Question types" d={3}><TypeDonut types={r.types || {}} /></Panel>
      <Panel title="Why pairs were removed" d={4}>
        <Bars empty="Nothing was removed." rows={Object.entries(r.drop_reasons || {}).sort((a, b) => b[1] - a[1]).map(([k, v]) => [k, v, /duplicate/.test(k) ? "muted" : "accent"])} />
      </Panel>
      <Panel title="Images to Q&A pairs" right={im.read_images === false ? "images ignored" : im.vision_analysis ? "vision model" : "OCR only"} d={5}><ImagePath im={im} /></Panel>
      <Panel title="Split per section" right={r.split_by ? `by ${r.split_by}` : ""} d={6} wide><SplitTable cov={r.split_coverage || {}} /></Panel>
    </div>
  );
}

function Panel({ title, right, d = 0, wide, children }) {
  return (
    <section className={`rs-panel${wide ? " wide" : ""}`} style={{ "--d": d }}>
      <header><h3>{title}</h3><div className="grow" />{right && <span className="rs-note-r">{right}</span>}</header>
      {children}
    </section>
  );
}

// ------------------------------------------------------------------ review list
function Review({ jobId, job, onCounts }) {
  const toast = useToast();
  const [filter, setFilter] = useState({ status: "kept", type: "", q: "", offset: 0 });
  const [rows, setRows] = useState(null);
  const [loading, setLoading] = useState(true);

  const load = useCallback(() => {
    const qs = new URLSearchParams({ status: filter.status, offset: filter.offset, limit: LIMIT });
    if (filter.type) qs.set("type", filter.type);
    if (filter.q) qs.set("q", filter.q);
    setLoading(true);
    api(`/jobs/${jobId}/rows?${qs}`).then(setRows).catch((e) => { if (e.status !== 409) toast(e.message, "error"); }).finally(() => setLoading(false));
  }, [jobId, filter, toast]);
  useEffect(() => { const t = setTimeout(load, filter.q ? 300 : 0); return () => clearTimeout(t); }, [load, filter.q]);

  useEffect(() => { if (rows?.counts) onCounts?.(rows.counts); }, [rows, onCounts]);
  const setF = (k, v) => setFilter((x) => ({ ...x, [k]: v, offset: k === "offset" ? v : 0 }));
  const c = rows?.counts || {};
  const more = MORE.find(([k]) => k === filter.status);
  return (
    <section className="rs-review">
      <div className="rs-filters">
        <div className="rs-seg" role="tablist">
          {SHOW.map(([k, l]) => (
            <button type="button" role="tab" aria-selected={filter.status === k} key={k} className={filter.status === k ? "on" : ""} onClick={() => setF("status", k)}>
              {l}<span className="num">{fmtN(c[k] ?? 0)}</span></button>
          ))}
          {more && <button type="button" className="on" role="tab" aria-selected="true">{more[1]}<span className="num">{fmtN(c[more[0]] ?? 0)}</span></button>}
        </div>
        <div className="grow" />
        <label className="search"><Icon name="search" size={14} /><input placeholder="Search" value={filter.q} onChange={(e) => setF("q", e.target.value)} /></label>
        <label className="rs-select" title="Filter">
          <select value={filter.type || (more ? `@${more[0]}` : "")} aria-label="Filter"
            onChange={(e) => { const v = e.target.value; if (v.startsWith("@")) setFilter((x) => ({ ...x, status: v.slice(1), type: "", offset: 0 })); else setF("type", v); }}>
            <optgroup label="Question type">{TYPES.map(([v, l]) => <option key={v} value={v}>{l}</option>)}</optgroup>
            <optgroup label="Show only">{MORE.map(([k, l]) => <option key={k} value={`@${k}`}>{l} ({fmtN(c[k] ?? 0)})</option>)}</optgroup>
          </select>
        </label>
      </div>

      <div className={`rs-list${loading ? " is-loading" : ""}`} key={`${filter.status}|${filter.type}|${filter.offset}`}>
        {!rows ? <div className="rs-skel rows"><i /><i /><i /></div>
          : rows.rows.length === 0 ? <Empty icon="search" title="Nothing here">Try another filter or search.</Empty>
          : rows.rows.map((row, i) => (
            <Pair key={row.q_id} i={i} row={row} jobId={jobId} job={job}
              onChange={(review) => setRows((x) => {
                const was = !!row.review?.action, now = !!review?.action;
                const counts = { ...x.counts, review: (x.counts?.review ?? 0) + (was && !now ? 1 : !was && now ? -1 : 0) };
                return { ...x, counts, rows: x.rows.map((y) => (y.q_id === row.q_id ? { ...y, review } : y)) };
              })} />
          ))}
      </div>

      {rows && rows.total > LIMIT && (
        <footer className="rs-pager">
          <span className="muted small"><b className="num">{filter.offset + 1}–{Math.min(filter.offset + LIMIT, rows.total)}</b> of <b className="num">{fmtN(rows.total)}</b></span>
          <div className="grow" />
          <button type="button" className="btn small ghost" disabled={filter.offset === 0} onClick={() => setF("offset", Math.max(0, filter.offset - LIMIT))}><Icon name="back" size={13} />Previous</button>
          <button type="button" className="btn small ghost" disabled={filter.offset + LIMIT >= rows.total} onClick={() => setF("offset", filter.offset + LIMIT)}>Next<Icon name="arrow" size={13} /></button>
        </footer>
      )}
    </section>
  );
}
