//Comparison.tsx
import { useEffect, useMemo, useState } from 'react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchModels } from '../../store/slices/modelsSlice';
import { fetchBenchmarks } from '../../store/slices/benchmarksSlice';
import RadarChart from '../common/RadarChart';
import ScoreRing from '../common/ScoreRing';
import styles from './Comparison.module.scss';

const COLORS = ['#1428A0', '#F59E0B', '#10B981'];

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
    <div className="page-enter pg-shell">
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
    <div className="page-enter pg-shell">
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
                <Sparkline data={s.spark} color="#1428A0" width={72} height={32} />
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
                    <ScoreRing score={Math.round(run.top_score ?? 0)} size={44} stroke={4} color="#1428A0" />
                  ) : (
                    <div className={styles['dash__spinner']}><Loader2 size={18} color="#1428A0" style={{ animation: 'spin 1.5s linear infinite' }} /></div>
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
                    <span key={i}><span className={styles['dash__dot']} style={{ background: ['#1428A0', '#F59E0B', '#10B981'][i] }} /> {m.name}</span>
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



















//Datasets.module.scss
@use '../../styles/_variables' as *;

.header { flex-shrink: 0; padding: 32px 40px 0; }
.header__eyebrow { font-size: 12px; font-weight: 700; color: $indigo; text-transform: uppercase; letter-spacing: 1.5px; margin-bottom: 6px; }
.header__title { font-size: 28px; font-weight: 700; letter-spacing: -.5px; }
.header__subtitle { color: $text-secondary; font-size: 14px; margin-top: 6px; max-width: 560px; }
.header__row { display: flex; align-items: center; gap: 14px; margin-top: 16px; padding-bottom: 20px; }
.header__count { font-size: 13px; font-weight: 700; color: $text-secondary; }

.state {
  display: flex; flex-direction: column; align-items: center; gap: 12px; padding: 64px 24px;
  color: $text-secondary; background: $surface; border: 1px solid $border; border-radius: 16px; text-align: center;
}

.statRow {
  display: flex; gap: 14px; font-size: 12px; color: $text-secondary; font-weight: 600;
  padding: 10px 0; border-top: 1px solid $border-light; border-bottom: 1px solid $border-light; margin-bottom: 10px;
}
.pillRow { display: flex; flex-wrap: wrap; gap: 6px; margin-bottom: 4px; }
.pill { padding: 3px 10px; border-radius: 100px; font-size: 11px; font-weight: 700; }

.sourceLink { display: inline-flex; align-items: center; gap: 4px; font-size: 12px; color: $indigo; font-weight: 600; text-decoration: none; }
.sourceLink:hover { text-decoration: underline; }

.modalOverlay {
  position: fixed; inset: 0; background: rgba(17,24,39,.45); z-index: 200;
  display: flex; align-items: center; justify-content: center; padding: 24px;
}
.modal {
  width: 100%; max-width: 480px; max-height: 70vh; background: $surface; border-radius: 18px;
  box-shadow: $shadow-4; display: flex; flex-direction: column; overflow: hidden;
}
.modal__hdr {
  padding: 16px 20px; border-bottom: 1px solid $border-light; display: flex; justify-content: space-between; align-items: center;
  font-weight: 700; font-size: 14px;
}
.modal__body { padding: 8px 12px; overflow-y: auto; }
.modal__row {
  display: flex; justify-content: space-between; align-items: center; padding: 10px 12px; font-size: 13px;
  border-bottom: 1px solid $border-light;
  code { font-family: $font-mono; font-size: 11px; color: $text-muted; }
}
.modal__row:last-child { border-bottom: none; }


























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
  { bg: '#E9EBF8', fg: '#1428A0' }, { bg: '#FFFBEB', fg: '#D97706' }, { bg: '#ECFDF5', fg: '#059669' },
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
    <div className="page-enter pg-shell">
      <div className={styles.header}>
        <div className={styles.header__eyebrow}>Datasets</div>
        <h1 className={styles.header__title}>Test Suite Library</h1>
        <p className={styles.header__subtitle}>Browse every benchmark available for evaluations, independent of any single wizard run.</p>
        <div className={styles.header__row}>
          <span className={styles.header__count}>{items.length} suites available</span>
          <button className="btn btn-ghost btn-sm" onClick={() => dispatch(fetchBenchmarks())}><RefreshCw size={14} /> Refresh</button>
        </div>
      </div>

      <div className="pg-toolbar" style={{ paddingTop: 0 }}>
        <div className="toolbar">
          <div className="search-box"><Search size={16} color="#9CA3AF" /><input placeholder="Search datasets…" value={search} onChange={(e) => setSearch(e.target.value)} /></div>
          <div className="pills">{types.map((t) => <button key={t} className={`pill ${typeFilter === t ? 'on' : ''}`} onClick={() => setTypeFilter(t)}>{t}</button>)}</div>
        </div>
      </div>

      <div className="pg-body" style={{ paddingTop: 0 }}>

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





















//NewEvaluation.tsx
import { useEffect, useMemo, useState } from 'react';
import { useNavigate } from 'react-router-dom';
import {
  ChevronLeft, ChevronRight, Check, Cpu, Bot, Database, Play,
  FileText, ListChecks, Layers, Users, Gauge, ClipboardList, ExternalLink, Upload, Sparkles,
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchProviders } from '../../store/slices/providersSlice';
import { fetchModels } from '../../store/slices/modelsSlice';
import { fetchBenchmarks } from '../../store/slices/benchmarksSlice';
import { fetchMetrics } from '../../store/slices/metricsSlice';
import { launchEvaluation, setDraft, setDraftType } from '../../store/slices/evaluationsSlice';
import type { CreateEvaluationRequest, EvaluationDraft } from '../../types';
import styles from './NewEvaluation.module.scss';

const STEPS: { key: keyof typeof STEP_ICONS; label: string }[] = [
  { key: 'name', label: 'Name' },
  { key: 'type', label: 'Type' },
  { key: 'providers', label: 'Providers' },
  { key: 'models', label: 'Models' },
  { key: 'suite', label: 'Test Suite' },
  { key: 'metrics', label: 'Metrics' },
  { key: 'review', label: 'Review' },
];

const STEP_ICONS = {
  name: FileText,
  type: Layers,
  providers: Users,
  models: Cpu,
  suite: Database,
  metrics: Gauge,
  review: ClipboardList,
};

const NAME_SUGGESTIONS = ['Q3 Model Selection', 'Weekly Regression Check', 'Agent Framework Bake-off', 'RAG Pipeline Benchmark'];

export default function NewEvaluation() {
  const dispatch = useAppDispatch();
  const navigate = useNavigate();
  const [step, setStep] = useState(0);
  const [toast, setToast] = useState(false);
  const [suiteTab, setSuiteTab] = useState<'benchmarks' | 'upload'>('benchmarks');
  const [suiteCategory, setSuiteCategory] = useState('All');

  const draft = useAppSelector((s) => s.evaluations.draft);
  const launching = useAppSelector((s) => s.evaluations.launching);
  const launchError = useAppSelector((s) => s.evaluations.launchError);

  const providers = useAppSelector((s) => s.providers.items);
  const models = useAppSelector((s) => s.models.items);
  const benchmarks = useAppSelector((s) => s.benchmarks.items);
  const metrics = useAppSelector((s) => s.metrics);

  useEffect(() => {
    dispatch(fetchProviders());
    dispatch(fetchModels());
    dispatch(fetchBenchmarks());
    dispatch(fetchMetrics());
  }, [dispatch]);

  const connectedProviders = providers.filter((p) => p.status === 'connected');
  const availableModels = useMemo(
    () => models.filter((m) => draft.providers.includes(m.provider_id) && m.is_active),
    [models, draft.providers]
  );
  const modelCapabilities = useMemo(
    () => [...new Set(availableModels.flatMap((m) => m.capabilities))],
    [availableModels]
  );
  const [capFilter, setCapFilter] = useState<string[]>([]);
  const filteredModels = capFilter.length
    ? availableModels.filter((m) => capFilter.some((c) => m.capabilities.includes(c)))
    : availableModels;

  const activeMetricsList = draft.type === 'agent' ? metrics.allMetrics : metrics.allMetrics;
  const showCustomAgentMetrics = draft.type === 'agent' && metrics.customAgentMetrics.length > 0;

  const suiteCategories = useMemo(() => ['All', ...new Set(benchmarks.map((b) => b.type))], [benchmarks]);
  const filteredBenchmarks = suiteCategory === 'All' ? benchmarks : benchmarks.filter((b) => b.type === suiteCategory);
  const selectedBenchmark = benchmarks.find((b) => b.name === draft.dataset);

  const toggle = <T,>(list: T[], value: T): T[] => (list.includes(value) ? list.filter((v) => v !== value) : [...list, value]);

  const canGo = () => {
    if (step === 0) return draft.name.trim().length > 0;
    if (step === 1) return draft.type !== null;
    if (step === 2) return draft.providers.length > 0;
    if (step === 3) return draft.models.length > 0;
    if (step === 4) return Boolean(draft.dataset);
    if (step === 5) return draft.metrics.length > 0;
    return true;
  };

  const launch = async () => {
    const benchmark = benchmarks.find((b) => b.name === draft.dataset);
    const judgeModel = draft.judgeModelId ? models.find((m) => m.id === draft.judgeModelId) : undefined;
    const payload: CreateEvaluationRequest = {
      name: draft.name,
      eval_type: draft.type ?? 'model',
      dataset_id: benchmark?.huggingface_dataset || '',
      benchmark: draft.dataset || undefined,
      model_ids: draft.models,
      selected_metrics: draft.metrics,
      run_samples: draft.runSamples,
      selected_category: draft.subgroup.length ? draft.subgroup : undefined,
      // Per spec §1.4: no real API key is collected for the judge model —
      // the judge model's own id is sent in the api_key slot instead.
      ...(judgeModel
        ? { judge_config: { model_id: judgeModel.id, base_url: judgeModel.base_url || '', api_key: judgeModel.id } }
        : {}),
    };

    const result = await dispatch(launchEvaluation(payload));
    if (launchEvaluation.fulfilled.match(result)) {
      setToast(true);
      setTimeout(() => {
        setToast(false);
        navigate('/app/history');
      }, 1600);
    }
  };

  const setD = (partial: Partial<EvaluationDraft>) => dispatch(setDraft(partial));

  return (
    <div className="page-enter pg-shell">
      <div className="pg-hdr"><h1>New Evaluation</h1><p>Set up and launch a structured model evaluation</p></div>
      <div className="pg-body">
        <div className={styles.layout}>
          {/* ---------- Vertical stepper sidebar ---------- */}
          <aside className={styles.stepper}>
            <div className={styles['stepper__progress-label']}>
              Step {step + 1} of {STEPS.length} &middot; {Math.round(((step + 1) / STEPS.length) * 100)}%
            </div>
            <div className={styles['stepper__progress-track']}>
              <div className={styles['stepper__progress-fill']} style={{ width: `${((step + 1) / STEPS.length) * 100}%` }} />
            </div>
            <ol className={styles['stepper__list']}>
              {STEPS.map((s, i) => {
                const Icon = STEP_ICONS[s.key];
                const done = i < step;
                const current = i === step;
                const clickable = i <= step;
                return (
                  <li key={s.key} className={styles['stepper__item']}>
                    <button
                      className={`${styles['stepper__node']} ${done ? styles.done : ''} ${current ? styles.current : ''}`}
                      disabled={!clickable}
                      onClick={() => clickable && setStep(i)}
                    >
                      <span className={styles['stepper__icon']}>{done ? <Check size={14} /> : <Icon size={14} />}</span>
                      <span className={styles['stepper__label']}>{s.label}</span>
                    </button>
                    {i < STEPS.length - 1 && <div className={`${styles['stepper__line']} ${done ? styles.done : ''}`} />}
                  </li>
                );
              })}
            </ol>
          </aside>

          {/* ---------- Step content ---------- */}
          <div className={styles.wiz}>
            <div className={styles['wiz-body']}>
              {step === 0 && (
                <>
                  <h2>Name your evaluation</h2>
                  <p className={styles.sub}>Give it a recognizable name — you can always rename it later.</p>
                  <div className="fg">
                    <label className="fl">Evaluation Name</label>
                    <input className="fi" placeholder="e.g. Q3 Model Selection" value={draft.name} onChange={(e) => setD({ name: e.target.value })} />
                  </div>
                  <div className={styles['name-suggestions']}>
                    {NAME_SUGGESTIONS.map((s) => (
                      <button key={s} className={styles['name-chip']} onClick={() => setD({ name: s })}><Sparkles size={12} /> {s}</button>
                    ))}
                  </div>
                  <div className={styles['name-tips']}>
                    <div className={styles['name-tips__title']}>What's next</div>
                    <ul className={styles['name-tips__list']}>
                      <li>Choose what you're evaluating — a model, an agent, or a RAG pipeline</li>
                      <li>Pick providers and models to compare</li>
                      <li>Select a benchmark and how many questions to sample</li>
                      <li>Choose metrics and an optional judge model</li>
                      <li>Review the cost/time estimate and launch</li>
                    </ul>
                  </div>
                </>
              )}

              {step === 1 && (
                <>
                  <h2>What are you evaluating?</h2>
                  <p className={styles.sub}>This determines which metrics and judge options are available later.</p>
                  <div className="sel-grid">
                    {[
                      { v: 'model' as const, i: <Cpu size={18} />, s: 'General-purpose chat or text model' },
                      { v: 'agent' as const, i: <Bot size={18} />, s: 'Autonomous agent with tool use' },
                      { v: 'rag' as const, i: <Database size={18} />, s: 'Retrieval-augmented generation pipeline' },
                    ].map((o) => (
                      <div key={o.v} className={`sel-opt ${draft.type === o.v ? 'on' : ''}`} onClick={() => dispatch(setDraftType(o.v))}>
                        <div className="sel-chk">{draft.type === o.v && <Check size={13} />}</div>
                        <div><div className="sel-txt" style={{ display: 'flex', alignItems: 'center', gap: 6 }}>{o.i} {o.v === 'model' ? 'General Chat/Text' : o.v === 'agent' ? 'Agent' : 'RAG'}</div><div className="sel-sub">{o.s}</div></div>
                      </div>
                    ))}
                  </div>
                  {draft.type === 'agent' && (
                    <div className="fg" style={{ marginTop: 20 }}>
                      <label className="fl">Agent Framework <span className="opt">(optional)</span></label>
                      <select className="fi" value={draft.agentFramework || ''} onChange={(e) => setD({ agentFramework: e.target.value || null })}>
                        <option value="">None</option>
                        <option value="hermes">Hermes</option>
                        <option value="langgraph">LangGraph</option>
                      </select>
                    </div>
                  )}
                </>
              )}

              {step === 2 && (
                <>
                  <h2>Select providers</h2>
                  <p className={styles.sub}>
                    Choose which connected providers to draw models from.{' '}
                    <a className={styles['inline-link']} onClick={() => navigate('/app/providers')}>Connect more providers <ExternalLink size={12} /></a>
                  </p>
                  <div className="sel-grid">
                    {connectedProviders.map((p) => (
                      <div key={p.id} className={`sel-opt ${draft.providers.includes(p.id) ? 'on' : ''}`} onClick={() => setD({ providers: toggle(draft.providers, p.id) })}>
                        <div className="sel-chk">{draft.providers.includes(p.id) && <Check size={13} />}</div>
                        <div><div className="sel-txt">{p.name}</div><div className="sel-sub">{p.model_count} models available</div></div>
                      </div>
                    ))}
                    {connectedProviders.length === 0 && <div className={styles['wiz-empty']}>No connected providers yet — connect one to continue.</div>}
                  </div>
                </>
              )}

              {step === 3 && (
                <>
                  <h2>Choose models</h2>
                  <p className={styles.sub}>Pick which models to include in this evaluation.</p>
                  <div className={styles['models-layout']}>
                    <div className={styles['models-filters']}>
                      <div className={styles['models-filters__title']}>Capabilities</div>
                      {modelCapabilities.length === 0 && <div className={styles['wiz-empty-sm']}>No models yet</div>}
                      {modelCapabilities.map((c) => (
                        <label key={c} className={styles['models-filters__row']}>
                          <input type="checkbox" checked={capFilter.includes(c)} onChange={() => setCapFilter((prev) => toggle(prev, c))} />
                          {c}
                        </label>
                      ))}
                    </div>
                    <div className={styles['models-grid']}>
                      {filteredModels.map((m) => (
                        <div key={m.id} className={`sel-opt ${draft.models.includes(m.id) ? 'on' : ''}`} onClick={() => setD({ models: toggle(draft.models, m.id) })}>
                          <div className="sel-chk">{draft.models.includes(m.id) && <Check size={13} />}</div>
                          <div><div className="sel-txt">{m.name}</div><div className="sel-sub">{m.context_window.toLocaleString()} ctx &middot; {m.capabilities.join(', ')}</div></div>
                        </div>
                      ))}
                      {filteredModels.length === 0 && <div className={styles['wiz-empty']}>Select providers first, or adjust the capability filter.</div>}
                    </div>
                  </div>
                </>
              )}

              {step === 4 && (
                <>
                  <h2>Test suite</h2>
                  <div className="fg" style={{ maxWidth: 220 }}>
                    <label className="fl">Run Samples <span className="opt">(per model)</span></label>
                    <input
                      className="fi" type="number" min={0} value={draft.runSamples}
                      onChange={(e) => setD({ runSamples: Math.max(0, Number(e.target.value) || 0) })}
                    />
                  </div>
                  <div className={styles['suite-tabs']}>
                    <button className={`${styles['suite-tab']} ${suiteTab === 'benchmarks' ? styles.on : ''}`} onClick={() => setSuiteTab('benchmarks')}><ListChecks size={14} /> Benchmarks</button>
                    <button className={`${styles['suite-tab']} ${suiteTab === 'upload' ? styles.on : ''}`} onClick={() => setSuiteTab('upload')}><Upload size={14} /> Upload</button>
                  </div>

                  {suiteTab === 'upload' ? (
                    <div className={styles['upload-placeholder']}>
                      <Upload size={22} />
                      <div>Custom dataset upload is coming soon — not yet wired to an endpoint.</div>
                    </div>
                  ) : (
                    <>
                      <div className="pills" style={{ marginBottom: 14 }}>
                        {suiteCategories.map((c) => <button key={c} className={`pill ${suiteCategory === c ? 'on' : ''}`} onClick={() => setSuiteCategory(c)}>{c}</button>)}
                      </div>
                      <div className={selectedBenchmark && selectedBenchmark.tasks.length > 0 ? styles['suite-layout--split'] : styles['suite-layout']}>
                        <div className={styles['suite-grid']}>
                          {filteredBenchmarks.map((b) => (
                            <div
                              key={b.name}
                              className={`sel-opt ${draft.dataset === b.name ? 'on' : ''}`}
                              onClick={() => setD({ dataset: b.name, subgroup: [] })}
                              style={{ flexDirection: 'column', alignItems: 'flex-start', gap: 8 }}
                            >
                              <div style={{ display: 'flex', width: '100%', justifyContent: 'space-between', alignItems: 'center' }}>
                                <div className="sel-txt">{b.name}</div><div className="sel-chk">{draft.dataset === b.name && <Check size={13} />}</div>
                              </div>
                              <div className="sel-sub">{b.description}</div>
                              <div><span className="tag tag-amb">{b.type}</span><span style={{ fontSize: 11, color: '#9CA3AF', marginLeft: 8 }}>{b.task_count.toLocaleString()} tasks</span></div>
                            </div>
                          ))}
                        </div>
                        {selectedBenchmark && selectedBenchmark.tasks.length > 0 && (
                          <div className={styles['subgroup-panel']}>
                            <div className={styles['subgroup-panel__hdr']}>
                              Subgroups <span>{draft.subgroup.length} of {selectedBenchmark.tasks.length} selected</span>
                            </div>
                            <div className={styles['subgroup-panel__list']}>
                              {selectedBenchmark.tasks.map((t) => (
                                <label key={t.value} className={styles['subgroup-panel__row']}>
                                  <input type="checkbox" checked={draft.subgroup.includes(t.value)} onChange={() => setD({ subgroup: toggle(draft.subgroup, t.value) })} />
                                  {t.name}
                                </label>
                              ))}
                            </div>
                          </div>
                        )}
                      </div>
                    </>
                  )}
                </>
              )}

              {step === 5 && (
                <>
                  <h2>Configure metrics</h2>
                  <div className={styles['metrics-layout']}>
                    <div className={styles['metrics-col']}>
                      <div className={styles['metrics-col__hdr']}>
                        <span>All Metrics</span>
                        <div style={{ display: 'flex', gap: 6 }}>
                          <button className="btn btn-sm btn-ghost" onClick={() => setD({ metrics: [...activeMetricsList, ...(showCustomAgentMetrics ? metrics.customAgentMetrics : [])] })}>Select all</button>
                          <button className="btn btn-sm btn-ghost" onClick={() => setD({ metrics: [] })}>Unselect all</button>
                        </div>
                      </div>
                      <div className={styles['metrics-col__body']}>
                        <div className="sel-grid" style={{ gridTemplateColumns: 'repeat(auto-fill,minmax(160px,1fr))' }}>
                          {activeMetricsList.map((m) => (
                            <div key={m} className={`sel-opt ${draft.metrics.includes(m) ? 'on' : ''}`} onClick={() => setD({ metrics: toggle(draft.metrics, m) })}>
                              <div className="sel-chk">{draft.metrics.includes(m) && <Check size={13} />}</div><div className="sel-txt">{m}</div>
                            </div>
                          ))}
                        </div>
                        {showCustomAgentMetrics && (
                          <>
                            <div className={styles['metrics-col__subhdr']}>Custom Agent Metrics</div>
                            <div className="sel-grid" style={{ gridTemplateColumns: 'repeat(auto-fill,minmax(160px,1fr))' }}>
                              {metrics.customAgentMetrics.map((m) => (
                                <div key={m} className={`sel-opt ${draft.metrics.includes(m) ? 'on' : ''}`} onClick={() => setD({ metrics: toggle(draft.metrics, m) })}>
                                  <div className="sel-chk">{draft.metrics.includes(m) && <Check size={13} />}</div><div className="sel-txt">{m}</div>
                                </div>
                              ))}
                            </div>
                          </>
                        )}
                      </div>
                    </div>

                    <div className={styles['metrics-col']}>
                      <div className={styles['metrics-col__hdr']}><span>Judge Model</span></div>
                      <p className={styles['judge-note']}>Shows every model in the catalog — judge eligibility is independent of the models selected in Step 4. No API key required.</p>
                      <div className={styles['metrics-col__body']}>
                        <div className={styles['judge-list']}>
                          <label className={styles['judge-row']}>
                            <input type="radio" name="judge" checked={draft.judgeModelId === null} onChange={() => setD({ judgeModelId: null })} />
                            None
                          </label>
                          {models.map((m) => (
                            <label key={m.id} className={styles['judge-row']}>
                              <input type="radio" name="judge" checked={draft.judgeModelId === m.id} onChange={() => setD({ judgeModelId: m.id })} />
                              {m.name}
                            </label>
                          ))}
                        </div>
                      </div>
                    </div>
                  </div>
                </>
              )}

              {step === 6 && (() => {
                const suite = benchmarks.find((b) => b.name === draft.dataset);
                const judgeModel = draft.judgeModelId ? models.find((m) => m.id === draft.judgeModelId) : null;
                const modelChips = draft.models.map((id) => {
                  const m = models.find((mm) => mm.id === id);
                  const p = providers.find((pp) => pp.id === m?.provider_id);
                  return { id, label: m ? `${m.name} (${p?.name || m.provider_id})` : id };
                });
                const totalQuestions = draft.runSamples * Math.max(draft.models.length, 1);
                const estCost = (draft.models.length * draft.runSamples * 0.004).toFixed(2);
                const estMinutes = Math.max(1, Math.round((draft.runSamples * draft.models.length) / 12));
                return (
                  <>
                    <h2>Review & Launch</h2>
                    <p className={styles.sub}>Confirm your evaluation setup.</p>

                    <div className={styles['review-stats']}>
                      {[{ l: 'Est. Cost', v: `$${estCost}` }, { l: 'Est. Time', v: `~${estMinutes} min` }, { l: 'Questions', v: totalQuestions.toLocaleString() }, { l: 'Models', v: String(draft.models.length) }].map((d, i) => (
                        <div key={i} className={styles['review-stat']}><div className={styles['review-stat__label']}>{d.l}</div><div className={styles['review-stat__val']}>{d.v}</div></div>
                      ))}
                    </div>

                    <div className={styles['review-section']}>
                      <div className={styles['review-section__title']}>Overview</div>
                      <div className={styles['review-section__row']}><span>Name</span><strong>{draft.name}</strong></div>
                      <div className={styles['review-section__row']}><span>Type</span><strong>{draft.type}</strong></div>
                      {draft.agentFramework && <div className={styles['review-section__row']}><span>Agent Framework</span><strong>{draft.agentFramework}</strong></div>}
                    </div>

                    <div className={styles['review-section']}>
                      <div className={styles['review-section__title']}>Models</div>
                      <div>{modelChips.map((c) => <span key={c.id} className="tag tag-ind">{c.label}</span>)}</div>
                    </div>

                    <div className={styles['review-section']}>
                      <div className={styles['review-section__title']}>Test Suite</div>
                      <div className={styles['review-section__row']}><span>Name</span><strong>{suite?.name || '—'}</strong></div>
                      <div className={styles['review-section__row']}><span>Source</span><strong>{suite?.huggingface_dataset || '—'}</strong></div>
                      <div className={styles['review-section__row']}><span>Run Samples</span><strong>{draft.runSamples}</strong></div>
                      {draft.subgroup.length > 0 && <div className={styles['review-section__row']}><span>Subgroups</span><strong>{draft.subgroup.length} selected</strong></div>}
                      <div>{(suite?.required_capabilities ?? []).map((c) => <span key={c} className="tag tag-em">{c}</span>)}</div>
                    </div>

                    <div className={styles['review-section']}>
                      <div className={styles['review-section__title']}>Metrics ({draft.metrics.length})</div>
                      <div>{draft.metrics.map((m) => <span key={m} className="tag tag-amb">{m}</span>)}</div>
                    </div>

                    <div className={styles['review-section']}>
                      <div className={styles['review-section__title']}>Judge Model</div>
                      <div className={styles['review-section__row']}><span>Model</span><strong>{judgeModel?.name || 'None'}</strong></div>
                    </div>

                    {launchError && <div className={styles['wiz-error']}>{launchError}</div>}
                  </>
                );
              })()}
            </div>

            <div className={styles['wiz-foot']}>
              <button className="btn btn-ghost" onClick={() => (step > 0 ? setStep(step - 1) : navigate('/app/dashboard'))}>
                <ChevronLeft size={16} /> {step === 0 ? 'Cancel' : 'Back'}
              </button>
              {step < STEPS.length - 1 ? (
                <button className="btn btn-ind" onClick={() => setStep(step + 1)} disabled={!canGo()}>Continue <ChevronRight size={16} /></button>
              ) : (
                <button className={`btn btn-ind ${styles['wiz-launch']}`} onClick={launch} disabled={launching}>
                  <Play size={16} /> {launching ? 'Launching…' : 'Start Evaluation'}
                </button>
              )}
            </div>
          </div>
        </div>
      </div>

      {toast && (
        <div className="toast">
          <div className={styles['toast__icon']}><Check size={18} color="#10B981" /></div>
          <div><div style={{ fontWeight: 700, fontSize: 14 }}>Evaluation launched</div><div style={{ fontSize: 12, color: '#6B7280' }}>Redirecting to History…</div></div>
        </div>
      )}
    </div>
  );
}



















//History.module.scss
@use '../../styles/_variables' as *;

@property --angle {
  syntax: '<angle>';
  initial-value: 0deg;
  inherits: false;
}
@keyframes rotate-angle {
  to { --angle: 360deg; }
}
@keyframes live-dot-pulse {
  0%, 100% { opacity: 1; transform: scale(1); }
  50% { opacity: .5; transform: scale(1.3); }
}

// Fixed-shell override: History's list + detail panels need to scroll
// independently of each other, so pg-body itself must not scroll — this
// makes it a plain flex:1/min-height:0 pass-through instead.
.pg-body-fixed {
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.layout {
  display: grid;
  grid-template-columns: 380px 1fr;
  gap: 20px;
  flex: 1;
  min-height: 0;
}

.sidebar {
  background: $surface; border: 1px solid $border; border-radius: 20px; padding: 18px;
  display: flex; flex-direction: column; min-height: 0;
}
.filters { flex-shrink: 0; }
.empty { padding: 24px; text-align: center; color: $text-secondary; font-size: 13px; display: flex; align-items: center; gap: 8px; justify-content: center; }

.rows { flex: 1; min-height: 0; overflow-y: auto; display: flex; flex-direction: column; gap: 10px; margin-top: 14px; }

.row {
  border: 1px solid $border; border-radius: 14px; padding: 14px; cursor: pointer; transition: all .15s;
  position: relative;
}
.row:hover { border-color: $indigo-light; }
.row.selected { border-color: $indigo; background: $indigo-pale; }

// Running-state animation: a thin light continuously traveling around the
// card border (spec §2.2), built with a rotating conic-gradient angle.
.row--running {
  --angle: 0deg;
  border: 1px solid transparent;
  background:
    linear-gradient(#fff, #fff) padding-box,
    conic-gradient(from var(--angle), $border 0%, $indigo 8%, $border 16%) border-box;
  animation: rotate-angle 2.4s linear infinite;
}
.row--running.selected {
  background:
    linear-gradient($indigo-pale, $indigo-pale) padding-box,
    conic-gradient(from var(--angle), $border 0%, $indigo 8%, $border 16%) border-box;
}

.row__top { display: flex; align-items: center; gap: 10px; margin-bottom: 8px; }
.row__icon { width: 30px; height: 30px; border-radius: 9px; background: $indigo-pale; color: $indigo; display: flex; align-items: center; justify-content: center; flex-shrink: 0; }
.row__name { font-weight: 700; font-size: 14px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.row__badges { display: flex; align-items: center; gap: 6px; margin-bottom: 6px; flex-wrap: wrap; }
.row__meta { font-size: 11px; color: $text-muted; margin-bottom: 8px; }
.row__stats { display: flex; gap: 12px; font-size: 11px; color: $text-secondary; margin-bottom: 10px; flex-wrap: wrap; }
.row__actions { display: flex; gap: 6px; }

.live-dot {
  width: 6px; height: 6px; border-radius: 50%; background: currentColor; display: inline-block;
  animation: live-dot-pulse 1.4s ease-in-out infinite;
}

.detail { background: $surface; border: 1px solid $border; border-radius: 20px; padding: 24px; min-height: 0; overflow-y: auto; }
.detail-empty { padding: 80px 24px; text-align: center; color: $text-secondary; font-size: 14px; }

.detail-hdr { display: flex; justify-content: space-between; align-items: flex-start; margin-bottom: 24px; flex-wrap: wrap; gap: 16px; }
.detail-hdr__badges { display: flex; gap: 8px; margin-bottom: 10px; }
.detail-hdr__name { font-size: 22px; font-weight: 700; letter-spacing: -.3px; }
.detail-hdr__date { font-size: 12px; color: $text-muted; margin-top: 4px; }
.detail-hdr__actions { display: flex; gap: 8px; flex-wrap: wrap; }

.summary-cards { display: grid; grid-template-columns: repeat(3,1fr); gap: 14px; margin-bottom: 24px; }
.summary-card { display: flex; align-items: center; gap: 12px; padding: 16px; background: $surface-alt; border-radius: 14px; }
.summary-card__label { font-size: 11px; color: $text-muted; font-weight: 700; text-transform: uppercase; letter-spacing: .5px; }
.summary-card__val { font-size: 13px; font-weight: 700; margin-top: 2px; }

.status-message { padding: 40px; text-align: center; background: $surface-alt; border-radius: 14px; color: $text-secondary; font-size: 14px; }

@media (max-width: 900px) {
  .layout { grid-template-columns: 1fr; }
  .summary-cards { grid-template-columns: 1fr; }
  // Independent-panel scrolling relies on the grid stretching each panel to
  // a definite row height; once stacked to one column that no longer holds,
  // so fall back to one normal scrolling column instead of risking clipped
  // content under pg-body-fixed's overflow:hidden.
  .pg-body-fixed { overflow-y: auto; }
  .sidebar, .detail { overflow-y: visible; min-height: 0; }
  .rows { overflow-y: visible; }
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
    <div className="page-enter pg-shell">
      <div className="pg-hdr"><h1>History</h1><p>All past and in-progress evaluations</p></div>
      {/* This page needs the list and detail panels to scroll independently
          of each other (spec: point 4 of the fixed-header pattern), so
          pg-body itself doesn't scroll here — .layout fills it instead. */}
      <div className={`pg-body ${styles['pg-body-fixed']}`}>
        <div className={styles.layout}>
          {/* ---------- Sidebar list ---------- */}
          <div className={styles.sidebar}>
            <div className={styles.filters}>
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
              <select className="fi" style={{ marginBottom: 0 }} value={dateFilter} onChange={(e) => setDateFilter(e.target.value)}>
                <option value="all">All time</option>
                <option value="30">Last 30 days</option>
                <option value="7">Last 7 days</option>
              </select>

              {listStatus === 'loading' && list.length === 0 && <div className={styles.empty}>Loading evaluations…</div>}
              {listStatus === 'failed' && list.length === 0 && <div className={styles.empty}>{listError || 'Failed to load evaluations.'}</div>}
              {listStatus !== 'loading' && filtered.length === 0 && list.length > 0 && <div className={styles.empty}>No evaluations match your filters.</div>}
            </div>

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
                    <ListChecks size={16} color="#1428A0" />
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



















//AppShell.module.scss
@use '../../styles/_variables' as *;

.app-shell {
  display: flex;
  height: 100vh;
  background: $bg;

  &__main {
    flex: 1;
    min-height: 0;
    display: flex;
    flex-direction: column;
    overflow: hidden; // scrolling now happens inside each page's .pg-body
    background: $bg;
  }

  &__content {
    flex: 1;
    min-height: 0;
    // Reserves space so the fixed Footer (see Footer.module.scss) never
    // overlaps the bottom of the page content. Each page's .pg-shell
    // reclaims most of this via its height/margin-bottom calc, leaving a
    // small breathing-room gap above the footer.
    padding-bottom: $footer-height;
  }
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
    <div className="page-enter pg-shell">
      <div className="pg-hdr"><h1>Model Catalog</h1><p>All models across connected providers</p></div>
      <div className="pg-toolbar">
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
      </div>
      <div className="pg-body">
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





















//Providers.tsx
import { useEffect, useState } from 'react';
import { Search, Check, Plus, Settings, Unlink, Loader2 } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchProviders, connectProvider, disconnectProvider } from '../../store/slices/providersSlice';
import styles from './Providers.module.scss';

type Filter = 'all' | 'connected' | 'available';

export default function Providers() {
  const dispatch = useAppDispatch();
  const { items, status, mutatingId } = useAppSelector((s) => s.providers);
  const [search, setSearch] = useState('');
  const [filter, setFilter] = useState<Filter>('all');
  const [keyPromptFor, setKeyPromptFor] = useState<string | null>(null);
  const [apiKeyInput, setApiKeyInput] = useState('');

  useEffect(() => {
    dispatch(fetchProviders());
  }, [dispatch]);

  const filtered = items.filter((p) => {
    if (filter === 'connected' && p.status !== 'connected') return false;
    if (filter === 'available' && p.status === 'connected') return false;
    return !search || p.name.toLowerCase().includes(search.toLowerCase());
  });

  const submitConnect = (providerId: string) => {
    if (!apiKeyInput.trim()) return;
    dispatch(connectProvider({ providerId, payload: { api_key: apiKeyInput } }));
    setKeyPromptFor(null);
    setApiKeyInput('');
  };

  return (
    <div className="page-enter pg-shell">
      <div className="pg-hdr"><h1>Providers</h1><p>Manage your AI provider connections</p></div>
      <div className="pg-toolbar">
        <div className="toolbar">
          <div className="search-box">
            <Search size={16} color="#9CA3AF" />
            <input placeholder="Search providers…" value={search} onChange={(e) => setSearch(e.target.value)} />
          </div>
          <div className="pills">
            {(['all', 'connected', 'available'] as Filter[]).map((f) => (
              <button key={f} className={`pill ${filter === f ? 'on' : ''}`} onClick={() => setFilter(f)}>
                {f[0].toUpperCase() + f.slice(1)}
              </button>
            ))}
          </div>
        </div>
      </div>
      <div className="pg-body">
        {status === 'loading' && <div className={styles['providers__loading']}><Loader2 size={18} style={{ animation: 'spin 1.5s linear infinite' }} /> Loading providers…</div>}

        <div className="cards-grid">
          {filtered.map((p) => (
            <div className="card" key={p.id}>
              <div className="card-hdr">
                <div style={{ display: 'flex', alignItems: 'center', gap: 14 }}>
                  <div className={`card-icon ${styles['providers__icon']}`}>{p.logo_url ? <img src={p.logo_url} alt={p.name} /> : p.name[0]}</div>
                  <div>
                    <div className="card-title">{p.name}</div>
                    <div style={{ fontSize: 12, color: '#6B7280', marginTop: 2 }}>{p.model_count} models</div>
                  </div>
                </div>
                <span className={`badge ${p.status === 'connected' ? 'badge-green' : 'badge-gray'}`}>
                  {p.status === 'connected' ? <><Check size={11} /> Connected</> : 'Not connected'}
                </span>
              </div>
              <div className="card-desc">{p.description}</div>

              {keyPromptFor === p.id ? (
                <div className={styles['providers__key-form']}>
                  <input
                    className="fi"
                    type="password"
                    placeholder="Paste API key…"
                    value={apiKeyInput}
                    onChange={(e) => setApiKeyInput(e.target.value)}
                    autoFocus
                  />
                  <div className={styles['providers__key-actions']}>
                    <button className="btn btn-sm btn-ind" onClick={() => submitConnect(p.id)}>Save</button>
                    <button className="btn btn-sm btn-ghost" onClick={() => setKeyPromptFor(null)}>Cancel</button>
                  </div>
                </div>
              ) : (
                <div className="card-foot">
                  <button
                    className={`btn btn-sm ${p.status === 'connected' ? 'btn-ghost' : 'btn-ind'}`}
                    disabled={mutatingId === p.id}
                    onClick={() => setKeyPromptFor(p.id)}
                  >
                    {mutatingId === p.id ? (
                      <Loader2 size={14} style={{ animation: 'spin 1.5s linear infinite' }} />
                    ) : p.status === 'connected' ? (
                      <><Settings size={14} /> Configure</>
                    ) : (
                      <><Plus size={14} /> Connect</>
                    )}
                  </button>
                  {p.status === 'connected' && (
                    <button className="btn btn-sm btn-danger" disabled={mutatingId === p.id} onClick={() => dispatch(disconnectProvider(p.id))}>
                      <Unlink size={14} /> Disconnect
                    </button>
                  )}
                </div>
              )}
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}



















//_variables.scss
// Design tokens — ported 1:1 from the original theme object (T)
$bg: #F7F8FC;
$surface: #FFFFFF;
$surface-alt: #F1F4F9;
$surface-hover: #F8F9FD;

$indigo: #1428A0;
$indigo-light: #4C63C7;
$indigo-dark: #0E1C74;
$violet: #2B45C9;
$indigo-pale: #E9EBF8;

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

$footer-height: 60px;
$sidebar-width: 256px;
// Small gap reclaimed above the fixed footer when a page's scroll shell
// pulls back the workspace content wrapper's bottom padding — see
// .pg-shell in global.scss.
$page-bottom-reclaim: 0.75rem;

$grad-primary: linear-gradient(135deg, #1428A0, #2B45C9);
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
.btn-ind { background: $grad-primary; color: #fff; box-shadow: 0 2px 8px rgba(20, 40, 160, .2); }
.btn-ind:hover { box-shadow: 0 4px 16px rgba(20, 40, 160, .3); transform: translateY(-1px); }
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
.search-box:focus-within { border-color: $indigo; box-shadow: 0 0 0 4px rgba(20, 40, 160, .08); }
.search-box input { border: none; outline: none; font-size: 14px; flex: 1; color: $text-primary; font-family: $font-body; background: transparent; }
.search-box input::placeholder { color: $text-muted; }

.pills { display: flex; gap: 6px; flex-wrap: wrap; }
.pill {
  padding: 7px 16px; border-radius: 100px; font-size: 13px; font-weight: 600;
  border: 1px solid $border; background: $surface; color: $text-secondary; cursor: pointer; transition: all .2s;
}
.pill:hover { border-color: $indigo; color: $indigo; }
.pill.on { background: $grad-primary; color: #fff; border-color: transparent; box-shadow: 0 2px 8px rgba(20, 40, 160, .25); }

.toolbar { display: flex; align-items: center; justify-content: space-between; margin-bottom: 24px; flex-wrap: wrap; gap: 12px; }

.cards-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(330px, 1fr)); gap: 16px; }
.card {
  background: $surface; border: 1px solid $border; border-radius: 16px; padding: 24px;
  transition: all .25s; position: relative; overflow: hidden;
}
.card:hover { border-color: rgba(20, 40, 160, .2); box-shadow: $shadow-3; transform: translateY(-2px); }
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
.fi:focus { outline: none; border-color: $indigo; box-shadow: 0 0 0 4px rgba(20, 40, 160, .08); }
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
.sel-opt.on .sel-chk { background: $grad-primary; border-color: $indigo; color: #fff; box-shadow: 0 2px 4px rgba(20, 40, 160, .25); }
.sel-txt { font-size: 14px; font-weight: 600; }
.sel-sub { font-size: 12px; color: $text-secondary; margin-top: 2px; }

.toast {
  position: fixed; bottom: 32px; right: 32px; background: $surface; border: 1px solid $border;
  border-radius: 16px; padding: 18px 24px; display: flex; align-items: center; gap: 12px;
  box-shadow: $shadow-4; z-index: 999; animation: toastIn .4s ease both;
}

// Shared by Dashboard and Comparison — kept global since both need it.
.radar-wrap { display: flex; justify-content: center; align-items: center; padding: 20px; }

// ─────────────────────────────────────────────────────────────────────────
// Fixed-header page shell. Every /app page's root uses `pg-shell`; the
// header (`pg-hdr`) and, if present, a filters/toolbar row (`pg-toolbar`)
// stay pinned while only `pg-body` scrolls beneath them.
//
// `pg-shell`'s height is derived from the parent flex chain (AppShell's
// __main -> __content, both flex:1/min-height:0 — see AppShell.module.scss)
// rather than grown from content, then extended via calc to reclaim
// __content's $footer-height bottom padding (reserved for the fixed
// Footer), leaving a small $page-bottom-reclaim gap above it.
// ─────────────────────────────────────────────────────────────────────────
.pg-shell {
  display: flex;
  flex-direction: column;
  min-height: 0;
  height: calc(100% + #{$footer-height} - #{$page-bottom-reclaim});
  margin-bottom: calc(-#{$footer-height} + #{$page-bottom-reclaim});
}

.pg-hdr { flex-shrink: 0; padding: 32px 40px 0; }
.pg-hdr h1 { font-size: 28px; font-weight: 700; letter-spacing: -.5px; }
.pg-hdr p { color: $text-secondary; font-size: 14px; margin-top: 4px; }

// Wraps a page's search/filter row (`.toolbar`) so it stays fixed directly
// below the header, above the scrolling body.
.pg-toolbar { flex-shrink: 0; padding: 20px 40px 0; }

// The ONLY element that scrolls. When it directly follows `.pg-toolbar`,
// that block already provides the header-to-content gap (plus `.toolbar`'s
// own margin-bottom), so drop pg-body's own top padding to avoid doubling up.
.pg-body { flex: 1; min-height: 0; overflow-y: auto; padding: 24px 40px 40px; }
.pg-toolbar + .pg-body { padding-top: 0; }

@media (max-width: 768px) {
  .cards-grid { grid-template-columns: 1fr; }
  .pg-hdr, .pg-toolbar, .pg-body { padding-left: 20px; padding-right: 20px; }
}
