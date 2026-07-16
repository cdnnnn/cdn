// ═══════════════════════════════════════════════
// pages/UploadInfer/FileSidebar.tsx
// Content Analytics · Shared 400px file sidebar
// Used by both the Infer tab (mode="select") and the Workspace tab
// (mode="view") so the file list + date filter only exist once.
// ═══════════════════════════════════════════════
import React, { useCallback, useMemo, useState } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import {
  setDateFrom, setDateTo,
  serverFilesLoading, serverFilesSuccess, serverFilesFailure,
  toggleServerFileSelection, toggleSelectAllServerFiles,
} from '../../store/uploadSlice';
import api from '../../services/api';
import styles from './FileSidebar.module.scss';

const getExt = (n: string): 'vtt' | 'srt' => n.toLowerCase().endsWith('.srt') ? 'srt' : 'vtt';

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
  activeFileId?: number | null;
  onFileClick?: (fileId: number) => void;
}

const FileSidebar: React.FC<Props> = ({ mode, activeFileId = null, onFileClick }) => {
  const { t } = useTranslation();
  const dispatch = useAppDispatch();
  const {
    serverFiles, filesTotal, serverFilesLoading: filesLoading,
    dateFrom, dateTo, selectedServerIds, isBatchRunning,
    filesPageSize, filesSortBy, filesSortOrder, filesSearch,
  } = useAppSelector(s => s.upload);

  const [query, setQuery] = useState('');
  const [applying, setApplying] = useState(false);

  const filtered = useMemo(() => {
    const q = query.trim().toLowerCase();
    if (!q) return serverFiles;
    return serverFiles.filter(f => f.original_name.toLowerCase().includes(q));
  }, [serverFiles, query]);

  // Same /files/by-date/ call FilePanel uses, kept independent so this
  // sidebar's date filter doesn't have to coordinate with the Upload
  // tab's pagination/sort/search UI — it just refreshes the shared
  // `serverFiles` list everyone reads from.
  const handleApply = useCallback(async () => {
    setApplying(true);
    dispatch(serverFilesLoading());
    try {
      const res = await api.post('/files/by-date/', {
        start_date: dateFrom,
        end_date: dateTo,
        page: 1,
        page_size: filesPageSize,
        sort_by: filesSortBy,
        sort_order: filesSortOrder,
        ...(filesSearch ? { search: filesSearch } : {}),
      });
      const d = (res.data as any)?.data ?? {};
      dispatch(serverFilesSuccess({
        files: d.data ?? [],
        total: d.total ?? 0,
        page: d.page ?? 1,
        pageSize: d.page_size ?? filesPageSize,
        totalPages: d.total_pages ?? 1,
        sortBy: filesSortBy, sortOrder: filesSortOrder, search: filesSearch,
      }));
    } catch (err) {
      dispatch(serverFilesFailure(err instanceof Error ? err.message : 'Failed to load files.'));
    } finally {
      setApplying(false);
    }
  }, [dispatch, dateFrom, dateTo, filesPageSize, filesSortBy, filesSortOrder, filesSearch]);

  const pageSelectedCount = serverFiles.filter(f => selectedServerIds.includes(f.id)).length;
  const allSelected = serverFiles.length > 0 && pageSelectedCount === serverFiles.length;
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
            onChange={e => dispatch(setDateFrom(e.target.value))}
          />
        </div>
        <div className={styles.dateSep}>—</div>
        <div className={styles.dateField}>
          <label className={styles.dateLabel}>{t('uploadInfer.filePanel.dateTo')}</label>
          <input
            type="date" className={styles.dateInput} value={dateTo} min={dateFrom}
            onChange={e => dispatch(setDateTo(e.target.value))}
          />
        </div>
        <button className={styles.applyBtn} onClick={handleApply} disabled={applying}>
          {applying ? <span className={styles.miniSpinner} /> : t('uploadInfer.filePanel.applyDate')}
        </button>
      </div>

      <div className={styles.head}>
        {mode === 'select' && (
          <Checkbox
            checked={allSelected}
            indeterminate={someSelected}
            onChange={() => dispatch(toggleSelectAllServerFiles())}
            disabled={filesTotal === 0 || isBatchRunning}
          />
        )}
        <span className={styles.headTitle}>{t('uploadInfer.workspace.filesTitle')}</span>
        {filesTotal > 0 && <span className={styles.headCount}>{filesTotal}</span>}
        {mode === 'select' && selectedServerIds.length > 0 && (
          <span className={styles.headSelected}>
            {t('uploadInfer.filePanel.selectedTotal', { count: selectedServerIds.length, total: filesTotal })}
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
        {filesLoading && serverFiles.length === 0 && (
          <div className={styles.empty}>{t('uploadInfer.workspace.loadingFiles')}</div>
        )}
        {!filesLoading && serverFiles.length === 0 && (
          <div className={styles.empty}>{t('uploadInfer.workspace.noFilesYet')}</div>
        )}
        {!filesLoading && serverFiles.length > 0 && filtered.length === 0 && (
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



// ── Action grid (Option C — 3×2 icon + label tiles) ──────────
.actionsGrid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 4px;
  padding: 0 10px 8px;
}

.actionTile {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 3px;
  padding: 5px 2px 4px;
  border-radius: 6px;
  border: 1px solid var(--bdr);
  background: var(--bg3);
  color: var(--t2);
  font-size: 9px;
  font-weight: 500;
  font-family: var(--font-ui);
  letter-spacing: 0.01em;
  cursor: pointer;
  transition: all 0.13s;
  user-select: none;

  svg {
    width: 12px;
    height: 12px;
    flex-shrink: 0;
  }

  &:hover:not(:disabled) {
    background: var(--bg2);
    border-color: var(--bdr2);
    color: var(--t0);
  }

  &:active:not(:disabled) {
    transform: scale(0.97);
  }

  &:disabled {
    opacity: 0.3;
    cursor: not-allowed;
  }
}

// Infer — blue tint
.actionTileInfer {
  border-color: var(--blue-bdr);
  background: var(--blue-dim);
  color: var(--blue);
  animation: inferPulse 2.4s ease-in-out infinite;

  &:hover:not(:disabled) {
    background: var(--blue);
    border-color: var(--blue);
    color: #fff;
    animation: none;
  }
}

// Dictionary — violet tint
.actionTileDictionary {
  color: var(--violet);
  border-color: var(--violet-dim);
  background: var(--violet-dim);

  &:hover:not(:disabled) {
    background: var(--violet);
    border-color: var(--violet);
    color: #fff;
  }
}

// Template — teal tint
.actionTileTemplate {
  color: var(--teal);
  border-color: var(--teal-bdr);
  background: var(--teal-dim);

  &:hover:not(:disabled) {
    background: var(--teal);
    border-color: var(--teal);
    color: #fff;
  }
}

// Search — amber tint
.actionTileSearch {
  color: var(--amber);
  border-color: rgba(240, 160, 48, 0.25);
  background: rgba(240, 160, 48, 0.08);

  &:hover:not(:disabled) {
    background: rgba(240, 160, 48, 0.18);
    border-color: rgba(240, 160, 48, 0.45);
  }
}

// Active search
.actionTileActive {
  background: rgba(240, 160, 48, 0.15) !important;
  border-color: rgba(240, 160, 48, 0.45) !important;
}

// Export — green tint
.actionTileExport {
  color: var(--green);
  border-color: var(--green-bdr);
  background: var(--green-dim);

  &:hover:not(:disabled) {
    background: var(--green);
    border-color: var(--green);
    color: #fff;
  }
}

// Delete — red tint
.actionTileDelete {
  color: #ef4444;
  border-color: rgba(239, 68, 68, 0.22);
  background: rgba(239, 68, 68, 0.07);

  &:hover:not(:disabled) {
    background: rgba(239, 68, 68, 0.15);
    border-color: rgba(239, 68, 68, 0.4);
  }
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


@keyframes inferPulse {
  0%, 100% { box-shadow: 0 0 0 0 rgba(91, 164, 239, 0.4); }
  50%       { box-shadow: 0 0 0 4px rgba(91, 164, 239, 0); }
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
.dateFilter {
  display: flex;
  align-items: flex-end;
  gap: 6px;
  padding: 8px 12px 7px;
  border-bottom: none;
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
  background: var(--bg0);
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

.btnApply {
  align-self: flex-end;
  padding: 4px 10px !important;
  font-size: 13px !important;
  flex-shrink: 0;
}

// Uploaded file cards
.uploadedBody {
  flex: 1;
  overflow-y: auto;
  padding: 10px;
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(160px, 1fr));
  grid-auto-rows: 168px;
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
  padding: 16px 12px 30px;
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
  top: 8px;
  left: 8px;
  z-index: 1;
}

.fcardIcon {
  width: 38px;
  height: 38px;
  border-radius: 8px;
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
  font-size: 12px;
  font-weight: 500;
  color: var(--t0);
  line-height: 1.35;
  display: -webkit-box;
  -webkit-line-clamp: 2;
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
  bottom: 9px;
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
  background: var(--bg2);
  border: 1px solid var(--bdr);
  border-radius: var(--r);
  padding: 0 8px;
  gap: 6px;
  flex-shrink: 0;
  margin-bottom: 6px;
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
  padding: 6px 0;
  white-space: nowrap;
  flex-shrink: 0;
  border-right: 1px solid var(--bdr);
  padding-right: 8px;

  svg {
    width: 11px;
    height: 11px;
    opacity: 0.6;
  }
}

.sortCols {
  display: flex;
  flex: 1;
}

.sortCol {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 4px;
  padding: 6px 4px;
  background: transparent;
  border: none;
  border-right: 1px solid var(--bdr);
  color: var(--t2);
  font-size: 12px;
  font-family: var(--font-mono);
  cursor: pointer;
  transition: all 0.12s;

  &:last-child {
    border-right: none;
  }

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

  // Action grid — tighter
  .actionsGrid {
    padding: 0 8px 6px;
    gap: 3px;
  }

  .actionTile {
    padding: 4px 2px 3px;
    gap: 2px;
    font-size: 8px;

    svg {
      width: 11px;
      height: 11px;
    }
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
