//History.tsx
import { useEffect, useMemo, useState } from 'react';
import { useNavigate, useSearchParams } from 'react-router-dom';
import {
  Search, Copy, Trash2, FileBarChart, Cpu, Bot, Database, Loader2,
  Award, ListChecks, Clock, History as HistoryIcon,
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
      <div className={styles['history__header']}>
        <div>
          <p className={styles['history__header-eyebrow']}>Activity</p>
          <h1>History</h1>
          <p className={styles['history__header-sub']}>All past and in-progress evaluations</p>
        </div>
        <div className={styles['history__header-meta']}>
          <HistoryIcon size={13} />
          {list.length} evaluation{list.length === 1 ? '' : 's'} tracked
        </div>
      </div>
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

              {listStatus === 'loading' && list.length === 0 && <div className={styles.empty}><Loader2 size={16} style={{ animation: 'spin 1.5s linear infinite' }} /> Loading evaluations…</div>}
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


















//History.module.scss
@use '../../styles/_variables' as *;

.history {
  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding: 24px 32px 18px;
    margin-bottom: 24px;
    border-bottom: 1px solid $border-light;

    h1 {
      font-family: $font-display;
      font-size: 1.5rem;
      font-weight: 800;
      letter-spacing: -0.02em;
      color: $text-primary;
      line-height: 1.2;
    }
  }

  &__header-eyebrow {
    display: flex;
    align-items: center;
    gap: 8px;
    font-family: $font-mono;
    font-size: 0.6875rem;
    font-weight: 700;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    color: $indigo;
    margin-bottom: 6px;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $indigo;
    }
  }

  &__header-sub {
    margin-top: 4px;
    font-size: 0.875rem;
    color: $text-secondary;
  }

  &__header-meta {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-secondary;
    background: $surface-alt;
    border: 1px solid $border-light;
    border-radius: 999px;
    padding: 7px 13px;
    white-space: nowrap;
    margin-bottom: 3px;
  }
}

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
















//Comparison.tsx
import { useEffect, useMemo, useState } from 'react';
import { Layers } from 'lucide-react';
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
    <div className="page-enter pg-shell">
      <div className={styles['comparison__header']}>
        <div>
          <p className={styles['comparison__header-eyebrow']}>Analysis</p>
          <h1>Model Comparison</h1>
          <p className={styles['comparison__header-sub']}>Side-by-side performance across a shared benchmark</p>
        </div>
        <div className={styles['comparison__header-meta']}>
          <Layers size={13} />
          {comp.length} model{comp.length === 1 ? '' : 's'} compared
        </div>
      </div>
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
                    {comp.map((m, i) => <td key={i} style={{ fontFamily: "'JetBrains Mono',monospace", fontWeight: 700 }}>{m[met.k]}</td>)}
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

















//Comparison.module.scss
@use '../../styles/_variables' as *;

.comparison {
  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding: 24px 32px 18px;
    margin-bottom: 24px;
    border-bottom: 1px solid $border-light;

    h1 {
      font-family: $font-display;
      font-size: 1.5rem;
      font-weight: 800;
      letter-spacing: -0.02em;
      color: $text-primary;
      line-height: 1.2;
    }
  }

  &__header-eyebrow {
    display: flex;
    align-items: center;
    gap: 8px;
    font-family: $font-mono;
    font-size: 0.6875rem;
    font-weight: 700;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    color: $indigo;
    margin-bottom: 6px;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $indigo;
    }
  }

  &__header-sub {
    margin-top: 4px;
    font-size: 0.875rem;
    color: $text-secondary;
  }

  &__header-meta {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-secondary;
    background: $surface-alt;
    border: 1px solid $border-light;
    border-radius: 999px;
    padding: 7px 13px;
    white-space: nowrap;
    margin-bottom: 3px;
  }

  &__controls { display: flex; gap: 14px; margin-bottom: 24px; align-items: center; flex-wrap: wrap; }
  &__label { font-size: 13px; font-weight: 700; color: $text-secondary; }
  &__select { width: auto; padding: 8px 14px; cursor: pointer; border-radius: 10px; }
  &__grid { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; margin-bottom: 24px; }
  &__panel-title { font-family: $font-display; font-weight: 700; font-size: 15px; }
  &__panel-sub { font-size: 13px; color: $text-secondary; margin-bottom: 8px; }
  &__legend { display: flex; justify-content: center; gap: 20px; margin-top: 4px; flex-wrap: wrap; }
  &__legend span { display: flex; align-items: center; gap: 6px; font-size: 12px; font-weight: 600; color: $text-secondary; }
  &__dot { width: 8px; height: 8px; border-radius: 50%; display: inline-block; }
  &__scores { display: grid; grid-template-columns: repeat(3, 1fr); gap: 32px; }
  &__score-item { display: flex; flex-direction: column; align-items: center; gap: 12px; }
}

.model-chip {
  display: inline-flex; align-items: center; gap: 6px; padding: 7px 16px; border-radius: 100px;
  font-size: 13px; font-weight: 700; border: 2px solid; cursor: pointer; transition: all .2s;
}
.model-chip:hover { transform: translateY(-1px); }
.model-chip__dot { width: 8px; height: 8px; border-radius: 50%; }

@media (max-width: 768px) {
  .comparison__grid { grid-template-columns: 1fr; }
  .comparison__scores { grid-template-columns: 1fr; }
}
