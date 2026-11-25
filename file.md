//Insights.jsx
// Charts for the Results page, drawn with Recharts. Colours come from the theme (light/dark) and the coral palette.
import { useEffect, useState } from "react";
import { Bar, BarChart, Cell, LabelList, Pie, PieChart, ResponsiveContainer, Tooltip, XAxis, YAxis } from "recharts";
import { fmtN } from "../model.js";
import { SCORES } from "./Pair.jsx";

const calm = () => typeof matchMedia === "function" && matchMedia("(prefers-reduced-motion: reduce)").matches;

// counts a number up from 0 with an ease-out, for the headline figures
export function useCount(target, ms = 900) {
  const [v, setV] = useState(calm() ? target : 0);
  useEffect(() => {
    if (typeof target !== "number" || calm()) { setV(target); return undefined; }
    let raf, t0;
    const step = (t) => { t0 ??= t; const k = Math.min(1, (t - t0) / ms); setV(target * (1 - (1 - k) ** 3)); if (k < 1) raf = requestAnimationFrame(step); };
    raf = requestAnimationFrame(step);
    return () => cancelAnimationFrame(raf);
  }, [target, ms]);
  return v;
}

export function Meter({ value }) {
  return <span className="rs-meter"><i style={{ "--w": `${Math.max(0, Math.min(1, value)) * 100}%` }} /></span>;
}

export function Donut({ value, label, sub }) {
  const r = 26, c = 2 * Math.PI * r;
  return (
    <span className="rs-donut">
      <svg viewBox="0 0 64 64" aria-hidden="true">
        <defs><linearGradient id="rs-g" x1="0" y1="0" x2="1" y2="1"><stop offset="0" stopColor="var(--grad-a)" /><stop offset="1" stopColor="var(--grad-b)" /></linearGradient></defs>
        <circle cx="32" cy="32" r={r} className="trk" />
        <circle cx="32" cy="32" r={r} className="val" style={{ "--c": c, "--o": c * (1 - Math.max(0, Math.min(1, value))) }} />
      </svg>
      <span className="rs-donut-t"><b className="num">{label}</b>{sub && <small>{sub}</small>}</span>
    </span>
  );
}


// ---- theme -------------------------------------------------------------------------------------
// Categorical palette (coral first), checked for colour-blind separation in both modes.
const CAT = {
  light: ["#F2545B", "#257DE0", "#00B894", "#F2A600", "#6A4CE2", "#EE7FAC", "#008A17"],
  dark: ["#EB6263", "#338CED", "#00A888", "#D28E05", "#9887EF", "#E24D8B", "#0AAC23"],
};
// One-hue coral ramp for judge scores 1 → 5 (low → high).
const RAMP = {
  light: ["#FDE3E4", "#F9B4B7", "#F5858A", "#F2545B", "#C2333A"],
  dark: ["#4A2529", "#7C3439", "#B4464C", "#EB6263", "#FF9A9C"],
};
const TOK = ["ink", "ink-2", "muted", "faint", "line", "line-2", "surface", "surface-3", "accent"];
const readTheme = () => {
  const el = document.documentElement, cs = getComputedStyle(el);
  const mode = el.dataset.theme === "dark" ? "dark" : "light";
  const t = { mode, cat: CAT[mode], ramp: RAMP[mode] };
  TOK.forEach((k) => { t[k] = cs.getPropertyValue(`--${k}`).trim(); });
  return t;
};
function useChartTheme() {
  const [t, setT] = useState(readTheme);
  useEffect(() => {
    const mo = new MutationObserver(() => setT(readTheme()));
    mo.observe(document.documentElement, { attributes: true, attributeFilter: ["data-theme", "class", "style"] });
    return () => mo.disconnect();
  }, []);
  return t;
}
const cap = (s) => String(s).charAt(0).toUpperCase() + String(s).slice(1);
const anim = () => ({ isAnimationActive: !calm(), animationDuration: 1000, animationEasing: "ease-out" });
const tick = (t) => ({ fill: t["ink-2"], fontSize: 12.5, fontFamily: "inherit" });

// ---- tooltip -----------------------------------------------------------------------------------
function Tip({ active, payload, label, fmt }) {
  if (!active || !payload?.length) return null;
  const rows = fmt ? fmt(payload, label) : payload.map((p) => ({ c: p.payload?.fill || p.color, l: p.name, v: fmtN(p.value) }));
  const title = rows.title ?? label;
  return (
    <div className="rs-tip">
      {title != null && title !== "" && <b className="rs-tip-h">{title}</b>}
      {(rows.items || rows).map((r) => <span key={r.l}><i style={{ background: r.c }} />{r.l}<b className="num">{r.v}</b></span>)}
    </div>
  );
}
const tipProps = (t, fmt) => ({ content: <Tip fmt={fmt} />, cursor: { fill: t["surface-3"], radius: 6 }, isAnimationActive: false, wrapperStyle: { outline: "none", zIndex: 5 } });

// ---- what the dataset is made of: one 100% stacked bar ---------------------------------------
export function StackBar({ parts, note }) {
  const t = useChartTheme();
  const live = parts.filter((p) => p[1]);
  const tot = live.reduce((a, p) => a + p[1], 0) || 1;
  const row = Object.fromEntries(live.map(([l, v]) => [l, v]));
  const col = (k) => t.cat[[0, 3, 1, 2][k] ?? k];
  return (
    <div className="rs-stack">
      <ResponsiveContainer width="100%" height={34}>
        <BarChart layout="vertical" data={[row]} margin={{ top: 4, right: 0, bottom: 4, left: 0 }} barCategoryGap={0}>
          <XAxis type="number" hide domain={[0, tot]} />
          <YAxis type="category" hide />
          <Tooltip {...tipProps(t, (p) => ({ title: "", items: p.map((x) => ({ c: x.color, l: x.name, v: `${fmtN(x.value)} · ${Math.round((100 * x.value) / tot)}%` })) }))} cursor={false} shared={false} />
          {live.map(([l], k) => (
            <Bar key={l} dataKey={l} name={l} stackId="s" fill={col(parts.findIndex((p) => p[0] === l))} stroke={t.surface} strokeWidth={2} {...anim()}
              radius={live.length === 1 ? 6 : k === 0 ? [6, 0, 0, 6] : k === live.length - 1 ? [0, 6, 6, 0] : 0} />
          ))}
        </BarChart>
      </ResponsiveContainer>
      <div className="rs-legend">
        {parts.map(([l, v], k) => <span key={l}><i style={{ background: col(k) }} />{l}<b className="num">{fmtN(v)}</b></span>)}
      </div>
      {note && <small className="muted">{note}</small>}
    </div>
  );
}

// ---- where the questions went: funnel as horizontal bars --------------------------------------
export function Funnel({ steps }) {
  const t = useChartTheme();
  const data = steps.map(([l, v, end], k) => {
    const prev = k ? steps[k - 1][1] : null;
    const drop = prev != null && v != null && prev > 0 && v < prev ? Math.round((100 * (prev - v)) / prev) : null;
    return { l, v: v || 0, fin: !!end, drop };
  });
  const label = ({ x, y, width, height, index }) => {
    const d = data[index];
    return (
      <text x={x + width + 8} y={y + height / 2} dy="0.35em" fontSize={12.5} fontFamily="inherit">
        <tspan fill={t.ink} fontWeight={650}>{fmtN(d.v)}</tspan>
        {d.drop != null && <tspan fill={t.muted} fontSize={11.5} dx={6}>−{d.drop}%</tspan>}
      </text>
    );
  };
  return (
    <ResponsiveContainer width="100%" height={data.length * 38 + 6}>
      <BarChart layout="vertical" data={data} margin={{ top: 0, right: 78, bottom: 0, left: 0 }} barCategoryGap={8}>
        <XAxis type="number" hide domain={[0, "dataMax"]} />
        <YAxis type="category" dataKey="l" width={118} tickLine={false} axisLine={false} tick={tick(t)} />
        <Tooltip {...tipProps(t, (p) => { const d = p[0].payload; return { title: d.l, items: [{ c: d.fin ? t.accent : t.ramp[2], l: "Questions", v: fmtN(d.v) }, ...(d.drop != null ? [{ c: t.faint, l: "Lost from previous step", v: `${d.drop}%` }] : [])] }; })} />
        <Bar dataKey="v" name="Questions" radius={[0, 6, 6, 0]} background={{ fill: t["surface-3"], radius: 6 }} {...anim()}>
          {data.map((d) => <Cell key={d.l} fill={d.fin ? t.accent : t.ramp[1]} />)}
          <LabelList dataKey="v" content={label} />
        </Bar>
      </BarChart>
    </ResponsiveContainer>
  );
}

// ---- judge scores: 100% stacked bar per criterion, 1 → 5 ------------------------------------
export function ScoreStacks({ scores }) {
  const t = useChartTheme();
  const rows = Object.entries(scores);
  if (!rows.length) return <div className="muted small">No judged pairs kept.</div>;
  const data = rows.map(([k, m]) => {
    const tot = Object.values(m).reduce((a, b) => a + b, 0) || 1;
    const d = { name: SCORES[k] || k, tot, avg: Object.entries(m).reduce((a, [n, c]) => a + Number(n) * c, 0) / tot };
    [1, 2, 3, 4, 5].forEach((n) => { d[`s${n}`] = (100 * (m[n] || 0)) / tot; d[`c${n}`] = m[n] || 0; });
    return d;
  });
  const fmt = (p) => { const d = p[0].payload; return { title: `${d.name} · average ${d.avg.toFixed(1)}`, items: [5, 4, 3, 2, 1].filter((n) => d[`c${n}`]).map((n) => ({ c: t.ramp[n - 1], l: `Score ${n}`, v: `${d[`c${n}`]} · ${Math.round(d[`s${n}`])}%` })) }; };
  return (
    <div className="rs-scores">
      <ResponsiveContainer width="100%" height={data.length * 34 + 4}>
        <BarChart layout="vertical" data={data} margin={{ top: 0, right: 0, bottom: 0, left: 0 }} barCategoryGap={9}>
          <XAxis type="number" hide domain={[0, 100]} />
          <YAxis yAxisId="l" type="category" dataKey="name" width={108} tickLine={false} axisLine={false} tick={tick(t)} />
          <YAxis yAxisId="r" orientation="right" type="category" dataKey="avg" width={34} tickLine={false} axisLine={false}
            tick={{ ...tick(t), fill: t.ink, fontWeight: 650 }} tickFormatter={(v) => Number(v).toFixed(1)} />
          <Tooltip {...tipProps(t, fmt)} />
          {[1, 2, 3, 4, 5].map((n) => (
            <Bar key={n} yAxisId="l" dataKey={`s${n}`} name={`Score ${n}`} stackId="s" fill={t.ramp[n - 1]} stroke={t.surface} strokeWidth={2} {...anim()} />
          ))}
        </BarChart>
      </ResponsiveContainer>
      <div className="rs-sleg">{[1, 2, 3, 4, 5].map((n) => <span key={n}><i style={{ background: t.ramp[n - 1] }} />{n}</span>)}<span className="muted">· score, low to high · average on the right</span></div>
    </div>
  );
}

// ---- question types: donut ------------------------------------------------------------------
const TYPE_ORDER = ["factual", "how-to", "explanation", "comparison", "from_images", "decline", "general"];
export function TypeDonut({ types }) {
  const t = useChartTheme();
  const rows = Object.entries(types).filter((r) => r[1]).sort((a, b) => b[1] - a[1]);
  const tot = rows.reduce((a, r) => a + r[1], 0);
  if (!tot) return <div className="muted small">No questions yet.</div>;
  // colour follows the type, never its rank
  const col = (k) => { const i = TYPE_ORDER.indexOf(k); return i < 0 ? t.faint : t.cat[i]; };
  const data = rows.map(([k, v]) => ({ name: k.replace(/_/g, " "), key: k, value: v }));
  return (
    <div className="rs-types">
      <div className="rs-pie">
        <ResponsiveContainer width="100%" height="100%">
          <PieChart>
            <Tooltip {...tipProps(t, (p) => ({ title: "", items: [{ c: col(p[0].payload.key), l: cap(p[0].name), v: `${fmtN(p[0].value)} · ${Math.round((100 * p[0].value) / tot)}%` }] }))} cursor={false} />
            <Pie data={data} dataKey="value" nameKey="name" innerRadius="66%" outerRadius="100%" paddingAngle={data.length > 1 ? 2 : 0} cornerRadius={4}
              stroke={t.surface} strokeWidth={2} startAngle={90} endAngle={-270} {...anim()} animationDuration={1100}>
              {data.map((d) => <Cell key={d.key} fill={col(d.key)} />)}
            </Pie>
          </PieChart>
        </ResponsiveContainer>
        <span className="rs-pie-c"><b className="num">{fmtN(tot)}</b><small>pairs</small></span>
      </div>
      <div className="rs-tlist">
        {data.map((d) => (
          <span key={d.key}><i style={{ background: col(d.key) }} />{d.name}<b className="num">{fmtN(d.value)}</b><small className="num">{Math.round((100 * d.value) / tot)}%</small></span>
        ))}
      </div>
    </div>
  );
}

// ---- simple horizontal bars (drop reasons) ---------------------------------------------------
export function Bars({ rows, empty }) {
  const t = useChartTheme();
  if (!rows.length) return <div className="muted small">{empty}</div>;
  const data = rows.map(([l, v, c]) => ({ l, v: v || 0, fill: c === "muted" ? t.faint : t.accent }));
  return (
    <ResponsiveContainer width="100%" height={data.length * 32 + 4}>
      <BarChart layout="vertical" data={data} margin={{ top: 0, right: 36, bottom: 0, left: 0 }} barCategoryGap={9}>
        <XAxis type="number" hide domain={[0, "dataMax"]} />
        <YAxis type="category" dataKey="l" width={150} tickLine={false} axisLine={false} tick={{ ...tick(t), textAnchor: "end" }}
          tickFormatter={(s) => { const x = s.charAt(0).toUpperCase() + s.slice(1); return x.length > 22 ? `${x.slice(0, 21)}…` : x; }} />
        <Tooltip {...tipProps(t, (p) => ({ title: p[0].payload.l, items: [{ c: p[0].payload.fill, l: "Pairs removed", v: fmtN(p[0].value) }] }))} />
        <Bar dataKey="v" name="Pairs" radius={[0, 5, 5, 0]} maxBarSize={14} background={{ fill: t["line-2"], radius: 5 }} {...anim()}>
          {data.map((d) => <Cell key={d.l} fill={d.fill} />)}
          <LabelList dataKey="v" position="right" offset={8} fill={t.ink} fontSize={12.5} fontWeight={650} formatter={fmtN} />
        </Bar>
      </BarChart>
    </ResponsiveContainer>
  );
}

export function ImagePath({ im }) {
  const steps = [["Found", im.found], ["Read", im.read], ["Chunks", im.image_chunks], ["Questions", im.questions], ["Answered", im.answered], ["Kept", im.kept]];
  return (
    <>
      <div className="rs-ipath">
        {steps.map(([l, v], k) => <div key={l} className={k === steps.length - 1 ? "end" : ""} style={{ "--k": k }}><b className="num">{fmtN(v ?? 0)}</b><small>{l}</small></div>)}
      </div>
      {(im.hint || im.without_image_data > 0) && <div className="rs-note">{[im.hint, im.without_image_data ? `${im.without_image_data} picture(s) had no image data.` : ""].filter(Boolean).join(" ")}</div>}
      {!im.hint && Object.keys(im.skip_reasons || {}).length > 0 && <div className="muted small">Skipped: {Object.entries(im.skip_reasons).map(([k, v]) => `${v} × ${k}`).join(" · ")}</div>}
    </>
  );
}

export function SplitTable({ cov }) {
  const rows = Object.entries(cov);
  if (!rows.length) return <div className="muted small">No sections.</div>;
  return (
    <table className="rs-tbl">
      <thead><tr><th>Section</th><th>Train</th><th>Val</th><th>Test</th></tr></thead>
      <tbody>{rows.map(([sec, c]) => (
        <tr key={sec} className={!c.train && (c.val || c.test) ? "bad" : ""}><td title={sec}>{sec}</td>
          <td className="num">{fmtN(c.train || 0)}</td><td className="num">{fmtN(c.val || 0)}</td><td className="num">{fmtN(c.test || 0)}</td></tr>
      ))}</tbody>
    </table>
  );
}





















//ResultsPage.jsx
// Results page as a guided, step-by-step flow (like Generate): pick a dataset, see the overview,
// review the pairs, look at the insights if you like, then accept and download.
import { useCallback, useEffect, useLayoutEffect, useRef, useState } from "react";
import { api, DEMO, fileUrl, post } from "./source.js";
import { emit } from "../bus.js";
import { fmtN } from "../model.js";
import { Empty, Icon, useBusy, useToast } from "../ui.jsx";
import { Bars, Funnel, ImagePath, ScoreStacks, SplitTable, StackBar, TypeDonut, useCount } from "./Insights.jsx";
import Pair from "./Pair.jsx";
import SourceDrawer from "./SourceDrawer.jsx";
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
  const [jobId, setJobId] = useState(firstJob);
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
    api(`/jobs/${jobId}`).then(setJob).catch(() => {});
    api(`/jobs/${jobId}/report`).then(setReport)
      .catch((e) => setProblem(e.status === 409 ? { kind: "running", text: "This job is still running. Its results appear here as soon as it finishes." } : { kind: "failed", text: e.message }));
  }, [jobId]);
  // counts of the review tabs, loaded once so the steps can show progress
  useEffect(() => {
    if (!report) return;
    api(`/jobs/${jobId}/rows?status=kept&limit=1`).then((x) => setCounts((c) => c || x.counts)).catch(() => {});
  }, [report, jobId]);
  useEffect(() => { emit("results-job", jobId ? { id: jobId, name: job?.name || job?.request?.name } : null); }, [jobId, job]);

  // the soft card behind the active step, measured so it always sits under the right one
  const stepperRef = useRef(null);
  const [ind, setInd] = useState(null);
  useLayoutEffect(() => {
    const place = () => {
      const el = stepperRef.current?.querySelector(".stp.on");
      const n = el ? { left: el.offsetLeft, top: el.offsetTop, width: el.offsetWidth, height: el.offsetHeight } : null;
      setInd((c) => (c && n && c.left === n.left && c.top === n.top && c.width === n.width && c.height === n.height) || c === n ? c : n);
    };
    place();
    window.addEventListener("resize", place);
    return () => window.removeEventListener("resize", place);
  });

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
        <span className="stp-progress" aria-hidden="true"><i /></span>
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
              {DEMO && <span className="rs-demo" title="Dummy data: set DEMO to false in src/results/source.js to use the real API">Demo data</span>}
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
            ? <button type="button" className="btn dark" disabled={!ready} onClick={() => setStep(step + 1)}>Next: {STEPS[step + 1].title}<Icon name="arrow" size={15} /></button>
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

// ------------------------------------------------------------------ step 5: accept and download
function FinishStep({ jobId, job, r, counts, accepted, setAccepted, goReview }) {
  const toast = useToast();
  const [busy, run] = useBusy();
  const [all, setAll] = useState(null);
  useEffect(() => { api(`/jobs/${jobId}/rows?status=kept&limit=10000`).then((x) => setAll(x.rows)).catch(() => {}); }, [jobId]);
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























//results.css
/* ================================================================== Results page (simple version)
   Calm surfaces, coral only where it means something (the dataset, high scores, the main action).
   Every colour comes from the theme tokens, so light and dark mode both work. */
.rs { display: grid; gap: 18px; --sc-1: color-mix(in srgb, var(--accent) 10%, var(--line)); --sc-2: color-mix(in srgb, var(--accent) 28%, var(--surface));
  --sc-3: color-mix(in srgb, var(--accent) 50%, var(--surface)); --sc-4: color-mix(in srgb, var(--accent) 75%, var(--surface)); --sc-5: var(--accent); }
.rs .num { font-variant-numeric: tabular-nums; }

/* ---- entrance: cards rise in one after another */
.rs-in { animation: rs-rise .55s cubic-bezier(.22, 1, .36, 1) both; animation-delay: calc(var(--d, 0) * 55ms); }
@keyframes rs-rise { from { opacity: 0; transform: translateY(12px); } to { opacity: 1; transform: none; } }

.rs-stack { display: grid; gap: 12px; }
.rs-stack-bar { display: flex; height: 10px; border-radius: 99px; overflow: hidden; gap: 2px; background: var(--line-2); }
.rs-stack-bar i { width: var(--w); background: var(--c); transform-origin: left; animation: rs-grow .9s cubic-bezier(.22, 1, .36, 1) both; animation-delay: calc(.25s + var(--k) * 90ms); }
.rs-legend { display: grid; grid-template-columns: 1fr 1fr; gap: 6px 14px; }
.rs-legend span { display: flex; align-items: center; gap: 7px; font-size: 13px; color: var(--ink-2); min-width: 0; }
.rs-legend i { width: 8px; height: 8px; border-radius: 3px; background: var(--c); flex: none; }
.rs-legend b { margin-left: auto; font-weight: 650; color: var(--ink); }
.rs-stack > small { font-size: 12.5px; }
.rs-funnel { display: grid; gap: 8px; }
.rs-fstep { display: grid; grid-template-columns: 112px minmax(0, 1fr) 30px 40px; gap: 10px; align-items: center; font-size: 13px; position: relative; }
.rs-fl { color: var(--ink-2); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.rs-ft { height: 22px; border-radius: 7px; background: var(--surface-3); overflow: hidden; }
.rs-ft i { display: block; height: 100%; width: var(--w); border-radius: 7px; background: color-mix(in srgb, var(--accent) 22%, var(--surface));
  transform-origin: left; animation: rs-grow .9s cubic-bezier(.22, 1, .36, 1) both; animation-delay: calc(.3s + var(--k) * 90ms); }
.rs-fstep.end .rs-ft i { background: var(--grad); }
.rs-fstep b { text-align: right; font-weight: 650; }
.rs-fd { font-size: 11.5px; font-weight: 600; color: var(--faint); text-align: right; }

.rs-scores { display: grid; gap: 9px; }
.rs-srow { display: grid; grid-template-columns: 104px minmax(0, 1fr) 30px; gap: 10px; align-items: center; font-size: 13px; }
.rs-sl { color: var(--ink-2); }
.rs-sbar { display: flex; height: 12px; border-radius: 99px; overflow: hidden; gap: 2px; background: var(--line-2); }
.rs-sbar i { width: var(--w); background: var(--c); transform-origin: left; animation: rs-grow .8s cubic-bezier(.22, 1, .36, 1) both; animation-delay: calc(.3s + var(--k) * 70ms); }
.rs-srow b { text-align: right; font-weight: 650; }
.rs-sleg { display: flex; gap: 10px; flex-wrap: wrap; font-size: 12px; color: var(--muted); padding-top: 4px; }
.rs-sleg span { display: inline-flex; align-items: center; gap: 4px; }
.rs-sleg i { width: 9px; height: 9px; border-radius: 3px; background: var(--c); }

.rs-types { display: flex; gap: 18px; align-items: center; }
.rs-tlist { display: grid; gap: 7px; flex: 1; min-width: 0; }
.rs-tlist span { display: grid; grid-template-columns: 10px minmax(0, 1fr) auto 36px; gap: 8px; align-items: center; font-size: 13px; color: var(--ink-2); text-transform: capitalize; }
.rs-tlist i { width: 9px; height: 9px; border-radius: 3px; background: var(--c); }
.rs-tlist b { font-weight: 650; color: var(--ink); } .rs-tlist small { color: var(--muted); text-align: right; font-size: 12px; }

.rs-bars { display: grid; gap: 9px; }
.rs-brow { display: grid; grid-template-columns: minmax(0, 140px) minmax(0, 1fr) 32px; gap: 10px; align-items: center; font-size: 13px; }
.rs-bl { color: var(--ink-2); overflow: hidden; text-overflow: ellipsis; white-space: nowrap; text-transform: capitalize; }
.rs-bt { height: 8px; border-radius: 99px; background: var(--line-2); overflow: hidden; }
.rs-bt i { display: block; height: 100%; width: var(--w); border-radius: 99px; background: var(--c); transform-origin: left;
  animation: rs-grow .8s cubic-bezier(.22, 1, .36, 1) both; animation-delay: calc(.3s + var(--k) * 80ms); }
.rs-brow b { text-align: right; font-weight: 650; }

.rs-ipath { display: grid; grid-template-columns: repeat(6, 1fr); gap: 0; position: relative; }
.rs-ipath::before { content: ""; position: absolute; left: 8%; right: 8%; top: 17px; height: 2px; background: linear-gradient(90deg, var(--line), color-mix(in srgb, var(--accent) 45%, var(--line))); }
.rs-ipath > div { display: grid; justify-items: center; gap: 5px; position: relative; animation: rs-rise .5s cubic-bezier(.22, 1, .36, 1) both; animation-delay: calc(.3s + var(--k) * 70ms); }
.rs-ipath b { width: 36px; height: 36px; border-radius: 50%; display: grid; place-items: center; font-size: 14px; font-weight: 700; background: var(--surface);
  border: 1px solid var(--line-strong); color: var(--ink); box-shadow: 0 0 0 4px var(--surface); }
.rs-ipath .end b { background: var(--grad); color: #fff; border-color: transparent; box-shadow: 0 0 0 4px var(--surface), 0 4px 10px -3px rgba(var(--accent-rgb), .6); }
.rs-ipath small { font-size: 11.5px; color: var(--muted); }
.rs-note { font-size: 12.5px; color: var(--amber-ink); background: var(--amber-50); border: 1px solid var(--amber-200); border-radius: 9px; padding: 7px 10px; }

.rs-tbl { width: 100%; border-collapse: separate; border-spacing: 0; font-size: 13px; }
.rs-tbl th { text-align: left; font-size: 11px; font-weight: 700; letter-spacing: .5px; text-transform: uppercase; color: var(--faint); padding: 0 8px 8px; border-bottom: 1px solid var(--line); }
.rs-tbl th:not(:first-child), .rs-tbl td:not(:first-child) { text-align: right; width: 52px; }
.rs-tbl td { padding: 8px; border-bottom: 1px solid var(--line-2); color: var(--ink-2); }
.rs-tbl td:first-child { max-width: 170px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; color: var(--ink); font-weight: 500; }
.rs-tbl tr:last-child td { border-bottom: 0; }
.rs-tbl tbody tr { transition: background .12s; } .rs-tbl tbody tr:hover { background: var(--surface-2); }
.rs-tbl tr.bad td { color: var(--red); }

@keyframes rs-grow { from { transform: scaleX(0); } }
.rs-skel { display: grid; grid-template-columns: 1fr 1fr; gap: 14px; padding: 12px; }
.rs-skel i { height: 110px; border-radius: 14px; background: linear-gradient(90deg, var(--surface-3) 0%, var(--surface-2) 50%, var(--surface-3) 100%); background-size: 200% 100%;
  animation: rs-shimmer 1.4s ease-in-out infinite; }
.rs-skel.rows { grid-template-columns: 1fr; padding: 0; } .rs-skel.rows i { height: 120px; }
@keyframes rs-shimmer { from { background-position: 100% 0; } to { background-position: -100% 0; } }
.rs-empty { padding: 8px; }
.rs-empty .empty > span { display: grid; justify-items: center; gap: 14px; }
.rs-chev { color: var(--faint); transform: rotate(90deg); transition: transform .25s cubic-bezier(.22, 1, .36, 1); flex: none; }
.rs-chev.sm { margin-left: -2px; }

/* ---- the four key numbers, in one calm strip */
.rs-sum { display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); background: var(--surface); border: 1px solid var(--line); border-radius: 14px; animation: rs-rise .5s cubic-bezier(.22, 1, .36, 1) both; }
.rs-sum > div { display: grid; gap: 4px; padding: 18px 22px; min-width: 0; }
.rs-sum > div + div { border-left: 1px solid var(--line-2); }
.rs-sum span { font-size: 13px; color: var(--muted); }
.rs-sum b { font-family: var(--font); font-size: 28px; font-weight: 700; letter-spacing: -.6px; line-height: 1.15; color: var(--ink); }
.rs-sum b em { font-style: normal; font-size: 16px; font-weight: 500; color: var(--muted); letter-spacing: 0; margin-left: 3px; }
.rs-sum small { font-size: 12.5px; color: var(--faint); }
.rs-sum > div:first-child b { background: var(--grad); -webkit-background-clip: text; background-clip: text; color: transparent; }

/* ---- review */
.rs-review { display: grid; gap: 14px; animation: rs-rise .45s cubic-bezier(.22, 1, .36, 1) both; }
.rs-filters { display: flex; align-items: center; gap: 10px; flex-wrap: wrap; }
.rs-seg { display: inline-flex; gap: 2px; padding: 3px; border-radius: 10px; background: var(--surface-3); border: 1px solid var(--line-2); }
.rs-seg button { display: inline-flex; align-items: center; gap: 6px; height: 30px; padding: 0 12px; border: 0; border-radius: 7px; background: none; cursor: pointer;
  font: inherit; font-size: 13.5px; font-weight: 600; color: var(--muted); transition: color .15s, background .15s, box-shadow .15s; }
.rs-seg button:hover { color: var(--ink); }
.rs-seg button.on { background: var(--surface); color: var(--ink); box-shadow: 0 1px 2px rgba(16, 24, 40, .1); }
.rs-seg button span { font-size: 12px; color: var(--faint); font-weight: 600; }
.rs-seg button.on span { color: var(--accent-600); }
.rs-filters .search { height: 36px; }
.rs-filters .search input { width: 170px !important; background: transparent !important; color: var(--ink); }
.rs-select { display: inline-flex; align-items: center; height: 36px; border: 1px solid var(--line-strong); border-radius: 9px; background: var(--surface); box-shadow: var(--shadow-xs); }
.rs-select:focus-within { border-color: var(--accent-400); box-shadow: var(--ring); }
.rs-select select { border: 0 !important; box-shadow: none !important; background-color: transparent; height: 34px; padding: 0 28px 0 12px !important; font-size: 13.5px; color: var(--ink); }

.rs-list { display: grid; background: var(--surface); border: 1px solid var(--line); border-radius: 14px; overflow: hidden; transition: opacity .2s; }
.rs-list.is-loading { opacity: .6; }
.rs-list > .empty { padding: 48px 20px; }
.rs-list > .rs-skel { padding: 12px; }
.rs-pager { display: flex; align-items: center; gap: 8px; padding: 0 4px; }

/* ---- one pair: a row in the list */
.qa { display: grid; grid-template-columns: minmax(0, 1fr) auto; gap: 20px; padding: 18px 20px; position: relative; transition: background .2s;
  animation: rs-rise .4s cubic-bezier(.22, 1, .36, 1) both; animation-delay: calc(var(--i, 0) * 35ms); }
.qa + .qa { border-top: 1px solid var(--line-2); }
.qa:hover { background: var(--surface-2); }
.qa::before { content: ""; position: absolute; left: 0; top: 18px; bottom: 18px; width: 3px; border-radius: 0 3px 3px 0; background: transparent; transition: background .2s; }
.qa.is-approve::before { background: var(--green); }
.qa.is-reject::before { background: var(--red); }
.qa.is-reject .qa-q, .qa.is-reject .qa-a { opacity: .5; }
.qa.is-reject .qa-q { text-decoration: line-through; text-decoration-color: color-mix(in srgb, var(--red) 50%, transparent); }
.qa-main { display: grid; gap: 6px; min-width: 0; }
.qa-q { margin: 0; font-size: 15.5px; font-weight: 600; color: var(--ink); line-height: 1.45; }
.qa-a { margin: 0; font-size: 14.5px; color: var(--ink-2); line-height: 1.6; white-space: pre-wrap; }
.qa.is-out .qa-a { color: var(--muted); }
.qa-why { margin: 2px 0 0; font-size: 13px; color: var(--red-ink); padding-left: 10px; border-left: 2px solid var(--red-200); }
.qa-meta { display: flex; align-items: center; gap: 8px; margin-top: 4px; font-size: 12.5px; color: var(--muted); min-width: 0; flex-wrap: wrap; }
.qa-meta > span.ellipsis { max-width: 360px; }
.qa-type { padding: 1px 8px; border-radius: 99px; background: var(--surface-3); color: var(--ink-2); font-weight: 500; text-transform: capitalize; }
.qa-avg { font-weight: 650; color: var(--ink-2); }
.qa-avg small { font-weight: 400; color: var(--faint); margin-left: 1px; }
.qa-more-btn { display: inline-flex; align-items: center; gap: 3px; border: 0; background: none; padding: 0; cursor: pointer; font: inherit; font-size: 12.5px; font-weight: 600; color: var(--accent-600); }
.qa-more-btn svg { transform: rotate(90deg); transition: transform .2s; }
.qa-more-btn.on svg { transform: rotate(-90deg); }
.qa-more { display: grid; gap: 12px; margin-top: 8px; padding: 14px; border-radius: 12px; background: var(--surface-2); border: 1px solid var(--line-2); animation: rs-rise .3s ease both; }
.qa:hover .qa-more { background: var(--surface); }
.qa-scores { display: grid; grid-template-columns: repeat(auto-fill, minmax(190px, 1fr)); gap: 8px 18px; }
.qa-score { display: grid; grid-template-columns: 1fr auto 16px; gap: 8px; align-items: center; font-size: 13px; color: var(--ink-2); }
.qa-score b { text-align: right; font-weight: 650; }
.qa-score small { grid-column: 1 / -1; font-size: 12px; color: var(--muted); margin-top: -3px; }
.qa-pips { display: inline-flex; gap: 2px; }
.qa-pips i { width: 12px; height: 5px; border-radius: 3px; background: var(--line); }
.qa-score.hi .qa-pips i.on { background: var(--accent); }
.qa-score.mid .qa-pips i.on { background: var(--amber); }
.qa-score.lo .qa-pips i.on { background: var(--red); }
.qa-kv { display: grid; grid-template-columns: 80px minmax(0, 1fr); gap: 10px; font-size: 13px; }
.qa-kv > span { color: var(--faint); font-weight: 600; font-size: 12px; padding-top: 2px; }
.qa-kv p { margin: 0; color: var(--ink-2); }
.qa-chips { display: flex; flex-wrap: wrap; gap: 5px; align-items: center; }

.qa-edit { display: grid; gap: 10px; animation: rs-rise .3s ease both; }
.qa-edit label { display: grid; gap: 4px; }
.qa-edit span { font-size: 12px; font-weight: 600; color: var(--faint); }
.qa-edit textarea { width: 100%; resize: vertical; }
.qa-edit-act { display: flex; gap: 6px; justify-content: flex-end; }

.qa-side { display: flex; flex-direction: column; align-items: flex-end; gap: 10px; }
.qa-state { font-size: 12px; font-weight: 600; padding: 2px 9px; border-radius: 99px; white-space: nowrap; background: var(--surface-3); color: var(--muted); }
.qa-state.ok { background: var(--green-50); color: var(--green-600); } .qa-state.bad { background: var(--red-50); color: var(--red); }
.qa-act { display: flex; gap: 4px; }
.qa-btn { width: 32px; height: 32px; display: grid; place-items: center; border-radius: 9px; border: 1px solid var(--line); background: var(--surface); color: var(--muted);
  cursor: pointer; transition: color .15s, background .15s, border-color .15s, transform .15s; }
.qa-btn:hover:not(:disabled) { color: var(--ink); border-color: var(--line-strong); }
.qa-btn:active:not(:disabled) { transform: scale(.94); }
.qa-btn:disabled { opacity: .5; cursor: default; }
.qa-btn.yes:hover:not(:disabled) { color: var(--green-600); border-color: var(--green-200); background: var(--green-50); }
.qa-btn.no:hover:not(:disabled) { color: var(--red); border-color: var(--red-200); background: var(--red-50); }
.qa-btn.yes.on, .qa-btn.yes.on:hover:not(:disabled) { color: #fff; background: var(--green-600); border-color: var(--green-600); }
.qa-btn.no.on, .qa-btn.no.on:hover:not(:disabled) { color: #fff; background: var(--red); border-color: var(--red); }

/* ---- insights */
.rs-ins { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 16px; align-items: start; }
.rs-panel { display: grid; gap: 14px; align-content: start; padding: 18px 20px; background: var(--surface); border: 1px solid var(--line); border-radius: 14px; animation: rs-rise .5s cubic-bezier(.22, 1, .36, 1) both; animation-delay: calc(var(--d, 0) * 50ms); }
.rs-panel.wide { grid-column: 1 / -1; }
.rs-panel > header { display: flex; align-items: center; gap: 8px; }
.rs-panel h3 { margin: 0; font-size: 14.5px; font-weight: 650; }
.rs-note-r { font-size: 12.5px; color: var(--muted); }

/* ---- guided flow pieces */
.rs-main { min-height: 520px; }
.rs .stp:disabled { cursor: default; opacity: .55; }
.rs-demo { font-size: 11.5px; font-weight: 600; color: var(--muted); padding: 1px 8px; border-radius: 99px; border: 1px dashed var(--line-strong); }
.rs-dot { width: 7px; height: 7px; border-radius: 50%; background: var(--faint); flex: none; display: inline-block; }
.rs-dot.ok { background: var(--green); } .rs-dot.run { background: var(--accent); } .rs-dot.bad { background: var(--red); }

.rs-pickstep { display: grid; gap: 16px; max-width: 760px; }
.rs-jobs { display: grid; gap: 10px; }
.rs-job { display: flex; align-items: center; gap: 14px; padding: 14px 16px; border-radius: 12px; border: 1px solid var(--line); background: var(--surface);
  cursor: pointer; text-align: left; font: inherit; color: var(--ink); transition: border-color .15s, box-shadow .15s, background .15s; }
.rs-job:hover { border-color: var(--line-strong); box-shadow: var(--shadow-md); }
.rs-job.on { border-color: var(--accent-400); box-shadow: var(--ring); background: color-mix(in srgb, var(--accent) 3%, var(--surface)); }
.rs-radio { width: 18px; height: 18px; border-radius: 50%; border: 2px solid var(--line-strong); flex: none; display: grid; place-items: center; transition: border-color .15s; }
.rs-radio::after { content: ""; width: 8px; height: 8px; border-radius: 50%; background: var(--grad); transform: scale(0); transition: transform .25s cubic-bezier(.34, 1.56, .64, 1); }
.rs-job.on .rs-radio { border-color: var(--accent); } .rs-job.on .rs-radio::after { transform: scale(1); }
.rs-job-t { display: grid; gap: 2px; flex: 1; min-width: 0; }
.rs-job-t b { font-size: 15px; font-weight: 600; }
.rs-job-t small { font-size: 12px; color: var(--muted); background: none; padding: 0; }
.rs-status { display: inline-flex; align-items: center; gap: 6px; font-size: 12.5px; font-weight: 600; color: var(--muted); white-space: nowrap; }
.rs-status.ok { color: var(--green-600); } .rs-status.bad { color: var(--red); } .rs-status.run { color: var(--accent-600); }

.rs-callout { display: flex; align-items: flex-start; gap: 12px; padding: 14px 16px; border-radius: 12px; border: 1px solid var(--line); background: var(--surface-2); font-size: 14px; }
.rs-callout > svg { margin-top: 2px; flex: none; }
.rs-callout > div { flex: 1; min-width: 0; }
.rs-callout b { display: block; font-weight: 600; margin-bottom: 2px; }
.rs-callout p { margin: 0; color: var(--ink-2); }
.rs-callout ul { margin: 4px 0 0; padding-left: 18px; line-height: 1.6; }
.rs-callout.warn { background: var(--amber-50); border-color: var(--amber-200); color: var(--amber-ink-2); } .rs-callout.warn b, .rs-callout.warn > svg { color: var(--amber-ink); }
.rs-callout.ok { background: var(--green-50); border-color: var(--green-200); } .rs-callout.ok > svg, .rs-callout.ok b { color: var(--green-600); }
.rs-callout.failed { background: var(--red-50); border-color: var(--red-200); } .rs-callout.failed > svg, .rs-callout.failed b { color: var(--red); }
.rs-callout.running > svg, .rs-callout.running b { color: var(--accent-600); }
.rs-hint { display: flex; align-items: center; gap: 12px; padding: 12px 16px; border-radius: 12px; border: 1px dashed var(--line-strong); color: var(--ink-2); font-size: 14px; }
.rs-hint > svg { color: var(--accent); flex: none; } .rs-hint > span { flex: 1; }

.rs-fin { display: flex; align-items: center; gap: 28px; flex-wrap: wrap; padding: 18px 22px; border-radius: 14px; border: 1px solid var(--line); background: var(--surface-2); }
.rs-fin-n { display: grid; }
.rs-fin-n b { font-size: 40px; font-weight: 750; letter-spacing: -1px; line-height: 1; background: var(--grad); -webkit-background-clip: text; background-clip: text; color: transparent; font-family: var(--font); }
.rs-fin-n span { font-size: 13px; color: var(--muted); margin-top: 4px; }
.rs-fin-l { list-style: none; margin: 0; padding: 0 0 0 28px; border-left: 1px solid var(--line); display: grid; gap: 6px; font-size: 14px; color: var(--ink-2); flex: 1; }
.rs-fin-l li { display: flex; align-items: center; gap: 8px; }
.rs-fin-l i { width: 8px; height: 8px; border-radius: 50%; background: var(--line-strong); } .rs-fin-l i.ok { background: var(--green); } .rs-fin-l i.bad { background: var(--red); }
.rs-fin-l b { font-weight: 650; color: var(--ink); min-width: 18px; }
.rs-num { width: 28px; height: 28px; border-radius: 50%; display: grid; place-items: center; font-size: 13px; font-weight: 700; background: var(--surface-3); color: var(--ink-2); flex: none; }
.rs-accept { display: flex; align-items: center; gap: 14px; padding: 16px 18px; border-radius: 14px; border: 1px solid color-mix(in srgb, var(--accent) 22%, var(--line));
  background: color-mix(in srgb, var(--accent) 4%, var(--surface)); }
.rs-accept .rs-num { background: var(--grad); color: #fff; }
.rs-accept.done { border-color: var(--green-200); background: var(--green-50); } .rs-accept.done .rs-num { background: var(--green-600); }
.rs-accept b, .rs-dl-h b { font-size: 15px; font-weight: 650; }
.rs-accept p, .rs-dl-h p { margin: 2px 0 0; font-size: 13.5px; color: var(--muted); }
.rs-accept .mono { color: var(--green-600); }
.rs-dl { display: grid; gap: 12px; padding: 16px 18px; border-radius: 14px; border: 1px solid var(--line); }
.rs-dl-h { display: flex; align-items: center; gap: 14px; }
.rs-files { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 10px; padding-left: 42px; }
.rs-file { display: flex; align-items: center; gap: 12px; padding: 11px 14px; border-radius: 11px; border: 1px solid var(--line); background: var(--surface); color: var(--ink);
  text-decoration: none; transition: border-color .15s, box-shadow .15s, transform .15s; }
.rs-file:hover { border-color: var(--accent-200); box-shadow: var(--shadow-md); transform: translateY(-1px); }
.rs-file-ico { width: 34px; height: 34px; border-radius: 9px; display: grid; place-items: center; background: var(--accent-50); color: var(--accent-600); flex: none; }
.rs-file-t { display: grid; flex: 1; min-width: 0; } .rs-file-t b { font-size: 14px; font-weight: 600; } .rs-file-t small { font-size: 12.5px; color: var(--muted); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.rs-file-n { font-size: 11.5px; color: var(--faint); }
.rs-file-dl { color: var(--faint); transition: color .15s, transform .2s; flex: none; }
.rs-file:hover .rs-file-dl { color: var(--accent); transform: translateY(2px); }

@media (max-width: 1100px) { .rs-files { grid-template-columns: 1fr; padding-left: 0; } .rs-sum { grid-template-columns: 1fr 1fr; } .rs-sum > div:nth-child(3) { border-left: 0; } .rs-sum > div:nth-child(n+3) { border-top: 1px solid var(--line-2); } .rs-ins { grid-template-columns: 1fr; } }
@media (max-width: 760px) { .qa { grid-template-columns: 1fr; } .qa-side { flex-direction: row; justify-content: space-between; } }
@media (prefers-reduced-motion: reduce) { .rs *, .rs { animation: none !important; transition: none !important; } }

/* Recharts */
.rs-pie { width: 128px; height: 128px; flex: none; position: relative; }
.rs-pie-c { position: absolute; inset: 0; display: grid; place-content: center; text-align: center; line-height: 1.1; pointer-events: none; }
.rs-pie-c b { font-size: 20px; font-weight: 750; color: var(--ink); } .rs-pie-c small { font-size: 11px; color: var(--muted); }
.rs .recharts-wrapper, .rs .recharts-surface { overflow: visible; }
.rs .recharts-wrapper *:focus { outline: none; }
.rs-tip { background: var(--surface); border: 1px solid var(--line); border-radius: 10px; padding: 9px 11px; display: grid; gap: 5px; min-width: 150px;
  box-shadow: 0 10px 30px -10px rgb(0 0 0 / .25); font-size: 12.5px; color: var(--ink-2); }
.rs-tip-h { font-size: 12.5px; font-weight: 700; color: var(--ink); text-transform: capitalize; }
.rs-tip span { display: grid; grid-template-columns: 9px minmax(0, 1fr) auto; gap: 8px; align-items: center; }
.rs-tip i { width: 9px; height: 9px; border-radius: 3px; }
.rs-tip span b { font-weight: 650; color: var(--ink); text-transform: none; }
