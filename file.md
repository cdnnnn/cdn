/HBar.tsx
import { useEffect, useState } from 'react';

interface HBarProps {
  value: number;
  max?: number;
  color?: string;
  label: string;
  sublabel?: string;
}

export default function HBar({ value, max = 100, color = '#6366F1', label, sublabel }: HBarProps) {
  const [width, setWidth] = useState(0);
  useEffect(() => {
    const t = setTimeout(() => setWidth(value), 50);
    return () => clearTimeout(t);
  }, [value]);

  return (
    <div style={{ marginBottom: 14 }}>
      <div style={{ display: 'flex', justifyContent: 'space-between', marginBottom: 5 }}>
        <span style={{ fontSize: 13, fontWeight: 600 }}>{label}</span>
        <span style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", fontSize: 13, fontWeight: 700, color }}>{value}%</span>
      </div>
      {sublabel && <div style={{ fontSize: 11, color: '#9CA3AF', marginBottom: 4 }}>{sublabel}</div>}
      <div style={{ height: 8, background: '#F1F4F9', borderRadius: 4, overflow: 'hidden' }}>
        <div
          style={{
            height: '100%',
            width: `${(width / max) * 100}%`,
            background: color,
            borderRadius: 4,
            transition: 'width 1s cubic-bezier(.16,1,.3,1)',
          }}
        />
      </div>
    </div>
  );
}










//RadarChart.tsx
import { useState } from 'react';

export interface RadarModel {
  name: string;
  values: number[]; // 0..1, one per metric, same order as `metrics`
}

interface RadarChartProps {
  models: RadarModel[];
  metrics?: string[];
  size?: number;
  colors?: string[];
}

const DEFAULT_METRICS = ['Accuracy', 'Speed', 'Cost Eff.', 'Context', 'Capability'];
const DEFAULT_COLORS = ['#6366F1', '#F59E0B', '#10B981'];

export default function RadarChart({ models, metrics = DEFAULT_METRICS, size = 300, colors = DEFAULT_COLORS }: RadarChartProps) {
  const [hovered, setHovered] = useState<number | null>(null);
  const cx = size / 2;
  const cy = size / 2;
  const r = size / 2 - 44;
  const angleStep = (2 * Math.PI) / metrics.length;
  const pt = (angle: number, radius: number) => ({ x: cx + radius * Math.sin(angle), y: cy - radius * Math.cos(angle) });

  return (
    <svg width={size} height={size} viewBox={`0 0 ${size} ${size}`} style={{ overflow: 'visible' }}>
      {[0.2, 0.4, 0.6, 0.8, 1].map((l, i) => (
        <polygon
          key={i}
          points={metrics.map((_, mi) => { const p = pt(mi * angleStep, r * l); return `${p.x},${p.y}`; }).join(' ')}
          fill={i < 4 ? '#F1F4F9' : 'none'}
          stroke="#E5E7EB"
          strokeWidth={1}
          opacity={0.6}
        />
      ))}
      {metrics.map((m, i) => {
        const label = pt(i * angleStep, r + 24);
        const lineEnd = pt(i * angleStep, r);
        return (
          <g key={i}>
            <line x1={cx} y1={cy} x2={lineEnd.x} y2={lineEnd.y} stroke="#E5E7EB" strokeWidth={1} strokeDasharray="3,3" />
            <text x={label.x} y={label.y} textAnchor="middle" dominantBaseline="middle" fill="#6B7280" fontSize={11} fontWeight={600} fontFamily="'Segoe UI', Roboto, Arial, sans-serif">
              {m}
            </text>
          </g>
        );
      })}
      {models.map((model, mi) => (
        <g key={mi} onMouseEnter={() => setHovered(mi)} onMouseLeave={() => setHovered(null)} style={{ cursor: 'pointer' }}>
          <polygon
            points={model.values.map((v, vi) => { const p = pt(vi * angleStep, r * v); return `${p.x},${p.y}`; }).join(' ')}
            fill={colors[mi]}
            fillOpacity={hovered === mi ? 0.18 : 0.08}
            stroke={colors[mi]}
            strokeWidth={hovered === mi ? 3 : 2}
            strokeLinejoin="round"
            style={{ transition: 'all .25s' }}
          />
          {model.values.map((v, vi) => {
            const p = pt(vi * angleStep, r * v);
            return <circle key={vi} cx={p.x} cy={p.y} r={hovered === mi ? 5 : 3.5} fill="#FFFFFF" stroke={colors[mi]} strokeWidth={2} style={{ transition: 'r .2s' }} />;
          })}
        </g>
      ))}
    </svg>
  );
}




















// src\components\common\ScoreRing.module.scss
@use '../../styles/_variables' as *;

.score-ring {
  position: relative;
  flex-shrink: 0;

  &__progress { transition: stroke-dashoffset 1.2s cubic-bezier(.16, 1, .3, 1); }
  &__center {
    position: absolute; inset: 0; display: flex; flex-direction: column;
    align-items: center; justify-content: center;
  }
  &__value { font-family: $font-mono; font-weight: 700; color: #111827; line-height: 1; }
  &__label { font-size: 9px; color: #9CA3AF; font-weight: 600; margin-top: 1px; }
}















//Comparison.tsx
import { useEffect, useMemo, useState } from 'react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchModels } from '../../store/slices/modelsSlice';
import { fetchBenchmarks } from '../../store/slices/benchmarksSlice';
import RadarChart from '../common/RadarChart';
import ScoreRing from '../common/ScoreRing';
import styles from './Comparison.module.scss';

const COLORS = ['#6366F1', '#F59E0B', '#10B981'];

export default function Comparison() {
  const dispatch = useAppDispatch();
  const models = useAppSelector((s) => s.models.items);
  const benchmarks = useAppSelector((s) => s.benchmarks.items);

  // Explicit user selections; null/empty means "no override yet", in which
  // case the memoized defaults below take over. This avoids writing state
  // from inside an effect just to seed a default.
  const [selSuiteOverride, setSelSuiteOverride] = useState<string | null>(null);
  // No model-picker UI yet — reserved for when users can swap which models
  // are compared; falls back to the first three models in the catalog.
  const [selModelIdsOverride] = useState<string[] | null>(null);

  useEffect(() => {
    dispatch(fetchModels());
    dispatch(fetchBenchmarks());
  }, [dispatch]);

  const selSuite = selSuiteOverride ?? benchmarks[0]?.name ?? '';
  const defaultModelIds = useMemo(() => models.slice(0, 3).map((m) => m.id), [models]);
  const selModelIds = selModelIdsOverride ?? defaultModelIds;

  const comp = selModelIds
    .map((id) => models.find((m) => m.id === id))
    .filter((m): m is NonNullable<typeof m> => Boolean(m))
    .map((m) => ({
      name: m.name,
      accuracy: m.accuracy_score ?? 0,
      speed: m.input_price ?? 0,
      cost: m.output_price ?? 0,
      ctx: m.context_window,
      values: [
        (m.accuracy_score || 0) / 100,
        0.75,
        m.input_price ? Math.max(0, 1 - m.input_price / 10) : 0.5,
        Math.min(1, m.context_window / 200000),
        (m.agent_score || m.accuracy_score || 0) / 100,
      ],
    }));

  const mets = [
    { k: 'accuracy' as const, l: 'Accuracy' },
    { k: 'ctx' as const, l: 'Context Window' },
    { k: 'speed' as const, l: 'Input Price' },
    { k: 'cost' as const, l: 'Output Price' },
  ];

  return (
    <div className="page-enter">
      <div className="pg-hdr"><h1>Model Comparison</h1><p>Side-by-side performance across a shared benchmark</p></div>
      <div className="pg-body">
        <div className={styles['comparison__controls']}>
          <span className={styles['comparison__label']}>Test Suite:</span>
          <select className={`fi ${styles['comparison__select']}`} value={selSuite} onChange={(e) => setSelSuiteOverride(e.target.value)}>
            {benchmarks.map((b) => <option key={b.name} value={b.name}>{b.name}</option>)}
          </select>
          <span className={styles['comparison__label']}>Comparing:</span>
          {comp.map((m, i) => (
            <span key={m.name} className={styles['model-chip']} style={{ borderColor: COLORS[i], color: COLORS[i], background: `${COLORS[i]}14` }}>
              <span className={styles['model-chip__dot']} style={{ background: COLORS[i] }} /> {m.name}
            </span>
          ))}
        </div>

        <div className={styles['comparison__grid']}>
          <div className="card">
            <div className={styles['comparison__panel-title']}>Strength Profile</div>
            <div className={styles['comparison__panel-sub']}>Multi-dimensional performance comparison</div>
            <div className="radar-wrap"><RadarChart models={comp} size={280} colors={COLORS} /></div>
            <div className={styles['comparison__legend']}>
              {comp.map((m, i) => <span key={i}><span className={styles['comparison__dot']} style={{ background: COLORS[i] }} /> {m.name}</span>)}
            </div>
          </div>

          <div className="card" style={{ padding: 0 }}>
            <div className={styles['comparison__panel-title']} style={{ padding: '20px 24px', borderBottom: '1px solid #F3F4F6' }}>Metric Breakdown</div>
            <table className="tbl">
              <thead><tr><th>Metric</th>{comp.map((m, i) => <th key={i} style={{ color: COLORS[i] }}>{m.name}</th>)}</tr></thead>
              <tbody>
                {mets.map((met) => (
                  <tr key={met.k}>
                    <td style={{ fontWeight: 700, fontSize: 13 }}>{met.l}</td>
                    {comp.map((m, i) => <td key={i} style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", fontWeight: 700 }}>{m[met.k]}</td>)}
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        </div>

        <div className="card">
          <div className={styles['comparison__panel-title']} style={{ marginBottom: 20 }}>Score Comparison</div>
          <div className={styles['comparison__scores']}>
            {comp.map((m, i) => (
              <div key={i} className={styles['comparison__score-item']}>
                <ScoreRing score={Math.round(m.accuracy)} size={100} stroke={7} color={COLORS[i]} label="ACCURACY" />
                <div style={{ fontWeight: 700, fontSize: 14, textAlign: 'center' }}>{m.name}</div>
                <div style={{ fontSize: 12, color: '#6B7280' }}>{m.ctx.toLocaleString()} ctx</div>
              </div>
            ))}
          </div>
        </div>
      </div>
    </div>
  );
}


















//Dashboard.tsx
import { useEffect } from 'react';
import { useNavigate } from 'react-router-dom';
import { Loader2, TrendingUp, Play, Plus, GitCompare, BookOpen, ChevronRight } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchProviders } from '../../store/slices/providersSlice';
import { fetchModels } from '../../store/slices/modelsSlice';
import { fetchEvaluations } from '../../store/slices/evaluationsSlice';
import ScoreRing from '../common/ScoreRing';
import Sparkline from '../common/Sparkline';
import RadarChart from '../common/RadarChart';
import { useCounter } from '../common/useCounter';
import styles from './Dashboard.module.scss';

export default function Dashboard() {
  const dispatch = useAppDispatch();
  const navigate = useNavigate();

  const providers = useAppSelector((s) => s.providers.items);
  const models = useAppSelector((s) => s.models.items);
  const runs = useAppSelector((s) => s.evaluations.list);

  useEffect(() => {
    dispatch(fetchProviders());
    dispatch(fetchModels());
    dispatch(fetchEvaluations());
  }, [dispatch]);

  const connectedCount = providers.filter((p) => p.status === 'connected').length;
  const totalEvals = useCounter(runs.length, 900);
  const connectedAnim = useCounter(connectedCount, 900);
  const avgAccuracy = models.length
    ? (models.reduce((sum, m) => sum + (m.accuracy_score || 0), 0) / models.length).toFixed(1)
    : '0.0';

  const stats = [
    { label: 'Total Evaluations', value: totalEvals, change: `${runs.length} tracked locally`, spark: [2, 4, 3, 5, 6, 5, runs.length || 1] },
    { label: 'Models Connected', value: connectedAnim, change: `${models.length} in catalog`, spark: [1, 2, 2, 3, 3, 4, connectedCount || 1] },
    { label: 'Avg. Accuracy', value: `${avgAccuracy}%`, change: 'Across active models', spark: [85, 86, 87, 88, 89, 90, Number(avgAccuracy) || 90] },
  ];

  const radarModels = models.slice(0, 3).map((m) => ({
    name: m.name,
    values: [
      (m.accuracy_score || 0) / 100,
      0.8,
      m.input_price ? Math.max(0, 1 - m.input_price / 10) : 0.5,
      Math.min(1, m.context_window / 200000),
      (m.agent_score || m.accuracy_score || 0) / 100,
    ],
  }));

  return (
    <div className="page-enter">
      <div className="pg-hdr"><h1>Dashboard</h1><p>Your evaluation activity at a glance</p></div>
      <div className="pg-body">
        <div className={styles['d-stats']}>
          {stats.map((s, i) => (
            <div className={styles['d-stat']} key={i}>
              <div className={styles['d-stat-top']}>
                <div>
                  <div className={styles['d-stat-label']}>{s.label}</div>
                  <div className={styles['d-stat-val']}>{s.value}</div>
                  <div className={styles['d-stat-change']}><TrendingUp size={12} /> {s.change}</div>
                </div>
                <Sparkline data={s.spark} color="#6366F1" width={72} height={32} />
              </div>
            </div>
          ))}
        </div>

        <div className={styles['dash__grid']}>
          <div className="card" style={{ padding: 0 }}>
            <div className={styles['dash__panel-hdr']}>
              <span>Recent Evaluations</span>
              <button className="btn btn-ghost btn-sm" onClick={() => navigate('/app/history')}>View All <ChevronRight size={14} /></button>
            </div>
            {runs.length === 0 && (
              <div className={styles['dash__empty']}>No evaluations yet — launch one from Quick Actions below.</div>
            )}
            {runs.slice(0, 4).map((run) => (
              <div key={run.id} className={styles['dash__run-row']} onClick={() => navigate(`/app/history?id=${run.id}`)}>
                <div style={{ display: 'flex', alignItems: 'center', gap: 14 }}>
                  {run.status === 'completed' ? (
                    <ScoreRing score={Math.round(run.top_score ?? 0)} size={44} stroke={4} color="#6366F1" />
                  ) : (
                    <div className={styles['dash__spinner']}><Loader2 size={18} color="#6366F1" style={{ animation: 'spin 1.5s linear infinite' }} /></div>
                  )}
                  <div>
                    <div style={{ fontWeight: 600, fontSize: 14 }}>{run.name}</div>
                    <div style={{ fontSize: 12, color: '#6B7280' }}>{run.benchmark || '—'} &middot; {new Date(run.created_at).toLocaleDateString()}</div>
                  </div>
                </div>
                <span className={`badge ${run.status === 'completed' ? 'badge-green' : 'badge-run'}`}>{run.status}</span>
              </div>
            ))}
          </div>

          <div className="card">
            <div style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", fontWeight: 700, fontSize: 15, marginBottom: 4 }}>Top Models</div>
            <div style={{ fontSize: 13, color: '#6B7280', marginBottom: 12 }}>Strength comparison across 5 dimensions</div>
            {radarModels.length > 0 ? (
              <>
                <div className="radar-wrap"><RadarChart models={radarModels} size={260} /></div>
                <div className={styles['dash__legend']}>
                  {radarModels.map((m, i) => (
                    <span key={i}><span className={styles['dash__dot']} style={{ background: ['#6366F1', '#F59E0B', '#10B981'][i] }} /> {m.name}</span>
                  ))}
                </div>
              </>
            ) : (
              <div className={styles['dash__empty']}>Connect a provider to see model comparisons.</div>
            )}
          </div>
        </div>

        <div className="card" style={{ padding: 24 }}>
          <div style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", fontWeight: 700, fontSize: 15, marginBottom: 20 }}>Quick Actions</div>
          <div className={styles['dash__actions']}>
            {[
              { icon: <Play size={20} />, label: 'New Evaluation', desc: 'Start a benchmark run', to: '/app/run-evaluation', cls: 'ind' },
              { icon: <Plus size={20} />, label: 'Add Provider', desc: 'Connect an API', to: '/app/providers', cls: 'em' },
              { icon: <GitCompare size={20} />, label: 'Compare Models', desc: 'Side-by-side analysis', to: '/app/comparison', cls: 'amb' },
              { icon: <BookOpen size={20} />, label: 'Datasets', desc: 'Browse benchmark suites', to: '/app/datasets', cls: 'sky' },
            ].map((a, i) => (
              <button key={i} onClick={() => navigate(a.to)} className={`${styles['qa-btn']} ${styles[`qa-btn--${a.cls}`]}`}>
                <div className={styles['qa-btn__icon']}>{a.icon}</div>
                <div><div className={styles['qa-btn__label']}>{a.label}</div><div className={styles['qa-btn__desc']}>{a.desc}</div></div>
              </button>
            ))}
          </div>
        </div>
      </div>
    </div>
  );
}



















//Datasets.tsx
import { useEffect, useMemo, useState } from 'react';
import { RefreshCw, Search, ExternalLink, Layers, AlertTriangle } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchBenchmarks } from '../../store/slices/benchmarksSlice';
import type { Benchmark } from '../../types';
import styles from './Datasets.module.scss';

// Deterministic color hash so the same capability always gets the same pill
// color across cards/renders, without a hardcoded lookup table.
const PILL_COLORS = [
  { bg: '#EEF2FF', fg: '#6366F1' }, { bg: '#FFFBEB', fg: '#D97706' }, { bg: '#ECFDF5', fg: '#059669' },
  { bg: '#FFF1F2', fg: '#F43F5E' }, { bg: '#F0F9FF', fg: '#0EA5E9' }, { bg: '#FEF2F2', fg: '#EF4444' },
];
function hashColor(label: string) {
  const sum = [...label].reduce((acc, ch) => acc + ch.charCodeAt(0), 0);
  return PILL_COLORS[sum % PILL_COLORS.length];
}

export default function Datasets() {
  const dispatch = useAppDispatch();
  const { items, status, error } = useAppSelector((s) => s.benchmarks);
  const [search, setSearch] = useState('');
  const [typeFilter, setTypeFilter] = useState('All');
  const [subgroupsFor, setSubgroupsFor] = useState<Benchmark | null>(null);

  useEffect(() => {
    dispatch(fetchBenchmarks());
  }, [dispatch]);

  const types = useMemo(() => ['All', ...new Set(items.map((b) => b.type))], [items]);
  const filtered = items.filter((b) => {
    if (typeFilter !== 'All' && b.type !== typeFilter) return false;
    const q = search.toLowerCase();
    return !q || b.name.toLowerCase().includes(q) || b.description.toLowerCase().includes(q);
  });

  return (
    <div className="page-enter">
      <div className={styles.header}>
        <div className={styles.header__eyebrow}>Datasets</div>
        <h1 className={styles.header__title}>Test Suite Library</h1>
        <p className={styles.header__subtitle}>Browse every benchmark available for evaluations, independent of any single wizard run.</p>
        <div className={styles.header__row}>
          <span className={styles.header__count}>{items.length} suites available</span>
          <button className="btn btn-ghost btn-sm" onClick={() => dispatch(fetchBenchmarks())}><RefreshCw size={14} /> Refresh</button>
        </div>
      </div>

      <div className="pg-body" style={{ paddingTop: 0 }}>
        <div className="toolbar">
          <div className="search-box"><Search size={16} color="#9CA3AF" /><input placeholder="Search datasets…" value={search} onChange={(e) => setSearch(e.target.value)} /></div>
          <div className="pills">{types.map((t) => <button key={t} className={`pill ${typeFilter === t ? 'on' : ''}`} onClick={() => setTypeFilter(t)}>{t}</button>)}</div>
        </div>

        {status === 'loading' && (
          <div className={styles.state}><Layers size={28} /><div>Loading datasets…</div></div>
        )}

        {status === 'failed' && (
          <div className={styles.state}>
            <AlertTriangle size={28} color="#EF4444" />
            <div>{error || 'Failed to load datasets.'}</div>
            <button className="btn btn-ind btn-sm" onClick={() => dispatch(fetchBenchmarks())}>Retry</button>
          </div>
        )}

        {status === 'succeeded' && filtered.length === 0 && (
          <div className={styles.state}><Layers size={28} /><div>No datasets match your search.</div></div>
        )}

        <div className="cards-grid">
          {filtered.map((b) => {
            // Defensive per spec §3.2 / §5 — tasks & required_capabilities are
            // normalized to [] in benchmarksApi.list(), but reads still fall
            // back defensively here in case a consumer bypasses that layer.
            const tasks = b.tasks ?? [];
            const caps = b.required_capabilities ?? [];
            return (
              <div className="card" key={b.name}>
                <div className="card-hdr">
                  <div>
                    <div className="card-title">{b.name}</div>
                    <div style={{ marginTop: 6 }}><span className="tag tag-amb">{b.type}</span></div>
                  </div>
                  <div style={{ textAlign: 'right' }}>
                    <div style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", fontSize: 24, fontWeight: 700 }}>{b.task_count.toLocaleString()}</div>
                    <div style={{ fontSize: 11, color: '#9CA3AF', fontWeight: 600 }}>tasks</div>
                  </div>
                </div>
                <div className="card-desc">{b.description}</div>

                <div className={styles.statRow}>
                  <span>{b.task_count.toLocaleString()} Tasks</span>
                  <span>{caps.length} Capabilities</span>
                  <span>{b.huggingface_dataset.split('/')[0]}</span>
                </div>

                <div className={styles.pillRow}>
                  {caps.map((c) => {
                    const color = hashColor(c);
                    return <span key={c} className={styles.pill} style={{ background: color.bg, color: color.fg }}>{c}</span>;
                  })}
                </div>

                <div className="card-foot">
                  {tasks.length > 0 ? (
                    <button className="btn btn-sm btn-ghost" onClick={() => setSubgroupsFor(b)}>View subgroups</button>
                  ) : <span />}
                  <a
                    className={styles.sourceLink}
                    href={`https://huggingface.co/datasets/${b.huggingface_dataset}`}
                    target="_blank" rel="noreferrer"
                  >
                    Source <ExternalLink size={12} />
                  </a>
                </div>
              </div>
            );
          })}
        </div>
      </div>

      {subgroupsFor && (
        <div className={styles.modalOverlay} onClick={() => setSubgroupsFor(null)}>
          <div className={styles.modal} onClick={(e) => e.stopPropagation()}>
            <div className={styles.modal__hdr}>
              <span>{subgroupsFor.name} — subgroups</span>
              <button className="btn btn-sm btn-ghost" onClick={() => setSubgroupsFor(null)}>Close</button>
            </div>
            <div className={styles.modal__body}>
              {(subgroupsFor.tasks ?? []).map((t) => (
                <div key={t.value} className={styles.modal__row}><span>{t.name}</span><code>{t.value}</code></div>
              ))}
            </div>
          </div>
        </div>
      )}
    </div>
  );
}



















//History.tsx
import { useEffect, useMemo, useState } from 'react';
import { useNavigate, useSearchParams } from 'react-router-dom';
import {
  Search, Copy, Trash2, FileBarChart, Cpu, Bot, Database, Loader2,
  Award, ListChecks, Clock,
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import {
  fetchEvaluations, fetchEvaluationResults, removeEvaluationLocal, setDraft,
} from '../../store/slices/evaluationsSlice';
import type { EvaluationListItem, EvaluationStatusValue } from '../../types';
import styles from './History.module.scss';

const TYPE_ICON: Record<string, typeof Cpu> = { model: Cpu, agent: Bot, rag: Database };
const TYPE_LABEL: Record<string, string> = { model: 'AI Model', agent: 'Agent', rag: 'RAG' };

function statusBadgeClass(status: EvaluationStatusValue): string {
  switch (status) {
    case 'completed': return 'badge-green';
    case 'running': return 'badge-run';
    case 'pending': return 'badge-amber';
    case 'failed': case 'canceled': return 'badge-gray';
    default: return 'badge-blue';
  }
}

function withinDateRange(iso: string, range: string): boolean {
  if (range === 'all') return true;
  const days = range === '7' ? 7 : 30;
  const cutoff = Date.now() - days * 86400000;
  return new Date(iso).getTime() >= cutoff;
}

export default function History() {
  const dispatch = useAppDispatch();
  const navigate = useNavigate();
  const [searchParams, setSearchParams] = useSearchParams();
  const selectedId = searchParams.get('id');

  const { list, listStatus, listError, resultsByEvalId, resultsStatusByEvalId, resultsErrorByEvalId } = useAppSelector((s) => s.evaluations);
  const models = useAppSelector((s) => s.models.items);
  const providers = useAppSelector((s) => s.providers.items);

  const [search, setSearch] = useState('');
  const [typeFilter, setTypeFilter] = useState('All');
  const [dateFilter, setDateFilter] = useState('all');

  // Initial load + silent 10s poll (spec §2.4) — no spinner/error disruption
  // on background refreshes; the slice only flips listStatus when list is empty.
  useEffect(() => {
    dispatch(fetchEvaluations());
    const interval = setInterval(() => dispatch(fetchEvaluations()), 10000);
    return () => clearInterval(interval);
  }, [dispatch]);

  const filtered = useMemo(() => {
    return list.filter((e) => {
      if (search && !e.name.toLowerCase().includes(search.toLowerCase())) return false;
      if (typeFilter !== 'All' && e.eval_type !== typeFilter) return false;
      if (!withinDateRange(e.created_at, dateFilter)) return false;
      return true;
    });
  }, [list, search, typeFilter, dateFilter]);

  const selected = list.find((e) => e.id === selectedId) || filtered[0] || null;

  // Keyed on id + status (not the whole object) so a background poll that
  // doesn't change either doesn't re-trigger the results fetch (spec §2.4).
  useEffect(() => {
    if (selected && selected.status === 'completed' && !resultsByEvalId[selected.id]) {
      dispatch(fetchEvaluationResults(selected.id));
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [selected?.id, selected?.status]);

  const selectRow = (id: string) => setSearchParams({ id });

  const duplicate = (e: EvaluationListItem) => {
    dispatch(setDraft({
      name: `${e.name} (copy)`,
      type: (e.eval_type as 'model' | 'agent' | 'rag') || 'model',
      models: e.model_ids,
      dataset: e.benchmark || null,
      subgroup: e.selected_category,
      runSamples: e.run_samples,
      metrics: e.selected_metrics,
    }));
    navigate('/app/run-evaluation');
  };

  const modelName = (id: string) => models.find((m) => m.id === id)?.name || id;
  const providerName = (id: string) => {
    const model = models.find((m) => m.id === id);
    return providers.find((p) => p.id === model?.provider_id)?.name || model?.provider_id || '—';
  };

  const results = selected ? resultsByEvalId[selected.id] : undefined;
  const resultsStatus = selected ? resultsStatusByEvalId[selected.id] : undefined;
  const resultsError = selected ? resultsErrorByEvalId[selected.id] : undefined;

  return (
    <div className="page-enter">
      <div className="pg-hdr"><h1>History</h1><p>All past and in-progress evaluations</p></div>
      <div className="pg-body">
        <div className={styles.layout}>
          {/* ---------- Sidebar list ---------- */}
          <div className={styles.sidebar}>
            <div className="search-box" style={{ minWidth: 0, marginBottom: 10 }}>
              <Search size={16} color="#9CA3AF" />
              <input placeholder="Search evaluations…" value={search} onChange={(e) => setSearch(e.target.value)} />
            </div>
            <div className="pills" style={{ marginBottom: 8 }}>
              {['All', 'model', 'agent', 'rag'].map((t) => (
                <button key={t} className={`pill ${typeFilter === t ? 'on' : ''}`} onClick={() => setTypeFilter(t)}>
                  {t === 'All' ? 'All' : TYPE_LABEL[t]}
                </button>
              ))}
            </div>
            <select className="fi" style={{ marginBottom: 14 }} value={dateFilter} onChange={(e) => setDateFilter(e.target.value)}>
              <option value="all">All time</option>
              <option value="30">Last 30 days</option>
              <option value="7">Last 7 days</option>
            </select>

            {listStatus === 'loading' && list.length === 0 && <div className={styles.empty}>Loading evaluations…</div>}
            {listStatus === 'failed' && list.length === 0 && <div className={styles.empty}>{listError || 'Failed to load evaluations.'}</div>}
            {listStatus !== 'loading' && filtered.length === 0 && list.length > 0 && <div className={styles.empty}>No evaluations match your filters.</div>}

            <div className={styles.rows}>
              {filtered.map((e) => {
                const Icon = TYPE_ICON[e.eval_type] || Cpu;
                const isSelected = selected?.id === e.id;
                return (
                  <div
                    key={e.id}
                    className={`${styles.row} ${isSelected ? styles.selected : ''} ${e.status === 'running' ? styles['row--running'] : ''}`}
                    onClick={() => selectRow(e.id)}
                  >
                    <div className={styles.row__top}>
                      <div className={styles.row__icon}><Icon size={16} /></div>
                      <div className={styles.row__name}>{e.name}</div>
                    </div>
                    <div className={styles.row__badges}>
                      <span className="tag tag-ind">{TYPE_LABEL[e.eval_type] || e.eval_type}</span>
                      <span className={`badge ${statusBadgeClass(e.status)}`}>
                        {e.status === 'running' && <span className={styles['live-dot']} />}
                        {e.status}
                      </span>
                    </div>
                    <div className={styles.row__meta}>{new Date(e.created_at).toLocaleDateString()}</div>
                    <div className={styles.row__stats}>
                      <span>{e.top_model ? `🏆 ${e.top_model}` : '—'}</span>
                      <span>{e.top_score != null ? `${e.top_score}%` : '—'}</span>
                      <span>{e.model_ids.length} models</span>
                    </div>
                    <div className={styles.row__actions} onClick={(ev) => ev.stopPropagation()}>
                      <button className="btn btn-sm btn-ghost" onClick={() => duplicate(e)}><Copy size={12} /> Duplicate</button>
                      <button className="btn btn-sm btn-danger" onClick={() => dispatch(removeEvaluationLocal(e.id))}><Trash2 size={12} /> Delete</button>
                    </div>
                  </div>
                );
              })}
            </div>
          </div>

          {/* ---------- Detail panel ---------- */}
          <div className={styles.detail}>
            {!selected ? (
              <div className={styles['detail-empty']}>Select an evaluation to see its details.</div>
            ) : (
              <>
                <div className={styles['detail-hdr']}>
                  <div>
                    <div className={styles['detail-hdr__badges']}>
                      <span className="tag tag-ind">{TYPE_LABEL[selected.eval_type] || selected.eval_type}</span>
                      <span className={`badge ${statusBadgeClass(selected.status)}`}>
                        {selected.status === 'running' && <span className={styles['live-dot']} />}
                        {selected.status}
                      </span>
                    </div>
                    <h2 className={styles['detail-hdr__name']}>{selected.name}</h2>
                    <div className={styles['detail-hdr__date']}>Created {new Date(selected.created_at).toLocaleString()}</div>
                  </div>
                  <div className={styles['detail-hdr__actions']}>
                    <button className="btn btn-ghost" onClick={() => duplicate(selected)}><Copy size={14} /> Duplicate</button>
                    <button className="btn btn-danger" onClick={() => dispatch(removeEvaluationLocal(selected.id))}><Trash2 size={14} /> Delete</button>
                    <button className="btn btn-ind" disabled={selected.status !== 'completed'}><FileBarChart size={14} /> View Report</button>
                  </div>
                </div>

                <div className={styles['summary-cards']}>
                  <div className={styles['summary-card']}>
                    <Award size={16} color="#F59E0B" />
                    <div>
                      <div className={styles['summary-card__label']}>Winner</div>
                      <div className={styles['summary-card__val']}>{selected.top_model || '—'}{selected.top_score != null ? ` · ${selected.top_score}%` : ''}</div>
                    </div>
                  </div>
                  <div className={styles['summary-card']}>
                    <ListChecks size={16} color="#6366F1" />
                    <div>
                      <div className={styles['summary-card__label']}>Questions / Models</div>
                      <div className={styles['summary-card__val']}>{selected.total_questions.toLocaleString()} &middot; {selected.model_ids.length} models</div>
                    </div>
                  </div>
                  <div className={styles['summary-card']}>
                    <Clock size={16} color="#10B981" />
                    <div>
                      <div className={styles['summary-card__label']}>Status</div>
                      <div className={styles['summary-card__val']}>{selected.status}{selected.completed_at ? ` · ${new Date(selected.completed_at).toLocaleDateString()}` : ''}</div>
                    </div>
                  </div>
                </div>

                {selected.status === 'completed' ? (
                  <>
                    {resultsStatus === 'loading' && <div className={styles.empty}><Loader2 size={16} style={{ animation: 'spin 1.5s linear infinite' }} /> Loading results…</div>}
                    {resultsStatus === 'failed' && <div className={styles.empty}>{resultsError}</div>}
                    {results && (
                      <div className="tw">
                        <table className="tbl">
                          <thead><tr><th>Rank</th><th>Model</th><th>Provider</th><th>Score</th><th>Accuracy</th><th>Passed</th><th>Failed</th></tr></thead>
                          <tbody>
                            {results.results.map((r) => (
                              <tr key={r.model_id} className={r.rank === 1 ? 'winner' : ''}>
                                <td style={{ fontWeight: 700 }}>{r.rank === 1 ? '🏆 ' : ''}{r.rank}</td>
                                <td style={{ fontWeight: 700 }}>{modelName(r.model_id)}</td>
                                <td style={{ color: '#6B7280' }}>{providerName(r.model_id)}</td>
                                <td style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", fontWeight: 700 }}>{r.score}%</td>
                                <td style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif" }}>{r.accuracy}%</td>
                                <td style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", color: '#10B981' }}>{r.passed_tests}</td>
                                <td style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", color: '#EF4444' }}>{r.failed_tests}</td>
                              </tr>
                            ))}
                          </tbody>
                        </table>
                      </div>
                    )}
                  </>
                ) : (
                  <div className={styles['status-message']}>
                    {selected.status === 'running' && 'This evaluation is still running — results will appear once it completes.'}
                    {selected.status === 'pending' && 'This evaluation hasn\u2019t started yet.'}
                    {selected.status === 'failed' && 'This evaluation failed to complete.'}
                    {selected.status === 'canceled' && 'This evaluation was canceled.'}
                  </div>
                )}
              </>
            )}
          </div>
        </div>
      </div>
    </div>
  );
}














//Landing.module.scss
@use '../../styles/_variables' as *;

.landing { min-height: 100vh; background: $surface; overflow-x: hidden; }

.l-nav {
  display: flex; align-items: center; justify-content: space-between; padding: 18px 48px;
  max-width: 1440px; margin: 0 auto; position: relative; z-index: 10;
}
.l-logo {
  font-family: $font-display; font-size: 22px; font-weight: 700; letter-spacing: -.5px;
  display: flex; align-items: center; gap: 10px; color: $text-primary;

  .mark {
    width: 34px; height: 34px; background: $grad-primary; border-radius: 10px;
    display: flex; align-items: center; justify-content: center; color: #fff; font-size: 18px;
    box-shadow: 0 2px 8px rgba(99, 102, 241, .35);
  }
}

.btn-primary {
  display: inline-flex; align-items: center; gap: 8px; background: $grad-primary; color: #fff; border: none;
  padding: 14px 28px; border-radius: 14px; font-size: 15px; font-weight: 600; cursor: pointer; transition: all .25s;
  font-family: $font-display; box-shadow: 0 4px 14px rgba(99, 102, 241, .3);
}
.btn-primary:hover { transform: translateY(-2px); box-shadow: 0 8px 24px rgba(99, 102, 241, .35); }
.btn-primary:disabled { opacity: .6; cursor: default; transform: none; }

.btn-secondary {
  display: inline-flex; align-items: center; gap: 8px; background: $surface; color: $text-primary;
  border: 1px solid $border; padding: 14px 28px; border-radius: 14px; font-size: 15px; font-weight: 600;
  cursor: pointer; transition: all .25s; font-family: $font-display;
}
.btn-secondary:hover { border-color: $indigo; color: $indigo; background: $indigo-pale; box-shadow: $shadow-2; }

.hero-section {
  position: relative; max-width: 1440px; margin: 0 auto; padding: 64px 48px 80px;
  display: grid; grid-template-columns: 1fr 1fr; gap: 80px; align-items: center;
}
.hero-bg-dots {
  position: absolute; inset: 0; background-image: radial-gradient(circle, $border 1px, transparent 1px);
  background-size: 24px 24px; opacity: .5; pointer-events: none; animation: dotPulse 4s ease infinite;
}
.hero-content { position: relative; z-index: 2; }
.hero-badge {
  display: inline-flex; align-items: center; gap: 8px; background: $indigo-pale;
  border: 1px solid rgba(99, 102, 241, .15); border-radius: 100px; padding: 6px 16px 6px 8px;
  font-size: 13px; color: $indigo; margin-bottom: 28px; font-weight: 600;

  .badge-dot { width: 8px; height: 8px; border-radius: 50%; background: $indigo; animation: pulse 2s infinite; }
}
.hero-content h1 {
  font-size: 54px; font-weight: 700; line-height: 1.08; letter-spacing: -2.5px; margin-bottom: 24px;
  .grad { background: $grad-primary; -webkit-background-clip: text; -webkit-text-fill-color: transparent; }
}
.hero-content > p { font-size: 17px; color: $text-secondary; line-height: 1.7; margin-bottom: 40px; max-width: 480px; }
.hero-actions { display: flex; gap: 14px; }

.hero-visual { position: relative; z-index: 2; }
.hero-card { background: $surface; border: 1px solid $border; border-radius: 20px; padding: 28px; box-shadow: $shadow-4; }
.hero-card-hdr { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; }
.hero-card-title { font-family: $font-display; font-size: 13px; font-weight: 700; color: $text-muted; text-transform: uppercase; letter-spacing: 1.5px; }
.hero-bar { display: flex; align-items: center; gap: 12px; margin-bottom: 12px; }
.hero-bar:last-child { margin-bottom: 0; }
.hero-bar-label { font-size: 13px; color: $text-secondary; width: 120px; flex-shrink: 0; font-weight: 600; }
.hero-bar-track { flex: 1; height: 34px; background: $surface-alt; border-radius: 10px; overflow: hidden; }
.hero-bar-fill {
  height: 100%; border-radius: 10px; display: flex; align-items: center; padding: 0 14px;
  font-family: $font-mono; font-size: 12px; font-weight: 700; color: #fff; transition: width 1.8s cubic-bezier(.16, 1, .3, 1);

  &--primary { background: $grad-primary; }
  &--warm { background: $grad-warm; }
  &--cool { background: $grad-cool; }
  &--gray { background: linear-gradient(135deg, #6B7280, #9CA3AF); }
}

.float-badge {
  position: absolute; background: $surface; border: 1px solid $border; border-radius: 14px;
  padding: 12px 18px; display: flex; align-items: center; gap: 10px; box-shadow: $shadow-3;
  z-index: 3; animation: float 4s ease infinite; font-size: 13px; color: $text-secondary;
  strong { color: $text-primary; }
}
.float-badge.tr { top: -24px; right: -16px; animation-delay: .5s; }
.float-badge.bl { bottom: -20px; left: -16px; animation-delay: 1.5s; }
.pulse-dot { width: 8px; height: 8px; border-radius: 50%; background: $emerald; animation: pulse 2s infinite; }

.features { max-width: 1440px; margin: 0 auto; padding: 96px 48px; background: $bg; }
.feat-header { text-align: center; margin-bottom: 72px; }
.feat-header h2 { font-size: 38px; font-weight: 700; letter-spacing: -1.5px; margin-bottom: 14px; }
.feat-header p { color: $text-secondary; font-size: 16px; max-width: 460px; margin: 0 auto; line-height: 1.6; }
.feat-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 20px; }
.feat-card {
  background: $surface; border: 1px solid $border; border-radius: 18px; padding: 32px;
  transition: all .3s; cursor: default; position: relative; overflow: hidden;

  &::before { content: ''; position: absolute; top: 0; left: 0; right: 0; height: 3px; background: $grad-primary; opacity: 0; transition: opacity .3s; }
  &:hover { border-color: transparent; box-shadow: $shadow-3; transform: translateY(-6px); }
  &:hover::before { opacity: 1; }
  h3 { font-size: 17px; font-weight: 700; margin-bottom: 8px; letter-spacing: -.2px; }
  p { font-size: 14px; color: $text-secondary; line-height: 1.65; }
}
.feat-icon {
  width: 50px; height: 50px; border-radius: 14px; display: flex; align-items: center; justify-content: center; margin-bottom: 22px;
  &--ind { background: $indigo-pale; color: $indigo; }
  &--amb { background: $amber-pale; color: $amber; }
  &--em { background: $emerald-pale; color: $emerald; }
  &--sky { background: $sky-pale; color: $sky; }
  &--rose { background: $rose-pale; color: $rose; }
}

.stats-section {
  max-width: 1440px; margin: 0 auto; padding: 64px 48px; display: grid; grid-template-columns: repeat(4, 1fr);
  gap: 24px; background: $surface; border-top: 1px solid $border; border-bottom: 1px solid $border;
}
.stat-box { text-align: center; padding: 16px; }
.stat-val { font-family: $font-mono; font-size: 44px; font-weight: 700; letter-spacing: -2px; background: $grad-primary; -webkit-background-clip: text; -webkit-text-fill-color: transparent; }
.stat-lbl { font-size: 14px; color: $text-secondary; margin-top: 4px; font-weight: 500; }

.cta-section { max-width: 1440px; margin: 0 auto; padding: 96px 48px; text-align: center; background: $bg; }
.cta-box {
  background: $surface; border: 1px solid $border; border-radius: 28px; padding: 72px;
  position: relative; overflow: hidden; box-shadow: $shadow-2;
  &::before { content: ''; position: absolute; inset: -2px; background: $grad-primary; border-radius: 30px; z-index: -1; opacity: .15; }
  h2 { font-size: 38px; font-weight: 700; letter-spacing: -1.5px; margin-bottom: 14px; }
  p { color: $text-secondary; font-size: 16px; margin-bottom: 36px; line-height: 1.6; }
}

// .l-footer moved to src/components/layout/Footer.module.scss — shared with /app.

@media (max-width: 768px) {
  .hero-section { grid-template-columns: 1fr; padding: 48px 24px; gap: 40px; }
  .hero-content h1 { font-size: 36px; }
  .feat-grid { grid-template-columns: 1fr; }
  .stats-section { grid-template-columns: repeat(2, 1fr); }
}















//Landing.tsx
import { useEffect, useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { ArrowRight, Play, Award, Link2, Cpu, FlaskConical, BarChart3, GitCompare, Shield } from 'lucide-react';
import Footer from '../layout/Footer';
import styles from './Landing.module.scss';

const features = [
  { icon: <Link2 size={22} />, cls: 'ind', title: 'Provider Hub', desc: 'Connect any AI provider with API keys. Manage credentials, monitor status, and see available models instantly.' },
  { icon: <Cpu size={22} />, cls: 'amb', title: 'Model Catalog', desc: 'Browse all models across providers. Filter by capability, compare pricing, and register custom endpoints.' },
  { icon: <FlaskConical size={22} />, cls: 'em', title: 'Guided Evaluations', desc: 'A step-by-step wizard for model selection, test suite choice, and metric configuration.' },
  { icon: <BarChart3 size={22} />, cls: 'sky', title: 'Results & History', desc: 'Every evaluation stored with full breakdowns. Duplicate past runs, track trends, export findings.' },
  { icon: <GitCompare size={22} />, cls: 'rose', title: 'Visual Comparison', desc: 'Radar charts and metric tables make it obvious where each model excels or falls short.' },
  { icon: <Shield size={22} />, cls: 'ind', title: 'SSO & Security', desc: 'Enterprise-grade sign-in. API keys encrypted and isolated to your environment.' },
];

export default function Landing() {
  const [animated, setAnimated] = useState(false);
  const navigate = useNavigate();

  useEffect(() => {
    const t = setTimeout(() => setAnimated(true), 200);
    return () => clearTimeout(t);
  }, []);

  // Landing is now gated by AuthGuard (wrapping "/"), so by the time this
  // page renders the user is already authenticated — these buttons are
  // plain navigation into the app, not sign-in triggers.
  const goToDashboard = () => navigate('/app/dashboard');

  const bars = [
    { label: 'Claude Sonnet 4', pct: animated ? 94 : 0, cls: 'primary' },
    { label: 'GPT-4o', pct: animated ? 91 : 0, cls: 'warm' },
    { label: 'Gemini 2.5 Pro', pct: animated ? 89 : 0, cls: 'cool' },
    { label: 'Mistral Large', pct: animated ? 85 : 0, cls: 'gray' },
  ];

  return (
    <div className={styles.landing}>
      <nav className={styles['l-nav']}>
        <div className={styles['l-logo']}><div className={styles.mark}>&#9670;</div>SemcoEval</div>
      </nav>

      <section className={styles['hero-section']}>
        <div className={styles['hero-bg-dots']} />
        <div className={styles['hero-content']}>
          <div className={styles['hero-badge']}><div className={styles['badge-dot']} /> Now supporting 40+ models</div>
          <h1>Evaluate AI models<br />with <span className={styles.grad}>measured evidence</span></h1>
          <p>Stop guessing which model fits your use case. Run structured benchmarks, compare results side-by-side, and make selection decisions backed by real data.</p>
          <div className={styles['hero-actions']}>
            <button className={styles['btn-primary']} onClick={goToDashboard}>Open Dashboard <ArrowRight size={16} /></button>
            <button className={styles['btn-secondary']}><Play size={16} /> Watch Demo</button>
          </div>
        </div>
        <div className={styles['hero-visual']}>
          <div className={styles['hero-card']}>
            <div className={styles['hero-card-hdr']}>
              <span className={styles['hero-card-title']}>Live Benchmark</span>
              <span className="badge badge-green"><div className={styles['pulse-dot']} /> Running</span>
            </div>
            {bars.map((b, i) => (
              <div className={styles['hero-bar']} key={i}>
                <span className={styles['hero-bar-label']}>{b.label}</span>
                <div className={styles['hero-bar-track']}>
                  <div className={`${styles['hero-bar-fill']} ${styles[`hero-bar-fill--${b.cls}`]}`} style={{ width: `${b.pct}%`, transitionDelay: `${i * 250}ms` }}>
                    {b.pct > 0 && <span>{b.pct}%</span>}
                  </div>
                </div>
              </div>
            ))}
          </div>
          <div className={`${styles['float-badge']} ${styles.tr}`}><Award size={16} style={{ color: '#F59E0B' }} /><span>Winner: <strong>Claude Sonnet 4</strong></span></div>
          <div className={`${styles['float-badge']} ${styles.bl}`}><div className={styles['pulse-dot']} /><span>3 evaluations running</span></div>
        </div>
      </section>

      <section className={styles.features}>
        <div className={styles['feat-header']}><h2>Everything you need to decide</h2><p>From connecting providers to comparing results — a complete evaluation workflow</p></div>
        <div className={styles['feat-grid']}>
          {features.map((f, i) => (
            <div className={styles['feat-card']} key={i}>
              <div className={`${styles['feat-icon']} ${styles[`feat-icon--${f.cls}`]}`}>{f.icon}</div>
              <h3>{f.title}</h3>
              <p>{f.desc}</p>
            </div>
          ))}
        </div>
      </section>

      <section className={styles['stats-section']}>
        {[{ v: '40+', l: 'Models supported' }, { v: '6', l: 'Benchmark suites' }, { v: '12K+', l: 'Evaluation tasks' }, { v: '<5min', l: 'Average eval time' }].map((s, i) => (
          <div className={styles['stat-box']} key={i}><div className={styles['stat-val']}>{s.v}</div><div className={styles['stat-lbl']}>{s.l}</div></div>
        ))}
      </section>

      <section className={styles['cta-section']}>
        <div className={styles['cta-box']}>
          <h2>Ready to evaluate with confidence?</h2>
          <p>Connect your first provider and run a benchmark in under five minutes.</p>
          <button className={styles['btn-primary']} onClick={goToDashboard}>Get Started Free <ArrowRight size={16} /></button>
        </div>
      </section>

      <Footer />
    </div>
  );
}


















//AppShell.module.scss
@use '../../styles/_variables' as *;

.app-shell {
  display: flex;
  height: 100vh;
  background: $bg;

  &__main {
    flex: 1;
    overflow-y: auto;
    background: $bg;
    display: flex;
    flex-direction: column;
    min-height: 100%;
  }

  &__content {
    flex: 1;
  }
}














//AppShell.tsx
import { Outlet } from 'react-router-dom';
import Sidebar from './Sidebar';
import Footer from './Footer';
import styles from './AppShell.module.scss';

export default function AppShell() {
  return (
    <div className={styles['app-shell']}>
      <Sidebar />
      <main className={styles['app-shell__main']}>
        <div className={styles['app-shell__content']}>
          <Outlet />
        </div>
        <Footer />
      </main>
    </div>
  );
}
















//Footer.module.scss
@use '../../styles/_variables' as *;

.footer {
  text-align: center;
  padding: 32px 48px;
  color: $text-muted;
  font-size: 13px;
  border-top: 1px solid $border;
  background: $surface;
  flex-shrink: 0;
}


















//Footer.tsx
// Shared footer used on both the public Landing page and every /app route
// (rendered from AppShell). Keeping it as one component means copy/links
// only ever need to be updated in one place.
import styles from './Footer.module.scss';

export default function Footer() {
  const year = new Date().getFullYear();
  return (
    <footer className={styles.footer}>
      &copy; {year} SemcoEval &middot; Privacy &middot; Terms &middot; Documentation
    </footer>
  );
}
















//ModelCatalog.tsx
import { useEffect, useMemo, useState } from 'react';
import { Search, Plus } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchModels, createCustomModel } from '../../store/slices/modelsSlice';
import { fetchProviders } from '../../store/slices/providersSlice';
import AddCustomModelDrawer from './AddCustomModelDrawer';
import styles from './ModelCatalog.module.scss';

export default function ModelCatalog() {
  const dispatch = useAppDispatch();
  const { items, status, creating } = useAppSelector((s) => s.models);
  const providers = useAppSelector((s) => s.providers.items);
  const [search, setSearch] = useState('');
  const [capFilter, setCapFilter] = useState('All');
  const [drawerOpen, setDrawerOpen] = useState(false);

  useEffect(() => {
    dispatch(fetchModels());
    dispatch(fetchProviders());
  }, [dispatch]);

  const caps = useMemo(() => ['All', ...new Set(items.flatMap((m) => m.capabilities))], [items]);
  const providerName = (id: string) => providers.find((p) => p.id === id)?.name || id;

  const filtered = items.filter((m) => {
    if (capFilter !== 'All' && !m.capabilities.includes(capFilter)) return false;
    const q = search.toLowerCase();
    return !q || m.name.toLowerCase().includes(q) || providerName(m.provider_id).toLowerCase().includes(q);
  });

  return (
    <div className="page-enter">
      <div className="pg-hdr"><h1>Model Catalog</h1><p>All models across connected providers</p></div>
      <div className="pg-body">
        <div className="toolbar">
          <div className="search-box">
            <Search size={16} color="#9CA3AF" />
            <input placeholder="Search models or providers…" value={search} onChange={(e) => setSearch(e.target.value)} />
          </div>
          <div style={{ display: 'flex', gap: 8, alignItems: 'center' }}>
            <div className="pills">{caps.map((c) => <button key={c} className={`pill ${capFilter === c ? 'on' : ''}`} onClick={() => setCapFilter(c)}>{c}</button>)}</div>
            <button className="btn btn-ind btn-sm" onClick={() => setDrawerOpen(true)}><Plus size={14} /> Register Custom</button>
          </div>
        </div>

        {status === 'loading' && <div className={styles['model-catalog__loading']}>Loading models…</div>}

        <div className="tw">
          <table className="tbl">
            <thead>
              <tr><th>Model</th><th>Provider</th><th>Capabilities</th><th>Context</th><th>Price (in/out)</th><th>Accuracy</th><th>Status</th></tr>
            </thead>
            <tbody>
              {filtered.map((m) => (
                <tr key={m.id}>
                  <td style={{ fontWeight: 700 }}>{m.name}</td>
                  <td style={{ color: '#6B7280' }}>{providerName(m.provider_id)}</td>
                  <td>{m.capabilities.map((c) => <span key={c} className="tag tag-ind">{c}</span>)}</td>
                  <td style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", fontSize: 13 }}>{m.context_window.toLocaleString()}</td>
                  <td style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", fontSize: 13, color: '#6B7280' }}>
                    {m.input_price != null ? `$${m.input_price.toFixed(2)}` : '—'} / {m.output_price != null ? `$${m.output_price.toFixed(2)}` : '—'}
                  </td>
                  <td>
                    <span style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", fontWeight: 700, color: (m.accuracy_score || 0) >= 90 ? '#10B981' : '#111827' }}>
                      {m.accuracy_score != null ? `${m.accuracy_score}%` : '—'}
                    </span>
                  </td>
                  <td><span className={`badge ${m.is_active ? 'badge-green' : 'badge-gray'}`}>{m.is_active ? 'Active' : 'Inactive'}</span></td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      </div>

      {drawerOpen && (
        <AddCustomModelDrawer
          submitting={creating}
          onClose={() => setDrawerOpen(false)}
          onSubmit={(payload) => {
            dispatch(createCustomModel(payload)).then(() => setDrawerOpen(false));
          }}
        />
      )}
    </div>
  );
}

















//AppRoutes.tsx
import { Routes, Route, Navigate } from 'react-router-dom';
import Landing from '../components/landing/Landing';
import SsoLogin from '../components/auth/SsoLogin';
import AuthGuard from '../components/AuthGuard/AuthGuard';
import AppShell from '../components/layout/AppShell';
import Dashboard from '../components/dashboard/Dashboard';
import Providers from '../components/providers/Providers';
import ModelCatalog from '../components/models/ModelCatalog';
import Datasets from '../components/datasets/Datasets';
import NewEvaluation from '../components/evaluations/NewEvaluation';
import History from '../components/history/History';
import Comparison from '../components/comparison/Comparison';

export default function AppRoutes() {
  return (
    <Routes>
      {/* AuthGuard redirects here with { from, errorMessage } on error/logged_out.
          This is the only route not gated by AuthGuard, so it must stay outside it. */}
      <Route path="/sso-login" element={<SsoLogin />} />

      {/* AuthGuard now wraps BOTH "/" and "/app/*" — the landing page's content
          only renders once authenticated, same as the rest of the app. It
          triggers the SSO WebSocket handshake (useSsoAuth) as soon as it
          mounts, which happens on every fresh page load / refresh (auth
          state is in-memory only, never persisted), and shows AuthSpinner
          while that's in flight. */}
      <Route element={<AuthGuard />}>
        <Route path="/" element={<Landing />} />

        <Route path="/app" element={<AppShell />}>
          <Route index element={<Navigate to="dashboard" replace />} />
          <Route path="dashboard" element={<Dashboard />} />
          <Route path="providers" element={<Providers />} />
          <Route path="models" element={<ModelCatalog />} />
          <Route path="datasets" element={<Datasets />} />
          <Route path="run-evaluation" element={<NewEvaluation />} />
          <Route path="history" element={<History />} />
          <Route path="comparison" element={<Comparison />} />
        </Route>
      </Route>

      <Route path="*" element={<Navigate to="/" replace />} />
    </Routes>
  );
}



















//_variables.scss
// Design tokens — ported 1:1 from the original theme object (T)
$bg: #F7F8FC;
$surface: #FFFFFF;
$surface-alt: #F1F4F9;
$surface-hover: #F8F9FD;

$indigo: #6366F1;
$indigo-light: #818CF8;
$indigo-dark: #4F46E5;
$violet: #8B5CF6;
$indigo-pale: #EEF2FF;

$amber: #F59E0B;
$amber-dark: #D97706;
$amber-pale: #FFFBEB;

$emerald: #10B981;
$emerald-dark: #059669;
$emerald-pale: #ECFDF5;

$red: #EF4444;
$red-pale: #FEF2F2;
$sky: #0EA5E9;
$sky-pale: #F0F9FF;
$rose: #F43F5E;
$rose-pale: #FFF1F2;

$border: #E5E7EB;
$border-light: #F3F4F6;

$text-primary: #111827;
$text-secondary: #6B7280;
$text-muted: #9CA3AF;

$shadow-2: 0 2px 8px rgba(0, 0, 0, .06), 0 1px 2px rgba(0, 0, 0, .04);
$shadow-3: 0 8px 24px rgba(0, 0, 0, .08), 0 2px 6px rgba(0, 0, 0, .04);
$shadow-4: 0 16px 48px rgba(0, 0, 0, .1), 0 4px 12px rgba(0, 0, 0, .05);

$grad-primary: linear-gradient(135deg, #6366F1, #8B5CF6);
$grad-warm: linear-gradient(135deg, #F59E0B, #F97316);
$grad-cool: linear-gradient(135deg, #10B981, #0EA5E9);

$font-display: 'Segoe UI', Roboto, Arial, sans-serif;
$font-body: 'Segoe UI', Roboto, Arial, sans-serif;
$font-mono: 'Segoe UI', Roboto, Arial, sans-serif;



















//global.scss
@use './_variables' as *;

* { margin: 0; padding: 0; box-sizing: border-box; }

body {
  font-family: $font-body;
  background: $bg;
  color: $text-primary;
  -webkit-font-smoothing: antialiased;
}

h1, h2, h3, h4, h5 { font-family: $font-display; }

.page-enter { animation: pageIn .35s ease both; }
@keyframes pageIn { from { opacity: 0; transform: translateY(12px); } to { opacity: 1; transform: translateY(0); } }
@keyframes spin { to { transform: rotate(360deg); } }
@keyframes pulse { 0%, 100% { opacity: 1; } 50% { opacity: .4; } }
@keyframes float { 0%, 100% { transform: translateY(0); } 50% { transform: translateY(-8px); } }
@keyframes dotPulse { 0%, 100% { opacity: .15; } 50% { opacity: .35; } }
@keyframes toastIn { from { opacity: 0; transform: translateY(20px) scale(.95); } to { opacity: 1; transform: translateY(0) scale(1); } }

// ---- Shared primitives reused across feature components ----
.btn {
  display: inline-flex; align-items: center; gap: 6px;
  padding: 10px 20px; border-radius: 12px; font-size: 14px; font-weight: 600;
  cursor: pointer; transition: all .2s; border: none; font-family: $font-display;
}
.btn-ind { background: $grad-primary; color: #fff; box-shadow: 0 2px 8px rgba(99, 102, 241, .2); }
.btn-ind:hover { box-shadow: 0 4px 16px rgba(99, 102, 241, .3); transform: translateY(-1px); }
.btn-ghost { background: $surface; color: $text-secondary; border: 1px solid $border; }
.btn-ghost:hover { border-color: $indigo; color: $indigo; background: $indigo-pale; }
.btn-sm { padding: 7px 14px; font-size: 13px; border-radius: 10px; }
.btn-danger { background: $red-pale; color: $red; border: 1px solid transparent; }
.btn-danger:hover { background: #FEE2E2; }
.btn:disabled { opacity: .45; cursor: not-allowed; }

.badge {
  display: inline-flex; align-items: center; gap: 6px;
  padding: 5px 12px; border-radius: 100px; font-size: 12px; font-weight: 700; letter-spacing: .2px;
}
.badge-green { background: $emerald-pale; color: $emerald-dark; }
.badge-gray { background: $surface-alt; color: $text-muted; }
.badge-blue { background: $indigo-pale; color: $indigo; }
.badge-amber { background: $amber-pale; color: $amber-dark; }
.badge-run { background: $indigo-pale; color: $indigo; }

.tag { display: inline-block; padding: 3px 9px; border-radius: 7px; font-size: 11px; font-weight: 700; margin-right: 4px; margin-bottom: 4px; }
.tag-ind { background: $indigo-pale; color: $indigo; }
.tag-amb { background: $amber-pale; color: $amber-dark; }
.tag-em { background: $emerald-pale; color: $emerald-dark; }

.search-box {
  display: flex; align-items: center; gap: 8px;
  background: $surface; border: 1px solid $border; border-radius: 12px;
  padding: 10px 16px; min-width: 300px; transition: all .2s;
}
.search-box:focus-within { border-color: $indigo; box-shadow: 0 0 0 4px rgba(99, 102, 241, .08); }
.search-box input { border: none; outline: none; font-size: 14px; flex: 1; color: $text-primary; font-family: $font-body; background: transparent; }
.search-box input::placeholder { color: $text-muted; }

.pills { display: flex; gap: 6px; flex-wrap: wrap; }
.pill {
  padding: 7px 16px; border-radius: 100px; font-size: 13px; font-weight: 600;
  border: 1px solid $border; background: $surface; color: $text-secondary; cursor: pointer; transition: all .2s;
}
.pill:hover { border-color: $indigo; color: $indigo; }
.pill.on { background: $grad-primary; color: #fff; border-color: transparent; box-shadow: 0 2px 8px rgba(99, 102, 241, .25); }

.toolbar { display: flex; align-items: center; justify-content: space-between; margin-bottom: 24px; flex-wrap: wrap; gap: 12px; }

.cards-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(330px, 1fr)); gap: 16px; }
.card {
  background: $surface; border: 1px solid $border; border-radius: 16px; padding: 24px;
  transition: all .25s; position: relative; overflow: hidden;
}
.card:hover { border-color: rgba(99, 102, 241, .2); box-shadow: $shadow-3; transform: translateY(-2px); }
.card-hdr { display: flex; justify-content: space-between; align-items: flex-start; margin-bottom: 14px; }
.card-icon { width: 44px; height: 44px; border-radius: 12px; display: flex; align-items: center; justify-content: center; flex-shrink: 0; }
.card-title { font-family: $font-display; font-size: 16px; font-weight: 700; }
.card-desc { font-size: 13px; color: $text-secondary; line-height: 1.6; margin-bottom: 16px; }
.card-foot { display: flex; justify-content: space-between; align-items: center; margin-top: 16px; padding-top: 16px; border-top: 1px solid $border-light; }

.tw { background: $surface; border: 1px solid $border; border-radius: 16px; overflow: hidden; }
.tbl { width: 100%; border-collapse: collapse; }
.tbl th {
  text-align: left; padding: 14px 20px; font-size: 11px; font-weight: 700; color: $text-muted;
  text-transform: uppercase; letter-spacing: 1px; background: $surface-alt; border-bottom: 1px solid $border;
  font-family: $font-display;
}
.tbl td { padding: 14px 20px; font-size: 14px; border-bottom: 1px solid $border-light; }
.tbl tr:last-child td { border-bottom: none; }
.tbl tr { transition: background .15s; }
.tbl tr:hover td { background: $surface-hover; }
.tbl .winner td { background: $amber-pale; }
.tbl .winner td:first-child { box-shadow: inset 3px 0 0 $amber; }

.fg { margin-bottom: 22px; }
.fl { display: block; font-size: 13px; font-weight: 700; margin-bottom: 8px; }
.fl .opt { color: $text-muted; font-weight: 400; font-size: 12px; }
.fi {
  width: 100%; padding: 12px 16px; border: 1px solid $border; border-radius: 12px;
  font-size: 14px; font-family: $font-body; color: $text-primary; transition: all .2s; background: $surface;
}
.fi:focus { outline: none; border-color: $indigo; box-shadow: 0 0 0 4px rgba(99, 102, 241, .08); }
.fi::placeholder { color: $text-muted; }

.sel-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(200px, 1fr)); gap: 10px; }
.sel-opt {
  display: flex; align-items: center; gap: 12px; padding: 16px;
  border: 1.5px solid $border; border-radius: 14px; cursor: pointer; transition: all .2s; background: $surface;
}
.sel-opt:hover { border-color: $indigo-light; background: rgba(238, 242, 255, .4); }
.sel-opt.on { border-color: $indigo; background: $indigo-pale; }
.sel-chk {
  width: 22px; height: 22px; border: 2px solid $border; border-radius: 7px;
  display: flex; align-items: center; justify-content: center; flex-shrink: 0; transition: all .2s;
}
.sel-opt.on .sel-chk { background: $grad-primary; border-color: $indigo; color: #fff; box-shadow: 0 2px 4px rgba(99, 102, 241, .25); }
.sel-txt { font-size: 14px; font-weight: 600; }
.sel-sub { font-size: 12px; color: $text-secondary; margin-top: 2px; }

.toast {
  position: fixed; bottom: 32px; right: 32px; background: $surface; border: 1px solid $border;
  border-radius: 16px; padding: 18px 24px; display: flex; align-items: center; gap: 12px;
  box-shadow: $shadow-4; z-index: 999; animation: toastIn .4s ease both;
}

// Shared by Dashboard and Comparison — kept global since both need it.
.radar-wrap { display: flex; justify-content: center; align-items: center; padding: 20px; }

.pg-hdr { padding: 32px 40px 0; }
.pg-hdr h1 { font-size: 28px; font-weight: 700; letter-spacing: -.5px; }
.pg-hdr p { color: $text-secondary; font-size: 14px; margin-top: 4px; }
.pg-body { padding: 24px 40px 40px; }

@media (max-width: 768px) {
  .cards-grid { grid-template-columns: 1fr; }
  .pg-hdr, .pg-body { padding-left: 20px; padding-right: 20px; }
}
