//History.tsx
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
import { MOCK_HARNESS_MODEL_NAMES, USE_HARNESS_MOCK, isMockHarnessId } from '../../api/mock/harnessMock';
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

  const modelName = (id: string) => models.find((m) => m.id === id)?.name || MOCK_HARNESS_MODEL_NAMES[id] || id;
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



















//Harnessscorecard.tsx
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
//   1. Agent scorecard   — one column per agent (Hermes vs SEMCo-Work), task
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

  // Same roll-up, grouped by the model behind the agents instead.
  const modelSums: AgentSummary[] = (data?.model_ids ?? []).map((id) => ({
    id,
    title: modelName(id),
    ...summarize((data?.cells ?? []).filter((c) => c.model_id === id)),
  }));

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
  const isBest = (row: (typeof metricRows)[number], a: AgentSummary, group: AgentSummary[] = agents) => {
    if (group.length < 2) return false;
    const own = row.value(a);
    if (own === null) return false;
    const all = group.map((x) => row.value(x)).filter((v): v is number => v !== null);
    return row.better === 'high' ? own >= Math.max(...all) : own <= Math.min(...all);
  };

  const bestModelRate = Math.max(0, ...modelSums.map((m) => m.rate));

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

      {/* ---------- Model comparison — same numbers, grouped by model ---------- */}
      {modelSums.length > 1 && (
        <section className={styles.hs__sec}>
          <div className={styles['hs__sec-head']}>
            <h3>Model comparison</h3>
            <span>
              {modelSums.map((m) => m.title).join(' vs ')}, averaged over {agents.length} agent
              {agents.length === 1 ? '' : 's'}
            </span>
          </div>

          <div
            className={styles.hs__agents}
            style={{ gridTemplateColumns: `repeat(${modelSums.length}, minmax(0, 1fr))` }}
          >
            {modelSums.map((m) => {
              const best = m.rate === bestModelRate;
              return (
                <div key={m.id} className={`${styles.hs__agent} ${best ? styles['hs__agent--best'] : ''}`}>
                  <div className={styles['hs__agent-top']}>
                    <span className={styles['hs__agent-name']}>{m.title}</span>
                    {best && (
                      <span className={styles.hs__badge}>
                        <Award size={11} /> Best success rate
                      </span>
                    )}
                  </div>
                  <div className={styles['hs__agent-rate']}>{pct(m.rate)}</div>
                  <div className={styles['hs__agent-label']}>
                    Task success rate · {m.passed.toLocaleString()} of {m.tasks.toLocaleString()} tasks
                  </div>
                  <div className={styles.hs__bar}>
                    <div className={styles['hs__bar-fill']} style={{ width: pct(m.rate) }} />
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
                  {modelSums.map((m) => (
                    <th key={m.id} className={styles.hs__num}>
                      {m.title}
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
                    {modelSums.map((m) => {
                      const v = row.value(m);
                      return (
                        <td
                          key={m.id}
                          className={`${styles.hs__num} ${isBest(row, m, modelSums) ? styles['hs__num--best'] : ''}`}
                        >
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
      )}

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



















//Harnessmock.ts
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
// Display names for the models in the seeded demo run. They are not in the
// model catalog, so History falls back to this map before showing a raw id.
export const MOCK_HARNESS_MODEL_NAMES: Record<string, string> = {
  'gpt-6-astra': 'GPT-6 Astra',
  glm: 'GLM',
};

export const HARNESS_AGENTS: { id: HarnessAdapterId; title: string; desc: string }[] = [
  { id: 'hermes', title: 'Hermes', desc: 'Lightweight tool-calling agent runtime' },
  { id: 'cowork', title: 'SEMCo-Work', desc: 'Document-focused agent (API / MCP endpoint)' },
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
    evaluation_name: 'GPT-6 Astra vs GLM · agent scorecard (sample run)',
    agents: ['hermes', 'cowork'],
    model_ids: ['gpt-6-astra', 'glm'],
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

















//Harnessresultsmock.ts
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
// stronger tool caller, SEMCo-Work as the stronger document worker, so the
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





















//Harnessresults.ts
// api/endpoints/harnessResults.ts
//
// Results for a Document-agent harness run (Workstream 2) — what History's
// detail panel shows for eval_type 'agent_harness': the agent scorecard
// (Hermes vs SEMCo-Work), per-model / per-suite breakdown, error analysis and
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

















//NewEvaluation.tsx
import { Component, useEffect, useMemo, useRef, useState, type ErrorInfo, type ReactNode } from 'react';
import { useNavigate } from 'react-router-dom';
import {
  ChevronLeft,
  ChevronRight,
  Check,
  Minus,
  Timer,
  RotateCw,
  Cpu,
  Bot,
  Database,
  Tag,
  LayoutGrid,
  Gavel,
  Layers,
  Loader2,
  Waypoints,
  Lightbulb,
  Plus,
  Upload,
  FileText,
  HeartPulse,
  ShieldCheck,
  ShieldAlert,
  RefreshCw,
  Search,
  X,
  AlertTriangle,
  Info,
  Image as ImageIcon,
  type LucideIcon,
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchProviders } from '../../store/slices/providersSlice';
import { fetchModels, checkModelHealth } from '../../store/slices/modelsSlice';
import { fetchDatasets, uploadDataset, resetUploadStatus } from '../../store/slices/datasetsSlice';
import { SUPPORTED_UPLOAD_EXTENSIONS } from '../../api/endpoints/datasets';
// `evaluationsApi.previewDataset` — GET /datasets/{id}/preview?limit={limit} —
// is a thin, one-off read used only by the Test Suite preview slider below.
// It lives in the evaluations API module (not datasets) and is called
// directly rather than round-tripped through Redux.
import { evaluationsApi } from '../../api/endpoints/evaluations';
import { fetchMetrics } from '../../store/slices/metricsSlice';
import {
  launchEvaluation,
  runAgentBenchmark,
  runAgentBenchmarkMulti,
  runAgentHarness,
  setDraft,
  setDraftType,
  setLaunching,
  setLaunchError,
} from '../../store/slices/evaluationsSlice';
// Document-agent harness (Workstream 2) — agent catalog + use-case list.
// The suites/metrics/launch calls themselves go through evaluationsApi
// (mocked behind USE_HARNESS_MOCK, see api/mock/harnessMock.ts).
import { HARNESS_AGENTS, HARNESS_USE_CASES } from '../../api/mock/harnessMock';
import type {
  CreateEvaluationRequest,
  EvaluationDraft,
  DatasetPreviewResponse,
  CustomMetric,
  ModelRetryConfig,
  Model,
  Dataset,
  HarnessAdapterId,
} from '../../types';
import styles from './NewEvaluation.module.scss';

// ─────────────────────────────────────────────────────────────────────────
// This component is built against the REAL evaluationsSlice draft shape:
//   { name, type, providers, models, dataset, subgroup, runSamplesMode,
//     runSamples, metrics, judgeModelId, agentFramework }
// `type` is lowercase: 'model' | 'agent' | 'rag' | null.
// setDraftType(type) — clears metrics, and clears agentFramework unless
// type === 'agent' (handled in the slice itself).
// runSamplesMode is 'custom' (default) or 'full' — see runSamplesControl
// below and the `launch` function for how 'full' maps to run_samples: 0.
//
// Other slice assumptions this component depends on:
//
// providersSlice / modelsSlice / datasetsSlice / metricsSlice — lazy fetch:
//   - fetchProviders() is dispatched once, the first time Step 2 is opened.
//   - fetchModels() is dispatched once, the first time Step 3 is opened.
//   - fetchDatasets(type) is dispatched the first time Step 4 is opened for
//     a given dataset type/framework combination.
//   - fetchMetrics({ evalType, useCodeExecution }) — normally GET /metrics?
//     eval_type={type}, dispatched the first time Step 5 is opened for a
//     given draft.type/dataset combination. When eval_type is 'model' and
//     the selected Test Suite dataset's name is 'Livecodebench_code_generation_lite'
//     or 'MdEval' (see CODE_EXECUTION_DATASET_NAMES), useCodeExecution is
//     true and it calls GET /code-execution-metrics instead — same
//     response shape.
//     Response: { eval_type, metrics: string[], all_metrics: string[] }.
//     Only `all_metrics` is consumed (see metricsCatalog below).
//   - None of the above fire on mount or the instant their prerequisite
//     (e.g. draft.type) is set — only on actually navigating to the step.
//     Each step's "refresh" button bypasses this and calls the thunk
//     directly, any time it's clicked.
//
// modelsSlice — health checks:
//   - checkModelHealth(modelId: string) — GET /models/health/{model_id}
//     Response: { success, message, model_id, response }
//   - state.models.healthById: Record<string, 'idle'|'loading'|'success'|'failed'>
//   - Fired automatically, in parallel, for every model in the current
//     provider selection as soon as Step 3's model list is available (see
//     the auto health-check effect below) — a model can't be selected
//     until its check resolves to 'success'. The manual "Check health"
//     button on each card still works too, e.g. to retry a failure.
//
// datasetsSlice:
//   - fetchDatasets(type: string) — GET /datasets?eval_type={type}
//     `type` is one of: 'model' | 'rag' | 'agent_benchmark' | 'agent_custom'
//     (the last two both represent draft.type === 'agent', distinguished by
//     whether draft.agentFramework is set).
//   - Dataset items carry `dataset_type` (used to detect "custom" datasets)
//     and `dataset_categories: string[]` (used for the subgroup rail).
//
// Search (client-side only, no new endpoints):
//   - Providers/Models/Test Suite/Metrics steps each get a small toolbar
//     with a search box that filters the already-fetched list by name, and
//     a refresh button that re-dispatches that step's existing fetch thunk.
// ─────────────────────────────────────────────────────────────────────────

const STEPS = [
  { label: 'Name' },
  { label: 'Type' },
  { label: 'Providers' },
  { label: 'Models' },
  { label: 'Test Suite' },
  { label: 'Metrics' },
  { label: 'Review' },
];

const STAGE = [
  { title: 'Name your run', sub: 'A recognizable name makes this run easy to find later in your history.' },
  { title: 'What are you evaluating?', sub: 'The system under test shapes which datasets and metrics you can pick.' },
  { title: 'Select providers', sub: 'Choose which connected providers to draw candidate models from.' },
  {
    title: 'Choose models',
    sub: 'Tick a model to add it. Health is checked automatically, and only models that pass can join the run.',
  },
  { title: 'Pick a test suite', sub: 'Select a dataset to evaluate against, or upload your own.' },
  { title: 'Set up scoring', sub: 'Work top to bottom. Each part shows what is still missing.' },
  { title: 'Review and launch', sub: 'This is exactly what will run. Use Edit to change any part.' },
];

// `value` matches draft.type exactly (lowercase); `label` is for display.
const TYPE_OPTIONS: {
  value: EvaluationDraft['type'];
  label: string;
  icon: LucideIcon;
  sub: string;
  variant: string;
  disabled: boolean;
}[] = [
  {
    value: 'model',
    label: 'Model',
    icon: Cpu,
    sub: 'Benchmark a general-purpose LLM on standard tasks like reasoning, coding, and knowledge — ideal for comparing raw model quality across providers.',
    variant: '',
    disabled: false,
  },
  {
    value: 'agent',
    label: 'Agent',
    icon: Bot,
    sub: 'Test an autonomous agent that plans, calls tools, and completes multi-step tasks — measures task completion, not just single-turn output.',
    variant: 'agent',
    disabled: false,
  },
  {
    value: 'rag',
    label: 'RAG',
    icon: Database,
    sub: 'Evaluate a retrieval-augmented pipeline for grounding accuracy — checks how well answers stay faithful to your retrieved context.',
    variant: 'rag',
    disabled: false,
  },
];

// Only Hermes remains as a selectable framework once "Agent" is chosen.
const AGENT_FRAMEWORKS = [
  { id: 'hermes', title: 'Hermes', desc: 'Lightweight tool-calling agent runtime' },
];

// Per-model retry/timeout config (Models step) — bounds and defaults for
// the two fields shown on each selected model card.
const MIN_MAX_RETRIES = 1;
const MAX_MAX_RETRIES = 30;
const DEFAULT_MAX_RETRIES = 1;
const MIN_TIMEOUT = 60;
const MAX_TIMEOUT = 900;
const DEFAULT_TIMEOUT = 60;
const DEFAULT_MODEL_RETRY_CONFIG: ModelRetryConfig = { max_retries: DEFAULT_MAX_RETRIES, timeout: DEFAULT_TIMEOUT };
// Pure clamp helper — used both on blur (see updateModelRetryConfig/
// updateRetryConfigAll below) and as a final safety net in `launch()` in
// case the payload is built while a field is still mid-edit (e.g. the user
// clicks "Launch" without blurring the input first).
const clampRetryField = (field: keyof ModelRetryConfig, value: number): number => {
  const min = field === 'max_retries' ? MIN_MAX_RETRIES : MIN_TIMEOUT;
  const max = field === 'max_retries' ? MAX_MAX_RETRIES : MAX_TIMEOUT;
  return Math.min(max, Math.max(min, Number.isNaN(value) ? min : value));
};
const clampModelRetryConfigValue = (config: ModelRetryConfig): ModelRetryConfig => ({
  max_retries: clampRetryField('max_retries', config.max_retries),
  timeout: clampRetryField('timeout', config.timeout),
});

// Document-agent harness settings (Metrics step) — bounds for the two
// number fields. task_timeout reuses the same 60–900s range as the
// per-model timeout, so clampRetryField('timeout', …) is used for it.
const MIN_HARNESS_STEPS = 1;
const MAX_HARNESS_STEPS = 100;
const clampHarnessSteps = (value: number): number =>
  Math.min(MAX_HARNESS_STEPS, Math.max(MIN_HARNESS_STEPS, Number.isNaN(value) ? MIN_HARNESS_STEPS : value));

// Model-only Metrics-step "Retest on Wrong" — bounds/default for Retest
// Max Rounds.
const MIN_RETEST_ROUNDS = 1;
const MAX_RETEST_ROUNDS = 30;
const DEFAULT_RETEST_ROUNDS = 3;
const clampRetestRounds = (value: number): number =>
  Math.min(MAX_RETEST_ROUNDS, Math.max(MIN_RETEST_ROUNDS, Number.isNaN(value) ? MIN_RETEST_ROUNDS : value));

// Model-type datasets whose `name` (GET /datasets?eval_type=model) selects
// GET /code-execution-metrics instead of the normal GET /metrics?eval_type=
// model for the Metrics step — same response shape, different metric set.
const CODE_EXECUTION_DATASET_NAMES = new Set(['Livecodebench_code_generation_lite', 'MdEval']);

// Preview is normally hidden for agent_benchmark (agent type, no framework
// selected — see isAgentBenchmarkNoFramework), since that endpoint (POST
// /agent-benchmark/run) doesn't use the dataset preview/question-browsing
// flow at all. These three datasets are the exception — Preview is shown
// for them even under agent_benchmark.
const AGENT_BENCHMARK_PREVIEWABLE_DATASETS = new Set(['ToolBench', 'BFCLv3', 'GAIA']);

// A preview question's `metadata` is usually flat (e.g. { level: "1",
// source: "..." }), but some datasets nest an object inside it (e.g.
// { level: "1", annotator: { steps: "...", number_of_steps: "4", tools:
// "..." } }). Rendered naively with String(value), a nested object would
// show up as the useless literal "[object Object]" — this flattens any
// such nesting into dot-notation key/value pairs (annotator.steps,
// annotator.number_of_steps, ...) so every field is still visible as its
// own chip. Arrays are joined into a single readable string rather than
// flattened by index.
function flattenMetadata(obj: Record<string, unknown>, prefix = ''): [string, string][] {
  const entries: [string, string][] = [];
  for (const [key, value] of Object.entries(obj)) {
    if (value === null || value === undefined) continue;
    const label = prefix ? `${prefix}.${key}` : key;
    if (Array.isArray(value)) {
      entries.push([
        label,
        value.map((v) => (v !== null && typeof v === 'object' ? JSON.stringify(v) : String(v))).join(', '),
      ]);
    } else if (typeof value === 'object') {
      entries.push(...flattenMetadata(value as Record<string, unknown>, label));
    } else {
      entries.push([label, String(value)]);
    }
  }
  return entries;
}

// Number inputs for retries/timeout/retest rounds: an emptied field is
// held as NaN in the draft (so backspacing to clear it shows an empty box
// instead of snapping to 0), and clamped back into range on blur and at
// launch time (clampRetryField / clampRetestRounds both map NaN -> min).
const parseNumInput = (s: string): number => (s === '' ? NaN : Number(s));
const displayNumInput = (v: number | undefined): number | string =>
  v === undefined || Number.isNaN(v) ? '' : v;

const SUGGESTED_NAMES = [
  'Q3 Model Selection',
  'Support Bot Regression',
  'RAG Accuracy v2',
  'GLM-4.6 vs Claude',
];

// Type step: what each system under test is about — shown in the "About"
// panel under the three cards for whichever one is selected.
const TYPE_ABOUT: Record<string, { tagline: string; measures: string[]; goodFor: string[]; flow: string[] }> = {
  model: {
    tagline: 'Models answer the same questions directly, with no tools and no retrieval.',
    measures: [
      'Answer quality on every question, per model',
      'Pass and fail counts for each metric you pick',
      'A side-by-side ranking with a top model',
    ],
    goodFor: [
      'Choosing between models or providers',
      'Checking a new model version against the current one',
      'Benchmarking on standard suites or your own dataset',
    ],
    flow: [
      'Every selected model answers the same set of questions.',
      'Answers are scored by your metrics, with a judge model.',
      'Models are ranked side by side in History.',
    ],
  },
  agent: {
    tagline: 'An agent plans, calls tools and works through a task over several steps.',
    measures: [
      'Whether multi-step tasks are actually completed',
      'Tool use: the right tool with the right arguments',
      'Steps, cost and time per task in the document-agent harness',
    ],
    goodFor: [
      'Testing an agent before rolling it out',
      'Comparing agents on the same tasks, such as Hermes and SEMCo-Work',
      'Comparing the models behind an agent',
    ],
    flow: [
      'The agent receives a task and plans its approach.',
      'It calls tools and works through the steps.',
      'The outcome is scored on task completion.',
    ],
  },
  rag: {
    tagline: 'A retrieval pipeline finds documents first, then an LLM answers from them.',
    measures: [
      'Whether answers stay faithful to the retrieved context',
      'How well retrieval surfaces the right documents',
      'The combined quality of embedding, reranker and LLM',
    ],
    goodFor: [
      'Comparing LLMs on the same retrieval setup',
      'Trying different Top K values',
      'Checking grounding for a knowledge assistant',
    ],
    flow: [
      'The embedding model and reranker fetch the Top K documents for each question.',
      'Each LLM answers using that retrieved context.',
      'Answers are scored for accuracy and grounding.',
    ],
  },
};

const NAMING_TIPS = [
  "Include what you're testing — a model, a product feature, or a use case.",
  'Add a date or version so you can track changes over time (e.g. "Q3", "v2").',
  'Keep it specific enough to tell apart from similar past runs later.',
];

function formatContextWindow(tokens: number | null | undefined): string {
  if (tokens === null || tokens === undefined || Number.isNaN(tokens)) return '—';
  if (tokens >= 1_000_000) return `${(tokens / 1_000_000).toLocaleString()}M`;
  if (tokens >= 1_000) return `${Math.round(tokens / 1000)}k`;
  return `${tokens}`;
}

function formatPrice(price: number | null | undefined): string {
  return price === null || price === undefined || Number.isNaN(price) ? '—' : `$${price.toFixed(2)}`;
}

function providerInitials(name: string | null | undefined): string {
  const safeName = name?.trim() || '';
  if (!safeName) return '??';
  const parts = safeName.replace(/[^a-zA-Z0-9 ]/g, '').split(' ').filter(Boolean);
  const letters = parts.slice(0, 2).map((w) => w[0]).join('');
  return (letters || safeName.slice(0, 2)).toUpperCase();
}

type HealthStatus = 'idle' | 'loading' | 'success' | 'failed';

// Pulls a displayable message out of a rejected createAsyncThunk action,
// regardless of whether that slice uses rejectWithValue (payload is a
// string, or an object with a `message`) or lets RTK's default serializer
// handle it (action.error.message).
function getThunkErrorMessage(action: any, fallback: string): string {
  const payload = action?.payload;
  if (typeof payload === 'string' && payload) return payload;
  if (payload && typeof payload === 'object' && typeof payload.message === 'string' && payload.message) {
    return payload.message;
  }
  return action?.error?.message || fallback;
}

// Preview slider (Change-2): user picks how many sample questions to pull,
// clamped server-side-friendly at 1–20 inclusive.
// Preview slider (Change-2, later moved to offset-based pagination): fixed
// page size of 20 questions per page, starting at offset 0. Total page
// count is derived from the selected dataset's own `question_count`
// (surfaced in the /datasets list) rather than from the preview response.
const PREVIEW_PAGE_SIZE = 20;

type DatasetTypeFilter = 'all' | 'custom' | 'deepeval';

const DATASET_TYPE_FILTERS: { value: DatasetTypeFilter; label: string }[] = [
  { value: 'all', label: 'All' },
  { value: 'custom', label: 'Custom' },
  { value: 'deepeval', label: 'Deepeval' },
];

// Skeleton placeholder counts — just enough to plausibly fill the grid
// while a refresh is in flight, not meant to match the real result count.
const SKELETON_CARD_COUNT = 6;
const SKELETON_CHIP_COUNT = 8;

// ---------------------------------------------------------------------------
// StepErrorBoundary — catches render-time exceptions thrown while rendering
// a single wizard step (e.g. an unexpected null/undefined field from the
// API) and shows a small recoverable card in place of just that step's
// content, instead of the whole page going blank. Must be a class component
// — React only supports error boundaries via getDerivedStateFromError /
// componentDidCatch, there's no hook equivalent.
//
// Rendered with `key={step}-${retryKey}` by the caller, so navigating to a
// different step (or clicking "Try again", which bumps retryKey) always
// remounts a fresh instance with hasError reset — no manual reset wiring
// needed here.
// ---------------------------------------------------------------------------
interface StepErrorBoundaryProps {
  children: ReactNode;
  onRetry: () => void;
  onBack: () => void;
  canGoBack: boolean;
}
interface StepErrorBoundaryState {
  hasError: boolean;
}
class StepErrorBoundary extends Component<StepErrorBoundaryProps, StepErrorBoundaryState> {
  state: StepErrorBoundaryState = { hasError: false };

  static getDerivedStateFromError(): StepErrorBoundaryState {
    return { hasError: true };
  }

  componentDidCatch(error: Error, info: ErrorInfo) {
    // eslint-disable-next-line no-console
    console.error('[NewEvaluation] step failed to render:', error, info.componentStack);
  }

  render() {
    if (!this.state.hasError) return this.props.children;
    return (
      <div className={styles.ev__error} style={{ flexDirection: 'column', alignItems: 'flex-start', gap: 10 }}>
        <div style={{ display: 'flex', alignItems: 'center', gap: 8 }}>
          <AlertTriangle size={16} />
          <strong>This step hit an unexpected error.</strong>
        </div>
        <p style={{ margin: 0, fontWeight: 400 }}>
          Your progress on earlier steps is safe. Try again, or go back and retry from there.
        </p>
        <div style={{ display: 'flex', gap: 10 }}>
          <button type="button" className={`${styles.ev__btn} ${styles['ev__btn--primary']}`} onClick={this.props.onRetry}>
            Try again
          </button>
          {this.props.canGoBack && (
            <button type="button" className={`${styles.ev__btn} ${styles['ev__btn--ghost']}`} onClick={this.props.onBack}>
              Back
            </button>
          )}
        </div>
      </div>
    );
  }
}

export default function NewEvaluation() {
  const dispatch = useAppDispatch();
  const navigate = useNavigate();
  const [step, setStep] = useState(0);
  // Bumped by StepErrorBoundary's "Try again" button to force a fresh
  // remount of the current step's content without changing `step` itself.
  const [stepRetryKey, setStepRetryKey] = useState(0);
  const [toast, setToast] = useState(false);
  const [datasetTab, setDatasetTab] = useState<'browse' | 'upload'>('browse');
  const [uploadName, setUploadName] = useState('');
  const [uploadDescription, setUploadDescription] = useState('');
  const [uploadFile, setUploadFile] = useState<File | null>(null);
  const [uploadFileError, setUploadFileError] = useState<string | null>(null);
  const totalSteps = STEPS.length;

  // ---- search (client-side filter) + refresh (re-fetch) state, one pair
  // per searchable step: Providers, Models, Test Suite, Metrics. ----------
  const [providerSearch, setProviderSearch] = useState('');
  const [modelSearch, setModelSearch] = useState('');
  // Models table: view-only provider filter (All / one provider). Doesn't
  // change draft.providers — that's still set on the Providers step.
  const [modelProviderFilter, setModelProviderFilter] = useState<string>('all');
  // RAG-only: one independent search box per category (Embedding/
  // Reranker/LLM) instead of the single global model search used by the
  // Model/Agent grid — see RAG_CATEGORIES below.
  const [ragCategorySearch, setRagCategorySearch] = useState<Record<string, string>>({
    Embedding: '',
    Reranker: '',
    LLM: '',
  });
  const [judgeSearch, setJudgeSearch] = useState('');
  const [datasetSearch, setDatasetSearch] = useState('');
  const [metricSearch, setMetricSearch] = useState('');

  const [providersLoading, setProvidersLoading] = useState(false);
  const [modelsLoading, setModelsLoading] = useState(false);
  // Which models have had a health check *complete during this visit* to
  // the page, and whether the model list itself has been (re)fetched this
  // visit. Redux keeps healthById/models/draft.models between visits, so a
  // model can come back already marked 'success' from last time — these
  // make sure Step 3 can't be continued on that stale result while this
  // visit's re-check/refetch is still in flight (see canGo / selectedModels
  // NotReady).
  const [verifiedModelIds, setVerifiedModelIds] = useState<Record<string, true>>({});
  const [modelsFetchedThisVisit, setModelsFetchedThisVisit] = useState(false);
  const [datasetsRefreshing, setDatasetsRefreshing] = useState(false);
  const [metricsRefreshing, setMetricsRefreshing] = useState(false);
  // Tracked locally rather than read from the datasets slice's own `error`
  // field — that field wasn't reliably cleared on a subsequent successful
  // fetch, so a stale error from an earlier failed attempt kept showing
  // even after Refresh (or renavigating) succeeded. Deriving this purely
  // from the outcome of our own dispatch calls below fixes that.
  const [datasetsErrorLocal, setDatasetsErrorLocal] = useState<string | null>(null);

  // ---- Test Suite: All/Custom/Deepeval filter (Change-1) -----------------
  const [datasetTypeFilter, setDatasetTypeFilter] = useState<DatasetTypeFilter>('all');

  // ---- Test Suite (Document-agent harness mode) --------------------------
  // The harness suite pool is fetched straight from evaluationsApi
  // .getHarnessSuites() (mocked for now) into local state, rather than
  // through the datasets slice — keeps the existing dataset thunks/reducers
  // completely untouched. `useCaseFilter` replaces the All/Custom/Deepeval
  // chips in that mode.
  const [harnessSuites, setHarnessSuites] = useState<Dataset[]>([]);
  const [useCaseFilter, setUseCaseFilter] = useState<string>('all');

  // ---- Test Suite: preview slider (Change-2, now paginated) --------------
  const [previewOpen, setPreviewOpen] = useState(false);
  const [previewDatasetId, setPreviewDatasetId] = useState<string | null>(null);
  const [previewDatasetName, setPreviewDatasetName] = useState<string>('');
  // Total known question count for the dataset being previewed — comes
  // from the dataset card (Dataset.question_count), not from the preview
  // response itself. Used only to compute how many pages to offer.
  const [previewQuestionCount, setPreviewQuestionCount] = useState(0);
  const [previewOffset, setPreviewOffset] = useState(0);
  const [previewLoading, setPreviewLoading] = useState(false);
  const [previewError, setPreviewError] = useState<string | null>(null);
  const [previewData, setPreviewData] = useState<DatasetPreviewResponse | null>(null);

  // ---- Test Suite: image lightbox (MinIO-backed question images) --------
  // A question's input.images[] holds MinIO object keys, not usable
  // directly as <img src> — each one is resolved to a blob URL on demand
  // via evaluationsApi.getImageBlobUrl (POST /datasets/image) when its
  // thumbnail button is clicked, mirroring MinIO's own object-preview UI.
  const [imagePreviewOpen, setImagePreviewOpen] = useState(false);
  const [imagePreviewPaths, setImagePreviewPaths] = useState<string[]>([]);
  const [imagePreviewIndex, setImagePreviewIndex] = useState(0);
  const [imagePreviewUrl, setImagePreviewUrl] = useState<string | null>(null);
  const [imagePreviewLoading, setImagePreviewLoading] = useState(false);
  const [imagePreviewError, setImagePreviewError] = useState<string | null>(null);

  // ---- Metrics step: "Generate Instruction" ------------------------------
  const [instructionGenerating, setInstructionGenerating] = useState(false);
  const [instructionError, setInstructionError] = useState<string | null>(null);

  const draft = useAppSelector((s) => s.evaluations.draft);
  const launching = useAppSelector((s) => s.evaluations.launching);
  const launchError = useAppSelector((s) => s.evaluations.launchError);

  const providers = useAppSelector((s) => s.providers.items) ?? [];
  const models = useAppSelector((s) => s.models.items) ?? [];
  const healthById = useAppSelector((s) => (s.models as any).healthById) as Record<string, HealthStatus> | undefined;

  const metricsState = useAppSelector((s) => s.metrics) ?? { allMetrics: [], customMetrics: [], status: 'idle' as const, error: null };
  // Only `all_metrics` from the API response is used for the built-in
  // catalog — it's the full list rendered as selectable chips. Loading
  // state for this step is tracked locally (metricsRefreshing) rather than
  // read from metricsState.status, for the same staleness reason as
  // datasets below.
  const metricsCatalog: string[] = (metricsState as any).allMetrics ?? [];
  // `custom` from the same response, normalized by metricsSlice into
  // state.metrics.customMetrics — richer objects (id/name/description/
  // required_judge/etc.), rendered as their own section below the built-in
  // metrics chips. Selection stores the metric's `id` (not its `name`) in
  // draft.metrics, alongside the plain metric-name strings from
  // metricsCatalog — both end up in the same selected_metrics array on
  // launch, the backend only cares that each entry resolves to a metric.
  const customMetricsCatalog: CustomMetric[] = (metricsState as any).customMetrics ?? [];
  const customMetricsForType = useMemo(
    () => customMetricsCatalog.filter((c) => !Array.isArray(c?.eval_types) || c.eval_types.length === 0 || (draft.type && c.eval_types.includes(draft.type))),
    [customMetricsCatalog, draft.type]
  );

  // Document-agent harness: Agent type + the "Document-agent harness" mode
  // switch on the Type step. Mutually exclusive with agentFramework. Every
  // harness-specific branch below is gated on this, so Model / RAG / Agent
  // (benchmark) / Agent (Hermes) behave exactly as before.
  const isHarness = draft.type === 'agent' && draft.agentHarness;

  const storeDatasets = useAppSelector((s) => s.datasets.items) ?? [];
  // In harness mode the Test Suite pool comes from the harness catalog
  // (local state) instead of the datasets slice.
  const datasets: Dataset[] = isHarness ? harnessSuites : (storeDatasets as Dataset[]);
  const datasetUploading = useAppSelector((s) => s.datasets.uploadStatus === 'loading');
  const datasetUploadError = useAppSelector((s) => s.datasets.uploadError);

  // ---- lazy fetch guards: each API is only called the first time the
  // user actually navigates to the step that needs it, not on mount and
  // not the moment its prerequisite (e.g. draft.type) is set. Refresh
  // buttons bypass these guards entirely (they call the thunk directly).
  const providersFetchedRef = useRef(false);
  const modelsFetchedRef = useRef(false);
  const datasetsFetchedForTypeRef = useRef<string | null>(null);
  const metricsFetchedForTypeRef = useRef<string | null>(null);

  // GET /providers — fetched once, the first time Step 2 is reached. If
  // that fetch fails, the marker is rolled back so leaving and coming back
  // to Step 2 retries it automatically, instead of silently reusing a
  // failed attempt forever (the Refresh button already bypasses this).
  useEffect(() => {
    if (step !== 2 || providersFetchedRef.current) return;
    providersFetchedRef.current = true;
    (async () => {
      setProvidersLoading(true);
      const result = await dispatch(fetchProviders());
      setProvidersLoading(false);
      if (fetchProviders.rejected.match(result)) {
        providersFetchedRef.current = false;
      }
    })();
  }, [step, dispatch]);

  // GET /models — fetched once, the first time Step 3 is reached (by then
  // providers are already selected, since Step 2 requires it to advance).
  // Same retry-on-failure behavior as providers above.
  useEffect(() => {
    if (step !== 3 || modelsFetchedRef.current) return;
    modelsFetchedRef.current = true;
    (async () => {
      setModelsLoading(true);
      const result = await dispatch(fetchModels());
      setModelsLoading(false);
      if (fetchModels.rejected.match(result)) {
        modelsFetchedRef.current = false;
      } else {
        setModelsFetchedThisVisit(true);
      }
    })();
  }, [step, dispatch]);

  // ---- (1) dataset "type" query param, split for Agent by framework -------
  // Model/RAG: type = draft.type
  // Agent, no framework chosen:  type = 'agent_benchmark'
  // Agent, framework chosen:     type = 'agent_custom'
  const datasetType = useMemo(() => {
    if (!draft.type) return '';
    if (draft.type === 'agent') {
      if (draft.agentHarness) return 'agent_harness';
      return draft.agentFramework ? 'agent_custom' : 'agent_benchmark';
    }
    return draft.type;
  }, [draft.type, draft.agentFramework, draft.agentHarness]);

  // GET /datasets?eval_type={type} — fetched the first time Step 4 is reached
  // for a given type/framework combination. If the user goes back and
  // changes type/framework, the pool is stale, so a different datasetType
  // triggers one refetch the next time Step 4 is (re)entered. If the fetch
  // itself fails, the marker is rolled back so re-selecting the *same*
  // type and coming back to Step 4 retries automatically — previously a
  // failed attempt permanently "used up" the marker for that type, so
  // going back and forward again silently reused the failure.
  useEffect(() => {
    if (step !== 4 || !datasetType) return;
    if (datasetsFetchedForTypeRef.current === datasetType) return;
    datasetsFetchedForTypeRef.current = datasetType;
    (async () => {
      setDatasetsRefreshing(true);

      // Document-agent harness: suites come from evaluationsApi
      // .getHarnessSuites() (mocked), not from the datasets slice.
      if (datasetType === 'agent_harness') {
        try {
          const items = await evaluationsApi.getHarnessSuites();
          setHarnessSuites(items);
          setDatasetsErrorLocal(null);
        } catch (err) {
          datasetsFetchedForTypeRef.current = null;
          setDatasetsErrorLocal((err as Error)?.message || 'Failed to load test suites');
        } finally {
          setDatasetsRefreshing(false);
        }
        return;
      }

      const result = await dispatch(fetchDatasets(datasetType));
      setDatasetsRefreshing(false);
      if (fetchDatasets.rejected.match(result)) {
        datasetsFetchedForTypeRef.current = null;
        setDatasetsErrorLocal(getThunkErrorMessage(result, 'Failed to load test suites'));
      } else {
        setDatasetsErrorLocal(null);
      }
    })();
  }, [step, datasetType, dispatch]);

  // Any previously chosen dataset is invalid once the dataset "type" changes
  // (different type/framework combination = different dataset pool). Also
  // clear any in-progress dataset search since it applied to the old pool.
  // This is a pure UI-state reset, independent of the fetch timing above.
  useEffect(() => {
    if (!datasetType) return;
    dispatch(setDraft({ dataset: null, harnessSuiteIds: [] }));
    setDatasetSearch('');
    setDatasetTypeFilter('all');
    setUseCaseFilter('all');
    setDatasetsErrorLocal(null);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [datasetType]);

  const suite = datasets.find((d) => d?.id === draft.dataset);

  // Model type + one of the two code-execution built-in datasets selected
  // in the Test Suite step -> Metrics step calls GET /code-execution-
  // metrics instead of the normal GET /metrics?eval_type=model.
  const isCodeExecutionDataset =
    draft.type === 'model' && Boolean(suite?.name) && CODE_EXECUTION_DATASET_NAMES.has(suite!.name);

  // GET /metrics?eval_type={type} (or GET /code-execution-metrics — see
  // isCodeExecutionDataset above) — fetched the first time Step 5 is
  // reached for a given draft.type/dataset combination. Only `all_metrics`
  // is consumed (see metricsCatalog below) — `metrics` is ignored
  // entirely. Same retry-on-failure rollback as datasets above.
  //
  // The cache key includes isCodeExecutionDataset (not just draft.type) —
  // switching between a code-execution dataset and a regular one without
  // changing draft.type would otherwise be treated as "already fetched"
  // and silently keep showing the wrong metric set.
  //
  // Harness mode fetches its own scorer list ('agent_harness' — metricsSlice
  // routes that to the harness endpoint, mocked for now), so it's part of
  // the key too.
  const metricsEvalType = isHarness ? 'agent_harness' : draft.type;
  const metricsCacheKey = `${metricsEvalType ?? ''}:${isCodeExecutionDataset}`;
  useEffect(() => {
    if (step !== 5 || !draft.type || !metricsEvalType) return;
    if (metricsFetchedForTypeRef.current === metricsCacheKey) return;
    metricsFetchedForTypeRef.current = metricsCacheKey;
    (async () => {
      setMetricsRefreshing(true);
      const result = await dispatch(fetchMetrics({ evalType: metricsEvalType, useCodeExecution: isCodeExecutionDataset }));
      setMetricsRefreshing(false);
      if (fetchMetrics.rejected.match(result)) {
        metricsFetchedForTypeRef.current = null;
      }
    })();
  }, [step, draft.type, metricsEvalType, metricsCacheKey, dispatch]);

  // ---- (6) auto-select every subgroup on dataset pick ----------------------
  useEffect(() => {
    const cats = (suite as any)?.dataset_categories ?? [];
    dispatch(setDraft({ subgroup: cats }));
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [draft.dataset]);

  const selectAllSubgroups = () => dispatch(setDraft({ subgroup: (suite as any)?.dataset_categories ?? [] }));
  const clearAllSubgroups = () => dispatch(setDraft({ subgroup: [] }));

  // ---- Run Samples default/max, driven by the selected dataset's size ----
  // More than 10 questions available: default to 10, cap the input at the
  // dataset's total. 10 or fewer available: default (and cap) at the
  // dataset's total, since there's nothing more to sample.
  //
  // Harness mode: Run Samples applies per suite, so the cap is the largest
  // selected suite (a smaller suite simply runs in full). The suite key
  // below re-derives the default whenever the set of picked suites changes.
  const selectedHarnessSuites = useMemo(
    () => harnessSuites.filter((s) => draft.harnessSuiteIds.includes(s.id)),
    [harnessSuites, draft.harnessSuiteIds]
  );
  const selectedDatasetQuestionCount = isHarness
    ? Math.max(0, ...selectedHarnessSuites.map((s) => s.question_count ?? 0))
    : suite?.question_count ?? 0;
  const maxRunSamples = Math.max(1, selectedDatasetQuestionCount);
  const defaultRunSamples = selectedDatasetQuestionCount > 10 ? 10 : maxRunSamples;
  const harnessSuiteKey = draft.harnessSuiteIds.join(',');

  // Re-derive the default whenever the selected dataset itself changes
  // (not on every render) — matches the same trigger as the subgroup
  // auto-select effect above, and won't stomp on a value the user typed
  // in manually unless they've actually picked a different dataset since.
  useEffect(() => {
    if (isHarness ? draft.harnessSuiteIds.length === 0 : !draft.dataset) return;
    dispatch(setDraft({ runSamples: defaultRunSamples }));
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [draft.dataset, harnessSuiteKey]);

  const connectedProviders = providers.filter((p) => p?.status === 'connected');
  const filteredProviders = useMemo(
    () => connectedProviders.filter((p) => (p?.name ?? '').toLowerCase().includes(providerSearch.trim().toLowerCase())),
    [connectedProviders, providerSearch]
  );

  const availableModels = useMemo(
    () => models.filter((m) => draft.providers.includes(m?.provider_id)),
    [models, draft.providers]
  );
  // Model/Agent grid: LLM-category models only — Embedding and Reranker
  // models are only meaningful for RAG (which groups all three categories
  // into its own columns via modelsByCategory below, not through this).
  const filteredModels = useMemo(
    () =>
      availableModels
        .filter((m) => (m?.category ?? '').toLowerCase() === 'llm')
        .filter((m) => (m?.name ?? '').toLowerCase().includes(modelSearch.trim().toLowerCase())),
    [availableModels, modelSearch]
  );

  // RAG-only: exactly three model categories are possible — Embedding,
  // Reranker, LLM — always shown in that order (see STEP 3 — MODELS
  // below), regardless of what order the API returns models in.
  // modelsByCategory (full, unfiltered) drives category membership and
  // the per-category validation in canGo() below; filteredModelsByCategory
  // applies each column's own search box (ragCategorySearch) on top of
  // that — independent per column, unlike the Model/Agent grid's single
  // shared modelSearch.
  const RAG_CATEGORIES = ['Embedding', 'Reranker', 'LLM'] as const;
  const modelsByCategory = useMemo(() => {
    const map = new Map<string, Model[]>();
    RAG_CATEGORIES.forEach((cat) => map.set(cat, []));
    for (const m of availableModels) {
      if (m?.category && map.has(m.category)) map.get(m.category)!.push(m);
    }
    return map;
  }, [availableModels]);
  const filteredModelsByCategory = useMemo(() => {
    const map = new Map<string, Model[]>();
    RAG_CATEGORIES.forEach((cat) => {
      const term = (ragCategorySearch[cat] ?? '').trim().toLowerCase();
      map.set(cat, (modelsByCategory.get(cat) ?? []).filter((m) => (m?.name ?? '').toLowerCase().includes(term)));
    });
    return map;
  }, [modelsByCategory, ragCategorySearch]);

  // Counts per dataset_type, for the filter chip labels — computed off the
  // full (unsearched, unfiltered) list so the counts don't shift as the
  // user types in the search box.
  const datasetTypeCounts = useMemo(
    () => ({
      all: datasets.length,
      custom: datasets.filter((d) => (d as any)?.dataset_type === 'custom').length,
      deepeval: datasets.filter((d) => (d as any)?.dataset_type === 'deepeval').length,
    }),
    [datasets]
  );

  const filteredDatasets = useMemo(
    () =>
      datasets.filter((d) => {
        const matchesSearch = (d?.name ?? '').toLowerCase().includes(datasetSearch.trim().toLowerCase());
        // Harness mode filters by use case; every other mode by dataset_type.
        const matchesType = isHarness
          ? useCaseFilter === 'all' || d?.use_case === useCaseFilter
          : datasetTypeFilter === 'all' || (d as any)?.dataset_type === datasetTypeFilter;
        return matchesSearch && matchesType;
      }),
    [datasets, datasetSearch, datasetTypeFilter, useCaseFilter, isHarness]
  );

  // Harness suite selection: multi-select (harnessSuiteIds) instead of the
  // single draft.dataset used everywhere else.
  const isSuiteOn = (id: string) => (isHarness ? draft.harnessSuiteIds.includes(id) : draft.dataset === id);
  const pickSuite = (id: string) =>
    isHarness
      ? dispatch(setDraft({ harnessSuiteIds: toggle(draft.harnessSuiteIds, id) }))
      : dispatch(setDraft({ dataset: id }));

  // Use cases actually present in the fetched pool, in the spec's order.
  const harnessUseCases = useMemo(
    () => HARNESS_USE_CASES.filter((u) => harnessSuites.some((s) => s.use_case === u)),
    [harnessSuites]
  );

  const filteredMetrics = useMemo(
    () => metricsCatalog.filter((m) => (m ?? '').toLowerCase().includes(metricSearch.trim().toLowerCase())),
    [metricsCatalog, metricSearch]
  );

  const filteredCustomMetrics = useMemo(
    () => customMetricsForType.filter((c) => (c?.name ?? '').toLowerCase().includes(metricSearch.trim().toLowerCase())),
    [customMetricsForType, metricSearch]
  );

  // ---- refresh handlers: re-dispatch each step's existing fetch thunk ----
  // Each sets its own `*Refreshing` flag so the step can swap its grid for
  // skeleton placeholders while the request is in flight (Change-3).
  const refreshProviders = async () => {
    setProvidersLoading(true);
    await dispatch(fetchProviders());
    setProvidersLoading(false);
  };

  const refreshModels = async () => {
    setModelsLoading(true);
    const result = await dispatch(fetchModels());
    setModelsLoading(false);
    if (!fetchModels.rejected.match(result)) setModelsFetchedThisVisit(true);
  };

  const refreshDatasets = async () => {
    if (!datasetType) return;
    setDatasetsRefreshing(true);

    if (datasetType === 'agent_harness') {
      try {
        setHarnessSuites(await evaluationsApi.getHarnessSuites());
        datasetsFetchedForTypeRef.current = datasetType;
        setDatasetsErrorLocal(null);
      } catch (err) {
        datasetsFetchedForTypeRef.current = null;
        setDatasetsErrorLocal((err as Error)?.message || 'Failed to load test suites');
      } finally {
        setDatasetsRefreshing(false);
      }
      return;
    }

    const result = await dispatch(fetchDatasets(datasetType));
    setDatasetsRefreshing(false);
    if (fetchDatasets.rejected.match(result)) {
      datasetsFetchedForTypeRef.current = null;
      setDatasetsErrorLocal(getThunkErrorMessage(result, 'Failed to load test suites'));
    } else {
      datasetsFetchedForTypeRef.current = datasetType;
      setDatasetsErrorLocal(null);
    }
  };

  const refreshMetrics = async () => {
    if (!draft.type || !metricsEvalType) return;
    setMetricsRefreshing(true);
    const result = await dispatch(fetchMetrics({ evalType: metricsEvalType, useCodeExecution: isCodeExecutionDataset }));
    setMetricsRefreshing(false);
    if (fetchMetrics.rejected.match(result)) {
      metricsFetchedForTypeRef.current = null;
    } else {
      metricsFetchedForTypeRef.current = metricsCacheKey;
    }
  };

  // ---- preview slider (Change-2, paginated): GET /datasets/{id}/preview?
  // limit=20&offset={offset} ---------------------------------------------
  const previewTotalPages = Math.max(1, Math.ceil(previewQuestionCount / PREVIEW_PAGE_SIZE));
  const previewCurrentPage = Math.floor(previewOffset / PREVIEW_PAGE_SIZE) + 1;
  const previewHasPrevPage = previewOffset > 0;
  const previewHasNextPage = previewCurrentPage < previewTotalPages;

  const openPreview = (datasetId: string, datasetName: string, questionCount: number) => {
    setPreviewDatasetId(datasetId);
    setPreviewDatasetName(datasetName);
    setPreviewQuestionCount(questionCount || 0);
    setPreviewOffset(0);
    setPreviewData(null);
    setPreviewError(null);
    setPreviewOpen(true);
  };

  const closePreview = () => {
    setPreviewOpen(false);
  };

  const goToPreviewPrevPage = () => {
    setPreviewOffset((o) => Math.max(0, o - PREVIEW_PAGE_SIZE));
  };

  const goToPreviewNextPage = () => {
    setPreviewOffset((o) => {
      const maxOffset = Math.max(0, (previewTotalPages - 1) * PREVIEW_PAGE_SIZE);
      return Math.min(maxOffset, o + PREVIEW_PAGE_SIZE);
    });
  };

  // Fetches whatever page `previewOffset` currently points at. Called both
  // by the auto-fetch effect below (on open / page change) and by the
  // manual reload button (same page, fresh data).
  const fetchPreviewPage = async (datasetId: string, offset: number) => {
    setPreviewLoading(true);
    setPreviewError(null);
    try {
      const data = await evaluationsApi.previewDataset(datasetId, PREVIEW_PAGE_SIZE, offset);
      setPreviewData(data);
    } catch (err) {
      const detail =
        (err as { response?: { data?: { detail?: string } } })?.response?.data?.detail ||
        (err as Error)?.message ||
        'Failed to load preview';
      setPreviewError(detail);
    } finally {
      setPreviewLoading(false);
    }
  };

  // Auto-loads a page whenever the slider opens or the page changes — no
  // separate "Load preview" click needed, Prev/Next just work.
  useEffect(() => {
    if (!previewOpen || !previewDatasetId) return;
    fetchPreviewPage(previewDatasetId, previewOffset);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [previewOpen, previewDatasetId, previewOffset]);

  // ---- Test Suite: image lightbox (MinIO-backed question images) --------
  const openImagePreview = (paths: string[], index: number) => {
    setImagePreviewPaths(paths);
    setImagePreviewIndex(index);
    setImagePreviewOpen(true);
  };

  const closeImagePreview = () => {
    setImagePreviewOpen(false);
    setImagePreviewError(null);
  };

  const showPrevImage = () => setImagePreviewIndex((i) => Math.max(0, i - 1));
  const showNextImage = () =>
    setImagePreviewIndex((i) => Math.min(imagePreviewPaths.length - 1, i + 1));

  // Resolves the object key at the current index to a blob URL whenever
  // the lightbox opens or the user navigates between images. Cleans up
  // (revokes) the previous blob URL both between images and on close, so
  // an abandoned in-flight request never leaks — or overwrites — the URL
  // for whatever image is now showing.
  useEffect(() => {
    if (!imagePreviewOpen) {
      setImagePreviewUrl(null);
      return;
    }
    const path = imagePreviewPaths[imagePreviewIndex];
    if (!path) return;

    let cancelled = false;
    let objectUrl: string | null = null;
    setImagePreviewLoading(true);
    setImagePreviewError(null);
    setImagePreviewUrl(null);

    evaluationsApi
      .getImageBlobUrl(path)
      .then((url) => {
        if (cancelled) {
          URL.revokeObjectURL(url);
          return;
        }
        objectUrl = url;
        setImagePreviewUrl(url);
      })
      .catch((err) => {
        if (cancelled) return;
        const detail =
          (err as { response?: { data?: { detail?: string } } })?.response?.data?.detail ||
          (err as Error)?.message ||
          'Failed to load image';
        setImagePreviewError(detail);
      })
      .finally(() => {
        if (!cancelled) setImagePreviewLoading(false);
      });

    return () => {
      cancelled = true;
      if (objectUrl) URL.revokeObjectURL(objectUrl);
    };
  }, [imagePreviewOpen, imagePreviewIndex, imagePreviewPaths]);

  // ---- Metrics step: "Generate Instruction" -------------------------------
  // POST /evaluations/generate-instruction — optional, pre-fills the
  // instruction textarea; the user can edit or overwrite it freely
  // afterward either way.
  const generateInstruction = async () => {
    if (!draft.type || !draft.dataset || draft.models.length === 0) return;

    setInstructionGenerating(true);
    setInstructionError(null);
    try {
      // 1. Model with the largest context window among the ones selected
      // in Step 3 — not necessarily the judge model, just whichever
      // selected model can hold the most context for this call.
      const selectedModelObjs = draft.models
        .map((id) => models.find((m) => m?.id === id))
        .filter(Boolean) as typeof models;
      let bestModel: (typeof models)[number] | null = null;
      for (const m of selectedModelObjs) {
        const mCtx = m.context_window ?? 0;
        const bestCtx = bestModel?.context_window ?? -1;
        if (mCtx > bestCtx) bestModel = m;
      }
      if (!bestModel) {
        setInstructionError('None of the selected models could be found — try reselecting them in Step 4.');
        return;
      }

      // 2. Always a fresh 5-question, offset-0 sample — independent of
      // whatever page the preview slider (if ever opened) last showed.
      const sample = await evaluationsApi.previewDataset(draft.dataset, 5, 0);
      const questions = (Array.isArray(sample?.questions) ? sample.questions : []).map((q) => ({
        input: q?.input,
        expected: q?.expected,
      }));

      // 3. eval_type is just draft.type, chosen back in Step 2.
      const response = await evaluationsApi.generateInstruction({
        model_id: bestModel.id,
        eval_type: draft.type,
        questions,
      });
      dispatch(setDraft({ instruction: response.instruction || '' }));
    } catch (err) {
      const detail =
        (err as { response?: { data?: { detail?: string } } })?.response?.data?.detail ||
        (err as Error)?.message ||
        'Failed to generate an instruction';
      setInstructionError(detail);
    } finally {
      setInstructionGenerating(false);
    }
  };

  // Manual, single-model health check — still available via the "Check
  // health" button on each card, alongside the automatic parallel check
  // below.
  // Every health check goes through here so we know when one has actually
  // *completed this visit* (verifiedModelIds) — a stale 'success' left in
  // Redux from an earlier visit doesn't count until it's re-confirmed.
  const trackedHealthCheck = (modelId: string) => {
    setVerifiedModelIds((prev) => {
      if (!prev[modelId]) return prev;
      const next = { ...prev };
      delete next[modelId];
      return next;
    });
    Promise.resolve(dispatch(checkModelHealth(modelId))).then(() =>
      setVerifiedModelIds((prev) => ({ ...prev, [modelId]: true }))
    );
  };

  const runHealthCheck = (modelId: string) => {
    trackedHealthCheck(modelId);
  };

  // Auto health-check: as soon as the models list for the currently
  // selected providers is available (and while Step 3 is open), fire a
  // parallel health check for every visible model — the user isn't
  // expected to click "Check health" one by one. Guarded on the raw
  // `models` array reference so it re-runs once per fresh fetch (initial
  // navigation-triggered fetch, or a manual refresh) rather than on every
  // render or on unrelated state changes like typing in the search box.
  const autoHealthCheckedForModelsRef = useRef<typeof models | null>(null);
  const autoHealthCheckedIdsRef = useRef<Set<string>>(new Set());
  useEffect(() => {
    if (step !== 3 || availableModels.length === 0) return;
    if (autoHealthCheckedForModelsRef.current !== models) {
      autoHealthCheckedForModelsRef.current = models;
      autoHealthCheckedIdsRef.current = new Set();
    }
    availableModels.forEach((m) => {
      if (autoHealthCheckedIdsRef.current.has(m.id)) return;
      autoHealthCheckedIdsRef.current.add(m.id);
      trackedHealthCheck(m.id);
    });
  }, [step, models, availableModels, dispatch]);

  const toggle = (list: string[], value: string) =>
    list.includes(value) ? list.filter((v) => v !== value) : [...list, value];

  // Built-in metrics (plain name strings) and custom metrics (ids) share
  // the same draft.metrics array, so "select all" / "clear" for one
  // section must only touch its own entries — otherwise selecting all
  // built-in metrics would silently wipe out any chosen custom metrics
  // (and vice versa).
  const customMetricIdSet = new Set(customMetricsForType.map((c) => c.id));
  const selectAllBuiltinMetrics = () => {
    const keptCustom = draft.metrics.filter((m) => customMetricIdSet.has(m));
    dispatch(setDraft({ metrics: [...keptCustom, ...metricsCatalog] }));
  };
  const clearBuiltinMetrics = () => {
    dispatch(setDraft({ metrics: draft.metrics.filter((m) => customMetricIdSet.has(m)) }));
  };
  const selectAllCustomMetrics = () => {
    const builtinSet = new Set(metricsCatalog);
    const keptBuiltin = draft.metrics.filter((m) => builtinSet.has(m));
    dispatch(setDraft({ metrics: [...keptBuiltin, ...customMetricsForType.map((c) => c.id)] }));
  };
  const clearCustomMetrics = () => {
    dispatch(setDraft({ metrics: draft.metrics.filter((m) => !customMetricIdSet.has(m)) }));
  };

  // Model-only "Retest on Wrong" — same free-typing-then-clamp-on-blur
  // pattern as the Models step's retry/timeout inputs, so clearing/
  // retyping the field isn't fought by immediate clamping.
  const updateRetestMaxRounds = (raw: number) => {
    dispatch(setDraft({ retestMaxRounds: raw }));
  };
  const clampRetestMaxRoundsOnBlur = () => {
    const clamped = clampRetestRounds(draft.retestMaxRounds);
    if (clamped !== draft.retestMaxRounds) dispatch(setDraft({ retestMaxRounds: clamped }));
  };

  const getFileExtension = (filename: string) => {
    const idx = filename.lastIndexOf('.');
    return idx >= 0 ? filename.slice(idx + 1).toLowerCase() : '';
  };

  const openUploadPanel = () => {
    dispatch(resetUploadStatus());
    setUploadName('');
    setUploadDescription('');
    setUploadFile(null);
    setUploadFileError(null);
    setDatasetTab('upload');
  };

  const handleUploadFileChange = (file: File | null) => {
    setUploadFile(file);
    if (!file) {
      setUploadFileError(null);
      return;
    }
    const ext = getFileExtension(file.name);
    if (!SUPPORTED_UPLOAD_EXTENSIONS.includes(ext)) {
      setUploadFileError('Unsupported file type. Please choose a .json, .jsonl, .arrow, or .parquet file.');
    } else {
      setUploadFileError(null);
    }
  };

  const canUpload =
    Boolean(uploadName.trim()) && Boolean(uploadFile) && !uploadFileError && Boolean(draft.type) && !datasetUploading;

  const submitUpload = async () => {
    if (!uploadFile || !canUpload) return;

    // Hermes uploads (Agent type, agentFramework selected) go through
    // /upload-jsonl (for .jsonl files, which also needs category: 'Agents')
    // or /upload (any other supported extension) — both variants expect
    // eval_type: 'agent' rather than the wizard's internal 'agent_custom'
    // dataset-type discriminator. Every other case (Model, RAG, Agent
    // benchmark with no framework) keeps sending datasetType unchanged.
    const isHermesUpload = isAgentWithFramework;
    const uploadFileExt = getFileExtension(uploadFile.name);

    const result = await dispatch(
      uploadDataset({
        file: uploadFile,
        name: uploadName.trim(),
        description: uploadDescription.trim(),
        evalType: isHermesUpload ? 'agent' : datasetType,
        ...(isHermesUpload && uploadFileExt === 'jsonl' ? { category: 'Agents' } : {}),
      })
    );
    if (uploadDataset.fulfilled.match(result)) {
      dispatch(setDraft({ dataset: result.payload.id }));
      setDatasetTab('browse');
    }
  };

  // ---- (5) Model type + non-custom dataset ⇒ hide metrics & judge ---------
  const isCustomDataset = (suite as any)?.dataset_type === 'custom';
  const modelHidesMetrics = draft.type === 'model' && Boolean(suite) && !isCustomDataset;

  // Agent type with no framework selected (draft.agentFramework null) maps
  // to POST /agent-benchmark/run, which only ever accepts dataset_id,
  // model_ids, evaluation_name, run_samples — there's no metrics or judge
  // concept for this path at all, so the Metrics step (and the Test
  // Suite step's Upload tab, further below) simplify down the same way
  // the Model + standard-dataset case does.
  // Harness mode is its own path (POST /agent-harness/run) with full
  // metrics + judge, so it's excluded here.
  const isAgentBenchmarkNoFramework = draft.type === 'agent' && !draft.agentFramework && !draft.agentHarness;
  const hideMetricsStep = modelHidesMetrics || isAgentBenchmarkNoFramework;

  // RAG datasets are always pre-built retrieval corpora — there's no
  // "upload your own" flow for them (only Top K applies, further below).
  // Same for the harness: its suites are curated (the in-house set is
  // maintained by the team), so no upload there either.
  const isRag = draft.type === 'rag';
  const hideUploadTab = isAgentBenchmarkNoFramework || isRag || isHarness;

  // The Upload tab isn't offered for Agent benchmarks with no framework
  // selected, or for RAG (see Test Suite step below) — if the user had it
  // open and then goes back and changes type/framework into one of those
  // states, snap back to Browse so there's no dangling reference to a
  // hidden tab.
  useEffect(() => {
    if (hideUploadTab && datasetTab === 'upload') {
      setDatasetTab('browse');
    }
  }, [hideUploadTab, datasetTab]);

  // Clear any selected metrics the moment this simplified mode kicks in, so
  // neither the manifest nor the launch payload carries stale selections.
  useEffect(() => {
    if (hideMetricsStep && draft.metrics.length > 0) {
      dispatch(setDraft({ metrics: [] }));
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [hideMetricsStep]);

  const selectedModels = draft.models.map((id) => models.find((m) => m?.id === id)).filter(Boolean) as typeof models;
  const judgeModel = draft.judgeModelId ? models.find((m) => m?.id === draft.judgeModelId) : null;

  // Agent type WITH a framework selected (draft.agentFramework truthy, e.g.
  // Hermes) maps to POST /agent-benchmark/run-multi — used below to pick
  // the right upload payload shape, not for judge-related logic anymore
  // (see the always-on judge model requirement, further below).
  const isAgentWithFramework = draft.type === 'agent' && Boolean(draft.agentFramework) && !draft.agentHarness;

  const isModelSelectable = (modelId: string) => healthById?.[modelId] === 'success';

  // Selected models (that are still on screen in Step 3) whose health isn't
  // confirmed 'success' right now — e.g. a selection carried over from an
  // earlier visit to this step whose check is re-running, never ran, or
  // failed. Step 3 can't be continued past until this is empty, so a
  // stale selection doesn't let someone proceed with an unverified model.
  const selectedModelsNotReady = draft.models.filter(
    (id) =>
      availableModels.some((m) => m.id === id) && !(healthById?.[id] === 'success' && verifiedModelIds[id])
  );
  const selectedModelsChecking = selectedModelsNotReady.some((id) => healthById?.[id] !== 'failed');

  // Judge model options — every model that's actually usable (health check
  // passed) AND belongs to a currently-selected provider, regardless of
  // whether the model itself was picked in Step 3 for evaluation (judge
  // selection is independent of that). Scoped to availableModels
  // (draft.providers-filtered) rather than the full model list — a model
  // whose provider gets unselected back in Step 2 needs to disappear from
  // here too, not linger as a stale option.
  //
  // Only LLM-category models can act as a judge — Embedding and Reranker
  // models (which RAG's Models step lists alongside LLMs) can't grade
  // anything, so they're excluded here for every eval type.
  const judgeCandidateModels = useMemo(
    () =>
      availableModels.filter(
        (m) => (m?.category ?? '').toLowerCase() === 'llm' && isModelSelectable(m?.id)
      ),
    [availableModels, healthById]
  );

  const filteredJudgeModels = useMemo(
    () => judgeCandidateModels.filter((m) => (m?.name ?? '').toLowerCase().includes(judgeSearch.trim().toLowerCase())),
    [judgeCandidateModels, judgeSearch]
  );

  // Auto-scrolls down to the launch error message on the Review & Launch
  // step whenever a launch attempt fails — the error renders at the very
  // bottom of that step's content, below the whole review summary, so
  // without this the person has to notice it needs scrolling themselves.
  const launchErrorRef = useRef<HTMLParagraphElement | null>(null);
  useEffect(() => {
    if (launchError && step === totalSteps - 1) {
      launchErrorRef.current?.scrollIntoView({ behavior: 'smooth', block: 'end' });
    }
  }, [launchError, step, totalSteps]);

  useEffect(() => {
    if (draft.judgeModelId && !judgeCandidateModels.some((m) => m.id === draft.judgeModelId)) {
      dispatch(setDraft({ judgeModelId: null }));
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [judgeCandidateModels, draft.judgeModelId]);

  const toggleModel = (modelId: string) => {
    const alreadySelected = draft.models.includes(modelId);
    if (!alreadySelected && !isModelSelectable(modelId)) return;
    dispatch(setDraft({ models: toggle(draft.models, modelId) }));
    if (alreadySelected) {
      // Deselecting — drop its retry config so draft.modelRetryConfig
      // always mirrors draft.models exactly.
      const next = { ...draft.modelRetryConfig };
      delete next[modelId];
      dispatch(setDraft({ modelRetryConfig: next }));
    } else if (!draft.modelRetryConfig[modelId]) {
      // Selecting — seed defaults so the payload always has an entry for
      // every selected model, even if the user never touches the fields.
      dispatch(
        setDraft({
          modelRetryConfig: { ...draft.modelRetryConfig, [modelId]: { ...DEFAULT_MODEL_RETRY_CONFIG } },
        })
      );
    }
  };

  // RAG-only variant of toggleModel: models are grouped by category into
  // one column each (see modelsByCategory below), and exactly one model
  // per category is required — so selecting a model here also deselects
  // any other model already chosen from the SAME category (categoryIds),
  // while selections in every other category are left untouched. Clicking
  // the currently-selected model in a category deselects it, same as
  // toggleModel.
  const toggleModelForCategory = (modelId: string, categoryIds: string[]) => {
    const alreadySelected = draft.models.includes(modelId);
    if (!alreadySelected && !isModelSelectable(modelId)) return;

    const categoryIdSet = new Set(categoryIds);
    const nextModels = alreadySelected
      ? draft.models.filter((id) => id !== modelId)
      : [...draft.models.filter((id) => !categoryIdSet.has(id)), modelId];

    // Keep modelRetryConfig in sync with the new selection: drop entries
    // for anything this action removed from the category, seed a default
    // for the newly-picked model (if it doesn't already have one).
    const nextRetryConfig = { ...draft.modelRetryConfig };
    if (alreadySelected) {
      delete nextRetryConfig[modelId];
    } else {
      draft.models.filter((id) => categoryIdSet.has(id) && id !== modelId).forEach((id) => delete nextRetryConfig[id]);
      if (!nextRetryConfig[modelId]) nextRetryConfig[modelId] = { ...DEFAULT_MODEL_RETRY_CONFIG };
    }

    dispatch(setDraft({ models: nextModels, modelRetryConfig: nextRetryConfig }));
  };

  // Shared model-card renderer — used by both the normal grid (Model/Agent,
  // and RAG's own toggle behavior isn't baked in here) and the RAG
  // per-category columns below. `onToggle` is whichever selection rule
  // applies (toggleModel vs. toggleModelForCategory); everything else
  // about the card (health check, stats, retry/timeout panel) is
  // identical either way.
  // inlineRetry: show the per-model retry/timeout fields inside the card
  // (RAG columns). The Model/Agent view passes false — those fields live in
  // the "In this run" side panel instead.
  const renderModelCard = (m: Model, onToggle: () => void, inlineRetry = true) => {
    const on = draft.models.includes(m.id);
    const health: HealthStatus = healthById?.[m.id] ?? 'idle';
    const selectable = health === 'success';
    const caps = (m as any).capabilities as string[] | undefined;
    const inputPrice = (m as any).input_price as number | null | undefined;
    const outputPrice = (m as any).output_price as number | null | undefined;
    const accuracy = (m as any).accuracy_score as number | null | undefined;
    const providerName = providers.find((p) => p?.id === m.provider_id)?.name ?? m.provider_id;
    // Why this card can't be selected yet. Shown as a line inside the card in
    // the RAG columns; in the Model/Agent grid it's the card's tooltip
    // instead, so every card keeps the same content and the same size.
    const lockedHint =
      !selectable && !on
        ? health === 'loading'
          ? 'Waiting for health check to complete…'
          : health === 'failed'
            ? 'This model failed its health check and can\u2019t be selected.'
            : 'Run a health check to enable selection.'
        : undefined;

    // Health badge / action — shared by both card layouts below.
    const healthBadge = (
      <>
        {health === 'success' && (
          <span className={`${styles['ev__health-badge']} ${styles['ev__health-badge--success']}`}>
            <ShieldCheck size={12} /> Available
          </span>
        )}
        {health === 'failed' && (
          <button
            type="button"
            className={`${styles['ev__health-badge']} ${styles['ev__health-badge--failed']}`}
            onClick={(e) => {
              e.stopPropagation();
              runHealthCheck(m.id);
            }}
            title="Retry health check"
          >
            <ShieldAlert size={12} /> Unavailable
          </button>
        )}
        {health === 'loading' && (
          <span className={`${styles['ev__health-badge']} ${styles['ev__health-badge--loading']}`}>
            <Loader2 size={12} className={styles.ev__spin} /> Checking…
          </span>
        )}
        {health === 'idle' && (
          <button
            type="button"
            className={styles['ev__health-check-btn']}
            onClick={(e) => {
              e.stopPropagation();
              runHealthCheck(m.id);
            }}
          >
            <HeartPulse size={12} /> Check health
          </button>
        )}
      </>
    );

    // Model / Agent grid: same card design as the Test Suite step — header
    // (selection control, icon tile, name), tags, a labelled chip section,
    // and a footer of figures. Retries and timeout live in the side panel.
    if (!inlineRetry) {
      return (
        <div
          key={m.id}
          role="button"
          tabIndex={0}
          className={`${styles.ev__srow} ${styles['ev__srow--model']} ${on ? styles['ev__srow--on'] : ''} ${
            !selectable && !on ? styles['ev__srow--locked'] : ''
          } ${health === 'loading' ? styles['ev__mcard--checking'] : ''}`}
          onClick={onToggle}
          onKeyDown={(e) => {
            if (e.key === 'Enter' || e.key === ' ') {
              e.preventDefault();
              onToggle();
            }
          }}
          aria-pressed={on}
          aria-disabled={!selectable && !on}
          title={lockedHint}
        >
          <div className={styles['ev__srow-head']}>
            <span className={`${styles.ev__check} ${on ? styles['ev__check--on'] : ''}`}>
              {on && <Check size={11} strokeWidth={3} />}
            </span>
            <span className={styles['ev__srow-icon']}>
              <Cpu size={15} />
            </span>
            <span className={styles['ev__srow-name']} title={m.name || 'Unnamed model'}>
              {m.name || 'Unnamed model'}
            </span>
          </div>

          <div className={styles['ev__srow-tags']}>
            <span className={`${styles.ev__pilltag} ${styles['ev__pilltag--muted']}`}>{providerName}</span>
            {m.category && <span className={`${styles.ev__pilltag} ${styles['ev__pilltag--muted']}`}>{m.category}</span>}
            <span className={styles['ev__srow-health']}>{healthBadge}</span>
          </div>

          <div className={styles['ev__srow-subs']}>
            <span className={styles['ev__srow-k']}>Capabilities</span>
            <div className={styles['ev__srow-chips']}>
              {caps && caps.length > 0 ? (
                <>
                  {caps.slice(0, 4).map((c) => (
                    <span key={c} className={styles.ev__cap}>
                      {c}
                    </span>
                  ))}
                  {caps.length > 4 && (
                    <span className={styles.ev__cap} title={caps.slice(4).join(', ')}>
                      +{caps.length - 4} more
                    </span>
                  )}
                </>
              ) : (
                <span className={styles['ev__srow-none']}>Not listed</span>
              )}
            </div>
          </div>

          <div className={styles['ev__srow-side']}>
            <div className={styles.ev__stat}>
              <span className={styles['ev__stat-k']}>Context</span>
              <span className={styles['ev__stat-v']}>{formatContextWindow(m.context_window)}</span>
            </div>
            <div className={styles.ev__stat}>
              <span className={styles['ev__stat-k']}>Input /1M</span>
              <span className={styles['ev__stat-v']}>{formatPrice(inputPrice)}</span>
            </div>
            <div className={styles.ev__stat}>
              <span className={styles['ev__stat-k']}>Output /1M</span>
              <span className={styles['ev__stat-v']}>{formatPrice(outputPrice)}</span>
            </div>
            {typeof accuracy === 'number' && Number.isFinite(accuracy) && (
              <div className={styles.ev__stat}>
                <span className={styles['ev__stat-k']}>Accuracy</span>
                <span className={styles['ev__stat-v']}>{accuracy.toFixed(1)}%</span>
              </div>
            )}
          </div>
        </div>
      );
    }

    return (
      // Not a <button> — it contains a nested "Check health" control, so
      // it's a clickable div with keyboard support instead (nested
      // interactive elements aren't valid HTML).
      <div
        key={m.id}
        role="button"
        tabIndex={0}
        className={`${styles.ev__mcard} ${on ? styles['ev__mcard--on'] : ''} ${
          !selectable && !on ? styles['ev__mcard--locked'] : ''
        } ${health === 'loading' ? styles['ev__mcard--checking'] : ''}`}
        onClick={onToggle}
        onKeyDown={(e) => {
          if (e.key === 'Enter' || e.key === ' ') {
            e.preventDefault();
            onToggle();
          }
        }}
        aria-pressed={on}
        aria-disabled={!selectable && !on}
      >
        <div className={styles['ev__mcard-top']}>
          <div className={styles['ev__mcard-name']}>{m.name || 'Unnamed model'}</div>
          {on && (
            <span className={styles['ev__mcard-mark']}>
              <Check size={12} strokeWidth={3} />
            </span>
          )}
        </div>

        {/* Provider name + health badge, same line, badge on the right */}
        <div className={styles['ev__mcard-provider-row']}>
          <span className={styles['ev__mcard-provider']}>{providerName}</span>
          {healthBadge}
        </div>

        {caps && caps.length > 0 && (
          <div className={styles.ev__caps}>
            {caps.slice(0, 3).map((c) => (
              <span key={c} className={styles.ev__cap}>
                {c}
              </span>
            ))}
          </div>
        )}
        <div className={styles['ev__mcard-stats']}>
          <span className={styles.ev__stat}>
            <span className={styles['ev__stat-k']}>Context</span>
            <span className={styles['ev__stat-v']}>{formatContextWindow(m.context_window)}</span>
          </span>
          {(inputPrice !== undefined || outputPrice !== undefined) && (
            <span className={styles.ev__stat}>
              <span className={styles['ev__stat-k']}>Price /1M</span>
              <span className={styles['ev__stat-v']}>
                {formatPrice(inputPrice)}/{formatPrice(outputPrice)}
              </span>
            </span>
          )}
          {typeof accuracy === 'number' && Number.isFinite(accuracy) && (
            <span className={styles.ev__stat}>
              <span className={styles['ev__stat-k']}>Accuracy</span>
              <span className={styles['ev__stat-v']}>{accuracy.toFixed(1)}%</span>
            </span>
          )}
        </div>

        {inlineRetry && lockedHint && <p className={styles['ev__mcard-hint']}>{lockedHint}</p>}

        {/* Per-model retry/timeout — only rendered once the card is
            selected AND the user has chosen "Individually" mode (in
            "Apply to all" mode, the shared config above the grid covers
            every selected model, so no per-card panel is needed at all).
            Unselected cards stay compact since this only exists in the
            DOM for selected ones — &__grid's align-items: start keeps the
            taller selected card from stretching its unselected neighbors
            in the same row. stopPropagation on the wrapper (and the
            inputs themselves) keeps clicks here from also toggling the
            card's selection via the outer onClick. */}
        {inlineRetry && on && draft.retryConfigMode === 'individual' && (
          <div
            className={styles['ev__mcard-retry']}
            onClick={(e) => e.stopPropagation()}
            onKeyDown={(e) => e.stopPropagation()}
          >
            <div className={styles['ev__mcard-retry-fields']}>
              <div className={styles['ev__mcard-retry-field']}>
                <label>Max retries</label>
                <input
                  type="number"
                  min={MIN_MAX_RETRIES}
                  max={MAX_MAX_RETRIES}
                  value={displayNumInput(draft.modelRetryConfig[m.id]?.max_retries ?? DEFAULT_MAX_RETRIES)}
                  onChange={(e) => updateModelRetryConfig(m.id, 'max_retries', parseNumInput(e.target.value))}
                  onBlur={() => clampModelRetryConfig(m.id, 'max_retries')}
                />
              </div>
              <div className={styles['ev__mcard-retry-field']}>
                <label>Timeout (s)</label>
                <input
                  type="number"
                  min={MIN_TIMEOUT}
                  max={MAX_TIMEOUT}
                  value={displayNumInput(draft.modelRetryConfig[m.id]?.timeout ?? DEFAULT_TIMEOUT)}
                  onChange={(e) => updateModelRetryConfig(m.id, 'timeout', parseNumInput(e.target.value))}
                  onBlur={() => clampModelRetryConfig(m.id, 'timeout')}
                />
              </div>
            </div>
            <p className={styles['ev__mcard-retry-hint']}>
              <Info size={11} /> Retries {MIN_MAX_RETRIES}–{MAX_MAX_RETRIES} · Timeout {MIN_TIMEOUT}
              –{MAX_TIMEOUT}s
            </p>
          </div>
        )}
      </div>
    );
  };

  // Update a selected model's retry config field while the user is typing —
  // stores exactly what's on screen (no clamping) so intermediate/out-of-
  // range values while typing (e.g. clearing the field, or typing "6" on
  // the way to "65") aren't immediately overwritten and don't fight the
  // user's keystrokes. Range is enforced on blur (see below). Clicks/
  // keystrokes inside the config panel call e.stopPropagation() so they
  // don't also toggle the card's selection (see the card's onClick).
  const updateModelRetryConfig = (modelId: string, field: keyof ModelRetryConfig, raw: number) => {
    const current = draft.modelRetryConfig[modelId] || { ...DEFAULT_MODEL_RETRY_CONFIG };
    const safe = raw;
    dispatch(
      setDraft({
        modelRetryConfig: { ...draft.modelRetryConfig, [modelId]: { ...current, [field]: safe } },
      })
    );
  };

  // Clamps a selected model's retry config field into its valid range —
  // called on blur, once the user has finished typing.
  const clampModelRetryConfig = (modelId: string, field: keyof ModelRetryConfig) => {
    const current = draft.modelRetryConfig[modelId] || { ...DEFAULT_MODEL_RETRY_CONFIG };
    const clamped = clampRetryField(field, current[field]);
    if (clamped === current[field]) return;
    dispatch(
      setDraft({
        modelRetryConfig: { ...draft.modelRetryConfig, [modelId]: { ...current, [field]: clamped } },
      })
    );
  };

  // Same pattern as above, but for the single shared "apply to all" config
  // used when retryConfigMode === 'all'.
  const updateRetryConfigAll = (field: keyof ModelRetryConfig, raw: number) => {
    const safe = raw;
    dispatch(setDraft({ retryConfigAll: { ...draft.retryConfigAll, [field]: safe } }));
  };

  const clampRetryConfigAll = (field: keyof ModelRetryConfig) => {
    const clamped = clampRetryField(field, draft.retryConfigAll[field]);
    if (clamped === draft.retryConfigAll[field]) return;
    dispatch(setDraft({ retryConfigAll: { ...draft.retryConfigAll, [field]: clamped } }));
  };

  // Why the current step can't be continued yet — null means good to go.
  // Same rules as the old canGo(), but each one carries its reason, shown
  // next to the (disabled) Next button.
  const stepBlocker = (): string | null => {
    if (step === 0) return draft.name.trim() ? null : 'Give the run a name.';
    if (step === 1) {
      if (!draft.type) return 'Choose a system under test.';
      if (isHarness && draft.harnessAgents.length === 0) return 'Select at least one agent.';
      return null;
    }
    if (step === 2) return draft.providers.length > 0 ? null : 'Select at least one provider.';
    if (step === 3) {
      if (modelsLoading || !modelsFetchedThisVisit) return 'Loading models…';
      if (draft.type === 'rag') {
        const countIn = (cat: string) =>
          (modelsByCategory.get(cat) ?? []).filter((m) => draft.models.includes(m.id)).length;
        if (countIn('Embedding') !== 1 || countIn('Reranker') !== 1 || countIn('LLM') === 0) {
          return 'Select one Embedding, one Reranker and at least one LLM.';
        }
      } else if (draft.models.length === 0) {
        return 'Tick at least one model.';
      }
      if (selectedModelsNotReady.length > 0) {
        const id = selectedModelsNotReady[0];
        const name = models.find((m) => m?.id === id)?.name || id;
        return healthById?.[id] === 'failed'
          ? `${name} failed its health check. Retry it or untick it.`
          : `Waiting for the health check on ${name}.`;
      }
      return null;
    }
    if (step === 4) {
      if (isHarness) {
        if (selectedHarnessSuites.length === 0) return 'Select at least one suite.';
        if (!selectedHarnessSuites.every((d) => (d.question_count ?? 0) > 0)) {
          return 'Untick the suites that have no questions.';
        }
        return null;
      }
      if (!draft.dataset) return 'Select a test suite.';
      if ((suite?.question_count ?? 0) <= 0) return 'This suite has no questions. Pick another.';
      return null;
    }
    if (step === 5) {
      const metricsAvailable = !hideMetricsStep && (metricsCatalog.length > 0 || customMetricsForType.length > 0);
      if (metricsAvailable && draft.metrics.length === 0) {
        return isHarness ? 'Select at least one scorer.' : 'Select at least one metric.';
      }
      if (!draft.judgeModelId) return 'Select a judge model.';
      const retestApplies =
        draft.type === 'model' || draft.type === 'rag' || (draft.type === 'agent' && Boolean(draft.agentFramework));
      if (retestApplies && draft.retestOnWrong && !draft.retestVerifyMetric) {
        return 'Pick a verify metric for Retest on wrong.';
      }
      if (isHarness && (Number.isNaN(draft.harnessSettings.max_steps) || Number.isNaN(draft.harnessSettings.task_timeout))) {
        return 'Fill in the limits per task.';
      }
      return null;
    }
    return null;
  };
  const blocker = stepBlocker();
  const canGo = () => blocker === null;

  const goNext = () => {
    if (!canGo()) return;
    setStep((s) => Math.min(totalSteps - 1, s + 1));
  };
  // Navigating backward away from the Metrics step clears any instruction
  // that was there — whether typed or generated — so coming back to it
  // later (after e.g. changing the dataset or models on an earlier step)
  // never shows a stale instruction that no longer matches the current
  // selections. Forward navigation (goNext, and reaching Review) leaves it
  // untouched, since that's the normal "continue with what I have" path.
  const goBack = () => {
    if (step === 5) dispatch(setDraft({ instruction: '' }));
    setStep((s) => Math.max(0, s - 1));
  };
  const goToStep = (target: number) => {
    if (target < step) {
      if (step === 5) dispatch(setDraft({ instruction: '' }));
      setStep(target);
    }
  };

  // ---- (3) & (4) launch: three different endpoints depending on type ------
  const launch = async () => {
    const dataset = datasets.find((d) => d?.id === draft.dataset);
    const judgeModelObj = draft.judgeModelId ? models.find((m) => m?.id === draft.judgeModelId) : undefined;

    // Multimodal pre-flight — only when the selected dataset carries
    // images (has_images, from GET /datasets?eval_type={type}). Checks
    // every selected model PLUS the judge model against POST /models/
    // check-multimodal before creating the evaluation at all: a non-2xx
    // response means at least one of them can't handle image input, so
    // the create/run call below is skipped entirely and the error is
    // surfaced the same way a launch failure normally is. Applies equally
    // to Model, Agent, and RAG — done once here, before branching into
    // whichever of the three endpoints this draft.type ends up calling.
    // Harness: any of the picked suites carrying images triggers the check.
    const selectedHasImages = isHarness
      ? selectedHarnessSuites.some((s) => s.has_images)
      : Boolean((dataset as any)?.has_images);
    if (selectedHasImages) {
      const multimodalModelIds = Array.from(
        new Set([...draft.models, ...(draft.judgeModelId ? [draft.judgeModelId] : [])])
      );
      dispatch(setLaunchError(null));
      dispatch(setLaunching(true));
      try {
        await evaluationsApi.checkMultimodal(multimodalModelIds);
      } catch (err) {
        const detail =
          (err as { response?: { data?: { detail?: string } } })?.response?.data?.detail ||
          (err as Error)?.message ||
          'One or more selected models cannot handle this dataset\u2019s images.';
        dispatch(setLaunchError(detail));
        dispatch(setLaunching(false));
        return;
      }
    }

    // Change: "Full" mode means "use the whole dataset" — for the agent
    // benchmark endpoints the backend contract for that is run_samples: 0.
    const effectiveRunSamples =
      draft.runSamplesMode === 'full' || Number.isNaN(draft.runSamples) ? 0 : draft.runSamples;
    // For POST /evaluations (Model/RAG) specifically, "Full" sends
    // run_samples as null — the backend's contract for "use the whole
    // dataset" on this endpoint, distinct from the agent-benchmark
    // endpoints above which use 0 for the same concept.
    const createEvalRunSamples: number | null =
      draft.runSamplesMode === 'full' ? null : Number.isNaN(draft.runSamples) ? 0 : draft.runSamples;

    let result: any;

    // Custom metrics (their `id`s) are a subset of whichever metrics list
    // applies — draft.metrics mixes built-in names and custom ids together
    // (see customMetricIdSet above), so this pulls out just the custom
    // ones to send as the additive selected_metric_ids field. Only added to
    // the payload when non-empty.
    const selectedMetricIdsFor = (effectiveMetrics: string[]): Pick<CreateEvaluationRequest, 'selected_metric_ids'> => {
      const ids = effectiveMetrics.filter((m) => customMetricIdSet.has(m));
      return ids.length > 0 ? { selected_metric_ids: ids } : {};
    };

    // Same "Apply to all" / "Individually" shape used for the Model/RAG
    // create-evaluation payload below — reused here for the two
    // agent-benchmark endpoints so retry/timeout behaves identically
    // regardless of which API call the Agent step ends up making.
    // Clamped defensively in case a field is still mid-edit when Launch
    // is clicked.
    const retryConfigPayload: Pick<CreateEvaluationRequest, 'max_retries' | 'timeout' | 'model_retry_config'> =
      draft.retryConfigMode === 'all'
        ? clampModelRetryConfigValue(draft.retryConfigAll)
        : {
            model_retry_config: draft.models.reduce<Record<string, ModelRetryConfig>>((acc, id) => {
              acc[id] = clampModelRetryConfigValue(draft.modelRetryConfig[id] || DEFAULT_MODEL_RETRY_CONFIG);
              return acc;
            }, {}),
          };

    // Retest on Wrong — Model and RAG always; Agent only when a framework
    // is selected (POST /agent-benchmark/run has no metrics-related
    // payload at all, so this stays {} there). retest_on_wrong is always
    // sent once applicable; retest_max_rounds/retest_verify_metric only
    // when it's true. Computed once here and reused for both the
    // Model/RAG create-eval payload and the run-multi payload below, so
    // the two stay in sync.
    const retestApplies =
      draft.type === 'model' || draft.type === 'rag' || (draft.type === 'agent' && Boolean(draft.agentFramework));
    const retestPayload: Pick<CreateEvaluationRequest, 'retest_on_wrong' | 'retest_max_rounds' | 'retest_verify_metric'> =
      retestApplies
        ? {
            retest_on_wrong: draft.retestOnWrong,
            ...(draft.retestOnWrong
              ? {
                  retest_max_rounds: clampRetestRounds(draft.retestMaxRounds),
                  ...(draft.retestVerifyMetric ? { retest_verify_metric: draft.retestVerifyMetric } : {}),
                }
              : {}),
          }
        : {};

    if (isHarness) {
      // POST /agent-harness/run — Document-agent harness. Same retry shape
      // and judge config as the Model/RAG create-evaluation path; suites
      // each carry their own run_samples (same "0 = full" convention as the
      // agent-benchmark endpoints).
      const harnessJudgeRetry: ModelRetryConfig = clampModelRetryConfigValue(
        draft.retryConfigMode === 'all'
          ? draft.retryConfigAll
          : (draft.judgeModelId && draft.modelRetryConfig[draft.judgeModelId]) || DEFAULT_MODEL_RETRY_CONFIG
      );
      result = await dispatch(
        runAgentHarness({
          evaluation_name: draft.name,
          agents: draft.harnessAgents,
          model_ids: draft.models,
          suites: draft.harnessSuiteIds.map((id) => ({ dataset_id: id, run_samples: effectiveRunSamples })),
          selected_metrics: draft.metrics,
          ...selectedMetricIdsFor(draft.metrics),
          judge_config: {
            model_id: draft.judgeModelId || '',
            base_url: judgeModelObj?.base_url || '',
            api_key: draft.judgeModelId || '',
            max_retries: harnessJudgeRetry.max_retries,
            timeout: harnessJudgeRetry.timeout,
          },
          harness: {
            max_steps: clampHarnessSteps(draft.harnessSettings.max_steps),
            task_timeout: clampRetryField('timeout', draft.harnessSettings.task_timeout),
            save_traces: draft.harnessSettings.save_traces,
          },
          ...retryConfigPayload,
        })
      );
    } else if (draft.type === 'agent' && !draft.agentFramework) {
      // POST /agent-benchmark/run
      result = await dispatch(
        runAgentBenchmark({
          dataset_id: dataset?.id || '',
          model_ids: draft.models,
          evaluation_name: draft.name,
          run_samples: effectiveRunSamples,
          ...retryConfigPayload,
        })
      );
    } else if (draft.type === 'agent' && draft.agentFramework) {
      // POST /agent-benchmark/run-multi
      result = await dispatch(
        runAgentBenchmarkMulti({
          dataset_id: dataset?.id || '',
          model_ids: draft.models,
          evaluation_name: draft.name,
          selected_metrics: draft.metrics,
          ...selectedMetricIdsFor(draft.metrics),
          selected_categories: draft.subgroup,
          run_samples: effectiveRunSamples,
          ...retryConfigPayload,
          ...retestPayload,
        })
      );
    } else {
      // POST /evaluations then /evaluations/{id}/start — Model or RAG

      // The Models step's "Apply to all" / "Individually" toggle decides
      // whether retry/timeout for the judge model comes from the shared
      // config or from that specific model's own entry (falling back to
      // defaults if the judge model was never one of the selected models,
      // so it never got its own individual entry). Clamped defensively in
      // case a field is still mid-edit (not yet blurred) when Launch is
      // clicked.
      const judgeRetryConfig: ModelRetryConfig = clampModelRetryConfigValue(
        draft.retryConfigMode === 'all'
          ? draft.retryConfigAll
          : (draft.judgeModelId && draft.modelRetryConfig[draft.judgeModelId]) || DEFAULT_MODEL_RETRY_CONFIG
      );

      // RAG-only — split draft.models (which holds the Embedding, Reranker,
      // AND LLM selections all together, same as any other type) back out
      // by category for the payload: model_ids carries only the LLM
      // selection(s), Embedding/Reranker each go in their own dedicated
      // field instead. canGo's step-3 check already guarantees exactly one
      // Embedding and one Reranker are selected by the time launch() runs.
      const ragEmbeddingId = isRag
        ? draft.models.find((id) => (modelsByCategory.get('Embedding') ?? []).some((m) => m.id === id))
        : undefined;
      const ragRerankerId = isRag
        ? draft.models.find((id) => (modelsByCategory.get('Reranker') ?? []).some((m) => m.id === id))
        : undefined;
      const ragLlmIds = isRag
        ? draft.models.filter((id) => (modelsByCategory.get('LLM') ?? []).some((m) => m.id === id))
        : draft.models;

      const payload: CreateEvaluationRequest = {
        name: draft.name,
        eval_type: draft.type || '',
        dataset_id: dataset?.id || '',
        // RAG doesn't use `benchmark` — omitted entirely for that type
        // rather than sent as the dataset name (or null), matching how
        // top_k is likewise only ever included for RAG below.
        ...(isRag ? {} : { benchmark: dataset?.name || undefined }),
        model_ids: ragLlmIds,
        ...(isRag ? { embedding_model_id: ragEmbeddingId, reranker_model_id: ragRerankerId } : {}),
        // Exactly one of these two shapes, matching retryConfigMode:
        //   'all'        -> shared top-level max_retries/timeout
        //   'individual' -> model_retry_config, one entry per model, seeded
        //                   with defaults as soon as a model is selected
        //                   (see toggleModel) so it's always fully populated.
        // Same retryConfigPayload used for the agent-benchmark endpoints
        // above, so retry/timeout behaves identically across all launch
        // paths (Model/RAG create-eval, and both agent-benchmark calls).
        ...retryConfigPayload,
        selected_metrics: hideMetricsStep ? [] : draft.metrics,
        ...selectedMetricIdsFor(hideMetricsStep ? [] : draft.metrics),
        run_samples: createEvalRunSamples,
        selected_category: draft.subgroup.length > 0 ? draft.subgroup : dataset ? [dataset.category] : undefined,
        // Judge model is mandatory now (see canGo's step-5 check), so
        // draft.judgeModelId is always set by the time launch() runs — the
        // {} fallback is just defensive. max_retries/timeout mirror
        // whichever value applies per judgeRetryConfig above.
        judge_config: draft.judgeModelId
          ? {
              model_id: draft.judgeModelId,
              base_url: judgeModelObj?.base_url || '',
              api_key: draft.judgeModelId,
              max_retries: judgeRetryConfig.max_retries,
              timeout: judgeRetryConfig.timeout,
            }
          : {},
        // RAG-only — how many retrieved documents to consider per query.
        // Omitted entirely for Model/Agent rather than sent as undefined,
        // so it doesn't show up as a spurious key in the request body.
        ...(isRag ? { top_k: draft.topK } : {}),
        // Optional — omitted entirely (rather than sent as an empty
        // string) when the user hasn't typed or generated one.
        ...(draft.instruction.trim() ? { instruction: draft.instruction.trim() } : {}),
        // Model and RAG — Metrics step's "Retest on Wrong" control (see
        // retestApplies above). Same retestPayload used for the run-multi
        // call above, so Agent-with-framework gets it too.
        ...retestPayload,
      };
      result = await dispatch(launchEvaluation(payload));
    }

    const succeeded =
      launchEvaluation.fulfilled.match(result) ||
      runAgentBenchmark.fulfilled.match(result) ||
      runAgentBenchmarkMulti.fulfilled.match(result) ||
      runAgentHarness.fulfilled.match(result);

    if (succeeded) {
      setToast(true);
      setTimeout(() => {
        setToast(false);
        navigate('/app/history');
      }, 2000);
    }
  };


  // ---- live Run Manifest values (one per step) ----------------------------
  const providerNames = draft.providers.map((id) => providers.find((p) => p?.id === id)?.name || id);
  const mf = (value: string, filled: boolean) => ({ value: filled ? value : 'Not set', empty: !filled });
  const typeLabel = TYPE_OPTIONS.find((o) => o.value === draft.type)?.label ?? '';
  const frameworkTitle = draft.agentFramework ? AGENT_FRAMEWORKS.find((f) => f.id === draft.agentFramework)?.title : null;
  const harnessAgentTitles = draft.harnessAgents.map((id) => HARNESS_AGENTS.find((a) => a.id === id)?.title || id);
  // Run matrix (Review step + manifest): every agent runs every selected
  // model on every selected suite — that's the scorecard's grain.
  const harnessRunCount = draft.harnessAgents.length * draft.models.length * draft.harnessSuiteIds.length;
  const typeManifestValue = isHarness
    ? `${typeLabel} · Harness${harnessAgentTitles.length > 0 ? ` (${harnessAgentTitles.join(', ')})` : ''}`
    : frameworkTitle
      ? `${typeLabel} · ${frameworkTitle}`
      : typeLabel;
  const scoringManifestValue = [
    hideMetricsStep || draft.metrics.length === 0
      ? null
      : `${draft.metrics.length} ${isHarness ? 'scorer' : 'metric'}${draft.metrics.length === 1 ? '' : 's'}`,
    judgeModel ? `Judge: ${judgeModel.name || 'Unnamed model'}` : null,
  ]
    .filter(Boolean)
    .join(', ');
  const manifest = [
    mf(draft.name, Boolean(draft.name)),
    mf(typeManifestValue, Boolean(draft.type)),
    mf(providerNames.join(', '), draft.providers.length > 0),
    mf(`${draft.models.length} model${draft.models.length === 1 ? '' : 's'}`, draft.models.length > 0),
    isHarness
      ? mf(
          selectedHarnessSuites.length === 1 ? selectedHarnessSuites[0].name : `${selectedHarnessSuites.length} suites`,
          selectedHarnessSuites.length > 0
        )
      : mf(suite?.name || '', Boolean(suite)),
    mf(scoringManifestValue, scoringManifestValue !== ''),
    mf('Ready to launch', step === totalSteps - 1),
  ];

  const stageTitle = step === 4 && isHarness ? 'Pick test suites' : STAGE[step].title;
  const stageSub =
    step === 4 && isHarness
      ? 'Each suite covers one kind of work. Pick one per use case for a full scorecard.'
      : STAGE[step].sub;
  const scorerWord = isHarness ? 'scorers' : 'metrics';

  // ---- run size (Scoring step formula, Review banner + run matrix) ---------
  // sampleN null = every question. Per suite, a sample larger than the suite
  // simply runs the whole suite.
  const sampleN = draft.runSamplesMode === 'full' || Number.isNaN(draft.runSamples) ? null : draft.runSamples;
  const suiteQuestions = (count: number) => (sampleN === null ? count : Math.min(sampleN, count));
  const harnessQuestionsPerPass = selectedHarnessSuites.reduce((n, s) => n + suiteQuestions(s.question_count ?? 0), 0);
  const harnessTaskCount = harnessQuestionsPerPass * draft.models.length * draft.harnessAgents.length;
  const coveredUseCases = harnessUseCases.filter((u) => selectedHarnessSuites.some((s) => s.use_case === u));
  const uncoveredUseCases = harnessUseCases.filter((u) => !coveredUseCases.includes(u));

  // ---- Models step (Model / Agent): table + "In this run" panel ------------
  const llmAvailable = availableModels.filter((m) => (m?.category ?? '').toLowerCase() === 'llm');
  const tableModels = filteredModels.filter(
    (m) => modelProviderFilter === 'all' || m.provider_id === modelProviderFilter
  );
  // ---- Type step: "About" panel -------------------------------------------
  // What the selected type measures, what it's good for, and how a run of
  // that type works (copy in TYPE_ABOUT).
  const typeAboutPanel = (() => {
    const opt = TYPE_OPTIONS.find((o) => o.value === draft.type);
    const about = draft.type ? TYPE_ABOUT[draft.type] : undefined;
    if (!opt || !about) {
      return (
        <p className={styles['ev__about-empty']}>
          <Info size={14} /> Select a system under test to see what it measures and when to use it.
        </p>
      );
    }
    const Icon = opt.icon;
    return (
      <div className={`${styles.ev__about} ${styles.ev__anim}`} key={opt.label}>
        <div className={styles['ev__about-head']}>
          <span
            className={`${styles['ev__option-icon']} ${
              opt.variant ? styles[`ev__option-icon--${opt.variant}`] : ''
            }`}
          >
            <Icon size={18} />
          </span>
          <div>
            <h3>About {opt.label} evaluation</h3>
            <p>{about.tagline}</p>
          </div>
        </div>
        <div className={styles['ev__about-cols']}>
          <div className={styles['ev__about-col']}>
            <p className={styles['ev__about-k']}>What it measures</p>
            <ul className={styles['ev__about-list']}>
              {about.measures.map((t) => (
                <li key={t}>
                  <Check size={13} strokeWidth={3} /> {t}
                </li>
              ))}
            </ul>
          </div>
          <div className={styles['ev__about-col']}>
            <p className={styles['ev__about-k']}>Good for</p>
            <ul className={styles['ev__about-list']}>
              {about.goodFor.map((t) => (
                <li key={t}>
                  <ChevronRight size={13} strokeWidth={3} /> {t}
                </li>
              ))}
            </ul>
          </div>
          <div className={styles['ev__about-col']}>
            <p className={styles['ev__about-k']}>How a run works</p>
            <ol className={styles['ev__about-flow']}>
              {about.flow.map((t, n) => (
                <li key={t}>
                  <span>{n + 1}</span> {t}
                </li>
              ))}
            </ol>
          </div>
        </div>
      </div>
    );
  })();

  // ---- Name step: quick-add tokens ----------------------------------------
  // One-click suffixes for the run name: this quarter, a version, today's
  // date. Clicking a token that's already in the name takes it back out.
  const nameTokens = (() => {
    const now = new Date();
    return [
      `Q${Math.floor(now.getMonth() / 3) + 1} ${now.getFullYear()}`,
      'v1',
      now.toLocaleDateString(undefined, { day: 'numeric', month: 'short' }),
    ];
  })();
  const toggleNameToken = (token: string) => {
    const at = draft.name.toLowerCase().indexOf(token.toLowerCase());
    const next =
      at === -1
        ? `${draft.name.trim()} ${token}`.trim()
        : `${draft.name.slice(0, at)}${draft.name.slice(at + token.length)}`.replace(/\s{2,}/g, ' ').trim();
    dispatch(setDraft({ name: next }));
  };

  // Search field of whichever list header is on screen (one step at a time).
  // Pressing "/" anywhere outside a text field jumps to it.
  const listSearchRef = useRef<HTMLInputElement>(null);
  useEffect(() => {
    const onKey = (e: KeyboardEvent) => {
      if (e.key !== '/' || e.metaKey || e.ctrlKey || e.altKey) return;
      const t = e.target as HTMLElement | null;
      if (t && (t.tagName === 'INPUT' || t.tagName === 'TEXTAREA' || t.isContentEditable)) return;
      if (!listSearchRef.current) return;
      e.preventDefault();
      listSearchRef.current.focus();
    };
    window.addEventListener('keydown', onKey);
    return () => window.removeEventListener('keydown', onKey);
  }, []);

  // ---- list header ---------------------------------------------------------
  // Shared header for the Providers, Models and Test Suite lists: a title
  // with a live count on the left, search + refresh on the right, an optional
  // row of filter chips underneath, and a rule separating it from the list.
  const listHead = (o: {
    title: string;
    meta: ReactNode;
    // how many items the list is showing right now (drives the result chip)
    count: number;
    search: { placeholder: string; value: string; onChange: (v: string) => void };
    refresh: { onClick: () => void; busy: boolean; title: string };
    filters?: ReactNode;
    sticky?: boolean;
  }) => {
    const hasQuery = o.search.value.trim() !== '';
    return (
      <div className={`${styles.ev__lhead} ${o.sticky ? styles['ev__lhead--sticky'] : ''}`}>
        <div className={styles['ev__lhead-row']}>
        <div className={styles['ev__lhead-title']}>
          <h3>{o.title}</h3>
          <span>{o.meta}</span>
        </div>

        {/* Search bar: one compact control holding the icon, the field, a
            live result count, the "/" hint or clear button, and refresh. */}
        <div
          className={`${styles.ev__sbar} ${hasQuery ? styles['ev__sbar--active'] : ''}`}
          onMouseDown={(e) => {
            // clicking anywhere on the bar (not on its buttons) focuses the field
            if ((e.target as HTMLElement).closest('button, input')) return;
            e.preventDefault();
            listSearchRef.current?.focus();
          }}
        >
          <span className={styles['ev__sbar-icon']}>
            <Search size={13} strokeWidth={2.6} />
          </span>
          <input
            ref={listSearchRef}
            type="text"
            placeholder={o.search.placeholder}
            aria-label={o.search.placeholder}
            value={o.search.value}
            onChange={(e) => o.search.onChange(e.target.value)}
            onKeyDown={(e) => {
              // Esc clears first, then leaves the field
              if (e.key !== 'Escape') return;
              if (o.search.value) o.search.onChange('');
              else e.currentTarget.blur();
            }}
          />
          {hasQuery && (
            <span
              className={`${styles['ev__sbar-count']} ${o.count === 0 ? styles['ev__sbar-count--none'] : ''}`}
              aria-live="polite"
            >
              {o.count === 0 ? 'No matches' : `${o.count} result${o.count === 1 ? '' : 's'}`}
            </span>
          )}
          {o.search.value ? (
            <button
              type="button"
              className={styles['ev__sbar-clear']}
              onClick={() => {
                o.search.onChange('');
                listSearchRef.current?.focus();
              }}
              aria-label="Clear search"
            >
              <X size={12} strokeWidth={2.5} />
            </button>
          ) : (
            <kbd className={styles['ev__sbar-kbd']} title="Press / to search">
              /
            </kbd>
          )}
          <span className={styles['ev__sbar-sep']} />
          <button
            type="button"
            className={styles['ev__sbar-refresh']}
            onClick={o.refresh.onClick}
            disabled={o.refresh.busy}
            title={o.refresh.title}
            aria-label={o.refresh.title}
          >
            <RefreshCw size={14} className={o.refresh.busy ? styles.ev__spin : ''} />
          </button>
        </div>
        </div>

        {o.filters && <div className={styles['ev__lhead-filters']}>{o.filters}</div>}
      </div>
    );
  };

  // One − / value / + control. The buttons step by `step` and clamp to the
  // field's range straight away; typing still works as before (free typing,
  // clamped on blur).
  const retryStepper = (
    field: keyof ModelRetryConfig,
    label: string,
    icon: ReactNode,
    step: number,
    value: ModelRetryConfig | undefined,
    onChange: (field: keyof ModelRetryConfig, raw: number) => void,
    onBlur: (field: keyof ModelRetryConfig) => void
  ) => {
    const min = field === 'max_retries' ? MIN_MAX_RETRIES : MIN_TIMEOUT;
    const max = field === 'max_retries' ? MAX_MAX_RETRIES : MAX_TIMEOUT;
    const fallback = field === 'max_retries' ? DEFAULT_MAX_RETRIES : DEFAULT_TIMEOUT;
    const raw = value?.[field] ?? fallback;
    const current = Number.isNaN(raw) ? fallback : raw;
    const bump = (dir: 1 | -1) => onChange(field, clampRetryField(field, current + dir * step));
    return (
      <div className={styles.ev__stepfield}>
        <span className={styles['ev__stepfield-label']}>
          {icon} {label}
        </span>
        <div className={styles.ev__stepper}>
          <button type="button" onClick={() => bump(-1)} disabled={current <= min} aria-label={`Decrease ${label}`}>
            <Minus size={13} />
          </button>
          <input
            type="number"
            min={min}
            max={max}
            aria-label={label}
            value={displayNumInput(raw)}
            onChange={(e) => onChange(field, parseNumInput(e.target.value))}
            onBlur={() => onBlur(field)}
          />
          <button type="button" onClick={() => bump(1)} disabled={current >= max} aria-label={`Increase ${label}`}>
            <Plus size={13} />
          </button>
        </div>
      </div>
    );
  };

  const retryFields = (
    value: ModelRetryConfig | undefined,
    onChange: (field: keyof ModelRetryConfig, raw: number) => void,
    onBlur: (field: keyof ModelRetryConfig) => void
  ) => (
    <div className={styles.ev__stepfields}>
      {retryStepper('max_retries', 'Max retries', <RotateCw size={12} />, 1, value, onChange, onBlur)}
      {retryStepper('timeout', 'Timeout (s)', <Timer size={12} />, 30, value, onChange, onBlur)}
    </div>
  );

  const modelsTableView = (
    <div className={styles.ev__split}>
      <div className={styles['ev__split-main']}>
        {listHead({
          title: 'Available models',
          meta: (
            <>
              {tableModels.length} of {llmAvailable.length} shown · <b>{draft.models.length}</b> selected
            </>
          ),
          count: tableModels.length,
          search: { placeholder: 'Search models', value: modelSearch, onChange: setModelSearch },
          refresh: { onClick: refreshModels, busy: modelsLoading, title: 'Refresh models' },
          filters: (
            <>
            <button
              type="button"
              className={`${styles['ev__filter-chip']} ${modelProviderFilter === 'all' ? styles['ev__filter-chip--on'] : ''}`}
              onClick={() => setModelProviderFilter('all')}
            >
              All <span className={styles['ev__filter-chip-count']}>{llmAvailable.length}</span>
            </button>
            {draft.providers.map((id) => {
              const on = modelProviderFilter === id;
              return (
                <button
                  key={id}
                  type="button"
                  className={`${styles['ev__filter-chip']} ${on ? styles['ev__filter-chip--on'] : ''}`}
                  onClick={() => setModelProviderFilter(on ? 'all' : id)}
                >
                  {providers.find((p) => p?.id === id)?.name || id}
                  <span className={styles['ev__filter-chip-count']}>
                    {llmAvailable.filter((m) => m.provider_id === id).length}
                  </span>
                </button>
              );
            })}
            </>
          ),
        })}

        {/* Model cards (original card design). Retries and timeout for the
            selected models are managed in the side panel, so the cards
            don't carry those fields here. */}
        <div className={styles['ev__models-scroll']}>
          {modelsLoading ? (
            <div className={`${styles.ev__grid} ${styles['ev__grid--wide']}`} aria-busy="true" aria-label="Refreshing models">
              {Array.from({ length: SKELETON_CARD_COUNT }).map((_, i) => (
                <div key={i} className={styles['ev__skel-mcard']}>
                  <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--line']}`} style={{ width: '60%', height: 14 }} />
                  <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--line']}`} style={{ width: '35%' }} />
                  <span className={styles['ev__skel-caps']}>
                    <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--tag']}`} />
                    <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--tag']}`} />
                  </span>
                  <span className={styles['ev__skel-stats']}>
                    <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--stat']}`} />
                    <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--stat']}`} />
                  </span>
                </div>
              ))}
            </div>
          ) : (
            <div className={`${styles.ev__grid} ${styles['ev__grid--wide']}`}>
              {tableModels.map((m) => renderModelCard(m, () => toggleModel(m.id), false))}
              {tableModels.length === 0 && (
                <p className={styles.ev__empty}>
                  {modelSearch ? `No models match "${modelSearch}".` : 'No models for this filter.'}
                </p>
              )}
            </div>
          )}
        </div>
      </div>

      {/* Retries and timeout stay pinned at the top of the panel; the list
          of selected models scrolls underneath. */}
      <aside className={`${styles.ev__panel} ${styles['ev__panel--pinned']}`}>
        <div className={styles['ev__panel-fixed']}>
          <p className={styles['ev__panel-title']}>Retries and timeout</p>
          <div className={`${styles.ev__segctl} ${styles['ev__segctl--full']}`}>
            {[
              { v: 'all' as const, l: 'Same for all' },
              { v: 'individual' as const, l: 'Per model' },
            ].map((o) => (
              <button
                key={o.v}
                type="button"
                className={`${styles.ev__segbtn} ${draft.retryConfigMode === o.v ? styles['ev__segbtn--on'] : ''}`}
                onClick={() => dispatch(setDraft({ retryConfigMode: o.v }))}
              >
                {o.l}
              </button>
            ))}
          </div>
          {draft.retryConfigMode === 'all' && retryFields(draft.retryConfigAll, updateRetryConfigAll, clampRetryConfigAll)}
          <p className={styles['ev__panel-note']}>
            {draft.retryConfigMode === 'individual' ? 'Set on each model below. ' : ''}Retries {MIN_MAX_RETRIES} to{' '}
            {MAX_MAX_RETRIES}. Timeout {MIN_TIMEOUT} to {MAX_TIMEOUT} seconds.
          </p>
        </div>
        <div className={styles['ev__panel-scroll']}>
          <p className={styles['ev__panel-title']}>
            In this run: {selectedModels.length} model{selectedModels.length === 1 ? '' : 's'}
          </p>
          {selectedModels.length === 0 ? (
            <p className={styles['ev__panel-note']}>Nothing selected yet. Pick a healthy model card.</p>
          ) : (
            <div className={styles['ev__panel-list']}>
              {selectedModels.map((m) => {
                const health: HealthStatus = healthById?.[m.id] ?? 'idle';
                const ready = health === 'success' && Boolean(verifiedModelIds[m.id]);
                return (
                  <div key={m.id} className={styles.ev__pm}>
                    <div className={styles['ev__pm-head']}>
                      <span
                        className={`${styles['ev__pm-dot']} ${
                          ready ? styles['ev__pm-dot--ok'] : health === 'failed' ? styles['ev__pm-dot--fail'] : ''
                        }`}
                        title={ready ? 'Healthy' : health === 'failed' ? 'Health check failed' : 'Waiting for health check'}
                      />
                      <span className={styles['ev__pm-name']}>
                        {m.name || 'Unnamed model'}
                        <small>
                          {providers.find((p) => p?.id === m.provider_id)?.name ?? m.provider_id}
                          {!ready && (health === 'failed' ? ' · health check failed' : ' · waiting for health check')}
                        </small>
                      </span>
                      <button
                        type="button"
                        className={styles['ev__panel-x']}
                        onClick={() => toggleModel(m.id)}
                        aria-label={`Remove ${m.name || 'model'}`}
                      >
                        <X size={14} />
                      </button>
                    </div>
                    {draft.retryConfigMode === 'individual' && (
                      <div className={styles['ev__pm-body']}>
                        {retryFields(
                          draft.modelRetryConfig[m.id],
                          (field, raw) => updateModelRetryConfig(m.id, field, raw),
                          (field) => clampModelRetryConfig(m.id, field)
                        )}
                      </div>
                    )}
                  </div>
                );
              })}
            </div>
          )}
        </div>
      </aside>
    </div>
  );

  // ---- Test Suite step (harness): scorecard coverage panel ----------------
  const coveragePanel = (
    <aside className={styles.ev__panel}>
      <p className={styles['ev__panel-title']}>Scorecard coverage</p>
      <p className={styles['ev__panel-sub']}>
        {coveredUseCases.length} of {harnessUseCases.length} use cases covered
      </p>
      <div className={styles['ev__panel-list']}>
        {harnessUseCases.map((u) => {
          const on = coveredUseCases.includes(u);
          return (
            <div key={u} className={`${styles['ev__panel-item']} ${on ? '' : styles['ev__panel-item--off']}`}>
              <div className={styles['ev__panel-cov']}>
                <span className={`${styles.ev__dot} ${on ? styles['ev__dot--on'] : ''}`}>
                  {on && <Check size={10} strokeWidth={3.5} />}
                </span>
                {on ? u : `${u}: not covered`}
              </div>
            </div>
          );
        })}
      </div>
      <div className={styles['ev__panel-stats']}>
        <div>
          <span>Suites selected</span>
          <b>{selectedHarnessSuites.length}</b>
        </div>
        <div>
          <span>Questions available</span>
          <b>{selectedHarnessSuites.reduce((n, d) => n + (d.question_count ?? 0), 0).toLocaleString()}</b>
        </div>
      </div>
      <p className={styles['ev__panel-note']}>You choose how many questions to run in the next step.</p>
    </aside>
  );

  // ---- Review step: banner + settings rows --------------------------------
  const metricDisplayNames = draft.metrics.map((m) => customMetricsCatalog.find((c) => c.id === m)?.name ?? m);
  const reviewStats = isHarness
    ? [
        { k: 'Runs', v: harnessRunCount.toLocaleString() },
        { k: 'Tasks', v: harnessTaskCount.toLocaleString() },
        { k: 'Scorers', v: String(draft.metrics.length) },
      ]
    : [
        { k: 'Questions', v: suite ? suiteQuestions(suite.question_count ?? 0).toLocaleString() : '—' },
        { k: 'Models', v: String(selectedModels.length) },
        { k: 'Metrics', v: hideMetricsStep ? '—' : String(draft.metrics.length) },
      ];
  const reviewSummaryLine = isHarness
    ? `Agent harness comparing ${harnessAgentTitles.join(' and ') || 'no agents yet'}`
    : `${typeLabel || 'Evaluation'}${frameworkTitle ? ` (${frameworkTitle})` : ''} on ${suite?.name ?? 'no suite yet'}`;
  const retryText =
    draft.retryConfigMode === 'all'
      ? `${clampRetryField('max_retries', draft.retryConfigAll.max_retries)} ${
          clampRetryField('max_retries', draft.retryConfigAll.max_retries) === 1 ? 'retry' : 'retries'
        }, ${clampRetryField('timeout', draft.retryConfigAll.timeout)} second timeout`
      : 'Retries and timeout set per model';
  const retestShown =
    draft.type === 'model' || draft.type === 'rag' || (draft.type === 'agent' && Boolean(draft.agentFramework));
  const reviewRows: { k: string; v: string; step: number }[] = [
    { k: 'Type', v: typeManifestValue || '—', step: 1 },
    { k: 'Providers', v: providerNames.join(', ') || '—', step: 2 },
    {
      k: 'Models',
      v: selectedModels.length > 0 ? `${selectedModels.map((m) => m.name || 'Unnamed model').join(', ')}. ${retryText}` : '—',
      step: 3,
    },
    {
      k: isHarness ? 'Test suites' : 'Test suite',
      v: isHarness
        ? selectedHarnessSuites.map((d) => d.name).join(', ') || '—'
        : [
            suite?.name ?? '—',
            draft.subgroup.length > 0 ? `${draft.subgroup.length} subgroup${draft.subgroup.length === 1 ? '' : 's'}` : null,
            isRag ? `Top K ${draft.topK}` : null,
          ]
            .filter(Boolean)
            .join(', '),
      step: 4,
    },
    {
      k: 'Sample size',
      v: sampleN === null ? 'Every question' : `${sampleN} questions per ${isHarness ? 'suite' : 'model'}`,
      step: 5,
    },
    ...(hideMetricsStep
      ? []
      : [{ k: isHarness ? 'Scorers' : 'Metrics', v: metricDisplayNames.join(', ') || '—', step: 5 }]),
    { k: 'Judge model', v: judgeModel ? judgeModel.name || 'Unnamed model' : '—', step: 5 },
    ...(isHarness
      ? [
          {
            k: 'Limits per task',
            v: `${clampHarnessSteps(draft.harnessSettings.max_steps)} steps, ${clampRetryField(
              'timeout',
              draft.harnessSettings.task_timeout
            )} seconds, ${draft.harnessSettings.save_traces ? 'traces saved for replay' : 'traces not saved'}`,
            step: 5,
          },
        ]
      : []),
    ...(retestShown && draft.retestOnWrong
      ? [
          {
            k: 'Retest on wrong',
            v: `Up to ${clampRetestRounds(draft.retestMaxRounds)} rounds, verified by ${draft.retestVerifyMetric ?? '—'}`,
            step: 5,
          },
        ]
      : []),
    ...(draft.instruction.trim() ? [{ k: 'Instruction', v: draft.instruction.trim(), step: 5 }] : []),
  ];

  return (
    <div className="page-enter" style={{ height: '100%', display: 'flex', flexDirection: 'column' }}>
      {/* ---- header (matches History/Reports/Comparison/Sidebar pattern) ---- */}
      <div className={styles['ev__header']}>
        <div>
          <p className={styles['ev__header-eyebrow']}>Evaluation console</p>
          <h1>New run</h1>
          <p className={styles['ev__header-sub']}>Assemble and launch a new evaluation run</p>
        </div>
        <div className={styles['ev__header-meta']}>
          <span className={styles['ev__header-status']} data-state={launching ? 'live' : 'draft'}>
            {launching ? 'Launching' : 'Draft'}
          </span>
        </div>
      </div>

      <div className={styles.page}>
        <div className={styles.ev}>
          {/* ---- shell ---- */}
          <div className={styles.ev__shell}>
            {/* SIGNATURE: Run Manifest */}
            <aside className={styles.ev__manifest}>
              <div className={styles['ev__manifest-head']}>
                <div className={styles['ev__manifest-eyebrow']}>
                  <span>Run manifest</span>
                  <span className={styles['ev__manifest-pct']}>{Math.round((step / totalSteps) * 100)}%</span>
                </div>
                <div className={styles['ev__manifest-title']} data-empty={!draft.name}>
                  {draft.name || 'Untitled run'}
                </div>
                <div className={styles.ev__segs}>
                  {STEPS.map((st, i) => (
                    <span
                      key={st.label}
                      className={`${styles.ev__seg} ${
                        i < step ? styles['ev__seg--done'] : i === step ? styles['ev__seg--active'] : ''
                      }`}
                    />
                  ))}
                </div>
                <p className={styles['ev__manifest-done']}>
                  {step} of {totalSteps} steps done
                </p>
              </div>
              <div className={styles.ev__spec}>
                {STEPS.map((st, i) => {
                  const state = i === step ? 'active' : i < step ? 'done' : 'todo';
                  const row = manifest[i];
                  return (
                    <button
                      key={st.label}
                      type="button"
                      className={`${styles['ev__spec-row']} ${styles[`ev__spec-row--${state}`]}`}
                      onClick={() => goToStep(i)}
                      disabled={i > step}
                    >
                      <span className={styles['ev__spec-tick']}>
                        {state === 'done' ? <Check size={13} strokeWidth={3} /> : i + 1}
                      </span>
                      <span className={styles['ev__spec-body']}>
                        <span className={styles['ev__spec-label']}>{st.label}</span>
                        <span className={styles['ev__spec-value']} data-empty={row.empty}>
                          {row.value}
                        </span>
                      </span>
                      {state === 'done' && <span className={styles['ev__spec-edit']}>Edit</span>}
                    </button>
                  );
                })}
              </div>
              <p className={styles['ev__manifest-foot']}>Select a completed step to go back and edit it.</p>
            </aside>

            {/* STAGE */}
            <section className={styles.ev__stage}>
              <div className={styles['ev__stage-head']}>
                <p className={styles['ev__stage-step']}>
                  Step {step + 1} of {totalSteps}
                </p>
                <h2 className={styles['ev__stage-title']}>{stageTitle}</h2>
                <p className={styles['ev__stage-sub']}>{stageSub}</p>
              </div>

              <div className={styles['ev__stage-body']}>
                <StepErrorBoundary
                  key={`${step}-${stepRetryKey}`}
                  onRetry={() => setStepRetryKey((k) => k + 1)}
                  onBack={goBack}
                  canGoBack={step > 0}
                >
                <div className={styles.ev__anim} style={{ flex: 1, minHeight: 0, display: 'flex', flexDirection: 'column' }}>
                  {/* STEP 0 — NAME */}
                  {/* Left: the name field, quick-add tokens, presets and naming
                      tips. Right: a live preview of the row this run will get in
                      History, so "how it appears in your history" is something
                      you can see while typing. */}
                  {step === 0 && (
                    <div className={styles.ev__namegrid}>
                      <div className={styles['ev__namegrid-main']}>
                        <div className={styles.ev__field}>
                          <label className={styles.ev__label}>Run name</label>
                          <div className={styles['ev__name-wrap']}>
                            <input
                              className={styles['ev__name-input']}
                              placeholder="Untitled run"
                              value={draft.name}
                              onChange={(e) => dispatch(setDraft({ name: e.target.value }))}
                              onKeyDown={(e) => {
                                if (e.key === 'Enter' && canGo()) goNext();
                              }}
                              autoFocus
                            />
                            {draft.name && (
                              <button
                                type="button"
                                className={styles['ev__name-clear']}
                                onClick={() => dispatch(setDraft({ name: '' }))}
                                aria-label="Clear name"
                              >
                                <X size={14} />
                              </button>
                            )}
                          </div>
                          {draft.name.trim() ? (
                            <p className={`${styles['ev__name-caption']} ${styles['ev__name-caption--ok']}`}>
                              <Check size={13} strokeWidth={3} /> Ready. Press Enter to continue, or keep refining.
                            </p>
                          ) : (
                            <p className={styles['ev__name-caption']}>
                              <Tag size={13} /> This is how the run appears in your history.
                            </p>
                          )}
                        </div>

                        {/* Quick add: append a period, version or date to the name
                            in one click (click again to take it back off). */}
                        <div className={styles.ev__quick}>
                          <p className={styles['ev__quick-head']}>Quick add</p>
                          <div className={styles['ev__quick-row']}>
                            {nameTokens.map((t) => {
                              const on = draft.name.toLowerCase().includes(t.toLowerCase());
                              return (
                                <button
                                  key={t}
                                  type="button"
                                  className={`${styles.ev__preset} ${on ? styles['ev__preset--on'] : ''}`}
                                  aria-pressed={on}
                                  onClick={() => toggleNameToken(t)}
                                >
                                  {on ? <Check size={13} strokeWidth={3} /> : <Plus size={13} strokeWidth={2.5} />} {t}
                                </button>
                              );
                            })}
                          </div>
                        </div>

                          <div className={styles.ev__quick}>
                            <p className={styles['ev__quick-head']}>Presets</p>
                            <div className={styles['ev__quick-row']}>
                              {SUGGESTED_NAMES.map((s) => {
                                const on = draft.name === s;
                                return (
                                  <button
                                    key={s}
                                    type="button"
                                    className={`${styles.ev__preset} ${on ? styles['ev__preset--on'] : ''}`}
                                    onClick={() => dispatch(setDraft({ name: s }))}
                                  >
                                    {on ? <Check size={13} strokeWidth={3} /> : <Plus size={13} strokeWidth={2.5} />} {s}
                                  </button>
                                );
                              })}
                            </div>
                          </div>

                          <div className={styles.ev__note}>
                            <span className={styles['ev__note-icon']}>
                              <Lightbulb size={16} />
                            </span>
                            <div>
                              <p className={styles['ev__note-title']}>What makes a good name</p>
                              <ul className={styles['ev__note-list']}>
                                {NAMING_TIPS.map((tip) => (
                                  <li key={tip}>{tip}</li>
                                ))}
                              </ul>
                            </div>
                          </div>
                      </div>

                      <aside className={styles.ev__hprev}>
                        <p className={styles['ev__hprev-head']}>Preview in History</p>
                        <div className={`${styles['ev__hprev-card']} ${draft.name.trim() ? styles['ev__hprev-card--live'] : ''}`}>
                          <div className={styles['ev__hprev-top']}>
                            <span className={styles['ev__hprev-icon']}>
                              <Tag size={15} />
                            </span>
                            <span className={styles['ev__hprev-name']} data-empty={!draft.name.trim()}>
                              {draft.name.trim() || 'Untitled run'}
                            </span>
                          </div>
                          <div className={styles['ev__hprev-badges']}>
                            <span className={styles.ev__tag}>{typeLabel || 'Type not set'}</span>
                            <span className={styles['ev__hprev-status']}>Draft</span>
                          </div>
                          <div className={styles['ev__hprev-meta']}>
                            <span>{new Date().toLocaleDateString()}</span>
                            <span>
                              {draft.models.length} model{draft.models.length === 1 ? '' : 's'}
                            </span>
                          </div>
                        </div>
                        <p className={styles['ev__hprev-note']}>
                          Updates as you type. The type and models fill in as you complete the next steps.
                        </p>
                      </aside>
                    </div>
                  )}

                  {/* STEP 1 — TYPE */}
                  {step === 1 && (
                    <>
                      <div className={styles.ev__options}>
                        {TYPE_OPTIONS.map((o) => {
                          const Icon = o.icon;
                          const on = draft.type === o.value;
                          return (
                            <button
                              key={o.value}
                              type="button"
                              className={`${styles.ev__option} ${on ? styles['ev__option--on'] : ''} ${
                                o.disabled ? styles['ev__option--off'] : ''
                              }`}
                              onClick={() => !o.disabled && dispatch(setDraftType(o.value))}
                              disabled={o.disabled}
                            >
                              <span
                                className={`${styles['ev__option-icon']} ${
                                  o.variant ? styles[`ev__option-icon--${o.variant}`] : ''
                                }`}
                              >
                                <Icon size={20} />
                              </span>
                              <span className={styles['ev__option-main']}>
                                <span className={styles['ev__option-name']}>
                                  {o.label}
                                  {o.disabled && <span className={styles.ev__badge}>Soon</span>}
                                </span>
                                <span className={styles['ev__option-desc']}>{o.sub}</span>
                              </span>
                              {on && (
                                <span className={styles.ev__mark}>
                                  <Check size={13} strokeWidth={3} />
                                </span>
                              )}
                            </button>
                          );
                        })}
                      </div>

                      {/* About the selected type. For Agent it goes below the
                          agent options instead, so those stay near the top. */}
                      {draft.type !== 'agent' && typeAboutPanel}

                      {/* Agent setup — the two follow-up choices that only apply to
                          Agent, as one panel with numbered steps:
                            1. how to test (Standard benchmark / Document-agent harness)
                            2. harness: which agents to compare (required)
                               standard: which framework (optional)
                          The two modes are mutually exclusive, so switching clears
                          agentFramework (harness) / harness picks (standard). */}
                      {draft.type === 'agent' && (
                        <div className={`${styles.ev__asetup} ${styles.ev__anim}`}>
                          <div className={styles['ev__asetup-head']}>
                            <span className={`${styles['ev__option-icon']} ${styles['ev__option-icon--agent']}`}>
                              <Bot size={18} />
                            </span>
                            <div className={styles['ev__asetup-title']}>
                              <h3>Agent setup</h3>
                              <p>Two more choices, because you picked Agent.</p>
                            </div>
                            {isHarness && draft.harnessAgents.length === 0 ? (
                              <span className={`${styles['ev__asetup-status']} ${styles['ev__asetup-status--todo']}`}>
                                1 choice left
                              </span>
                            ) : (
                              <span className={styles['ev__asetup-status']}>
                                <Check size={12} strokeWidth={3} /> All set
                              </span>
                            )}
                          </div>

                          {/* 1 — how to test */}
                          <div className={styles['ev__asetup-step']}>
                            <span className={`${styles['ev__asetup-n']} ${styles['ev__asetup-n--done']}`}>
                              <Check size={12} strokeWidth={3} />
                            </span>
                            <div className={styles['ev__asetup-body']}>
                              <p className={styles['ev__asetup-q']}>How should the agent be tested?</p>
                              <div className={styles['ev__asetup-grid']}>
                                {[
                                  {
                                    v: false,
                                    icon: Bot,
                                    l: 'Standard benchmark',
                                    d: 'Run one agent on a single agent benchmark. You can add the framework it runs on.',
                                    tags: ['One benchmark', 'Optional framework'],
                                  },
                                  {
                                    v: true,
                                    icon: Layers,
                                    l: 'Document-agent harness',
                                    d: 'Compare agents on the same document tasks, each in a clean sandbox, with one scorecard.',
                                    tags: [...HARNESS_USE_CASES],
                                  },
                                ].map((o) => {
                                  const on = draft.agentHarness === o.v;
                                  const ModeIcon = o.icon;
                                  return (
                                    <button
                                      key={String(o.v)}
                                      type="button"
                                      className={`${styles.ev__acard} ${on ? styles['ev__acard--on'] : ''}`}
                                      aria-pressed={on}
                                      onClick={() => {
                                        if (on) return;
                                        dispatch(
                                          setDraft({
                                            agentHarness: o.v,
                                            agentFramework: o.v ? null : draft.agentFramework,
                                            harnessAgents: o.v ? draft.harnessAgents : [],
                                            harnessSuiteIds: [],
                                            // the scorer catalog differs per mode, so drop stale picks
                                            metrics: [],
                                            retestOnWrong: false,
                                            retestVerifyMetric: null,
                                          })
                                        );
                                      }}
                                    >
                                      <span className={styles['ev__acard-head']}>
                                        <span className={`${styles.ev__radio} ${on ? styles['ev__radio--on'] : ''}`} />
                                        <span className={styles['ev__srow-icon']}>
                                          <ModeIcon size={15} />
                                        </span>
                                        <span className={styles['ev__acard-name']}>{o.l}</span>
                                      </span>
                                      <span className={styles['ev__acard-desc']}>{o.d}</span>
                                      <span className={styles['ev__acard-tags']}>
                                        {o.tags.map((t) => (
                                          <span key={t} className={`${styles.ev__pilltag} ${on ? '' : styles['ev__pilltag--muted']}`}>
                                            {t}
                                          </span>
                                        ))}
                                      </span>
                                    </button>
                                  );
                                })}
                              </div>
                            </div>
                          </div>

                          {/* 2 — harness: agents to compare (required) */}
                          {isHarness && (
                            <div className={styles['ev__asetup-step']}>
                              <span
                                className={`${styles['ev__asetup-n']} ${
                                  draft.harnessAgents.length > 0 ? styles['ev__asetup-n--done'] : styles['ev__asetup-n--active']
                                }`}
                              >
                                {draft.harnessAgents.length > 0 ? <Check size={12} strokeWidth={3} /> : 2}
                              </span>
                              <div className={styles['ev__asetup-body']}>
                                <p className={styles['ev__asetup-q']}>
                                  Which agents should be compared? <span>Pick one or more</span>
                                </p>
                                <div className={styles['ev__asetup-grid']}>
                                  {HARNESS_AGENTS.map((a) => {
                                    const on = draft.harnessAgents.includes(a.id);
                                    return (
                                      <button
                                        key={a.id}
                                        type="button"
                                        className={`${styles.ev__acard} ${on ? styles['ev__acard--on'] : ''}`}
                                        aria-pressed={on}
                                        onClick={() =>
                                          dispatch(
                                            setDraft({ harnessAgents: toggle(draft.harnessAgents, a.id) as HarnessAdapterId[] })
                                          )
                                        }
                                      >
                                        <span className={styles['ev__acard-head']}>
                                          <span className={`${styles.ev__check} ${on ? styles['ev__check--on'] : ''}`}>
                                            {on && <Check size={11} strokeWidth={3} />}
                                          </span>
                                          <span className={styles['ev__srow-icon']}>
                                            <Waypoints size={15} />
                                          </span>
                                          <span className={styles['ev__acard-name']}>{a.title}</span>
                                        </span>
                                        <span className={styles['ev__acard-desc']}>{a.desc}</span>
                                      </button>
                                    );
                                  })}
                                </div>
                                {draft.harnessAgents.length === 0 ? (
                                  <p className={`${styles['ev__asetup-sum']} ${styles['ev__asetup-sum--req']}`}>
                                    <AlertTriangle size={13} /> Select at least one agent to continue.
                                  </p>
                                ) : (
                                  <p className={styles['ev__asetup-sum']}>
                                    <Info size={13} />
                                    {draft.harnessAgents.length === 1 ? (
                                      <span>
                                        <b>{harnessAgentTitles[0]}</b> runs alone. Add another agent to compare them side by side.
                                      </span>
                                    ) : (
                                      <span>
                                        <b>{harnessAgentTitles.join(' vs ')}</b>: same tasks, same sandbox, one scorecard.
                                      </span>
                                    )}
                                  </p>
                                )}
                              </div>
                            </div>
                          )}

                          {/* 2 — standard: framework (optional) */}
                          {!isHarness && (
                            <div className={styles['ev__asetup-step']}>
                              <span
                                className={`${styles['ev__asetup-n']} ${draft.agentFramework ? styles['ev__asetup-n--done'] : ''}`}
                              >
                                {draft.agentFramework ? <Check size={12} strokeWidth={3} /> : 2}
                              </span>
                              <div className={styles['ev__asetup-body']}>
                                <p className={styles['ev__asetup-q']}>
                                  Which framework does the agent run on? <span>Optional</span>
                                </p>
                                <div className={styles['ev__asetup-grid']}>
                                  {AGENT_FRAMEWORKS.map((f) => {
                                    const on = draft.agentFramework === f.id;
                                    return (
                                      <button
                                        key={f.id}
                                        type="button"
                                        className={`${styles.ev__acard} ${on ? styles['ev__acard--on'] : ''}`}
                                        aria-pressed={on}
                                        onClick={() => dispatch(setDraft({ agentFramework: on ? null : f.id }))}
                                      >
                                        <span className={styles['ev__acard-head']}>
                                          <span className={`${styles.ev__check} ${on ? styles['ev__check--on'] : ''}`}>
                                            {on && <Check size={11} strokeWidth={3} />}
                                          </span>
                                          <span className={styles['ev__srow-icon']}>
                                            <Waypoints size={15} />
                                          </span>
                                          <span className={styles['ev__acard-name']}>{f.title}</span>
                                        </span>
                                        <span className={styles['ev__acard-desc']}>{f.desc}</span>
                                      </button>
                                    );
                                  })}
                                </div>
                                <p className={styles['ev__asetup-sum']}>
                                  <Info size={13} />
                                  {draft.agentFramework ? (
                                    <span>
                                      Runs on <b>{frameworkTitle}</b>. This also decides which test suites the next steps offer.
                                    </span>
                                  ) : (
                                    <span>No framework selected. The run uses an agent benchmark, and scoring needs only a judge model.</span>
                                  )}
                                </p>
                              </div>
                            </div>
                          )}
                        </div>
                      )}

                      {draft.type === 'agent' && typeAboutPanel}
                    </>
                  )}

                  {/* STEP 2 — PROVIDERS */}
                  {step === 2 && (
                    <div className={styles.ev__scroll}>
                      {listHead({
                        title: 'Connected providers',
                        meta: (
                          <>
                            {filteredProviders.length} of {connectedProviders.length} shown · <b>{draft.providers.length}</b> selected
                          </>
                        ),
                        count: filteredProviders.length,
                        search: { placeholder: 'Search providers', value: providerSearch, onChange: setProviderSearch },
                        refresh: { onClick: refreshProviders, busy: providersLoading, title: 'Refresh providers' },
                      })}
                      {providersLoading ? (
                        <div className={styles.ev__grid} aria-busy="true" aria-label="Refreshing providers">
                          {Array.from({ length: SKELETON_CARD_COUNT }).map((_, i) => (
                            <div key={i} className={styles['ev__skel-pcard']}>
                              <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--icon']}`} />
                              <span className={styles['ev__skel-lines']}>
                                <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--line']}`} style={{ width: '70%' }} />
                                <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--line']}`} style={{ width: '45%' }} />
                                <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--pill']}`} />
                              </span>
                            </div>
                          ))}
                        </div>
                      ) : (
                        <div className={`${styles.ev__grid} ${styles['ev__grid--providers']}`}>
                          {filteredProviders.map((p) => {
                            const on = draft.providers.includes(p.id);
                            return (
                              <button
                                key={p.id}
                                type="button"
                                className={`${styles.ev__option} ${styles['ev__option--provider']} ${
                                  on ? styles['ev__option--on'] : ''
                                }`}
                                onClick={() => dispatch(setDraft({ providers: toggle(draft.providers, p.id) }))}
                                aria-pressed={on}
                              >
                                {/* Same card as the Type step: icon tile, name,
                                    description, selection mark on the right. */}
                                <span className={styles['ev__option-icon']}>{providerInitials(p.name)}</span>
                                <span className={styles['ev__option-main']}>
                                  <span className={styles['ev__option-name']}>
                                    {p.name || 'Unnamed provider'}
                                    <span className={styles['ev__pv-status']}>Connected</span>
                                  </span>
                                  {p.description && (
                                    <span className={styles['ev__option-desc']} title={p.description}>
                                      {p.description}
                                    </span>
                                  )}
                                  <span className={styles['ev__option-meta']}>
                                    <Cpu size={13} />
                                    <b>{p.model_count ?? 0}</b> model{(p.model_count ?? 0) === 1 ? '' : 's'} available
                                  </span>
                                </span>
                                {on && (
                                  <span className={styles.ev__mark}>
                                    <Check size={13} strokeWidth={3} />
                                  </span>
                                )}
                              </button>
                            );
                          })}
                          {connectedProviders.length === 0 && (
                            <p className={styles.ev__empty}>No connected providers yet. Connect one from the Providers page to continue.</p>
                          )}
                          {connectedProviders.length > 0 && filteredProviders.length === 0 && (
                            <p className={styles.ev__empty}>No providers match "{providerSearch}".</p>
                          )}
                        </div>
                      )}
                    </div>
                  )}

                  {/* STEP 3 — MODELS */}
                  {step === 3 &&
                    (availableModels.length > 0 ? (
                      !isRag ? (
                        modelsTableView
                      ) : (
                      <div className={styles['ev__models-panel']}>
                        <div className={styles['ev__step-toolbar']}>
                          {/* RAG uses per-category search boxes instead (see
                              the Embedding/Reranker/LLM columns below) — no
                              shared global search box for it. */}
                          {!isRag && (
                            <div className={styles['ev__toolbar-search']}>
                              <Search size={14} />
                              <input
                                placeholder="Search models…"
                                value={modelSearch}
                                onChange={(e) => setModelSearch(e.target.value)}
                              />
                            </div>
                          )}
                          <button
                            type="button"
                            className={styles['ev__toolbar-refresh']}
                            onClick={refreshModels}
                            disabled={modelsLoading}
                            title="Refresh models"
                          >
                            <RefreshCw size={14} className={modelsLoading ? styles.ev__spin : ''} />
                          </button>
                        </div>

                        {(!modelsFetchedThisVisit || selectedModelsNotReady.length > 0) && (
                          <p className={styles['ev__metrics-required']}>
                            {!modelsFetchedThisVisit || selectedModelsChecking
                              ? 'Checking the health of your selected models — you can continue once they\u2019re all available.'
                              : 'Some selected models aren\u2019t available yet — run a health check on them (or deselect them) to continue.'}
                          </p>
                        )}

                        {/* Retry/timeout mode — lets the user apply one shared
                            max_retries/timeout to every selected model at once
                            ("Apply to all"), or set it per model on each card
                            below ("Individually"). Only relevant once at least
                            one model is selected. */}
                        {draft.models.length > 0 && (
                          <div className={styles['ev__retry-mode']}>
                            <div className={styles['ev__retry-mode-head']}>
                              <label className={styles.ev__label}>Retries &amp; timeout</label>
                              <div className={styles['ev__radio-row']}>
                                <button
                                  type="button"
                                  className={`${styles['ev__radio-opt']} ${
                                    draft.retryConfigMode === 'all' ? styles['ev__radio-opt--on'] : ''
                                  }`}
                                  onClick={() => dispatch(setDraft({ retryConfigMode: 'all' }))}
                                >
                                  <span
                                    className={`${styles.ev__radio} ${
                                      draft.retryConfigMode === 'all' ? styles['ev__radio--on'] : ''
                                    }`}
                                  />
                                  Apply to all
                                </button>
                                <button
                                  type="button"
                                  className={`${styles['ev__radio-opt']} ${
                                    draft.retryConfigMode === 'individual' ? styles['ev__radio-opt--on'] : ''
                                  }`}
                                  onClick={() => dispatch(setDraft({ retryConfigMode: 'individual' }))}
                                >
                                  <span
                                    className={`${styles.ev__radio} ${
                                      draft.retryConfigMode === 'individual' ? styles['ev__radio--on'] : ''
                                    }`}
                                  />
                                  Individually
                                </button>
                              </div>
                            </div>

                            {draft.retryConfigMode === 'all' ? (
                              <div className={styles['ev__retry-mode-all']}>
                                <div className={styles['ev__mcard-retry-field']}>
                                  <label>Max retries</label>
                                  <input
                                    type="number"
                                    min={MIN_MAX_RETRIES}
                                    max={MAX_MAX_RETRIES}
                                    value={displayNumInput(draft.retryConfigAll.max_retries)}
                                    onChange={(e) => updateRetryConfigAll('max_retries', parseNumInput(e.target.value))}
                                    onBlur={() => clampRetryConfigAll('max_retries')}
                                  />
                                </div>
                                <div className={styles['ev__mcard-retry-field']}>
                                  <label>Timeout (s)</label>
                                  <input
                                    type="number"
                                    min={MIN_TIMEOUT}
                                    max={MAX_TIMEOUT}
                                    value={displayNumInput(draft.retryConfigAll.timeout)}
                                    onChange={(e) => updateRetryConfigAll('timeout', parseNumInput(e.target.value))}
                                    onBlur={() => clampRetryConfigAll('timeout')}
                                  />
                                </div>
                                <p className={styles['ev__retry-mode-note']}>
                                  <Info size={11} /> Retries {MIN_MAX_RETRIES}–{MAX_MAX_RETRIES} · Timeout {MIN_TIMEOUT}
                                  –{MAX_TIMEOUT}s — applied to all {draft.models.length} selected model
                                  {draft.models.length === 1 ? '' : 's'}.
                                </p>
                              </div>
                            ) : (
                              <p className={styles['ev__retry-mode-note']}>
                                <Info size={11} /> Retries {MIN_MAX_RETRIES}–{MAX_MAX_RETRIES} · Timeout {MIN_TIMEOUT}
                                –{MAX_TIMEOUT}s — set on each selected model's card below.
                              </p>
                            )}
                          </div>
                        )}
                        <div className={styles['ev__models-scroll']}>
                        {modelsLoading ? (
                          <div className={`${styles.ev__grid} ${styles['ev__grid--wide']}`} aria-busy="true" aria-label="Refreshing models">
                            {Array.from({ length: SKELETON_CARD_COUNT }).map((_, i) => (
                              <div key={i} className={styles['ev__skel-mcard']}>
                                <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--line']}`} style={{ width: '60%', height: 14 }} />
                                <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--line']}`} style={{ width: '35%' }} />
                                <span className={styles['ev__skel-caps']}>
                                  <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--tag']}`} />
                                  <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--tag']}`} />
                                </span>
                                <span className={styles['ev__skel-stats']}>
                                  <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--stat']}`} />
                                  <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--stat']}`} />
                                </span>
                              </div>
                            ))}
                          </div>
                        ) : draft.type === 'rag' ? (
                          // RAG: exactly three columns, always in this order
                          // — Embedding, Reranker, LLM. Embedding/Reranker
                          // are single-select (toggleModelForCategory);
                          // LLM is multi-select (plain toggleModel). Each
                          // column has its own search box (ragCategorySearch)
                          // instead of the shared global search used by the
                          // Model/Agent grid.
                          <div className={styles['ev__rag-columns']}>
                            {RAG_CATEGORIES.map((category) => {
                              const allCategoryModels = modelsByCategory.get(category) ?? [];
                              const categoryModels = filteredModelsByCategory.get(category) ?? [];
                              const allCategoryIds = allCategoryModels.map((m) => m.id);
                              const selectedCount = allCategoryModels.filter((m) => draft.models.includes(m.id)).length;
                              const isMultiSelect = category === 'LLM';
                              return (
                                <div key={category} className={styles['ev__rag-col']}>
                                  <div className={styles['ev__rag-col-head']}>
                                    <span className={styles['ev__rag-col-title']}>{category}</span>
                                    <span
                                      className={`${styles['ev__rag-col-status']} ${
                                        selectedCount > 0 ? styles['ev__rag-col-status--on'] : ''
                                      }`}
                                    >
                                      {selectedCount > 0 ? (
                                        <>
                                          <Check size={11} strokeWidth={3} /> {selectedCount} selected
                                        </>
                                      ) : isMultiSelect ? (
                                        'Select one or more'
                                      ) : (
                                        'Select one'
                                      )}
                                    </span>
                                  </div>

                                  <div className={styles['ev__rag-col-search']}>
                                    <Search size={12} />
                                    <input
                                      placeholder={`Search ${category.toLowerCase()}…`}
                                      value={ragCategorySearch[category] ?? ''}
                                      onChange={(e) =>
                                        setRagCategorySearch((prev) => ({ ...prev, [category]: e.target.value }))
                                      }
                                    />
                                  </div>

                                  <div className={styles['ev__rag-col-list']}>
                                    {categoryModels.map((m) =>
                                      renderModelCard(
                                        m,
                                        isMultiSelect
                                          ? () => toggleModel(m.id)
                                          : () => toggleModelForCategory(m.id, allCategoryIds)
                                      )
                                    )}
                                    {categoryModels.length === 0 && allCategoryModels.length > 0 && (
                                      <p className={styles.ev__empty}>
                                        No models match "{ragCategorySearch[category]}".
                                      </p>
                                    )}
                                    {allCategoryModels.length === 0 && (
                                      <p className={styles.ev__empty}>No {category} models available.</p>
                                    )}
                                  </div>
                                </div>
                              );
                            })}
                          </div>
                        ) : (
                        <div className={`${styles.ev__grid} ${styles['ev__grid--wide']}`}>
                          {filteredModels.map((m) => renderModelCard(m, () => toggleModel(m.id)))}
                          {filteredModels.length === 0 && (
                            <p className={styles.ev__empty}>No models match "{modelSearch}".</p>
                          )}
                        </div>
                        )}
                        </div>
                      </div>
                      )
                    ) : (
                      <p className={styles.ev__empty}>Select providers first to see their available models.</p>
                    ))}

                  {/* STEP 4 — TEST SUITE */}
                  {step === 4 && (
                    <>
                      {/* Upload isn't offered for Agent benchmarks with no
                          framework selected (POST /agent-benchmark/run only
                          accepts existing datasets), or for RAG (fixed
                          retrieval corpora, no upload-your-own flow) — so
                          there's nothing to switch between and the tab bar
                          itself is hidden, not just the Upload button. */}
                      {!hideUploadTab && (
                        <div className={styles.ev__tabs}>
                          <button
                            type="button"
                            className={`${styles.ev__tab} ${datasetTab === 'browse' ? styles['ev__tab--on'] : ''}`}
                            onClick={() => setDatasetTab('browse')}
                          >
                            <LayoutGrid size={14} /> Browse
                          </button>
                          <button
                            type="button"
                            className={`${styles.ev__tab} ${datasetTab === 'upload' ? styles['ev__tab--on'] : ''}`}
                            onClick={openUploadPanel}
                          >
                            <Upload size={14} /> Upload
                          </button>
                        </div>
                      )}

                      {isRag && (
                        <div className={`${styles.ev__field} ${styles['ev__field--topk']}`}>
                          <label className={styles.ev__label}>
                            Top K <span className="opt">how many retrieved documents to consider per query</span>
                          </label>
                          <input
                            type="number"
                            min={1}
                            max={50}
                            className={`${styles.ev__input} ${styles['ev__input--topk']}`}
                            value={draft.topK}
                            onChange={(e) => {
                              const raw = e.target.value === '' ? 5 : Number(e.target.value);
                              const val = Number.isNaN(raw) ? 5 : Math.min(50, Math.max(1, Math.round(raw)));
                              dispatch(setDraft({ topK: val }));
                            }}
                          />
                        </div>
                      )}

                      {datasetTab === 'browse' && (
                        <div className={styles.ev__suite}>
                          <div className={styles['ev__suite-scroll']}>
                            {listHead({
                              title: 'Test suites',
                              meta: (
                                <>
                                  {filteredDatasets.length} shown ·{' '}
                                  <b>{isHarness ? draft.harnessSuiteIds.length : draft.dataset ? 1 : 0}</b> selected
                                </>
                              ),
                              count: filteredDatasets.length,
                              search: { placeholder: 'Search test suites', value: datasetSearch, onChange: setDatasetSearch },
                              refresh: { onClick: refreshDatasets, busy: datasetsRefreshing, title: 'Refresh test suites' },
                              sticky: true,
                              // Harness mode filters by use case; every other mode keeps
                              // the All / Custom / Deepeval chips.
                              filters: isHarness ? (
                                <>
                                {['all', ...harnessUseCases].map((u) => {
                                  const on = useCaseFilter === u;
                                  const count =
                                    u === 'all' ? harnessSuites.length : harnessSuites.filter((s) => s.use_case === u).length;
                                  return (
                                    <button
                                      key={u}
                                      type="button"
                                      className={`${styles['ev__filter-chip']} ${on ? styles['ev__filter-chip--on'] : ''}`}
                                      onClick={() => setUseCaseFilter(u)}
                                    >
                                      {u === 'all' ? 'All' : u}
                                      <span className={styles['ev__filter-chip-count']}>{count}</span>
                                    </button>
                                  );
                                })}
                                </>
                              ) : (
                                <>
                              {DATASET_TYPE_FILTERS.map((f) => {
                                const on = datasetTypeFilter === f.value;
                                const count = datasetTypeCounts[f.value];
                                return (
                                  <button
                                    key={f.value}
                                    type="button"
                                    className={`${styles['ev__filter-chip']} ${on ? styles['ev__filter-chip--on'] : ''}`}
                                    onClick={() => setDatasetTypeFilter(f.value)}
                                  >
                                    {f.label}
                                    <span className={styles['ev__filter-chip-count']}>{count}</span>
                                  </button>
                                );
                              })}
                                </>
                              ),
                            })}

                            {(() => {
                              // Both the busy flag and the error message are
                              // tracked locally (set from the outcome of our
                              // own dispatch calls) rather than read from the
                              // datasets slice's own status/error fields —
                              // see datasetsErrorLocal above for why.
                              if (datasetsRefreshing) {
                                return (
                                  <div className={styles.ev__dgrid} aria-busy="true" aria-label="Loading test suites">
                                    {Array.from({ length: SKELETON_CARD_COUNT }).map((_, i) => (
                                      <div key={i} className={styles['ev__skel-dcard']}>
                                        <span className={styles['ev__skel-dcard-top']}>
                                          <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--icon-sm']}`} />
                                          <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--line']}`} style={{ width: '55%' }} />
                                        </span>
                                        <span className={styles['ev__skel-caps']}>
                                          <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--tag']}`} />
                                          <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--tag']}`} />
                                          <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--tag']}`} />
                                        </span>
                                      </div>
                                    ))}
                                  </div>
                                );
                              }

                              if (datasetsErrorLocal) {
                                return <p className={styles.ev__error}>{datasetsErrorLocal}</p>;
                              }

                              return (
                                <div className={styles.ev__slist}>
                                  {filteredDatasets.map((d) => {
                                    const on = isSuiteOn(d.id);
                                    const isCustom = d?.dataset_type === 'custom';
                                    const isDeepeval = d?.dataset_type === 'deepeval';
                                    const hasNoQuestions = (d.question_count ?? 0) <= 0;
                                    const subgroups = d.dataset_categories || [];
                                    // Preview: hidden for harness suites while the catalog is
                                    // mocked, and for agent_benchmark except the three
                                    // previewable datasets (same rules as before).
                                    const canPreview =
                                      !isHarness &&
                                      (!isAgentBenchmarkNoFramework || AGENT_BENCHMARK_PREVIEWABLE_DATASETS.has(d.name));
                                    return (
                                      <div
                                        key={d.id}
                                        role="button"
                                        tabIndex={0}
                                        className={`${styles.ev__srow} ${on ? styles['ev__srow--on'] : ''} ${
                                          hasNoQuestions ? styles['ev__srow--empty'] : ''
                                        }`}
                                        onClick={() => pickSuite(d.id)}
                                        onKeyDown={(e) => {
                                          if (e.key === 'Enter' || e.key === ' ') {
                                            e.preventDefault();
                                            pickSuite(d.id);
                                          }
                                        }}
                                        aria-pressed={on}
                                      >
                                        {/* Header: selection control + name */}
                                        <div className={styles['ev__srow-head']}>
                                          {isHarness ? (
                                            <span className={`${styles.ev__check} ${on ? styles['ev__check--on'] : ''}`}>
                                              {on && <Check size={11} strokeWidth={3} />}
                                            </span>
                                          ) : (
                                            <span className={`${styles.ev__radio} ${on ? styles['ev__radio--on'] : ''}`} />
                                          )}
                                          <span className={styles['ev__srow-icon']}>
                                            <Database size={15} />
                                          </span>
                                          <span className={styles['ev__srow-name']} title={d.name || 'Untitled dataset'}>
                                            {d.name || 'Untitled dataset'}
                                          </span>
                                        </div>

                                        {/* Tags: use case / category, source, images */}
                                        <div className={styles['ev__srow-tags']}>
                                          {isHarness && d.use_case && (
                                            <span className={`${styles.ev__pilltag} ${on ? '' : styles['ev__pilltag--muted']}`}>
                                              {d.use_case}
                                            </span>
                                          )}
                                          {/* category tag, unless it would just repeat the Custom / Deepeval tag */}
                                          {!isHarness && d.category && d.category.toLowerCase() !== String(d.dataset_type ?? '').toLowerCase() && (
                                            <span className={`${styles.ev__pilltag} ${styles['ev__pilltag--muted']}`}>
                                              {d.category}
                                            </span>
                                          )}
                                          {isCustom && <span className={styles.ev__pilltag}>Custom</span>}
                                          {isDeepeval && (
                                            <span className={`${styles.ev__pilltag} ${styles['ev__pilltag--muted']}`}>Deepeval</span>
                                          )}
                                          {d.has_images && (
                                            <span
                                              className={`${styles.ev__pilltag} ${styles['ev__pilltag--muted']}`}
                                              title="Questions include images, so every selected model must accept image input"
                                            >
                                              <ImageIcon size={11} /> Images
                                            </span>
                                          )}
                                        </div>

                                        <p
                                          className={`${styles['ev__srow-desc']} ${
                                            d.description ? '' : styles['ev__srow-desc--none']
                                          }`}
                                          title={d.description || undefined}
                                        >
                                          {d.description || 'No description provided.'}
                                        </p>
                                        {d.scoring && (
                                          <p className={styles['ev__srow-scoring']}>
                                            <b>Scored by</b> {d.scoring}
                                          </p>
                                        )}

                                        {/* Subgroups this suite is split into (first few) */}
                                        {subgroups.length > 0 && (
                                          <div className={styles['ev__srow-subs']}>
                                            <span className={styles['ev__srow-k']}>Subgroups</span>
                                            <div className={styles['ev__srow-chips']}>
                                              {subgroups.slice(0, 3).map((g) => (
                                                <span key={g} className={styles.ev__cap}>
                                                  {g}
                                                </span>
                                              ))}
                                              {subgroups.length > 3 && (
                                                <span className={styles.ev__cap} title={subgroups.slice(3).join(', ')}>
                                                  +{subgroups.length - 3} more
                                                </span>
                                              )}
                                            </div>
                                          </div>
                                        )}

                                        {/* Footer: numbers on the left, Preview on the right */}
                                        <div className={styles['ev__srow-side']}>
                                          <div className={styles.ev__stat}>
                                            <span className={styles['ev__stat-k']}>Questions</span>
                                            <span
                                              className={`${styles['ev__stat-v']} ${
                                                hasNoQuestions ? styles['ev__srow-count--empty'] : ''
                                              }`}
                                              title={
                                                hasNoQuestions
                                                  ? "This test suite has no questions yet, so it can't be used."
                                                  : undefined
                                              }
                                            >
                                              {(d.question_count ?? 0).toLocaleString()}
                                            </span>
                                          </div>
                                          <div className={styles.ev__stat}>
                                            <span className={styles['ev__stat-k']}>Subgroups</span>
                                            <span className={styles['ev__stat-v']}>{subgroups.length || '—'}</span>
                                          </div>
                                          {canPreview && (
                                            <button
                                              type="button"
                                              className={styles['ev__srow-preview']}
                                              onClick={(e) => {
                                                e.stopPropagation();
                                                openPreview(d.id, d.name, d.question_count ?? 0);
                                              }}
                                            >
                                              <FileText size={13} /> Preview
                                            </button>
                                          )}
                                        </div>
                                      </div>
                                    );
                                  })}
                                  {datasets.length === 0 && (
                                    <p className={styles.ev__empty}>No test suites available for this type yet.</p>
                                  )}
                                  {datasets.length > 0 && filteredDatasets.length === 0 && (
                                    <p className={styles.ev__empty}>
                                      No test suites match {datasetSearch ? `"${datasetSearch}"` : 'this filter'}.
                                    </p>
                                  )}
                                </div>
                              );
                            })()}
                          </div>

                          {/* Subgroups apply to a single dataset — not shown for the
                              multi-suite harness. */}
                          {isHarness && coveragePanel}
                          {!isHarness && (
                          <aside className={styles.ev__rail}>
                            <div className={styles['ev__rail-head']}>
                              <div className={styles['ev__rail-head-row']}>
                                <p className={styles['ev__rail-title']}>
                                  <Layers size={13} /> Subgroups
                                </p>
                                {suite && (suite as any).dataset_categories?.length > 0 && (
                                  <div className={styles['ev__rail-actions']}>
                                    <button type="button" className={styles.ev__link} onClick={selectAllSubgroups}>
                                      Select all
                                    </button>
                                    <button type="button" className={styles.ev__link} onClick={clearAllSubgroups}>
                                      Unselect all
                                    </button>
                                  </div>
                                )}
                              </div>
                              <p className={styles['ev__rail-sub']}>
                                {suite
                                  ? `All of "${suite.name}"'s categories are selected by default — narrow as needed.`
                                  : 'Select a suite to see its subgroups.'}
                              </p>
                            </div>
                            <div className={styles['ev__rail-scroll']}>
                              {!suite && <p className={styles['ev__rail-empty']}>No suite selected yet.</p>}
                              {suite && (suite as any).dataset_categories?.length === 0 && (
                                <p className={styles['ev__rail-empty']}>This suite has no subgroups.</p>
                              )}
                              {suite &&
                                ((suite as any).dataset_categories ?? []).map((cat: string) => {
                                  const on = draft.subgroup.includes(cat);
                                  return (
                                    <button
                                      key={cat}
                                      type="button"
                                      className={`${styles['ev__check-row']} ${on ? styles['ev__check-row--on'] : ''}`}
                                      onClick={() => dispatch(setDraft({ subgroup: toggle(draft.subgroup, cat) }))}
                                    >
                                      <span className={`${styles.ev__check} ${on ? styles['ev__check--on'] : ''}`}>
                                        {on && <Check size={11} strokeWidth={3} />}
                                      </span>
                                      <span className={styles['ev__check-label']}>{cat}</span>
                                    </button>
                                  );
                                })}
                            </div>
                          </aside>
                          )}
                        </div>
                      )}

                      {isHarness && selectedHarnessSuites.some((s) => (s.question_count ?? 0) <= 0) && (
                        <div className={styles['ev__dataset-empty-warning']}>
                          <Info size={14} />
                          <span>
                            One or more selected suites have no questions, so they can't be used for a run. Deselect
                            them to continue.
                          </span>
                        </div>
                      )}

                      {suite && (suite.question_count ?? 0) <= 0 && (
                        <div className={styles['ev__dataset-empty-warning']}>
                          <Info size={14} />
                          <span>
                            <strong>{suite.name || 'This test suite'}</strong> has no questions, so it can't be used for a
                            run. Pick a different test suite to continue.
                          </span>
                        </div>
                      )}

                      {datasetTab === 'upload' && !hideUploadTab && (
                        <div className={styles.ev__upload}>
                          <div className={styles.ev__field}>
                            <label className={styles.ev__label}>Name</label>
                            <input
                              className={styles.ev__input}
                              placeholder="e.g. Internal QA set v1"
                              value={uploadName}
                              onChange={(e) => setUploadName(e.target.value)}
                              disabled={datasetUploading}
                            />
                          </div>
                          <div className={styles.ev__field}>
                            <label className={styles.ev__label}>
                              Description <span className="opt">optional</span>
                            </label>
                            <input
                              className={styles.ev__input}
                              placeholder="What does this dataset cover?"
                              value={uploadDescription}
                              onChange={(e) => setUploadDescription(e.target.value)}
                              disabled={datasetUploading}
                            />
                          </div>
                          <div className={styles.ev__field}>
                            <label className={styles.ev__label}>Evaluation type</label>
                            <input className={styles.ev__input} value={draft.type || '—'} disabled readOnly />
                          </div>
                          <div className={styles.ev__field}>
                            <label className={styles.ev__label}>File</label>
                            <label className={`${styles.ev__drop} ${uploadFile ? styles['ev__drop--has'] : ''}`}>
                              <input
                                type="file"
                                accept={SUPPORTED_UPLOAD_EXTENSIONS.map((e) => `.${e}`).join(',')}
                                onChange={(e) => handleUploadFileChange(e.target.files?.[0] ?? null)}
                                disabled={datasetUploading}
                                hidden
                              />
                              {uploadFile ? (
                                <span className={styles['ev__drop-file']}>
                                  <FileText size={15} /> {uploadFile.name}
                                </span>
                              ) : (
                                <>
                                  <FileText size={15} /> Choose a .json, .jsonl, .arrow or .parquet file
                                </>
                              )}
                            </label>
                            {uploadFileError && <p className={styles.ev__error}>{uploadFileError}</p>}
                          </div>
                          {datasetUploadError && <p className={styles.ev__error}>{datasetUploadError}</p>}
                          <div className={styles['ev__upload-actions']}>
                            <button
                              type="button"
                              className={`${styles.ev__btn} ${styles['ev__btn--ghost']}`}
                              onClick={() => setDatasetTab('browse')}
                              disabled={datasetUploading}
                            >
                              Cancel
                            </button>
                            <button
                              type="button"
                              className={`${styles.ev__btn} ${styles['ev__btn--primary']}`}
                              onClick={submitUpload}
                              disabled={!canUpload}
                            >
                              {datasetUploading ? (
                                <>
                                  <Loader2 size={15} className={styles.ev__spin} /> Uploading…
                                </>
                              ) : (
                                <>
                                  <Upload size={15} /> Upload &amp; use
                                </>
                              )}
                            </button>
                          </div>
                        </div>
                      )}
                    </>
                  )}

                  {/* STEP 5 — SCORING */}
                  {step === 5 && (
                    <div className={styles.ev__split}>
                    <div className={`${styles.ev__form} ${styles['ev__form--scroll']}`}>
                      {/* How much to run */}
                      <section className={styles.ev__frow}>
                        <div className={styles['ev__frow-side']}>
                          <h3>How much to run</h3>
                          <p>
                            {isHarness
                              ? 'Questions taken from each suite, for every agent and model.'
                              : 'Questions taken from the suite, for every model.'}
                          </p>
                        </div>
                        <div className={styles['ev__frow-main']}>
                          <div className={styles.ev__inline}>
                            <div className={styles.ev__segctl}>
                              {[
                                { v: 'custom' as const, l: 'A sample' },
                                { v: 'full' as const, l: 'Every question' },
                              ].map((o) => (
                                <button
                                  key={o.v}
                                  type="button"
                                  className={`${styles.ev__segbtn} ${
                                    draft.runSamplesMode === o.v ? styles['ev__segbtn--on'] : ''
                                  }`}
                                  onClick={() => dispatch(setDraft({ runSamplesMode: o.v }))}
                                >
                                  {o.l}
                                </button>
                              ))}
                            </div>
                            {draft.runSamplesMode === 'custom' && (
                              <>
                                <input
                                  type="number"
                                  min={0}
                                  max={maxRunSamples}
                                  className={styles.ev__num}
                                  value={displayNumInput(draft.runSamples)}
                                  onChange={(e) => {
                                    // An emptied field is held as NaN so it shows blank instead
                                    // of snapping to 0; onBlur below (and launch) resolve it.
                                    const raw = parseNumInput(e.target.value);
                                    const val = Number.isNaN(raw) ? NaN : Math.min(maxRunSamples, Math.max(0, raw));
                                    dispatch(setDraft({ runSamples: val }));
                                  }}
                                  onBlur={() => {
                                    if (Number.isNaN(draft.runSamples)) dispatch(setDraft({ runSamples: 0 }));
                                  }}
                                />
                                <span className={styles['ev__inline-note']}>
                                  questions per {isHarness ? 'suite' : 'model'}
                                </span>
                              </>
                            )}
                          </div>
                          {isHarness ? (
                            <p className={styles.ev__formula}>
                              {sampleN !== null &&
                              selectedHarnessSuites.every((d) => (d.question_count ?? 0) >= sampleN)
                                ? `${sampleN} questions × ${selectedHarnessSuites.length} suites`
                                : `${harnessQuestionsPerPass.toLocaleString()} questions across ${
                                    selectedHarnessSuites.length
                                  } suites`}{' '}
                              × {draft.models.length} models × {draft.harnessAgents.length} agents ={' '}
                              <b>{harnessTaskCount.toLocaleString()} tasks</b>
                            </p>
                          ) : (
                            <p className={styles['ev__inline-note']}>
                              {draft.runSamplesMode === 'full'
                                ? 'Every question in the suite will be used.'
                                : `Up to ${maxRunSamples.toLocaleString()} questions available in this suite.`}
                            </p>
                          )}
                        </div>
                      </section>

                      {/* What to score */}
                      <section className={styles.ev__frow}>
                        <div className={styles['ev__frow-side']}>
                          <h3>What to score</h3>
                          {hideMetricsStep ? (
                            <p>Fixed for this run.</p>
                          ) : (
                            <>
                              <p>
                                {draft.metrics.length} of {metricsCatalog.length + customMetricsForType.length} {scorerWord}{' '}
                                selected.
                              </p>
                              <div className={styles['ev__metrics-actions']}>
                                <button type="button" className={styles.ev__link} onClick={selectAllBuiltinMetrics}>
                                  Select all
                                </button>
                                <button type="button" className={styles.ev__link} onClick={clearBuiltinMetrics}>
                                  Clear
                                </button>
                              </div>
                              {(metricsCatalog.length > 0 || customMetricsForType.length > 0) &&
                                draft.metrics.length === 0 && (
                                  <p className={styles['ev__frow-req']}>Required: pick at least one</p>
                                )}
                            </>
                          )}
                        </div>
                        <div className={styles['ev__frow-main']}>
                          {hideMetricsStep ? (
                            <p className={styles['ev__inline-note']}>
                              {isAgentBenchmarkNoFramework
                                ? 'Metrics are not configurable without a selected framework. A judge model is still required.'
                                : 'Metrics are not configurable for standard (non-custom) benchmarks. A judge model is still required.'}
                            </p>
                          ) : (
                            <>
                              <div className={styles['ev__step-toolbar']} style={{ marginBottom: 0 }}>
                                <div className={styles['ev__toolbar-search']}>
                                  <Search size={14} />
                                  <input
                                    placeholder={`Search ${scorerWord}`}
                                    value={metricSearch}
                                    onChange={(e) => setMetricSearch(e.target.value)}
                                  />
                                </div>
                                <button
                                  type="button"
                                  className={styles['ev__toolbar-refresh']}
                                  onClick={refreshMetrics}
                                  disabled={metricsRefreshing}
                                  title={`Refresh ${scorerWord}`}
                                >
                                  <RefreshCw size={14} className={metricsRefreshing ? styles.ev__spin : ''} />
                                </button>
                              </div>

                              {metricsRefreshing ? (
                                <div className={styles.ev__chips} aria-busy="true" aria-label="Loading metrics">
                                  {Array.from({ length: SKELETON_CHIP_COUNT }).map((_, i) => (
                                    <span
                                      key={i}
                                      className={`${styles['ev__skel-block']} ${styles['ev__skel-block--chip']}`}
                                      style={{ width: 64 + ((i * 37) % 90) }}
                                    />
                                  ))}
                                </div>
                              ) : (
                                <div className={styles.ev__chips}>
                                  {filteredMetrics.map((name: string) => {
                                    const on = draft.metrics.includes(name);
                                    return (
                                      <button
                                        key={name}
                                        type="button"
                                        className={`${styles.ev__chip} ${on ? styles['ev__chip--on'] : ''}`}
                                        aria-pressed={on}
                                        onClick={() => dispatch(setDraft({ metrics: toggle(draft.metrics, name) }))}
                                      >
                                        {name}
                                      </button>
                                    );
                                  })}
                                  {metricsCatalog.length === 0 && (
                                    <p className={styles['ev__inline-note']}>No {scorerWord} available for this type.</p>
                                  )}
                                  {metricsCatalog.length > 0 && filteredMetrics.length === 0 && (
                                    <p className={styles['ev__inline-note']}>Nothing matches "{metricSearch}".</p>
                                  )}
                                </div>
                              )}

                              {/* Custom metrics: stored by id in draft.metrics, alongside
                                  the built-in names. Shown whenever the catalog has any. */}
                              {customMetricsCatalog.length > 0 && (
                                <>
                                  <p className={styles.ev__sublabel}>
                                    Custom
                                    <button type="button" className={styles.ev__link} onClick={selectAllCustomMetrics}>
                                      Select all
                                    </button>
                                    <button type="button" className={styles.ev__link} onClick={clearCustomMetrics}>
                                      Clear
                                    </button>
                                  </p>
                                  <div className={styles.ev__chips}>
                                    {filteredCustomMetrics.map((c) => {
                                      const on = draft.metrics.includes(c.id);
                                      return (
                                        <button
                                          key={c.id}
                                          type="button"
                                          className={`${styles.ev__chip} ${styles['ev__chip--custom']} ${
                                            on ? styles['ev__chip--on'] : ''
                                          }`}
                                          aria-pressed={on}
                                          onClick={() => dispatch(setDraft({ metrics: toggle(draft.metrics, c.id) }))}
                                          title={c.description || c.name}
                                        >
                                          {c.name}
                                          {c.required_judge && <Gavel size={11} className={styles['ev__chip-judge-icon']} />}
                                        </button>
                                      );
                                    })}
                                    {customMetricsForType.length === 0 && (
                                      <p className={styles['ev__inline-note']}>No custom metrics for this type.</p>
                                    )}
                                    {customMetricsForType.length > 0 && filteredCustomMetrics.length === 0 && (
                                      <p className={styles['ev__inline-note']}>Nothing matches "{metricSearch}".</p>
                                    )}
                                  </div>
                                </>
                              )}
                            </>
                          )}
                        </div>
                      </section>

                      {/* Limits per task — harness only */}
                      {isHarness && (
                        <section className={styles.ev__frow}>
                          <div className={styles['ev__frow-side']}>
                            <h3>Limits per task</h3>
                            <p>When the harness stops an agent, and what it keeps afterwards.</p>
                          </div>
                          <div className={styles['ev__frow-main']}>
                            <div className={styles.ev__inline} style={{ alignItems: 'flex-end' }}>
                              <div className={styles.ev__numfield}>
                                <label>Max steps</label>
                                <input
                                  type="number"
                                  className={styles.ev__num}
                                  min={MIN_HARNESS_STEPS}
                                  max={MAX_HARNESS_STEPS}
                                  value={displayNumInput(draft.harnessSettings.max_steps)}
                                  onChange={(e) =>
                                    dispatch(
                                      setDraft({
                                        harnessSettings: { ...draft.harnessSettings, max_steps: parseNumInput(e.target.value) },
                                      })
                                    )
                                  }
                                  onBlur={() =>
                                    dispatch(
                                      setDraft({
                                        harnessSettings: {
                                          ...draft.harnessSettings,
                                          max_steps: clampHarnessSteps(draft.harnessSettings.max_steps),
                                        },
                                      })
                                    )
                                  }
                                />
                              </div>
                              <div className={styles.ev__numfield}>
                                <label>Timeout (seconds)</label>
                                <input
                                  type="number"
                                  className={styles.ev__num}
                                  min={MIN_TIMEOUT}
                                  max={MAX_TIMEOUT}
                                  value={displayNumInput(draft.harnessSettings.task_timeout)}
                                  onChange={(e) =>
                                    dispatch(
                                      setDraft({
                                        harnessSettings: {
                                          ...draft.harnessSettings,
                                          task_timeout: parseNumInput(e.target.value),
                                        },
                                      })
                                    )
                                  }
                                  onBlur={() =>
                                    dispatch(
                                      setDraft({
                                        harnessSettings: {
                                          ...draft.harnessSettings,
                                          task_timeout: clampRetryField('timeout', draft.harnessSettings.task_timeout),
                                        },
                                      })
                                    )
                                  }
                                />
                              </div>
                              <button
                                type="button"
                                role="checkbox"
                                aria-checked={draft.harnessSettings.save_traces}
                                className={styles.ev__checkline}
                                onClick={() =>
                                  dispatch(
                                    setDraft({
                                      harnessSettings: {
                                        ...draft.harnessSettings,
                                        save_traces: !draft.harnessSettings.save_traces,
                                      },
                                    })
                                  )
                                }
                              >
                                <span
                                  className={`${styles.ev__check} ${
                                    draft.harnessSettings.save_traces ? styles['ev__check--on'] : ''
                                  }`}
                                >
                                  {draft.harnessSettings.save_traces && <Check size={11} strokeWidth={3} />}
                                </span>
                                Save traces for replay
                              </button>
                            </div>
                            <p className={styles['ev__inline-note']}>
                              Steps {MIN_HARNESS_STEPS} to {MAX_HARNESS_STEPS}. Timeout {MIN_TIMEOUT} to {MAX_TIMEOUT} seconds.
                            </p>
                          </div>
                        </section>
                      )}

                      {/* Retest on wrong — Model and RAG always; Agent only with a framework */}
                      {retestShown && (
                        <section className={styles.ev__frow}>
                          <div className={styles['ev__frow-side']}>
                            <h3>Retest on wrong</h3>
                            <p>Re-grade a wrong answer before marking it failed.</p>
                            {draft.retestOnWrong && !draft.retestVerifyMetric && (
                              <p className={styles['ev__frow-req']}>Required: pick a verify metric</p>
                            )}
                          </div>
                          <div className={styles['ev__frow-main']}>
                            <div className={styles.ev__inline}>
                              <div className={styles.ev__segctl}>
                                <button
                                  type="button"
                                  className={`${styles.ev__segbtn} ${!draft.retestOnWrong ? styles['ev__segbtn--on'] : ''}`}
                                  onClick={() =>
                                    dispatch(
                                      setDraft({
                                        retestOnWrong: false,
                                        retestMaxRounds: DEFAULT_RETEST_ROUNDS,
                                        retestVerifyMetric: null,
                                      })
                                    )
                                  }
                                >
                                  Off
                                </button>
                                <button
                                  type="button"
                                  className={`${styles.ev__segbtn} ${draft.retestOnWrong ? styles['ev__segbtn--on'] : ''}`}
                                  onClick={() => dispatch(setDraft({ retestOnWrong: true }))}
                                >
                                  On
                                </button>
                              </div>
                              {draft.retestOnWrong && (
                                <>
                                  <input
                                    type="number"
                                    className={styles.ev__num}
                                    min={MIN_RETEST_ROUNDS}
                                    max={MAX_RETEST_ROUNDS}
                                    value={displayNumInput(draft.retestMaxRounds)}
                                    onChange={(e) => updateRetestMaxRounds(parseNumInput(e.target.value))}
                                    onBlur={clampRetestMaxRoundsOnBlur}
                                  />
                                  <span className={styles['ev__inline-note']}>
                                    rounds at most ({MIN_RETEST_ROUNDS} to {MAX_RETEST_ROUNDS})
                                  </span>
                                </>
                              )}
                            </div>
                            {draft.retestOnWrong && (
                              <>
                                <p className={styles.ev__sublabel}>Verify each retest with one metric</p>
                                <div className={styles.ev__chips}>
                                  {metricsCatalog.map((name: string) => {
                                    const on = draft.retestVerifyMetric === name;
                                    return (
                                      <button
                                        key={name}
                                        type="button"
                                        className={`${styles.ev__chip} ${on ? styles['ev__chip--on'] : ''}`}
                                        aria-pressed={on}
                                        onClick={() => dispatch(setDraft({ retestVerifyMetric: on ? null : name }))}
                                      >
                                        {name}
                                      </button>
                                    );
                                  })}
                                  {metricsCatalog.length === 0 && (
                                    <p className={styles['ev__inline-note']}>No metrics available to verify against.</p>
                                  )}
                                </div>
                              </>
                            )}
                          </div>
                        </section>
                      )}

                      {/* Instruction — optional; harness suites carry their own */}
                      {!isHarness && (
                        <section className={styles.ev__frow}>
                          <div className={styles['ev__frow-side']}>
                            <h3>Instruction</h3>
                            <p>Optional. Extra context for how the models should approach this run.</p>
                            <div className={styles['ev__metrics-actions']}>
                              <button
                                type="button"
                                className={styles.ev__link}
                                onClick={generateInstruction}
                                disabled={instructionGenerating || !draft.dataset || draft.models.length === 0}
                                title="Generate a default instruction from a sample of this dataset"
                              >
                                {instructionGenerating ? 'Generating…' : 'Generate a draft'}
                              </button>
                            </div>
                          </div>
                          <div className={styles['ev__frow-main']}>
                            <textarea
                              className={styles['ev__instruction-textarea']}
                              placeholder="Describe how the models should approach this evaluation, or generate a draft to edit."
                              value={draft.instruction}
                              onChange={(e) => dispatch(setDraft({ instruction: e.target.value }))}
                              rows={4}
                            />
                            {instructionError && <p className={styles.ev__error}>{instructionError}</p>}
                          </div>
                        </section>
                      )}
                    </div>

                    {/* Who judges — mandatory for every type. Its own side panel,
                        same pattern as the Models step's "In this run". */}
                    <aside className={`${styles.ev__panel} ${styles['ev__panel--pinned']}`}>
                      {/* Pinned: heading, guidance, required note and search.
                          Only the list of candidate models below scrolls. */}
                      <div className={styles['ev__panel-fixed']}>
                        <div className={styles['ev__panel-headrow']}>
                          <p className={styles['ev__panel-title']}>Who judges</p>
                          {judgeModel ? (
                            <span className={styles['ev__panel-chip']}>
                              <Gavel size={11} /> {judgeModel.name || 'Unnamed model'}
                            </span>
                          ) : (
                            <span className={`${styles['ev__panel-chip']} ${styles['ev__panel-chip--req']}`}>Required</span>
                          )}
                        </div>
                        <p className={styles['ev__panel-note']}>
                          One model grades open-ended answers. Pick one that is not under test if you can.
                        </p>
                        <div className={styles.ev__sbar}>
                          <span className={styles['ev__sbar-icon']}>
                            <Search size={13} strokeWidth={2.6} />
                          </span>
                          <input
                            type="text"
                            placeholder="Search judge models"
                            aria-label="Search judge models"
                            value={judgeSearch}
                            onChange={(e) => setJudgeSearch(e.target.value)}
                          />
                          {judgeSearch && (
                            <>
                              <span
                                className={`${styles['ev__sbar-count']} ${
                                  filteredJudgeModels.length === 0 ? styles['ev__sbar-count--none'] : ''
                                }`}
                              >
                                {filteredJudgeModels.length === 0
                                  ? 'No matches'
                                  : `${filteredJudgeModels.length} result${filteredJudgeModels.length === 1 ? '' : 's'}`}
                              </span>
                              <button
                                type="button"
                                className={styles['ev__sbar-clear']}
                                onClick={() => setJudgeSearch('')}
                                aria-label="Clear search"
                              >
                                <X size={12} strokeWidth={2.5} />
                              </button>
                            </>
                          )}
                        </div>
                      </div>

                      <div className={styles['ev__panel-scroll']}>
                        {judgeCandidateModels.length === 0 ? (
                          <p className={styles['ev__panel-note']}>No healthy models available yet.</p>
                        ) : filteredJudgeModels.length === 0 ? (
                          <p className={styles['ev__panel-note']}>No models match "{judgeSearch}".</p>
                        ) : (
                          filteredJudgeModels.map((m) => {
                            const on = draft.judgeModelId === m.id;
                            const providerName = providers.find((p) => p?.id === m.provider_id)?.name ?? m.provider_id;
                            const underTest = draft.models.includes(m.id);
                            return (
                              <button
                                key={m.id}
                                type="button"
                                className={`${styles.ev__jcard} ${on ? styles['ev__jcard--on'] : ''}`}
                                aria-pressed={on}
                                onClick={() => dispatch(setDraft({ judgeModelId: on ? null : m.id }))}
                              >
                                <span className={styles['ev__jcard-head']}>
                                  <span className={`${styles.ev__radio} ${on ? styles['ev__radio--on'] : ''}`} />
                                  <span className={styles['ev__srow-icon']}>
                                    <Gavel size={14} />
                                  </span>
                                  <span className={styles['ev__jcard-id']}>
                                    <span className={styles['ev__jcard-name']} title={m.name || 'Unnamed model'}>
                                      {m.name || 'Unnamed model'}
                                    </span>
                                    <span className={styles['ev__jcard-provider']}>{providerName}</span>
                                  </span>
                                  {underTest && (
                                    <span
                                      className={styles['ev__jcard-flag']}
                                      title="This model is also being evaluated in this run"
                                    >
                                      Under test
                                    </span>
                                  )}
                                </span>
                                <span className={styles['ev__jcard-stats']}>
                                  <span className={styles.ev__stat}>
                                    <span className={styles['ev__stat-k']}>Context</span>
                                    <span className={styles['ev__stat-v']}>{formatContextWindow(m.context_window)}</span>
                                  </span>
                                  <span className={styles.ev__stat}>
                                    <span className={styles['ev__stat-k']}>Input /1M</span>
                                    <span className={styles['ev__stat-v']}>{formatPrice(m.input_price)}</span>
                                  </span>
                                  <span className={styles.ev__stat}>
                                    <span className={styles['ev__stat-k']}>Output /1M</span>
                                    <span className={styles['ev__stat-v']}>{formatPrice(m.output_price)}</span>
                                  </span>
                                </span>
                              </button>
                            );
                          })
                        )}
                      </div>
                    </aside>
                    </div>
                  )}

                  {/* STEP 6 — REVIEW */}
                  {step === 6 && (
                    <>
                      <div className={styles.ev__banner}>
                        <div style={{ minWidth: 0 }}>
                          <p className={styles['ev__banner-title']}>{draft.name || 'Untitled run'}</p>
                          <p className={styles['ev__banner-sub']}>{reviewSummaryLine}</p>
                        </div>
                        <div className={styles['ev__banner-stats']}>
                          {reviewStats.map((st) => (
                            <div key={st.k}>
                              <span>{st.k}</span>
                              <b>{st.v}</b>
                            </div>
                          ))}
                        </div>
                      </div>

                      {/* Run matrix — every agent runs every selected model on every
                          selected suite; each cell is that agent's task count. */}
                      {isHarness && (
                        <>
                          <div className={styles.ev__sechead}>
                            <h3>Run matrix</h3>
                            <span>
                              Each cell runs on{' '}
                              {selectedModels.map((m) => m.name || 'Unnamed model').join(' and ') || 'no models yet'}
                            </span>
                          </div>
                          <div className={`${styles['ev__table-wrap']} ${styles['ev__table-wrap--static']}`}>
                            <table className={`${styles.ev__table} ${styles['ev__table--static']}`}>
                              <thead>
                                <tr>
                                  <th>Suite</th>
                                  <th>Use case</th>
                                  {harnessAgentTitles.map((t) => (
                                    <th key={t} className={styles['ev__table-num']}>
                                      {t}
                                    </th>
                                  ))}
                                </tr>
                              </thead>
                              <tbody>
                                {selectedHarnessSuites.map((d) => (
                                  <tr key={d.id}>
                                    <td>
                                      <b>{d.name}</b>
                                    </td>
                                    <td>{d.use_case || '—'}</td>
                                    {harnessAgentTitles.map((t) => (
                                      <td key={t} className={styles['ev__table-num']}>
                                        {(suiteQuestions(d.question_count ?? 0) * draft.models.length).toLocaleString()} tasks
                                      </td>
                                    ))}
                                  </tr>
                                ))}
                              </tbody>
                            </table>
                          </div>
                          {uncoveredUseCases.length > 0 && (
                            <p className={styles.ev__secnote}>
                              {uncoveredUseCases.join(', ')} {uncoveredUseCases.length === 1 ? 'is' : 'are'} not covered by
                              this run.
                            </p>
                          )}
                        </>
                      )}

                      <div className={styles.ev__sechead}>
                        <h3>Settings</h3>
                      </div>
                      <div className={styles.ev__settings}>
                        {reviewRows.map((r) => (
                          <div key={r.k} className={styles['ev__settings-row']}>
                            <span className={styles['ev__settings-k']}>{r.k}</span>
                            <span className={styles['ev__settings-v']}>{r.v}</span>
                            <button type="button" className={styles.ev__link} onClick={() => goToStep(r.step)} disabled={launching}>
                              Edit
                            </button>
                          </div>
                        ))}
                      </div>

                      {launchError && (
                        <p className={styles.ev__error} ref={launchErrorRef}>
                          {launchError}
                        </p>
                      )}
                    </>
                  )}
                </div>
                </StepErrorBoundary>
              </div>

              {/* ---- footer nav ---- */}
              <div className={styles.ev__footer}>
                <button
                  type="button"
                  className={`${styles.ev__btn} ${styles['ev__btn--ghost']}`}
                  onClick={() => (step > 0 ? goBack() : navigate('/app/dashboard'))}
                  disabled={launching}
                >
                  {step === 0 ? 'Cancel' : 'Back'}
                </button>

                <div style={{ display: 'flex', alignItems: 'center', gap: 16 }}>
                  {step === 0 && canGo() && (
                    <span className={styles.ev__hint}>
                      <kbd>↵</kbd> Enter to continue
                    </span>
                  )}
                  {step < totalSteps - 1 ? (
                    <>
                      {blocker && <span className={styles.ev__blocker}>{blocker}</span>}
                      <button
                        type="button"
                        className={`${styles.ev__btn} ${styles['ev__btn--primary']}`}
                        onClick={goNext}
                        disabled={!canGo()}
                      >
                        Next
                      </button>
                    </>
                  ) : (
                    <>
                      <span className={styles.ev__blocker}>You can follow progress in History after launch.</span>
                      <button
                        type="button"
                        className={`${styles.ev__btn} ${styles['ev__btn--launch']}`}
                        onClick={launch}
                        disabled={launching}
                      >
                        {launching ? (
                          <>
                            <Loader2 size={16} className={styles.ev__spin} /> Launching…
                          </>
                        ) : (
                          'Launch run'
                        )}
                      </button>
                    </>
                  )}
                </div>
              </div>
            </section>
          </div>
        </div>
      </div>

      {/* Change-2: dataset preview slider — right-to-left panel showing
          GET /datasets/{id}/preview?limit={limit} for the clicked suite. */}
      {previewOpen && (
        <div className={styles['ev-preview-overlay']} onClick={closePreview}>
          <aside
            className={styles['ev-preview-panel']}
            role="dialog"
            aria-modal="true"
            aria-label={`Preview ${previewDatasetName}`}
            onClick={(e) => e.stopPropagation()}
          >
            <div className={styles['ev-preview-head']}>
              <div className={styles['ev-preview-head-main']}>
                <p className={styles['ev-preview-eyebrow']}>Test suite preview</p>
                <h3 className={styles['ev-preview-title']}>{previewDatasetName || 'Dataset'}</h3>
              </div>
              <button type="button" className={styles['ev-preview-close']} onClick={closePreview} title="Close preview">
                <X size={16} />
              </button>
            </div>

            <div className={styles['ev-preview-controls']}>
              <div className={styles['ev-preview-limit']}>
                <label className={styles['ev-preview-limit-label']}>
                  Page {previewCurrentPage} of {previewTotalPages}
                  <span className={styles['ev-preview-limit-range']}>
                    {' '}
                    ({PREVIEW_PAGE_SIZE} per page
                    {previewQuestionCount > 0 ? `, ${previewQuestionCount.toLocaleString()} total` : ''})
                  </span>
                </label>
                <div className={styles['ev-preview-limit-controls']}>
                  <button
                    type="button"
                    className={styles['ev-preview-stepper-btn']}
                    onClick={goToPreviewPrevPage}
                    disabled={previewLoading || !previewHasPrevPage}
                    aria-label="Previous page"
                    title="Previous page"
                  >
                    <ChevronLeft size={14} />
                  </button>
                  <button
                    type="button"
                    className={styles['ev-preview-stepper-btn']}
                    onClick={goToPreviewNextPage}
                    disabled={previewLoading || !previewHasNextPage}
                    aria-label="Next page"
                    title="Next page"
                  >
                    <ChevronRight size={14} />
                  </button>
                </div>
              </div>
              <button
                type="button"
                className={`${styles.ev__btn} ${styles['ev__btn--primary']}`}
                onClick={() => previewDatasetId && fetchPreviewPage(previewDatasetId, previewOffset)}
                disabled={previewLoading}
                title="Reload this page"
              >
                {previewLoading ? (
                  <>
                    <Loader2 size={15} className={styles.ev__spin} /> Loading…
                  </>
                ) : (
                  <>
                    <RefreshCw size={15} /> Reload
                  </>
                )}
              </button>
            </div>

            <div className={styles['ev-preview-body']}>
              {previewError && <p className={styles.ev__error}>{previewError}</p>}

              {previewLoading && (
                <div className={styles['ev-preview-skel-list']} aria-busy="true" aria-label="Loading preview">
                  {Array.from({ length: 5 }).map((_, i) => (
                    <div key={i} className={styles['ev-preview-skel-card']}>
                      <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--line']}`} style={{ width: '85%' }} />
                      <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--line']}`} style={{ width: '60%' }} />
                      <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--line']}`} style={{ width: '40%' }} />
                    </div>
                  ))}
                </div>
              )}

              {!previewLoading &&
                !previewError &&
                previewData &&
                (() => {
                  // Guard against `questions` being missing/non-array on the
                  // response — accessing .length/.map directly on that would
                  // throw if the API ever omits or nulls the field.
                  const previewQuestions = Array.isArray(previewData.questions) ? previewData.questions : [];

                  if (previewQuestions.length === 0) {
                    return <p className={styles.ev__empty}>This suite returned no sample questions.</p>;
                  }

                  return (
                    <div className={styles['ev-preview-list']}>
                      {previewQuestions.map((q, i) => {
                        // Two response shapes are both seen in practice —
                        // resolve whichever fields are actually present
                        // rather than assuming one fixed shape (see
                        // DatasetPreviewQuestion in types).
                        const promptText = q?.input?.prompt ?? q?.input?.question;
                        const inputSource = q?.input?.source;
                        const inputType = q?.input?.type;
                        const inputLanguage = q?.input?.language;
                        const expectedAnswer = q?.expected?.answer;
                        const expectedDocId = q?.expected?.doc_id;
                        const expectedSectionId = q?.expected?.section_id;
                        const metadataEntries =
                          q?.metadata && typeof q.metadata === 'object' ? flattenMetadata(q.metadata) : [];
                        const questionImages = Array.isArray(q?.input?.images) ? (q.input!.images as string[]) : [];

                        return (
                          <div key={q?.id ?? i} className={styles['ev-preview-q']}>
                            <div className={styles['ev-preview-q-head']}>
                              <span className={styles['ev-preview-q-index']}>Q{previewOffset + i + 1}</span>
                              {q?.category && <span className={styles['ev-preview-q-cat']}>{q.category}</span>}
                              {q?.subgroup && <span className={styles['ev-preview-q-cat']}>{q.subgroup}</span>}
                              {inputType && <span className={styles['ev-preview-q-cat']}>{String(inputType)}</span>}
                              {inputLanguage && <span className={styles['ev-preview-q-cat']}>{String(inputLanguage)}</span>}
                              {q?.question_type && <span className={styles['ev-preview-q-cat']}>{String(q.question_type)}</span>}
                            </div>

                            {promptText != null && (
                              <p className={styles['ev-preview-q-prompt']}>{String(promptText)}</p>
                            )}
                            {inputSource != null && (
                              <p className={styles['ev-preview-q-source']}>
                                <span>Source</span> {String(inputSource)}
                              </p>
                            )}

                            {questionImages.length > 0 && (
                              <div className={styles['ev-preview-q-images']}>
                                {questionImages.map((imgPath, imgIdx) => (
                                  <button
                                    key={`${imgPath}-${imgIdx}`}
                                    type="button"
                                    className={styles['ev-preview-q-image-btn']}
                                    onClick={() => openImagePreview(questionImages, imgIdx)}
                                    title={imgPath}
                                  >
                                    <ImageIcon size={13} />
                                    {imgPath.split('/').pop() || `Image ${imgIdx + 1}`}
                                  </button>
                                ))}
                              </div>
                            )}

                            {Array.isArray(q?.choices) && q.choices.length > 0 && (
                              <ul className={styles['ev-preview-q-choices']}>
                                {q.choices.map((c, ci) => (
                                  <li key={ci}>{String(c)}</li>
                                ))}
                              </ul>
                            )}

                            {expectedAnswer !== undefined && expectedAnswer !== null && (
                              <p className={styles['ev-preview-q-answer']}>
                                <span>Expected</span> {String(expectedAnswer)}
                              </p>
                            )}
                            {(expectedDocId != null || expectedSectionId != null) && (
                              <p className={styles['ev-preview-q-answer']}>
                                <span>Reference</span>{' '}
                                {[expectedDocId != null ? `doc ${expectedDocId}` : null, expectedSectionId != null ? `section ${expectedSectionId}` : null]
                                  .filter(Boolean)
                                  .join(' · ')}
                              </p>
                            )}

                            {metadataEntries.length > 0 && (
                              <div className={styles['ev-preview-q-meta']}>
                                {metadataEntries.map(([key, value]) => (
                                  <span key={key} className={styles['ev-preview-q-meta-item']}>
                                    <b>{key}</b>
                                    {String(value)}
                                  </span>
                                ))}
                              </div>
                            )}
                          </div>
                        );
                      })}
                    </div>
                  );
                })()}
            </div>
          </aside>
        </div>
      )}

      {/* Image lightbox — resolves a MinIO object key from a preview
          question's input.images[] to a blob URL on demand and shows it
          full-size, mirroring MinIO's own object-preview panel: dark
          overlay, centered image, filename header, close button, and
          prev/next navigation when a question has more than one image. */}
      {imagePreviewOpen && (
        <div className={styles['ev-imgview-overlay']} onClick={closeImagePreview}>
          <div
            className={styles['ev-imgview-panel']}
            onClick={(e) => e.stopPropagation()}
            role="dialog"
            aria-modal="true"
            aria-label="Image preview"
          >
            <div className={styles['ev-imgview-head']}>
              <span className={styles['ev-imgview-path']} title={imagePreviewPaths[imagePreviewIndex]}>
                <ImageIcon size={14} />
                {imagePreviewPaths[imagePreviewIndex]?.split('/').pop() || 'Image'}
              </span>
              {imagePreviewPaths.length > 1 && (
                <span className={styles['ev-imgview-count']}>
                  {imagePreviewIndex + 1} / {imagePreviewPaths.length}
                </span>
              )}
              <button type="button" className={styles['ev-imgview-close']} onClick={closeImagePreview} title="Close">
                <X size={16} />
              </button>
            </div>

            <div className={styles['ev-imgview-body']}>
              {imagePreviewPaths.length > 1 && (
                <button
                  type="button"
                  className={`${styles['ev-imgview-nav']} ${styles['ev-imgview-nav--prev']}`}
                  onClick={showPrevImage}
                  disabled={imagePreviewIndex === 0}
                  title="Previous image"
                >
                  <ChevronLeft size={18} />
                </button>
              )}

              {imagePreviewLoading && (
                <div className={styles['ev-imgview-status']}>
                  <Loader2 size={22} className={styles.ev__spin} />
                  <span>Loading image…</span>
                </div>
              )}

              {!imagePreviewLoading && imagePreviewError && (
                <div className={styles['ev-imgview-status']}>
                  <p className={styles.ev__error}>{imagePreviewError}</p>
                </div>
              )}

              {!imagePreviewLoading && !imagePreviewError && imagePreviewUrl && (
                <img
                  src={imagePreviewUrl}
                  alt={imagePreviewPaths[imagePreviewIndex] || 'Question image'}
                  className={styles['ev-imgview-img']}
                />
              )}

              {imagePreviewPaths.length > 1 && (
                <button
                  type="button"
                  className={`${styles['ev-imgview-nav']} ${styles['ev-imgview-nav--next']}`}
                  onClick={showNextImage}
                  disabled={imagePreviewIndex === imagePreviewPaths.length - 1}
                  title="Next image"
                >
                  <ChevronRight size={18} />
                </button>
              )}
            </div>
          </div>
        </div>
      )}

      {toast && (
        <div className={styles['ev-toast']}>
          <div className={styles['ev-toast__icon']}>
            <Check size={18} />
          </div>
          <div>
            <div className={styles['ev-toast__title']}>Run launched</div>
            <div className={styles['ev-toast__sub']}>You'll find it in your history once it completes.</div>
          </div>
        </div>
      )}
    </div>
  );
}
