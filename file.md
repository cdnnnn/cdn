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
















//Reports.ts
import api from '../axiosInstance';

export type ReportStatus = 'running' | 'completed' | 'failed';

export interface ReportListItem {
  id: string;
  title: string;
  evaluation_id: string;
  status: ReportStatus;
  file_path: string | null;
  top_model: string | null;
  top_score: number | null;
  eval_type: string;
  eval_name: string;
  metrics_tested: string[];
  created_at: string;
}

export interface ReportModelResult {
  model_id: string;
  rank: number;
  score: number;
  passed: number;
  failed: number;
  total: number;
  metrics: Record<string, unknown>;
}

export interface ReportDetail {
  id: string;
  title: string;
  evaluation_id: string;
  date: string;
  summary: string;
  topModel: string | null;
  verdict: string | null;
  metricsTested: string[];
  downloadSize: number | null;
  status: ReportStatus | 'pending';
  file_path: string | null;
  eval_type: string;
  eval_name: string;
  top_score: number | null;
  total_questions: number;
  passed_tests: number;
  failed_tests: number;
  models: ReportModelResult[];
  created_at: string;
}

export type ReportDownloadFormat = 'json' | 'csv' | 'csv_detailed' | 'pdf' | 'html';

const DOWNLOAD_MIME: Record<ReportDownloadFormat, string> = {
  json: 'application/json',
  csv: 'text/csv',
  csv_detailed: 'text/csv',
  pdf: 'application/pdf',
  html: 'text/html',
};

const DOWNLOAD_EXT: Record<ReportDownloadFormat, string> = {
  json: 'json',
  csv: 'csv',
  csv_detailed: 'csv',
  pdf: 'pdf',
  html: 'html',
};

export const reportsApi = {
  list: () => api.get<{ reports: ReportListItem[] }>('/reports').then((r) => r.data.reports),

  getById: (reportId: string) =>
    api.get<ReportDetail>(`/reports/${reportId}`).then((r) => r.data),

  // Downloads go through the shared axios instance (so the auth interceptor
  // still attaches the bearer token) as a blob, then get saved client-side
  // via an object URL — a plain <a href> to the API wouldn't carry auth.
  download: async (
    reportId: string,
    format: ReportDownloadFormat,
    filenameHint = reportId
  ) => {
    const response = await api.get(`/reports/${reportId}/download`, {
      params: { format },
      responseType: 'blob',
    });
    const blob = new Blob([response.data], { type: DOWNLOAD_MIME[format] });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `${filenameHint}.${DOWNLOAD_EXT[format]}`;
    document.body.appendChild(a);
    a.click();
    a.remove();
    URL.revokeObjectURL(url);
  },
};

















//Reports.tsx
import { useEffect, useMemo, useRef, useState } from 'react';
import { useSearchParams } from 'react-router-dom';
import {
  Search, FileText, Loader2, Download, Award, ListChecks, Clock,
  FileBarChart, SlidersHorizontal, X,
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchReports, fetchReportDetail, downloadReport } from '../../store/slices/reportsSlice';
import type { ReportDownloadFormat } from '../../api/endpoints/reports';
import styles from './Reports.module.scss';

// Base four formats, always offered. HTML is agent-only (appended at
// render time below) since it's not a meaningful export for model/RAG
// evaluations — mirrors the same rule in History.tsx.
const DOWNLOAD_OPTIONS: { format: ReportDownloadFormat; label: string }[] = [
  { format: 'json', label: 'JSON' },
  { format: 'csv', label: 'CSV' },
  { format: 'csv_detailed', label: 'CSV (Detailed)' },
  { format: 'pdf', label: 'PDF' },
];
const HTML_DOWNLOAD_OPTION: { format: ReportDownloadFormat; label: string } = { format: 'html', label: 'HTML' };

// Maps a raw status to the module status-pill variant suffix (mirrors History).
function statusVariant(status: string): string {
  switch (status) {
    case 'completed': return 'completed';
    case 'running': return 'running';
    case 'pending': return 'pending';
    case 'failed': return 'failed';
    default: return 'pending';
  }
}

export default function Reports() {
  const dispatch = useAppDispatch();
  const [searchParams, setSearchParams] = useSearchParams();
  const selectedId = searchParams.get('id');

  const { list, listStatus, listError, detailById, detailStatusById, detailErrorById, downloadingId } =
    useAppSelector((s) => s.reports);

  const [search, setSearch] = useState('');
  const [typeFilter, setTypeFilter] = useState('All');
  const [activeFilter, setActiveFilter] = useState<'search' | 'type' | null>(null);
  const searchInputRef = useRef<HTMLInputElement>(null);

  const toggleFilter = (key: 'search' | 'type') => {
    setActiveFilter((prev) => (prev === key ? null : key));
  };

  useEffect(() => {
    if (activeFilter === 'search') searchInputRef.current?.focus();
  }, [activeFilter]);

  useEffect(() => {
    dispatch(fetchReports());
  }, [dispatch]);

  const types = useMemo(() => ['All', ...new Set(list.map((r) => r.eval_type))], [list]);

  const filtered = useMemo(() => {
    return list.filter((r) => {
      if (search && !r.title.toLowerCase().includes(search.toLowerCase())) return false;
      if (typeFilter !== 'All' && r.eval_type !== typeFilter) return false;
      return true;
    });
  }, [list, search, typeFilter]);

  const selected = list.find((r) => r.id === selectedId) || filtered[0] || null;

  useEffect(() => {
    if (selected && !detailById[selected.id] && detailStatusById[selected.id] !== 'loading') {
      dispatch(fetchReportDetail(selected.id));
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [selected?.id]);

  const selectRow = (id: string) => setSearchParams({ id });

  const detail = selected ? detailById[selected.id] : undefined;
  const detailStatus = selected ? detailStatusById[selected.id] : undefined;
  const detailError = selected ? detailErrorById[selected.id] : undefined;

  // Base formats plus HTML when this report's evaluation is agent-type —
  // same rule History.tsx applies to its own download buttons.
  const downloadOptions = selected?.eval_type === 'agent'
    ? [...DOWNLOAD_OPTIONS, HTML_DOWNLOAD_OPTION]
    : DOWNLOAD_OPTIONS;

  const StatusBadge = ({ status }: { status: string }) => (
    <span className={`${styles.status} ${styles[`status--${statusVariant(status)}`]}`}>
      {status === 'running' && <span className={styles['live-dot']} />}
      {status}
    </span>
  );

  return (
    <div className="page-enter pg-shell">
      <div className={styles['reports__header']}>
        <div>
          <p className={styles['reports__header-eyebrow']}>Output</p>
          <h1>Reports</h1>
          <p className={styles['reports__header-sub']}>Generated reports for completed and in-progress evaluations</p>
        </div>
        <div className={styles['reports__header-meta']}>
          <FileBarChart size={13} />
          {list.length} report{list.length === 1 ? '' : 's'} generated
        </div>
      </div>

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

                <div className={styles['filter-toolbar__summary']}>
                  {search && (
                    <span className={styles['filter-chip']}>
                      <span>“{search}”</span>
                      <X size={11} onClick={() => setSearch('')} />
                    </span>
                  )}
                  {typeFilter !== 'All' && (
                    <span className={styles['filter-chip']}>
                      <span>{typeFilter}</span>
                      <X size={11} onClick={() => setTypeFilter('All')} />
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
                        placeholder="Search reports…"
                        value={search}
                        onChange={(e) => setSearch(e.target.value)}
                      />
                    </div>
                  </div>
                )}
                {activeFilter === 'type' && (
                  <div>
                    <div className={styles['panel-pills']}>
                      {types.map((t) => (
                        <button
                          key={t}
                          className={`${styles['panel-pill']} ${typeFilter === t ? styles.on : ''}`}
                          onClick={() => {
                            setTypeFilter(t);
                            setActiveFilter(null);
                          }}
                        >
                          {t}
                        </button>
                      ))}
                    </div>
                  </div>
                )}
              </div>

              {listStatus === 'loading' && list.length === 0 && (
                <div className={styles.empty}><Loader2 size={16} className={styles.spin} /> Loading reports…</div>
              )}
              {listStatus === 'failed' && list.length === 0 && (
                <div className={styles.empty}>{listError || 'Failed to load reports.'}</div>
              )}
              {listStatus !== 'loading' && filtered.length === 0 && list.length > 0 && (
                <div className={styles.empty}>No reports match your filters.</div>
              )}
            </div>

            <div className={styles.rows}>
              {filtered.map((r) => {
                const isSelected = selected?.id === r.id;
                return (
                  <div
                    key={r.id}
                    className={`${styles.row} ${isSelected ? styles.selected : ''}`}
                    onClick={() => selectRow(r.id)}
                  >
                    <div className={styles.row__top}>
                      <div className={styles.row__icon}><FileText size={16} /></div>
                      <div className={styles.row__name}>{r.title}</div>
                    </div>
                    <div className={styles.row__badges}>
                      <span className={styles['type-tag']}>{r.eval_type}</span>
                      <StatusBadge status={r.status} />
                    </div>
                    <div className={styles.row__meta}>{new Date(r.created_at).toLocaleDateString()}</div>
                    <div className={styles.row__stats}>
                      <span>{r.top_model ? `🏆 ${r.top_model}` : '—'}</span>
                      <span>{r.top_score != null ? `${Math.round(r.top_score * 100)}%` : '—'}</span>
                    </div>
                  </div>
                );
              })}
            </div>
          </div>

          {/* ---------- Detail panel ---------- */}
          <div className={styles.detail}>
            {!selected ? (
              <div className={styles['detail-empty']}>Select a report to see its details.</div>
            ) : (
              <>
                <div className={styles['detail-hdr']}>
                  <div>
                    <div className={styles['detail-hdr__badges']}>
                      <span className={styles['type-tag']}>{selected.eval_type}</span>
                      <StatusBadge status={selected.status} />
                    </div>
                    <h2 className={styles['detail-hdr__name']}>{selected.title}</h2>
                    <div className={styles['detail-hdr__date']}>Created {new Date(selected.created_at).toLocaleString()}</div>
                  </div>
                  <div className={styles['detail-hdr__actions']}>
                    {downloadOptions.map((opt) => (
                      <button
                        key={opt.format}
                        className={styles['dl-btn']}
                        disabled={selected.status !== 'completed' || downloadingId === selected.id}
                        onClick={() => dispatch(downloadReport({ reportId: selected.id, format: opt.format, filenameHint: selected.title }))}
                      >
                        {downloadingId === selected.id ? <Loader2 size={12} className={styles.spin} /> : <Download size={12} />}
                        {opt.label}
                      </button>
                    ))}
                  </div>
                </div>

                {detailStatus === 'loading' && (
                  <div className={styles.empty}><Loader2 size={16} className={styles.spin} /> Loading report…</div>
                )}
                {detailStatus === 'failed' && <div className={styles.empty}>{detailError}</div>}

                {detail && (
                  <>
                    <div className={styles['summary-cards']}>
                      <div className={styles['summary-card']}>
                        <span className={`${styles['summary-card__icon']} ${styles['summary-card__icon--win']}`}>
                          <Award size={16} />
                        </span>
                        <div>
                          <div className={styles['summary-card__label']}>Top Model</div>
                          <div className={styles['summary-card__val']}>
                            {detail.topModel || '—'}
                            {detail.top_score != null ? ` · ${Math.round(detail.top_score * 100)}%` : ''}
                          </div>
                        </div>
                      </div>
                      <div className={styles['summary-card']}>
                        <span className={`${styles['summary-card__icon']} ${styles['summary-card__icon--info']}`}>
                          <ListChecks size={16} />
                        </span>
                        <div>
                          <div className={styles['summary-card__label']}>Questions</div>
                          <div className={styles['summary-card__val']}>
                            {detail.total_questions.toLocaleString()} &middot; {detail.passed_tests} passed &middot; {detail.failed_tests} failed
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
                            {detail.status}{detail.date ? ` · ${detail.date}` : ''}
                          </div>
                        </div>
                      </div>
                    </div>

                    {detail.summary && <div className={styles['summary-text']}>{detail.summary}</div>}

                    {detail.models.length > 0 ? (
                      <div className={styles.results}>
                        <table className={styles['results-table']}>
                          <thead>
                            <tr>
                              <th>Rank</th>
                              <th>Model</th>
                              <th>Score</th>
                              <th>Passed</th>
                              <th>Failed</th>
                              <th>Total</th>
                            </tr>
                          </thead>
                          <tbody>
                            {detail.models.map((m) => (
                              <tr key={m.model_id} className={m.rank === 1 ? styles.winner : ''}>
                                <td className={styles['cell-rank']}>{m.rank === 1 ? '🏆 ' : ''}{m.rank}</td>
                                <td className={styles['cell-model']}>{m.model_id}</td>
                                <td className={styles['cell-num']}>{Math.round(m.score * 100)}%</td>
                                <td className={styles['cell-pass']}>{m.passed}</td>
                                <td className={styles['cell-fail']}>{m.failed}</td>
                                <td className={`${styles['cell-num']} ${styles['cell-num--muted']}`}>{m.total}</td>
                              </tr>
                            ))}
                          </tbody>
                        </table>
                      </div>
                    ) : (
                      <div className={styles['status-message']}>
                        {detail.status === 'pending' && 'This report hasn\u2019t been generated yet.'}
                        {detail.status === 'running' && 'This report is still being generated.'}
                        {detail.status === 'failed' && 'Report generation failed.'}
                      </div>
                    )}
                  </>
                )}
              </>
            )}
          </div>
        </div>
      </div>
    </div>
  );
}
