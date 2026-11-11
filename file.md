//history.tsx
import { useEffect, useMemo, useRef, useState } from 'react';
import { useSearchParams, useNavigate } from 'react-router-dom';
import {
  Search, Sparkles, Bot, Layers, Loader2, Download, ListTree, CheckCircle2, XCircle,
  Award, ListChecks, Clock, History as HistoryIcon, SlidersHorizontal, CalendarDays, X, PlaySquare,
  StopCircle, AlertTriangle, Trash2, ChevronLeft, ChevronRight,
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import {
  fetchEvaluations, fetchEvaluationResults, cancelEvaluation, deleteEvaluation, fetchEvaluationProgress,
} from '../../store/slices/evaluationsSlice';
import { downloadReport } from '../../store/slices/reportsSlice';
import type { ReportDownloadFormat } from '../../api/endpoints/reports';
import type { EvaluationListItem, EvaluationStatusValue, ModelResult, TestDetail } from '../../types';
import { SkeletonListRows } from '../common/Skeleton';
// Document-agent harness (Workstream 2): runs are mocked for now — see
// api/mock/harnessMock.ts. HarnessScorecard renders their detail view.
import { USE_HARNESS_MOCK, isMockHarnessId } from '../../api/mock/harnessMock';
import { listMockHarnessEvaluations } from '../../api/mock/harnessResultsMock';
import HarnessScorecard from './HarnessScorecard';
import styles from './History.module.scss';

const TYPE_ICON: Record<string, typeof Sparkles> = { model: Sparkles, agent: Bot, rag: Layers, agent_harness: Bot };
const TYPE_LABEL: Record<string, string> = { model: 'AI Model', agent: 'Agent', rag: 'RAG', agent_harness: 'Agent harness' };

// Document-agent harness runs get their own detail view (HarnessScorecard)
// instead of the per-model results table, and never go through
// GET /evaluations/{id}/results.
const HARNESS_EVAL_TYPE = 'agent_harness';
const isHarnessEval = (e: { eval_type: string } | null | undefined) => e?.eval_type === HARNESS_EVAL_TYPE;

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
  const serverList = rawList || [];
  const models = useAppSelector((s) => s.models.items) || [];
  const providers = useAppSelector((s) => s.providers.items) || [];
  const downloadingId = useAppSelector((s) => s.reports.downloadingId);
  const cancelingId = useAppSelector((s) => s.evaluations.cancelingId);
  const deletingId = useAppSelector((s) => s.evaluations.deletingId);
  const progressByEvalId = useAppSelector((s) => s.evaluations.progressByEvalId);

  const [search, setSearch] = useState('');
  const [typeFilter, setTypeFilter] = useState('All');
  const [dateFilter, setDateFilter] = useState('all');
  const [statusFilter, setStatusFilter] = useState('All');
  const [activeFilter, setActiveFilter] = useState<'search' | 'type' | 'date' | 'status' | null>(null);
  const [detailsModel, setDetailsModel] = useState<ModelResult | null>(null);
  // Which metric the drawer's test list is scoped to — null means "no
  // metric breakdown for this model, show every test case" (the fallback
  // row). Set alongside detailsModel by openDrawer below.
  const [detailsMetric, setDetailsMetric] = useState<string | null>(null);
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

  // While the harness backend is mocked, its runs don't exist in
  // GET /evaluations — merge the mock ones in at the top of page 1, honouring
  // the type/status filters the same way the server would (they're 'agent'
  // runs, always 'completed'). Once USE_HARNESS_MOCK is off this is just
  // `serverList`, and harness runs arrive from the API like any other row.
  const list = useMemo(() => {
    const showMock =
      USE_HARNESS_MOCK &&
      pageState === 1 &&
      (typeFilter === 'All' || typeFilter === 'agent') &&
      (statusFilter === 'All' || statusFilter === 'completed');
    return showMock ? [...listMockHarnessEvaluations(), ...serverList] : serverList;
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [rawList, pageState, typeFilter, statusFilter]);

  // Initial load + refetch on page/page-size/filter change, plus a silent
  // 10s background poll of whatever's currently being viewed (spec §2.4) —
  // the poll never disrupts the loading/error UI over data already on
  // screen (see fetchEvaluations.pending in the slice).
  //
  // After each fetch resolves, kick off a progress poll (GET
  // /evaluations/{id}/status) for every row that came back 'running' —
  // that's the only status this endpoint is meaningful for, so it's never
  // called for anything else.
  useEffect(() => {
    const loadList = async (silent: boolean) => {
      const result = await dispatch(
        fetchEvaluations({ page: pageState, pageSize: pageSizeState, status: statusFilter, evalType: typeFilter, silent })
      );
      if (fetchEvaluations.fulfilled.match(result)) {
        result.payload.evaluations
          .filter((e) => e.status === 'running')
          .forEach((e) => dispatch(fetchEvaluationProgress(e.id)));
      }
    };
    loadList(false);
    const interval = setInterval(() => loadList(true), 10000);
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
    if (selected && !isHarnessEval(selected) && !resultsByEvalId[selected.id]) {
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
    // Harness runs load their own scorecard (HarnessScorecard) instead.
    if (clicked && !isHarnessEval(clicked)) {
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

  // Opens the test-details drawer for a model, scoped to one metric row's
  // worth of test cases when opened from a metric-specific row in the
  // results table (metric === null for the no-breakdown fallback row,
  // which shows every test case for that model).
  const openDrawer = (model: ModelResult, metric: string | null) => {
    setDetailsModel(model);
    setDetailsMetric(metric);
    setTestFilter('all');
  };

  // ---- drawer width, drag-to-resize from the left edge -------------------
  const DRAWER_MIN_WIDTH = 380;
  // Matches the CSS max-width below (calc(100vw - 256px)) — 256px is the
  // app's global sidebar width, so the drawer can never be dragged wide
  // enough to sit underneath/behind it.
  const getDrawerMaxWidth = () => window.innerWidth - 256;
  const [drawerWidth, setDrawerWidth] = useState(480);
  const [isResizingDrawer, setIsResizingDrawer] = useState(false);
  const resizeStartRef = useRef<{ x: number; width: number } | null>(null);

  const startDrawerResize = (e: React.MouseEvent) => {
    e.preventDefault();
    resizeStartRef.current = { x: e.clientX, width: drawerWidth };
    setIsResizingDrawer(true);
  };

  useEffect(() => {
    if (!isResizingDrawer) return;
    const onMove = (e: MouseEvent) => {
      if (!resizeStartRef.current) return;
      // Drawer is anchored to the right edge and the handle sits on its
      // left edge, so dragging left (clientX decreasing) should widen it.
      const delta = resizeStartRef.current.x - e.clientX;
      const next = Math.min(getDrawerMaxWidth(), Math.max(DRAWER_MIN_WIDTH, resizeStartRef.current.width + delta));
      setDrawerWidth(next);
    };
    const onUp = () => setIsResizingDrawer(false);
    window.addEventListener('mousemove', onMove);
    window.addEventListener('mouseup', onUp);
    // Keep the resize cursor and prevent text selection for the whole page
    // while dragging, not just while the pointer is exactly over the handle.
    document.body.style.cursor = 'ew-resize';
    document.body.style.userSelect = 'none';
    return () => {
      window.removeEventListener('mousemove', onMove);
      window.removeEventListener('mouseup', onUp);
      document.body.style.cursor = '';
      document.body.style.userSelect = '';
    };
  }, [isResizingDrawer]);

  // ---- test-details pass/fail filter --------------------------------------
  const [testFilter, setTestFilter] = useState<'all' | 'passed' | 'failed'>('all');

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

  // Whether the Run Configuration panel has anything worth showing —
  // top-level max_retries/timeout are deliberately excluded from display,
  // so their presence alone shouldn't be enough to render an otherwise-
  // empty panel (just a title with nothing under it).
  const mc = results?.metrics_config;
  const hasConfigToShow = !!(
    mc &&
    (mc.retest_on_wrong != null ||
      mc.retest_verify_metric ||
      (mc.model_retry_config && Object.keys(mc.model_retry_config).length > 0))
  );

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
              {listStatus === 'succeeded' && total === 0 && list.length === 0 && (
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
                      ) : isMockHarnessId(e.id) ? null : (
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
                    {isRunning &&
                      (() => {
                        const p = progressByEvalId[e.id];
                        const current = p?.celeryState?.current ?? 0;
                        const total = p?.celeryState?.total ?? 0;
                        const pct = total > 0 ? Math.min(100, Math.round((current / total) * 100)) : 0;
                        return (
                          <div className={styles['row__progress']}>
                            <div className={styles['row__progress-top']}>
                              <span className={styles['row__progress-caption']}>Progress</span>
                              <span className={styles['row__progress-pct']}>{pct}%</span>
                            </div>
                            <div className={styles['row__progress-track']}>
                              <div className={styles['row__progress-fill']} style={{ width: `${pct}%` }} />
                            </div>
                            {total > 0 && (
                              <div className={styles['row__progress-count']}>
                                {current} / {total} completed
                              </div>
                            )}
                          </div>
                        );
                      })()}
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
                      offered when the results fetch reports has_html_report
                      === true (not available until that fetch resolves, so
                      the HTML button can pop in a moment after the others). */}
                  {selected.report?.report_id && (
                    <div className={styles['detail-hdr__actions']}>
                      {[...DOWNLOAD_OPTIONS, ...(results?.has_html_report ? [HTML_DOWNLOAD_OPTION] : [])].map((opt) => (
                        <button
                          key={opt.format}
                          className={styles['dl-btn']}
                          disabled={downloadingId === selected.report?.report_id}
                          onClick={() => {
                            if (!selected.report?.report_id) return;
                            dispatch(
                              downloadReport({
                                reportId: selected.report.report_id,
                                format: opt.format,
                                filenameHint: selected.report.title || selected.name,
                              })
                            );
                          }}
                        >
                          {downloadingId === selected.report?.report_id ? (
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
                      <div className={styles['summary-card__label']}>
                        {isHarnessEval(selected) ? 'Tasks / Models' : 'Questions / Models'}
                      </div>
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
                {/* Document-agent harness runs: agent scorecard, per-use-case
                    breakdown, error analysis and trace store — see
                    HarnessScorecard.tsx. Everything else keeps the existing
                    results flow below, unchanged. */}
                {isHarnessEval(selected) ? (
                  <HarnessScorecard evaluationId={selected.id} modelName={modelName} />
                ) : (
                <>
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
                    {hasConfigToShow && (
                      <div className={styles['config-panel']}>
                        <div className={styles['config-panel__title']}>Run configuration</div>
                        <div className={styles['config-panel__grid']}>
                          {/* Top-level max_retries/timeout deliberately not shown —
                              only the per-model overrides below are relevant here. */}
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
                              <th>Metric</th>
                              <th>Score</th>
                              <th>Passed</th>
                              <th>Failed</th>
                              <th />
                            </tr>
                          </thead>
                          <tbody>
                            {(results.results || []).map((r) => {
                              const breakdown = Object.entries(r.metric_breakdown || {});
                              const rowSpan = breakdown.length || 1;
                              const winnerClass = r.rank === 1 ? styles.winner : '';

                              // One row per metric when the model has a
                              // breakdown — Rank/Model/Provider are merged
                              // across those rows via rowSpan rather than
                              // repeated on each line.
                              if (breakdown.length > 0) {
                                return breakdown.map(([metricLabel, m], idx) => (
                                  <tr key={`${r.model_id}-${metricLabel}`} className={winnerClass}>
                                    {idx === 0 && (
                                      <>
                                        <td className={styles['cell-rank']} rowSpan={rowSpan}>
                                          {r.rank === 1 ? '🏆 ' : ''}
                                          {r.rank ?? '—'}
                                        </td>
                                        <td className={styles['cell-model']} rowSpan={rowSpan}>{modelName(r.model_id)}</td>
                                        <td className={styles['cell-provider']} rowSpan={rowSpan}>{r.provider || providerName(r.model_id)}</td>
                                      </>
                                    )}
                                    <td className={styles['cell-metric']}>{metricLabel}</td>
                                    <td className={styles['cell-num']}>{Math.round(Math.max(0, Math.min(1, m.score ?? 0)) * 100)}%</td>
                                    <td className={styles['cell-pass']}>{m.passed ?? 0}</td>
                                    <td className={styles['cell-fail']}>{m.failed ?? 0}</td>
                                    <td className={styles['cell-details']}>
                                      {(r.details?.some((d) => d.metric_passed && metricLabel in d.metric_passed) ?? false) && (
                                        <button
                                          type="button"
                                          className={styles['details-btn']}
                                          title={`View test-by-test details for ${metricLabel}`}
                                          onClick={() => openDrawer(r, metricLabel)}
                                        >
                                          <ListTree size={14} />
                                        </button>
                                      )}
                                    </td>
                                  </tr>
                                ));
                              }

                              // No metric breakdown for this model — a
                              // single plain row, details icon (if any)
                              // shows every test case unscoped.
                              return (
                                <tr key={r.model_id} className={winnerClass}>
                                  <td className={styles['cell-rank']}>
                                    {r.rank === 1 ? '🏆 ' : ''}
                                    {r.rank ?? '—'}
                                  </td>
                                  <td className={styles['cell-model']}>{modelName(r.model_id)}</td>
                                  <td className={styles['cell-provider']}>{r.provider || providerName(r.model_id)}</td>
                                  <td className={styles['cell-metric']}>—</td>
                                  <td className={styles['cell-num']}>{r.score ?? 0}%</td>
                                  <td className={styles['cell-pass']}>{r.passed_tests ?? 0}</td>
                                  <td className={styles['cell-fail']}>{r.failed_tests ?? 0}</td>
                                  <td className={styles['cell-details']}>
                                    {(r.details?.length ?? 0) > 0 && (
                                      <button
                                        type="button"
                                        className={styles['details-btn']}
                                        title="View test-by-test details"
                                        onClick={() => openDrawer(r, null)}
                                      >
                                        <ListTree size={14} />
                                      </button>
                                    )}
                                  </td>
                                </tr>
                              );
                            })}
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
              </>
            )}
          </div>
        </div>
      </div>

      {/* ---------- Test-detail / metric-score slide-over ---------- */}
      <div className={`${styles['drawer-overlay']} ${detailsModel ? styles['drawer-overlay--open'] : ''}`} onClick={() => setDetailsModel(null)} />
      <div
        className={`${styles.drawer} ${detailsModel ? styles['drawer--open'] : ''} ${isResizingDrawer ? styles['drawer--resizing'] : ''}`}
        style={{ width: drawerWidth }}
        role="dialog"
        aria-hidden={!detailsModel}
      >
        {/* Drag left/right from here to resize. Only meaningful once the
            drawer is actually open, but kept mounted (not conditionally
            rendered) so a drag in progress doesn't get cut off mid-gesture
            if detailsModel changes. */}
        <div
          className={styles['drawer__resize-handle']}
          onMouseDown={startDrawerResize}
          role="separator"
          aria-orientation="vertical"
          title="Drag to resize"
        />
        {detailsModel && (
          <>
            {(() => {
              // Scoped to one metric's test cases when opened from a metric
              // row (detailsMetric set) — a test case belongs to that scope
              // when it has an entry for this metric in metric_passed, since
              // one test case can be scored against several metrics at once.
              // Otherwise every test case for the model (no-breakdown
              // fallback row).
              const scopedDetails = (detailsModel.details || []).filter(
                (d) => !detailsMetric || (d.metric_passed && detailsMetric in d.metric_passed)
              );
              // Pass/fail state for a given test case, resolved for the
              // current scope — the metric-specific verdict when scoped to
              // one metric (a test case can pass overall but fail a
              // particular metric, or vice versa), otherwise its overall
              // passed flag.
              const isPassed = (d: TestDetail) =>
                detailsMetric ? d.metric_passed?.[detailsMetric] ?? d.passed : d.passed;

              const metricEntry = detailsMetric ? detailsModel.metric_breakdown?.[detailsMetric] : undefined;
              const headerScorePct = metricEntry
                ? Math.round(Math.max(0, Math.min(1, metricEntry.score ?? 0)) * 100)
                : detailsModel.score ?? 0;
              const total = scopedDetails.length;
              const passedCount = scopedDetails.filter(isPassed).length;
              const failedCount = total - passedCount;
              const visible = scopedDetails.filter((d) =>
                testFilter === 'all' ? true : testFilter === 'passed' ? isPassed(d) : !isPassed(d)
              );
              const filters: { key: 'all' | 'passed' | 'failed'; label: string; count: number }[] = [
                { key: 'all', label: 'All', count: total },
                { key: 'passed', label: 'Passed', count: passedCount },
                { key: 'failed', label: 'Failed', count: failedCount },
              ];

              return (
                <>
                  <div className={styles['drawer__header']}>
                    <div>
                      <div className={styles['drawer__eyebrow']}>
                        Test-by-test details{detailsMetric ? ` · ${detailsMetric}` : ''}
                      </div>
                      <h3 className={styles['drawer__title']}>{modelName(detailsModel.model_id)}</h3>
                      <div className={styles['drawer__sub']}>
                        {(detailsModel.provider || providerName(detailsModel.model_id))} · {headerScorePct}% score
                      </div>
                    </div>
                    <button type="button" className={styles['drawer__close']} onClick={() => setDetailsModel(null)} title="Close">
                      <X size={16} />
                    </button>
                  </div>

                  <div className={styles['drawer__filterbar']}>
                    {filters.map((f) => (
                      <button
                        key={f.key}
                        type="button"
                        className={`${styles['test-filter']} ${styles[`test-filter--${f.key}`]} ${testFilter === f.key ? styles.on : ''}`}
                        onClick={() => setTestFilter(f.key)}
                      >
                        {f.key === 'passed' && <CheckCircle2 size={12} />}
                        {f.key === 'failed' && <XCircle size={12} />}
                        {f.label}
                        <span className={styles['test-filter__count']}>{f.count}</span>
                      </button>
                    ))}
                  </div>

                  <div className={styles['drawer__body']}>
                    {visible.length === 0 ? (
                      <div className={styles['drawer__empty']}>
                        {testFilter === 'passed' && 'No passed tests to show.'}
                        {testFilter === 'failed' && 'No failed tests to show.'}
                        {testFilter === 'all' && 'No test details available.'}
                      </div>
                    ) : (
                      visible.map((d, i) => {
                        const passedHere = isPassed(d);
                        const metricScorePct =
                          detailsMetric && d.metric_scores?.[detailsMetric] != null
                            ? Math.round(Math.max(0, Math.min(1, d.metric_scores[detailsMetric])) * 100)
                            : null;
                        return (
                          <div key={i} className={`${styles['detail-card']} ${passedHere ? styles['detail-card--pass'] : styles['detail-card--fail']}`}>
                            <div className={styles['detail-card__hdr']}>
                              <span className={styles['detail-card__task']}>{d.task || `Test ${i + 1}`}</span>
                              <span className={`${styles['detail-card__badge']} ${passedHere ? styles['detail-card__badge--pass'] : styles['detail-card__badge--fail']}`}>
                                {passedHere ? <CheckCircle2 size={12} /> : <XCircle size={12} />}
                                {passedHere ? 'Passed' : 'Failed'}
                                {metricScorePct != null && ` · ${metricScorePct}%`}
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
                                <div className={`${styles['detail-card__text']} ${!passedHere ? styles['detail-card__text--fail'] : ''}`}>
                                  {d.actual_output || '—'}
                                </div>
                              </div>
                            </div>
                          </div>
                        );
                      })
                    )}
                  </div>
                </>
              );
            })()}
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
























//harnessscorecard.tsx
import { Fragment, useEffect, useMemo, useState } from 'react';
import { Award, CheckCircle2, ChevronDown, ChevronRight, Loader2, XCircle } from 'lucide-react';
import { harnessResultsApi } from '../../api/endpoints/harnessResults';
import type {
  HarnessCell,
  HarnessErrorType,
  HarnessResultsResponse,
} from '../../api/endpoints/harnessResults';
import styles from './HarnessScorecard.module.scss';

// ---------------------------------------------------------------------------
// HarnessScorecard — the detail view for a Document-agent harness run
// (eval_type 'agent_harness'), rendered by History.tsx in place of the
// per-model results table. Four parts, matching the Workstream 2 outputs:
//   1. Agent scorecard   — one column per agent (Hermes vs Co-Work), task
//                          success rate as the headline metric
//   2. By use case       — success rate per agent x model x suite
//   3. Error analysis    — failed tasks by error type, per agent
//   4. Trace store       — stored runs, expandable to their tool calls
//
// Self-contained on purpose: it fetches its own data through
// harnessResultsApi (mocked for now) into local state, so the evaluations
// slice and the existing results flow stay untouched.
// ---------------------------------------------------------------------------

const ERROR_LABEL: Record<HarnessErrorType, string> = {
  wrong_tool: 'Wrong tool',
  wrong_argument: 'Wrong argument',
  gave_up: 'Gave up',
  loop: 'Loop',
  file_broken: 'File broken',
};
const ERROR_TYPES = Object.keys(ERROR_LABEL) as HarnessErrorType[];

interface AgentSummary {
  id: string;
  title: string;
  tasks: number;
  passed: number;
  rate: number;
  toolAcc: number;
  steps: number;
  wasted: number;
  faithfulness: number | null;
  cost: number;
  time: number;
  errors: Record<HarnessErrorType, number>;
}

// Task-weighted roll-up of a set of cells.
function summarize(cells: HarnessCell[]) {
  const tasks = cells.reduce((n, c) => n + c.tasks, 0);
  const w = (pick: (c: HarnessCell) => number) =>
    tasks > 0 ? cells.reduce((n, c) => n + pick(c) * c.tasks, 0) / tasks : 0;
  const faithCells = cells.filter((c) => c.faithfulness !== null);
  const faithTasks = faithCells.reduce((n, c) => n + c.tasks, 0);
  const errors = { wrong_tool: 0, wrong_argument: 0, gave_up: 0, loop: 0, file_broken: 0 };
  cells.forEach((c) => ERROR_TYPES.forEach((t) => (errors[t] += c.errors?.[t] ?? 0)));
  const passed = cells.reduce((n, c) => n + c.passed, 0);
  return {
    tasks,
    passed,
    rate: tasks > 0 ? passed / tasks : 0,
    toolAcc: w((c) => c.tool_call_accuracy),
    steps: w((c) => c.avg_steps),
    wasted: w((c) => c.wasted_steps),
    faithfulness:
      faithTasks > 0 ? faithCells.reduce((n, c) => n + (c.faithfulness ?? 0) * c.tasks, 0) / faithTasks : null,
    cost: w((c) => c.cost_per_task),
    time: w((c) => c.time_per_task),
    errors,
  };
}

const pct = (n: number) => `${Math.round(n * 100)}%`;
const tier = (n: number) => (n >= 0.8 ? 'high' : n >= 0.6 ? 'mid' : 'low');

interface Props {
  evaluationId: string;
  // History's own id -> display-name lookup, so model names match the rest
  // of the page.
  modelName: (id: string) => string;
}

export default function HarnessScorecard({ evaluationId, modelName }: Props) {
  const [data, setData] = useState<HarnessResultsResponse | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(true);
  const [traceFilter, setTraceFilter] = useState<'all' | 'failed'>('all');
  const [traceAgent, setTraceAgent] = useState<string>('all');
  const [openTraceId, setOpenTraceId] = useState<string | null>(null);

  useEffect(() => {
    let cancelled = false;
    setLoading(true);
    setError(null);
    setData(null);
    setOpenTraceId(null);
    setTraceAgent('all');
    harnessResultsApi
      .get(evaluationId)
      .then((res) => {
        if (!cancelled) setData(res);
      })
      .catch((err) => {
        if (cancelled) return;
        setError(
          (err as { response?: { data?: { detail?: string } } })?.response?.data?.detail ||
            (err as Error)?.message ||
            'Failed to load the scorecard.'
        );
      })
      .finally(() => {
        if (!cancelled) setLoading(false);
      });
    return () => {
      cancelled = true;
    };
  }, [evaluationId]);

  const agents: AgentSummary[] = useMemo(
    () =>
      (data?.agents ?? []).map((a) => ({
        id: a.id,
        title: a.title,
        ...summarize((data?.cells ?? []).filter((c) => c.agent === a.id)),
      })),
    [data]
  );

  if (loading) {
    return (
      <div className={styles.hs__state}>
        <Loader2 size={16} className={styles.hs__spin} /> Loading the agent scorecard…
      </div>
    );
  }
  if (error || !data) {
    return <div className={styles.hs__state}>{error || 'No scorecard available.'}</div>;
  }

  const bestRate = Math.max(0, ...agents.map((a) => a.rate));
  const agentTitle = (id: string) => data.agents.find((a) => a.id === id)?.title ?? id;
  const suiteName = (id: string) => data.suites.find((s) => s.id === id)?.name ?? id;

  // Comparison rows: label, formatted value per agent, and which direction
  // is better (so the best cell can be marked).
  const metricRows: {
    label: string;
    hint?: string;
    value: (a: AgentSummary) => number | null;
    format: (n: number) => string;
    better: 'high' | 'low';
  }[] = [
    { label: 'Tool-call accuracy', hint: 'Right tool, right arguments, no extra calls', value: (a) => a.toolAcc, format: pct, better: 'high' },
    { label: 'Answer faithfulness', hint: 'To the source document', value: (a) => a.faithfulness, format: pct, better: 'high' },
    { label: 'Steps per task', value: (a) => a.steps, format: (n) => n.toFixed(1), better: 'low' },
    { label: 'Wasted steps', hint: 'Per task', value: (a) => a.wasted, format: (n) => n.toFixed(1), better: 'low' },
    { label: 'Cost per task', value: (a) => a.cost, format: (n) => `$${n.toFixed(3)}`, better: 'low' },
    { label: 'Time per task', value: (a) => a.time, format: (n) => `${n.toFixed(1)}s`, better: 'low' },
  ];
  const isBest = (row: (typeof metricRows)[number], a: AgentSummary) => {
    if (agents.length < 2) return false;
    const own = row.value(a);
    if (own === null) return false;
    const all = agents.map((x) => row.value(x)).filter((v): v is number => v !== null);
    return row.better === 'high' ? own >= Math.max(...all) : own <= Math.min(...all);
  };

  const maxErrorCount = Math.max(1, ...agents.flatMap((a) => ERROR_TYPES.map((t) => a.errors[t])));
  const totalFailed = agents.reduce((n, a) => n + (a.tasks - a.passed), 0);

  const visibleTraces = data.traces.filter(
    (t) => (traceFilter === 'all' || !t.passed) && (traceAgent === 'all' || t.agent === traceAgent)
  );

  return (
    <div className={styles.hs}>
      {/* ---------- 1. Agent scorecard ---------- */}
      <section className={styles.hs__sec}>
        <div className={styles['hs__sec-head']}>
          <h3>Agent scorecard</h3>
          <span>
            Same {data.suites.length} suite{data.suites.length === 1 ? '' : 's'}, same harness, averaged over{' '}
            {data.model_ids.length} model{data.model_ids.length === 1 ? '' : 's'}
          </span>
        </div>

        <div className={styles.hs__agents} style={{ gridTemplateColumns: `repeat(${Math.max(1, agents.length)}, minmax(0, 1fr))` }}>
          {agents.map((a) => {
            const best = agents.length > 1 && a.rate === bestRate;
            return (
              <div key={a.id} className={`${styles.hs__agent} ${best ? styles['hs__agent--best'] : ''}`}>
                <div className={styles['hs__agent-top']}>
                  <span className={styles['hs__agent-name']}>{a.title}</span>
                  {best && (
                    <span className={styles.hs__badge}>
                      <Award size={11} /> Best success rate
                    </span>
                  )}
                </div>
                <div className={styles['hs__agent-rate']}>{pct(a.rate)}</div>
                <div className={styles['hs__agent-label']}>
                  Task success rate · {a.passed.toLocaleString()} of {a.tasks.toLocaleString()} tasks
                </div>
                <div className={styles.hs__bar}>
                  <div className={styles['hs__bar-fill']} style={{ width: pct(a.rate) }} />
                </div>
              </div>
            );
          })}
        </div>

        <div className={styles['hs__table-wrap']}>
          <table className={styles.hs__table}>
            <thead>
              <tr>
                <th>Metric</th>
                {agents.map((a) => (
                  <th key={a.id} className={styles.hs__num}>
                    {a.title}
                  </th>
                ))}
              </tr>
            </thead>
            <tbody>
              {metricRows.map((row) => (
                <tr key={row.label}>
                  <td>
                    <b>{row.label}</b>
                    {row.hint && <small>{row.hint}</small>}
                  </td>
                  {agents.map((a) => {
                    const v = row.value(a);
                    return (
                      <td key={a.id} className={`${styles.hs__num} ${isBest(row, a) ? styles['hs__num--best'] : ''}`}>
                        {v === null ? '—' : row.format(v)}
                      </td>
                    );
                  })}
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      </section>

      {/* ---------- 2. By use case, per agent and model ---------- */}
      <section className={styles.hs__sec}>
        <div className={styles['hs__sec-head']}>
          <h3>Success rate by use case</h3>
          <span>Per agent and the model behind it</span>
        </div>
        <div className={styles['hs__table-wrap']}>
          <table className={styles.hs__table}>
            <thead>
              <tr>
                <th>Agent</th>
                <th>Model</th>
                {data.suites.map((s) => (
                  <th key={s.id} className={styles.hs__num} title={s.name}>
                    {s.use_case || s.name}
                    <small>{s.name}</small>
                  </th>
                ))}
                <th className={styles.hs__num}>Overall</th>
              </tr>
            </thead>
            <tbody>
              {data.agents.map((a) =>
                data.model_ids.map((modelId, mi) => {
                  const own = data.cells.filter((c) => c.agent === a.id && c.model_id === modelId);
                  const overall = summarize(own);
                  return (
                    <tr key={`${a.id}-${modelId}`}>
                      {mi === 0 && (
                        <td rowSpan={data.model_ids.length} className={styles['hs__cell-agent']}>
                          <b>{a.title}</b>
                        </td>
                      )}
                      <td>{modelName(modelId)}</td>
                      {data.suites.map((s) => {
                        const cell = own.find((c) => c.suite_id === s.id);
                        if (!cell || cell.tasks === 0) {
                          return (
                            <td key={s.id} className={styles.hs__num}>
                              —
                            </td>
                          );
                        }
                        const rate = cell.passed / cell.tasks;
                        return (
                          <td key={s.id} className={styles.hs__num} title={`${cell.passed} of ${cell.tasks} tasks passed`}>
                            <span className={`${styles.hs__rate} ${styles[`hs__rate--${tier(rate)}`]}`}>{pct(rate)}</span>
                          </td>
                        );
                      })}
                      <td className={`${styles.hs__num} ${styles['hs__num--strong']}`}>{pct(overall.rate)}</td>
                    </tr>
                  );
                })
              )}
            </tbody>
          </table>
        </div>
        <p className={styles.hs__legend}>
          <span className={`${styles.hs__rate} ${styles['hs__rate--high']}`}>80%+</span>
          <span className={`${styles.hs__rate} ${styles['hs__rate--mid']}`}>60–79%</span>
          <span className={`${styles.hs__rate} ${styles['hs__rate--low']}`}>Below 60%</span>
        </p>
      </section>

      {/* ---------- 3. Error analysis ---------- */}
      <section className={styles.hs__sec}>
        <div className={styles['hs__sec-head']}>
          <h3>Error analysis</h3>
          <span>
            {totalFailed.toLocaleString()} failed task{totalFailed === 1 ? '' : 's'}, by what went wrong
          </span>
        </div>
        <div className={styles['hs__table-wrap']}>
          <table className={styles.hs__table}>
            <thead>
              <tr>
                <th>Error type</th>
                {agents.map((a) => (
                  <th key={a.id}>{a.title}</th>
                ))}
              </tr>
            </thead>
            <tbody>
              {ERROR_TYPES.map((t) => (
                <tr key={t}>
                  <td>
                    <b>{ERROR_LABEL[t]}</b>
                  </td>
                  {agents.map((a) => (
                    <td key={a.id}>
                      <span className={styles.hs__errbar}>
                        <span className={styles['hs__errbar-track']}>
                          <span
                            className={styles['hs__errbar-fill']}
                            style={{ width: `${(a.errors[t] / maxErrorCount) * 100}%` }}
                          />
                        </span>
                        <span className={styles['hs__errbar-n']}>{a.errors[t]}</span>
                      </span>
                    </td>
                  ))}
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      </section>

      {/* ---------- 4. Trace store ---------- */}
      <section className={styles.hs__sec}>
        <div className={styles['hs__sec-head']}>
          <h3>Trace store</h3>
          <span>
            {data.settings.save_traces
              ? `Sample of stored runs · limits ${data.settings.max_steps} steps, ${data.settings.task_timeout}s per task`
              : 'Traces were not saved for this run'}
          </span>
        </div>

        {data.settings.save_traces && (
          <>
            <div className={styles.hs__filters}>
              {(['all', 'failed'] as const).map((f) => (
                <button
                  key={f}
                  type="button"
                  className={`${styles.hs__chip} ${traceFilter === f ? styles['hs__chip--on'] : ''}`}
                  onClick={() => setTraceFilter(f)}
                >
                  {f === 'all' ? 'All runs' : 'Failed only'}
                </button>
              ))}
              <span className={styles['hs__filters-sep']} />
              <button
                type="button"
                className={`${styles.hs__chip} ${traceAgent === 'all' ? styles['hs__chip--on'] : ''}`}
                onClick={() => setTraceAgent('all')}
              >
                All agents
              </button>
              {data.agents.map((a) => (
                <button
                  key={a.id}
                  type="button"
                  className={`${styles.hs__chip} ${traceAgent === a.id ? styles['hs__chip--on'] : ''}`}
                  onClick={() => setTraceAgent(a.id)}
                >
                  {a.title}
                </button>
              ))}
            </div>

            <div className={styles['hs__table-wrap']}>
              <table className={`${styles.hs__table} ${styles['hs__table--rows']}`}>
                <thead>
                  <tr>
                    <th />
                    <th>Task</th>
                    <th>Agent · Model</th>
                    <th>Result</th>
                    <th className={styles.hs__num}>Steps</th>
                    <th className={styles.hs__num}>Tokens</th>
                    <th className={styles.hs__num}>Time</th>
                  </tr>
                </thead>
                <tbody>
                  {visibleTraces.map((t) => {
                    const open = openTraceId === t.id;
                    return (
                      <Fragment key={t.id}>
                        <tr
                          className={open ? styles['hs__row--open'] : ''}
                          onClick={() => setOpenTraceId(open ? null : t.id)}
                        >
                          <td className={styles.hs__toggle}>
                            <button
                              type="button"
                              aria-expanded={open}
                              aria-label={open ? 'Hide trace' : 'Show trace'}
                              onClick={(e) => {
                                e.stopPropagation();
                                setOpenTraceId(open ? null : t.id);
                              }}
                            >
                              {open ? <ChevronDown size={14} /> : <ChevronRight size={14} />}
                            </button>
                          </td>
                          <td>
                            <b>{t.task}</b>
                            <small>{suiteName(t.suite_id)}</small>
                          </td>
                          <td>
                            {agentTitle(t.agent)}
                            <small>{modelName(t.model_id)}</small>
                          </td>
                          <td>
                            <span className={`${styles.hs__result} ${t.passed ? styles['hs__result--pass'] : styles['hs__result--fail']}`}>
                              {t.passed ? <CheckCircle2 size={12} /> : <XCircle size={12} />}
                              {t.passed ? 'Passed' : t.error_type ? ERROR_LABEL[t.error_type] : 'Failed'}
                            </span>
                          </td>
                          <td className={styles.hs__num}>{t.steps.length}</td>
                          <td className={styles.hs__num}>{t.tokens.toLocaleString()}</td>
                          <td className={styles.hs__num}>{t.time_sec}s</td>
                        </tr>
                        {open && (
                          <tr className={styles['hs__row--trace']}>
                            <td />
                            <td colSpan={6}>
                              <ol className={styles.hs__steps}>
                                {t.steps.map((st, i) => (
                                  <li key={i} className={st.ok ? '' : styles['hs__step--fail']}>
                                    <span className={styles['hs__step-n']}>{i + 1}</span>
                                    <code>{st.tool}</code>
                                    <span className={styles['hs__step-sum']}>{st.summary}</span>
                                    <span className={styles['hs__step-ms']}>{(st.ms / 1000).toFixed(1)}s</span>
                                  </li>
                                ))}
                              </ol>
                            </td>
                          </tr>
                        )}
                      </Fragment>
                    );
                  })}
                  {visibleTraces.length === 0 && (
                    <tr>
                      <td colSpan={7} className={styles.hs__none}>
                        No stored runs match this filter.
                      </td>
                    </tr>
                  )}
                </tbody>
              </table>
            </div>
          </>
        )}
      </section>
    </div>
  );
}




















//harnessscorecard.module.scss
@use '../../styles/_variables' as *;

// ===========================================================================
// HarnessScorecard — detail view for a Document-agent harness run, shown in
// History's detail panel. Same "ink" tokens and em-based type scale as the
// New run wizard (NewEvaluation.module.scss); nothing is redeclared here.
// ===========================================================================

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

.hs {
  font-size: 0.8125rem; // base every em below is relative to
  display: flex;
  flex-direction: column;
  gap: 26px;
  margin-top: 20px;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  &__state {
    display: flex;
    align-items: center;
    gap: 8px;
    margin-top: 20px;
    padding: 18px;
    border: 1px dashed $line;
    background: $paper;
    color: $ink-2;
    font-size: 0.875rem;
  }

  &__spin { animation: hs-spin 0.8s linear infinite; }

  // ---- section ------------------------------------------------------------
  &__sec { display: flex; flex-direction: column; gap: 12px; }

  &__sec-head {
    display: flex;
    align-items: baseline;
    justify-content: space-between;
    flex-wrap: wrap;
    gap: 6px 16px;

    h3 { font-family: $display; font-size: 1.1538em; font-weight: 800; letter-spacing: -0.01em; color: $ink; }
    span { font-size: 0.9615em; color: $ink-2; }
  }

  // ---- agent cards --------------------------------------------------------
  &__agents { display: grid; gap: 12px; }

  &__agent {
    padding: 16px 18px;
    border: 1px solid $line;
    background: $card;

    &--best { border-color: $signal; background: $wash; }
  }

  &__agent-top { display: flex; align-items: center; justify-content: space-between; gap: 10px; }
  &__agent-name { font-family: $display; font-size: 1.1538em; font-weight: 700; color: $ink; }

  &__badge {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 3px 9px;
    border-radius: 999px;
    background: $signal;
    color: #fff;
    font-size: 0.8462em;
    font-weight: 650;
    white-space: nowrap;
  }

  &__agent-rate {
    margin-top: 10px;
    font-family: $display;
    font-size: 2.4615em;
    font-weight: 800;
    letter-spacing: -0.03em;
    line-height: 1;
    color: $ink;
  }
  &__agent-label { margin-top: 6px; font-size: 0.9615em; color: $ink-2; }

  &__bar { margin-top: 12px; height: 6px; border-radius: 999px; background: $line; overflow: hidden; }
  &__bar-fill { height: 100%; border-radius: 999px; background: $signal; }

  // ---- tables -------------------------------------------------------------
  &__table-wrap { overflow-x: auto; border: 1px solid $line; }

  &__table {
    width: 100%;
    border-collapse: collapse;
    font-size: 1em;

    th {
      padding: 10px 14px;
      background: $paper;
      border-bottom: 1px solid $line;
      text-align: left;
      font-family: $sans;
      font-size: 0.9231em;
      font-weight: 650;
      color: $ink-2;
      white-space: nowrap;
      vertical-align: bottom;

      small { display: block; font-weight: 500; color: $ink-3; }
    }

    td {
      padding: 11px 14px;
      border-bottom: 1px solid $line-2;
      color: $ink;
      vertical-align: middle;

      b { display: block; font-weight: 700; }
      small { display: block; font-size: 0.9231em; color: $ink-2; }
    }
    tbody tr:last-child td { border-bottom: 0; }

    th.hs__num,
    td.hs__num { text-align: right; white-space: nowrap; }
    td.hs__num { font-family: $mono; font-size: 0.9615em; }
    td.hs__num--best { font-weight: 700; color: $signal; }
    td.hs__num--strong { font-weight: 700; }
    td.hs__cell-agent { vertical-align: top; border-right: 1px solid $line-2; }
    td.hs__none { text-align: center; color: $ink-2; }

    // clickable rows (trace store)
    &--rows tbody tr { cursor: pointer; transition: background 0.12s ease; }
    &--rows tbody tr:hover td { background: $paper; }
    &--rows tbody tr.hs__row--open td { background: $wash; border-bottom-color: transparent; }
    &--rows tbody tr.hs__row--trace { cursor: default; }
    &--rows tbody tr.hs__row--trace td,
    &--rows tbody tr.hs__row--trace:hover td { background: $wash; padding-top: 0; }
  }

  // success-rate pill, three tiers (legend under the table)
  &__rate {
    display: inline-block;
    min-width: 46px;
    padding: 3px 8px;
    border-radius: 6px;
    text-align: center;
    font-family: $mono;
    font-size: 0.9615em;
    font-weight: 700;

    &--high { color: $ok; background: $ok-wash; }
    &--mid { color: $ink-2; background: $paper; }
    &--low { color: $danger; background: $danger-wash; }
  }

  &__legend { display: flex; flex-wrap: wrap; gap: 8px; }

  // ---- error bars ---------------------------------------------------------
  &__errbar { display: flex; align-items: center; gap: 10px; }
  &__errbar-track { flex: 1; min-width: 60px; height: 8px; border-radius: 999px; background: $line-2; overflow: hidden; }
  &__errbar-fill { display: block; height: 100%; border-radius: 999px; background: $danger; }
  &__errbar-n { min-width: 22px; text-align: right; font-family: $mono; font-size: 0.9615em; font-weight: 700; color: $ink; }

  // ---- trace store --------------------------------------------------------
  &__filters { display: flex; flex-wrap: wrap; align-items: center; gap: 8px; }
  &__filters-sep { width: 1px; height: 18px; background: $line; }

  &__chip {
    padding: 6px 12px;
    border: 1.5px solid $line;
    border-radius: 999px;
    background: $card;
    font-family: $sans;
    font-size: 0.9615em;
    font-weight: 650;
    color: $ink-2;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $ink-3; color: $ink; }
    &--on { border-color: $signal; background: $wash; color: $signal; }
  }

  &__toggle {
    width: 34px;

    button {
      display: inline-flex;
      padding: 2px;
      border: 0;
      background: transparent;
      color: $ink-2;
      cursor: pointer;
    }
  }

  &__result {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 3px 9px;
    border-radius: 999px;
    font-size: 0.9231em;
    font-weight: 650;
    white-space: nowrap;

    &--pass { color: $ok; background: $ok-wash; }
    &--fail { color: $danger; background: $danger-wash; }
  }

  &__steps {
    display: flex;
    flex-direction: column;
    gap: 6px;
    padding: 4px 0 12px;
    list-style: none;

    li {
      display: grid;
      grid-template-columns: 24px minmax(120px, 190px) minmax(0, 1fr) auto;
      align-items: center;
      gap: 10px;
      padding: 7px 10px;
      border: 1px solid $line;
      background: $card;
      font-size: 0.9615em;
      color: $ink;
    }

    code { font-family: $mono; font-weight: 700; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
  }
  &__step-n { font-family: $mono; font-size: 0.9231em; color: $ink-3; }
  &__step-sum { color: $ink-2; }
  &__step-ms { font-family: $mono; font-size: 0.9231em; color: $ink-3; }
  &__steps li.hs__step--fail { border-color: rgba($danger, 0.35); }
  &__step--fail &__step-sum { color: $danger; font-weight: 650; }
}

@keyframes hs-spin { to { transform: rotate(360deg); } }

@media (max-width: 900px) {
  .hs__agents { grid-template-columns: 1fr !important; }
}

@media (prefers-reduced-motion: reduce) {
  .hs * { animation: none !important; transition: none !important; }
}


















//harnessresults.ts
// api/endpoints/harnessResults.ts
//
// Results for a Document-agent harness run (Workstream 2) — what History's
// detail panel shows for eval_type 'agent_harness': the agent scorecard
// (Hermes vs Co-Work), per-model / per-suite breakdown, error analysis and
// the stored traces.
//
// MOCKED while USE_HARNESS_MOCK is true (same switch as the wizard — see
// api/mock/harnessMock.ts). With the flag off this calls:
//   GET /agent-harness/{evaluation_id}/results -> HarnessResultsResponse
// The path is a placeholder until the backend contract exists; the shape
// below is the proposal for it.
import api from '../axiosInstance';
import { USE_HARNESS_MOCK } from '../mock/harnessMock';
import { mockGetHarnessResults } from '../mock/harnessResultsMock';

// Error taxonomy from the spec: wrong tool, wrong argument, gave up, loop,
// file broken.
export type HarnessErrorType = 'wrong_tool' | 'wrong_argument' | 'gave_up' | 'loop' | 'file_broken';

// One scorecard cell = one agent running one model on one suite.
export interface HarnessCell {
  agent: string;
  model_id: string;
  suite_id: string;
  tasks: number;
  passed: number; // task success rate = passed / tasks (main metric)
  tool_call_accuracy: number; // 0–1: right tool, right arguments, no extra calls
  avg_steps: number; // steps per task
  wasted_steps: number; // steps per task that didn't move the task forward
  faithfulness: number | null; // 0–1, null where the suite has no source document
  tokens_per_task: number;
  cost_per_task: number; // USD
  time_per_task: number; // seconds
  errors: Record<HarnessErrorType, number>; // counts over the failed tasks
}

export interface HarnessTraceStep {
  tool: string;
  summary: string;
  ms: number;
  ok: boolean;
}

// One stored run of one task (trace store) — kept for replay and error study.
export interface HarnessTrace {
  id: string;
  agent: string;
  model_id: string;
  suite_id: string;
  task: string;
  passed: boolean;
  tokens: number;
  time_sec: number;
  error_type: HarnessErrorType | null;
  steps: HarnessTraceStep[];
}

export interface HarnessResultsResponse {
  evaluation_id: string;
  name: string;
  agents: { id: string; title: string }[];
  model_ids: string[];
  suites: { id: string; name: string; use_case: string }[];
  selected_metrics: string[];
  settings: { max_steps: number; task_timeout: number; save_traces: boolean };
  cells: HarnessCell[];
  // A sample of the trace store (empty when save_traces was off).
  traces: HarnessTrace[];
}

export const harnessResultsApi = {
  get: (evaluationId: string): Promise<HarnessResultsResponse> =>
    USE_HARNESS_MOCK
      ? mockGetHarnessResults(evaluationId)
      : api.get<HarnessResultsResponse>(`/agent-harness/${evaluationId}/results`).then((r) => r.data),
};






















//harnessresultsmock.ts
// api/mock/harnessResultsMock.ts
//
// MOCK results for Document-agent harness runs, so the whole flow can be
// tested without a backend: launch a harness run in the wizard -> it shows
// up in History -> its scorecard renders.
//
// Everything is generated deterministically from the launch payload (which
// agents / models / suites / sample size were picked), seeded by the run
// id — the same run always shows the same numbers. Nothing here is real
// measurement data.
import { HARNESS_AGENTS, MOCK_HARNESS_SUITES, listMockHarnessRuns, type MockHarnessRun } from './harnessMock';
import type { EvaluationListItem } from '../../types';
import type {
  HarnessCell,
  HarnessErrorType,
  HarnessResultsResponse,
  HarnessTrace,
  HarnessTraceStep,
} from '../endpoints/harnessResults';

const delay = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

// ---- seeded randomness ----------------------------------------------------
function hash(str: string): number {
  let h = 2166136261;
  for (let i = 0; i < str.length; i += 1) {
    h ^= str.charCodeAt(i);
    h = Math.imul(h, 16777619);
  }
  return h >>> 0;
}
function rng(seed: string): () => number {
  let a = hash(seed);
  return () => {
    a = (a + 0x6d2b79f5) | 0;
    let t = Math.imul(a ^ (a >>> 15), 1 | a);
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t;
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };
}
const clamp01 = (n: number) => Math.min(0.99, Math.max(0.05, n));

// ---- what the mock "knows" about each agent / use case ---------------------
// Baseline task success by agent and use case — Hermes is tuned as the
// stronger tool caller, Co-Work as the stronger document worker, so the
// scorecard has a visible shape to it.
const BASE_SUCCESS: Record<string, Record<string, number>> = {
  hermes: { 'Tool Calling': 0.88, 'Document reading': 0.71, 'Spreadsheet editing': 0.62, 'Document editing': 0.55 },
  cowork: { 'Tool Calling': 0.79, 'Document reading': 0.83, 'Spreadsheet editing': 0.74, 'Document editing': 0.81 },
};
const STEP_RANGE: Record<string, [number, number]> = {
  'Tool Calling': [3, 5],
  'Document reading': [6, 9],
  'Spreadsheet editing': [9, 14],
  'Document editing': [11, 16],
};
const ERROR_WEIGHTS: Record<string, Record<HarnessErrorType, number>> = {
  hermes: { wrong_tool: 0.2, wrong_argument: 0.3, gave_up: 0.15, loop: 0.25, file_broken: 0.1 },
  cowork: { wrong_tool: 0.15, wrong_argument: 0.2, gave_up: 0.3, loop: 0.1, file_broken: 0.25 },
};
const ERROR_TYPES: HarnessErrorType[] = ['wrong_tool', 'wrong_argument', 'gave_up', 'loop', 'file_broken'];

const TASKS: Record<string, string[]> = {
  'Tool Calling': [
    'Book the cheapest direct flight and add it to the calendar',
    'Convert 250 EUR to JPY, then email the result',
    'Answer without tools: what is the capital of Australia?',
    'Look up two orders in parallel and compare delivery dates',
  ],
  'Document reading': [
    'What was FY2023 operating margin in the annual report?',
    'Find the year-over-year change in free cash flow',
    'Which segment had the highest capital expenditure?',
  ],
  'Spreadsheet editing': [
    'Add a VLOOKUP column for region and total by quarter',
    'Remove duplicate customer rows and fix the date format',
    'Rebuild the broken SUMIFS on the summary sheet',
  ],
  'Document editing': [
    'Update the pricing table in the Word proposal',
    'Make a 6-slide deck from the quarterly report',
    'Write a one-page user guide from the release notes',
    'Summarise the incident review for leadership',
  ],
};
const TOOLS: Record<string, string[]> = {
  'Tool Calling': ['search_flights', 'get_exchange_rate', 'send_email', 'calendar.create', 'orders.lookup'],
  'Document reading': ['pdf.open', 'pdf.search', 'pdf.read_page', 'calculator'],
  'Spreadsheet editing': ['xlsx.open', 'xlsx.read_range', 'xlsx.write_formula', 'xlsx.delete_rows', 'xlsx.save'],
  'Document editing': ['docx.open', 'docx.find_table', 'docx.update_cell', 'pptx.add_slide', 'file.save'],
};
const ERROR_NOTE: Record<HarnessErrorType, string> = {
  wrong_tool: 'Called a tool that cannot do this step',
  wrong_argument: 'Right tool, wrong argument value',
  gave_up: 'Stopped before the task was finished',
  loop: 'Repeated the same call until the step limit',
  file_broken: 'Output file could not be opened by the checker',
};

// ---- generator --------------------------------------------------------------
function buildResults(run: MockHarnessRun): HarnessResultsResponse {
  const { payload } = run;
  const suites = payload.suites
    .map((s) => ({ run: s, meta: MOCK_HARNESS_SUITES.find((m) => m.id === s.dataset_id) }))
    .filter((s): s is { run: (typeof payload.suites)[number]; meta: (typeof MOCK_HARNESS_SUITES)[number] } => Boolean(s.meta));

  const cells: HarnessCell[] = [];
  const traces: HarnessTrace[] = [];

  payload.agents.forEach((agent) => {
    payload.model_ids.forEach((modelId) => {
      suites.forEach(({ run: suiteRun, meta }) => {
        const useCase = meta.use_case || 'Tool Calling';
        const rand = rng(`${run.id}|${agent}|${modelId}|${meta.id}`);
        // run_samples 0 = the full suite (same convention as the launch payload)
        const tasks =
          suiteRun.run_samples > 0 ? Math.min(suiteRun.run_samples, meta.question_count) : meta.question_count;

        const modelShift = ((hash(modelId) % 9) - 4) / 100; // -0.04 … +0.04 per model
        const success = clamp01((BASE_SUCCESS[agent]?.[useCase] ?? 0.7) + modelShift + (rand() - 0.5) * 0.06);
        const passed = Math.round(tasks * success);
        const failed = tasks - passed;

        const [minSteps, maxSteps] = STEP_RANGE[useCase] ?? [5, 9];
        const avgSteps = minSteps + rand() * (maxSteps - minSteps);
        const wasted = avgSteps * (0.08 + rand() * 0.17);
        const tokens = Math.round(avgSteps * (1500 + rand() * 700));

        // spread the failed tasks over the error types
        const weights = { ...(ERROR_WEIGHTS[agent] ?? ERROR_WEIGHTS.hermes) };
        if (useCase === 'Tool Calling') weights.file_broken = 0; // no output file to break
        const weightSum = ERROR_TYPES.reduce((n, t) => n + weights[t], 0);
        const errors = { wrong_tool: 0, wrong_argument: 0, gave_up: 0, loop: 0, file_broken: 0 };
        let left = failed;
        ERROR_TYPES.forEach((t, i) => {
          const n = i === ERROR_TYPES.length - 1 ? left : Math.min(left, Math.round((failed * weights[t]) / weightSum));
          errors[t] = n;
          left -= n;
        });
        if (useCase === 'Tool Calling' && errors.file_broken > 0) {
          errors.wrong_argument += errors.file_broken;
          errors.file_broken = 0;
        }

        cells.push({
          agent,
          model_id: modelId,
          suite_id: meta.id,
          tasks,
          passed,
          tool_call_accuracy: clamp01(success + 0.04 + (rand() - 0.5) * 0.05),
          avg_steps: Number(avgSteps.toFixed(1)),
          wasted_steps: Number(wasted.toFixed(1)),
          faithfulness:
            useCase === 'Document reading' || useCase === 'Document editing' ? clamp01(0.8 + rand() * 0.15) : null,
          tokens_per_task: tokens,
          cost_per_task: Number(((tokens / 1_000_000) * 6).toFixed(4)),
          time_per_task: Number((avgSteps * (3.4 + rand() * 1.6)).toFixed(1)),
          errors,
        });

        // trace store sample: one passing and (if any) one failing task per cell
        if (payload.harness.save_traces) {
          const taskPool = TASKS[useCase] ?? TASKS['Tool Calling'];
          const toolPool = TOOLS[useCase] ?? TOOLS['Tool Calling'];
          // pick one of the error types this cell actually had, so the sample varies
          const present = ERROR_TYPES.filter((t) => errors[t] > 0);
          const failType = present.length > 0 ? present[Math.floor(rand() * present.length)] : null;
          [true, ...(failed > 0 ? [false] : [])].forEach((ok, k) => {
            const stepCount = Math.max(2, Math.round(avgSteps + (ok ? -1 : 2)));
            const steps: HarnessTraceStep[] = Array.from({ length: stepCount }).map((_, i) => {
              const last = i === stepCount - 1;
              const tool =
                !ok && failType === 'loop' && i > 1 ? toolPool[1 % toolPool.length] : toolPool[i % toolPool.length];
              return {
                tool,
                summary: last && !ok && failType ? ERROR_NOTE[failType] : last ? 'Returned the final answer' : `Step ${i + 1} completed`,
                ms: Math.round(400 + rand() * 4200),
                ok: ok || !last,
              };
            });
            traces.push({
              id: `${run.id}-${agent}-${hash(modelId + meta.id) % 1000}-${k}`,
              agent,
              model_id: modelId,
              suite_id: meta.id,
              task: taskPool[Math.floor(rand() * taskPool.length)],
              passed: ok,
              tokens: Math.round(stepCount * (1500 + rand() * 700)),
              time_sec: Number((steps.reduce((n, s) => n + s.ms, 0) / 1000).toFixed(1)),
              error_type: ok ? null : failType,
              steps,
            });
          });
        }
      });
    });
  });

  return {
    evaluation_id: run.id,
    name: payload.evaluation_name,
    agents: payload.agents.map((id) => ({ id, title: HARNESS_AGENTS.find((a) => a.id === id)?.title ?? id })),
    model_ids: payload.model_ids,
    suites: suites.map(({ meta }) => ({ id: meta.id, name: meta.name, use_case: meta.use_case || '' })),
    selected_metrics: payload.selected_metrics,
    settings: payload.harness,
    cells,
    traces,
  };
}

// ---- mock "API" ---------------------------------------------------------------
export async function mockGetHarnessResults(evaluationId: string): Promise<HarnessResultsResponse> {
  await delay(500);
  const run = listMockHarnessRuns().find((r) => r.id === evaluationId);
  if (!run) throw new Error('Harness run not found.');
  return buildResults(run);
}

// History list rows for the mock runs (newest first). In the real flow these
// come back from GET /evaluations like any other evaluation; while mocked,
// History.tsx merges them in on page 1.
export function listMockHarnessEvaluations(): EvaluationListItem[] {
  return listMockHarnessRuns().map((run) => {
    const res = buildResults(run);
    const byAgent = res.agents.map((a) => {
      const own = res.cells.filter((c) => c.agent === a.id);
      const tasks = own.reduce((n, c) => n + c.tasks, 0);
      const passed = own.reduce((n, c) => n + c.passed, 0);
      return { title: a.title, rate: tasks > 0 ? passed / tasks : 0 };
    });
    const best = [...byAgent].sort((x, y) => y.rate - x.rate)[0];
    return {
      id: run.id,
      name: run.payload.evaluation_name,
      description: 'Document-agent harness (mock)',
      eval_type: 'agent_harness',
      dataset_id: res.suites[0]?.id ?? '',
      datasets_config: res.suites.map((s) => ({ dataset_id: s.id })),
      benchmark: res.suites.map((s) => s.name).join(', '),
      model_ids: run.payload.model_ids,
      selected_metrics: run.payload.selected_metrics,
      run_samples: run.payload.suites[0]?.run_samples ?? 0,
      selected_category: [],
      status: 'completed',
      progress: 100,
      total_questions: res.cells.reduce((n, c) => n + c.tasks, 0),
      top_model: best ? best.title : null,
      top_score: best ? Math.round(best.rate * 100) : null,
      created_at: run.created_at,
      started_at: run.created_at,
      completed_at: run.created_at,
      report: null,
    };
  });
}


























//harnessmock.ts
// api/mock/harnessMock.ts
//
// MOCK DATA for the Document-agent harness flow (Workstream 2).
// Everything the wizard needs from the backend for harness mode lives here,
// behind one switch, so swapping to real endpoints later is a one-line change:
//
//   export const USE_HARNESS_MOCK = false;
//
// With the flag off, evaluationsApi (api/endpoints/evaluations.ts) calls:
//   GET  /datasets?eval_type=agent_harness   -> Dataset[] (with use_case)
//   GET  /metrics?eval_type=agent_harness    -> MetricsResponse
//   POST /agent-harness/run                  -> 200 OK (optionally { id })
//
// Shapes below match the real types in ../../types so nothing else changes.
import type {
  AgentHarnessRunRequest,
  Dataset,
  HarnessAdapterId,
  MetricsResponse,
} from '../../types';

export const USE_HARNESS_MOCK = true;

// Simulated network latency so skeletons/spinners are exercised.
const delay = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

// ---- Agents (static catalog — adapters live server-side) ------------------
export const HARNESS_AGENTS: { id: HarnessAdapterId; title: string; desc: string }[] = [
  { id: 'hermes', title: 'Hermes', desc: 'Lightweight tool-calling agent runtime' },
  { id: 'cowork', title: 'Co-Work', desc: 'Document-focused agent (API / MCP endpoint)' },
];

// ---- Use cases (filter chips on the Test Suite step) ----------------------
export const HARNESS_USE_CASES = [
  'Tool Calling',
  'Document reading',
  'Spreadsheet editing',
  'Document editing',
] as const;

// ---- Suites: the 4 benchmark suites from the spec -------------------------
export const MOCK_HARNESS_SUITES: Dataset[] = [
  {
    id: 'harness-bfcl',
    name: 'BFCL',
    description:
      'Berkeley Function Calling Leaderboard — right tool, right arguments: simple, parallel, multi-turn and "no tool needed" cases.',
    category: 'Agents',
    eval_type: 'agent_harness',
    dataset_type: 'deepeval',
    question_count: 400,
    dataset_categories: ['Simple', 'Parallel', 'Multi-turn', 'No tool needed'],
    use_case: 'Tool Calling',
    scoring: 'Tool name + argument match; state check for multi-turn',
  },
  {
    id: 'harness-financebench',
    name: 'FinanceBench',
    description: 'Answering questions from long financial reports.',
    category: 'Agents',
    eval_type: 'agent_harness',
    dataset_type: 'deepeval',
    question_count: 150,
    dataset_categories: ['Metrics-generated', 'Domain-relevant', 'Novel-generated'],
    use_case: 'Document reading',
    scoring: 'Answer correctness (LLM judge + human spot check)',
  },
  {
    id: 'harness-spreadsheet',
    name: 'Spreadsheet',
    description: 'Real spreadsheet tasks: formulas, cleaning and edits.',
    category: 'Agents',
    eval_type: 'agent_harness',
    dataset_type: 'deepeval',
    question_count: 200,
    dataset_categories: ['Formulas', 'Cleaning', 'Edits'],
    use_case: 'Spreadsheet editing',
    scoring: 'Output file compared with expected file',
  },
  {
    id: 'harness-inhouse-docs',
    name: 'In-house document task set',
    description:
      'Edit docx / pptx / xlsx, write reports and documentation from sources. 60–100 real team tasks with a domain-written pass/fail checklist.',
    category: 'Agents',
    eval_type: 'agent_harness',
    dataset_type: 'custom',
    question_count: 80,
    dataset_categories: ['Word tables', 'Slides from report', 'Spreadsheet fixes', 'Summaries', 'User guides'],
    use_case: 'Document editing',
    scoring: 'Checklist rubric + LLM judge + file checks',
  },
];

// ---- Scorers (shown as the Metrics step's chips in harness mode) ----------
export const MOCK_HARNESS_METRICS: MetricsResponse = {
  eval_type: 'agent_harness',
  all_metrics: [
    'Task Success Rate',
    'Task Completion',
    'Tool Correctness',
    'Tool-call Accuracy',
    'Answer Faithfulness',
    'Steps per Task',
    'Wasted Steps',
    'Cost per Task',
    'Time per Task',
    'Error Types',
  ],
  custom: [],
};

// ---- Mock "API" -----------------------------------------------------------
export async function mockGetHarnessSuites(): Promise<Dataset[]> {
  await delay(450);
  return MOCK_HARNESS_SUITES.map((s) => ({ ...s }));
}

export async function mockGetHarnessMetrics(): Promise<MetricsResponse> {
  await delay(350);
  return { ...MOCK_HARNESS_METRICS, all_metrics: [...MOCK_HARNESS_METRICS.all_metrics] };
}

// ---- Launched runs (so History has something to show while mocked) --------
// Every mock launch is remembered here (sessionStorage, so it survives a
// reload but not a new tab/session) and surfaced in History as a completed
// harness evaluation — see api/mock/harnessResultsMock.ts for the generated
// scorecard. One seeded sample run is always present so History's harness
// view can be checked without launching anything first.
export interface MockHarnessRun {
  id: string;
  created_at: string;
  payload: AgentHarnessRunRequest;
}

const MOCK_RUNS_KEY = 'harness-mock-runs';
export const isMockHarnessId = (id: string) => id.startsWith('harness-mock-');

const SEED_RUN: MockHarnessRun = {
  id: 'harness-mock-seed',
  created_at: '2026-11-10T09:30:00.000Z',
  payload: {
    evaluation_name: 'Agent scorecard (sample run)',
    agents: ['hermes', 'cowork'],
    model_ids: ['gpt-4.1', 'claude-sonnet'],
    suites: MOCK_HARNESS_SUITES.map((s) => ({ dataset_id: s.id, run_samples: 20 })),
    selected_metrics: [...MOCK_HARNESS_METRICS.all_metrics],
    judge_config: { model_id: 'claude-opus', base_url: '', api_key: 'claude-opus', max_retries: 1, timeout: 60 },
    harness: { max_steps: 30, task_timeout: 300, save_traces: true },
    max_retries: 1,
    timeout: 60,
  },
};

function loadMockRuns(): MockHarnessRun[] {
  try {
    const raw = sessionStorage.getItem(MOCK_RUNS_KEY);
    const parsed = raw ? JSON.parse(raw) : [];
    return Array.isArray(parsed) ? parsed : [];
  } catch {
    return [];
  }
}

// Newest first, seeded sample last.
export function listMockHarnessRuns(): MockHarnessRun[] {
  return [...loadMockRuns(), SEED_RUN];
}

export function removeMockHarnessRun(id: string): void {
  try {
    sessionStorage.setItem(MOCK_RUNS_KEY, JSON.stringify(loadMockRuns().filter((r) => r.id !== id)));
  } catch {
    // storage unavailable — nothing to remove
  }
}

export async function mockRunAgentHarness(payload: AgentHarnessRunRequest): Promise<void> {
  await delay(900);
  try {
    const run: MockHarnessRun = { id: `harness-mock-${Date.now()}`, created_at: new Date().toISOString(), payload };
    sessionStorage.setItem(MOCK_RUNS_KEY, JSON.stringify([run, ...loadMockRuns()]));
  } catch {
    // storage unavailable — the run just won't appear in History
  }
  // Visible in the console so you can inspect exactly what the real backend
  // will receive once USE_HARNESS_MOCK is turned off.
  // eslint-disable-next-line no-console
  console.info('[harness mock] POST /agent-harness/run', payload);
}
