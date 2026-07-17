// ═══════════════════════════════════════════════
// pages/UploadInfer/FileSidebar.tsx
// Content Analytics · Shared 400px file sidebar
// Used by both the Infer tab (mode="select") and the Workspace tab
// (mode="view"). Deliberately does NOT read the Upload tab's shared
// `serverFiles` — it keeps its own local list, date filter, and
// pagination, and fetches independently every time its tab becomes
// active. The only thing still shared with the rest of the app is
// `selectedServerIds`, since InferencePanel needs that to run a batch.
// ═══════════════════════════════════════════════
import React, { useCallback, useEffect, useMemo, useState } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import { toggleServerFileSelection, toggleSelectAllServerFiles, ServerFile } from '../../store/uploadSlice';
import api from '../../services/api';
import styles from './FileSidebar.module.scss';

const getExt = (n: string): 'vtt' | 'srt' => n.toLowerCase().endsWith('.srt') ? 'srt' : 'vtt';

function toDateStr(d: Date) { return d.toISOString().slice(0, 10); }
function defaultFrom() { const d = new Date(); d.setMonth(d.getMonth() - 1); return toDateStr(d); }
function defaultTo() { return toDateStr(new Date()); }

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
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const [dateFrom, setDateFrom] = useState(defaultFrom());
  const [dateTo, setDateTo] = useState(defaultTo());
  const [applying, setApplying] = useState(false);

  const fetchFiles = useCallback(async (opts?: { page?: number; pageSize?: number; from?: string; to?: string }) => {
    const p = opts?.page ?? page;
    const ps = opts?.pageSize ?? pageSize;
    const from = opts?.from ?? dateFrom;
    const to = opts?.to ?? dateTo;
    setLoading(true);
    setError(null);
    try {
      const res = await api.post('/files/by-date/', {
        start_date: from,
        end_date: to,
        page: p,
        page_size: ps,
      });
      const d = (res.data as any)?.data ?? {};
      setFiles(d.data ?? []);
      setTotal(d.total ?? 0);
      setPage(d.page ?? p);
      setPageSize(d.page_size ?? ps);
      setTotalPages(d.total_pages ?? 1);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to load files.');
    } finally {
      setLoading(false);
    }
  }, [page, pageSize, dateFrom, dateTo]);

  // Own API call, fired every time this tab becomes active — entirely
  // separate from whatever the Upload tab is doing.
  useEffect(() => {
    if (active) fetchFiles({ page: 1 });
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [active]);

  const handleApply = useCallback(async () => {
    setApplying(true);
    await fetchFiles({ page: 1 });
    setApplying(false);
  }, [fetchFiles]);

  const goToPage = (p: number) => {
    if (p < 1 || p > totalPages || p === page) return;
    fetchFiles({ page: p });
  };

  const handlePageSizeChange = (size: number) => {
    fetchFiles({ page: 1, pageSize: size });
  };

  const [query, setQuery] = useState('');
  const filtered = useMemo(() => {
    const q = query.trim().toLowerCase();
    if (!q) return files;
    return files.filter(f => f.original_name.toLowerCase().includes(q));
  }, [files, query]);

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
    <div className={styles.sidebar}>
      <div className={styles.dateFilter}>
        <div className={styles.dateField}>
          <label className={styles.dateLabel}>{t('uploadInfer.filePanel.dateFrom')}</label>
          <input
            type="date" className={styles.dateInput} value={dateFrom} max={dateTo}
            disabled={loading}
            onChange={e => setDateFrom(e.target.value)}
          />
        </div>
        <div className={styles.dateSep}>—</div>
        <div className={styles.dateField}>
          <label className={styles.dateLabel}>{t('uploadInfer.filePanel.dateTo')}</label>
          <input
            type="date" className={styles.dateInput} value={dateTo} min={dateFrom}
            disabled={loading}
            onChange={e => setDateTo(e.target.value)}
          />
        </div>
        <button className={styles.applyBtn} onClick={handleApply} disabled={applying || loading}>
          {applying ? <span className={styles.miniSpinner} /> : t('uploadInfer.filePanel.applyDate')}
        </button>
      </div>

      <div className={styles.head}>
        {mode === 'select' && (
          <Checkbox
            checked={allSelected}
            indeterminate={someSelected}
            onChange={() => dispatch(toggleSelectAllServerFiles())}
            disabled={files.length === 0 || isBatchRunning}
          />
        )}
        <span className={styles.headTitle}>{t('uploadInfer.workspace.filesTitle')}</span>
        {total > 0 && <span className={styles.headCount}>{total}</span>}
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
          value={query}
          onChange={e => setQuery(e.target.value)}
        />
      </div>

      <div className={styles.rows}>
        {error && (
          <div className={styles.empty}>{error}</div>
        )}
        {!error && loading && files.length === 0 && (
          <div className={styles.empty}>{t('uploadInfer.workspace.loadingFiles')}</div>
        )}
        {!error && !loading && files.length === 0 && (
          <div className={styles.empty}>{t('uploadInfer.workspace.noFilesYet')}</div>
        )}
        {!error && !loading && files.length > 0 && filtered.length === 0 && (
          <div className={styles.empty}>{t('uploadInfer.workspace.noFilesMatch')}</div>
        )}
        {filtered.map(f => {
          const ext = getExt(f.original_name);
          const isChecked = mode === 'select' && selectedServerIds.includes(f.id);
          const isActiveView = mode === 'view' && f.id === activeFileId;
          const disabled = mode === 'select' && isBatchRunning;
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
              <span className={`${styles.badge} ${isChecked ? styles.bSelected : f.status === 'completed' ? styles.bInferred : styles.bNotInferred}`}>
                {isChecked ? (
                  <svg viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" width="10" height="10">
                    <path d="M2 6l3 3 5-5" />
                  </svg>
                ) : f.status === 'completed' ? (
                  <>
                    <svg viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" width="9" height="9">
                      <path d="M2 6l3 3 5-5" />
                    </svg>
                    {t('uploadInfer.filePanel.inferenced')}
                  </>
                ) : t('uploadInfer.filePanel.notInferenced')}
              </span>
            </div>
          );
        })}
      </div>

      {/* Pagination footer */}
      {!loading && !error && total > 0 && (
        <div className={styles.paginationBar}>
          <select
            className={styles.pageSizeSelect}
            value={pageSize}
            onChange={e => handlePageSizeChange(Number(e.target.value))}
          >
            {PAGE_SIZES.map(size => (
              <option key={size} value={size}>{t('uploadInfer.filePanel.perPageShort', { count: size })}</option>
            ))}
          </select>

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
            <span className={styles.pageIndicator}>{page} / {totalPages}</span>
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
.rows {
  flex: 1;
  overflow-y: auto;
  padding: 0 8px 12px;
  @include m.scrollbar;
}

.empty {
  padding: 20px 12px;
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
  background: var(--bg2);
  color: var(--t2);
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

// ── Pagination footer ──────────────────────────
.paginationBar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  padding: 8px 12px;
  border-top: 1px solid var(--bdr);
  flex-shrink: 0;
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
  gap: 6px;
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

.pageIndicator {
  font-size: 12px;
  color: var(--t2);
  @include m.mono;
  min-width: 44px;
  text-align: center;
}












// ═══════════════════════════════════════════════
// pages/UploadInfer/InferencePanel.tsx
// Content Analytics · Inference configuration + batch status
// ═══════════════════════════════════════════════
import React, { useEffect, useRef, useCallback, useState } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import {
  inferenceStatusSuccess, updateRunningProgress,
  updateSummaryPrompt, updateKeywordPrompt, updateQuestionPrompt,
  updateShortAnswerPrompt, updateTrueFalsePrompt, updateSettings,
  modelsLoading, modelsSuccess, modelsFailure, setSelectedModel,
  updateFilePrompts,
  type ServerFile, type ServerFilesData, type TimeInterval,
} from '../../store/uploadSlice';
import api from '../../services/api';
import styles from './InferencePanel.module.scss';
import { addToast } from '../../store/toastSlice';

// ── Batch status columns ──────────────────────────
const StatusCard: React.FC<{ file: ServerFile; variant: 'queued' | 'running' | 'completed'; onStop?: (id: number) => void; stopping?: boolean }> = ({ file, variant, onStop, stopping }) => {
  const { t } = useTranslation();
  const ext = file.original_name.toLowerCase().endsWith('.srt') ? 'srt' : 'vtt';
  return (
    <div className={`${styles.statusCardWrap} ${styles[variant + 'Wrap']}`}>
      <div className={`${styles.statusCard} ${styles[variant]}`}>

        {/* Left icon — ext badge for queued/running, green check for completed */}
        {variant === 'completed' ? (
          <div className={styles.completedCheck}>
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
              <path d="M3 8l3.5 3.5L13 5" />
            </svg>
          </div>
        ) : (
          <div className={`${styles.statusExt} ${styles[ext]}`}>{ext.toUpperCase()}</div>
        )}

        <div className={styles.statusInfo}>
          <div className={styles.statusNameRow}>
            <span className={`${styles.statusIdBadge} ${styles['statusIdBadge_' + variant]}`}>#{file.id}</span>
            <div className={styles.statusName}>{file.original_name}</div>
          </div>

          {variant === 'running' && typeof file.progress === 'number' && (
            <div className={styles.statusProgress}>
              <div className={styles.statusBar}>
                <div className={styles.statusFill} style={{ width: `${file.progress}%` }} />
              </div>
              <span className={styles.statusPct}>{file.progress}%</span>
            </div>
          )}

          {variant === 'queued' && (
            <div className={styles.queuedMeta}>
              <span className={styles.queuedDot} />
              {t('uploadInfer.inferencePanel.waitingQueue')}
            </div>
          )}

          {variant === 'completed' && (
            <div className={styles.completedMeta}>{file.inserted_at}</div>
          )}

          {variant === 'running' && typeof file.progress !== 'number' && (
            <div className={styles.statusDate}>{file.inserted_at}</div>
          )}
        </div>

        {/* Stop button — queued files only */}
        {variant === 'queued' && onStop && (
          <button
            className={styles.stopBtn}
            onClick={() => onStop(file.id)}
            disabled={stopping}
            title={t('uploadInfer.inferencePanel.stopFile')}
          >
            {stopping ? (
              <span className={styles.stopSpinner} />
            ) : (
              <svg viewBox="0 0 16 16" fill="currentColor" stroke="none">
                <rect x="4" y="4" width="8" height="8" rx="1.5" />
              </svg>
            )}
          </button>
        )}

      </div>
    </div>
  );
};

// ── InferencePanel ───────────────────────────────
interface InferencePanelProps {
  onClose?: () => void;
  minimized?: boolean;
  onToggleMinimize?: () => void;
}

const InferencePanel: React.FC<InferencePanelProps> = ({ onClose, minimized = false, onToggleMinimize }) => {
  const { t } = useTranslation();
  const dispatch = useAppDispatch();
  const {
    settings,
    selectedServerIds, isBatchRunning,
    serverFiles,
    models, modelsLoading: mlLoading, selectedModel,
    dateFrom, dateTo,
  } = useAppSelector(s => s.upload);

  const [running, setRunning] = React.useState(false);
  const [stoppingIds, setStoppingIds] = React.useState<Set<number>>(new Set());

  // ── Prompt save state ──────────────────────────
  const [promptSaving, setPromptSaving] = useState(false);
  const [promptSaveError, setPromptSaveError] = useState<string | null>(null);
  const promptSnapshot = useRef({ summary: '', keyword: '', question: '', shortAnswer: '', trueFalse: '' });
  const promptsDirty =
    settings.summaryPromptOverride !== promptSnapshot.current.summary ||
    settings.keywordPromptOverride !== promptSnapshot.current.keyword ||
    settings.questionPromptOverride !== promptSnapshot.current.question ||
    settings.shortAnswerPromptOverride !== promptSnapshot.current.shortAnswer ||
    settings.trueFalsePromptOverride !== promptSnapshot.current.trueFalse;

  const emptyBatch: ServerFilesData = { queued: [], running: [], completed: [], pending: [] };
  const [batchData, setBatchData] = React.useState<ServerFilesData>(emptyBatch);
  const [countdown, setCountdown] = React.useState(10);
  const countdownRef = useRef<ReturnType<typeof setInterval> | null>(null);
  const n = selectedServerIds.length;
  const canRunInference = n > 0 && !isBatchRunning && !!selectedModel;
  const pollingRef = useRef<ReturnType<typeof setInterval> | null>(null);

  // ── Fetch models ────────────────────────────
  const fetchModels = useCallback(async () => {
    dispatch(modelsLoading());
    try {
      const res = await api.get('/get_models');
      const result = (res.data as any)?.result ?? [];
      dispatch(modelsSuccess(result));
    } catch {
      dispatch(modelsFailure());
    }
  }, [dispatch]); // eslint-disable-line

  useEffect(() => { fetchModels(); }, []); // eslint-disable-line

  // ── Polling helper ───────────────────────────
  const fetchFilesForPolling = useCallback(async () => {
    try {
      const statusRes = await api.post('/files/by-progress/', { start_date: dateFrom, end_date: dateTo });
      const d = (statusRes.data as any)?.data;
      const data: ServerFilesData = {
        queued: d?.queued ?? [],
        completed: d?.completed ?? [],
        pending: d?.pending ?? [],
        running: d?.running ?? [],
      };
      dispatch(inferenceStatusSuccess(data));
      setBatchData(data);

      const stillRunning = data.running.length > 0 || data.queued.length > 0;

      if (data.running.length > 0) {
        const runningIds = data.running.map(f => f.id);
        try {
          const progressRes = await api.post('/files/progress/', { file_ids: runningIds });
          const progressMap = (progressRes.data as any)?.result ?? {};
          dispatch(updateRunningProgress(progressMap));
          setBatchData(prev => ({
            ...prev,
            running: prev.running.map(f => {
              const raw = progressMap[String(f.id)];
              if (raw === undefined) return f;
              const pct = typeof raw === 'string' ? parseFloat(raw) : raw;
              return { ...f, progress: isNaN(pct) ? f.progress : Math.min(100, Math.max(0, pct)) };
            }),
          }));
        } catch {
          // progress fetch failing is non-critical — silently ignore
        }
      }

      if (!stillRunning) setBatchData(emptyBatch);
      return stillRunning;
    } catch {
      return false;
    }
  }, [dispatch, dateFrom, dateTo]);

  const stopCountdown = useCallback(() => {
    if (countdownRef.current) {
      clearInterval(countdownRef.current);
      countdownRef.current = null;
    }
  }, []);

  const startCountdown = useCallback(() => {
    stopCountdown();
    setCountdown(10);
    countdownRef.current = setInterval(() => {
      setCountdown(prev => (prev <= 1 ? 10 : prev - 1));
    }, 1000);
  }, [stopCountdown]);

  const stopPolling = useCallback(() => {
    if (pollingRef.current) {
      clearInterval(pollingRef.current);
      pollingRef.current = null;
    }
    stopCountdown();
    setCountdown(10);
  }, [stopCountdown]);

  const startPolling = useCallback(() => {
    stopPolling();
    startCountdown();
    pollingRef.current = setInterval(async () => {
      const stillRunning = await fetchFilesForPolling();
      if (!stillRunning) stopPolling();
      else startCountdown();
    }, 10000);
  }, [fetchFilesForPolling, stopPolling, startCountdown]);

  useEffect(() => {
    if (isBatchRunning) {
      fetchFilesForPolling().then(stillRunning => {
        if (stillRunning) startPolling();
        else stopPolling();
      });
    } else {
      stopPolling();
    }
    return stopPolling;
  }, [isBatchRunning]); // eslint-disable-line

  // ── One-time bootstrap check on mount ──
  // /files/by-date/ is now paginated and no longer tells us whether a batch
  // is running (a running/queued file may simply be on another page), so we
  // can't rely on its response to seed isBatchRunning like before. Ask
  // /files/by-progress/ directly once on mount to catch a batch that was
  // already running before this page loaded (e.g. after a refresh).
  useEffect(() => {
    fetchFilesForPolling().then(stillRunning => { if (stillRunning) startPolling(); });
  }, []); // eslint-disable-line

  // ── Stop a queued file ───────────────────────
  const handleStop = useCallback(async (fileId: number) => {
    setStoppingIds(prev => new Set(prev).add(fileId));
    try {
      await api.patch('/files/stop', { fileID: [fileId] });
    } catch (err: any) {
      if (err?.response?.status === 403) {
        dispatch(addToast(t('uploadInfer.inferencePanel.stopAlready'), 'error'));
      } else {
        console.error('Stop file failed:', err);
      }
    } finally {
      await fetchFilesForPolling();
      setStoppingIds(prev => { const s = new Set(prev); s.delete(fileId); return s; });
    }
  }, [fetchFilesForPolling]);

  // ── Run inference ────────────────────────────
  const handleRun = async () => {
    if (!canRunInference) return;
    setRunning(true);
    try {
      await api.post('/batch_process', {
        all: false, // always false for now — file_ids-driven runs only
        file_ids: selectedServerIds,
        start_date: dateFrom,
        end_date: dateTo,
        model_name: selectedModel,
        summary_prompt: settings.summaryPromptOverride,
        faq_prompt: settings.questionPromptOverride,
        keywords_prompt: settings.keywordPromptOverride,
        short_answers_prompt: settings.shortAnswerPromptOverride,
        true_false_prompt: settings.trueFalsePromptOverride,
        generate_summary: settings.generateSummary,
        generate_keywords: settings.generateKeywords,
        generate_faq: settings.generateQuestions,
        generate_short_answer: settings.generateShortAnswer,
        generate_true_false: settings.generateTrueFalse,
        // Only meaningful — and only ever sent true — when Keywords itself is on.
        generate_keyword_insights: settings.generateKeywords && settings.generateKeywordInsights,
        timestamped_summary: settings.timestampedSummary,
        time_interval: settings.timeInterval,
      });
      const stillRunning = await fetchFilesForPolling();
      if (stillRunning) startPolling();
      dispatch(addToast(t('uploadInfer.inferencePanel.inferenceStarted'), 'success'));
    } catch (err) {
      console.error('Batch process failed:', err);
    } finally {
      setRunning(false);
    }
  };

  // ── Run button sub-label ─────────────────────
  const runBtnSubLabel = (() => {
    if (n === 0) return t('uploadInfer.inferencePanel.selectFilesFirst');
    if (!selectedModel) return t('uploadInfer.inferencePanel.noModelSelected');
    return `${n} file${n !== 1 ? 's' : ''} ready`;
  })();

  // ── Auto-fill prompts when exactly 1 file is selected ────────────
  const prevSelectionCount = useRef(0);
  useEffect(() => {
    const prev = prevSelectionCount.current;
    prevSelectionCount.current = n;

    if (n === 1) {
      const file = serverFiles.find(f => f.id === selectedServerIds[0]);
      if (file) {
        const s = file.summary_prompt ?? '';
        const k = file.keywords_prompt ?? '';
        const q = file.faq_prompt ?? '';
        const sa = file.short_answers_prompt ?? '';
        const tf = file.true_false_prompt ?? '';
        dispatch(updateSummaryPrompt(s));
        dispatch(updateKeywordPrompt(k));
        dispatch(updateQuestionPrompt(q));
        dispatch(updateShortAnswerPrompt(sa));
        dispatch(updateTrueFalsePrompt(tf));
        promptSnapshot.current = { summary: s, keyword: k, question: q, shortAnswer: sa, trueFalse: tf };
      }
    } else if (n !== prev) {
      dispatch(updateSummaryPrompt(''));
      dispatch(updateKeywordPrompt(''));
      dispatch(updateQuestionPrompt(''));
      dispatch(updateShortAnswerPrompt(''));
      dispatch(updateTrueFalsePrompt(''));
      promptSnapshot.current = { summary: '', keyword: '', question: '', shortAnswer: '', trueFalse: '' };
    }
    setPromptSaveError(null);
  }, [selectedServerIds]); // eslint-disable-line

  // ── Per-file prompt preview expand/collapse ──
  const [summaryExpanded, setSummaryExpanded] = useState(false);
  const [keywordExpanded, setKeywordExpanded] = useState(false);
  const [questionExpanded, setQuestionExpanded] = useState(false);
  const [shortAnswerExpanded, setShortAnswerExpanded] = useState(false);
  const [trueFalseExpanded, setTrueFalseExpanded] = useState(false);

  const prevSelectedCount = useRef(selectedServerIds.length);
  useEffect(() => {
    if (selectedServerIds.length !== prevSelectedCount.current) {
      prevSelectedCount.current = selectedServerIds.length;
      setSummaryExpanded(false);
      setKeywordExpanded(false);
      setQuestionExpanded(false);
      setShortAnswerExpanded(false);
      setTrueFalseExpanded(false);
    }
  });

  const selectedFiles = serverFiles.filter(f => selectedServerIds.includes(f.id));

  const handlePromptCancel = () => {
    dispatch(updateSummaryPrompt(promptSnapshot.current.summary));
    dispatch(updateKeywordPrompt(promptSnapshot.current.keyword));
    dispatch(updateQuestionPrompt(promptSnapshot.current.question));
    dispatch(updateShortAnswerPrompt(promptSnapshot.current.shortAnswer));
    dispatch(updateTrueFalsePrompt(promptSnapshot.current.trueFalse));
    setPromptSaveError(null);
  };

  const handlePromptSave = async () => {
    setPromptSaving(true);
    setPromptSaveError(null);
    try {
      const payload: Record<string, unknown> = {
        file_ids: selectedServerIds,
      };
      if (settings.generateSummary) payload.summary_prompt = settings.summaryPromptOverride;
      if (settings.generateKeywords) payload.keywords_prompt = settings.keywordPromptOverride;
      if (settings.generateQuestions) payload.faq_prompt = settings.questionPromptOverride;
      if (settings.generateShortAnswer) payload.short_answers_prompt = settings.shortAnswerPromptOverride;
      if (settings.generateTrueFalse) payload.true_false_prompt = settings.trueFalsePromptOverride;

      await api.post('/prompt_update', payload);

      dispatch(updateFilePrompts({
        fileIds: selectedServerIds,
        ...(settings.generateSummary && { summaryPrompt: settings.summaryPromptOverride }),
        ...(settings.generateKeywords && { keywordsPrompt: settings.keywordPromptOverride }),
        ...(settings.generateQuestions && { faqPrompt: settings.questionPromptOverride }),
        ...(settings.generateShortAnswer && { shortAnswerPrompt: settings.shortAnswerPromptOverride }),
        ...(settings.generateTrueFalse && { trueFalsePrompt: settings.trueFalsePromptOverride }),
      }));

      promptSnapshot.current = {
        summary: settings.summaryPromptOverride,
        keyword: settings.keywordPromptOverride,
        question: settings.questionPromptOverride,
        shortAnswer: settings.shortAnswerPromptOverride,
        trueFalse: settings.trueFalsePromptOverride,
      };
    } catch {
      setPromptSaveError(t('uploadInfer.inferencePanel.promptSaveFail'));
    } finally {
      setPromptSaving(false);
    }
  };

  return (
    <div className={`${styles.infpanel} ${minimized ? styles.infpanelMinimized : ''}`}>

      {/* ── Minimized rail ── */}
      {minimized ? (
        <button
          type="button"
          className={styles.minRail}
          onClick={onToggleMinimize}
          title={t('uploadInfer.inferencePanel.expandStep2')}
          aria-label={t('uploadInfer.inferencePanel.expandStep2')}
        >
          <span className={styles.minRailIcon}>
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
              strokeWidth="1.6" strokeLinecap="round" strokeLinejoin="round">
              <path d="M6 4l4 4-4 4" />
            </svg>
          </span>
          <span className={styles.minRailLabel}>
            {t('uploadInfer.inferencePanel.step2Label').replace('Configuration', '').replace('2 —', '2 —')}
            {isBatchRunning && <span className={styles.minRailDot} />}
          </span>
        </button>
      ) : (
        <>

          {/* ── Header ── */}
          <div className={styles.infpanelHead}>
            <div>
              <div className={styles.slbl}>
                {isBatchRunning ? (
                  <span className={styles.inferenceRunningLabel}>{t('uploadInfer.inferencePanel.step2Running')}</span>
                ) : t('uploadInfer.inferencePanel.step2Label')}
              </div>
              <div className={styles.selSummary}>
                {t('uploadInfer.inferencePanel.filesSelected', { count: n })}
                {isBatchRunning && <span className={styles.batchRunPill}><span className={styles.batchRunDot} />{t('uploadInfer.inferencePanel.batchRunPill')}</span>}
              </div>
            </div>
            <div className={styles.headActions}>
              {!isBatchRunning && (
                <>
                  {/* ── Run Inference button — large, self-describing ── */}
                  <button
                    className={`${styles.runBtn} ${canRunInference ? styles.runBtnReady : styles.runBtnDisabled}`}
                    onClick={handleRun}
                    disabled={!canRunInference}
                    aria-label={`Run inference on ${n} file${n !== 1 ? 's' : ''}`}
                  >
                    <span className={styles.runBtnIconWrap}>
                      <svg width="15" height="15" viewBox="0 0 16 16" aria-hidden="true">
                        <path d="M5 3l8 5-8 5V3z" fill="currentColor" />
                      </svg>
                    </span>
                    <span className={styles.runBtnText}>
                      <span className={styles.runBtnTitle}>Run inference</span>
                      <span className={styles.runBtnSub}>{runBtnSubLabel}</span>
                    </span>
                  </button>

                  {onToggleMinimize && (
                    <button
                      className={styles.minimizeStepBtn}
                      onClick={onToggleMinimize}
                      title={t('uploadInfer.inferencePanel.minimizeStep2')}
                      aria-label={t('uploadInfer.inferencePanel.minimizeStep2')}
                    >
                      <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                        strokeWidth="1.6" strokeLinecap="round">
                        <path d="M3 8h10" />
                      </svg>
                    </button>
                  )}
                  {onClose && (
                    <button
                      className={styles.closeStepBtn}
                      onClick={onClose}
                      title={t('uploadInfer.inferencePanel.closeStep2')}
                    >
                      <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                        <path d="M4 4l8 8M12 4l-8 8" />
                      </svg>
                    </button>
                  )}
                </>
              )}
              {isBatchRunning && onToggleMinimize && (
                <button
                  className={styles.minimizeStepBtn}
                  onClick={onToggleMinimize}
                  title={t('uploadInfer.inferencePanel.minimizeStep2')}
                  aria-label={t('uploadInfer.inferencePanel.minimizeStep2')}
                >
                  <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                    strokeWidth="1.6" strokeLinecap="round">
                    <path d="M3 8h10" />
                  </svg>
                </button>
              )}
            </div>
          </div>


          {/* ── Main body ── */}
          <div className={`${styles.infpanelBody} ${isBatchRunning ? styles.infpanelBodyRunning : styles.infpanelBodyConfig}`}>

            {/* ── Submitting overlay — shown while awaiting /batch_process ── */}
            {running && (
              <div className={styles.submittingOverlay}>
                <div className={styles.submittingCard}>
                  <div className={styles.submittingSpinner} />
                  <div className={styles.submittingTitle}>{t('uploadInfer.inferencePanel.submitting')}</div>
                  <div className={styles.submittingDesc}>
                    {t('uploadInfer.inferencePanel.submittingDesc', { count: n }).split('\n').map((line, i) => <React.Fragment key={i}>{line}{i === 0 && <br />}</React.Fragment>)}
                  </div>
                  <div className={styles.submittingFiles}>
                    {selectedServerIds.slice(0, 5).map((id, i) => {
                      const f = serverFiles.find(sf => sf.id === id);
                      return f ? (
                        <div key={id} className={styles.submittingFile}>
                          <span className={styles.submittingDot} style={{ animationDelay: `${i * 0.15}s` }} />
                          {f.original_name}
                        </div>
                      ) : null;
                    })}
                    {selectedServerIds.length > 5 && (
                      <div className={styles.submittingMore}>+{selectedServerIds.length - 5} more</div>
                    )}
                  </div>
                </div>
              </div>
            )}

            {/* Selection banner — hidden while batch running or submitting */}
            {!isBatchRunning && !running && <div className={styles.selBanner}>
              <svg width="12" height="12" viewBox="0 0 16 16" fill="none" stroke="var(--blue)" strokeWidth="1.5" strokeLinecap="round">
                <path d="M2 8h12M8 3l5 5-5 5" />
              </svg>
              <span className={styles.selCt}>{n} file{n !== 1 ? 's' : ''}</span>
              <span className={styles.selNm}>
                {n === 0 ? t('uploadInfer.inferencePanel.selBannerEmpty') : `${n} file${n !== 1 ? 's' : ''} selected for inference`}
              </span>
            </div>}

            {/* ── Settings — hidden while batch running or submitting ── */}
            {!isBatchRunning && !running && <div className={styles.infSettingsWrap}>
              <div className={styles.settingsGrid}>

                {/* Generate content card */}
                <div className={styles.card}>
                  <div className={styles.cardT}>{t('uploadInfer.inferencePanel.generateContent')}</div>

                  {/* Summary */}
                  <div
                    className={`${styles.cr} ${settings.generateSummary ? styles.ck : ''} ${styles.mb8}`}
                    onClick={() => dispatch(updateSettings({ generateSummary: !settings.generateSummary }))}
                  >
                    <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateSummary')}</label>
                  </div>

                  {settings.generateSummary && (
                    <div className={styles.promptInline} onClick={(e) => e.stopPropagation()}>
                      <div className={styles.fg}>
                        <div className={styles.flRow}>
                          <label className={styles.fl}>
                            {t('uploadInfer.inferencePanel.summaryPrompt')} <span className={styles.optTag}>{t('uploadInfer.inferencePanel.optional')}</span>
                          </label>
                          {n > 1 && (
                            <button
                              className={`${styles.perFileBtn} ${summaryExpanded ? styles.perFileBtnActive : ''}`}
                              onClick={(e) => { e.stopPropagation(); setSummaryExpanded(v => !v); }}
                              title={summaryExpanded ? 'Collapse per-file prompts' : 'View per-file prompts'}
                            >
                              <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                <rect x="1.5" y="2" width="11" height="10" rx="2" />
                                <path d="M4 5h6M4 7.5h4" />
                              </svg>
                              {summaryExpanded ? 'Hide' : t('uploadInfer.inferencePanel.perFile')}
                              <svg className={`${styles.perFileChevron} ${summaryExpanded ? styles.perFileChevronOpen : ''}`} viewBox="0 0 10 10" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                                <path d="M2 3.5l3 3 3-3" />
                              </svg>
                            </button>
                          )}
                        </div>
                        <div className={`${styles.perFilePanel} ${summaryExpanded ? styles.perFilePanelOpen : ''}`}>
                          <div className={styles.perFilePanelInner}>
                            {selectedFiles.map(f => (
                              <div key={f.id} className={styles.perFileRow}>
                                <div className={styles.perFileName}>{f.original_name}</div>
                                <div className={styles.perFilePrompt}>{f.summary_prompt || <span className={styles.perFileEmpty}>{t('uploadInfer.inferencePanel.noPromptSet')}</span>}</div>
                              </div>
                            ))}
                          </div>
                        </div>
                        <textarea className={styles.fc} rows={3}
                          value={settings.summaryPromptOverride}
                          placeholder={n === 1 ? t('uploadInfer.inferencePanel.autofillPlaceholder') : t('uploadInfer.inferencePanel.manualPlaceholder')}
                          onChange={e => dispatch(updateSummaryPrompt(e.target.value))} />
                      </div>
                    </div>
                  )}

                  {/* Keywords + nested Keyword Insights */}
                  <div
                    className={`${styles.cr} ${settings.generateKeywords ? styles.ck : ''} ${styles.mb8}`}
                    onClick={() => dispatch(updateSettings({
                      generateKeywords: !settings.generateKeywords,
                      // Force the nested toggle off when the parent turns off,
                      // so a stale "on" state can never leak into the request.
                      ...(settings.generateKeywords ? { generateKeywordInsights: false } : {}),
                    }))}
                  >
                    <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateKeywords')}</label>
                  </div>

                  {settings.generateKeywords && (
                    <div className={styles.promptInline} onClick={(e) => e.stopPropagation()}>
                      <div className={styles.fg}>
                        <div className={styles.flRow}>
                          <label className={styles.fl}>
                            {t('uploadInfer.inferencePanel.keywordPrompt')} <span className={styles.optTag}>{t('uploadInfer.inferencePanel.optional')}</span>
                          </label>
                          {n > 1 && (
                            <button
                              className={`${styles.perFileBtn} ${keywordExpanded ? styles.perFileBtnActive : ''}`}
                              onClick={(e) => { e.stopPropagation(); setKeywordExpanded(v => !v); }}
                              title={keywordExpanded ? 'Collapse per-file prompts' : 'View per-file prompts'}
                            >
                              <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                <rect x="1.5" y="2" width="11" height="10" rx="2" />
                                <path d="M4 5h6M4 7.5h4" />
                              </svg>
                              {keywordExpanded ? t('uploadInfer.inferencePanel.hide') : t('uploadInfer.inferencePanel.perFile')}
                              <svg className={`${styles.perFileChevron} ${keywordExpanded ? styles.perFileChevronOpen : ''}`} viewBox="0 0 10 10" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                                <path d="M2 3.5l3 3 3-3" />
                              </svg>
                            </button>
                          )}
                        </div>
                        <div className={`${styles.perFilePanel} ${keywordExpanded ? styles.perFilePanelOpen : ''}`}>
                          <div className={styles.perFilePanelInner}>
                            {selectedFiles.map(f => (
                              <div key={f.id} className={styles.perFileRow}>
                                <div className={styles.perFileName}>{f.original_name}</div>
                                <div className={styles.perFilePrompt}>{f.keywords_prompt || <span className={styles.perFileEmpty}>No prompt set</span>}</div>
                              </div>
                            ))}
                          </div>
                        </div>
                        <textarea className={styles.fc} rows={3}
                          value={settings.keywordPromptOverride}
                          placeholder={n === 1 ? 'Autofilled from file — edit to override…' : 'Leave blank to use per-file prompts…'}
                          onChange={e => dispatch(updateKeywordPrompt(e.target.value))} />
                      </div>
                    </div>
                  )}
                  {settings.generateKeywords && (
                    <div
                      className={`${styles.crNested} ${settings.generateKeywordInsights ? styles.ck : ''} ${styles.mb8}`}
                      onClick={(e) => { e.stopPropagation(); dispatch(updateSettings({ generateKeywordInsights: !settings.generateKeywordInsights })); }}
                    >
                      <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateKeywordInsights')}</label>
                    </div>
                  )}

                  {/* Assessment Questions */}
                  <div
                    className={`${styles.cr} ${settings.generateQuestions ? styles.ck : ''} ${styles.mb8}`}
                    onClick={() => dispatch(updateSettings({ generateQuestions: !settings.generateQuestions }))}
                  >
                    <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateQuestions')}</label>
                  </div>

                  {settings.generateQuestions && (
                    <div className={styles.promptInline} onClick={(e) => e.stopPropagation()}>
                      <div className={styles.fg}>
                        <div className={styles.flRow}>
                          <label className={styles.fl}>
                            {t('uploadInfer.inferencePanel.questionPrompt')} <span className={styles.optTag}>{t('uploadInfer.inferencePanel.optional')}</span>
                          </label>
                          {n > 1 && (
                            <button
                              className={`${styles.perFileBtn} ${questionExpanded ? styles.perFileBtnActive : ''}`}
                              onClick={(e) => { e.stopPropagation(); setQuestionExpanded(v => !v); }}
                              title={questionExpanded ? 'Collapse per-file prompts' : 'View per-file prompts'}
                            >
                              <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                <rect x="1.5" y="2" width="11" height="10" rx="2" />
                                <path d="M4 5h6M4 7.5h4" />
                              </svg>
                              {questionExpanded ? t('uploadInfer.inferencePanel.hide') : t('uploadInfer.inferencePanel.perFile')}
                              <svg className={`${styles.perFileChevron} ${questionExpanded ? styles.perFileChevronOpen : ''}`} viewBox="0 0 10 10" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                                <path d="M2 3.5l3 3 3-3" />
                              </svg>
                            </button>
                          )}
                        </div>
                        <div className={`${styles.perFilePanel} ${questionExpanded ? styles.perFilePanelOpen : ''}`}>
                          <div className={styles.perFilePanelInner}>
                            {selectedFiles.map(f => (
                              <div key={f.id} className={styles.perFileRow}>
                                <div className={styles.perFileName}>{f.original_name}</div>
                                <div className={styles.perFilePrompt}>{f.faq_prompt || <span className={styles.perFileEmpty}>No prompt set</span>}</div>
                              </div>
                            ))}
                          </div>
                        </div>
                        <textarea className={styles.fc} rows={3}
                          value={settings.questionPromptOverride}
                          placeholder={n === 1 ? 'Autofilled from file — edit to override…' : 'Leave blank to use per-file prompts…'}
                          onChange={e => dispatch(updateQuestionPrompt(e.target.value))} />
                      </div>
                    </div>
                  )}

                  {/* Short Answer */}
                  <div
                    className={`${styles.cr} ${settings.generateShortAnswer ? styles.ck : ''} ${styles.mb8}`}
                    onClick={() => dispatch(updateSettings({ generateShortAnswer: !settings.generateShortAnswer }))}
                  >
                    <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateShortAnswer')}</label>
                  </div>

                  {settings.generateShortAnswer && (
                    <div className={styles.promptInline} onClick={(e) => e.stopPropagation()}>
                      <div className={styles.fg}>
                        <div className={styles.flRow}>
                          <label className={styles.fl}>
                            {t('uploadInfer.inferencePanel.shortAnswerPrompt')} <span className={styles.optTag}>{t('uploadInfer.inferencePanel.optional')}</span>
                          </label>
                          {n > 1 && (
                            <button
                              className={`${styles.perFileBtn} ${shortAnswerExpanded ? styles.perFileBtnActive : ''}`}
                              onClick={(e) => { e.stopPropagation(); setShortAnswerExpanded(v => !v); }}
                              title={shortAnswerExpanded ? t('uploadInfer.inferencePanel.hide') : t('uploadInfer.inferencePanel.viewPerFile')}
                            >
                              <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                <rect x="1.5" y="2" width="11" height="10" rx="2" />
                                <path d="M4 5h6M4 7.5h4" />
                              </svg>
                              {shortAnswerExpanded ? t('uploadInfer.inferencePanel.hide') : t('uploadInfer.inferencePanel.perFile')}
                              <svg className={`${styles.perFileChevron} ${shortAnswerExpanded ? styles.perFileChevronOpen : ''}`} viewBox="0 0 10 10" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                                <path d="M2 3.5l3 3 3-3" />
                              </svg>
                            </button>
                          )}
                        </div>
                        <div className={`${styles.perFilePanel} ${shortAnswerExpanded ? styles.perFilePanelOpen : ''}`}>
                          <div className={styles.perFilePanelInner}>
                            {selectedFiles.map(f => (
                              <div key={f.id} className={styles.perFileRow}>
                                <div className={styles.perFileName}>{f.original_name}</div>
                                <div className={styles.perFilePrompt}>{f.short_answers_prompt || <span className={styles.perFileEmpty}>{t('uploadInfer.inferencePanel.noPromptSet')}</span>}</div>
                              </div>
                            ))}
                          </div>
                        </div>
                        <textarea className={styles.fc} rows={3}
                          value={settings.shortAnswerPromptOverride}
                          placeholder={n === 1 ? t('uploadInfer.inferencePanel.autofillPlaceholder') : t('uploadInfer.inferencePanel.manualPlaceholder')}
                          onChange={e => dispatch(updateShortAnswerPrompt(e.target.value))} />
                      </div>
                    </div>
                  )}

                  {/* True/False */}
                  <div
                    className={`${styles.cr} ${settings.generateTrueFalse ? styles.ck : ''} ${styles.mb8}`}
                    onClick={() => dispatch(updateSettings({ generateTrueFalse: !settings.generateTrueFalse }))}
                  >
                    <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateTrueFalse')}</label>
                  </div>

                  {settings.generateTrueFalse && (
                    <div className={styles.promptInline} onClick={(e) => e.stopPropagation()}>
                      <div className={styles.fg}>
                        <div className={styles.flRow}>
                          <label className={styles.fl}>
                            {t('uploadInfer.inferencePanel.trueFalsePrompt')} <span className={styles.optTag}>{t('uploadInfer.inferencePanel.optional')}</span>
                          </label>
                          {n > 1 && (
                            <button
                              className={`${styles.perFileBtn} ${trueFalseExpanded ? styles.perFileBtnActive : ''}`}
                              onClick={(e) => { e.stopPropagation(); setTrueFalseExpanded(v => !v); }}
                              title={trueFalseExpanded ? t('uploadInfer.inferencePanel.hide') : t('uploadInfer.inferencePanel.viewPerFile')}
                            >
                              <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                <rect x="1.5" y="2" width="11" height="10" rx="2" />
                                <path d="M4 5h6M4 7.5h4" />
                              </svg>
                              {trueFalseExpanded ? t('uploadInfer.inferencePanel.hide') : t('uploadInfer.inferencePanel.perFile')}
                              <svg className={`${styles.perFileChevron} ${trueFalseExpanded ? styles.perFileChevronOpen : ''}`} viewBox="0 0 10 10" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                                <path d="M2 3.5l3 3 3-3" />
                              </svg>
                            </button>
                          )}
                        </div>
                        <div className={`${styles.perFilePanel} ${trueFalseExpanded ? styles.perFilePanelOpen : ''}`}>
                          <div className={styles.perFilePanelInner}>
                            {selectedFiles.map(f => (
                              <div key={f.id} className={styles.perFileRow}>
                                <div className={styles.perFileName}>{f.original_name}</div>
                                <div className={styles.perFilePrompt}>{f.true_false_prompt || <span className={styles.perFileEmpty}>{t('uploadInfer.inferencePanel.noPromptSet')}</span>}</div>
                              </div>
                            ))}
                          </div>
                        </div>
                        <textarea className={styles.fc} rows={3}
                          value={settings.trueFalsePromptOverride}
                          placeholder={n === 1 ? t('uploadInfer.inferencePanel.autofillPlaceholder') : t('uploadInfer.inferencePanel.manualPlaceholder')}
                          onChange={e => dispatch(updateTrueFalsePrompt(e.target.value))} />
                      </div>
                    </div>
                  )}

                  {/* Timestamp Summary + nested interval dropdown */}
                  <div
                    className={`${styles.cr} ${settings.timestampedSummary ? styles.ck : ''} ${styles.mb8}`}
                    onClick={() => dispatch(updateSettings({ timestampedSummary: !settings.timestampedSummary }))}
                  >
                    <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.timestampedSummary')}</label>
                  </div>
                  {settings.timestampedSummary && (
                    <div className={styles.nestedField} onClick={(e) => e.stopPropagation()}>
                      <label>{t('uploadInfer.inferencePanel.timeInterval')}</label>
                      <select
                        className={styles.fc}
                        value={settings.timeInterval}
                        onChange={e => dispatch(updateSettings({ timeInterval: Number(e.target.value) as TimeInterval }))}
                      >
                        {([5, 10, 15, 20, 30, 45, 60] as const).map(min => (
                          <option key={min} value={min}>{t('uploadInfer.inferencePanel.minutesOption', { count: min })}</option>
                        ))}
                      </select>
                    </div>
                  )}

                  {/* Save / Cancel bar — persists any edited prompt overrides above */}
                  {n > 0 && (settings.generateSummary || settings.generateKeywords || settings.generateQuestions
                    || settings.generateShortAnswer || settings.generateTrueFalse) && (
                    <div className={styles.promptActions}>
                      {promptSaveError && (
                        <span className={styles.promptSaveError}>{promptSaveError}</span>
                      )}
                      <button
                        className={`${styles.btn} ${styles.btnSm} ${styles.btnGhost}`}
                        onClick={handlePromptCancel}
                        disabled={!promptsDirty || promptSaving}
                      >
                        Cancel
                      </button>
                      <button
                        className={`${styles.btn} ${styles.btnSm} ${styles.btnPrimary}`}
                        onClick={handlePromptSave}
                        disabled={!promptsDirty || promptSaving}
                      >
                        {promptSaving ? (
                          <><span className={styles.btnSpinner} />{t('uploadInfer.inferencePanel.saving')}</>
                        ) : t('uploadInfer.inferencePanel.savePrompts')}
                      </button>
                    </div>
                  )}
                </div>

                {/* Output settings card */}
                <div className={styles.card}>
                  <div className={styles.cardT}>{t('uploadInfer.inferencePanel.outputSettings')}</div>

                  {/* Model dropdown */}
                  <div className={styles.fg}>
                    <div className={styles.modelLabelRow}>
                      <label className={styles.fl}>
                        {t('uploadInfer.inferencePanel.modelLabel')} {mlLoading && <span className={styles.loadingDot}>…</span>}
                      </label>
                      <button
                        className={styles.refreshBtn}
                        onClick={fetchModels}
                        disabled={mlLoading}
                        title={t('uploadInfer.inferencePanel.refreshModels')}
                      >
                        <svg
                          viewBox="0 0 16 16" fill="none" stroke="currentColor"
                          strokeWidth="1.6" strokeLinecap="round" strokeLinejoin="round"
                          className={mlLoading ? styles.spinning : undefined}
                        >
                          <path d="M13.5 8A5.5 5.5 0 1 1 10 3.07" />
                          <path d="M10 2v3h3" />
                        </svg>
                        Refresh
                      </button>
                    </div>
                    <select className={styles.fc} value={selectedModel}
                      onChange={e => dispatch(setSelectedModel(e.target.value))}
                      disabled={mlLoading || models.length === 0}>
                      {models.length === 0
                        ? <option value="">{t('uploadInfer.inferencePanel.noModelsOption')}</option>
                        : models.map(m => <option key={m} value={m}>{m}</option>)
                      }
                    </select>
                    {!mlLoading && models.length === 0 && (
                      <div className={styles.modelWarn}>
                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                          <path d="M8 2L14 13H2L8 2z" /><path d="M8 7v3M8 11.5v.1" />
                        </svg>
                        {t('uploadInfer.inferencePanel.noModels')}
                      </div>
                    )}
                    {!mlLoading && models.length > 0 && (
                      <div className={styles.modelOk}>
                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                          <circle cx="8" cy="8" r="5.5" /><path d="M5.5 8l2 2 3-3" />
                        </svg>
                        {t('uploadInfer.inferencePanel.modelsAvailable', { count: models.length })}
                      </div>
                    )}
                  </div>
                </div>
              </div>
            </div>}

            {/* ══ 3-Column batch status ══ */}
            {isBatchRunning && (
              <div className={styles.batchSection}>
                <div className={styles.batchSectionTitle}>
                  <span className={styles.liveLabel}>{t('uploadInfer.inferencePanel.inferenceStatus')}</span>
                  {isBatchRunning && (
                    <span className={styles.liveStatus}>
                      <span className={styles.liveDot} />
                      Live
                      <span className={styles.liveSep}>·</span>
                      {t('uploadInfer.inferencePanel.refreshingIn', { sec: countdown })}
                      {batchData.running.length > 0 && (
                        <><span className={styles.liveSep}>·</span>
                          <span className={styles.liveFiles}>
                            {t('uploadInfer.inferencePanel.filesRunning', { count: batchData.running.length })}
                          </span></>
                      )}
                    </span>
                  )}
                </div>
                <div className={styles.batchColumns}>

                  {/* Queued */}
                  <div className={styles.batchCol}>
                    <div className={`${styles.batchColHead} ${styles.headQueued}`}>
                      <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round">
                        <circle cx="7" cy="7" r="5" /><path d="M7 4v3.5l2 1.5" />
                      </svg>
                      Queued
                      <span className={styles.batchColCount}>{batchData.queued.length}</span>
                    </div>
                    <div className={styles.batchColBody}>
                      {batchData.queued.length === 0
                        ? <div className={styles.batchEmpty}>—</div>
                        : batchData.queued.map(f => (
                          <StatusCard
                            key={f.id}
                            file={f}
                            variant="queued"
                            onStop={handleStop}
                            stopping={stoppingIds.has(f.id)}
                          />
                        ))
                      }
                    </div>
                  </div>

                  {/* Running */}
                  <div className={styles.batchCol}>
                    <div className={`${styles.batchColHead} ${styles.headRunning}`}>
                      <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round">
                        <circle cx="7" cy="7" r="5" /><path d="M7 4v3M7 9.5v.2" />
                      </svg>
                      Running
                      <span className={styles.batchColCount}>{batchData.running.length}</span>
                    </div>
                    <div className={styles.batchColBody}>
                      {batchData.running.length === 0
                        ? <div className={styles.batchEmpty}>—</div>
                        : batchData.running.map(f => <StatusCard key={f.id} file={f} variant="running" />)
                      }
                    </div>
                  </div>

                  {/* Completed */}
                  <div className={styles.batchCol}>
                    <div className={`${styles.batchColHead} ${styles.headCompleted}`}>
                      <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                        <circle cx="7" cy="7" r="5" /><path d="M4.5 7l2 2 3-3" />
                      </svg>
                      Completed
                      <span className={styles.batchColCount}>{batchData.completed.length}</span>
                    </div>
                    <div className={styles.batchColBody}>
                      {batchData.completed.length === 0
                        ? <div className={styles.batchEmpty}>—</div>
                        : batchData.completed.map(f => <StatusCard key={f.id} file={f} variant="completed" />)
                      }
                    </div>
                  </div>

                </div>
              </div>
            )}

          </div>
        </>
      )}
    </div>
  );
};

export default InferencePanel;












// ═══════════════════════════════════════════════
// InferencePanel.module.scss
// ═══════════════════════════════════════════════
@use '../../styles/mixins' as m;

// ── Panel shell ──
.infpanel {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  position: relative;
}

// ── Inference Running label in step header ──
.inferenceRunningLabel {
  background: linear-gradient(90deg, var(--blue), #a78bfa, var(--amber));
  background-size: 200% auto;
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  animation: labelShimmer 2.5s linear infinite;
  font-weight: 700;
}

@keyframes labelShimmer {
  0% {
    background-position: 0% center;
  }

  100% {
    background-position: 200% center;
  }
}

// ── Panel header ──
.infpanelHead {
  padding: 13px 18px 10px;
  border-bottom: none;
  background: var(--bg1);
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  position: relative;

  &::after {
    content: '';
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    height: 1px;
    background: linear-gradient(90deg,
        rgba(139, 92, 246, 0.0) 0%,
        rgba(139, 92, 246, 0.6) 20%,
        rgba(56, 196, 186, 0.7) 50%,
        rgba(240, 160, 48, 0.6) 80%,
        rgba(240, 160, 48, 0.0) 100%);
    pointer-events: none;
  }
}

.slbl {
  font-size: 12px;
  font-weight: 600;
  color: var(--t2);
  text-transform: uppercase;
  letter-spacing: 0.1em;
  @include m.mono;
  margin-bottom: 2px;
}

.selSummary {
  font-size: 13px;
  color: var(--t1);
}

.headActions {
  display: flex;
  align-items: center;
  gap: 8px;
}

.closeStepBtn {
  width: 28px;
  height: 28px;
  border-radius: 7px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  flex-shrink: 0;
  transition: all 0.14s;

  svg {
    width: 13px;
    height: 13px;
  }

  &:hover {
    background: rgba(239, 68, 68, 0.08);
    border-color: rgba(239, 68, 68, 0.3);
    color: #ef4444;
  }
}

// ══════════════════════════════════════
// RUN INFERENCE BUTTON
// ══════════════════════════════════════

.runBtn {
  display: inline-flex;
  align-items: center;
  gap: 10px;
  padding: 8px 16px 8px 10px;
  border-radius: 10px;
  border: 1px solid transparent;
  background: var(--blue);
  color: #ffffff;
  font-family: var(--font-ui);
  cursor: pointer;
  transition: background 0.15s, box-shadow 0.15s, opacity 0.15s, border-color 0.15s;
  white-space: nowrap;
  user-select: none;
  flex-shrink: 0;

  &:hover:not(:disabled) {
    background: #6bb3f5;
  }

  &:active:not(:disabled) {
    background: #4a90d9;
  }
}

// Icon chip inside the button
.runBtnIconWrap {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 30px;
  height: 30px;
  border-radius: 7px;
  background: rgba(255, 255, 255, 0.18);
  flex-shrink: 0;
  transition: background 0.14s;

  svg {
    display: block;
  }

  .runBtn:hover:not(:disabled) & {
    background: rgba(255, 255, 255, 0.25);
  }
}

// Two-line text stack
.runBtnText {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  line-height: 1.2;
  gap: 2px;
}

.runBtnTitle {
  font-size: 14px;
  font-weight: 700;
  letter-spacing: -0.1px;
}

.runBtnSub {
  font-size: 11px;
  font-weight: 500;
  opacity: 0.82;
  font-family: var(--font-mono);
}

// Ready state — glowing outline pulse to draw the eye
.runBtnReady {
  box-shadow:
    0 0 0 1px rgba(91, 164, 239, 0.7),
    0 2px 12px rgba(91, 164, 239, 0.35);
  animation: runReadyPulse 2.4s ease-in-out infinite;
}

@keyframes runReadyPulse {

  0%,
  100% {
    box-shadow:
      0 0 0 1px rgba(91, 164, 239, 0.7),
      0 2px 12px rgba(91, 164, 239, 0.30);
  }

  50% {
    box-shadow:
      0 0 0 2px rgba(91, 164, 239, 0.55),
      0 4px 20px rgba(91, 164, 239, 0.50);
  }
}

// Disabled state — muted, clearly not actionable, sub-label explains why
.runBtnDisabled {
  opacity: 0.48;
  cursor: not-allowed;
  pointer-events: none;
  box-shadow: none;
}

// Large screen tweaks
@media (min-width: 1920px) {
  .runBtnTitle {
    font-size: 15px;
  }

  .runBtnSub {
    font-size: 12px;
  }

  .runBtnIconWrap {
    width: 32px;
    height: 32px;
  }

  .runBtn {
    padding: 9px 18px 9px 11px;
    gap: 11px;
    border-radius: 11px;
  }
}

// ── Main body ──
.infpanelBody {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  padding: 14px 18px 0;
}

// Configuration mode — body itself scrolls, batch section is absent
.infpanelBodyConfig {
  overflow-y: auto;
  @include m.scrollbar;
}

// Running mode — no scroll on body; each batch column scrolls independently
.infpanelBodyRunning {
  overflow: hidden;
}

// ── Selection banner ──
.selBanner {
  background: var(--bg2);
  border: 1px solid var(--blue-bdr);
  border-radius: var(--rl);
  padding: 9px 13px;
  margin-bottom: 13px;
  display: flex;
  align-items: center;
  gap: 9px;
}

.selCt {
  font-size: 13px;
  color: var(--blue);
  font-weight: 600;
  @include m.mono;
  flex-shrink: 0;
}

.selNm {
  font-size: 12px;
  color: var(--t2);
  @include m.mono;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  flex: 1;
}

// ── Collapsible settings wrap ──
.infSettingsWrap {
  padding-bottom: 16px;
  flex-shrink: 0;
}

.settingsGrid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 11px;
  margin-bottom: 12px;
}

.promptGrid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 11px;
}

.promptStack {
  display: flex;
  flex-direction: column;
  gap: 11px;
}

.noPromptHint {
  font-size: 13px;
  color: var(--t2);
  font-family: var(--font-mono);
  text-align: center;
  padding: 10px 0 2px;
}

// ── Card ──
.card {
  background: var(--bg1);
  border: 1px solid var(--bdr);
  border-radius: var(--rl);
  padding: 14px 16px;
}

.cardT {
  font-size: 10px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  color: var(--t2);
  margin-bottom: 12px;
  @include m.mono;
}

.promptMode {
  font-size: 12px;
  margin-left: 8px;
  font-weight: 400;
  text-transform: none;
  letter-spacing: 0;
  @include m.mono;
}

// ── Form ──
.fg {
  margin-bottom: 11px;
}

// Indents a prompt block under the checkbox it belongs to (same 20px
// indent convention as .nestedField, used for the timestamp interval).
.promptInline {
  padding-left: 20px;
  margin-bottom: 4px;
  cursor: default;
}

.mb0 {
  margin-bottom: 0 !important;
}

.mb8 {
  margin-bottom: 8px;
}

.mb12 {
  margin-bottom: 12px !important;
}

.flRow {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-bottom: 4px;
}

.fl {
  display: block;
  font-size: 13px;
  color: var(--t1);
  margin-bottom: 0;
  font-weight: 500;
}

.fc {
  width: 100%;
  padding: 7px 11px;
  background: var(--bg0);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 13px;
  transition: border-color 0.12s;
  outline: none;
  appearance: none;

  &:focus {
    border-color: var(--blue);
    box-shadow: 0 0 0 3px rgba(91, 164, 239, 0.07);
  }

  &::placeholder {
    color: var(--t2);
  }
}

select.fc {
  cursor: pointer;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='10' height='7'%3E%3Cpath d='M1 1l4 4 4-4' stroke='%234e5668' stroke-width='1.5' fill='none' stroke-linecap='round'/%3E%3C/svg%3E");
  background-repeat: no-repeat;
  background-position: right 10px center;
  padding-right: 26px;
}

textarea.fc {
  resize: vertical;
  min-height: 56px;
  line-height: 1.6;
}

.optTag {
  font-size: 12px;
  color: var(--t2);
  margin-left: 4px;
  font-weight: 400;
  @include m.mono;
}

// ── Checkboxes ──
.cr {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 3px 0;
  cursor: pointer;
  user-select: none;

  label {
    font-size: 13px;
    color: var(--t1);
    cursor: pointer;
  }
}

.cb {
  width: 14px;
  height: 14px;
  border: 1.5px solid var(--bdr2);
  border-radius: 4px;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  transition: all 0.12s;
}

.ck {
  .cb {
    background: var(--blue);
    border-color: var(--blue);

    &::after {
      content: '';
      width: 7px;
      height: 4px;
      border-left: 1.5px solid #0b0d10;
      border-bottom: 1.5px solid #0b0d10;
      transform: rotate(-45deg) translate(0, -1px);
      display: block;
    }
  }

  label {
    color: var(--t0);
  }
}

// Nested option, e.g. "Keyword Insights" under Keywords — indented, only
// rendered while its parent checkbox is on.
.crNested {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 3px 0 3px 20px;
  cursor: pointer;
  user-select: none;
  position: relative;

  &::before {
    content: '';
    position: absolute;
    left: 6px;
    top: 0;
    bottom: 50%;
    width: 8px;
    border-left: 1.5px solid var(--bdr2);
    border-bottom: 1.5px solid var(--bdr2);
    border-radius: 0 0 0 4px;
  }

  label {
    font-size: 12.5px;
    color: var(--t1);
    cursor: pointer;
  }

  &.ck label {
    color: var(--t0);
  }
}

// Nested field, e.g. the Timestamp Summary Interval dropdown — indented to
// visually sit under its parent checkbox.
.nestedField {
  padding-left: 20px;
  margin-bottom: 8px;

  label {
    display: block;
    font-size: 11.5px;
    color: var(--t2);
    margin-bottom: 4px;
  }

  select {
    max-width: 200px;
  }
}

// ── Range sliders ──
.rangeInput {
  -webkit-appearance: none;
  width: 100%;
  height: 3px;
  background: var(--bg4);
  border-radius: 99px;
  outline: none;
  cursor: pointer;
  display: block;
  margin-top: 4px;

  &::-webkit-slider-thumb {
    -webkit-appearance: none;
    width: 14px;
    height: 14px;
    background: var(--blue);
    border-radius: 50%;
    border: 2px solid var(--bg0);
    cursor: pointer;
    box-shadow: 0 0 0 2px rgba(91, 164, 239, 0.25);
  }
}

.slim {
  display: flex;
  justify-content: space-between;
  font-size: 12px;
  color: var(--t2);
  margin-top: 3px;
  @include m.mono;
}

.sv {
  @include m.mono;
  font-size: 13px;
  color: var(--blue);
}

.qTypeLabel {
  font-size: 13px;
  color: var(--t1);
  font-weight: 500;
  margin-bottom: 5px;
}

.dimLabel {
  color: var(--t2) !important;
}

.feasibilityTag {
  font-size: 10px;
  background: var(--amber-dim);
  color: var(--amber);
  padding: 1px 5px;
  border-radius: 99px;
  margin-left: 3px;
}

// ── Divider ──
.divider {
  border: none;
  height: 1px;
  margin: 12px 0;
  background: linear-gradient(90deg,
      rgba(139, 92, 246, 0.0) 0%,
      rgba(139, 92, 246, 0.35) 25%,
      rgba(56, 196, 186, 0.4) 50%,
      rgba(240, 160, 48, 0.35) 75%,
      rgba(240, 160, 48, 0.0) 100%);
}

// ── Callout ──
.callout {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  padding: 9px 12px;
  border-radius: var(--r);
  font-size: 13px;
  margin-bottom: 10px;
}

.cInfo {
  background: var(--blue-dim);
  border: 1px solid var(--blue-bdr);
  color: var(--blue);
}

// ── Generic utility buttons (used for save/cancel etc.) ──
.btn {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 6px 13px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t1);
  font-family: var(--font-ui);
  font-size: 13px;
  cursor: pointer;
  transition: all 0.12s;
  white-space: nowrap;
  user-select: none;

  &:hover {
    background: var(--bg3);
    color: var(--t0);
    border-color: var(--bdr3);
  }
}

.btnP {
  background: var(--blue);
  color: #ffffff;
  border-color: var(--blue);
  font-weight: 600;

  &:hover {
    background: #a78bfa;
    border-color: #a78bfa;
    color: #ffffff;
  }

  &:disabled {
    opacity: 0.4;
    cursor: not-allowed;
    pointer-events: none;
  }
}

// ── Batch running pill in header ─────────────────
.batchRunPill {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  font-size: 12px;
  color: var(--amber);
  background: var(--amber-dim);
  border: 1px solid var(--amber-bdr);
  padding: 1px 8px;
  border-radius: 99px;
  margin-left: 10px;
  font-family: var(--font-mono);
  vertical-align: middle;
}

.batchRunDot {
  width: 5px;
  height: 5px;
  border-radius: 50%;
  background: var(--amber);
  animation: breathe 1.2s ease-in-out infinite;
}

@keyframes breathe {

  0%,
  100% {
    opacity: 1;
  }

  50% {
    opacity: 0.35;
  }
}

// ── Model loading indicator ───────────────────────
.loadingDot {
  font-size: 13px;
  color: var(--t2);
  font-family: var(--font-mono);
}

// ══════════════════════════════════════
// 3-COLUMN BATCH STATUS
// ══════════════════════════════════════
.batchSection {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  margin-top: 16px;
  animation: fadeIn 0.2s ease;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(4px);
  }

  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.batchSectionTitle {
  font-size: 12px;
  font-weight: 600;
  color: var(--t2);
  text-transform: uppercase;
  letter-spacing: 0.1em;
  font-family: var(--font-mono);
  margin-bottom: 10px;
  display: flex;
  align-items: center;
  gap: 10px;
  flex-shrink: 0;
  flex-wrap: wrap;
}

.liveLabel {
  flex-shrink: 0;
}

// ── Live status pill ─────────────────────────────
.liveStatus {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  font-size: 12px;
  font-weight: 500;
  color: var(--green);
  background: var(--green-dim);
  border: 1px solid var(--green-bdr);
  padding: 2px 9px 2px 7px;
  border-radius: 99px;
  font-family: var(--font-mono);
  text-transform: none;
  letter-spacing: 0;
  white-space: nowrap;
}

.liveDot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: var(--green);
  flex-shrink: 0;
  animation: livePulse 1.4s ease-in-out infinite;
}

@keyframes livePulse {

  0%,
  100% {
    opacity: 1;
    transform: scale(1);
  }

  50% {
    opacity: 0.4;
    transform: scale(0.75);
  }
}

.liveSep {
  color: var(--green);
  opacity: 0.4;
  font-size: 12px;
}

.liveCountdown {
  color: var(--green);
  font-weight: 700;
  font-variant-numeric: tabular-nums;
  min-width: 2ch;
  display: inline-block;
  text-align: right;
}

.liveFiles {
  color: var(--green);
  opacity: 0.85;
}

.batchColumns {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 10px;
  flex: 1;
  overflow: hidden;
  min-height: 0;
}

.batchCol {
  display: flex;
  flex-direction: column;
  gap: 6px;
  min-width: 0;
  min-height: 0;
  overflow: hidden;
}

// Column headers
.batchColHead {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 12px;
  font-weight: 700;
  padding: 6px 10px;
  border-radius: 8px;
  font-family: var(--font-mono);
  text-transform: uppercase;
  letter-spacing: 0.06em;
  flex-shrink: 0;

  svg {
    width: 12px;
    height: 12px;
    flex-shrink: 0;
  }

  &.headQueued {
    background: rgba(240, 160, 48, 0.08);
    color: var(--amber);
    border: 1px solid var(--amber-bdr);

    svg {
      stroke: var(--amber);
    }
  }

  &.headRunning {
    background: var(--blue-dim);
    color: var(--blue);
    border: 1px solid var(--blue-bdr);

    svg {
      stroke: var(--blue);
    }
  }

  &.headCompleted {
    background: var(--green-dim);
    color: var(--green);
    border: 1px solid var(--green-bdr);

    svg {
      stroke: var(--green);
    }
  }
}

.batchColCount {
  margin-left: auto;
  font-size: 13px;
  font-weight: 700;
  min-width: 16px;
  text-align: center;
}

.batchColBody {
  display: flex;
  flex-direction: column;
  gap: 5px;
  flex: 1;
  overflow-y: auto;
  padding-bottom: 14px;
  @include m.scrollbar;
}

.batchEmpty {
  text-align: center;
  font-size: 13px;
  color: var(--t2);
  font-family: var(--font-mono);
  padding: 12px 0;
}

// ── Status card wrapper (handles enter/exit animation) ──
.statusCardWrap {
  position: relative;
  border-radius: 10px;
}

// ── Queued: slow breathing amber dashed-style border ──
.queuedWrap {
  padding: 2px;
  background: conic-gradient(from 0deg,
      rgba(240, 160, 48, 0.7) 0deg,
      rgba(240, 160, 48, 0.15) 60deg,
      rgba(240, 160, 48, 0.05) 120deg,
      rgba(240, 160, 48, 0.15) 180deg,
      rgba(240, 160, 48, 0.05) 240deg,
      rgba(240, 160, 48, 0.15) 300deg,
      rgba(240, 160, 48, 0.7) 360deg);
  animation: queuedBreath 2.8s ease-in-out infinite;
}

@keyframes queuedBreath {

  0%,
  100% {
    opacity: 0.5;
  }

  50% {
    opacity: 1;
  }
}

// ── Completed: static clean green gradient border, no animation ──
.completedWrap {
  padding: 2px;
  background: linear-gradient(135deg,
      rgba(74, 222, 128, 0.9) 0%,
      rgba(74, 222, 128, 0.3) 40%,
      rgba(74, 222, 128, 0.6) 70%,
      rgba(74, 222, 128, 0.9) 100%);
}

// ── Animated gradient border for running cards ──
.runningWrap {
  border-radius: 10px;
  padding: 2px;
  background: conic-gradient(from var(--border-angle, 0deg),
      transparent 0deg,
      transparent 30deg,
      #5ba4ef 50deg,
      #a78bfa 65deg,
      #f0a030 80deg,
      transparent 100deg,
      transparent 360deg);
  animation: borderBeamSpin 2.4s linear infinite;
}

@property --border-angle {
  syntax: '<angle>';
  initial-value: 0deg;
  inherits: false;
}

@keyframes borderBeamSpin {
  to {
    --border-angle: 360deg;
  }
}

// Status cards
.statusCard {
  display: flex;
  align-items: flex-start;
  gap: 7px;
  padding: 8px 10px;
  border-radius: 8px;
  border: 1px solid;
  min-width: 0;
  position: relative;

  &.queued {
    background: var(--bg1);
    border-color: transparent;
    border-radius: 8px;
  }

  &.running {
    background: var(--bg1);
    border-color: transparent;
    border-radius: 8px;
  }

  &.completed {
    background: var(--bg1);
    border-color: transparent;
    border-radius: 8px;
  }
}

.statusExt {
  width: 26px;
  height: 26px;
  border-radius: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 10px;
  font-weight: 800;
  font-family: var(--font-mono);
  flex-shrink: 0;

  &.vtt {
    background: var(--blue-dim);
    color: var(--blue);
  }

  &.srt {
    background: var(--green-dim);
    color: var(--green);
  }
}

// ── Completed check icon ──
.completedCheck {
  width: 26px;
  height: 26px;
  border-radius: 50%;
  background: var(--green-dim);
  border: 1.5px solid var(--green-bdr);
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  color: var(--green);

  svg {
    width: 13px;
    height: 13px;
  }
}

// ── Queued "waiting" meta line ──
.queuedMeta {
  display: flex;
  align-items: center;
  gap: 5px;
  font-size: 10px;
  color: var(--amber);
  font-family: var(--font-mono);
  opacity: 0.8;
}

.queuedDot {
  width: 5px;
  height: 5px;
  border-radius: 50%;
  background: var(--amber);
  flex-shrink: 0;
  animation: queuedDotPulse 1.6s ease-in-out infinite;
}

@keyframes queuedDotPulse {

  0%,
  100% {
    opacity: 0.3;
    transform: scale(0.8);
  }

  50% {
    opacity: 1;
    transform: scale(1.1);
  }
}

// ── Completed date meta ──
.completedMeta {
  font-size: 10px;
  color: var(--green);
  font-family: var(--font-mono);
  opacity: 0.75;
}

.statusInfo {
  flex: 1;
  min-width: 0;
}

.statusNameRow {
  display: flex;
  align-items: center;
  gap: 5px;
  min-width: 0;
  margin-bottom: 3px;
  overflow: hidden;
}

.statusName {
  font-size: 13px;
  font-weight: 500;
  color: var(--t0);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  flex: 1;
  min-width: 0;
}

.statusIdBadge {
  font-size: 10px;
  font-weight: 700;
  font-family: var(--font-mono);
  border-radius: 4px;
  padding: 1px 5px;
  flex-shrink: 0;
  white-space: nowrap;
  letter-spacing: 0.02em;
  color: var(--blue);
  background: var(--blue-dim);
  border: 1px solid var(--blue-bdr);
}

.statusIdBadge_queued {
  color: var(--amber);
  background: rgba(240, 160, 48, 0.12);
  border-color: rgba(240, 160, 48, 0.3);
}

.statusIdBadge_running {
  color: var(--blue);
  background: var(--blue-dim);
  border-color: var(--blue-bdr);
}

.statusIdBadge_completed {
  color: var(--green);
  background: var(--green-dim);
  border-color: var(--green-bdr);
}

.statusDate {
  font-size: 10px;
  color: var(--t2);
  font-family: var(--font-mono);
}

.statusProgress {
  display: flex;
  align-items: center;
  gap: 5px;
}

.statusBar {
  flex: 1;
  height: 3px;
  background: var(--bg4);
  border-radius: 99px;
  overflow: hidden;
}

.statusFill {
  height: 100%;
  background: var(--blue);
  border-radius: 99px;
  transition: width 0.4s ease;
}

.statusPct {
  font-size: 10px;
  color: var(--blue);
  font-family: var(--font-mono);
  flex-shrink: 0;
}

// ── Model row (label + refresh) ──────────────────
.modelLabelRow {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 4px;

  .fl {
    margin-bottom: 0;
  }
}

.refreshBtn {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 2px 8px;
  border-radius: 99px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  font-family: var(--font-ui);
  font-size: 12px;
  cursor: pointer;
  transition: all 0.12s;
  flex-shrink: 0;

  svg {
    width: 10px;
    height: 10px;
    flex-shrink: 0;
    transition: transform 0.12s;
  }

  &:hover:not(:disabled) {
    background: var(--bg3);
    color: var(--t1);
    border-color: var(--bdr3);
  }

  &:disabled {
    opacity: 0.45;
    cursor: not-allowed;
  }
}

.spinning {
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  from {
    transform: rotate(0deg);
  }

  to {
    transform: rotate(360deg);
  }
}

// ── Model status messages ─────────────────────────
.modelWarn {
  display: flex;
  align-items: flex-start;
  gap: 6px;
  margin-top: 7px;
  padding: 7px 10px;
  border-radius: var(--r);
  background: var(--amber-dim);
  border: 1px solid var(--amber-bdr);
  color: var(--amber);
  font-size: 13px;
  line-height: 1.5;

  svg {
    width: 12px;
    height: 12px;
    flex-shrink: 0;
    margin-top: 1px;
    stroke: var(--amber);
  }
}

.modelOk {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-top: 6px;
  font-size: 12px;
  color: var(--t2);
  font-family: var(--font-mono);

  svg {
    width: 11px;
    height: 11px;
    flex-shrink: 0;
    stroke: var(--green);
  }
}

// ══════════════════════════════════════
// SUBMITTING OVERLAY
// ══════════════════════════════════════
.submittingOverlay {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  background: rgba(var(--bg0-rgb, 11, 13, 16), 0.82);
  backdrop-filter: blur(4px);
  z-index: 20;
  animation: fadeIn 0.18s ease;
}

.submittingCard {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
  padding: 28px 32px;
  background: var(--bg1);
  border: 1px solid var(--bdr2);
  border-radius: var(--rxl, 12px);
  max-width: 320px;
  width: 90%;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.28);
  text-align: center;
}

.submittingSpinner {
  width: 36px;
  height: 36px;
  border: 3px solid var(--bdr2);
  border-top-color: var(--blue);
  border-right-color: rgba(139, 92, 246, 0.5);
  border-radius: 50%;
  animation: spin 0.75s linear infinite;
  flex-shrink: 0;
}

.submittingTitle {
  font-size: 14px;
  font-weight: 600;
  color: var(--t0);
  letter-spacing: -0.2px;
}

.submittingDesc {
  font-size: 13px;
  color: var(--t2);
  font-family: var(--font-mono);
  line-height: 1.65;
}

.submittingFiles {
  display: flex;
  flex-direction: column;
  gap: 5px;
  width: 100%;
  margin-top: 2px;
}

.submittingFile {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 13px;
  color: var(--t1);
  font-family: var(--font-mono);
  background: var(--bg2);
  border: 1px solid var(--bdr);
  border-radius: var(--r);
  padding: 5px 10px;
  text-align: left;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.submittingDot {
  width: 5px;
  height: 5px;
  border-radius: 50%;
  background: var(--blue);
  flex-shrink: 0;
  animation: breathe 1.2s ease-in-out infinite;
}

.submittingMore {
  font-size: 12px;
  color: var(--t2);
  font-family: var(--font-mono);
  text-align: center;
  padding: 2px 0;
}

// ── Stop button on queued file cards ─────────────────────
.stopBtn {
  width: 24px;
  height: 24px;
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 6px;
  border: 1px solid rgba(239, 68, 68, 0.35);
  background: rgba(239, 68, 68, 0.08);
  color: #ef4444;
  cursor: pointer;
  padding: 0;
  transition: all 0.15s;
  margin-left: auto;

  svg {
    width: 9px;
    height: 9px;
  }

  &:hover:not(:disabled) {
    background: rgba(239, 68, 68, 0.18);
    border-color: rgba(239, 68, 68, 0.6);
    box-shadow: 0 0 6px rgba(239, 68, 68, 0.2);
  }

  &:disabled {
    opacity: 0.5;
    cursor: default;
  }
}

.stopSpinner {
  width: 10px;
  height: 10px;
  border: 1.5px solid rgba(239, 68, 68, 0.3);
  border-top-color: #ef4444;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
  flex-shrink: 0;
}

// ── Large screen overrides (> 1900px) ────────────────────
@media (min-width: 1920px) {
  .slbl {
    font-size: 13px;
  }

  .selSummary {
    font-size: 14px;
  }

  .selCt {
    font-size: 14px;
  }

  .selNm {
    font-size: 13px;
  }

  .cardT {
    font-size: 11px;
  }

  .fl {
    font-size: 14px;
  }

  .fc {
    font-size: 14px;
  }

  .optTag {
    font-size: 13px;
  }

  .cr label {
    font-size: 14px;
  }

  .slim {
    font-size: 13px;
  }

  .sv {
    font-size: 14px;
  }

  .qTypeLabel {
    font-size: 14px;
  }

  .noPromptHint {
    font-size: 14px;
  }

  .btn {
    font-size: 14px;
  }

  .batchSectionTitle {
    font-size: 13px;
  }

  .batchColHead {
    font-size: 13px;
  }

  .batchColCount {
    font-size: 14px;
  }

  .batchEmpty {
    font-size: 14px;
  }

  .statusName {
    font-size: 14px;
  }

  .statusIdBadge {
    font-size: 11px;
  }

  .statusDate {
    font-size: 11px;
  }

  .statusPct {
    font-size: 11px;
  }

  .queuedMeta {
    font-size: 11px;
  }

  .completedMeta {
    font-size: 11px;
  }

  .loadingDot {
    font-size: 14px;
  }

  .refreshBtn {
    font-size: 13px;
  }

  .modelWarn {
    font-size: 14px;
  }

  .modelOk {
    font-size: 13px;
  }

  .submittingTitle {
    font-size: 15px;
  }

  .submittingDesc {
    font-size: 14px;
  }

  .submittingFile {
    font-size: 14px;
  }

  .submittingMore {
    font-size: 13px;
  }

  .liveStatus {
    font-size: 13px;
  }
}

// ── Per-file prompt preview ───────────────────
.perFileBtn {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  margin-left: 8px;
  padding: 2px 8px 2px 6px;
  border-radius: 5px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  font-size: 11px;
  font-family: var(--font-ui);
  cursor: pointer;
  vertical-align: middle;
  transition: all 0.14s;

  svg:first-child {
    width: 12px;
    height: 12px;
    flex-shrink: 0;
  }
}

.perFileBtn:hover {
  background: var(--bg3);
  border-color: var(--bdr3);
  color: var(--t0);
}

.perFileBtnActive {
  background: var(--blue-dim);
  border-color: var(--blue-bdr);
  color: var(--blue);
}

.perFileBtnActive:hover {
  background: var(--blue-dim);
  border-color: var(--blue);
  color: var(--blue);
}

.perFileChevron {
  width: 10px;
  height: 10px;
  flex-shrink: 0;
  transition: transform 0.18s ease;
}

.perFileChevronOpen {
  transform: rotate(180deg);
}

// Animated expand container
.perFilePanel {
  display: grid;
  grid-template-rows: 0fr;
  opacity: 0;
  transition:
    grid-template-rows 0.22s cubic-bezier(0.4, 0, 0.2, 1),
    opacity 0.18s ease,
    margin 0.2s ease;
  margin-bottom: 0;
}

.perFilePanelOpen {
  grid-template-rows: 1fr;
  opacity: 1;
  margin-bottom: 6px;
}

.perFilePanelInner {
  min-height: 0;
  overflow: hidden;
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  background: var(--bg1);
  max-height: 200px;
  overflow-y: auto;
}

.perFileRow {
  padding: 8px 10px;
  border-bottom: 1px solid var(--bdr1);
  display: flex;
  flex-direction: column;
  gap: 3px;

  &:last-child {
    border-bottom: none;
  }
}

.perFileName {
  font-size: 11px;
  font-family: var(--font-mono);
  color: var(--t2);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.perFilePrompt {
  font-size: 12px;
  color: var(--t0);
  line-height: 1.5;
  white-space: pre-wrap;
  word-break: break-word;
}

.perFileEmpty {
  font-size: 12px;
  color: var(--t2);
  font-style: italic;
  opacity: 0.6;
}

@media (min-width: 1920px) {
  .perFileBtn {
    font-size: 12px;
  }

  .perFileName {
    font-size: 12px;
  }

  .perFilePrompt {
    font-size: 13px;
  }

  .perFileEmpty {
    font-size: 13px;
  }
}

// ── Prompt save / cancel bar ──────────────────
.promptActions {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 8px;
  padding: 10px 12px 12px;
  border-top: 1px solid var(--bdr1);
  margin-top: 4px;
}

.promptSaveError {
  flex: 1;
  font-size: 12px;
  color: var(--red, #ef4444);
  margin-right: 4px;
}

.btnSm {
  height: 30px;
  padding: 0 14px;
  font-size: 12px;
  border-radius: 6px;
  border: 1px solid transparent;
  font-family: var(--font-ui);
  font-weight: 500;
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  gap: 6px;
  transition: all 0.14s;

  &:disabled {
    opacity: 0.4;
    cursor: not-allowed;
  }
}

.btnGhost {
  background: transparent;
  border-color: var(--bdr2);
  color: var(--t1);

  &:hover:not(:disabled) {
    background: var(--bg3);
    border-color: var(--bdr3);
  }
}

.btnPrimary {
  background: var(--blue);
  border-color: var(--blue);
  color: #fff;

  &:hover:not(:disabled) {
    opacity: 0.88;
  }
}

.btnSpinner {
  width: 11px;
  height: 11px;
  border: 1.5px solid rgba(255, 255, 255, 0.35);
  border-top-color: #fff;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
  flex-shrink: 0;
}

@media (min-width: 1920px) {
  .btnSm {
    height: 34px;
    font-size: 13px;
    padding: 0 16px;
  }

  .promptSaveError {
    font-size: 13px;
  }
}

// ─────────────────────────────────────────────
// Minimize / minimized rail
// ─────────────────────────────────────────────

.minimizeStepBtn {
  width: 28px;
  height: 28px;
  border-radius: 7px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  flex-shrink: 0;
  transition: all 0.14s;

  svg {
    width: 13px;
    height: 13px;
  }

  &:hover {
    background: var(--bg3);
    border-color: var(--bdr3);
    color: var(--t0);
  }
}

.infpanelMinimized {
  flex: 1;
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.minRail {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: flex-start;
  gap: 14px;
  padding: 14px 0;
  width: 100%;
  background: var(--bg2);
  border: 1px solid var(--bdr);
  border-radius: var(--rl);
  color: var(--t1);
  cursor: pointer;
  transition: background 0.14s, border-color 0.14s, color 0.14s;

  &:hover {
    background: var(--bg3);
    border-color: var(--bdr2);
    color: var(--t0);
  }
}

.minRailIcon {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  border-radius: 7px;
  border: 1px solid var(--bdr2);
  color: var(--t2);
  flex-shrink: 0;

  svg {
    width: 14px;
    height: 14px;
  }

  .minRail:hover & {
    color: var(--t0);
    border-color: var(--bdr3);
  }
}

.minRailLabel {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  writing-mode: vertical-rl;
  transform: rotate(180deg);
  font-size: 12px;
  font-weight: 500;
  letter-spacing: 0.04em;
  color: var(--t1);
  white-space: nowrap;
  user-select: none;
}

.minRailDot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: var(--amber);
  animation: minRailPulse 1.2s infinite;
}

@keyframes minRailPulse {

  0%,
  100% {
    opacity: 1;
  }

  50% {
    opacity: 0.4;
  }
}















// ═══════════════════════════════════════════════
// pages/UploadInfer/UploadInfer.tsx
// LectureAI · Upload & Inference page
// ═══════════════════════════════════════════════
import React, { useState, useEffect, useCallback, useRef } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppSelector, useAppDispatch } from '../../store/hooks';
import { clearServerSelection } from '../../store/uploadSlice';
import api from '../../services/api';
import FilePanel from './FilePanel';
import InferencePanel from './InferencePanel';
import WorkspacePanel from './WorkspacePanel';
import FileSidebar from './FileSidebar';
import styles from './UploadInfer.module.scss';

// ── Keyword Insights (from GET /files/{id}) ──────
export interface KGEdge { type: string; source: string; target: string; }
export interface KGNode { id: string; label: string; title?: string; value?: number; }
export interface KnowledgeGraph { edges: KGEdge[]; nodes: KGNode[]; }
export interface WordCloudData { wordcloud: [string, number][]; complexity_map: Record<string, string>; }
export interface TimelineDataset { data: number[]; label: string; }
export interface TimelineData { labels: string[]; datasets: TimelineDataset[]; timestamped: boolean; }
export interface HeatmapData { matrix: number[][]; keywords: string[]; segments: string[]; timestamped: boolean; }
export interface ClusterItem { id: string; label: string; value: number; }
export interface ClustersData { clusters: Record<string, ClusterItem[]>; }
export interface FrequencyItem { count: number; keyword: string; relative_pct: number; first_mention_pct: number; }
export interface FrequencyData { data: FrequencyItem[]; }
export interface PrereqEdge { reason: string; enables: string; prerequisite: string; }
export interface PrereqNode { id: string; label: string; value: number; }
export interface PrerequisitesData { edges: PrereqEdge[]; nodes: PrereqNode[]; }
export interface ImportanceComplexityItem { reason: string; keyword: string; frequency: number; complexity: string; importance: number; }
export interface ImportanceComplexityData { data: ImportanceComplexityItem[]; }
export interface CooccurrenceData { matrix: number[][]; keywords: string[]; }
export interface GlossaryItem { term: string; definition: string; first_mentioned_ms: number; }
export interface GlossaryData { glossary: GlossaryItem[]; }

export interface KeywordInsights {
  enriched_keywords: unknown | null;
  knowledge_graph: KnowledgeGraph | null;
  word_cloud: WordCloudData | null;
  timeline: TimelineData | null;
  heatmap: HeatmapData | null;
  clusters: ClustersData | null;
  frequency: FrequencyData | null;
  prerequisites: PrerequisitesData | null;
  importance_complexity: ImportanceComplexityData | null;
  cooccurrence: CooccurrenceData | null;
  congnitive_Load: unknown | null;
  segments: { segments: unknown[] } | null;
  glossary: GlossaryData | null;
}

export interface FileResult {
  summary: string;
  keywords: string[];
  faq: string;
  shortAnswer: string;
  trueFalse: string;
  timestampedSummary: string;
  keywordInsights: KeywordInsights | null;
  fileName: string;
  fileId: number;
  insertedAt: string;
}

// ── Tabs ──────────────────────────────────────────
// All three are always clickable — there's no gating on a previous tab
// being "complete". Every panel below stays mounted at all times (just
// hidden with CSS) so file lists, batch polling, and scroll position
// survive switching tabs instead of resetting.
type TabId = 'upload' | 'infer' | 'results';

const UploadInfer: React.FC = () => {
  const { t } = useTranslation();
  const isBatchRunning = useAppSelector(s => s.upload.isBatchRunning);
  const serverFilesData = useAppSelector(s => s.upload.serverFilesData);
  const filesTotal = useAppSelector(s => s.upload.filesTotal);
  const selectedServerIds = useAppSelector(s => s.upload.selectedServerIds);
  const dispatch = useAppDispatch();

  const [activeTab, setActiveTab] = useState<TabId>('upload');

  // "selectMode" is FilePanel's checkbox-selection UI for building the set
  // of files to run inference on. Turning it on stays on the Upload tab
  // (so people can keep browsing/checking files there) — the Infer tab
  // badge below nudges them over once something's selected.
  const [selectMode, setSelectMode] = useState(false);
  useEffect(() => { if (!isBatchRunning) setSelectMode(false); }, [isBatchRunning]);
  useEffect(() => { if (!selectMode) dispatch(clearServerSelection()); }, [selectMode]); // eslint-disable-line
  useEffect(() => { return () => { dispatch(clearServerSelection()); }; }, []); // eslint-disable-line

  const enterInferSelection = useCallback(() => setSelectMode(true), []);
  const exitInferSelection = useCallback(() => setSelectMode(false), []);

  const [fileResult, setFileResult] = useState<FileResult | null>(null);
  const [fileLoading, setFileLoading] = useState(false);
  const [activeFileId, setActiveFileId] = useState<number | null>(null);

  const fetchFileData = useCallback(async (fileId: number) => {
    setFileLoading(true);
    try {
      const res = await api.get(`/files/${fileId}`);
      const d = (res.data as any)?.data ?? {};
      setFileResult(prev => ({
        fileId,
        fileName: d.original_name ?? prev?.fileName ?? String(fileId),
        insertedAt: d.inserted_at ?? prev?.insertedAt ?? '',
        summary: d.summary ?? '',
        keywords: d.keywords ?? [],
        faq: d.faq ?? '[]',
        shortAnswer: d.short_answer ?? '[]',
        trueFalse: d.true_false ?? '[]',
        // NOTE: backend key is genuinely "timstamped_summary" (missing an "e") — match it exactly.
        timestampedSummary: d.timstamped_summary ?? '[]',
        keywordInsights: d.keyword_insights ?? null,
      }));
    } catch { setFileResult(null); } finally { setFileLoading(false); }
  }, []);

  // Clicking a file (from the Upload tab's list, or the Workspace tab's
  // own picker) loads its results and jumps straight to the Workspace tab.
  const handleFileClick = useCallback(async (fileId: number) => {
    setActiveTab('results');
    if (fileId === activeFileId) return;
    setActiveFileId(fileId);
    setFileResult(null);
    await fetchFileData(fileId);
  }, [activeFileId, fetchFileData]);

  const handleDeleteComplete = useCallback((deletedIds: number[], all?: boolean) => {
    if (all || (activeFileId !== null && deletedIds.includes(activeFileId))) {
      setActiveFileId(null);
      setFileResult(null);
    }
  }, [activeFileId]);

  const prevBatchRunning = useRef(false);
  useEffect(() => {
    const justFinished = prevBatchRunning.current && !isBatchRunning;
    prevBatchRunning.current = isBatchRunning;
    if (justFinished && activeFileId !== null) fetchFileData(activeFileId);
  }, [isBatchRunning]); // eslint-disable-line

  // ── Tab bar badges ──
  const inferBusyCount = serverFilesData.queued.length + serverFilesData.running.length;

  const tabs: { id: TabId; label: string }[] = [
    { id: 'upload', label: t('uploadInfer.tabs.upload') },
    { id: 'infer', label: t('uploadInfer.tabs.infer') },
    { id: 'results', label: t('uploadInfer.tabs.results') },
  ];

  return (
    <div className={styles.page}>
      <div className={styles.ph}>
        <div className={styles.phRow}>
          <div>
            <div className={styles.phTitle}>{t('uploadInfer.pageTitle')}</div>
          </div>
        </div>
      </div>

      {/* ── Tab bar — three connected step buttons, every one always clickable ── */}
      <div className={styles.tabbarWrap}>
        <div className={styles.tabbar} role="tablist">
          {tabs.map((tab, i) => (
            <React.Fragment key={tab.id}>
              <button
                type="button"
                role="tab"
                aria-selected={activeTab === tab.id}
                className={`${styles.tabBtn} ${activeTab === tab.id ? styles.tabBtnActive : ''}`}
                onClick={() => setActiveTab(tab.id)}
              >
                <span className={styles.tabLabel}>{tab.label}</span>

                {tab.id === 'upload' && filesTotal > 0 && (
                  <span className={styles.tabBadge}>
                    {t('uploadInfer.tabs.fileCount', { count: filesTotal })}
                  </span>
                )}

                {tab.id === 'infer' && isBatchRunning && (
                  <span className={styles.tabBadge}>
                    <span className={styles.tabLiveDot} />
                    {inferBusyCount > 0
                      ? t('uploadInfer.tabs.runningCount', { count: inferBusyCount })
                      : t('uploadInfer.tabs.running')}
                  </span>
                )}
                {tab.id === 'infer' && !isBatchRunning && selectMode && selectedServerIds.length > 0 && (
                  <span className={styles.tabBadge}>
                    {t('uploadInfer.tabs.selectedCount', { count: selectedServerIds.length })}
                  </span>
                )}

                {tab.id === 'results' && activeFileId === null && (
                  <span className={styles.tabHint}>{t('uploadInfer.tabs.resultsHint')}</span>
                )}
              </button>

              {i < tabs.length - 1 && (
                <div className={styles.tabConnector} aria-hidden="true">
                  <span className={styles.tabConnectorArrowWrap}>
                    <svg className={styles.tabConnectorArrow} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.8" strokeLinecap="round" strokeLinejoin="round">
                      <path d="M5 3l5 5-5 5" />
                    </svg>
                  </span>
                </div>
              )}
            </React.Fragment>
          ))}
        </div>
      </div>

      <div className={styles.upbody}>
        <div className={styles.tabPane} style={{ display: activeTab === 'upload' ? 'flex' : 'none' }}>
          <FilePanel
            selectMode={selectMode}
            onEnterSelectMode={enterInferSelection}
            onExitSelectMode={exitInferSelection}
            onFileClick={handleFileClick}
            onDeleteComplete={handleDeleteComplete}
            activeFileId={activeFileId}
            onGoToInfer={() => setActiveTab('infer')}
          />
        </div>

        <div className={styles.tabPane} style={{ display: activeTab === 'infer' ? 'flex' : 'none' }}>
          <FileSidebar mode="select" active={activeTab === 'infer'} />
          <InferencePanel />
        </div>

        <div className={styles.tabPane} style={{ display: activeTab === 'results' ? 'flex' : 'none' }}>
          <FileSidebar mode="view" active={activeTab === 'results'} activeFileId={activeFileId} onFileClick={handleFileClick} />
          <WorkspacePanel
            step2Visible={false}
            fileResult={fileResult}
            fileLoading={fileLoading}
            activeFileId={activeFileId}
            onResultUpdate={patch => setFileResult(prev => prev ? { ...prev, ...patch } : prev)}
          />
        </div>
      </div>
    </div>
  );
};

export default UploadInfer;











// ═══════════════════════════════════════════════
// FilePanel.module.scss
// Content Analytics · Upload panel — two sections
// ═══════════════════════════════════════════════
@use '../../styles/mixins' as m;

// ── Outer panel shell ─────────────────────────
.panel {
  width: 100%;
  flex: 1;
  border-right: none;
  display: flex;
  flex-direction: row;
  align-items: stretch;
  overflow: hidden;
  background: var(--bg0);
  position: relative;

  &::after {
    content: '';
    position: absolute;
    top: 0;
    right: 0;
    bottom: 0;
    width: 1px;
    background: linear-gradient(180deg,
        rgba(139, 92, 246, 0.0) 0%,
        rgba(139, 92, 246, 0.6) 20%,
        rgba(56, 196, 186, 0.7) 55%,
        rgba(240, 160, 48, 0.6) 85%,
        rgba(240, 160, 48, 0.0) 100%);
    pointer-events: none;
    z-index: 1;
  }
}

// ══════════════════════════════════════
// SECTION 1 — Step header + upload zone
// ══════════════════════════════════════
.step1 {
  width: 400px;
  flex-shrink: 0;
  border-bottom: none;
  border-right: 1px solid var(--bdr);
  background: var(--bg0);
  display: flex;
  flex-direction: column;
  overflow: hidden;
  position: relative;
  padding: 16px;
  transition: width 0.25s cubic-bezier(0.4, 0, 0.2, 1), padding 0.25s, border-color 0.25s;
}

.step1Collapsed {
  width: 0;
  padding: 16px 0;
  border-right-color: transparent;
}

// ── The upload zone itself, as a distinct card sitting in the sidebar ──
.step1Card {
  flex: 1;
  min-height: 0;
  display: flex;
  flex-direction: column;
  background: var(--bg1);
  border: 1px solid var(--bdr);
  border-radius: var(--rl);
  overflow: hidden;
  box-shadow: var(--shadow-sm);
}

// Reopen affordance shown in the file-list header once the upload
// column has been collapsed to width: 0 (and its own header bar with it).
.reopenUploadBtn {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  margin-right: 8px;
  padding: 3px 9px;
  border-radius: 99px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  font-size: 12px;
  @include m.mono;
  cursor: pointer;
  transition: all 0.12s;

  svg { width: 10px; height: 10px; }

  &:hover {
    background: var(--bg3);
    color: var(--t1);
    border-color: var(--bdr3);
  }
}

// ── Step-1 header bar (original design) ──────
.step1Bar {
  padding: 10px 14px;
  display: flex;
  align-items: center;
  gap: 9px;
  border-bottom: none;
  background: var(--bg1);
  flex-shrink: 0;
  white-space: nowrap;
  position: relative;

  &::after {
    content: '';
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    height: 1px;
    background: linear-gradient(90deg,
        rgba(139, 92, 246, 0.0) 0%,
        rgba(139, 92, 246, 0.6) 20%,
        rgba(56, 196, 186, 0.7) 50%,
        rgba(240, 160, 48, 0.6) 80%,
        rgba(240, 160, 48, 0.0) 100%);
    pointer-events: none;
  }
}

.slbl {
  font-size: 12px;
  font-weight: 600;
  color: var(--t2);
  text-transform: uppercase;
  letter-spacing: 0.1em;
  @include m.mono;
}

// Collapse toggle button — matches original
.collapseBtn {
  display: flex;
  align-items: center;
  gap: 5px;
  margin-left: auto;
  padding: 3px 8px;
  border-radius: 99px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  font-size: 12px;
  @include m.mono;
  cursor: pointer;
  transition: all 0.12s;
  user-select: none;
  flex-shrink: 0;

  svg {
    width: 10px;
    height: 10px;
  }

  &:hover {
    background: var(--bg3);
    color: var(--t1);
    border-color: var(--bdr3);
  }
}

// ── Collapsible content area — fills the rest of the column, scrolls
// internally so a long browsed/uploading list doesn't blow out the height ──
.step1Content {
  flex: 1;
  min-height: 0;
  overflow-y: auto;
  @include m.scrollbar;
}

// ── Drop zone (original style) ────────────────
.dropzone {
  border: 1.5px dashed var(--bdr2);
  border-radius: var(--rxl);
  padding: 24px 20px;
  margin: 12px 14px;
  text-align: center;
  background: var(--bg1);
  cursor: pointer;
  transition: all 0.18s;
  position: relative;
  overflow: hidden;

  &::before {
    content: '';
    position: absolute;
    inset: 0;
    background: radial-gradient(ellipse at 50% 0%, rgba(139, 92, 246, 0.04), transparent 65%);
    pointer-events: none;
  }

  &:hover,
  &.dragOver {
    border-color: var(--blue);
    background: var(--bg2);
  }

  &.dragOver {
    background: var(--blue-dim);
  }
}

.dzIc {
  width: 38px;
  height: 38px;
  border-radius: 10px;
  background: var(--blue-dim);
  border: 1px solid var(--blue-bdr);
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 0 auto 11px;

  svg {
    width: 18px;
    height: 18px;
  }
}

.dzTitle {
  font-size: 14px;
  font-weight: 500;
  color: var(--t0);
  margin-bottom: 4px;
}

.dzSub {
  font-size: 13px;
  color: var(--t2);
  @include m.mono;
}

.chips {
  display: flex;
  justify-content: center;
  gap: 6px;
  flex-wrap: wrap;
  margin-top: 10px;
}

.chip {
  font-size: 12px;
  padding: 2px 8px;
  border-radius: 99px;
  background: var(--bg3);
  border: 1px solid var(--bdr);
  color: var(--t2);
  @include m.mono;
}

.dzActions {
  display: flex;
  gap: 8px;
  justify-content: center;
  margin-top: 11px;
  position: relative;
  z-index: 1;
}

// ── Preview / uploading ───────────────────────
.previewWrap {
  display: flex;
  flex-direction: column;
  animation: fadeSlide 0.18s ease;
}

.previewList {
  max-height: 240px;
  overflow-y: auto;
  padding: 8px 10px;
  display: flex;
  flex-direction: column;
  gap: 5px;
  @include m.scrollbar;
}

// Browsed file card
.fileCard {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 11px;
  background: var(--bg2);
  border: 1px solid var(--bdr);
  border-radius: var(--rxl);
  transition: border-color 0.15s, background 0.15s, box-shadow 0.15s;
  position: relative;
  overflow: hidden;

  // Subtle shimmer background
  &::before {
    content: '';
    position: absolute;
    inset: 0;
    background: radial-gradient(ellipse at 0% 50%, rgba(139, 92, 246, 0.04), transparent 70%);
    pointer-events: none;
  }

  // Left accent bar
  &::after {
    content: '';
    position: absolute;
    left: 0;
    top: 8px;
    bottom: 8px;
    width: 3px;
    border-radius: 0 3px 3px 0;
    background: var(--bdr2);
    transition: background 0.2s;
  }

  &.uploading {
    border-color: rgba(139, 92, 246, 0.35);
    background: rgba(139, 92, 246, 0.05);
    box-shadow: 0 0 0 1px rgba(139, 92, 246, 0.12) inset;

    &::after {
      background: linear-gradient(180deg, var(--blue), #a78bfa);
    }

    &::before {
      background: radial-gradient(ellipse at 0% 50%, rgba(139, 92, 246, 0.08), transparent 70%);
    }
  }

  &.success {
    border-color: var(--green-bdr);
    background: var(--green-dim);

    &::after {
      background: var(--green);
    }

    &::before {
      background: radial-gradient(ellipse at 0% 50%, rgba(52, 211, 153, 0.08), transparent 70%);
    }
  }

  &.failed {
    border-color: var(--red-bdr);
    background: var(--red-dim);

    &::after {
      background: var(--red);
    }

    &::before {
      background: radial-gradient(ellipse at 0% 50%, rgba(239, 68, 68, 0.08), transparent 70%);
    }
  }
}

// Extension badge (shared between browsed + uploaded cards)
.extBadge {
  width: 34px;
  height: 34px;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 10px;
  font-weight: 800;
  @include m.mono;
  flex-shrink: 0;
  letter-spacing: 0.03em;
  position: relative;
  z-index: 1;

  &.vtt {
    background: linear-gradient(135deg, rgba(91, 164, 239, 0.18) 0%, rgba(139, 92, 246, 0.14) 100%);
    color: var(--blue);
    border: 1px solid rgba(91, 164, 239, 0.3);
    box-shadow: 0 1px 4px rgba(91, 164, 239, 0.12);
  }

  &.srt {
    background: linear-gradient(135deg, rgba(52, 211, 153, 0.18) 0%, rgba(56, 196, 186, 0.14) 100%);
    color: var(--green);
    border: 1px solid rgba(52, 211, 153, 0.3);
    box-shadow: 0 1px 4px rgba(52, 211, 153, 0.12);
  }
}

.fileInfo {
  flex: 1;
  min-width: 0;
}

.fileNameRow {
  display: flex;
  align-items: center;
  gap: 5px;
  min-width: 0;
}

.fileName {
  font-size: 13px;
  font-weight: 500;
  color: var(--t0);
  @include m.truncate;
  margin-bottom: 1px;
  flex: 1;
  min-width: 0;
}

.fileId {
  font-size: 10px;
  color: var(--t2);
  @include m.mono;
  flex-shrink: 0;
  opacity: 0.65;
}

// ── File ID badge (shown before ext badge in card) ──
.fileIdBadge {
  font-size: 10px;
  font-weight: 700;
  font-family: var(--font-mono);
  color: #a78bfa;
  background: rgba(167, 139, 250, 0.12);
  border: 1px solid rgba(167, 139, 250, 0.3);
  border-radius: 4px;
  padding: 1px 5px;
  flex-shrink: 0;
  white-space: nowrap;
  letter-spacing: 0.02em;
}

.fileMeta {
  font-size: 12px;
  color: var(--t2);
  @include m.mono;
  display: flex;
  align-items: center;
  gap: 6px;
  margin-top: 2px;
  flex-wrap: wrap;
}

.fileSizeChip {
  font-size: 11px;
  padding: 1px 6px;
  border-radius: 99px;
  background: var(--bg3);
  border: 1px solid var(--bdr);
  color: var(--t2);
}

.fileStatusText {
  font-size: 11px;
  color: var(--t2);
  opacity: 0.7;
}

.fileStatusTextSuccess {
  font-size: 11px;
  color: var(--green);
  font-weight: 600;
}

.fileError {
  color: var(--red);
}

// Status icon (upload progress)
.statusIc {
  width: 18px;
  height: 18px;
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;

  svg {
    width: 15px;
    height: 15px;
  }

  &.pending {
    color: var(--t2);
  }

  &.uploading {
    color: var(--blue);
    animation: spin 0.9s linear infinite;
  }

  &.success {
    color: var(--green);
  }

  &.failed {
    color: var(--red);
  }
}

.removeBtn {
  width: 22px;
  height: 22px;
  border-radius: 5px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  padding: 0;
  transition: all 0.12s;

  svg {
    width: 10px;
    height: 10px;
  }

  &:hover {
    background: var(--red-dim);
    border-color: var(--red-bdr);
    color: var(--red);
  }
}

// Action bar (Upload / Cancel)
.actionBar {
  display: flex;
  gap: 7px;
  padding: 10px;
  border-top: none;
  background: var(--bg1);
  position: relative;

  &::before {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    height: 1px;
    background: linear-gradient(90deg,
        rgba(139, 92, 246, 0.0) 0%,
        rgba(139, 92, 246, 0.6) 20%,
        rgba(56, 196, 186, 0.7) 50%,
        rgba(240, 160, 48, 0.6) 80%,
        rgba(240, 160, 48, 0.0) 100%);
    pointer-events: none;
  }
}

// Upload progress bar
.uploadSummary {
  padding: 8px 10px 10px;
  border-top: none;
  background: var(--bg1);
  position: relative;

  &::before {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    height: 1px;
    background: linear-gradient(90deg,
        rgba(139, 92, 246, 0.0) 0%,
        rgba(139, 92, 246, 0.6) 20%,
        rgba(56, 196, 186, 0.7) 50%,
        rgba(240, 160, 48, 0.6) 80%,
        rgba(240, 160, 48, 0.0) 100%);
    pointer-events: none;
  }
}

.uploadProgressBar {
  height: 3px;
  background: var(--bg3);
  border-radius: 99px;
  overflow: hidden;
  margin-bottom: 6px;
}

.uploadProgressFill {
  height: 100%;
  background: linear-gradient(90deg, var(--blue), #a78bfa);
  border-radius: 99px;
  transition: width 0.4s ease;
}

.uploadProgressLabel {
  font-size: 12px;
  color: var(--t2);
  @include m.mono;
}

.failCount {
  color: var(--red);
}

// ══════════════════════════════════════
// SECTION 2 — Uploaded files list
// ══════════════════════════════════════
.section2 {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  background: var(--bg0);
}

// Section 2 header — stacked: title row + action row/grid below
.section2Header {
  display: flex;
  flex-direction: column;
  width: 100%;
  box-sizing: border-box;
  padding: 0;
  border-bottom: none;
  background: var(--bg1);
  flex-shrink: 0;
  position: relative;

  &::after {
    content: '';
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    height: 1px;
    background: linear-gradient(90deg,
        rgba(139, 92, 246, 0.0) 0%,
        rgba(139, 92, 246, 0.6) 20%,
        rgba(56, 196, 186, 0.7) 50%,
        rgba(240, 160, 48, 0.6) 80%,
        rgba(240, 160, 48, 0.0) 100%);
    pointer-events: none;
  }
}

// Title row — checkbox + title + optional search icon
.section2TitleRow {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 14px 8px;
  width: 100%;
  box-sizing: border-box;
}

.section2Title {
  font-size: 12px;
  font-weight: 600;
  color: var(--t2);
  text-transform: uppercase;
  letter-spacing: 0.1em;
  @include m.mono;
  display: flex;
  align-items: center;
  gap: 7px;
}

.filesCount {
  font-size: 10px;
  font-weight: 700;
  color: var(--blue);
  background: var(--blue-dim);
  border: 1px solid var(--blue-bdr);
  padding: 1px 6px;
  border-radius: 99px;
}

// Cross-page selection total — shown next to filesCount while in
// select/delete/export mode so it's clear selections persist across pages.
.selectedTotalHint {
  font-size: 10px;
  font-weight: 700;
  color: var(--green);
  background: var(--green-dim);
  border: 1px solid var(--green-bdr);
  padding: 1px 6px;
  border-radius: 99px;
  margin-left: 4px;
}



// ── Delete icon button (used inside mode-bar) ─────────────────
.deleteIconBtn {
  width: 28px;
  height: 28px;
  border-radius: 7px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  flex-shrink: 0;
  transition: all 0.15s;

  svg {
    width: 14px;
    height: 14px;
  }

  &:hover:not(:disabled) {
    background: rgba(239, 68, 68, 0.1);
    border-color: rgba(239, 68, 68, 0.35);
    color: #ef4444;
  }

  &:disabled {
    opacity: 0.3;
    cursor: default;
  }
}

// Confirm-delete state — always red, solid
.deleteIconBtnConfirm {
  border-color: rgba(239, 68, 68, 0.45);
  background: rgba(239, 68, 68, 0.12);
  color: #ef4444;

  &:hover:not(:disabled) {
    background: rgba(239, 68, 68, 0.22);
    border-color: rgba(239, 68, 68, 0.7);
    box-shadow: 0 0 8px rgba(239, 68, 68, 0.25);
  }
}

.deleteIconBtnDisabled {
  opacity: 0.4 !important;
  cursor: default !important;
}

// ── Export icon button (used inside mode-bar) ─────────────────
.exportIconBtn {
  width: 28px;
  height: 28px;
  border-radius: 7px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  flex-shrink: 0;
  transition: all 0.15s;

  svg {
    width: 14px;
    height: 14px;
  }

  &:hover:not(:disabled) {
    background: rgba(78, 200, 122, 0.10);
    border-color: rgba(78, 200, 122, 0.35);
    color: var(--green);
  }

  &:disabled {
    opacity: 0.3;
    cursor: default;
  }
}

// Confirm-export state — always green, solid
.exportIconBtnConfirm {
  border-color: rgba(78, 200, 122, 0.45);
  background: rgba(78, 200, 122, 0.10);
  color: var(--green);

  &:hover:not(:disabled) {
    background: rgba(78, 200, 122, 0.20);
    border-color: rgba(78, 200, 122, 0.70);
    box-shadow: 0 0 8px rgba(78, 200, 122, 0.22);
  }
}

.exportIconBtnDisabled {
  opacity: 0.4 !important;
  cursor: default !important;
}

.miniSpinner {
  width: 12px;
  height: 12px;
  border: 1.5px solid rgba(239, 68, 68, 0.3);
  border-top-color: #ef4444;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
  flex-shrink: 0;
}

.miniSpinnerGreen {
  width: 12px;
  height: 12px;
  border: 1.5px solid rgba(78, 200, 122, 0.3);
  border-top-color: #4ec87a;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
  flex-shrink: 0;
}


// ── Cancel icon button (used inside mode-bar) ─────────────────
.cancelIconBtn {
  width: 28px;
  height: 28px;
  border-radius: 7px;
  border: 1px solid rgba(239, 68, 68, 0.35);
  background: rgba(239, 68, 68, 0.08);
  color: #ef4444;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  flex-shrink: 0;
  transition: all 0.15s;

  svg {
    width: 13px;
    height: 13px;
  }

  &:hover {
    background: rgba(239, 68, 68, 0.16);
    border-color: rgba(239, 68, 68, 0.6);
    box-shadow: 0 0 6px rgba(239, 68, 68, 0.2);
  }
}

// Cards get pointer cursor only when in selection mode
.uploadedCardWrapSelectable {
  cursor: pointer;
}

// Date range filter — sits above section2Header
.filterSortRow {
  display: flex;
  flex-direction: column;
  gap: 10px;
  padding: 10px 12px;
  background: var(--bg1);
  flex-shrink: 0;
  position: relative;

  &::before {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    height: 1px;
    background: linear-gradient(90deg,
        rgba(139, 92, 246, 0.0) 0%,
        rgba(139, 92, 246, 0.6) 20%,
        rgba(56, 196, 186, 0.7) 50%,
        rgba(240, 160, 48, 0.6) 80%,
        rgba(240, 160, 48, 0.0) 100%);
    pointer-events: none;
  }

  &::after {
    content: '';
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    height: 1px;
    background: linear-gradient(90deg,
        rgba(139, 92, 246, 0.0) 0%,
        rgba(139, 92, 246, 0.6) 20%,
        rgba(56, 196, 186, 0.7) 50%,
        rgba(240, 160, 48, 0.6) 80%,
        rgba(240, 160, 48, 0.0) 100%);
    pointer-events: none;
  }
}

// Line 1 — date filter on the left, sort chip on the right
.dateSortLine {
  display: flex;
  align-items: center;
  justify-content: space-between;
  flex-wrap: wrap;
  gap: 12px;
  width: 100%;
}

.sortHeader {
  margin-left: auto;
}

// Clear horizontal separator between line 1 (date/sort) and line 2 (actions)
.filterSortHDivider {
  height: 1px;
  width: 100%;
  background: var(--bdr);
  flex-shrink: 0;
}

// Line 2 — action toolbar, full width
.actionsGrid {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 6px;
  width: 100%;
}

.actionsLabel {
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
  margin-right: 2px;

  svg {
    width: 11px;
    height: 11px;
    opacity: 0.6;
  }
}

.actionTile {
  display: inline-flex;
  flex-direction: row;
  align-items: center;
  height: 26px;
  gap: 4px;
  padding: 0 9px;
  border-radius: var(--r);
  border: 1px solid transparent;
  background: var(--bg3);
  color: var(--t2);
  font-size: 12px;
  font-weight: 500;
  font-family: var(--font-ui);
  letter-spacing: 0.01em;
  white-space: nowrap;
  cursor: pointer;
  transition: all 0.13s;
  user-select: none;

  svg {
    width: 12px;
    height: 12px;
    flex-shrink: 0;
  }

  &:active:not(:disabled) {
    transform: scale(0.97);
  }

  &:disabled {
    opacity: 0.3;
    cursor: not-allowed;
  }
}

.actionTileInfer {
  background: var(--blue-dim);
  color: var(--blue);
  border-color: var(--blue-bdr);

  &:hover:not(:disabled) { background: var(--blue-dim); border-color: var(--blue); }
}

.actionTileDictionary {
  background: var(--violet-dim);
  color: var(--violet);
  border-color: var(--violet-dim);

  &:hover:not(:disabled) { background: var(--violet-dim); border-color: var(--violet); }
}

.actionTileTemplate {
  background: var(--teal-dim);
  color: var(--teal);
  border-color: var(--teal-bdr);

  &:hover:not(:disabled) { background: var(--teal-dim); border-color: var(--teal); }
}

.actionTileSearch {
  background: var(--amber-dim);
  color: var(--amber);
  border-color: var(--amber-bdr);

  &:hover:not(:disabled) { background: var(--amber-dim); border-color: var(--amber); }
}

.actionTileExport {
  background: var(--green-dim);
  color: var(--green);
  border-color: var(--green-bdr);

  &:hover:not(:disabled) { background: var(--green-dim); border-color: var(--green); }
}

.actionTileDelete {
  background: var(--red-dim);
  color: var(--red);
  border-color: var(--red-bdr);

  &:hover:not(:disabled) { background: var(--red-dim); border-color: var(--red); }
}

.actionTileActive {
  outline: 2px solid var(--blue-bdr);
  outline-offset: -1px;
}

// Single-line, icon-led date filter — same 32px height as the rest of the row
.dateFilter {
  display: flex;
  align-items: center;
  height: 32px;
  gap: 6px;
  flex-shrink: 0;
}

.dateIcon {
  width: 14px;
  height: 14px;
  color: var(--t2);
  flex-shrink: 0;
}

.dateField {
  display: flex;
  flex-direction: column;
  gap: 3px;
  width: 118px;
  flex-shrink: 0;
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
  width: 115px;
  height: 32px;
  padding: 0 8px;
  background: var(--bg2);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 12.5px;
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
  font-size: 12px;
  color: var(--t2);
  flex-shrink: 0;
}

.applyBtn {
  height: 32px;
  padding: 0 12px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t1);
  font-family: var(--font-ui);
  font-size: 12.5px;
  font-weight: 500;
  cursor: pointer;
  flex-shrink: 0;
  transition: all 0.12s;

  &:hover:not(:disabled) {
    background: var(--bg3);
    border-color: var(--bdr3);
  }

  &:disabled {
    opacity: 0.5;
    cursor: default;
  }
}

// Uploaded file cards
.uploadedBody {
  flex: 1;
  overflow-y: auto;
  padding: 10px;
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(190px, 1fr));
  grid-auto-rows: 200px;
  align-content: start;
  gap: 12px;
  @include m.scrollbar;
}

// listState/errorState (loading, error, empty) span every column so they
// don't get squeezed into a single grid cell
.listState {
  grid-column: 1 / -1;
}

// ── Square file cards ─────────────────────────
.fcard {
  position: relative;
  height: 100%;
  min-width: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 8px;
  padding: 18px 14px 32px;
  border-radius: var(--rl);
  border: 1px solid var(--bdr);
  background: var(--bg1);
  cursor: pointer;
  text-align: center;
  overflow: hidden;
  transition: all 0.12s;
  user-select: none;

  &:hover {
    background: var(--bg2);
    border-color: var(--bdr3);
  }
}

.fcardActive,
.fcardActiveView {
  background: rgba(91, 164, 239, 0.06);
  border-color: var(--blue-bdr);
}

.fcardActiveDelete {
  background: var(--red-dim);
  border-color: var(--red-bdr);
}

.fcardActiveExport {
  background: var(--green-dim);
  border-color: var(--green-bdr);
}

.fcardCheck {
  position: absolute;
  top: 10px;
  left: 10px;
  z-index: 1;
}

.fcardIcon {
  width: 40px;
  height: 40px;
  border-radius: 9px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 11px;
  font-weight: 700;
  flex-shrink: 0;

  &.vtt { background: var(--blue-dim); color: var(--blue); }
  &.srt { background: var(--green-dim); color: var(--green); }
}

.fcardName {
  width: 100%;
  max-width: 100%;
  font-size: 12.5px;
  font-weight: 500;
  color: var(--t0);
  line-height: 1.35;
  display: -webkit-box;
  -webkit-line-clamp: 3;
  -webkit-box-orient: vertical;
  overflow: hidden;
  word-break: break-word;
  flex-shrink: 0;
}

.fcardMeta {
  font-size: 10px;
  color: var(--t2);
  flex-shrink: 0;
  @include m.mono;
}

.fcardBadge {
  position: absolute;
  bottom: 11px;
  left: 50%;
  transform: translateX(-50%);
  font-size: 9px;
  padding: 2px 7px;
  max-width: calc(100% - 16px);
  overflow: hidden;
  text-overflow: ellipsis;
}

// ── Uploaded card wrap — carries the static green gradient border ──
// ── Uploaded file card — mirrors HistoryPanel .hitm ──────────────
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

  // Normal-mode file view highlight — distinct purple/amber accent
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

  // Delete mode selected — red accent
  &.activeDelete {
    background: rgba(239, 68, 68, 0.05);
    border-color: rgba(239, 68, 68, 0.3);

    &::before {
      content: '';
      position: absolute;
      left: 0;
      top: 6px;
      bottom: 6px;
      width: 3px;
      border-radius: 0 3px 3px 0;
      background: linear-gradient(180deg, #ef4444, #f97316);
    }
  }

  // Export mode selected — green accent
  &.activeExport {
    background: rgba(78, 200, 122, 0.05);
    border-color: rgba(78, 200, 122, 0.28);

    &::before {
      content: '';
      position: absolute;
      left: 0;
      top: 6px;
      bottom: 6px;
      width: 3px;
      border-radius: 0 3px 3px 0;
      background: linear-gradient(180deg, #4ec87a, #38c4ba);
    }
  }
}

.hitmSelectable {
  cursor: pointer;
}

// ── Ext icon ──
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

  &.vtt {
    background: var(--blue-dim);
    color: var(--blue);
  }

  &.srt {
    background: var(--green-dim);
    color: var(--green);
  }
}

// ── File info ──
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

// Empty / loading / error state
.listState {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 10px;
  padding: 32px 16px;
  font-size: 13px;
  color: var(--t2);
  @include m.mono;
  text-align: center;

  svg {
    width: 18px;
    height: 18px;
    opacity: 0.5;
  }
}

.errorState {
  color: var(--red);

  svg {
    opacity: 0.7;
  }
}

.spinner {
  width: 18px;
  height: 18px;
  border: 2px solid var(--bdr2);
  border-top-color: var(--blue);
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

// ── Search icon button ──────────────────────────
.searchIconBtn {
  width: 28px;
  height: 28px;
  border-radius: 7px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  flex-shrink: 0;
  transition: all 0.15s;

  svg {
    width: 13px;
    height: 13px;
  }

  &:hover {
    background: var(--bg3);
    border-color: var(--bdr3);
    color: var(--t0);
  }
}

.searchIconBtnActive {
  border-color: var(--blue-bdr);
  background: var(--blue-dim);
  color: var(--blue);

  &:hover {
    background: var(--blue-dim);
    border-color: var(--blue);
    color: var(--blue);
  }
}

// ── Search bar (slides in below sortHeader) ──────
.searchBar {
  display: grid;
  grid-template-rows: 0fr;
  opacity: 0;
  transition:
    grid-template-rows 0.22s cubic-bezier(0.4, 0, 0.2, 1),
    opacity 0.18s ease;
  overflow: hidden;
  padding: 0 10px;
}

.searchBarOpen {
  grid-template-rows: 1fr;
  opacity: 1;
  padding: 6px 10px 4px;
}

// Flex row: input box + close button
.searchBarRow {
  min-height: 0;
  display: flex;
  align-items: center;
  gap: 6px;
}

.searchInner {
  flex: 1;
  min-width: 0;
  min-height: 0;
  display: flex;
  align-items: center;
  gap: 6px;
  background: var(--bg2);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  padding: 0 8px;
  transition: border-color 0.15s, box-shadow 0.15s;

  &:focus-within {
    border-color: var(--blue);
    box-shadow: 0 0 0 2px var(--blue-dim);
  }
}

.searchBarIcon {
  width: 12px;
  height: 12px;
  flex-shrink: 0;
  color: var(--t2);
  opacity: 0.6;
}

.searchInput {
  flex: 1;
  min-width: 0;
  padding: 6px 0;
  background: transparent;
  border: none;
  outline: none;
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 13px;

  &::placeholder {
    color: var(--t2);
    opacity: 0.55;
  }
}

.searchCloseBtn {
  width: 20px;
  height: 20px;
  border-radius: 5px;
  border: 1px solid rgba(239, 68, 68, 0.3);
  background: rgba(239, 68, 68, 0.08);
  color: #ef4444;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  flex-shrink: 0;
  transition: all 0.12s;

  svg {
    width: 8px;
    height: 8px;
  }

  &:hover {
    background: rgba(239, 68, 68, 0.18);
    border-color: rgba(239, 68, 68, 0.6);
    box-shadow: 0 0 6px rgba(239, 68, 68, 0.2);
  }
}

.searchCount {
  font-size: 11px;
  color: var(--t2);
  font-family: var(--font-mono);
  white-space: nowrap;
  flex-shrink: 0;
}


.badge {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-size: 12px;
  padding: 2px 8px;
  border-radius: 99px;
  font-weight: 500;
  border: 1px solid transparent;
  white-space: nowrap;
  @include m.mono;
}

.bReady {
  background: var(--green-dim);
  color: var(--green);
  border-color: var(--green-bdr);
}

// Inferenced — green (file has completed inference)
.bInferred {
  background: var(--green-dim);
  color: var(--green);
  border-color: var(--green-bdr);
}

// Not Inferenced — amber (file uploaded but not yet processed)
.bNotInferred {
  background: rgba(240, 160, 48, 0.1);
  color: var(--amber);
  border-color: rgba(240, 160, 48, 0.3);
}

.bSelected {
  background: var(--blue-dim);
  color: var(--blue);
  border-color: var(--blue-bdr);
  font-weight: 600;
}

.bDelete {
  background: rgba(239, 68, 68, 0.1);
  color: #ef4444;
  border-color: rgba(239, 68, 68, 0.3);
  font-weight: 600;
}

.bExport {
  background: var(--green-dim);
  color: var(--green);
  border-color: var(--green-bdr);
  font-weight: 600;
  gap: 4px;
}

.bInfo {
  background: var(--blue-dim);
  color: var(--blue);
  border-color: var(--blue-bdr);
}

// ── Buttons ───────────────────────────────────
.btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 5px;
  padding: 6px 13px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t1);
  font-family: var(--font-ui);
  font-size: 13px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.12s;
  white-space: nowrap;
  user-select: none;

  svg {
    width: 11px;
    height: 11px;
  }

  &:hover {
    background: var(--bg3);
    color: var(--t0);
    border-color: var(--bdr3);
  }

  &:disabled {
    opacity: 0.45;
    cursor: default;
  }
}

.btnP {
  background: var(--blue);
  color: #fff;
  border-color: var(--blue);
  font-weight: 600;

  &:hover {
    background: #a78bfa;
    border-color: #a78bfa;
    color: #fff;
  }
}

.btnSm {
  padding: 4px 10px;
  font-size: 13px;
}

.btnFull {
  flex: 1;
}

.btnDanger {
  color: var(--red);
  border-color: var(--red-bdr);

  &:hover {
    background: var(--red-dim);
    border-color: var(--red);
  }

  &:disabled {
    color: var(--t2);
    border-color: var(--bdr2);
    background: transparent;
    opacity: 0.4;
    cursor: not-allowed;
    pointer-events: none;

    &:hover {
      background: transparent;
      border-color: var(--bdr2);
    }
  }
}

// ── Animations ───────────────────────────────
@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

@keyframes fadeSlide {
  from {
    opacity: 0;
    transform: translateY(4px);
  }

  to {
    opacity: 1;
    transform: translateY(0);
  }
}

// ── Checkbox component ──────────────────────────
.cb {
  width: 14px;
  height: 14px;
  border: 1.5px solid var(--cb-bdr);
  border-radius: 4px;
  flex-shrink: 0;
  cursor: pointer;
  transition: all 0.12s;
  position: relative;
  outline: none;

  &:hover:not(.cbDisabled) {
    border-color: var(--blue);
  }

  &:focus-visible {
    box-shadow: 0 0 0 2px var(--blue-dim);
  }

  &.cbChecked {
    background: var(--blue);
    border-color: var(--blue);

    &::after {
      content: '';
      position: absolute;
      left: 2px;
      top: 5px;
      width: 7px;
      height: 4px;
      border-left: 1.5px solid #fff;
      border-bottom: 1.5px solid #fff;
      transform: rotate(-45deg) translate(0, -1px);
    }
  }

  &.cbIndet {
    background: var(--blue);
    border-color: var(--blue);

    &::after {
      content: '';
      position: absolute;
      left: 2px;
      top: 5px;
      width: 8px;
      height: 1.5px;
      background: #fff;
    }
  }

  &.cbDisabled {
    opacity: 0.35;
    cursor: not-allowed;
    pointer-events: none;
  }
}

// Grouped action buttons — used inside mode-bar (title row)
.headerActions {
  display: flex;
  align-items: center;
  gap: 4px;
  margin-left: auto;
  flex-shrink: 0;
}

// ── Mode action row: search LEFT, confirm+cancel RIGHT ────────
.modeActionsRow {
  display: flex;
  align-items: center;
  box-sizing: border-box;
  width: 100%;
  padding: 0 10px 8px;
  min-width: 0;
}

.modeActionsRight {
  display: flex;
  align-items: center;
  gap: 4px;
  margin-left: auto;
  flex-shrink: 0;
}

.modeActionsLeft {
  display: flex;
  align-items: center;
  gap: 4px;
  min-width: 0;
  flex-wrap: wrap;
}

// Base tile — icon + label side by side
.modeTile {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 4px 9px;
  border-radius: 6px;
  border: 1px solid var(--bdr);
  background: var(--bg3);
  color: var(--t2);
  font-size: 11px;
  font-weight: 500;
  font-family: var(--font-ui);
  cursor: pointer;
  transition: all 0.13s;
  user-select: none;
  white-space: nowrap;

  svg {
    width: 12px;
    height: 12px;
    flex-shrink: 0;
  }

  &:active:not(:disabled) {
    transform: scale(0.97);
  }

  &:disabled {
    opacity: 0.3;
    cursor: not-allowed;
  }
}

// Search tile — amber tint (left side)
.modeTileSearch {
  color: var(--amber);
  border-color: rgba(240, 160, 48, 0.25);
  background: rgba(240, 160, 48, 0.08);

  &:hover {
    background: rgba(240, 160, 48, 0.16);
    border-color: rgba(240, 160, 48, 0.45);
  }
}

.modeTileSearchActive {
  background: rgba(240, 160, 48, 0.15);
  border-color: rgba(240, 160, 48, 0.45);
}

// Confirm export — green
.modeTileConfirmExport {
  color: var(--green);
  border-color: var(--green-bdr);
  background: var(--green-dim);

  &:hover:not(:disabled) {
    background: var(--green);
    border-color: var(--green);
    color: #fff;
  }
}

// Confirm delete — red
.modeTileConfirmDelete {
  color: #ef4444;
  border-color: rgba(239, 68, 68, 0.25);
  background: rgba(239, 68, 68, 0.08);

  &:hover:not(:disabled) {
    background: rgba(239, 68, 68, 0.18);
    border-color: rgba(239, 68, 68, 0.5);
  }
}

// Cancel
.modeTileCancel {
  color: var(--t2);
  border-color: var(--bdr2);
  background: transparent;

  &:hover {
    background: var(--bg2);
    border-color: var(--bdr3);
    color: var(--t0);
  }
}

// Disabled state
.modeTileDisabled {
  opacity: 0.35 !important;
  cursor: not-allowed !important;
}

// "Export all files" trigger — mirrors modeTileDeleteAll but green/non-destructive
.modeTileExportAll {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 4px 9px;
  border-radius: 6px;
  border: 1px solid var(--green-bdr);
  background: var(--green-dim);
  color: var(--green);
  font-size: 11px;
  font-weight: 600;
  font-family: var(--font-ui);
  cursor: pointer;
  transition: all 0.13s;
  user-select: none;
  white-space: nowrap;

  svg {
    width: 12px;
    height: 12px;
    flex-shrink: 0;
  }

  &:hover:not(:disabled) {
    background: var(--green);
    border-color: var(--green);
    color: #fff;
  }

  &:disabled {
    opacity: 0.35;
    cursor: not-allowed;
  }
}

.modeActionsError {
  font-size: 11px;
  color: #ef4444;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 160px;
}

// "Delete all files" trigger — deliberately louder/more solid than the
// per-row delete tile since it's an account-wide destructive action.
.modeTileDeleteAll {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 4px 9px;
  border-radius: 6px;
  border: 1px solid rgba(239, 68, 68, 0.5);
  background: rgba(239, 68, 68, 0.14);
  color: #ef4444;
  font-size: 11px;
  font-weight: 600;
  font-family: var(--font-ui);
  cursor: pointer;
  transition: all 0.13s;
  user-select: none;
  white-space: nowrap;

  svg {
    width: 12px;
    height: 12px;
    flex-shrink: 0;
  }

  &:hover:not(:disabled) {
    background: #ef4444;
    border-color: #ef4444;
    color: #fff;
  }

  &:disabled {
    opacity: 0.3;
    cursor: not-allowed;
  }
}

// ── Delete-ALL confirmation (danger zone) ───────────────────────────
.dangerOverlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.55);
  backdrop-filter: blur(2px);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 200;
  padding: 16px;
}

.dangerModal {
  width: 100%;
  max-width: 360px;
  background: var(--bg1);
  border: 1px solid rgba(239, 68, 68, 0.4);
  border-radius: 10px;
  padding: 20px;
  box-shadow: 0 12px 40px rgba(0, 0, 0, 0.4);
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  gap: 4px;
}

.dangerIcon {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background: rgba(239, 68, 68, 0.14);
  color: #ef4444;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 6px;

  svg {
    width: 22px;
    height: 22px;
  }
}

.dangerTitle {
  font-size: 15px;
  font-weight: 700;
  color: var(--t0);
  font-family: var(--font-ui);
}

.dangerBody {
  font-size: 13px;
  color: var(--t2);
  line-height: 1.5;
  margin-bottom: 8px;
}

.dangerLabel {
  align-self: flex-start;
  font-size: 12px;
  color: var(--t1);
  margin-top: 4px;
}

.dangerInput {
  width: 100%;
  padding: 7px 10px;
  background: var(--bg0);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 13px;
  outline: none;
  margin-top: 4px;
  margin-bottom: 6px;
  text-align: center;
  transition: border-color 0.12s;

  &:focus {
    border-color: #ef4444;
    box-shadow: 0 0 0 2px rgba(239, 68, 68, 0.15);
  }

  &:disabled {
    opacity: 0.6;
  }
}

.dangerError {
  font-size: 12px;
  color: #ef4444;
  margin-bottom: 6px;
}

.dangerActions {
  display: flex;
  gap: 8px;
  width: 100%;
  margin-top: 6px;
}

// .uploadedCardSel replaced by .hitm.active

// Uploaded card disabled (batch running)
.uploadedCardDisabled {
  cursor: default !important;
  opacity: 0.65;
  pointer-events: none;
}

// ── Sort header ─────────────────────────────────────
.sortHeader {
  display: flex;
  align-items: center;
  height: 32px;
  background: var(--bg2);
  border: 1px solid var(--bdr);
  border-radius: var(--r);
  padding: 0 6px 0 10px;
  gap: 6px;
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
}

.sortCol {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 4px;
  height: 24px;
  padding: 0 8px;
  background: transparent;
  border: none;
  border-radius: 5px;
  color: var(--t2);
  font-size: 12px;
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

  &:hover {
    background: var(--bg3);
    color: var(--t1);
  }
}

.sortColActive {
  background: var(--blue-dim);
  color: var(--blue);
  font-weight: 600;

  &:hover {
    background: var(--blue-dim);
  }
}

.sortInactive {
  opacity: 0.25;
}

.sortAsc {
  transform: rotate(180deg);
}

// arrow up
.sortDesc {
  transform: rotate(0deg);
}

// arrow down (default path direction)
// ── Small screen overrides (height < 1000px) ─────────────────
@media (max-height: 999px) {

  // Step-1 bar — tighter
  .step1Bar {
    padding: 7px 14px;
  }

  // Dropzone — much more compact
  .dropzone {
    padding: 10px 16px;
    margin: 6px 10px;
  }

  .dzIc {
    width: 28px;
    height: 28px;
    margin-bottom: 6px;

    svg {
      width: 14px;
      height: 14px;
    }
  }

  .dzTitle {
    font-size: 12px;
    margin-bottom: 2px;
  }

  .dzSub {
    font-size: 11px;
  }

  .dzActions {
    margin-top: 7px;
    gap: 6px;
  }

  // Section2 title row
  .section2TitleRow {
    padding: 7px 14px 6px;
  }

  // Mode action row
  .modeActionsRow {
    padding: 0 8px 6px;
  }

  // Date filter — tighter
  .dateFilter {
    padding: 5px 10px 5px;
    gap: 5px;
  }

  .dateInput {
    padding: 3px 6px;
    font-size: 12px;
  }

  // Sort header — tighter
  .sortHeader {
    margin-bottom: 4px;
  }

  .sortCol {
    padding: 4px 4px;
    font-size: 11px;
  }

  .sortHeaderLabel {
    padding: 4px 0;
    padding-right: 6px;
    font-size: 9px;
  }

  // File list — tighter items, ensure it can grow
  .uploadedBody {
    padding: 4px 8px;
    gap: 2px;
  }

  .hitm {
    padding: 5px 8px;
    margin-bottom: 1px;
  }

  .ficon {
    width: 22px;
    height: 22px;
    font-size: 9px;
  }

  .hn {
    font-size: 12px;
  }

  .hm {
    font-size: 11px;
    margin-top: 1px;
  }

  .badge {
    font-size: 10px;
    padding: 1px 6px;
  }
}

// ── Pagination footer (bottom of the uploaded-files list) ──────────
.paginationBar {
  display: flex;
  flex-direction: column;
  gap: 6px;
  padding: 8px 10px;
  border-top: 1px solid var(--bdr2);
  background: var(--bg1);
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
  background: var(--bg0);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 12px;
  outline: none;
  cursor: pointer;
  transition: border-color 0.12s;

  &:focus {
    border-color: var(--blue);
    box-shadow: 0 0 0 2px var(--blue-dim);
  }
}

// Page-number row sits to the right of the page-size selector. It never
// wraps to its own line — if there isn't room for every number button,
// this row scrolls horizontally within itself instead.
.pageNav {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 2px;
  flex-wrap: nowrap;
  flex-shrink: 1;
  min-width: 0;
  overflow-x: auto;
  scrollbar-width: none;
  -ms-overflow-style: none;

  &::-webkit-scrollbar {
    display: none;
  }
}

.pageNavBtn,
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

  svg {
    width: 11px;
    height: 11px;
  }

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

@media (min-width: 1920px) {
  .section1Title {
    font-size: 14px;
  }

  .section2Title {
    font-size: 14px;
  }

  .uploadHint {
    font-size: 13px;
  }

  .uploadZoneText {
    font-size: 14px;
  }

  .sortHeaderLabel {
    font-size: 12px;
  }

  .sortBtn {
    font-size: 13px;
  }

  .fileName {
    font-size: 14px;
  }

  .fileMeta {
    font-size: 12px;
  }

  .fileDate {
    font-size: 12px;
  }

  .badge {
    font-size: 13px;
  }

  .btn {
    font-size: 13px;
  }

  .listState {
    font-size: 14px;
  }

  .searchInput {
    font-size: 14px;
  }

  .searchCount {
    font-size: 12px;
  }

  .dateLabel {
    font-size: 12px;
  }

  .dateInput {
    font-size: 13px;
  }

  .emptyState {
    font-size: 14px;
  }
}
