//History.tsx
import { useEffect, useMemo, useRef, useState } from 'react';
import { useSearchParams, useNavigate } from 'react-router-dom';
import {
  Search, Sparkles, Bot, Layers, Loader2, Download, ListTree, Gauge, CheckCircle2, XCircle,
  Award, ListChecks, Clock, History as HistoryIcon, SlidersHorizontal, CalendarDays, X, PlaySquare,
  StopCircle, AlertTriangle, Trash2, ChevronLeft, ChevronRight,
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import {
  fetchEvaluations, fetchEvaluationResults, cancelEvaluation, deleteEvaluation,
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

const PAGE_SIZE_OPTIONS = [20, 30, 40, 50];

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

  const { list: rawList, total, listStatus, listError, resultsByEvalId, resultsStatusByEvalId, resultsErrorByEvalId } = useAppSelector((s) => s.evaluations);
  const list = rawList || [];
  const models = useAppSelector((s) => s.models.items) || [];
  const providers = useAppSelector((s) => s.providers.items) || [];
  const downloadingId = useAppSelector((s) => s.reports.downloadingId);
  const cancelingId = useAppSelector((s) => s.evaluations.cancelingId);
  const deletingId = useAppSelector((s) => s.evaluations.deletingId);

  const [search, setSearch] = useState('');
  const [typeFilter, setTypeFilter] = useState('All');
  const [dateFilter, setDateFilter] = useState('all');
  const [statusFilter, setStatusFilter] = useState('All');
  const [activeFilter, setActiveFilter] = useState<'search' | 'type' | 'date' | 'status' | null>(null);
  const [detailsModel, setDetailsModel] = useState<ModelResult | null>(null);
  const [drawerView, setDrawerView] = useState<'tests' | 'metrics'>('tests');
  // Row + action awaiting "are you sure?" confirmation before calling the
  // cancel/delete API — one dialog handles both.
  const [confirmTarget, setConfirmTarget] = useState<{ evaluation: EvaluationListItem; action: 'cancel' | 'delete' } | null>(null);
  const searchInputRef = useRef<HTMLInputElement>(null);

  // typeFilter/statusFilter are now server-side (query params on GET
  // /evaluations), so changing either means a genuinely different result
  // set — jump back to page 1 rather than potentially landing on an
  // out-of-range page for the new filter.
  const changeTypeFilter = (value: string) => {
    setTypeFilter(value);
    setPage(1);
  };
  const changeStatusFilter = (value: string) => {
    setStatusFilter(value);
    setPage(1);
  };

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

  // Local page/page-size state drives the server request — Redux's `page`/
  // `pageSize` (set above) just echo back whatever the most recent fetch
  // actually requested, once it resolves.
  const [pageState, setPage] = useState(1);
  const [pageSizeState, setPageSizeState] = useState(20);
  const changePageSize = (value: number) => {
    setPageSizeState(value);
    setPage(1);
  };

  // Initial load + refetch on page/page-size/filter change, plus a silent
  // 10s background poll of whatever's currently being viewed (spec §2.4) —
  // the poll never disrupts the loading/error UI over data already on
  // screen (see fetchEvaluations.pending in the slice).
  useEffect(() => {
    dispatch(fetchEvaluations({ page: pageState, pageSize: pageSizeState, status: statusFilter, evalType: typeFilter }));
    const interval = setInterval(() => {
      dispatch(fetchEvaluations({ page: pageState, pageSize: pageSizeState, status: statusFilter, evalType: typeFilter, silent: true }));
    }, 10000);
    return () => clearInterval(interval);
  }, [dispatch, pageState, pageSizeState, statusFilter, typeFilter]);

  const totalPages = Math.max(1, Math.ceil(total / pageSizeState));
  const rangeStart = total === 0 ? 0 : (pageState - 1) * pageSizeState + 1;
  const rangeEnd = Math.min(pageState * pageSizeState, total);

  // search/date remain client-side, applied only to the current page's rows
  // — the API doesn't take a search or date-range param yet. type/status
  // are now handled server-side (see the fetch effect above) rather than
  // filtered here.
  const filtered = useMemo(() => {
    return list.filter((e) => {
      if (search && !(e.name || '').toLowerCase().includes(search.toLowerCase())) return false;
      if (!withinDateRange(e.created_at, dateFilter)) return false;
      return true;
    });
  }, [list, search, dateFilter]);

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
    // interactive element on that card. Same for a card mid-delete.
    if (clicked && (clicked.status === 'running' || deletingId === clicked.id)) return;
    setSearchParams({ id });
    setDetailsModel(null);
    if (clicked && clicked.status === 'completed') {
      dispatch(fetchEvaluationResults(id));
    }
  };

  // Drives the delete animation: the row pulses (red border + sheen) while
  // the API call is in flight, then — once it actually succeeds — collapses
  // and fades out over ~320ms instead of disappearing the instant Redux
  // removes it from `list`. We keep a stashed copy of the row (plus its
  // original position) so it can keep rendering through both phases even
  // after the real data is already gone from the store.
  const [deleteAnim, setDeleteAnim] = useState<{ item: EvaluationListItem; index: number; exiting: boolean } | null>(null);
  const DELETE_EXIT_MS = 320;

  // Confirm-and-act flow shared by the "Stop" (running cards) and "Delete"
  // (non-running cards) buttons.
  const requestConfirm = (e: React.MouseEvent, evaluation: EvaluationListItem, action: 'cancel' | 'delete') => {
    e.stopPropagation(); // don't let this bubble into selectRow
    setConfirmTarget({ evaluation, action });
  };
  const runConfirmedAction = async () => {
    if (!confirmTarget) return;
    const { evaluation, action } = confirmTarget;
    setConfirmTarget(null);

    if (action === 'cancel') {
      dispatch(cancelEvaluation(evaluation.id));
      return;
    }

    const index = filtered.findIndex((e) => e.id === evaluation.id);
    setDeleteAnim({ item: evaluation, index: index === -1 ? filtered.length : index, exiting: false });

    const result = await dispatch(deleteEvaluation(evaluation.id));
    if (deleteEvaluation.fulfilled.match(result)) {
      // If the row being deleted is currently selected/showing in the
      // detail panel, clear the selection so it doesn't linger on screen.
      if (selectedId === evaluation.id) setSearchParams({});
      // Flip to the collapse/fade-out phase, then drop the stashed row
      // once the CSS transition has had time to finish.
      setDeleteAnim((prev) => (prev && prev.item.id === evaluation.id ? { ...prev, exiting: true } : prev));
      window.setTimeout(() => {
        setDeleteAnim((prev) => (prev && prev.item.id === evaluation.id ? null : prev));
      }, DELETE_EXIT_MS);
    } else {
      // Delete failed — the row is still in the store, so just drop the
      // stashed copy and let it re-render normally.
      setDeleteAnim(null);
    }
  };

  // Merges the stashed row back into the rendered list for as long as
  // deleteAnim is active — covers both the brief window where the API call
  // has already succeeded and Redux has removed it from `list`/`filtered`,
  // and the exit-animation phase that follows.
  const displayList = useMemo(() => {
    if (!deleteAnim || filtered.some((e) => e.id === deleteAnim.item.id)) return filtered;
    const merged = [...filtered];
    merged.splice(Math.min(deleteAnim.index, merged.length), 0, deleteAnim.item);
    return merged;
  }, [filtered, deleteAnim]);

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

  // Close the stop/delete confirm dialog with Escape.
  useEffect(() => {
    if (!confirmTarget) return;
    const onKey = (e: KeyboardEvent) => {
      if (e.key === 'Escape') setConfirmTarget(null);
    };
    window.addEventListener('keydown', onKey);
    return () => window.removeEventListener('keydown', onKey);
  }, [confirmTarget]);

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
                      <X size={11} onClick={() => changeTypeFilter('All')} />
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
                      <X size={11} onClick={() => changeStatusFilter('All')} />
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
                            changeTypeFilter(t);
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
                            changeStatusFilter(value);
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
              {listStatus !== 'loading' && total > 0 && (list.length === 0 || filtered.length === 0) && (
                <div className={styles.empty}>No evaluations match your filters.</div>
              )}
            </div>

            <div className={styles.rows}>
              {listStatus === 'loading' && list.length === 0 && <SkeletonListRows count={5} />}

              {/* True empty state — the account has no evaluations at all yet
                  (total === 0), as opposed to "no evaluations match your
                  current page/filters" (handled above via styles.empty),
                  which can now happen even with list.length === 0 if a
                  server-side status/type filter or a later page just has no
                  rows while other pages/filters still do. */}
              {listStatus === 'succeeded' && total === 0 && (
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

              {displayList.map((e) => {
                const Icon = TYPE_ICON[e.eval_type] || Sparkles;
                const isSelected = selected?.id === e.id;
                const isRunning = e.status === 'running';
                const isDeletingRow = deleteAnim?.item.id === e.id;
                const isExitingRow = isDeletingRow && deleteAnim.exiting;
                return (
                  <div
                    key={e.id}
                    className={`${styles.row} ${isSelected ? styles.selected : ''} ${isRunning ? `${styles['row--running']} ${styles['row--unselectable']}` : ''} ${isDeletingRow && !isExitingRow ? `${styles['row--deleting']} ${styles['row--unselectable']}` : ''} ${isExitingRow ? styles['row--exiting'] : ''}`}
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
                      {isRunning ? (
                        <button
                          type="button"
                          className={styles['row__stop-btn']}
                          onClick={(evt) => requestConfirm(evt, e, 'cancel')}
                          disabled={cancelingId === e.id}
                          title="Stop this evaluation"
                        >
                          {cancelingId === e.id ? <Loader2 size={11} className={styles.spin} /> : <StopCircle size={11} />}
                          {cancelingId === e.id ? 'Stopping…' : 'Stop'}
                        </button>
                      ) : (
                        <button
                          type="button"
                          className={styles['row__delete-btn']}
                          onClick={(evt) => requestConfirm(evt, e, 'delete')}
                          disabled={deletingId === e.id}
                          title="Delete this evaluation"
                        >
                          {deletingId === e.id ? <Loader2 size={12} className={styles.spin} /> : <Trash2 size={12} />}
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

            {/* Pagination bar — only worth showing once there's more than
                one page, or the page-size control is worth exposing even at
                exactly one page so it's discoverable before it's needed. */}
            {total > 0 && (
              <div className={styles.pagination}>
                <div className={styles['pagination__info']}>
                  {rangeStart}–{rangeEnd} of {total}
                </div>
                <div className={styles['pagination__controls']}>
                  <select
                    className={styles['pagination__size-select']}
                    value={pageSizeState}
                    onChange={(e) => changePageSize(Number(e.target.value))}
                    title="Rows per page"
                  >
                    {PAGE_SIZE_OPTIONS.map((n) => (
                      <option key={n} value={n}>
                        {n} / page
                      </option>
                    ))}
                  </select>
                  <button
                    type="button"
                    className={styles['pagination__nav-btn']}
                    onClick={() => setPage((p) => Math.max(1, p - 1))}
                    disabled={pageState <= 1 || listStatus === 'loading'}
                    title="Previous page"
                  >
                    <ChevronLeft size={14} />
                  </button>
                  <span className={styles['pagination__page-label']}>
                    {pageState} / {totalPages}
                  </span>
                  <button
                    type="button"
                    className={styles['pagination__nav-btn']}
                    onClick={() => setPage((p) => Math.min(totalPages, p + 1))}
                    disabled={pageState >= totalPages || listStatus === 'loading'}
                    title="Next page"
                  >
                    <ChevronRight size={14} />
                  </button>
                </div>
              </div>
            )}
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

      {/* ---------- Stop / delete confirm dialog ---------- */}
      {confirmTarget && (
        <div className={styles['confirm-overlay']} onClick={() => setConfirmTarget(null)}>
          <div className={styles['confirm-dialog']} role="alertdialog" aria-modal="true" onClick={(evt) => evt.stopPropagation()}>
            <div className={styles['confirm-dialog__icon']}>
              <AlertTriangle size={20} />
            </div>
            {confirmTarget.action === 'cancel' ? (
              <>
                <h3 className={styles['confirm-dialog__title']}>Stop this evaluation?</h3>
                <p className={styles['confirm-dialog__body']}>
                  <strong>{confirmTarget.evaluation.name || 'This evaluation'}</strong> is still running. Stopping it
                  now will end the run early — progress made so far won't be recoverable.
                </p>
                <div className={styles['confirm-dialog__actions']}>
                  <button type="button" className={styles['confirm-dialog__btn--ghost']} onClick={() => setConfirmTarget(null)}>
                    Keep running
                  </button>
                  <button type="button" className={styles['confirm-dialog__btn--danger']} onClick={runConfirmedAction}>
                    <StopCircle size={14} />
                    Stop evaluation
                  </button>
                </div>
              </>
            ) : (
              <>
                <h3 className={styles['confirm-dialog__title']}>Delete this evaluation?</h3>
                <p className={styles['confirm-dialog__body']}>
                  <strong>{confirmTarget.evaluation.name || 'This evaluation'}</strong> and its results will be
                  permanently deleted. This can't be undone.
                </p>
                <div className={styles['confirm-dialog__actions']}>
                  <button type="button" className={styles['confirm-dialog__btn--ghost']} onClick={() => setConfirmTarget(null)}>
                    Cancel
                  </button>
                  <button type="button" className={styles['confirm-dialog__btn--danger']} onClick={runConfirmedAction}>
                    <Trash2 size={14} />
                    Delete evaluation
                  </button>
                </div>
              </>
            )}
          </div>
        </div>
      )}
    </div>
  );
}


























//History.module.scss
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
@keyframes history-row-delete-pulse {
  0%, 100% { box-shadow: 0 0 0 0 rgba($danger, 0.18); }
  50% { box-shadow: 0 0 0 5px rgba($danger, 0.08); }
}
@keyframes history-row-delete-sheen {
  0% { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}

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

// ---- pagination bar (bottom of sidebar) ------------------------------------
.pagination {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  flex-wrap: wrap;
  margin-top: 12px;
  padding-top: 12px;
  border-top: 1px solid $line;
}

.pagination__info {
  font-family: $font-mono;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  color: $ink-3;
  white-space: nowrap;
}

.pagination__controls {
  display: flex;
  align-items: center;
  gap: 6px;
}

.pagination__size-select {
  appearance: none;
  -webkit-appearance: none;
  font: inherit;
  font-family: $font-mono;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  font-weight: 650;
  color: $ink-2;
  background: $paper;
  border: 1px solid $line;
  border-radius: 8px;
  padding: 5px 22px 5px 9px;
  cursor: pointer;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='10' height='6' viewBox='0 0 10 6' fill='none'%3E%3Cpath d='M1 1L5 5L9 1' stroke='%238A909B' stroke-width='1.5' stroke-linecap='round' stroke-linejoin='round'/%3E%3C/svg%3E");
  background-repeat: no-repeat;
  background-position: right 8px center;

  &:hover { border-color: $ink-3; }
  &:focus { outline: none; border-color: $signal; box-shadow: 0 0 0 3px $wash; }
}

.pagination__nav-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 26px;
  height: 26px;
  border-radius: 8px;
  border: 1px solid $line;
  background: $paper;
  color: $ink-2;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

  &:hover:not(:disabled) { border-color: $signal; color: $signal; background: $wash; }

  &:disabled {
    opacity: 0.4;
    cursor: not-allowed;
  }
}

.pagination__page-label {
  min-width: 44px;
  text-align: center;
  font-family: $font-mono;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  font-weight: 700;
  color: $ink;
}

.row {
  position: relative;
  border: 1.5px solid $line;
  border-radius: 14px;
  padding: 14px;
  cursor: pointer;
  background: $card;
  overflow: hidden;
  // A concrete max-height (comfortably above real card content, ~130-160px)
  // is required for the collapse transition in .row--exiting below to
  // animate at all — CSS can't transition to/from `auto`.
  max-height: 260px;
  transition: border-color 0.15s ease, box-shadow 0.15s ease, transform 0.15s ease, background 0.15s ease,
    max-height 0.32s ease, opacity 0.28s ease, padding 0.32s ease, margin 0.32s ease;
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

// Running/deleting cards aren't clickable — the Stop/Delete confirm flow is
// the only interactive element on them, so the card itself shouldn't hint
// at being clickable.
.row--unselectable {
  cursor: default;

  &:hover { transform: none; }
}

// Delete in flight: a red pulsing border + light sheen sweep, same pattern
// as the running-state animation above but in the danger color, so it
// visually reads as "something destructive is happening" rather than
// "in progress" — while the DELETE request is out.
.row--deleting {
  position: relative;
  border-color: rgba($danger, 0.35);
  animation: history-row-delete-pulse 1.2s ease-in-out infinite;

  &::after {
    content: '';
    position: absolute;
    inset: 0;
    background: linear-gradient(100deg, transparent 30%, rgba($danger, 0.1) 50%, transparent 70%);
    background-size: 200% 100%;
    animation: history-row-delete-sheen 1.2s ease-in-out infinite;
    pointer-events: none;
  }

  // Dim everything except the delete button itself (which shows its own
  // spinner) so attention stays on the destructive action in progress.
  > *:not(.row__badges) {
    opacity: 0.55;
    transition: opacity 0.2s ease;
  }
}

// Delete succeeded: collapse the card away instead of having it vanish
// the instant Redux removes it from the list — opacity + scale + the
// max-height/padding collapse declared on .row above animate together.
.row--exiting {
  opacity: 0;
  transform: scale(0.97);
  max-height: 0;
  padding-top: 0;
  padding-bottom: 0;
  border-width: 0;
  pointer-events: none;
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

// Icon-only, ghost by default — deliberately quieter than the Stop pill
// since delete is available on every non-running card and shouldn't compete
// visually with the status badge next to it. Only turns red on hover/focus.
.row__delete-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  margin-left: auto; // pushes it to the far right of the badges row
  width: 22px;
  height: 22px;
  border-radius: 7px;
  border: 1px solid transparent;
  background: transparent;
  color: $ink-3;
  cursor: pointer;
  transition: border-color 0.15s ease, background 0.15s ease, color 0.15s ease, transform 0.15s ease;

  &:hover:not(:disabled) {
    border-color: rgba($danger, 0.3);
    background: $danger-wash;
    color: $danger;
  }

  &:active:not(:disabled) { transform: scale(0.94); }

  &:disabled {
    opacity: 0.5;
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






















//Evaluations.ts
import api from '../axiosInstance';
import type {
  AgentBenchmarkRunMultiRequest,
  AgentBenchmarkRunRequest,
  CreateEvaluationRequest,
  CreateEvaluationResponse,
  DatasetPreviewResponse,
  EvaluationsListResponse,
  EvaluationResultsResponse,
  EvaluationListItem,
  ModelResult,
  GenerateInstructionRequest,
  GenerateInstructionResponse,
} from '../../types';

// Re-exported for convenience so existing imports of these two request
// types from this module (e.g. in evaluationsSlice.ts) keep working —
// the canonical definitions now live in ../../types.
export type { AgentBenchmarkRunRequest, AgentBenchmarkRunMultiRequest };

// Normalizes one list-item so array fields the UI iterates over
// (model_ids.length, selected_metrics.map, etc.) are never null/undefined,
// even if the backend omits them for a given row. Same normalize-at-the-
// boundary pattern as benchmarksApi.list's `tasks`.
function normalizeListItem(e: EvaluationListItem): EvaluationListItem {
  return {
    ...e,
    model_ids: e.model_ids || [],
    selected_metrics: e.selected_metrics || [],
    selected_category: e.selected_category || [],
    datasets_config: e.datasets_config || [],
  };
}

export interface ListEvaluationsParams {
  offset: number;
  limit: number;
  // Omit (leave undefined) rather than passing 'All' — axios drops
  // undefined params from the query string automatically, which is how we
  // satisfy "don't send status/eval_type at all when no filter is applied".
  status?: string;
  eval_type?: string;
}

export const evaluationsApi = {
  // Populates the History sidebar list. Called on mount, on page/page-size/
  // filter changes, and every 10s (silent poll) — see History.tsx. Backed by
  // GET /evaluations?status=&eval_type=&offset=&limit= with a { evaluations,
  // total } response; `total` is the full filtered count across all pages,
  // not just this page's length, and drives the pagination bar.
  list: (params: ListEvaluationsParams) =>
    api.get<EvaluationsListResponse>('/evaluations', { params }).then((r) => ({
      evaluations: (r.data.evaluations || []).map(normalizeListItem),
      total: r.data.total ?? 0,
    })),

  create: (payload: CreateEvaluationRequest) =>
    api.post<CreateEvaluationResponse>('/evaluations', payload).then((r) => r.data),

  start: (evaluationId: string) =>
    api.post<void>(`/evaluations/${evaluationId}/start`).then(() => undefined),

  // Stops a running evaluation — used by the "Stop evaluation" button on a
  // running card in History.tsx (behind a confirm dialog). The backend
  // responds 200 with { status: 'cancelled', evaluation_id } on success.
  cancel: (evaluationId: string) =>
    api.post<{ status: string; evaluation_id: string }>(`/evaluations/${evaluationId}/cancel`).then((r) => r.data),

  // Deletes an evaluation outright — used by the "Delete" action on a
  // non-running card in History.tsx (behind a confirm dialog). The backend
  // responds 200 with { status: 'deleted', evaluation_id } on success.
  remove: (evaluationId: string) =>
    api.delete<{ status: string; evaluation_id: string }>(`/evaluations/${evaluationId}`).then((r) => r.data),

  // Only ever called when the selected evaluation's status === 'completed'.
  // The backend returns 400 with { detail: "Execution not completed." } if
  // called too early — callers should surface err.response.data.detail.
  //
  // Also normalizes at the boundary: `total_test` (singular, as sent by the
  // API) -> `total_tests`; and `results`/`metric_scores`/`details`/
  // `selected_metrics` default to []/{} when the backend omits them, so
  // downstream code can rely on them always being iterable.
  results: (evaluationId: string) =>
    api.get<EvaluationResultsResponse>(`/evaluations/${evaluationId}/results`).then((r) => {
      const data = r.data;
      return {
        ...data,
        selected_metrics: data.selected_metrics || [],
        results: (data.results || []).map((m) => {
          const raw = m as unknown as ModelResult & { total_test?: number };
          return {
            ...raw,
            total_tests: raw.total_tests ?? raw.total_test ?? 0,
            metric_scores: raw.metric_scores || {},
            details: raw.details || [],
          };
        }),
      };
    }),

  // Convenience helper used by the wizard's "Start Evaluation" (step 7):
  // create, then immediately start. Only for draft.type 'model' | 'rag'.
  createAndStart: async (payload: CreateEvaluationRequest) => {
    const created = await evaluationsApi.create(payload);
    const id = created.id || created.evaluation_id;
    if (!id) {
      throw new Error('Evaluation was created but no id was returned by the server.');
    }
    await evaluationsApi.start(id);
    return id;
  },

  // POST /agent-benchmark/run — draft.type === 'agent', no framework selected.
  // 200 OK response means successful submission; no meaningful body is relied upon.
  runAgentBenchmark: (payload: AgentBenchmarkRunRequest) =>
    api.post<void>('/agent-benchmark/run', payload).then(() => undefined),

  // POST /agent-benchmark/run-multi — draft.type === 'agent', framework selected.
  // 200 OK response means successful submission; no meaningful body is relied upon.
  runAgentBenchmarkMulti: (payload: AgentBenchmarkRunMultiRequest) =>
    api.post<void>('/agent-benchmark/run-multi', payload).then(() => undefined),

  // GET /datasets/{id}/preview?limit={limit}&offset={offset} — lives here
  // (not in the datasets API module) since it's only ever used from the
  // evaluation wizard's Test Suite step preview slider. Paginated, 20
  // questions per page by default (limit=20, offset starts at 0). Total
  // page count is derived on the caller's side from the dataset's own
  // `question_count` (from the /datasets list), not from this response.
  previewDataset: (datasetId: string, limit: number, offset: number) =>
    api
      .get<DatasetPreviewResponse>(`/datasets/${datasetId}/preview`, { params: { limit, offset } })
      .then((r) => r.data),

  // POST /evaluations/generate-instruction — Metrics step's "Generate
  // Instruction" button. See NewEvaluation.tsx `generateInstruction` for
  // how model_id/eval_type/questions are assembled.
  generateInstruction: (payload: GenerateInstructionRequest) =>
    api.post<GenerateInstructionResponse>('/evaluations/generate-instruction', payload).then((r) => r.data),
};




















//Evaluationsslice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import type { PayloadAction } from '@reduxjs/toolkit';
import { evaluationsApi } from '../../api/endpoints/evaluations';
import type { AgentBenchmarkRunMultiRequest, AgentBenchmarkRunRequest } from '../../api/endpoints/evaluations';
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
  //
  // Server-paginated as of the offset/limit + status/eval_type filter API:
  // `list` holds only the *current page's* rows, `total` is the full
  // filtered count across all pages (drives History's pagination bar), and
  // `page`/`pageSize` echo back whatever the most recent fetchEvaluations
  // call requested.
  list: EvaluationListItem[];
  total: number;
  page: number;
  pageSize: number;
  listStatus: AsyncStatus;
  listError: string | null;

  // Per-evaluation results (GET /evaluations/{id}/results), fetched lazily
  // and only once status === 'completed' (spec §2.3).
  resultsByEvalId: Record<string, EvaluationResultsResponse>;
  resultsStatusByEvalId: Record<string, AsyncStatus>;
  resultsErrorByEvalId: Record<string, string | null>;

  launching: boolean;
  launchError: string | null;

  // Cancel-in-flight tracking for the "Stop evaluation" action on a running
  // card in History.tsx (confirm dialog -> POST /evaluations/{id}/cancel).
  cancelingId: string | null;
  cancelError: string | null;

  // Delete-in-flight tracking for the "Delete" action on a non-running card
  // in History.tsx (confirm dialog -> DELETE /evaluations/{id}).
  deletingId: string | null;
  deleteError: string | null;
}

const initialDraft: EvaluationDraft = {
  name: '',
  type: null,
  providers: [],
  models: [],
  retryConfigMode: 'individual',
  retryConfigAll: { max_retries: 1, timeout: 60 },
  modelRetryConfig: {},
  dataset: null,
  subgroup: [],
  runSamplesMode: 'custom',
  runSamples: 10,
  metrics: [],
  judgeModelId: null,
  agentFramework: null,
  topK: 5,
  instruction: '',
  retestOnWrong: false,
  retestMaxRounds: 3,
  retestVerifyMetric: null,
};

const initialState: EvaluationsState = {
  draft: initialDraft,
  list: [],
  total: 0,
  page: 1,
  pageSize: 20,
  listStatus: 'idle',
  listError: null,
  resultsByEvalId: {},
  resultsStatusByEvalId: {},
  resultsErrorByEvalId: {},
  launching: false,
  launchError: null,
  cancelingId: null,
  cancelError: null,
  deletingId: null,
  deleteError: null,
};

// Args for fetchEvaluations — `status`/`evalType` of 'All' (or omitted) mean
// "no filter", which the thunk translates into leaving those query params
// off the request entirely (see evaluationsApi.list). `silent: true` is used
// for the 10s background poll so it doesn't flip listStatus back to
// 'loading' over data that's already on screen (spec §2.4) — explicit page/
// page-size/filter changes should still show the loading state, since the
// content is genuinely about to change.
export interface FetchEvaluationsArgs {
  page: number;
  pageSize: number;
  status?: string;
  evalType?: string;
  silent?: boolean;
}

export const fetchEvaluations = createAsyncThunk('evaluations/fetchList', async (args: FetchEvaluationsArgs) => {
  const offset = (args.page - 1) * args.pageSize;
  const result = await evaluationsApi.list({
    offset,
    limit: args.pageSize,
    status: args.status && args.status !== 'All' ? args.status : undefined,
    eval_type: args.evalType && args.evalType !== 'All' ? args.evalType : undefined,
  });
  return { ...result, page: args.page, pageSize: args.pageSize };
});


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

// Shared across all three launch thunks below, cancelEvaluation,
// deleteEvaluation, and fetchEvaluationResults above — the backend's error
// body on 4xx responses is { detail: string }, so that's what should end up
// in state, not axios's generic "Request failed with status code 400".
function extractErrorDetail(err: unknown, fallback: string): string {
  return (
    (err as { response?: { data?: { detail?: string } } })?.response?.data?.detail ||
    (err as Error)?.message ||
    fallback
  );
}

// POST /evaluations then /evaluations/{id}/start — draft.type 'model' | 'rag'.
export const launchEvaluation = createAsyncThunk(
  'evaluations/launch',
  async (payload: CreateEvaluationRequest, { rejectWithValue }) => {
    try {
      return await evaluationsApi.createAndStart(payload);
    } catch (err) {
      return rejectWithValue(extractErrorDetail(err, 'Failed to launch evaluation'));
    }
  }
);

// POST /agent-benchmark/run — draft.type 'agent', no agentFramework selected.
export const runAgentBenchmark = createAsyncThunk(
  'evaluations/runAgentBenchmark',
  async (payload: AgentBenchmarkRunRequest, { rejectWithValue }) => {
    try {
      return await evaluationsApi.runAgentBenchmark(payload);
    } catch (err) {
      return rejectWithValue(extractErrorDetail(err, 'Failed to launch agent benchmark'));
    }
  }
);

// POST /agent-benchmark/run-multi — draft.type 'agent', agentFramework selected.
export const runAgentBenchmarkMulti = createAsyncThunk(
  'evaluations/runAgentBenchmarkMulti',
  async (payload: AgentBenchmarkRunMultiRequest, { rejectWithValue }) => {
    try {
      return await evaluationsApi.runAgentBenchmarkMulti(payload);
    } catch (err) {
      return rejectWithValue(extractErrorDetail(err, 'Failed to launch agent benchmark'));
    }
  }
);

// POST /evaluations/{id}/cancel — "Stop evaluation" button on a running
// card in History.tsx, behind a confirm dialog. `evaluationId` is threaded
// through action.meta.arg (createAsyncThunk's default), so the reducers
// below can key cancelingId/list updates off it without it being part of
// the rejected payload.
export const cancelEvaluation = createAsyncThunk(
  'evaluations/cancel',
  async (evaluationId: string, { rejectWithValue }) => {
    try {
      return await evaluationsApi.cancel(evaluationId);
    } catch (err) {
      return rejectWithValue(extractErrorDetail(err, 'Failed to stop evaluation'));
    }
  }
);

// DELETE /evaluations/{id} — "Delete" action on a non-running card in
// History.tsx, behind a confirm dialog. Same evaluationId-via-meta.arg
// pattern as cancelEvaluation above.
export const deleteEvaluation = createAsyncThunk(
  'evaluations/delete',
  async (evaluationId: string, { rejectWithValue }) => {
    try {
      return await evaluationsApi.remove(evaluationId);
    } catch (err) {
      return rejectWithValue(extractErrorDetail(err, 'Failed to delete evaluation'));
    }
  }
);

const evaluationsSlice = createSlice({
  name: 'evaluations',
  initialState,
  reducers: {
    setDraft(state, action: PayloadAction<Partial<EvaluationDraft>>) {
      state.draft = { ...state.draft, ...action.payload };
    },
    // Step 2: changing type invalidates everything chosen after it — the
    // available providers/models/datasets/metrics all depend on type, so
    // stale selections from a previous type must not silently carry over.
    // Providers and models in particular are toggle-based multi-select
    // (see NewEvaluation.tsx `toggle`), so without this reset, switching
    // from e.g. Model -> Agent -> Model again and picking a *different*
    // model each time would leave BOTH models checked (the old selection
    // was never cleared, only added to) — that's the bug this fixes.
    setDraftType(state, action: PayloadAction<EvaluationDraft['type']>) {
      state.draft.type = action.payload;
      state.draft.providers = [];
      state.draft.models = [];
      state.draft.retryConfigMode = 'individual';
      state.draft.retryConfigAll = { max_retries: 1, timeout: 60 };
      state.draft.modelRetryConfig = {};
      state.draft.dataset = null;
      state.draft.subgroup = [];
      state.draft.metrics = [];
      state.draft.judgeModelId = null;
      state.draft.runSamplesMode = 'custom';
      state.draft.runSamples = 10;
      state.draft.topK = 5;
      state.draft.instruction = '';
      state.draft.retestOnWrong = false;
      state.draft.retestMaxRounds = 3;
      state.draft.retestVerifyMetric = null;
      if (action.payload !== 'agent') {
        state.draft.agentFramework = null;
      }
    },
    resetDraft(state) {
      state.draft = initialDraft;
    },
    // Local-only removal, kept as a manual escape hatch for edge cases (e.g.
    // clearing a stale row the backend won't return anymore). The primary
    // delete path is now the deleteEvaluation thunk below, which calls
    // DELETE /evaluations/{id} and removes the row on success — this
    // reducer does not call the API and does not persist.
    removeEvaluationLocal(state, action: PayloadAction<string>) {
      state.list = state.list.filter((e) => e.id !== action.payload);
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchEvaluations.pending, (state, action) => {
        // Silent (background poll) calls never show the loading state over
        // data that's already on screen. Explicit calls (page/page-size/
        // filter changes, or the very first load) do — the content is
        // genuinely about to change, not just being silently refreshed.
        if (!action.meta.arg.silent || state.list.length === 0) {
          state.listStatus = 'loading';
        }
      })
      .addCase(fetchEvaluations.fulfilled, (state, action) => {
        state.listStatus = 'succeeded';
        state.listError = null;
        state.list = action.payload.evaluations;
        state.total = action.payload.total;
        state.page = action.payload.page;
        state.pageSize = action.payload.pageSize;
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

      // ---- launch: three thunks (Model/RAG, Agent-benchmark, Agent-multi) ---
      // all share the same launching/launchError flags and all clear the
      // draft on success, exactly like the original launchEvaluation did.
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
        state.launchError = (action.payload as string) || action.error.message || 'Failed to launch evaluation';
      })

      .addCase(runAgentBenchmark.pending, (state) => {
        state.launching = true;
        state.launchError = null;
      })
      .addCase(runAgentBenchmark.fulfilled, (state) => {
        state.launching = false;
        state.draft = initialDraft;
      })
      .addCase(runAgentBenchmark.rejected, (state, action) => {
        state.launching = false;
        state.launchError = (action.payload as string) || action.error.message || 'Failed to launch agent benchmark';
      })

      .addCase(runAgentBenchmarkMulti.pending, (state) => {
        state.launching = true;
        state.launchError = null;
      })
      .addCase(runAgentBenchmarkMulti.fulfilled, (state) => {
        state.launching = false;
        state.draft = initialDraft;
      })
      .addCase(runAgentBenchmarkMulti.rejected, (state, action) => {
        state.launching = false;
        state.launchError = (action.payload as string) || action.error.message || 'Failed to launch agent benchmark';
      })

      // ---- cancel (stop a running evaluation) --------------------------------
      .addCase(cancelEvaluation.pending, (state, action) => {
        state.cancelingId = action.meta.arg;
        state.cancelError = null;
      })
      .addCase(cancelEvaluation.fulfilled, (state, action) => {
        state.cancelingId = null;
        // Optimistically flip the row to 'canceled' right away rather than
        // waiting for the next 10s background poll to pick it up.
        const item = state.list.find((e) => e.id === action.payload.evaluation_id);
        if (item) item.status = 'canceled';
      })
      .addCase(cancelEvaluation.rejected, (state, action) => {
        state.cancelingId = null;
        state.cancelError = (action.payload as string) || action.error.message || 'Failed to stop evaluation';
      })

      // ---- delete ------------------------------------------------------------
      .addCase(deleteEvaluation.pending, (state, action) => {
        state.deletingId = action.meta.arg;
        state.deleteError = null;
      })
      .addCase(deleteEvaluation.fulfilled, (state, action) => {
        state.deletingId = null;
        state.list = state.list.filter((e) => e.id !== action.payload.evaluation_id);
      })
      .addCase(deleteEvaluation.rejected, (state, action) => {
        state.deletingId = null;
        state.deleteError = (action.payload as string) || action.error.message || 'Failed to delete evaluation';
      });
  },
});

export const { setDraft, setDraftType, resetDraft, removeEvaluationLocal } = evaluationsSlice.actions;
export default evaluationsSlice.reducer;

























//index.ts
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

// Nested summary of the report generated for this evaluation, if any.
// Only present once the backend has created a report row for the eval —
// absent/undefined while the eval is still pending/running with no report yet.
export interface EvaluationReportSummary {
  report_id: string;
  title: string;
  status: string;
  created_at: string;
}

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
  // Present once a report has been generated for this evaluation (spec: new
  // "download from History" requirement). When `report.report_id` is set,
  // History should offer the same download options as the Reports page.
  report?: EvaluationReportSummary | null;
}
export interface EvaluationsListResponse {
  evaluations: EvaluationListItem[];
  // Total row count across all pages (not just the current page's length) —
  // used to compute page count for History's pagination bar.
  total: number;
}

// ---------- Evaluations: results ----------
export interface TestDetail {
  task: string;
  input: string;
  expected_output: string;
  actual_output: string;
  passed: boolean;
}
export interface ModelResult {
  model_id: string;
  provider: string | null;
  rank: number;
  score: number;
  accuracy: number;
  passed_tests: number;
  failed_tests: number;
  // Normalized from the API's `total_test` (singular) at the fetch boundary
  // (evaluationsApi.results) — see benchmarksApi.list for the same pattern.
  total_tests: number;
  metric_scores: Record<string, number>;
  details: TestDetail[];
}
export interface EvaluationResultsResponse {
  evaluation_id: string;
  name: string;
  eval_type: string;
  dataset_id: string;
  benchmark: string;
  model_ids: string[];
  selected_metrics: string[];
  status: EvaluationStatusValue;
  total_questions: number;
  top_model: string;
  top_score: number;
  started_at: string | null;
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
