// src/api/endpoints/benchmarks.ts
import api from '../axiosInstance';
import type { Benchmark, BenchmarksResponse } from '../../types';

// Spec §5 "Known data-contract gap": the real API doesn't always return
// `tasks` / `required_capabilities` on every benchmark object (sometimes
// omitted or null), even though the documented contract says they're
// required arrays. Rather than defend against this at every read site
// (DatasetStep, Datasets page, etc.), normalize once here so every
// consumer downstream can trust the declared type.
function normalizeBenchmark(b: Benchmark): Benchmark {
  return {
    ...b,
    tasks: b.tasks ?? [],
    required_capabilities: b.required_capabilities ?? [],
  };
}

export const benchmarksApi = {
  list: () =>
    api.get<BenchmarksResponse>('/benchmarks').then((r) => ({
      ...r.data,
      benchmarks: r.data.benchmarks.map(normalizeBenchmark),
    })),
};
















// src/api/endpoints/evaluations.ts
import api from '../axiosInstance';
import type {
  CreateEvaluationRequest,
  CreateEvaluationResponse,
  EvaluationsListResponse,
  EvaluationResultsResponse,
} from '../../types';

export const evaluationsApi = {
  // Populates the History sidebar list. Called on mount and every 10s
  // (silent poll) — see History.tsx.
  list: () => api.get<EvaluationsListResponse>('/evaluations').then((r) => r.data.evaluations),

  create: (payload: CreateEvaluationRequest) =>
    api.post<CreateEvaluationResponse>('/evaluations', payload).then((r) => r.data),

  start: (evaluationId: string) =>
    api.post<void>(`/evaluations/${evaluationId}/start`).then(() => undefined),

  // Only ever called when the selected evaluation's status === 'completed'.
  // The backend returns 400 with { detail: "Execution not completed." } if
  // called too early — callers should surface err.response.data.detail.
  results: (evaluationId: string) =>
    api.get<EvaluationResultsResponse>(`/evaluations/${evaluationId}/results`).then((r) => r.data),

  // Convenience helper used by the wizard's "Start Evaluation" (step 7):
  // create, then immediately start.
  createAndStart: async (payload: CreateEvaluationRequest) => {
    const created = await evaluationsApi.create(payload);
    const id = created.id || created.evaluation_id;
    if (!id) {
      throw new Error('Evaluation was created but no id was returned by the server.');
    }
    await evaluationsApi.start(id);
    return id;
  },
};















// src/components/dashboard/Dashboard.tsx
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
            <div style={{ fontFamily: "'JetBrains Mono', monospace", fontWeight: 700, fontSize: 15, marginBottom: 4 }}>Top Models</div>
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
          <div style={{ fontFamily: "'JetBrains Mono', monospace", fontWeight: 700, fontSize: 15, marginBottom: 20 }}>Quick Actions</div>
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














// src/datasets/Datasets.module.scss
@use '../../styles/_variables' as *;

.header { padding: 32px 40px 0; }
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
















// src/components/datasets/Datasets.tsx
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
                    <div style={{ fontFamily: "'JetBrains Mono',monospace", fontSize: 24, fontWeight: 700 }}>{b.task_count.toLocaleString()}</div>
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



















// src/components/evaluations/NewEvaluation.module.scss
@use '../../styles/_variables' as *;

.layout {
  display: grid;
  grid-template-columns: 240px 1fr;
  gap: 20px;
  align-items: start;
}

// ---------- Vertical stepper sidebar ----------
.stepper {
  background: $surface;
  border: 1px solid $border;
  border-radius: 20px;
  padding: 20px;
  position: sticky;
  top: 24px;

  &__progress-label { font-size: 12px; font-weight: 700; color: $text-secondary; margin-bottom: 8px; }
  &__progress-track { height: 6px; background: $surface-alt; border-radius: 3px; overflow: hidden; margin-bottom: 20px; }
  &__progress-fill { height: 100%; background: $grad-primary; border-radius: 3px; transition: width .4s cubic-bezier(.16,1,.3,1); }
  &__list { list-style: none; }
  &__item { position: relative; }
  &__node {
    display: flex; align-items: center; gap: 10px; width: 100%; padding: 8px 6px;
    background: none; border: none; cursor: pointer; text-align: left; border-radius: 10px; transition: background .15s;
  }
  &__node:disabled { cursor: not-allowed; opacity: .5; }
  &__node:not(:disabled):hover { background: $surface-alt; }
  &__icon {
    width: 26px; height: 26px; border-radius: 50%; flex-shrink: 0;
    display: flex; align-items: center; justify-content: center;
    background: $surface-alt; color: $text-muted; border: 2px solid $border; transition: all .2s;
  }
  &__node.current &__icon { background: $grad-primary; color: #fff; border-color: transparent; box-shadow: 0 0 0 4px rgba(99,102,241,.15); }
  &__node.done &__icon { background: $indigo; color: #fff; border-color: transparent; }
  &__label { font-size: 13px; font-weight: 600; color: $text-secondary; }
  &__node.current &__label { color: $indigo; font-weight: 700; }
  &__node.done &__label { color: $text-primary; }
  &__line { width: 2px; height: 14px; background: $border; margin-left: 19px; border-radius: 1px; }
  &__line.done { background: $indigo; }
}

// ---------- Wizard shell ----------
.wiz {
  background: $surface; border: 1px solid $border; border-radius: 20px; overflow: hidden; box-shadow: $shadow-2;
  display: flex; flex-direction: column; min-height: 560px;
}
.wiz-body { padding: 32px; flex: 1; overflow-y: auto; }
.wiz-body h2 { font-size: 22px; font-weight: 700; margin-bottom: 8px; letter-spacing: -.4px; }
.sub { font-size: 14px; color: $text-secondary; margin-bottom: 22px; line-height: 1.5; }
.wiz-foot { display: flex; justify-content: space-between; padding: 18px 32px; border-top: 1px solid $border; background: $surface-alt; }
.wiz-launch { background: $emerald; box-shadow: 0 4px 14px rgba(16,185,129,.3); }
.wiz-empty { padding: 40px; text-align: center; color: $text-secondary; background: $surface-alt; border-radius: 14px; grid-column: 1 / -1; }
.wiz-empty-sm { padding: 12px; text-align: center; color: $text-muted; font-size: 12px; }
.wiz-error { margin-top: 16px; padding: 12px 16px; background: $red-pale; color: $red; border-radius: 10px; font-size: 13px; font-weight: 600; }
.inline-link { color: $indigo; font-weight: 600; cursor: pointer; display: inline-flex; align-items: center; gap: 4px; }

// ---------- Step 1: Name ----------
.name-suggestions { display: flex; gap: 8px; flex-wrap: wrap; margin: 14px 0 24px; }
.name-chip {
  display: inline-flex; align-items: center; gap: 6px; padding: 6px 12px; border-radius: 100px;
  border: 1px solid $border; background: $surface; color: $text-secondary; font-size: 12px; font-weight: 600; cursor: pointer;
}
.name-chip:hover { border-color: $indigo; color: $indigo; background: $indigo-pale; }
.name-tips { background: $surface-alt; border-radius: 14px; padding: 18px 20px; }
.name-tips__title { font-weight: 700; font-size: 13px; margin-bottom: 10px; }
.name-tips__list { padding-left: 18px; font-size: 13px; color: $text-secondary; line-height: 1.9; }

// ---------- Step 4: Models (independently scrolling filter + grid) ----------
.models-layout { display: grid; grid-template-columns: 200px 1fr; gap: 20px; align-items: start; }
.models-filters {
  border: 1px solid $border; border-radius: 14px; padding: 14px; max-height: 420px; overflow-y: auto; position: sticky; top: 0;
}
.models-filters__title { font-size: 11px; font-weight: 700; text-transform: uppercase; letter-spacing: 1px; color: $text-muted; margin-bottom: 10px; }
.models-filters__row { display: flex; align-items: center; gap: 8px; font-size: 13px; padding: 6px 0; cursor: pointer; }
.models-grid { display: grid; grid-template-columns: repeat(auto-fill,minmax(240px,1fr)); gap: 10px; max-height: 420px; overflow-y: auto; align-content: start; }

// ---------- Step 5: Test Suite ----------
.suite-tabs { display: flex; gap: 8px; margin-bottom: 18px; }
.suite-tab {
  display: inline-flex; align-items: center; gap: 6px; padding: 9px 16px; border-radius: 10px;
  border: 1px solid $border; background: $surface; color: $text-secondary; font-weight: 600; font-size: 13px; cursor: pointer;
}
.suite-tab.on { background: $indigo-pale; border-color: $indigo; color: $indigo; }
.upload-placeholder {
  display: flex; flex-direction: column; align-items: center; gap: 10px; padding: 60px 20px; text-align: center;
  color: $text-secondary; background: $surface-alt; border: 2px dashed $border; border-radius: 16px; font-size: 13px;
}
.suite-layout { display: block; }
.suite-layout--split { display: grid; grid-template-columns: 1fr 400px; gap: 20px; align-items: start; }
.suite-grid {
  display: grid; grid-template-columns: repeat(auto-fill,minmax(260px,1fr)); gap: 10px;
  max-height: 480px; overflow-y: auto; align-content: start;
}
.subgroup-panel {
  width: 400px; border: 1px solid $border; border-radius: 14px; overflow: hidden; position: sticky; top: 0;
  display: flex; flex-direction: column; max-height: 480px;
}
.subgroup-panel__hdr {
  padding: 14px 16px; background: $surface-alt; border-bottom: 1px solid $border; font-weight: 700; font-size: 13px;
  display: flex; justify-content: space-between; align-items: center;
  span { font-weight: 600; font-size: 12px; color: $text-secondary; }
}
.subgroup-panel__list { flex: 1; overflow-y: auto; padding: 8px 4px; }
.subgroup-panel__row { display: flex; align-items: center; gap: 8px; font-size: 13px; padding: 8px 12px; cursor: pointer; }
.subgroup-panel__row:hover { background: $surface-alt; }

// ---------- Step 6: Metrics (two independently-scrolling columns) ----------
.metrics-layout { display: grid; grid-template-columns: 1fr 320px; gap: 20px; align-items: start; }
.metrics-col { border: 1px solid $border; border-radius: 14px; overflow: hidden; display: flex; flex-direction: column; max-height: 480px; }
.metrics-col__hdr {
  padding: 14px 16px; background: $surface-alt; border-bottom: 1px solid $border; font-weight: 700; font-size: 13px;
  display: flex; justify-content: space-between; align-items: center; flex-wrap: wrap; gap: 8px;
}
.metrics-col__subhdr { font-size: 11px; font-weight: 700; text-transform: uppercase; letter-spacing: 1px; color: $text-muted; margin: 16px 0 10px; }
.metrics-col__body { padding: 14px 16px; overflow-y: auto; flex: 1; }
.judge-note { font-size: 12px; color: $text-secondary; padding: 0 16px; margin-top: 10px; line-height: 1.5; }
.judge-list { display: flex; flex-direction: column; gap: 2px; }
.judge-row { display: flex; align-items: center; gap: 8px; font-size: 13px; padding: 8px 10px; border-radius: 8px; cursor: pointer; }
.judge-row:hover { background: $surface-alt; }

// ---------- Step 7: Review ----------
.review-stats { display: grid; grid-template-columns: repeat(4,1fr); gap: 14px; margin-bottom: 24px; }
.review-stat { padding: 18px; background: $indigo-pale; border-radius: 14px; border: 1px solid rgba(99,102,241,.12); }
.review-stat__label { font-size: 11px; color: $text-secondary; font-weight: 700; letter-spacing: 1px; text-transform: uppercase; }
.review-stat__val { font-family: $font-mono; font-size: 22px; font-weight: 700; margin-top: 4px; }
.review-section { padding: 16px 0; border-bottom: 1px solid $border-light; }
.review-section:last-child { border-bottom: none; }
.review-section__title { font-size: 12px; font-weight: 700; text-transform: uppercase; letter-spacing: 1px; color: $text-muted; margin-bottom: 10px; }
.review-section__row { display: flex; justify-content: space-between; font-size: 13px; padding: 5px 0; color: $text-secondary; }
.review-section__row strong { color: $text-primary; font-weight: 600; }

.toast__icon { width: 36px; height: 36px; border-radius: 10px; background: $emerald-pale; display: flex; align-items: center; justify-content: center; }

@media (max-width: 900px) {
  .layout { grid-template-columns: 1fr; }
  .stepper { position: static; }
  .stepper__list { display: flex; overflow-x: auto; gap: 12px; }
  .stepper__line { display: none; }
  .models-layout, .suite-layout--split, .metrics-layout { grid-template-columns: 1fr; }
  .subgroup-panel { width: 100%; }
  .review-stats { grid-template-columns: repeat(2,1fr); }
}




















// src/components/evaluations/NewEvaluation.tsx
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
    <div className="page-enter">
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



















// src/components/history/History.module.scss
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

.layout {
  display: grid;
  grid-template-columns: 380px 1fr;
  gap: 20px;
  align-items: start;
}

.sidebar { background: $surface; border: 1px solid $border; border-radius: 20px; padding: 18px; }
.empty { padding: 24px; text-align: center; color: $text-secondary; font-size: 13px; display: flex; align-items: center; gap: 8px; justify-content: center; }

.rows { display: flex; flex-direction: column; gap: 10px; max-height: 720px; overflow-y: auto; }

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

.detail { background: $surface; border: 1px solid $border; border-radius: 20px; padding: 24px; min-height: 400px; }
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
}





















// src/components/history/History.tsx
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
                                <td style={{ fontFamily: "'JetBrains Mono',monospace", fontWeight: 700 }}>{r.score}%</td>
                                <td style={{ fontFamily: "'JetBrains Mono',monospace" }}>{r.accuracy}%</td>
                                <td style={{ fontFamily: "'JetBrains Mono',monospace", color: '#10B981' }}>{r.passed_tests}</td>
                                <td style={{ fontFamily: "'JetBrains Mono',monospace", color: '#EF4444' }}>{r.failed_tests}</td>
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



















// src/components/layout/Sidebar.tsx
import { NavLink } from 'react-router-dom';
import { Home, Link2, Cpu, BookOpen, Play, FlaskConical, GitCompare, LogOut } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { logout } from '../../store/slices/authSlice';
import styles from './Sidebar.module.scss';

const navItems = [
  { to: '/app/dashboard', icon: <Home size={18} />, label: 'Dashboard' },
  { to: '/app/providers', icon: <Link2 size={18} />, label: 'Providers' },
  { to: '/app/models', icon: <Cpu size={18} />, label: 'Models' },
  { to: '/app/datasets', icon: <BookOpen size={18} />, label: 'Datasets' },
];

const workflowItems = [
  { to: '/app/run-evaluation', icon: <Play size={18} />, label: 'New Evaluation' },
  { to: '/app/history', icon: <FlaskConical size={18} />, label: 'History' },
  { to: '/app/comparison', icon: <GitCompare size={18} />, label: 'Comparison' },
];

export default function Sidebar() {
  const dispatch = useAppDispatch();
  const user = useAppSelector((s) => s.auth.user);

  const navLinkClass = ({ isActive }: { isActive: boolean }) =>
    `${styles['nav-item']} ${isActive ? styles.active : ''}`;

  return (
    <div className={styles.sidebar}>
      <div className={styles['sidebar__logo']}>
        <div className={styles['sidebar__mark']}>&#9670;</div>
        SemcoEval
      </div>
      <nav className={styles['sidebar__nav']}>
        {navItems.map((item) => (
          <NavLink key={item.to} to={item.to} className={navLinkClass}>
            {item.icon}
            {item.label}
          </NavLink>
        ))}
        <div className={styles['sidebar__section']}>Workflow</div>
        {workflowItems.map((item) => (
          <NavLink key={item.to} to={item.to} className={navLinkClass}>
            {item.icon}
            {item.label}
          </NavLink>
        ))}
      </nav>
      <div className={styles['sidebar__foot']}>
        <div className={styles['sidebar__user']}>
          <div className={styles['sidebar__avatar']}>{(user?.username || 'U').slice(0, 2).toUpperCase()}</div>
          <div className={styles['sidebar__user-info']}>
            <div className={styles['sidebar__user-name']}>{user?.profileName || user?.username || 'Guest'}</div>
            <div className={styles['sidebar__user-email']}>{user?.email || ''}</div>
          </div>
          <LogOut size={16} style={{ color: '#9CA3AF', cursor: 'pointer' }} onClick={() => dispatch(logout())} />
        </div>
      </div>
    </div>
  );
}



















// src/mocks/data.ts
// Seed data for the mock API layer — shaped to exactly match the response
// contracts in the API reference doc, so swapping MSW off for a real
// backend later is a no-op for every component/slice/type.
import type {
  Provider, Model, Benchmark, EvaluationListItem, EvaluationResultsResponse, ModelResult,
} from '../types';

export const mockProviders: Provider[] = [
  { id: 'openai', name: 'OpenAI', description: 'GPT family — industry-leading multimodal models', logo_url: null, base_url: 'https://api.openai.com/v1', url_template: null, model_count: 8, status: 'connected' },
  { id: 'anthropic', name: 'Anthropic', description: 'Claude family — safety-focused, high-capability models', logo_url: null, base_url: 'https://api.anthropic.com/v1', url_template: null, model_count: 5, status: 'connected' },
  { id: 'google-deepmind', name: 'Google DeepMind', description: 'Gemini & PaLM — massive context, multimodal', logo_url: null, base_url: 'https://generativelanguage.googleapis.com/v1', url_template: null, model_count: 6, status: 'not_connected' },
  { id: 'meta-ai', name: 'Meta AI', description: 'Llama — open-weight models for self-hosting', logo_url: null, base_url: null, url_template: null, model_count: 4, status: 'not_connected' },
  { id: 'mistral-ai', name: 'Mistral AI', description: 'Mistral & Mixtral — efficient European models', logo_url: null, base_url: 'https://api.mistral.ai/v1', url_template: null, model_count: 3, status: 'connected' },
  { id: 'cohere', name: 'Cohere', description: 'Command & Embed — enterprise RAG specialists', logo_url: null, base_url: 'https://api.cohere.ai/v1', url_template: null, model_count: 3, status: 'not_connected' },
];

export const mockModels: Model[] = [
  { id: 'gpt-4o', name: 'GPT-4o', provider_id: 'openai', category: 'General', capabilities: ['Text', 'Vision', 'Code'], context_window: 128000, input_price: 2.5, output_price: 10, accuracy_score: 92.4, agent_score: 88, is_active: true, base_url: null },
  { id: 'claude-sonnet-4', name: 'Claude Sonnet 4', provider_id: 'anthropic', category: 'General', capabilities: ['Text', 'Vision', 'Code', 'Analysis'], context_window: 200000, input_price: 3, output_price: 15, accuracy_score: 94.1, agent_score: 92, is_active: true, base_url: null },
  { id: 'gemini-2-5-pro', name: 'Gemini 2.5 Pro', provider_id: 'google-deepmind', category: 'General', capabilities: ['Text', 'Vision', 'Code'], context_window: 1000000, input_price: 1.25, output_price: 5, accuracy_score: 91.8, agent_score: 85, is_active: false, base_url: null },
  { id: 'gpt-4o-mini', name: 'GPT-4o mini', provider_id: 'openai', category: 'Efficient', capabilities: ['Text', 'Code'], context_window: 128000, input_price: 0.15, output_price: 0.6, accuracy_score: 87.2, agent_score: 79, is_active: true, base_url: null },
  { id: 'claude-haiku-4', name: 'Claude Haiku 4', provider_id: 'anthropic', category: 'Efficient', capabilities: ['Text', 'Code'], context_window: 200000, input_price: 0.25, output_price: 1.25, accuracy_score: 86.9, agent_score: 77, is_active: true, base_url: null },
  { id: 'mistral-large', name: 'Mistral Large', provider_id: 'mistral-ai', category: 'General', capabilities: ['Text', 'Code', 'Analysis'], context_window: 128000, input_price: 2, output_price: 6, accuracy_score: 89.7, agent_score: 81, is_active: true, base_url: null },
  { id: 'llama-3-1-405b', name: 'Llama 3.1 405B', provider_id: 'meta-ai', category: 'Open Weight', capabilities: ['Text', 'Code'], context_window: 128000, input_price: 0, output_price: 0, accuracy_score: 88.5, agent_score: 80, is_active: false, base_url: null },
  { id: 'command-r-plus', name: 'Command R+', provider_id: 'cohere', category: 'RAG', capabilities: ['Text', 'RAG', 'Search'], context_window: 128000, input_price: 3, output_price: 15, accuracy_score: 85.3, agent_score: 75, is_active: false, base_url: null },
];

// Deliberately includes one benchmark missing `tasks` / `required_capabilities`
// (MATH-500 below) to exercise the §5 normalization fallback in benchmarksApi.list().
export const mockBenchmarks: Benchmark[] = [
  { name: 'MMLU Pro', description: 'Massive multitask language understanding with harder questions', tasks: [{ name: 'STEM', value: 'stem' }, { name: 'Humanities', value: 'humanities' }, { name: 'Social Science', value: 'social-science' }], task_count: 12032, required_capabilities: ['Text', 'Reasoning'], huggingface_dataset: 'TIGER-Lab/MMLU-Pro', type: 'Knowledge' },
  { name: 'HumanEval+', description: 'Code generation with extended test cases and edge coverage', tasks: [{ name: 'Sample task 1', value: 'humaneval-plus-1' }], task_count: 820, required_capabilities: ['Code'], huggingface_dataset: 'evalplus/humanevalplus', type: 'Code' },
  { name: 'GPQA Diamond', description: 'Graduate-level science questions verified by domain experts', tasks: [{ name: 'Physics', value: 'physics' }, { name: 'Chemistry', value: 'chemistry' }, { name: 'Biology', value: 'biology' }], task_count: 448, required_capabilities: ['Reasoning', 'Analysis'], huggingface_dataset: 'Idavidrein/gpqa', type: 'Science' },
  { name: 'MT-Bench', description: 'Multi-turn conversational evaluation across 8 domains', tasks: [{ name: 'Sample task 1', value: 'mt-bench-1' }], task_count: 160, required_capabilities: ['Text', 'Reasoning'], huggingface_dataset: 'lmsys/mt_bench_human_judgments', type: 'Conversation' },
  { name: 'BigCodeBench', description: 'Complex programming tasks requiring deep library knowledge', tasks: [{ name: 'Sample task 1', value: 'bigcodebench-1' }], task_count: 1140, required_capabilities: ['Code', 'Analysis'], huggingface_dataset: 'bigcode/bigcodebench', type: 'Code' },
  // Intentionally omits `tasks` / `required_capabilities`
  { name: 'MATH-500', description: 'Competition-level mathematics problems with step verification', task_count: 500, huggingface_dataset: 'HuggingFaceH4/MATH-500', type: 'Math' } as Benchmark,
];

export const mockAllMetrics = ['Accuracy', 'Speed', 'Cost', 'Consistency', 'Coherence', 'Factuality', 'Creativity', 'Safety'];
export const mockCustomAgentMetrics = ['Tool Call Accuracy', 'Task Completion', 'Step Efficiency'];

// ─────────────────────────────────────────────────────────────────────────
// In-memory evaluation store used only by the mock handlers, so
// POST /evaluations -> POST /evaluations/{id}/start -> GET /evaluations
// (list, polled every 10s) -> GET /evaluations/{id}/results behaves like a
// real, stateful backend across calls.
// ─────────────────────────────────────────────────────────────────────────
interface MockEval extends Omit<EvaluationListItem, 'status' | 'progress' | 'top_model' | 'top_score' | 'started_at' | 'completed_at'> {
  startedAtMs: number | null;
  durationMs: number;
  // 'created' means POST /evaluations happened but not yet /start.
  lifecycle: 'created' | 'running' | 'completed' | 'failed';
}

export const mockEvals = new Map<string, MockEval>();
let mockEvalCounter = 1;

function pick<T>(arr: T[], seed: number): T {
  return arr[seed % arr.length];
}

function buildResultsFor(evalRecord: MockEval): ModelResult[] {
  return evalRecord.model_ids.map((modelId, i) => {
    const model = mockModels.find((m) => m.id === modelId);
    const total = evalRecord.total_questions;
    const baseAccuracy = (model?.accuracy_score ?? 85) / 100;
    // Deterministic per-model jitter so results are stable across refetches.
    const jitter = ((i * 37) % 11) / 100;
    const accuracy = Math.min(0.99, Math.max(0.4, baseAccuracy - jitter));
    const passed = Math.round(total * accuracy);
    return {
      model_id: modelId,
      provider: model?.provider_id ?? 'unknown',
      rank: i + 1,
      score: Math.round(accuracy * 1000) / 10,
      accuracy: Math.round(accuracy * 1000) / 10,
      passed_tests: passed,
      failed_tests: total - passed,
      total_tests: total,
      metric_scores: Object.fromEntries(
        evalRecord.selected_metrics.map((m, mi) => [m, Math.round((accuracy - mi * 0.02) * 1000) / 10])
      ),
      details: [],
    };
  }).sort((a, b) => b.score - a.score).map((r, i) => ({ ...r, rank: i + 1 }));
}

export function advanceMockEval(evalRecord: MockEval): EvaluationListItem {
  if (evalRecord.lifecycle === 'created' || evalRecord.startedAtMs === null) {
    return {
      ...evalRecord,
      status: 'pending',
      progress: 0,
      top_model: null,
      top_score: null,
      started_at: null,
      completed_at: null,
    };
  }

  if (evalRecord.lifecycle === 'failed') {
    return {
      ...evalRecord,
      status: 'failed',
      progress: 0,
      top_model: null,
      top_score: null,
      started_at: new Date(evalRecord.startedAtMs).toISOString(),
      completed_at: new Date(evalRecord.startedAtMs).toISOString(),
    };
  }

  const elapsed = Date.now() - evalRecord.startedAtMs;
  const pct = Math.min(1, elapsed / evalRecord.durationMs);
  const progress = pct; // fraction, 0..1

  if (pct >= 1) {
    if (evalRecord.lifecycle !== 'completed') evalRecord.lifecycle = 'completed';
    const results = buildResultsFor(evalRecord);
    const top = results[0];
    const topModel = mockModels.find((m) => m.id === top?.model_id);
    return {
      ...evalRecord,
      status: 'completed',
      progress: 1,
      top_model: topModel?.name ?? null,
      top_score: top?.score ?? null,
      started_at: new Date(evalRecord.startedAtMs).toISOString(),
      completed_at: new Date(evalRecord.startedAtMs + evalRecord.durationMs).toISOString(),
    };
  }

  return {
    ...evalRecord,
    status: 'running',
    progress,
    top_model: null,
    top_score: null,
    started_at: new Date(evalRecord.startedAtMs).toISOString(),
    completed_at: null,
  };
}

export function getMockResults(evalId: string): EvaluationResultsResponse | { error: string } {
  const record = mockEvals.get(evalId);
  if (!record) return { error: 'Evaluation not found' };
  const snapshot = advanceMockEval(record);
  if (snapshot.status !== 'completed') return { error: 'Execution not completed.' };

  const results = buildResultsFor(record);
  const top = results[0];
  return {
    evaluation_id: evalId,
    status: 'completed',
    top_model: top?.model_id ?? '',
    top_score: top?.score ?? 0,
    results,
  };
}

export function createMockEval(input: {
  name: string;
  description?: string;
  eval_type: string;
  dataset_id: string;
  benchmark?: string;
  model_ids: string[];
  selected_metrics: string[];
  run_samples: number;
  selected_category?: string[];
}): string {
  const id = `eval_${mockEvalCounter++}`;
  mockEvals.set(id, {
    id,
    name: input.name,
    description: input.description ?? '',
    eval_type: input.eval_type,
    dataset_id: input.dataset_id,
    datasets_config: [{ dataset_id: input.dataset_id }],
    benchmark: input.benchmark ?? '',
    model_ids: input.model_ids,
    selected_metrics: input.selected_metrics,
    run_samples: input.run_samples,
    selected_category: input.selected_category ?? [],
    total_questions: Math.max(input.run_samples, 1) * Math.max(input.model_ids.length, 1),
    created_at: new Date().toISOString(),
    startedAtMs: null,
    // ~10% chance of a simulated failure, ~ (samples*models)-scaled duration otherwise.
    durationMs: 8000 + pick([0, 4000, 8000], mockEvalCounter),
    lifecycle: 'created',
  });
  return id;
}

export function startMockEval(evalId: string): void {
  const record = mockEvals.get(evalId);
  if (!record) return;
  record.startedAtMs = Date.now();
  record.lifecycle = mockEvalCounter % 9 === 0 ? 'failed' : 'running';
}

// ---- Seed a small history so the History page has content on first load ----
function seedHistoricalEval(opts: {
  name: string;
  eval_type: string;
  benchmark: string;
  model_ids: string[];
  selected_metrics: string[];
  run_samples: number;
  lifecycle: MockEval['lifecycle'];
  startedMinutesAgo: number | null;
  durationMs: number;
}): void {
  const id = `eval_${mockEvalCounter++}`;
  mockEvals.set(id, {
    id,
    name: opts.name,
    description: '',
    eval_type: opts.eval_type,
    dataset_id: '',
    datasets_config: [],
    benchmark: opts.benchmark,
    model_ids: opts.model_ids,
    selected_metrics: opts.selected_metrics,
    run_samples: opts.run_samples,
    selected_category: [],
    total_questions: opts.run_samples * opts.model_ids.length,
    created_at: new Date(Date.now() - (opts.startedMinutesAgo ?? 0) * 60000 - 120000).toISOString(),
    startedAtMs: opts.startedMinutesAgo === null ? null : Date.now() - opts.startedMinutesAgo * 60000,
    durationMs: opts.durationMs,
    lifecycle: opts.lifecycle,
  });
}

seedHistoricalEval({
  name: 'Q3 Support Bot Test',
  eval_type: 'model',
  benchmark: 'MMLU Pro',
  model_ids: ['gpt-4o', 'claude-sonnet-4'],
  selected_metrics: ['Accuracy', 'Consistency'],
  run_samples: 25,
  lifecycle: 'completed',
  startedMinutesAgo: 240,
  durationMs: 5000,
});

seedHistoricalEval({
  name: 'Agent Framework Comparison',
  eval_type: 'agent',
  benchmark: 'MT-Bench',
  model_ids: ['claude-sonnet-4', 'mistral-large'],
  selected_metrics: ['Accuracy', 'Tool Call Accuracy'],
  run_samples: 15,
  lifecycle: 'completed',
  startedMinutesAgo: 1440,
  durationMs: 5000,
});

seedHistoricalEval({
  name: 'Code Generation Sprint',
  eval_type: 'model',
  benchmark: 'HumanEval+',
  model_ids: ['gpt-4o', 'claude-sonnet-4', 'mistral-large'],
  selected_metrics: ['Accuracy', 'Speed'],
  run_samples: 20,
  lifecycle: 'running',
  startedMinutesAgo: 0,
  durationMs: 45000,
});

seedHistoricalEval({
  name: 'RAG Pipeline Benchmark',
  eval_type: 'rag',
  benchmark: 'GPQA Diamond',
  model_ids: ['command-r-plus', 'gpt-4o'],
  selected_metrics: ['Accuracy', 'Factuality'],
  run_samples: 10,
  lifecycle: 'failed',
  startedMinutesAgo: 60,
  durationMs: 5000,
});

















// src/mocks/handlers.ts
import { http, HttpResponse, delay } from 'msw';
import {
  mockProviders, mockModels, mockBenchmarks, mockAllMetrics, mockCustomAgentMetrics,
  mockEvals, advanceMockEval, getMockResults, createMockEval, startMockEval,
} from './data';
import type {
  SsoLoginRequest, SsoLoginResponse,
  ConnectProviderRequest, ConnectProviderResponse, DisconnectProviderResponse,
  CustomModelRequest,
  BenchmarksResponse, MetricsResponse,
  CreateEvaluationRequest, CreateEvaluationResponse,
  EvaluationsListResponse,
} from '../types';

const API_BASE = import.meta.env.VITE_API_BASE_URL || '/api';
const url = (path: string) => `${API_BASE}${path}`;

let providers = mockProviders.map((p) => ({ ...p }));
let models = mockModels.map((m) => ({ ...m }));

export const handlers = [
  // ---------- Auth ----------
  http.post(url('/sso_login'), async ({ request }) => {
    await delay(400);
    const body = (await request.json()) as SsoLoginRequest;
    const response: SsoLoginResponse = {
      status: 'ok',
      message: 'Signed in',
      result: {
        token: `mock-token-${body.token}-${Date.now()}`,
        username: 'jane.doe',
        email: 'jane@semco.ai',
        language: 'en',
        profile_name: 'Jane Doe',
      },
    };
    return HttpResponse.json(response);
  }),

  // ---------- Providers ----------
  http.get(url('/providers'), async () => {
    await delay(300);
    return HttpResponse.json({ providers });
  }),

  http.post(url('/providers/:providerId/connect'), async ({ params, request }) => {
    await delay(500);
    const { providerId } = params as { providerId: string };
    const body = (await request.json()) as ConnectProviderRequest;
    if (!body.api_key?.trim()) {
      return HttpResponse.json({ message: 'api_key is required' }, { status: 400 });
    }
    providers = providers.map((p) => (p.id === providerId ? { ...p, status: 'connected' } : p));
    const response: ConnectProviderResponse = {
      status: 'connected',
      provider_id: providerId,
      models_synced: providers.find((p) => p.id === providerId)?.model_count ?? 0,
    };
    return HttpResponse.json(response);
  }),

  http.delete(url('/providers/:providerId/disconnect'), async ({ params }) => {
    await delay(300);
    const { providerId } = params as { providerId: string };
    providers = providers.map((p) => (p.id === providerId ? { ...p, status: 'not_connected' } : p));
    const response: DisconnectProviderResponse = { status: 'disconnected', provider_id: providerId };
    return HttpResponse.json(response);
  }),

  // ---------- Models ----------
  http.get(url('/models'), async () => {
    await delay(300);
    return HttpResponse.json({ models });
  }),

  http.post(url('/models/custom'), async ({ request }) => {
    await delay(500);
    const body = (await request.json()) as CustomModelRequest;
    models = [
      ...models,
      {
        id: body.model_id || `custom-${Date.now()}`,
        name: body.name,
        provider_id: 'custom',
        category: body.category,
        capabilities: ['Text'],
        context_window: body.context_window,
        input_price: null,
        output_price: null,
        accuracy_score: null,
        agent_score: null,
        is_active: true,
        base_url: body.base_url,
      },
    ];
    return new HttpResponse(null, { status: 200 });
  }),

  // ---------- Benchmarks ----------
  http.get(url('/benchmarks'), async () => {
    await delay(300);
    const response: BenchmarksResponse = { benchmarks: mockBenchmarks, total: mockBenchmarks.length };
    return HttpResponse.json(response);
  }),

  // ---------- Metrics ----------
  http.get(url('/metrics'), async () => {
    await delay(200);
    const response: MetricsResponse = { all_metrics: mockAllMetrics, custom_agent_metrics: mockCustomAgentMetrics };
    return HttpResponse.json(response);
  }),

  // ---------- Evaluations ----------
  http.get(url('/evaluations'), async () => {
    await delay(250);
    const evaluations = Array.from(mockEvals.values()).map(advanceMockEval);
    // Newest first, matching how the History page expects the list ordered.
    evaluations.sort((a, b) => (a.created_at < b.created_at ? 1 : -1));
    const response: EvaluationsListResponse = { evaluations };
    return HttpResponse.json(response);
  }),

  http.post(url('/evaluations'), async ({ request }) => {
    await delay(500);
    const body = (await request.json()) as CreateEvaluationRequest;
    const benchmark = mockBenchmarks.find((b) => b.name === body.benchmark);
    const id = createMockEval({
      name: body.name,
      description: body.description,
      eval_type: body.eval_type,
      dataset_id: body.dataset_id,
      benchmark: body.benchmark,
      model_ids: body.model_ids,
      selected_metrics: body.selected_metrics,
      run_samples: body.run_samples,
      selected_category: body.selected_category,
    });
    void benchmark; // task_count no longer drives total_questions — run_samples does (spec §1.2 Step 5)
    const response: CreateEvaluationResponse = { id, evaluation_id: id };
    return HttpResponse.json(response);
  }),

  http.post(url('/evaluations/:evaluationId/start'), async ({ params }) => {
    await delay(300);
    const { evaluationId } = params as { evaluationId: string };
    startMockEval(evaluationId);
    return new HttpResponse(null, { status: 200 });
  }),

  http.get(url('/evaluations/:evaluationId/results'), async ({ params }) => {
    await delay(300);
    const { evaluationId } = params as { evaluationId: string };
    const result = getMockResults(evaluationId);
    if ('error' in result) {
      return HttpResponse.json({ detail: result.error }, { status: 400 });
    }
    return HttpResponse.json(result);
  }),
];

























// src/routes/AppRoutes.tsx
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
      <Route path="/" element={<Landing />} />
      {/* AuthGuard redirects here with { from, errorMessage } on error/logged_out. */}
      <Route path="/sso-login" element={<SsoLogin />} />

      {/* AuthGuard triggers the SSO WebSocket handshake (useSsoAuth) as soon as
          this layout route mounts, and shows AuthSpinner until authenticated. */}
      <Route element={<AuthGuard />}>
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




















// src/store/slices/evaluationsSlice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import type { PayloadAction } from '@reduxjs/toolkit';
import { evaluationsApi } from '../../api/endpoints/evaluations';
import type {
  CreateEvaluationRequest,
  EvaluationDraft,
  EvaluationListItem,
  EvaluationResultsResponse,
} from '../../types';

type AsyncStatus = 'idle' | 'loading' | 'succeeded' | 'failed';

interface EvaluationsState {
  draft: EvaluationDraft;

  // History list (GET /evaluations) — silently re-fetched every 10s from
  // History.tsx. `listStatus` only gates the *initial* loading/error UI;
  // components should check `list.length === 0` alongside it so a failed
  // background poll never shows a spinner/error over existing data (spec §2.4).
  list: EvaluationListItem[];
  listStatus: AsyncStatus;
  listError: string | null;

  // Per-evaluation results (GET /evaluations/{id}/results), fetched lazily
  // and only once status === 'completed' (spec §2.3).
  resultsByEvalId: Record<string, EvaluationResultsResponse>;
  resultsStatusByEvalId: Record<string, AsyncStatus>;
  resultsErrorByEvalId: Record<string, string | null>;

  launching: boolean;
  launchError: string | null;
}

const initialDraft: EvaluationDraft = {
  name: '',
  type: null,
  providers: [],
  models: [],
  dataset: null,
  subgroup: [],
  runSamples: 10,
  metrics: [],
  judgeModelId: null,
  agentFramework: null,
};

const initialState: EvaluationsState = {
  draft: initialDraft,
  list: [],
  listStatus: 'idle',
  listError: null,
  resultsByEvalId: {},
  resultsStatusByEvalId: {},
  resultsErrorByEvalId: {},
  launching: false,
  launchError: null,
};

export const fetchEvaluations = createAsyncThunk('evaluations/fetchList', () => evaluationsApi.list());

export const fetchEvaluationResults = createAsyncThunk(
  'evaluations/fetchResults',
  async (evaluationId: string, { rejectWithValue }) => {
    try {
      const data = await evaluationsApi.results(evaluationId);
      return { evaluationId, data };
    } catch (err) {
      const detail =
        (err as { response?: { data?: { detail?: string } } })?.response?.data?.detail ||
        (err as Error)?.message ||
        'Failed to load results';
      return rejectWithValue({ evaluationId, message: detail });
    }
  }
);

export const launchEvaluation = createAsyncThunk(
  'evaluations/launch',
  (payload: CreateEvaluationRequest) => evaluationsApi.createAndStart(payload)
);

const evaluationsSlice = createSlice({
  name: 'evaluations',
  initialState,
  reducers: {
    setDraft(state, action: PayloadAction<Partial<EvaluationDraft>>) {
      state.draft = { ...state.draft, ...action.payload };
    },
    // Step 2: changing type clears any previously selected metrics (spec §1.2).
    setDraftType(state, action: PayloadAction<EvaluationDraft['type']>) {
      state.draft.type = action.payload;
      state.draft.metrics = [];
      if (action.payload !== 'agent') {
        state.draft.agentFramework = null;
      }
    },
    resetDraft(state) {
      state.draft = initialDraft;
    },
    // Local-only removal — no DELETE /evaluations/{id} endpoint exists yet
    // (spec §4.6). Does not persist; a background poll will bring it back
    // if the backend still has it.
    removeEvaluationLocal(state, action: PayloadAction<string>) {
      state.list = state.list.filter((e) => e.id !== action.payload);
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchEvaluations.pending, (state) => {
        if (state.list.length === 0) state.listStatus = 'loading';
      })
      .addCase(fetchEvaluations.fulfilled, (state, action) => {
        state.listStatus = 'succeeded';
        state.listError = null;
        state.list = action.payload;
      })
      .addCase(fetchEvaluations.rejected, (state, action) => {
        // Background polls fail silently (spec §2.4) — only surface the
        // error state when we have nothing on screen yet.
        if (state.list.length === 0) {
          state.listStatus = 'failed';
          state.listError = action.error.message || 'Failed to load evaluations';
        }
      })
      .addCase(fetchEvaluationResults.pending, (state, action) => {
        state.resultsStatusByEvalId[action.meta.arg] = 'loading';
        state.resultsErrorByEvalId[action.meta.arg] = null;
      })
      .addCase(fetchEvaluationResults.fulfilled, (state, action) => {
        const { evaluationId, data } = action.payload;
        state.resultsStatusByEvalId[evaluationId] = 'succeeded';
        state.resultsByEvalId[evaluationId] = data;
      })
      .addCase(fetchEvaluationResults.rejected, (state, action) => {
        const payload = action.payload as { evaluationId: string; message: string } | undefined;
        const id = payload?.evaluationId ?? action.meta.arg;
        state.resultsStatusByEvalId[id] = 'failed';
        state.resultsErrorByEvalId[id] = payload?.message || 'Failed to load results';
      })
      .addCase(launchEvaluation.pending, (state) => {
        state.launching = true;
        state.launchError = null;
      })
      .addCase(launchEvaluation.fulfilled, (state) => {
        state.launching = false;
        state.draft = initialDraft;
      })
      .addCase(launchEvaluation.rejected, (state, action) => {
        state.launching = false;
        state.launchError = action.error.message || 'Failed to launch evaluation';
      });
  },
});

export const { setDraft, setDraftType, resetDraft, removeEvaluationLocal } = evaluationsSlice.actions;
export default evaluationsSlice.reducer;


















// src/types/index.ts
// ---------- Auth ----------
export interface SsoLoginRequest {
  token: string;
  data: string;
}
export interface SsoLoginResult {
  token: string;
  username: string;
  email: string;
  language: string;
  profile_name: string;
}
export interface SsoLoginResponse {
  status: string;
  message: string;
  result: SsoLoginResult;
}

// ---------- Providers ----------
export interface Provider {
  id: string;
  name: string;
  description: string;
  logo_url: string | null;
  base_url: string | null;
  url_template: string | null;
  model_count: number;
  status: 'connected' | 'not_connected' | string;
}
export interface ConnectProviderRequest {
  api_key: string;
}
export interface ConnectProviderResponse {
  status: 'connected';
  provider_id: string;
  models_synced: number;
}
export interface DisconnectProviderResponse {
  status: 'disconnected';
  provider_id: string;
}

// ---------- Models ----------
export interface Model {
  id: string;
  name: string;
  provider_id: string;
  category: string;
  capabilities: string[];
  context_window: number;
  input_price: number | null;
  output_price: number | null;
  accuracy_score: number | null;
  agent_score: number | null;
  is_active: boolean;
  base_url: string | null;
}
export interface CustomModelRequest {
  base_url: string;
  category: string;
  api_key: string;
  model_id: string;
  name: string;
  context_window: number;
  description: string;
}

// ---------- Benchmarks ----------
export interface BenchmarkTask {
  name: string;
  value: string;
}
export interface Benchmark {
  name: string;
  description: string;
  // ⚠️ Not always present on the real API response — normalized to [] at the
  // fetch boundary (benchmarksApi.list), so consumers can trust these are
  // always arrays. See spec §5 "Known data-contract gap".
  tasks: BenchmarkTask[];
  task_count: number;
  required_capabilities: string[];
  huggingface_dataset: string;
  type: string;
}
export interface BenchmarksResponse {
  benchmarks: Benchmark[];
  total: number;
}

// ---------- Metrics ----------
export interface MetricsResponse {
  all_metrics: string[];
  custom_agent_metrics: string[];
}

// ---------- Evaluations: create/start ----------
export interface JudgeConfig {
  model_id: string;
  base_url: string;
  // NOTE: populated with the judge model's own id, not a real credential —
  // the Judge API Key field was removed from the UI entirely (spec §1.4).
  api_key: string;
}
export interface CreateEvaluationRequest {
  name: string;
  description?: string;
  eval_type: 'model' | 'agent' | 'rag' | string;
  dataset_id: string;
  benchmark?: string;
  model_ids: string[];
  metrics_config?: Record<string, unknown>;
  selected_metrics: string[];
  dataset_limit?: number;
  run_samples: number;
  selected_category?: string[];
  judge_config?: JudgeConfig;
}
export interface CreateEvaluationResponse {
  id?: string;
  evaluation_id?: string;
  [key: string]: unknown;
}

// ---------- Evaluations: list (History) ----------
export type EvaluationStatusValue = 'pending' | 'running' | 'completed' | 'failed' | 'canceled';

export interface EvaluationListItem {
  id: string;
  name: string;
  description: string;
  eval_type: string;
  dataset_id: string;
  datasets_config: { dataset_id: string }[];
  benchmark: string;
  model_ids: string[];
  selected_metrics: string[];
  run_samples: number;
  selected_category: string[];
  status: EvaluationStatusValue;
  progress: number;
  total_questions: number;
  top_model: string | null;
  top_score: number | null;
  created_at: string;
  started_at: string | null;
  completed_at: string | null;
}
export interface EvaluationsListResponse {
  evaluations: EvaluationListItem[];
}

// ---------- Evaluations: results ----------
export interface TestDetail {
  test_id: string;
  input: string;
  output: string;
  expected: string;
  latency_seconds: number;
  passed: boolean;
  score: number;
  metric_scores: Record<string, number>;
}
export interface ModelResult {
  model_id: string;
  provider: string;
  rank: number;
  score: number;
  accuracy: number;
  passed_tests: number;
  failed_tests: number;
  total_tests: number;
  metric_scores: Record<string, number>;
  details: TestDetail[];
}
export interface EvaluationResultsResponse {
  evaluation_id: string;
  status: EvaluationStatusValue;
  top_model: string;
  top_score: number;
  results: ModelResult[];
}

// UI-only draft built up across the wizard's 7 steps (spec §6).
export interface EvaluationDraft {
  name: string;
  type: 'model' | 'agent' | 'rag' | null;
  providers: string[];
  models: string[];
  dataset: string | null;
  subgroup: string[];
  runSamples: number; // default 10
  metrics: string[];
  judgeModelId: string | null;
  // judgeApiKey intentionally omitted — no longer collected (spec §1.4)
  agentFramework: string | null;
}
