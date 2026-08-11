import { useEffect, useMemo, useState } from 'react';
import { useNavigate, useSearchParams } from 'react-router-dom';
import {
  Search, Copy, Trash2, FileBarChart, BrainCircuit, Bot, Layers, Loader2,
  Award, ListChecks, Clock, History as HistoryIcon,
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import {
  fetchEvaluations, fetchEvaluationResults, removeEvaluationLocal, setDraft,
} from '../../store/slices/evaluationsSlice';
import type { EvaluationListItem, EvaluationStatusValue } from '../../types';
import { SkeletonListRows } from '../common/Skeleton';
import styles from './History.module.scss';

const TYPE_ICON: Record<string, typeof BrainCircuit> = { model: BrainCircuit, agent: Bot, rag: Layers };
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
        <div className="split-shell split-shell--fill">
          {/* ---------- Sidebar list ---------- */}
          <div className={`split-shell__sidebar ${styles.sidebar}`}>
            <div className={styles.filters}>
              <div className="search-box" style={{ minWidth: 0, marginBottom: 10 }}>
                <Search size={16} color="var(--text-muted)" />
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

              {listStatus === 'failed' && list.length === 0 && <div className={styles.empty}>{listError || 'Failed to load evaluations.'}</div>}
              {listStatus !== 'loading' && filtered.length === 0 && list.length > 0 && <div className={styles.empty}>No evaluations match your filters.</div>}
            </div>

            <div className={styles.rows}>
              {listStatus === 'loading' && list.length === 0 && <SkeletonListRows count={5} />}
              {filtered.map((e) => {
                const Icon = TYPE_ICON[e.eval_type] || BrainCircuit;
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
                      <div className={styles.row__actions} onClick={(ev) => ev.stopPropagation()}>
                        <button className="btn btn-sm btn-ghost" title="Duplicate" aria-label="Duplicate" onClick={() => duplicate(e)}><Copy size={12} /></button>
                        <button className="btn btn-sm btn-danger" title="Delete" aria-label="Delete" onClick={() => dispatch(removeEvaluationLocal(e.id))}><Trash2 size={12} /></button>
                      </div>
                    </div>
                    <div className={styles.row__meta}>{new Date(e.created_at).toLocaleDateString()}</div>
                    <div className={styles.row__stats}>
                      <span>{e.top_model ? `🏆 ${e.top_model}` : '—'}</span>
                      <span>{e.top_score != null ? `${e.top_score}%` : '—'}</span>
                      <span>{e.model_ids.length} models</span>
                    </div>
                  </div>
                );
              })}
            </div>
          </div>

          {/* ---------- Detail panel ---------- */}
          <div className={`split-shell__main ${styles.detail}`}>
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
                                <td style={{ color: 'var(--text-secondary)' }}>{providerName(r.model_id)}</td>
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
