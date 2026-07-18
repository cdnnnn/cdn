// ═══════════════════════════════════════════════
// pages/UploadInfer/FileSidebar.tsx
// Content Analytics · Shared 400px file sidebar
// Used by both the Infer tab (mode="select") and the Workspace tab
// (mode="view"). Deliberately does NOT read the Upload tab's shared
// `serverFiles` — it keeps its own local list, date filter, sort,
// search, and pagination, and fetches independently every time its
// tab becomes active. The only thing still shared with the rest of
// the app is `selectedServerIds`, since InferencePanel needs that to
// run a batch.
// ═══════════════════════════════════════════════
import React, { useCallback, useEffect, useRef, useState } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import { toggleServerFileSelection, toggleSelectAllServerFiles, ServerFile, FilesSortBy, SortOrder, FileStatus } from '../../store/uploadSlice';
import api from '../../services/api';
import styles from './FileSidebar.module.scss';

const getExt = (n: string): 'vtt' | 'srt' => n.toLowerCase().endsWith('.srt') ? 'srt' : 'vtt';

function toDateStr(d: Date) { return d.toISOString().slice(0, 10); }
function defaultFrom() { const d = new Date(); d.setMonth(d.getMonth() - 1); return toDateStr(d); }
function defaultTo() { return toDateStr(new Date()); }

const PAGE_WINDOW = 4;
const getPageNumbers = (current: number, total: number): number[] => {
  if (total <= PAGE_WINDOW) return Array.from({ length: total }, (_, i) => i + 1);
  const start = Math.max(1, Math.min(current - 1, total - PAGE_WINDOW + 1));
  return Array.from({ length: PAGE_WINDOW }, (_, i) => start + i);
};

type SortKey = 'id' | 'name' | 'date' | 'status';
const SORT_KEY_TO_API: Record<SortKey, FilesSortBy> = {
  id: 'id', name: 'original_name', date: 'inserted_at', status: 'status',
};
const API_TO_SORT_KEY: Record<FilesSortBy, SortKey> = {
  id: 'id', original_name: 'name', inserted_at: 'date', status: 'status',
};

// ── Status → badge label/color — same 5-state mapping as the Upload tab ──
const STATUS_META: Record<FileStatus, { labelKey: string; cls: string }> = {
  waiting: { labelKey: 'uploadInfer.filePanel.statusWaiting', cls: 'bWaiting' },
  running: { labelKey: 'uploadInfer.filePanel.statusRunning', cls: 'bRunning' },
  completed: { labelKey: 'uploadInfer.filePanel.inferenced', cls: 'bInferred' },
  error: { labelKey: 'uploadInfer.filePanel.notInferenced', cls: 'bNotInferred' },
  tbd: { labelKey: 'uploadInfer.filePanel.notInferenced', cls: 'bNotInferred' },
  queued: { labelKey: 'uploadInfer.filePanel.statusQueued', cls: 'bQueued' },
};

const STATUS_FILTER_OPTIONS: { value: FileStatus | ''; labelKey: string }[] = [
  { value: '', labelKey: 'uploadInfer.filePanel.statusAll' },
  { value: 'waiting', labelKey: 'uploadInfer.filePanel.statusWaiting' },
  { value: 'queued', labelKey: 'uploadInfer.filePanel.statusQueued' },
  { value: 'running', labelKey: 'uploadInfer.filePanel.statusRunning' },
  { value: 'completed', labelKey: 'uploadInfer.filePanel.inferenced' },
  { value: 'error', labelKey: 'uploadInfer.filePanel.statusError' },
  { value: 'tbd', labelKey: 'uploadInfer.filePanel.statusTbd' },
];

interface CbProps { checked: boolean; indeterminate?: boolean; onChange: () => void; disabled?: boolean; }
const Checkbox: React.FC<CbProps> = ({ checked, indeterminate, onChange, disabled }) => (
  <div
    className={`${styles.cb} ${checked ? styles.cbChecked : ''} ${indeterminate ? styles.cbIndet : ''} ${disabled ? styles.cbDisabled : ''}`}
    onClick={e => { e.stopPropagation(); if (!disabled) onChange(); }}
    role="checkbox" aria-checked={indeterminate ? 'mixed' : checked} tabIndex={0}
    onKeyDown={e => { if (e.key === ' ' || e.key === 'Enter') { e.preventDefault(); if (!disabled) onChange(); } }}
  />
);

interface Props {
  // "select" — checkbox multi-select for building the inference batch (Infer tab)
  // "view"   — click a file to load its results (Workspace tab)
  mode: 'select' | 'view';
  // Whether this sidebar's tab is the one currently showing. A fresh
  // fetch fires each time this flips true — i.e. every time the user
  // navigates to the Infer or Result tab, not just on first mount.
  active: boolean;
  activeFileId?: number | null;
  onFileClick?: (fileId: number) => void;
}

const PAGE_SIZES = [25, 50, 75, 100];

const FileSidebar: React.FC<Props> = ({ mode, active, activeFileId = null, onFileClick }) => {
  const { t } = useTranslation();
  const dispatch = useAppDispatch();
  const selectedServerIds = useAppSelector(s => s.upload.selectedServerIds);
  const isBatchRunning = useAppSelector(s => s.upload.isBatchRunning);

  // ── Independent local file list ──
  const [files, setFiles] = useState<ServerFile[]>([]);
  const [total, setTotal] = useState(0);
  const [totalPages, setTotalPages] = useState(1);
  const [page, setPage] = useState(1);
  const [pageSize, setPageSize] = useState(50);
  const [sortBy, setSortBy] = useState<FilesSortBy>('id');
  const [sortOrder, setSortOrder] = useState<SortOrder>('desc');
  const [search, setSearch] = useState('');
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const [dateFrom, setDateFrom] = useState(defaultFrom());
  const [dateTo, setDateTo] = useState(defaultTo());
  const [applying, setApplying] = useState(false);
  const [statusFilter, setStatusFilter] = useState<FileStatus | ''>('');

  const fetchFiles = useCallback(async (opts?: {
    page?: number; pageSize?: number; from?: string; to?: string;
    sortBy?: FilesSortBy; sortOrder?: SortOrder; search?: string;
    statusFilter?: FileStatus | '';
  }) => {
    const p = opts?.page ?? page;
    const ps = opts?.pageSize ?? pageSize;
    const from = opts?.from ?? dateFrom;
    const to = opts?.to ?? dateTo;
    const sBy = opts?.sortBy ?? sortBy;
    const sOrd = opts?.sortOrder ?? sortOrder;
    const q = opts?.search ?? search;
    const statusF = opts?.statusFilter ?? statusFilter;
    setLoading(true);
    setError(null);
    try {
      const res = await api.post('/files/by-date/', {
        start_date: from,
        end_date: to,
        page: p,
        page_size: ps,
        sort_by: sBy,
        sort_order: sOrd,
        ...(q ? { search: q } : {}),
        ...(statusF ? { status_filter: statusF } : {}),
      });
      const d = (res.data as any)?.data ?? {};
      setFiles(d.data ?? []);
      setTotal(d.total ?? 0);
      setPage(d.page ?? p);
      setPageSize(d.page_size ?? ps);
      setTotalPages(d.total_pages ?? 1);
      setSortBy(sBy);
      setSortOrder(sOrd);
      setStatusFilter(statusF);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to load files.');
    } finally {
      setLoading(false);
    }
  }, [page, pageSize, dateFrom, dateTo, sortBy, sortOrder, search, statusFilter]);

  const handleApply = useCallback(async () => {
    setApplying(true);
    await fetchFiles({ page: 1 });
    setApplying(false);
  }, [fetchFiles]);

  const handleStatusFilterChange = (value: FileStatus | '') => {
    fetchFiles({ page: 1, statusFilter: value });
  };

  const goToPage = (p: number) => {
    if (p < 1 || p > totalPages || p === page || loading) return;
    fetchFiles({ page: p });
  };

  const handlePageSizeChange = (size: number) => {
    fetchFiles({ page: 1, pageSize: size });
  };

  const handleSort = useCallback((key: SortKey) => {
    const apiKey = SORT_KEY_TO_API[key];
    const dir: SortOrder = sortBy === apiKey ? (sortOrder === 'asc' ? 'desc' : 'asc') : 'asc';
    fetchFiles({ sortBy: apiKey, sortOrder: dir, page: 1 });
  }, [sortBy, sortOrder, fetchFiles]);
  const sort = { key: API_TO_SORT_KEY[sortBy], dir: sortOrder };

  // ── Search — server-side, debounced (matches the Upload tab's search) ──
  const [searchQuery, setSearchQuery] = useState('');
  const skipNextSearchEffect = useRef(true);
  useEffect(() => {
    if (skipNextSearchEffect.current) { skipNextSearchEffect.current = false; return; }
    const q = searchQuery.trim();
    const handle = setTimeout(() => {
      setSearch(q);
      fetchFiles({ search: q, page: 1 });
    }, 400);
    return () => clearTimeout(handle);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [searchQuery]);

  // Own API call, fired every time this tab becomes active — entirely
  // separate from whatever the Upload tab is doing. When the tab goes
  // inactive (navigated away from), clear the list and reset search/
  // sort/filter/page back to defaults so a return visit shows a clean
  // loading state and a genuinely fresh fetch, instead of the previous
  // tab's stale results flashing before the new data arrives.
  useEffect(() => {
    if (active) {
      fetchFiles({ page: 1 });
    } else {
      skipNextSearchEffect.current = true; // don't let the reset below trigger a second debounced fetch
      setFiles([]);
      setTotal(0);
      setTotalPages(1);
      setPage(1);
      setSortBy('id');
      setSortOrder('desc');
      setSearch('');
      setSearchQuery('');
      setStatusFilter('');
      setError(null);
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [active]);

  const pageSelectedCount = files.filter(f => selectedServerIds.includes(f.id)).length;
  const allSelected = files.length > 0 && pageSelectedCount === files.length;
  const someSelected = pageSelectedCount > 0 && !allSelected;

  const handleRowClick = (fileId: number) => {
    if (mode === 'select') {
      if (isBatchRunning) return;
      dispatch(toggleServerFileSelection(fileId));
    } else {
      onFileClick?.(fileId);
    }
  };

  return (
    <div className={styles.sidebar} data-tour={mode === 'select' ? 'infer-sidebar' : 'results-sidebar'}>
      <div className={styles.dateFilter}>
        <div className={styles.dateField}>
          <label className={styles.dateLabel}>{t('uploadInfer.filePanel.dateFrom')}</label>
          <input
            type="date" className={styles.dateInput} value={dateFrom} max={dateTo}
            disabled={applying}
            onChange={e => setDateFrom(e.target.value)}
          />
        </div>
        <div className={styles.dateSep}>—</div>
        <div className={styles.dateField}>
          <label className={styles.dateLabel}>{t('uploadInfer.filePanel.dateTo')}</label>
          <input
            type="date" className={styles.dateInput} value={dateTo} min={dateFrom}
            disabled={applying}
            onChange={e => setDateTo(e.target.value)}
          />
        </div>
        <button className={styles.applyBtn} onClick={handleApply} disabled={applying || loading}>
          {applying ? <span className={styles.miniSpinner} /> : t('uploadInfer.filePanel.applyDate')}
        </button>
      </div>

      <div className={styles.statusFilterWrap}>
        <svg className={styles.statusFilterIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
          <path d="M2 4h12M4.5 8h7M7 12h2" />
        </svg>
        <select
          className={styles.statusFilterSelect}
          value={statusFilter}
          disabled={loading}
          onChange={e => handleStatusFilterChange(e.target.value as FileStatus | '')}
        >
          {STATUS_FILTER_OPTIONS.map(opt => (
            <option key={opt.value || 'all'} value={opt.value}>{t(opt.labelKey)}</option>
          ))}
        </select>
      </div>

      <div className={styles.head}>
        {mode === 'select' && (
          <Checkbox
            checked={allSelected}
            indeterminate={someSelected}
            onChange={() => dispatch(toggleSelectAllServerFiles(files.map(f => f.id)))}
            disabled={files.length === 0 || isBatchRunning}
          />
        )}
        <span className={styles.headTitle}>{t('uploadInfer.workspace.filesTitle')}</span>
        {total > 0 && <span className={styles.headCount}>{total}</span>}
        {loading && <span className={styles.headSpinner} aria-label="Loading" />}
        {mode === 'select' && selectedServerIds.length > 0 && (
          <span className={styles.headSelected}>
            {t('uploadInfer.filePanel.selectedTotal', { count: selectedServerIds.length, total })}
          </span>
        )}
      </div>

      <div className={styles.searchWrap}>
        <svg className={styles.searchIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
          <circle cx="6.5" cy="6.5" r="4" /><path d="M11 11l2.5 2.5" />
        </svg>
        <input
          type="text"
          className={styles.searchInput}
          placeholder={t('uploadInfer.workspace.searchFiles')}
          value={searchQuery}
          onChange={e => setSearchQuery(e.target.value)}
        />
      </div>

      {/* Sort header — same sort keys as the Upload tab */}
      <div className={styles.sortHeader}>
        <span className={styles.sortHeaderLabel}>
          <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
            <path d="M2 4h10M4 7h6M6 10h2" />
          </svg>
          {t('uploadInfer.filePanel.sortBy')}
        </span>
        <div className={styles.sortCols}>
          {([
            { key: 'id' as SortKey, label: t('uploadInfer.filePanel.sortId') },
            { key: 'name' as SortKey, label: t('uploadInfer.filePanel.sortName') },
            { key: 'date' as SortKey, label: t('uploadInfer.filePanel.sortDate') },
            { key: 'status' as SortKey, label: t('uploadInfer.filePanel.sortStatus') },
          ]).map(({ key, label }) => (
            <button
              key={key}
              className={`${styles.sortCol} ${sort.key === key ? styles.sortColActive : ''}`}
              onClick={() => handleSort(key)}
              disabled={loading}
            >
              {label}
              <svg viewBox="0 0 10 12" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round"
                className={sort.key === key ? (sort.dir === 'asc' ? styles.sortAsc : styles.sortDesc) : styles.sortInactive}>
                <path d="M5 1v10M2 8l3 3 3-3" />
              </svg>
            </button>
          ))}
        </div>
      </div>

      <div className={styles.rowsWrap}>
        <div className={styles.rows}>
        {error && (
          <div className={styles.empty}>{error}</div>
        )}
        {!error && loading && files.length === 0 && (
          <div className={styles.emptyLoading}>
            <span className={styles.rowsSpinner} />
            {t('uploadInfer.workspace.loadingFiles')}
          </div>
        )}
        {!error && !loading && files.length === 0 && (
          <div className={styles.empty}>
            {search ? t('uploadInfer.workspace.noFilesMatch') : t('uploadInfer.workspace.noFilesYet')}
          </div>
        )}
        {files.map(f => {
          const ext = getExt(f.original_name);
          const isChecked = mode === 'select' && selectedServerIds.includes(f.id);
          const isActiveView = mode === 'view' && f.id === activeFileId;
          const disabled = mode === 'select' && isBatchRunning;
          const statusMeta = STATUS_META[f.status] ?? STATUS_META.tbd;
          return (
            <div
              key={f.id}
              role="button"
              tabIndex={0}
              className={`${styles.hitm} ${styles.hitmSelectable}
                ${isChecked ? styles.active : ''}
                ${isActiveView ? styles.activeView : ''}
                ${disabled ? styles.rowDisabled : ''}`}
              onClick={() => !disabled && handleRowClick(f.id)}
              onKeyDown={e => { if (!disabled && (e.key === 'Enter' || e.key === ' ')) { e.preventDefault(); handleRowClick(f.id); } }}
            >
              {mode === 'select' && (
                <Checkbox checked={isChecked} onChange={() => handleRowClick(f.id)} disabled={isBatchRunning} />
              )}
              <div className={`${styles.ficon} ${styles[ext]}`}>{ext.toUpperCase()}</div>
              <div className={styles.hi}>
                <div className={styles.hn}>{f.original_name}</div>
                <div className={styles.hm}>{f.inserted_at} · #{f.id}</div>
              </div>
              <span className={`${styles.badge} ${isChecked ? styles.bSelected : styles[statusMeta.cls]}`}>
                {isChecked ? (
                  <svg viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" width="10" height="10">
                    <path d="M2 6l3 3 5-5" />
                  </svg>
                ) : (
                  <>
                    {f.status === 'completed' && (
                      <svg viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" width="9" height="9">
                        <path d="M2 6l3 3 5-5" />
                      </svg>
                    )}
                    {(f.status === 'running' || f.status === 'queued') && (
                      <span className={styles.badgeDot} />
                    )}
                    {t(statusMeta.labelKey)}
                  </>
                )}
              </span>
            </div>
          );
        })}
        </div>

        {/* Blocks interaction with the current list while a fetch is in
            flight (pagination, sort, search, filter) — only shown when
            there's already content on screen; the empty-state spinner
            above covers the first load. */}
        {loading && files.length > 0 && (
          <div className={styles.rowsLoadingOverlay}>
            <span className={styles.rowsSpinner} />
          </div>
        )}
      </div>

      {/* Pagination footer — same page-number layout as the Upload tab */}
      {!loading && !error && total > 0 && (
        <div className={styles.paginationBar}>
          <div className={styles.paginationTopRow}>
            <div className={styles.pageSizeGroup}>
              <span className={styles.pageSizeLabel}>{t('uploadInfer.filePanel.perPage')}</span>
              <select
                className={styles.pageSizeSelect}
                value={pageSize}
                onChange={e => handlePageSizeChange(Number(e.target.value))}
              >
                {PAGE_SIZES.map(size => (
                  <option key={size} value={size}>{size}</option>
                ))}
              </select>
            </div>

            <div className={styles.pageNav}>
              <button
                className={styles.pageNavBtn}
                onClick={() => goToPage(page - 1)}
                disabled={page <= 1}
                aria-label="Previous page"
              >
                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                  <path d="M10 3L6 8l4 5" />
                </svg>
              </button>

              {(() => {
                const pageWindow = getPageNumbers(page, totalPages);
                const showFirst = pageWindow[0] > 1;
                const showLast = pageWindow[pageWindow.length - 1] < totalPages;
                return (
                  <>
                    {showFirst && (
                      <>
                        <button className={styles.pageNumBtn} onClick={() => goToPage(1)}>1</button>
                        {pageWindow[0] > 2 && <span className={styles.pageEllipsis}>…</span>}
                      </>
                    )}
                    {pageWindow.map(p => (
                      <button
                        key={p}
                        className={`${styles.pageNumBtn} ${p === page ? styles.pageNumBtnActive : ''}`}
                        onClick={() => goToPage(p)}
                      >
                        {p}
                      </button>
                    ))}
                    {showLast && (
                      <>
                        {pageWindow[pageWindow.length - 1] < totalPages - 1 && <span className={styles.pageEllipsis}>…</span>}
                        <button className={styles.pageNumBtn} onClick={() => goToPage(totalPages)}>{totalPages}</button>
                      </>
                    )}
                  </>
                );
              })()}

              <button
                className={styles.pageNavBtn}
                onClick={() => goToPage(page + 1)}
                disabled={page >= totalPages}
                aria-label="Next page"
              >
                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                  <path d="M6 3l4 5-4 5" />
                </svg>
              </button>
            </div>
          </div>

          <div className={styles.pageInfo}>
            {t('uploadInfer.filePanel.pageInfo', { page, totalPages, total })}
          </div>
        </div>
      )}
    </div>
  );
};

export default FileSidebar;









// ═══════════════════════════════════════════════
// FileSidebar.module.scss
// Content Analytics · Shared sidebar for the Infer + Workspace tabs
// ═══════════════════════════════════════════════
@use '../../styles/mixins' as m;

.sidebar {
  width: 400px;
  flex-shrink: 0;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  background: var(--bg0);
  border-right: 1px solid var(--bdr);

  :global(html.light) & {
    background: #fff;
  }
}

// ── Date range filter ─────────────────────────
.dateFilter {
  display: flex;
  align-items: flex-end;
  gap: 6px;
  padding: 12px 14px;
  flex-shrink: 0;
  border-bottom: 1px solid var(--bdr);
}

.dateField {
  display: flex;
  flex-direction: column;
  gap: 3px;
  flex: 1;
  min-width: 0;
}

.dateLabel {
  font-size: 10px;
  font-weight: 600;
  color: var(--t2);
  text-transform: uppercase;
  letter-spacing: 0.1em;
  @include m.mono;
}

.dateInput {
  width: 100%;
  padding: 4px 7px;
  background: var(--bg1);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 13px;
  outline: none;
  appearance: none;
  transition: border-color 0.12s;
  cursor: pointer;

  &:focus {
    border-color: var(--blue);
    box-shadow: 0 0 0 2px var(--blue-dim);
  }

  &::-webkit-calendar-picker-indicator {
    opacity: 0.7;
    cursor: pointer;
    filter: var(--date-icon-filter);
  }
}

.dateSep {
  font-size: 13px;
  color: var(--t2);
  padding-bottom: 4px;
  flex-shrink: 0;
}

// ── Status filter ──────────────────────────────
.statusFilterWrap {
  display: flex;
  align-items: center;
  gap: 6px;
  margin: 0 14px 10px;
  flex-shrink: 0;
}

.statusFilterIcon {
  width: 14px;
  height: 14px;
  color: var(--t2);
  flex-shrink: 0;
}

.statusFilterSelect {
  flex: 1;
  height: 30px;
  padding: 0 8px;
  background: var(--bg1);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 12.5px;
  outline: none;
  cursor: pointer;
  transition: border-color 0.12s;

  &:focus { border-color: var(--blue); }
  &:disabled { opacity: 0.5; cursor: default; }
}

.applyBtn {
  align-self: flex-end;
  padding: 4px 12px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t1);
  font-family: var(--font-ui);
  font-size: 13px;
  font-weight: 500;
  cursor: pointer;
  flex-shrink: 0;
  transition: all 0.12s;

  &:hover:not(:disabled) {
    background: var(--bg3);
    border-color: var(--bdr3);
  }

  &:disabled {
    opacity: 0.6;
    cursor: default;
  }
}

.miniSpinner {
  display: inline-block;
  width: 12px;
  height: 12px;
  border: 2px solid var(--bdr3);
  border-top-color: var(--t1);
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
}

.headSpinner {
  display: inline-block;
  width: 11px;
  height: 11px;
  border: 2px solid var(--bdr3);
  border-top-color: var(--blue);
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
  flex-shrink: 0;
}

.rowsSpinner {
  display: inline-block;
  width: 18px;
  height: 18px;
  border: 2.5px solid var(--bdr3);
  border-top-color: var(--blue);
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
  margin-bottom: 8px;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

// ── List header ────────────────────────────────
.head {
  @include m.flex-center;
  gap: 8px;
  padding: 12px 14px 8px;
  flex-shrink: 0;
}

.headTitle {
  font-size: 12px;
  font-weight: 600;
  color: var(--t1);
}

.headCount {
  font-size: 11px;
  font-weight: 600;
  color: var(--t2);
  background: var(--bg2);
  border-radius: 99px;
  padding: 1px 7px;
}

.headSelected {
  margin-left: auto;
  font-size: 11px;
  color: var(--blue);
}

// ── Search ─────────────────────────────────────
.searchWrap {
  position: relative;
  margin: 0 14px 10px;
  flex-shrink: 0;
}

.searchIcon {
  position: absolute;
  left: 9px;
  top: 50%;
  transform: translateY(-50%);
  width: 13px;
  height: 13px;
  color: var(--t2);
  pointer-events: none;
}

.searchInput {
  width: 100%;
  height: 30px;
  padding: 0 10px 0 28px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: var(--bg1);
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 12px;

  &::placeholder { color: var(--t2); }
  &:focus { outline: none; border-color: var(--bdr3); }
}

// ── Rows ───────────────────────────────────────
.rowsWrap {
  position: relative;
  flex: 1;
  min-height: 0;
  display: flex;
  flex-direction: column;
}

.rows {
  flex: 1;
  overflow-y: auto;
  padding: 0 8px 12px;
  @include m.scrollbar;
}

.rowsLoadingOverlay {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--bg1);
  background: color-mix(in srgb, var(--bg0) 70%, transparent);
  cursor: wait;
  z-index: 2;
}

.empty {
  padding: 20px 12px;
  text-align: center;
  font-size: 12px;
  color: var(--t2);
}

.emptyLoading {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 24px 12px;
  text-align: center;
  font-size: 12px;
  color: var(--t2);
}

.row {
  display: flex;
  align-items: center;
  gap: 9px;
  padding: 8px 9px;
  border-radius: var(--r);
  color: var(--t1);
  cursor: pointer;
  @include m.theme-transition;

  &:hover { background: var(--bg2); }
}

.rowDisabled {
  opacity: 0.5;
  cursor: default;
  pointer-events: none;
}

// ── File row — mirrors FilePanel's pre-card .hitm design ──────
.hitm {
  position: relative;
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 9px;
  border-radius: var(--r);
  border: 1px solid transparent;
  transition: all 0.12s;
  margin-bottom: 3px;
  user-select: none;

  &:hover {
    background: var(--bg2);
    border-color: var(--bdr);
  }

  &.active {
    background: rgba(91, 164, 239, 0.06);
    border-color: var(--blue-bdr);

    &::before {
      content: '';
      position: absolute;
      left: 0;
      top: 6px;
      bottom: 6px;
      width: 3px;
      border-radius: 0 3px 3px 0;
      background: linear-gradient(180deg, var(--blue), #a78bfa);
    }
  }

  &.activeView {
    background: rgba(167, 139, 250, 0.06);
    border-color: rgba(167, 139, 250, 0.3);

    &::before {
      content: '';
      position: absolute;
      left: 0;
      top: 6px;
      bottom: 6px;
      width: 3px;
      border-radius: 0 3px 3px 0;
      background: linear-gradient(180deg, #a78bfa, var(--amber));
    }
  }
}

.hitmSelectable {
  cursor: pointer;
}

.ficon {
  width: 26px;
  height: 26px;
  border-radius: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 10px;
  font-weight: 700;
  @include m.mono;
  flex-shrink: 0;

  &.vtt { background: var(--blue-dim); color: var(--blue); }
  &.srt { background: var(--green-dim); color: var(--green); }
}

.hi {
  flex: 1;
  min-width: 0;
}

.hn {
  font-size: 13px;
  font-weight: 500;
  color: var(--t0);
  @include m.truncate;
}

.hm {
  font-size: 12px;
  color: var(--t2);
  margin-top: 2px;
  @include m.mono;
}

.badge {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-size: 11px;
  padding: 2px 7px;
  border-radius: 99px;
  font-weight: 500;
  border: 1px solid transparent;
  white-space: nowrap;
  flex-shrink: 0;
  @include m.mono;
}

.bSelected {
  background: var(--blue-dim);
  color: var(--blue);
  border-color: var(--blue-bdr);
}

.bInferred {
  background: var(--green-dim);
  color: var(--green);
  border-color: var(--green-bdr);
}

.bNotInferred {
  background: rgba(240, 160, 48, 0.1);
  color: var(--amber);
  border-color: rgba(240, 160, 48, 0.3);
}

.bRunning {
  background: var(--blue-dim);
  color: var(--blue);
  border-color: var(--blue-bdr);
}

.bQueued {
  background: rgba(240, 160, 48, 0.1);
  color: var(--amber);
  border-color: rgba(240, 160, 48, 0.3);
}

// Waiting — neutral gray, distinct from "not inferenced" (amber)
.bWaiting {
  background: var(--bg2);
  color: var(--t2);
  border-color: var(--bdr2);
}

.badgeDot {
  width: 5px;
  height: 5px;
  border-radius: 50%;
  background: currentColor;
  animation: fsBadgePulse 1.4s ease-in-out infinite;
}

@keyframes fsBadgePulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.35; }
}

// ── Checkbox (mirrors FilePanel's) ─────────────
.cb {
  width: 15px;
  height: 15px;
  flex-shrink: 0;
  border-radius: 4px;
  border: 1px solid var(--cb-bdr);
  background: transparent;
  cursor: pointer;
  position: relative;
  transition: all 0.12s;

  &:hover { border-color: var(--blue); }
}

.cbChecked {
  background: var(--blue);
  border-color: var(--blue);

  &::after {
    content: '';
    position: absolute;
    left: 4px;
    top: 1px;
    width: 4px;
    height: 8px;
    border: solid white;
    border-width: 0 2px 2px 0;
    transform: rotate(45deg);
  }
}

.cbIndet {
  background: var(--blue);
  border-color: var(--blue);

  &::after {
    content: '';
    position: absolute;
    left: 3px;
    top: 6px;
    width: 7px;
    height: 2px;
    background: white;
  }
}

.cbDisabled {
  opacity: 0.4;
  pointer-events: none;
}

// ── Pagination footer — matches the Upload tab's layout ──
.paginationBar {
  display: flex;
  flex-direction: column;
  gap: 6px;
  padding: 8px 12px;
  border-top: 1px solid var(--bdr);
  flex-shrink: 0;
}

.paginationTopRow {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  flex-wrap: nowrap;
  min-width: 0;
}

.pageSizeGroup {
  display: flex;
  align-items: center;
  gap: 6px;
  flex-shrink: 0;
}

.pageSizeLabel {
  font-size: 11px;
  color: var(--t2);
  white-space: nowrap;
}

.pageSizeSelect {
  padding: 3px 6px;
  background: var(--bg1);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  color: var(--t1);
  font-family: var(--font-ui);
  font-size: 12px;
  outline: none;
  cursor: pointer;
  transition: border-color 0.12s;

  &:focus { border-color: var(--blue); }
}

.pageNav {
  display: flex;
  align-items: center;
  gap: 4px;
}

.pageNavBtn {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 24px;
  height: 24px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: var(--bg1);
  color: var(--t2);
  cursor: pointer;
  transition: all 0.12s;
  flex-shrink: 0;

  svg { width: 13px; height: 13px; }

  &:hover:not(:disabled) {
    background: var(--bg2);
    color: var(--t0);
  }

  &:disabled {
    opacity: 0.35;
    cursor: default;
  }
}

.pageNumBtn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 24px;
  height: 24px;
  padding: 0 6px;
  border-radius: var(--r);
  border: 1px solid transparent;
  background: transparent;
  color: var(--t1);
  font-family: var(--font-ui);
  font-size: 12px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.12s;
  user-select: none;
  flex-shrink: 0;

  &:hover:not(:disabled) {
    background: var(--bg3);
    color: var(--t0);
    border-color: var(--bdr3);
  }

  &:disabled {
    opacity: 0.35;
    cursor: default;
  }
}

.pageNumBtnActive {
  background: var(--blue-dim);
  border-color: var(--blue-bdr);
  color: var(--blue);
  font-weight: 700;

  &:hover:not(:disabled) {
    background: var(--blue-dim);
    color: var(--blue);
  }
}

.pageEllipsis {
  color: var(--t2);
  font-size: 12px;
  padding: 0 2px;
  user-select: none;
  flex-shrink: 0;
}

.pageInfo {
  font-size: 11px;
  color: var(--t2);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  text-align: right;
  @include m.mono;
}

.pageIndicator {
  font-size: 12px;
  color: var(--t2);
  @include m.mono;
  min-width: 44px;
  text-align: center;
}

// ── Sort header — same as the Upload tab's sort chip ──
.sortHeader {
  display: flex;
  align-items: center;
  height: 32px;
  background: var(--bg2);
  border: 1px solid var(--bdr);
  border-radius: var(--r);
  padding: 0 6px 0 10px;
  gap: 6px;
  margin: 0 14px 10px;
  flex-shrink: 0;
  overflow: hidden;
}

.sortHeaderLabel {
  display: flex;
  align-items: center;
  gap: 5px;
  font-size: 10px;
  font-weight: 600;
  color: var(--t2);
  font-family: var(--font-mono);
  text-transform: uppercase;
  letter-spacing: 0.06em;
  white-space: nowrap;
  flex-shrink: 0;

  svg {
    width: 11px;
    height: 11px;
    opacity: 0.6;
  }
}

.sortCols {
  display: flex;
  align-items: center;
  gap: 2px;
  flex: 1;
}

.sortCol {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 4px;
  height: 24px;
  padding: 0 6px;
  background: transparent;
  border: none;
  border-radius: 5px;
  color: var(--t2);
  font-size: 11.5px;
  font-family: var(--font-mono);
  cursor: pointer;
  transition: all 0.12s;
  white-space: nowrap;

  svg {
    width: 8px;
    height: 10px;
    flex-shrink: 0;
    transition: transform 0.2s;
  }

  &:hover:not(:disabled) {
    background: var(--bg3);
    color: var(--t1);
  }

  &:disabled {
    opacity: 0.5;
    cursor: default;
  }
}

.sortColActive {
  background: var(--blue-dim);
  color: var(--blue);
  font-weight: 600;

  &:hover:not(:disabled) {
    background: var(--blue-dim);
  }
}

.sortInactive {
  opacity: 0.25;
}

.sortAsc {
  transform: rotate(180deg);
}
