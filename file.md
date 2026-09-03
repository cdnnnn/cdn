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

// Shown in place of the per-model results table whenever there's nothing to
// show there yet — either because the evaluation hasn't finished (results
// fetched for every status now, not just 'completed'), or because a fetch
// for a not-yet-completed run came back empty/errored.
const RESULTS_PLACEHOLDER: Partial<Record<EvaluationStatusValue, string>> = {
  running: 'This evaluation is still running — results will appear once it completes.',
  pending: "This evaluation hasn't started yet.",
  failed: 'This evaluation failed to complete.',
  canceled: 'This evaluation was canceled.',
  completed: 'No results available.',
};

// Mirrors Reports.tsx — same base four formats, same download endpoint.
// HTML is agent-only (appended at render time — see the detail header
// download buttons below) since it's not a meaningful export for
// model/RAG evaluations.
const DOWNLOAD_OPTIONS: { format: ReportDownloadFormat; label: string }[] = [
  { format: 'json', label: 'JSON' },
  { format: 'csv', label: 'CSV' },
  { format: 'csv_detailed', label: 'CSV (Detailed)' },
  { format: 'pdf', label: 'PDF' },
];
const HTML_DOWNLOAD_OPTION: { format: ReportDownloadFormat; label: string } = { format: 'html', label: 'HTML' };

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
  //
  // Fetched for every status now, not just 'completed' — a pending/failed/
  // canceled evaluation still has a run configuration (dataset, benchmark,
  // metrics tested, metrics_config) worth showing even with no per-model
  // results yet. The render below falls back gracefully when `results` is
  // empty/absent for a given status.
  useEffect(() => {
    if (selected && !resultsByEvalId[selected.id]) {
      dispatch(fetchEvaluationResults(selected.id));
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [selected?.id]);

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
    if (clicked) {
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
                      same options/behavior as the Reports page. HTML is only
                      offered for agent-type evaluations. */}
                  {selected.report?.report_id && (
                    <div className={styles['detail-hdr__actions']}>
                      {[...DOWNLOAD_OPTIONS, ...(selected.eval_type === 'agent' ? [HTML_DOWNLOAD_OPTION] : [])].map((opt) => (
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

                {/* Results — fetched for every status now (not just
                    'completed'), so pending/running/failed/canceled runs
                    still surface benchmark/dataset/metrics-tested/started
                    via the meta-strip even with no per-model rows yet. */}
                {resultsStatus === 'loading' && !results && (
                  <div className={styles.empty}>
                    <Loader2 size={16} className={styles.spin} /> Loading evaluation details…
                  </div>
                )}
                {resultsStatus === 'failed' && !results && (
                  <div className={styles['status-message']}>
                    {selected.status === 'completed'
                      ? resultsError || 'Failed to load results.'
                      : RESULTS_PLACEHOLDER[selected.status] || 'No results available yet.'}
                  </div>
                )}
                {results && (
                  <>
                    {/* Run configuration (metrics_config) — comes from the
                        results fetch (GET /evaluations/{id}/results), not
                        the list item. Genuinely dynamic on the wire (can be
                        {}, a subset, or the full shape), so every piece
                        below renders only if that specific field is
                        actually present rather than assuming a fixed shape. */}
                    {results.metrics_config && Object.keys(results.metrics_config).length > 0 && (
                      <div className={styles['config-panel']}>
                        <div className={styles['config-panel__title']}>Run configuration</div>
                        <div className={styles['config-panel__grid']}>
                          {results.metrics_config.max_retries != null && (
                            <div className={styles['config-panel__item']}>
                              <span className={styles['config-panel__label']}>Max retries</span>
                              <span className={styles['config-panel__val']}>{results.metrics_config.max_retries}</span>
                            </div>
                          )}
                          {results.metrics_config.timeout != null && (
                            <div className={styles['config-panel__item']}>
                              <span className={styles['config-panel__label']}>Timeout</span>
                              <span className={styles['config-panel__val']}>{results.metrics_config.timeout}s</span>
                            </div>
                          )}
                          {results.metrics_config.retest_on_wrong != null && (
                            <div className={styles['config-panel__item']}>
                              <span className={styles['config-panel__label']}>Retest on wrong</span>
                              <span className={styles['config-panel__val']}>{results.metrics_config.retest_on_wrong ? 'Yes' : 'No'}</span>
                            </div>
                          )}
                          {results.metrics_config.retest_on_wrong && results.metrics_config.retest_max_rounds != null && (
                            <div className={styles['config-panel__item']}>
                              <span className={styles['config-panel__label']}>Retest max rounds</span>
                              <span className={styles['config-panel__val']}>{results.metrics_config.retest_max_rounds}</span>
                            </div>
                          )}
                          {results.metrics_config.retest_verify_metric && (
                            <div className={styles['config-panel__item']}>
                              <span className={styles['config-panel__label']}>Retest verify metric</span>
                              <span className={styles['config-panel__val']}>{results.metrics_config.retest_verify_metric}</span>
                            </div>
                          )}
                        </div>

                        {results.metrics_config.model_retry_config &&
                          Object.keys(results.metrics_config.model_retry_config).length > 0 && (
                            <div className={styles['config-panel__models']}>
                              <span className={styles['config-panel__label']}>Per-model retry overrides</span>
                              <div className={styles['config-panel__model-list']}>
                                {Object.entries(results.metrics_config.model_retry_config).map(([modelId, cfg]) => (
                                  <div key={modelId} className={styles['config-panel__model-row']}>
                                    <span className={styles['config-panel__model-name']}>{modelName(modelId)}</span>
                                    <span className={styles['config-panel__model-vals']}>
                                      {cfg?.max_retries != null && <span>Retries: {cfg.max_retries}</span>}
                                      {cfg?.timeout != null && <span>Timeout: {cfg.timeout}s</span>}
                                    </span>
                                  </div>
                                ))}
                              </div>
                            </div>
                          )}
                      </div>
                    )}

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

                    {(results.results || []).length > 0 ? (
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
                    ) : (
                      <div className={styles['status-message']}>
                        {RESULTS_PLACEHOLDER[selected.status] || 'No results available.'}
                      </div>
                    )}
                  </>
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

// ---------- Datasets (Test Suite step) ----------
// `dataset_type` distinguishes the built-in DeepEval suites from datasets a
// user uploaded themselves. Both are shown in the Test Suite grid; only
// 'custom' gets the "Custom" tag, and both are filterable via the
// All / Custom / Deepeval chip group in the step header (spec: Test Suite
// Change-1).
export type DatasetType = 'custom' | 'deepeval' | string;

export interface Dataset {
  id: string;
  name: string;
  description?: string;
  category: string;
  eval_type: string;
  dataset_type: DatasetType;
  question_count: number;
  dataset_categories: string[];
}
export interface DatasetsResponse {
  datasets: Dataset[];
}

// GET /datasets/{id}/preview?limit={limit}&offset={offset} — paginated,
// 20 per page by default. Powers the right-to-left preview slider on each
// dataset card.
//
// Two response shapes are both observed in practice for `input`/`expected`
// (and whether `choices`/`metadata`/`subgroup` are present at all depends
// on the dataset), so every field below is optional except `id` — the UI
// renders whichever subset actually shows up rather than assuming one
// fixed shape:
//   Shape A (e.g. multiple-choice): input.prompt, expected.answer, choices
//   Shape B (e.g. RAG/retrieval):   input.question/source/type,
//                                   expected.answer/doc_id/section_id,
//                                   metadata, subgroup
export interface DatasetPreviewQuestion {
  id: string;
  input?: {
    prompt?: string;
    question?: string;
    source?: string;
    type?: string;
    [key: string]: unknown;
  };
  expected?: {
    answer?: string;
    doc_id?: string;
    section_id?: number;
    [key: string]: unknown;
  };
  metadata?: Record<string, unknown>;
  category?: string;
  subgroup?: string;
  choices?: string[];
}
export interface DatasetPreviewResponse {
  dataset_id: string;
  questions: DatasetPreviewQuestion[];
}

// ---------- Metrics ----------
// GET /metrics?eval_type={type}
export interface CustomMetric {
  id: string;
  name: string;
  metrics_type: string;
  // When true, this specific custom metric needs a judge model to grade
  // it — surfaced as a small indicator on its chip. Doesn't gate whether
  // the Judge Model picker itself shows (that's now always shown/mandatory
  // regardless of metric selection — see NewEvaluation.tsx).
  required_judge: boolean;
  eval_types: string[];
  description: string;
}
export interface MetricsResponse {
  eval_type: string;
  all_metrics: string[];
  custom: CustomMetric[];
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
  // RAG-only — how many retrieved documents/chunks to consider per query.
  // Test Suite step shows this input only when draft.type === 'rag'
  // (default 5, 1–50); omitted entirely for Model/Agent.
  top_k?: number;
  // Free-text evaluation instruction — optional. Either typed by the user
  // directly or pre-filled via POST /evaluations/generate-instruction and
  // then edited. See GenerateInstructionRequest/Response below.
  instruction?: string;
}
export interface CreateEvaluationResponse {
  id?: string;
  evaluation_id?: string;
  [key: string]: unknown;
}

// POST /evaluations/generate-instruction — Metrics step's "Generate
// Instruction" button. `questions` is built from a 5-question dataset
// preview (see NewEvaluation.tsx `generateInstruction`), reusing the same
// `input`/`expected` shapes as DatasetPreviewQuestion.
export interface GenerateInstructionQuestion {
  input?: DatasetPreviewQuestion['input'];
  expected?: DatasetPreviewQuestion['expected'];
}
export interface GenerateInstructionRequest {
  model_id: string;
  eval_type: string;
  questions: GenerateInstructionQuestion[];
}
export interface GenerateInstructionResponse {
  instruction: string;
}

// ---------- Evaluations: agent-benchmark launch ----------
// draft.type === 'agent'. Which request shape is sent depends on whether
// an agent framework was chosen in Step 2 (see NewEvaluation.tsx `launch`):
//   no framework  -> POST /agent-benchmark/run       (AgentBenchmarkRunRequest)
//   framework set -> POST /agent-benchmark/run-multi (AgentBenchmarkRunMultiRequest)
export interface AgentBenchmarkRunRequest {
  dataset_id: string;
  model_ids: string[];
  evaluation_name: string;
  run_samples: number;
}
export interface AgentBenchmarkRunMultiRequest {
  dataset_id: string;
  model_ids: string[];
  evaluation_name: string;
  selected_metrics: string[];
  selected_categories: string[];
  run_samples: number;
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

// The retry/retest configuration the user submitted for this run (mirrors
// the wizard's retryConfigMode='all'/'individual' + retest fields — see
// EvaluationDraft below). Genuinely dynamic on the wire: the API may send
// `{}`, a subset of fields, or the full shape depending on what the run
// actually specified, so every field here is optional and the UI renders
// only whatever keys are actually present rather than assuming a fixed set.
export interface EvaluationMetricsConfig {
  max_retries?: number;
  timeout?: number;
  // Keyed by model id; each entry is itself partial for the same reason.
  model_retry_config?: Record<string, { max_retries?: number; timeout?: number }>;
  retest_on_wrong?: boolean;
  retest_max_rounds?: number;
  retest_verify_metric?: string | null;
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
  // added alongside GET /evaluations?status=&eval_type=&offset=&limit=
  // pagination support. Drives History's pagination bar (evaluationsSlice.ts
  // `total`/`page`/`pageSize`).
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
  // The run's retry/retest configuration at submission time — see
  // EvaluationMetricsConfig above for why every field is optional. May be
  // entirely absent, `{}`, or partially populated.
  metrics_config?: EvaluationMetricsConfig | null;
}

// Per-model retry override under retryConfigMode === 'individual' — see
// EvaluationDraft.modelRetryConfig below. Keyed by model id in the draft.
export interface ModelRetryConfig {
  max_retries: number;
  timeout: number;
}

// UI-only draft built up across the wizard's 7 steps (spec §6).
export interface EvaluationDraft {
  name: string;
  type: 'model' | 'agent' | 'rag' | null;
  providers: string[];
  models: string[];
  // 'individual': each selected model can have its own max_retries/timeout,
  // stored per model id in modelRetryConfig. 'all': a single retryConfigAll
  // applies to every selected model. Default 'individual'.
  retryConfigMode: 'individual' | 'all';
  // Used when retryConfigMode === 'all'.
  retryConfigAll: ModelRetryConfig;
  // Used when retryConfigMode === 'individual' — keyed by model id; a model
  // with no entry yet falls back to sensible defaults in the UI.
  modelRetryConfig: Record<string, ModelRetryConfig>;
  dataset: string | null;
  subgroup: string[];
  // 'custom': runSamples is a user-entered count, sent as-is.
  // 'full': the whole dataset is used — runSamples is sent as 0 regardless
  // of the last custom value entered (see NewEvaluation.tsx `launch`).
  runSamplesMode: 'custom' | 'full'; // default 'custom'
  runSamples: number; // default 10 — only meaningful when runSamplesMode === 'custom'
  metrics: string[];
  judgeModelId: string | null;
  // judgeApiKey intentionally omitted — no longer collected (spec §1.4)
  agentFramework: string | null;
  // RAG-only — see CreateEvaluationRequest.top_k. Only meaningful (and
  // only shown in the UI) when type === 'rag'.
  topK: number; // default 5, 1–50
  // Optional free-text evaluation instruction (Metrics step). Either
  // typed directly or pre-filled via the "Generate Instruction" button
  // and then edited — always editable either way.
  instruction: string; // default ''
  // When true, a question a model gets wrong is retried up to
  // retestMaxRounds additional times before being scored as failed.
  retestOnWrong: boolean; // default false
  retestMaxRounds: number; // default 3 — only meaningful when retestOnWrong
  // Which metric's pass/fail determines whether a retest is triggered;
  // null falls back to the run's primary/first selected metric.
  retestVerifyMetric: string | null; // default null
}
