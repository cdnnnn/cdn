import { useEffect, useMemo, useRef, useState } from 'react';
import { useSearchParams, useNavigate } from 'react-router-dom';
import {
  Search, Sparkles, Bot, Layers, Loader2, Download, ListTree, Gauge, CheckCircle2, XCircle,
  Award, ListChecks, Clock, History as HistoryIcon, SlidersHorizontal, CalendarDays, X, PlaySquare,
  StopCircle, AlertTriangle,
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import {
  fetchEvaluations, fetchEvaluationResults, cancelEvaluation,
} from '../../store/slices/evaluationsSlice';
import { downloadReport } from '../../store/slices/reportsSlice';
import type { ReportDownloadFormat } from '../../api/endpoints/reports';
import type { EvaluationListItem, EvaluationStatusValue, ModelResult } from '../../types';
import { SkeletonListRows } from '../common/Skeleton';
import styles from './History.module.scss';

const TYPE_ICON: Record<string, typeof Sparkles> = { model: Sparkles, agent: Bot, rag: Layers };
const TYPE_LABEL: Record<string, string> = { model: 'AI Model', agent: 'Agent', rag: 'RAG' };

// Mirrors Reports.tsx — same four formats, same download endpoint.
const DOWNLOAD_OPTIONS: { format: ReportDownloadFormat; label: string }[] = [
  { format: 'json', label: 'JSON' },
  { format: 'csv', label: 'CSV' },
  { format: 'csv_detailed', label: 'CSV (Detailed)' },
  { format: 'pdf', label: 'PDF' },
];

// Maps a raw status to the module status-pill variant suffix.
function statusVariant(status: EvaluationStatusValue): string {
  switch (status) {
    case 'completed': return 'completed';
    case 'running': return 'running';
    case 'pending': return 'pending';
    case 'failed': return 'failed';
    case 'canceled': return 'canceled';
    default: return 'pending';
  }
}

function withinDateRange(iso: string | null | undefined, range: string): boolean {
  if (range === 'all') return true;
  if (!iso) return false;
  const time = new Date(iso).getTime();
  if (Number.isNaN(time)) return false;
  const days = range === '7' ? 7 : 30;
  const cutoff = Date.now() - days * 86400000;
  return time >= cutoff;
}

// Formats an ISO date string for display, falling back to an em dash when
// the value is missing or unparseable instead of showing "Invalid Date".
function formatDate(iso: string | null | undefined, withTime = false): string {
  if (!iso) return '—';
  const d = new Date(iso);
  if (Number.isNaN(d.getTime())) return '—';
  return withTime ? d.toLocaleString() : d.toLocaleDateString();
}

export default function History() {
  const dispatch = useAppDispatch();
  const navigate = useNavigate();
  const [searchParams, setSearchParams] = useSearchParams();
  const selectedId = searchParams.get('id');

  const { list: rawList, listStatus, listError, resultsByEvalId, resultsStatusByEvalId, resultsErrorByEvalId } = useAppSelector((s) => s.evaluations);
  const list = rawList || [];
  const models = useAppSelector((s) => s.models.items) || [];
  const providers = useAppSelector((s) => s.providers.items) || [];
  const downloadingId = useAppSelector((s) => s.reports.downloadingId);
  const cancelingId = useAppSelector((s) => s.evaluations.cancelingId);

  const [search, setSearch] = useState('');
  const [typeFilter, setTypeFilter] = useState('All');
  const [dateFilter, setDateFilter] = useState('all');
  const [statusFilter, setStatusFilter] = useState('All');
  const [activeFilter, setActiveFilter] = useState<'search' | 'type' | 'date' | 'status' | null>(null);
  const [detailsModel, setDetailsModel] = useState<ModelResult | null>(null);
  const [drawerView, setDrawerView] = useState<'tests' | 'metrics'>('tests');
  // Row awaiting "are you sure?" confirmation before we call the cancel API.
  const [cancelTarget, setCancelTarget] = useState<EvaluationListItem | null>(null);
  const searchInputRef = useRef<HTMLInputElement>(null);

  const toggleFilter = (key: 'search' | 'type' | 'date' | 'status') => {
    setActiveFilter((prev) => (prev === key ? null : key));
  };

  useEffect(() => {
    if (activeFilter === 'search') searchInputRef.current?.focus();
  }, [activeFilter]);

  const DATE_LABEL: Record<string, string> = { all: 'All time', '30': 'Last 30 days', '7': 'Last 7 days' };
  const STATUS_LABEL: Record<string, string> = {
    All: 'All',
    completed: 'Completed',
    running: 'Running',
    pending: 'Pending',
    failed: 'Failed',
  };

  // Initial load + silent 10s poll (spec §2.4) — no spinner/error disruption
  // on background refreshes; the slice only flips listStatus when list is empty.
  useEffect(() => {
    dispatch(fetchEvaluations());
    const interval = setInterval(() => dispatch(fetchEvaluations()), 10000);
    return () => clearInterval(interval);
  }, [dispatch]);

  const filtered = useMemo(() => {
    return list.filter((e) => {
      if (search && !(e.name || '').toLowerCase().includes(search.toLowerCase())) return false;
      if (typeFilter !== 'All' && e.eval_type !== typeFilter) return false;
      if (statusFilter !== 'All' && e.status !== statusFilter) return false;
      if (!withinDateRange(e.created_at, dateFilter)) return false;
      return true;
    });
  }, [list, search, typeFilter, dateFilter, statusFilter]);

  const selected = list.find((e) => e.id === selectedId) || filtered[0] || null;

  // Handles the initial/default selection (mount, or URL navigation to an
  // id we haven't fetched yet). Explicit row clicks trigger their own fetch
  // in selectRow below regardless of cache, so this only needs to cover the
  // "selection changed without a click" case — hence the cache guard stays
  // here, but not in selectRow.
  useEffect(() => {
    if (selected && selected.status === 'completed' && !resultsByEvalId[selected.id]) {
      dispatch(fetchEvaluationResults(selected.id));
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [selected?.id, selected?.status]);

  // Re-fetches on every click, including re-clicking the already-selected
  // row — results can change server-side (e.g. a report finishing after the
  // eval completed), so cached data shouldn't block a manual refresh.
  const selectRow = (id: string) => {
    const clicked = list.find((e) => e.id === id);
    // Running evaluations aren't selectable — there's nothing to show yet
    // (no results, no report), and the "Stop evaluation" button is the only
    // interactive element on that card.
    if (clicked && clicked.status === 'running') return;
    setSearchParams({ id });
    setDetailsModel(null);
    if (clicked && clicked.status === 'completed') {
      dispatch(fetchEvaluationResults(id));
    }
  };

  // Confirm-and-stop flow for a running evaluation's "Stop" button.
  const requestCancel = (e: React.MouseEvent, evaluation: EvaluationListItem) => {
    e.stopPropagation(); // don't let this bubble into selectRow
    setCancelTarget(evaluation);
  };
  const confirmCancel = () => {
    if (!cancelTarget) return;
    dispatch(cancelEvaluation(cancelTarget.id));
    setCancelTarget(null);
  };

  // Opens the details drawer for a model in a given view — reused by both
  // the "test details" and "metric scores" icon buttons in the results table.
  const openDrawer = (model: ModelResult, view: 'tests' | 'metrics') => {
    setDetailsModel(model);
    setDrawerView(view);
  };

  // Close the details drawer with Escape.
  useEffect(() => {
    if (!detailsModel) return;
    const onKey = (e: KeyboardEvent) => {
      if (e.key === 'Escape') setDetailsModel(null);
    };
    window.addEventListener('keydown', onKey);
    return () => window.removeEventListener('keydown', onKey);
  }, [detailsModel]);

  // Close the stop-evaluation confirm dialog with Escape.
  useEffect(() => {
    if (!cancelTarget) return;
    const onKey = (e: KeyboardEvent) => {
      if (e.key === 'Escape') setCancelTarget(null);
    };
    window.addEventListener('keydown', onKey);
    return () => window.removeEventListener('keydown', onKey);
  }, [cancelTarget]);

  const modelName = (id: string) => models.find((m) => m.id === id)?.name || id;
  const providerName = (id: string) => {
    const model = models.find((m) => m.id === id);
    return providers.find((p) => p.id === model?.provider_id)?.name || model?.provider_id || '—';
  };

  const results = selected ? resultsByEvalId[selected.id] : undefined;
  const resultsStatus = selected ? resultsStatusByEvalId[selected.id] : undefined;
  const resultsError = selected ? resultsErrorByEvalId[selected.id] : undefined;

  const StatusBadge = ({ status }: { status: EvaluationStatusValue }) => (
    <span className={`${styles.status} ${styles[`status--${statusVariant(status)}`]}`}>
      {status === 'running' && <span className={styles['live-dot']} />}
      {status}
    </span>
  );

  return (
    <div className={`page-enter pg-shell ${styles.history}`}>
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

      {/* List + detail panels scroll independently, so pg-body itself doesn't
          scroll here — .shell fills it instead. */}
      <div className={`pg-body ${styles['pg-body-fixed']}`}>
        <div className={styles.shell}>
          {/* ---------- Sidebar list ---------- */}
          <div className={styles.sidebar}>
            <div className={styles.filters}>
              <div className={styles['filter-toolbar']}>
                <span className={styles['filter-toolbar__label']}>Filters</span>
                <div className={styles['filter-toolbar__divider']} />
                <button
                  type="button"
                  className={`${styles['filter-toolbar__btn']} ${activeFilter === 'search' ? styles.on : ''}`}
                  onClick={() => toggleFilter('search')}
                  title="Search"
                >
                  <Search size={15} />
                  {search && <span className={styles['filter-toolbar__dot']} />}
                </button>
                <button
                  type="button"
                  className={`${styles['filter-toolbar__btn']} ${activeFilter === 'type' ? styles.on : ''}`}
                  onClick={() => toggleFilter('type')}
                  title="Filter by type"
                >
                  <SlidersHorizontal size={15} />
                  {typeFilter !== 'All' && <span className={styles['filter-toolbar__dot']} />}
                </button>
                <button
                  type="button"
                  className={`${styles['filter-toolbar__btn']} ${activeFilter === 'date' ? styles.on : ''}`}
                  onClick={() => toggleFilter('date')}
                  title="Filter by date"
                >
                  <CalendarDays size={15} />
                  {dateFilter !== 'all' && <span className={styles['filter-toolbar__dot']} />}
                </button>
                <button
                  type="button"
                  className={`${styles['filter-toolbar__btn']} ${activeFilter === 'status' ? styles.on : ''}`}
                  onClick={() => toggleFilter('status')}
                  title="Filter by status"
                >
                  <ListChecks size={15} />
                  {statusFilter !== 'All' && <span className={styles['filter-toolbar__dot']} />}
                </button>

                <div className={styles['filter-toolbar__summary']}>
                  {search && (
                    <span className={styles['filter-chip']}>
                      <span>“{search}”</span>
                      <X size={11} onClick={() => setSearch('')} />
                    </span>
                  )}
                  {typeFilter !== 'All' && (
                    <span className={styles['filter-chip']}>
                      <span>{TYPE_LABEL[typeFilter]}</span>
                      <X size={11} onClick={() => setTypeFilter('All')} />
                    </span>
                  )}
                  {dateFilter !== 'all' && (
                    <span className={styles['filter-chip']}>
                      <span>{DATE_LABEL[dateFilter]}</span>
                      <X size={11} onClick={() => setDateFilter('all')} />
                    </span>
                  )}
                  {statusFilter !== 'All' && (
                    <span className={styles['filter-chip']}>
                      <span>{STATUS_LABEL[statusFilter]}</span>
                      <X size={11} onClick={() => setStatusFilter('All')} />
                    </span>
                  )}
                </div>
              </div>

              <div className={`${styles['filter-panel']} ${activeFilter ? styles['filter-panel--open'] : ''}`}>
                {activeFilter === 'search' && (
                  <div>
                    <div className={styles['panel-search']}>
                      <Search size={16} />
                      <input
                        ref={searchInputRef}
                        placeholder="Search evaluations…"
                        value={search}
                        onChange={(e) => setSearch(e.target.value)}
                      />
                    </div>
                  </div>
                )}
                {activeFilter === 'type' && (
                  <div>
                    <div className={styles['panel-pills']}>
                      {['All', 'model', 'agent', 'rag'].map((t) => (
                        <button
                          key={t}
                          className={`${styles['panel-pill']} ${typeFilter === t ? styles.on : ''}`}
                          onClick={() => {
                            setTypeFilter(t);
                            setActiveFilter(null);
                          }}
                        >
                          {t === 'All' ? 'All' : TYPE_LABEL[t]}
                        </button>
                      ))}
                    </div>
                  </div>
                )}
                {activeFilter === 'date' && (
                  <div>
                    <div className={styles['panel-pills']}>
                      {Object.entries(DATE_LABEL).map(([value, label]) => (
                        <button
                          key={value}
                          className={`${styles['panel-pill']} ${dateFilter === value ? styles.on : ''}`}
                          onClick={() => {
                            setDateFilter(value);
                            setActiveFilter(null);
                          }}
                        >
                          {label}
                        </button>
                      ))}
                    </div>
                  </div>
                )}
                {activeFilter === 'status' && (
                  <div>
                    <div className={styles['panel-pills']}>
                      {Object.entries(STATUS_LABEL).map(([value, label]) => (
                        <button
                          key={value}
                          className={`${styles['panel-pill']} ${statusFilter === value ? styles.on : ''}`}
                          onClick={() => {
                            setStatusFilter(value);
                            setActiveFilter(null);
                          }}
                        >
                          {label}
                        </button>
                      ))}
                    </div>
                  </div>
                )}
              </div>

              {listStatus === 'failed' && list.length === 0 && (
                <div className={styles.empty}>{listError || 'Failed to load evaluations.'}</div>
              )}
              {listStatus !== 'loading' && list.length > 0 && filtered.length === 0 && (
                <div className={styles.empty}>No evaluations match your filters.</div>
              )}
            </div>

            <div className={styles.rows}>
              {listStatus === 'loading' && list.length === 0 && <SkeletonListRows count={5} />}

              {/* True empty state — the account has no evaluations at all yet,
                  as opposed to "no evaluations match your current filters"
                  (handled above via styles.empty). */}
              {listStatus === 'succeeded' && list.length === 0 && (
                <div className={styles['sidebar-empty']}>
                  <div className={styles['sidebar-empty__icon']}>
                    <HistoryIcon size={22} />
                  </div>
                  <h3 className={styles['sidebar-empty__title']}>No evaluations yet</h3>
                  <p className={styles['sidebar-empty__sub']}>
                    Once you launch an evaluation, it'll show up here so you can track progress and review results.
                  </p>
                  <button type="button" className={styles['sidebar-empty__cta']} onClick={() => navigate('/evaluations/new')}>
                    <PlaySquare size={14} />
                    Start your first evaluation
                  </button>
                </div>
              )}

              {filtered.map((e) => {
                const Icon = TYPE_ICON[e.eval_type] || Sparkles;
                const isSelected = selected?.id === e.id;
                const isRunning = e.status === 'running';
                return (
                  <div
                    key={e.id}
                    className={`${styles.row} ${isSelected ? styles.selected : ''} ${isRunning ? `${styles['row--running']} ${styles['row--unselectable']}` : ''}`}
                    onClick={() => selectRow(e.id)}
                  >
                    <div className={styles.row__top}>
                      <div className={styles.row__icon}>
                        <Icon size={16} />
                      </div>
                      <div className={styles.row__name}>{e.name}</div>
                    </div>
                    <div className={styles.row__badges}>
                      <span className={styles['type-tag']}>{TYPE_LABEL[e.eval_type] || e.eval_type}</span>
                      <StatusBadge status={e.status} />
                      {isRunning && (
                        <button
                          type="button"
                          className={styles['row__stop-btn']}
                          onClick={(evt) => requestCancel(evt, e)}
                          disabled={cancelingId === e.id}
                          title="Stop this evaluation"
                        >
                          {cancelingId === e.id ? <Loader2 size={11} className={styles.spin} /> : <StopCircle size={11} />}
                          {cancelingId === e.id ? 'Stopping…' : 'Stop'}
                        </button>
                      )}
                    </div>
                    <div className={styles.row__meta}>{formatDate(e.created_at)}</div>
                    <div className={styles.row__stats}>
                      <span>{e.top_model ? `🏆 ${e.top_model}` : '—'}</span>
                      <span>{e.top_score != null ? `${e.top_score}%` : '—'}</span>
                      <span>{(e.model_ids || []).length} models</span>
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
                      <span className={styles['type-tag']}>{TYPE_LABEL[selected.eval_type] || selected.eval_type}</span>
                      <StatusBadge status={selected.status} />
                    </div>
                    <h2 className={styles['detail-hdr__name']}>{selected.name || 'Untitled evaluation'}</h2>
                    <div className={styles['detail-hdr__date']}>Created {formatDate(selected.created_at, true)}</div>
                  </div>
                  {/* Only offered once the backend has generated a report row
                      for this evaluation (selected.report.report_id present) —
                      same options/behavior as the Reports page. */}
                  {selected.report?.report_id && (
                    <div className={styles['detail-hdr__actions']}>
                      {DOWNLOAD_OPTIONS.map((opt) => (
                        <button
                          key={opt.format}
                          className={styles['dl-btn']}
                          disabled={downloadingId === selected.report!.report_id}
                          onClick={() =>
                            dispatch(
                              downloadReport({
                                reportId: selected.report!.report_id,
                                format: opt.format,
                                filenameHint: selected.report!.title || selected.name,
                              })
                            )
                          }
                        >
                          {downloadingId === selected.report.report_id ? (
                            <Loader2 size={12} className={styles.spin} />
                          ) : (
                            <Download size={12} />
                          )}
                          {opt.label}
                        </button>
                      ))}
                    </div>
                  )}
                </div>

                <div className={styles['summary-cards']}>
                  <div className={styles['summary-card']}>
                    <span className={`${styles['summary-card__icon']} ${styles['summary-card__icon--win']}`}>
                      <Award size={16} />
                    </span>
                    <div>
                      <div className={styles['summary-card__label']}>Winner</div>
                      <div className={styles['summary-card__val']}>
                        {selected.top_model || '—'}
                        {selected.top_score != null ? ` · ${selected.top_score}%` : ''}
                      </div>
                    </div>
                  </div>
                  <div className={styles['summary-card']}>
                    <span className={`${styles['summary-card__icon']} ${styles['summary-card__icon--info']}`}>
                      <ListChecks size={16} />
                    </span>
                    <div>
                      <div className={styles['summary-card__label']}>Questions / Models</div>
                      <div className={styles['summary-card__val']}>
                        {(selected.total_questions ?? 0).toLocaleString()} &middot; {(selected.model_ids || []).length} models
                      </div>
                    </div>
                  </div>
                  <div className={styles['summary-card']}>
                    <span className={`${styles['summary-card__icon']} ${styles['summary-card__icon--status']}`}>
                      <Clock size={16} />
                    </span>
                    <div>
                      <div className={styles['summary-card__label']}>Status</div>
                      <div className={styles['summary-card__val']}>
                        {selected.status || '—'}
                        {selected.completed_at ? ` · ${formatDate(selected.completed_at)}` : ''}
                      </div>
                    </div>
                  </div>
                </div>

                {selected.status === 'completed' ? (
                  <>
                    {resultsStatus === 'loading' && !results && (
                      <div className={styles.empty}>
                        <Loader2 size={16} className={styles.spin} /> Loading results…
                      </div>
                    )}
                    {resultsStatus === 'failed' && !results && <div className={styles.empty}>{resultsError}</div>}
                    {results && (
                      <>
                        {/* Extra fields from GET /evaluations/{id}/results — benchmark,
                            dataset, metrics tested, and when the run actually started. */}
                        <div className={styles['meta-strip']}>
                          {results.benchmark && (
                            <div className={styles['meta-strip__item']}>
                              <span className={styles['meta-strip__label']}>Benchmark</span>
                              <span className={styles['meta-strip__val']}>{results.benchmark}</span>
                            </div>
                          )}
                          {results.dataset_id && (
                            <div className={styles['meta-strip__item']}>
                              <span className={styles['meta-strip__label']}>Dataset</span>
                              <span className={styles['meta-strip__val']}>{results.dataset_id}</span>
                            </div>
                          )}
                          {results.started_at && (
                            <div className={styles['meta-strip__item']}>
                              <span className={styles['meta-strip__label']}>Started</span>
                              <span className={styles['meta-strip__val']}>{formatDate(results.started_at, true)}</span>
                            </div>
                          )}
                          {(results.selected_metrics || []).length > 0 && (
                            <div className={styles['meta-strip__item']}>
                              <span className={styles['meta-strip__label']}>Metrics tested</span>
                              <span className={styles['meta-strip__chips']}>
                                {(results.selected_metrics || []).map((m) => (
                                  <span key={m} className={styles['type-tag']}>{m}</span>
                                ))}
                              </span>
                            </div>
                          )}
                        </div>

                        <div className={styles.results}>
                          <table className={styles['results-table']}>
                            <thead>
                              <tr>
                                <th>Rank</th>
                                <th>Model</th>
                                <th>Provider</th>
                                <th>Score</th>
                                <th>Accuracy</th>
                                <th>Passed</th>
                                <th>Failed</th>
                                <th />
                                <th />
                              </tr>
                            </thead>
                            <tbody>
                              {(results.results || []).map((r) => (
                                <tr key={r.model_id} className={r.rank === 1 ? styles.winner : ''}>
                                  <td className={styles['cell-rank']}>
                                    {r.rank === 1 ? '🏆 ' : ''}
                                    {r.rank ?? '—'}
                                  </td>
                                  <td className={styles['cell-model']}>{modelName(r.model_id)}</td>
                                  <td className={styles['cell-provider']}>{r.provider || providerName(r.model_id)}</td>
                                  <td className={styles['cell-num']}>{r.score ?? 0}%</td>
                                  <td className={`${styles['cell-num']} ${styles['cell-num--muted']}`}>{r.accuracy ?? 0}%</td>
                                  <td className={styles['cell-pass']}>{r.passed_tests ?? 0}</td>
                                  <td className={styles['cell-fail']}>{r.failed_tests ?? 0}</td>
                                  <td className={styles['cell-details']}>
                                    {(r.details?.length ?? 0) > 0 && (
                                      <button
                                        type="button"
                                        className={styles['details-btn']}
                                        title="View test-by-test details"
                                        onClick={() => openDrawer(r, 'tests')}
                                      >
                                        <ListTree size={14} />
                                      </button>
                                    )}
                                  </td>
                                  <td className={styles['cell-details']}>
                                    {Object.keys(r.metric_scores || {}).length > 0 && (
                                      <button
                                        type="button"
                                        className={styles['details-btn']}
                                        title="View metric scores"
                                        onClick={() => openDrawer(r, 'metrics')}
                                      >
                                        <Gauge size={14} />
                                      </button>
                                    )}
                                  </td>
                                </tr>
                              ))}
                            </tbody>
                          </table>
                        </div>
                      </>
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

      {/* ---------- Test-detail / metric-score slide-over ---------- */}
      <div className={`${styles['drawer-overlay']} ${detailsModel ? styles['drawer-overlay--open'] : ''}`} onClick={() => setDetailsModel(null)} />
      <div className={`${styles.drawer} ${detailsModel ? styles['drawer--open'] : ''}`} role="dialog" aria-hidden={!detailsModel}>
        {detailsModel && (
          <>
            <div className={styles['drawer__header']}>
              <div>
                <div className={styles['drawer__eyebrow']}>
                  {drawerView === 'tests' ? 'Test-by-test details' : 'Metric scores'}
                </div>
                <h3 className={styles['drawer__title']}>{modelName(detailsModel.model_id)}</h3>
                <div className={styles['drawer__sub']}>
                  {(detailsModel.provider || providerName(detailsModel.model_id))} · {detailsModel.score ?? 0}% score · {detailsModel.accuracy ?? 0}% accuracy
                </div>
              </div>
              <button type="button" className={styles['drawer__close']} onClick={() => setDetailsModel(null)} title="Close">
                <X size={16} />
              </button>
            </div>

            {/* Tabs let you flip between the two views without closing the drawer. */}
            <div className={styles['drawer__tabs']}>
              <button
                type="button"
                className={`${styles['drawer__tab']} ${drawerView === 'tests' ? styles.on : ''}`}
                onClick={() => setDrawerView('tests')}
                disabled={!(detailsModel.details?.length)}
              >
                <ListTree size={13} /> Test details
              </button>
              <button
                type="button"
                className={`${styles['drawer__tab']} ${drawerView === 'metrics' ? styles.on : ''}`}
                onClick={() => setDrawerView('metrics')}
                disabled={!Object.keys(detailsModel.metric_scores || {}).length}
              >
                <Gauge size={13} /> Metric scores
              </button>
            </div>

            {drawerView === 'tests' && (
              <div className={styles['drawer__stats']}>
                <span className={styles['drawer__stat']}>
                  <CheckCircle2 size={13} className={styles['drawer__stat-icon--pass']} />
                  {detailsModel.passed_tests ?? 0} passed
                </span>
                <span className={styles['drawer__stat']}>
                  <XCircle size={13} className={styles['drawer__stat-icon--fail']} />
                  {detailsModel.failed_tests ?? 0} failed
                </span>
              </div>
            )}

            <div className={styles['drawer__body']}>
              {drawerView === 'tests' &&
                (detailsModel.details || []).map((d, i) => (
                  <div key={i} className={`${styles['detail-card']} ${d.passed ? styles['detail-card--pass'] : styles['detail-card--fail']}`}>
                    <div className={styles['detail-card__hdr']}>
                      <span className={styles['detail-card__task']}>{d.task || `Test ${i + 1}`}</span>
                      <span className={`${styles['detail-card__badge']} ${d.passed ? styles['detail-card__badge--pass'] : styles['detail-card__badge--fail']}`}>
                        {d.passed ? <CheckCircle2 size={12} /> : <XCircle size={12} />}
                        {d.passed ? 'Passed' : 'Failed'}
                      </span>
                    </div>
                    <div className={styles['detail-card__field']}>
                      <span className={styles['detail-card__label']}>Input</span>
                      <div className={styles['detail-card__text']}>{d.input || '—'}</div>
                    </div>
                    <div className={styles['detail-card__row']}>
                      <div className={styles['detail-card__field']}>
                        <span className={styles['detail-card__label']}>Expected</span>
                        <div className={styles['detail-card__text']}>{d.expected_output || '—'}</div>
                      </div>
                      <div className={styles['detail-card__field']}>
                        <span className={styles['detail-card__label']}>Actual</span>
                        <div className={`${styles['detail-card__text']} ${!d.passed ? styles['detail-card__text--fail'] : ''}`}>
                          {d.actual_output || '—'}
                        </div>
                      </div>
                    </div>
                  </div>
                ))}

              {/* Metric-scores view — one bar per metric (e.g. Answer
                  Relevancy, Toxicity, Bias), values are 0-100. */}
              {drawerView === 'metrics' &&
                Object.entries(detailsModel.metric_scores || {}).map(([label, value]) => {
                  const numeric = typeof value === 'number' && !Number.isNaN(value) ? value : 0;
                  const pct = Math.max(0, Math.min(100, numeric));
                  const tier = pct >= 80 ? 'good' : pct >= 50 ? 'mid' : 'low';
                  return (
                    <div key={label} className={styles['metric-card']}>
                      <div className={styles['metric-card__hdr']}>
                        <span className={styles['metric-card__label']}>{label}</span>
                        <span className={`${styles['metric-card__value']} ${styles[`metric-card__value--${tier}`]}`}>
                          {numeric}%
                        </span>
                      </div>
                      <div className={styles['metric-card__track']}>
                        <div
                          className={`${styles['metric-card__fill']} ${styles[`metric-card__fill--${tier}`]}`}
                          style={{ width: `${pct}%` }}
                        />
                      </div>
                    </div>
                  );
                })}
            </div>
          </>
        )}
      </div>

      {/* ---------- Stop-evaluation confirm dialog ---------- */}
      {cancelTarget && (
        <div className={styles['confirm-overlay']} onClick={() => setCancelTarget(null)}>
          <div className={styles['confirm-dialog']} role="alertdialog" aria-modal="true" onClick={(evt) => evt.stopPropagation()}>
            <div className={styles['confirm-dialog__icon']}>
              <AlertTriangle size={20} />
            </div>
            <h3 className={styles['confirm-dialog__title']}>Stop this evaluation?</h3>
            <p className={styles['confirm-dialog__body']}>
              <strong>{cancelTarget.name || 'This evaluation'}</strong> is still running. Stopping it now will end the
              run early — progress made so far won't be recoverable.
            </p>
            <div className={styles['confirm-dialog__actions']}>
              <button type="button" className={styles['confirm-dialog__btn--ghost']} onClick={() => setCancelTarget(null)}>
                Keep running
              </button>
              <button type="button" className={styles['confirm-dialog__btn--danger']} onClick={confirmCancel}>
                <StopCircle size={14} />
                Stop evaluation
              </button>
            </div>
          </div>
        </div>
      )}
    </div>
  );
}






















@use '../../styles/_variables' as *;

// ===========================================================================
// History — matches the Run Console / Dashboard / Providers / Model Catalog /
// Datasets design system: ink/paper palette, ultramarine signal accent, mono
// instrument labels, hover-lift, mono numerals. Master–detail split shell is
// self-contained here (no dependency on global .split-shell*).
// ===========================================================================

// $ink, $ink-2, $ink-3, $paper, $card, $line, $line-2, $signal, $signal-2,
// $wash, $ok, $ok-wash, $danger, $danger-wash, $ink-wash all come from
// _variables.scss (imported above via `@use ... as *`) — they already
// resolve to theme-aware CSS custom properties, so History picks up dark
// mode automatically without redeclaring anything here.
//
// $amber / $amber-wash aren't part of the shared "ink" names (the shared
// tokens use $amber-ink / $amber-ink-wash to avoid clashing with the
// brand-palette $amber further up _variables.scss), so alias them locally
// rather than renaming every usage in this file.
$amber:      $amber-ink;
$amber-wash: $amber-ink-wash;

// $soft/$lift now point at the shared theme-aware shadow tokens instead of
// a hardcoded ink-black rgba — so card shadows get properly darker in
// dark mode instead of staying a flat, barely-visible tint.
$soft: $shadow-2;
$lift: $shadow-3;

// Base font-size the whole History component's internal `em` scale is built
// on. All descendant font-sizes in this file are expressed in `em` relative
// to this, so bumping it (e.g. on wide screens below) scales the whole
// component proportionally from one place — same convention as Model
// Catalog / Sidebar / Providers.
$history-base-font: 0.8125rem;

%micro {
  font-family: $font-mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

.history {
  // master scale control — every em-based font-size below responds to this
  font-size: $history-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding: 24px 32px 20px;
    margin-bottom: 0;
    border-bottom: 1px solid $line;
    background: $card;

    h1 {
      font-family: $font-display;
      font-size: 1.8462em; // 1.5rem / 0.8125rem
      font-weight: 800;
      letter-spacing: -0.02em;
      color: $ink;
      line-height: 1.2;
    }
  }

  &__header-eyebrow {
    @extend %micro;
    display: flex;
    align-items: center;
    gap: 8px;
    color: $signal;
    margin-bottom: 6px;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $signal;
    }
  }

  &__header-sub {
    margin-top: 4px;
    font-size: 1.0385em; // 0.84375rem / 0.8125rem
    color: $ink-2;
  }

  &__header-meta {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 7px;
    padding: 7px 13px;
    border-radius: 999px;
    border: 1px solid $line;
    background: $paper;
    font-family: $font-mono;
    font-size: 0.8846em; // 0.71875rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ink-2;
    white-space: nowrap;
    margin-bottom: 3px;
  }
}

@property --angle {
  syntax: '<angle>';
  initial-value: 0deg;
  inherits: false;
}
@keyframes history-rotate-angle {
  to { --angle: 360deg; }
}
@keyframes history-live-pulse {
  0%, 100% { opacity: 1; transform: scale(1); }
  50% { opacity: 0.5; transform: scale(1.3); }
}
@keyframes history-spin { to { transform: rotate(360deg); } }

// Fixed-shell override: list + detail scroll independently, so pg-body
// itself must not scroll — plain flex:1/min-height:0 pass-through.
.pg-body-fixed {
  overflow: hidden;
  display: flex;
  flex-direction: column;
  padding: 20px 32px 24px;
}

// ---- self-contained split shell -------------------------------------------
.shell {
  flex: 1;
  min-height: 0;
  display: flex;
  gap: 16px;
}

.sidebar {
  flex-shrink: 0;
  width: 380px;
  display: flex;
  flex-direction: column;
  min-height: 0;
  padding: 16px;
  background: $card;
  border: 1px solid $line;
  border-radius: 16px;
  box-shadow: $soft;
}

.detail {
  flex: 1;
  min-width: 0;
  min-height: 0;
  overflow-y: auto;
  padding: 24px;
  background: $card;
  border: 1px solid $line;
  border-radius: 16px;
  box-shadow: $soft;
}

.filters { flex-shrink: 0; }

// ---- filter toolbar --------------------------------------------------------
.filter-toolbar {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 6px 8px;
  margin-bottom: 8px;
  background: $paper;
  border: 1px solid $line;
  border-radius: 12px;
}

.filter-toolbar__label {
  flex-shrink: 0;
  @extend %micro;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  color: $ink-3;
  padding-left: 4px;
}

.filter-toolbar__divider {
  flex-shrink: 0;
  width: 1px;
  height: 16px;
  background: $line;
}

.filter-toolbar__btn {
  position: relative;
  flex-shrink: 0;
  width: 30px;
  height: 30px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 8px;
  border: 1px solid transparent;
  background: transparent;
  color: $ink-2;
  cursor: pointer;
  transition: background 0.15s ease, color 0.15s ease, border-color 0.15s ease, box-shadow 0.15s ease;

  &:hover { background: $card; color: $signal; box-shadow: $soft; }

  &.on {
    border-color: $signal;
    background: $card;
    color: $signal;
    box-shadow: 0 0 0 3px $wash;
  }
}

.filter-toolbar__dot {
  position: absolute;
  top: -2px;
  right: -2px;
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: $signal;
  border: 1.5px solid $paper;
}

.filter-toolbar__summary {
  display: flex;
  align-items: center;
  gap: 6px;
  flex-wrap: wrap;
  min-width: 0;
  overflow: hidden;
}

.filter-chip {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  font-family: $font-mono;
  font-size: 0.8077em; // 0.65625rem / 0.8125rem
  font-weight: 700;
  color: $signal;
  background: $wash;
  border: 1px solid rgba($signal, 0.18);
  border-radius: 999px;
  padding: 4px 8px;
  white-space: nowrap;
  max-width: 140px;

  span { overflow: hidden; text-overflow: ellipsis; }

  svg {
    cursor: pointer;
    flex-shrink: 0;
    opacity: 0.6;
    transition: opacity 0.15s ease;
    &:hover { opacity: 1; }
  }
}

// ---- collapsible filter panel ----------------------------------------------
.filter-panel {
  display: grid;
  grid-template-rows: 0fr;
  opacity: 0;
  transition: grid-template-rows 0.18s ease, opacity 0.15s ease, margin-bottom 0.18s ease;

  > * {
    overflow: hidden;
    min-height: 0;
    background: $paper;
    border: 1px solid $line;
    border-radius: 12px;
    padding: 10px;
  }

  &--open {
    grid-template-rows: 1fr;
    opacity: 1;
    margin-bottom: 8px;
  }
}

// ---- filter panel inner controls (self-contained search + pills) -----------
.panel-search {
  position: relative;

  svg {
    position: absolute;
    top: 50%;
    left: 12px;
    transform: translateY(-50%);
    color: $ink-3;
    pointer-events: none;
  }

  input {
    width: 100%;
    border: 1.5px solid $line;
    border-radius: 9px;
    padding: 8px 11px 8px 36px;
    font-size: 1em; // 0.8125rem / 0.8125rem
    font-family: $font-body;
    color: $ink;
    background: $card;

    &::placeholder { color: $ink-3; }
    &:focus { outline: none; border-color: $signal; box-shadow: 0 0 0 3px $wash; }
  }
}

.panel-pills {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.panel-pill {
  padding: 6px 12px;
  border: 1px solid $line;
  border-radius: 999px;
  background: $card;
  color: $ink-2;
  font-size: 0.9231em; // 0.75rem / 0.8125rem
  font-weight: 650;
  cursor: pointer;
  transition: all 0.14s ease;

  &:hover { border-color: $ink-3; color: $ink; }

  &.on {
    border-color: $signal;
    background: $signal;
    color: #fff;
  }
}

.empty {
  padding: 24px;
  text-align: center;
  color: $ink-3;
  font-size: 1em; // 0.8125rem / 0.8125rem
  display: flex;
  align-items: center;
  gap: 8px;
  justify-content: center;
}

// ---- true empty state (no evaluations at all yet) --------------------
.sidebar-empty {
  flex: 1;
  min-height: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  text-align: center;
  gap: 4px;
  padding: 40px 24px;
}
.sidebar-empty__icon {
  width: 52px;
  height: 52px;
  border-radius: 16px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: $wash;
  color: $signal;
  margin-bottom: 14px;
}
.sidebar-empty__title {
  font-family: $font-display;
  font-size: 1.1538em; // 0.9375rem / 0.8125rem
  font-weight: 800;
  letter-spacing: -0.01em;
  color: $ink;
}
.sidebar-empty__sub {
  max-width: 240px;
  margin-top: 6px;
  font-size: 0.9615em; // 0.78125rem / 0.8125rem
  line-height: 1.55;
  color: $ink-3;
}
.sidebar-empty__cta {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  margin-top: 18px;
  padding: 9px 16px;
  border-radius: 10px;
  border: 1px solid transparent;
  background: $signal;
  color: #fff;
  font-family: $font-mono;
  font-size: 0.8846em; // 0.71875rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.03em;
  cursor: pointer;
  transition: background 0.15s ease, transform 0.15s ease;

  &:hover { background: $signal-2; transform: translateY(-1px); }
}

// ---- evaluation list rows --------------------------------------------------
.rows {
  flex: 1;
  min-height: 0;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin-top: 14px;
  padding-right: 2px;
}

.row {
  position: relative;
  border: 1.5px solid $line;
  border-radius: 14px;
  padding: 14px;
  cursor: pointer;
  background: $card;
  transition: border-color 0.15s ease, box-shadow 0.15s ease, transform 0.15s ease, background 0.15s ease;
}
.row:hover { border-color: $ink-3; box-shadow: $soft; transform: translateY(-1px); }
.row.selected { border-color: $signal; background: $wash; box-shadow: 0 0 0 1px $signal inset; }

// Running-state: a multi-color light traveling around the border via a
// rotating conic angle.
.row--running {
  --angle: 0deg;
  border: 1.5px solid transparent;
  background:
    linear-gradient($card, $card) padding-box,
    conic-gradient(
      from var(--angle),
      $line 0%,
      $signal 4%,
      $violet-ink 8%,
      $rose-ink 12%,
      $line 18%
    ) border-box;
  animation: history-rotate-angle 2.4s linear infinite;
}
.row--running.selected {
  background:
    linear-gradient($wash, $wash) padding-box,
    conic-gradient(
      from var(--angle),
      $line 0%,
      $signal 4%,
      $violet-ink 8%,
      $rose-ink 12%,
      $line 18%
    ) border-box;
}

// Running cards aren't clickable (nothing to show yet — no results, no
// report) — the "Stop evaluation" button is the only interactive element,
// so the card itself shouldn't hint at being clickable.
.row--unselectable {
  cursor: default;

  &:hover { transform: none; }
}

.row__stop-btn {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  margin-left: auto; // pushes it to the far right of the badges row
  padding: 3px 9px 3px 7px;
  border-radius: 999px;
  border: 1px solid rgba($danger, 0.3);
  background: $card;
  color: $danger;
  font-family: $font-mono;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.02em;
  white-space: nowrap;
  cursor: pointer;
  transition: border-color 0.15s ease, background 0.15s ease, color 0.15s ease, transform 0.15s ease, box-shadow 0.15s ease;

  &:hover:not(:disabled) {
    background: $danger;
    border-color: $danger;
    color: #fff;
    box-shadow: 0 0 0 3px rgba($danger, 0.12);
  }

  &:active:not(:disabled) { transform: scale(0.96); }

  &:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }
}


.row__top { display: flex; align-items: center; gap: 10px; margin-bottom: 8px; }
.row__icon {
  width: 30px;
  height: 30px;
  border-radius: 9px;
  background: $wash;
  color: $signal;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}
.row__name {
  font-family: $font-display;
  font-weight: 700;
  font-size: 1.0769em; // 0.875rem / 0.8125rem
  color: $ink;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.row__badges { display: flex; align-items: center; gap: 6px; margin-bottom: 6px; flex-wrap: wrap; }
.row__meta { font-family: $font-mono; font-size: 0.8462em; /* 0.6875rem / 0.8125rem */ color: $ink-3; margin-bottom: 8px; }
.row__stats {
  display: flex;
  gap: 12px;
  font-family: $font-mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  color: $ink-2;
  flex-wrap: wrap;
}

// ---- type tag + status badge (shared visual grammar) -----------------------
.type-tag {
  display: inline-flex;
  align-items: center;
  font-family: $font-mono;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  color: $signal;
  background: $wash;
  border-radius: 6px;
  padding: 3px 8px;
  white-space: nowrap;
}

.status {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 3px 10px 3px 9px;
  border-radius: 999px;
  font-family: $font-mono;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.06em;
  text-transform: uppercase;

  &::before { content: ''; width: 5px; height: 5px; border-radius: 50%; }

  &--completed { color: $ok; background: $ok-wash; &::before { background: $ok; } }
  &--running   { color: $signal; background: $wash; &::before { display: none; } }
  &--pending   { color: $amber; background: $amber-wash; &::before { background: $amber; } }
  &--failed,
  &--canceled  { color: $ink-3; background: $ink-wash; &::before { background: $ink-3; } }
}

.live-dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: currentColor;
  display: inline-block;
  animation: history-live-pulse 1.4s ease-in-out infinite;
}

// ---- detail panel ----------------------------------------------------------
.detail-empty {
  padding: 80px 24px;
  text-align: center;
  color: $ink-3;
  font-size: 1.0769em; // 0.875rem / 0.8125rem
}

.detail-hdr {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 24px;
  flex-wrap: wrap;
  gap: 16px;
}
.detail-hdr__badges { display: flex; gap: 8px; margin-bottom: 12px; }
.detail-hdr__name {
  font-family: $font-display;
  font-size: 1.6923em; // 1.375rem / 0.8125rem
  font-weight: 800;
  letter-spacing: -0.02em;
  color: $ink;
}
.detail-hdr__date { font-family: $font-mono; font-size: 0.8846em; /* 0.71875rem / 0.8125rem */ color: $ink-3; margin-top: 6px; }
.detail-hdr__actions {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

.dl-btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 7px 12px;
  border-radius: 9px;
  border: 1px solid $line;
  background: $paper;
  color: $ink-2;
  font-family: $font-mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.03em;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

  &:hover:not(:disabled) { border-color: $signal; color: $signal; background: $wash; }

  &:disabled {
    opacity: 0.45;
    cursor: not-allowed;
  }
}

.summary-cards {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 12px;
  margin-bottom: 24px;
}
.summary-card {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 15px 16px;
  background: $paper;
  border: 1px solid $line;
  border-radius: 14px;
}
.summary-card__icon {
  flex-shrink: 0;
  width: 34px;
  height: 34px;
  border-radius: 10px;
  display: grid;
  place-items: center;
  background: $card;
  border: 1px solid $line;

  &--win { color: $amber; }
  &--info { color: $signal; }
  &--status { color: $ok; }
}
.summary-card__label {
  @extend %micro;
  font-size: 0.6923em; // 0.5625rem / 0.8125rem
  color: $ink-3;
}
.summary-card__val {
  font-size: 1em; // 0.8125rem / 0.8125rem
  font-weight: 700;
  color: $ink;
  margin-top: 3px;
}

// ---- results metadata strip (benchmark / dataset / started / metrics) ------
.meta-strip {
  display: flex;
  flex-wrap: wrap;
  gap: 20px;
  padding: 12px 16px;
  margin-bottom: 16px;
  background: $paper;
  border: 1px solid $line;
  border-radius: 12px;
}
.meta-strip__item {
  display: flex;
  flex-direction: column;
  gap: 3px;
  min-width: 0;
}
.meta-strip__label {
  @extend %micro;
  font-size: 0.6923em; // 0.5625rem / 0.8125rem
  color: $ink-3;
}
.meta-strip__val {
  font-family: $font-mono;
  font-size: 0.9231em; // 0.75rem / 0.8125rem
  font-weight: 600;
  color: $ink;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.meta-strip__chips {
  display: flex;
  flex-wrap: wrap;
  gap: 5px;
}

// ---- test-details launcher button (in results table) ------------------
.cell-details { width: 32px; text-align: center; }
.details-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 26px;
  height: 26px;
  border-radius: 7px;
  border: 1px solid $line;
  background: $paper;
  color: $ink-2;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

  &:hover { border-color: $signal; color: $signal; background: $wash; }
}

// ---- test-details slide-over drawer ------------------------------------
.drawer-overlay {
  position: fixed;
  inset: 0;
  // Deliberately hardcoded, not themed: an overlay scrim must stay dark
  // in both themes (same "always dark" reasoning as $ink-solid in
  // _theme.scss) — using the themed $ink here would go near-white, and
  // invisible, in dark mode.
  background: rgba(20, 22, 27, 0.32);
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.22s ease;
  z-index: 40;

  &--open { opacity: 1; pointer-events: auto; }
}

.drawer {
  position: fixed;
  top: 0;
  right: 0;
  height: 100%;
  width: 480px;
  max-width: 92vw;
  background: $card;
  border-left: 1px solid $line;
  box-shadow: -18px 0 40px -20px rgba(20, 22, 27, 0.35);
  transform: translateX(100%);
  transition: transform 0.26s cubic-bezier(0.22, 1, 0.36, 1);
  z-index: 41;
  display: flex;
  flex-direction: column;
  min-height: 0;

  &--open { transform: translateX(0); }
}

.drawer__header {
  flex-shrink: 0;
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 12px;
  padding: 20px 20px 16px;
  border-bottom: 1px solid $line;
}
.drawer__eyebrow {
  @extend %micro;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  color: $signal;
  margin-bottom: 6px;
}
.drawer__title {
  font-family: $font-display;
  font-size: 1.3846em; // 1.125rem / 0.8125rem
  font-weight: 800;
  letter-spacing: -0.01em;
  color: $ink;
}
.drawer__sub {
  margin-top: 5px;
  font-family: $font-mono;
  font-size: 0.8846em; // 0.71875rem / 0.8125rem
  color: $ink-3;
}
.drawer__close {
  flex-shrink: 0;
  width: 30px;
  height: 30px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 8px;
  border: 1px solid $line;
  background: $paper;
  color: $ink-2;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

  &:hover { border-color: $ink-3; color: $ink; }
}

// ---- tab switcher (test details vs metric scores) --------------------
.drawer__tabs {
  flex-shrink: 0;
  display: flex;
  gap: 6px;
  padding: 10px 20px 0;
  border-bottom: 1px solid $line;
  background: $card;
}
.drawer__tab {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 8px 12px;
  border: 1px solid transparent;
  border-bottom: 2px solid transparent;
  border-radius: 8px 8px 0 0;
  background: transparent;
  color: $ink-3;
  font-family: $font-mono;
  font-size: 0.8846em; // 0.71875rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.02em;
  cursor: pointer;
  transition: color 0.15s ease, border-color 0.15s ease;
  margin-bottom: -1px;

  &:hover:not(:disabled) { color: $ink-2; }

  &.on {
    color: $signal;
    border-bottom-color: $signal;
  }

  &:disabled {
    opacity: 0.35;
    cursor: not-allowed;
  }
}

.drawer__stats {
  flex-shrink: 0;
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 14px;
  padding: 12px 20px;
  border-bottom: 1px solid $line;
  background: $paper;
  font-family: $font-mono;
  font-size: 0.8846em; // 0.71875rem / 0.8125rem
  font-weight: 700;
  color: $ink-2;
}
.drawer__stat {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  text-transform: capitalize;
}
.drawer__stat-icon--pass { color: $ok; }
.drawer__stat-icon--fail { color: $danger; }

.drawer__body {
  flex: 1;
  min-height: 0;
  overflow-y: auto;
  padding: 16px 20px 24px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

// ---- metric-score bars (Answer Relevancy, Toxicity, Bias, ...) -------
.metric-card {
  border: 1px solid $line;
  border-radius: 12px;
  padding: 13px 16px;
  background: $paper;
}
.metric-card__hdr {
  display: flex;
  align-items: baseline;
  justify-content: space-between;
  gap: 10px;
  margin-bottom: 9px;
}
.metric-card__label {
  font-family: $font-display;
  font-weight: 700;
  font-size: 1em; // 0.8125rem / 0.8125rem
  color: $ink;
  text-transform: capitalize;
}
.metric-card__value {
  font-family: $font-mono;
  font-size: 1em; // 0.8125rem / 0.8125rem
  font-weight: 800;
  flex-shrink: 0;

  &--good { color: $ok; }
  &--mid  { color: $amber; }
  &--low  { color: $danger; }
}
.metric-card__track {
  height: 7px;
  border-radius: 999px;
  background: $line-2;
  overflow: hidden;
}
.metric-card__fill {
  height: 100%;
  border-radius: 999px;
  transition: width 0.3s ease;

  &--good { background: $ok; }
  &--mid  { background: $amber; }
  &--low  { background: $danger; }
}

.detail-card {
  border: 1px solid $line;
  border-left: 3px solid $line;
  border-radius: 12px;
  padding: 14px 16px;
  background: $paper;

  &--pass { border-left-color: $ok; }
  &--fail { border-left-color: $danger; background: rgba($danger, 0.03); }
}
.detail-card__hdr {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  margin-bottom: 10px;
}
.detail-card__task {
  font-family: $font-display;
  font-weight: 700;
  font-size: 1em; // 0.8125rem / 0.8125rem
  color: $ink;
}
.detail-card__badge {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 2px 8px;
  border-radius: 999px;
  font-family: $font-mono;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  flex-shrink: 0;

  &--pass { color: $ok; background: $ok-wash; }
  &--fail { color: $danger; background: $danger-wash; }
}
.detail-card__row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
  margin-top: 10px;
}
.detail-card__field {
  min-width: 0;
}
.detail-card__label {
  @extend %micro;
  font-size: 0.6923em; // 0.5625rem / 0.8125rem
  color: $ink-3;
  display: block;
  margin-bottom: 4px;
}
.detail-card__text {
  font-family: $font-mono;
  font-size: 0.9231em; // 0.75rem / 0.8125rem
  line-height: 1.5;
  color: $ink;
  white-space: pre-wrap;
  word-break: break-word;
  background: $card;
  border: 1px solid $line-2;
  border-radius: 8px;
  padding: 8px 10px;
}
.detail-card__text--fail { color: $danger; border-color: rgba($danger, 0.25); }

@media (max-width: 640px) {
  .drawer { width: 100%; max-width: 100vw; }
  .detail-card__row { grid-template-columns: 1fr; }
}

// ---- results table ---------------------------------------------------------
.results {
  border: 1px solid $line;
  border-radius: 14px;
  overflow: hidden;
}

.results-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 1.0385em; // 0.84375rem / 0.8125rem

  thead th {
    text-align: left;
    background: $paper;
    border-bottom: 1px solid $line;
    @extend %micro;
    font-size: 0.6923em; // 0.5625rem / 0.8125rem
    color: $ink-3;
    padding: 11px 14px;
    white-space: nowrap;
  }

  tbody tr {
    border-bottom: 1px solid $line-2;
    transition: background 0.13s ease;

    &:last-child { border-bottom: 0; }
    &:hover { background: $paper; }
  }

  tbody tr.winner {
    background: rgba($amber, 0.05);
    &:hover { background: rgba($amber, 0.09); }
  }

  tbody td {
    padding: 12px 14px;
    color: $ink;
  }
}

.cell-rank { font-family: $font-mono; font-weight: 700; color: $ink; }
.cell-model { font-family: $font-display; font-weight: 700; color: $ink; }
.cell-provider { color: $ink-2; }
.cell-num { font-family: $font-mono; font-size: 1em; /* 0.8125rem / 0.8125rem */ font-weight: 700; color: $ink; }
.cell-num--muted { font-weight: 500; color: $ink-2; }
.cell-pass { font-family: $font-mono; font-size: 1em; /* 0.8125rem / 0.8125rem */ font-weight: 700; color: $ok; }
.cell-fail { font-family: $font-mono; font-size: 1em; /* 0.8125rem / 0.8125rem */ font-weight: 700; color: $danger; }

.status-message {
  padding: 40px;
  text-align: center;
  background: $paper;
  border: 1px dashed $line;
  border-radius: 14px;
  color: $ink-2;
  font-size: 1.0769em; // 0.875rem / 0.8125rem
}

.spin { animation: history-spin 0.8s linear infinite; }

// ---- stop-evaluation confirm dialog ------------------------------------
.confirm-overlay {
  position: fixed;
  inset: 0;
  // Deliberately hardcoded, not themed: same "always dark" scrim reasoning
  // as the drawer overlay above.
  background: rgba(20, 22, 27, 0.4);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
  z-index: 50;
}

.confirm-dialog {
  width: 100%;
  max-width: 400px;
  background: $card;
  border: 1px solid $line;
  border-radius: 16px;
  box-shadow: $lift;
  padding: 24px;
}

.confirm-dialog__icon {
  width: 40px;
  height: 40px;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: $danger-wash;
  color: $danger;
  margin-bottom: 14px;
}

.confirm-dialog__title {
  font-family: $font-display;
  font-size: 1.1538em; // 0.9375rem / 0.8125rem
  font-weight: 800;
  letter-spacing: -0.01em;
  color: $ink;
}

.confirm-dialog__body {
  margin-top: 8px;
  font-size: 0.9615em; // 0.78125rem / 0.8125rem
  line-height: 1.55;
  color: $ink-2;

  strong { color: $ink; }
}

.confirm-dialog__actions {
  display: flex;
  justify-content: flex-end;
  gap: 8px;
  margin-top: 22px;
}

.confirm-dialog__btn--ghost {
  padding: 9px 15px;
  border-radius: 9px;
  border: 1px solid $line;
  background: $paper;
  color: $ink-2;
  font-size: 0.9231em; // 0.75rem / 0.8125rem
  font-weight: 650;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease;

  &:hover { border-color: $ink-3; color: $ink; }
}

.confirm-dialog__btn--danger {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 9px 15px;
  border-radius: 9px;
  border: 1px solid transparent;
  background: $danger;
  color: #fff;
  font-size: 0.9231em; // 0.75rem / 0.8125rem
  font-weight: 700;
  cursor: pointer;
  transition: background 0.15s ease, transform 0.15s ease;

  &:hover { background: darken($danger, 8%); transform: translateY(-1px); }
}

@media (max-width: 900px) {
  .shell { flex-direction: column; }
  .sidebar { width: 100%; }
  .summary-cards { grid-template-columns: 1fr; }
  // Once stacked, fall back to one normal scrolling column.
  .pg-body-fixed { overflow-y: auto; }
  .sidebar, .detail { overflow-y: visible; min-height: 0; }
  .rows { overflow-y: visible; }
}

@media (max-width: 640px) {
  .history__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .pg-body-fixed { padding: 16px 18px 22px; }
}
