// ═══════════════════════════════════════════════
// pages/UploadInfer/FilePanel.tsx
// Content Analytics · Step-1 upload + uploaded files list
// ═══════════════════════════════════════════════
import React, { useRef, useState, useCallback, useEffect } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import {
  toggleUploadZone,
  setDateFrom, setDateTo,
  serverFilesLoading, serverFilesSuccess, serverFilesFailure,
  setFilesPage, setFilesPageSize, setFilesSort, setFilesSearch,
  toggleServerFileSelection, toggleSelectAllServerFiles, clearServerSelection,
  type FilesSortBy, type SortOrder,
} from '../../store/uploadSlice';
import api from '../../services/api';
import styles from './FilePanel.module.scss';
import DictionaryAssociationModal from './DictionaryAssociationModal';
import PromptTemplateAssociationModal from './PromptTemplateAssociationModal';

// ── Types ────────────────────────────────────────
type UploadStatus = 'pending' | 'uploading' | 'success' | 'failed';
type PanelView = 'dropzone' | 'preview' | 'uploading';

interface BrowsedFile {
  id: string;
  file: File;
  name: string;
  size: string;
  ext: 'vtt' | 'srt';
  status: UploadStatus;
  error?: string;
}

// ── Helpers ──────────────────────────────────────
const ALLOWED = ['.vtt', '.srt'];
const isAllowed = (n: string) => ALLOWED.some(e => n.toLowerCase().endsWith(e));
const getExt = (n: string): 'vtt' | 'srt' => n.toLowerCase().endsWith('.srt') ? 'srt' : 'vtt';
const uid = () => Math.random().toString(36).slice(2);
const formatSize = (b: number) => b < 1024 ? `${b} B` : b < 1048576 ? `${(b / 1024).toFixed(1)} KB` : `${(b / 1048576).toFixed(1)} MB`;

const filesToBrowsed = (files: File[]): BrowsedFile[] =>
  files.filter(f => isAllowed(f.name)).map(f => ({
    id: uid(), file: f, name: f.name, size: formatSize(f.size), ext: getExt(f.name), status: 'pending' as UploadStatus,
  }));

// Windowed page-number list with '…' gaps, e.g. [1, '…', 4, 5, 6, '…', 20]
const getPageNumbers = (current: number, total: number): (number | '…')[] => {
  if (total <= 7) return Array.from({ length: total }, (_, i) => i + 1);
  const pages = new Set<number>([1, total, current, current - 1, current + 1, current - 2, current + 2]);
  const sorted = [...pages].filter(p => p >= 1 && p <= total).sort((a, b) => a - b);
  const result: (number | '…')[] = [];
  sorted.forEach((p, i) => {
    if (i > 0 && p - (sorted[i - 1] as number) > 1) result.push('…');
    result.push(p);
  });
  return result;
};

// ── Icons ────────────────────────────────────────
const IconPending: React.FC = () => <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round"><circle cx="8" cy="8" r="6" strokeDasharray="2 2" /></svg>;
const IconUploading: React.FC = () => <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round"><circle cx="8" cy="8" r="6" strokeOpacity="0.2" /><path d="M8 2a6 6 0 016 6" /></svg>;
const IconSuccess: React.FC = () => <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round"><circle cx="8" cy="8" r="6" /><path d="M5.5 8l2 2 3-3" /></svg>;
const IconFailed: React.FC = () => <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round"><circle cx="8" cy="8" r="6" /><path d="M6 6l4 4M10 6l-4 4" /></svg>;
const IconClose: React.FC = () => <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round"><path d="M4 4l8 8M12 4l-8 8" /></svg>;



const IconChevron: React.FC<{ up?: boolean }> = ({ up }) => (
  <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round"
    style={{ transform: up ? 'rotate(180deg)' : 'none', transition: 'transform 0.22s' }}>
    <path d="M4 6l4 4 4-4" />
  </svg>
);

// ── Checkbox ─────────────────────────────────────
interface CbProps { checked: boolean; indeterminate?: boolean; onChange: () => void; disabled?: boolean; }
const Checkbox: React.FC<CbProps> = ({ checked, indeterminate, onChange, disabled }) => {
  const ref = useRef<HTMLDivElement>(null);
  return (
    <div
      ref={ref}
      className={`${styles.cb} ${checked ? styles.cbChecked : ''} ${indeterminate ? styles.cbIndet : ''} ${disabled ? styles.cbDisabled : ''}`}
      onClick={e => { e.stopPropagation(); if (!disabled) onChange(); }}
      role="checkbox" aria-checked={indeterminate ? 'mixed' : checked} tabIndex={0}
      onKeyDown={e => { if (e.key === ' ' || e.key === 'Enter') { e.preventDefault(); if (!disabled) onChange(); } }}
    />
  );
};

// ── FilePanel ────────────────────────────────────
type SortKey = 'id' | 'name' | 'date' | 'status';
type SortDir = 'asc' | 'desc';

interface FilePanelProps {
  selectMode: boolean;
  onEnterSelectMode: () => void;
  onExitSelectMode: () => void;
  onFileClick: (fileId: number) => void;
  onDeleteComplete: (deletedIds: number[], all?: boolean) => void;
  activeFileId: number | null;
}

const FilePanel: React.FC<FilePanelProps> = ({ selectMode, onEnterSelectMode, onExitSelectMode, onFileClick, onDeleteComplete, activeFileId }) => {
  const { t } = useTranslation();
  const dispatch = useAppDispatch();
  const {
    uploadZoneCollapsed,
    serverFiles,
    serverFilesLoading: filesLoading, serverFilesError: filesError,
    selectedServerIds, isBatchRunning,
    dateFrom, dateTo,
    lastBatchFinishedAt,
    filesPage, filesPageSize, filesTotal, filesTotalPages,
    filesSortBy, filesSortOrder, filesSearch,
  } = useAppSelector(s => s.upload);

  const [view, setView] = useState<PanelView>('dropzone');
  const [browsed, setBrowsed] = useState<BrowsedFile[]>([]);
  const [isDragOver, setIsDragOver] = useState(false);

  // ── Delete mode ──────────────────────────────
  const [deleteMode, setDeleteMode] = useState(false);
  const [deleteSelectedIds, setDeleteSelectedIds] = useState<number[]>([]);
  const [isDeleting, setIsDeleting] = useState(false);

  // ── Delete ALL (danger zone — wipes every file on the account) ──
  const [deleteAllConfirmOpen, setDeleteAllConfirmOpen] = useState(false);
  const [deleteAllConfirmText, setDeleteAllConfirmText] = useState('');
  const [isDeletingAll, setIsDeletingAll] = useState(false);
  const [deleteAllError, setDeleteAllError] = useState<string | null>(null);
  const DELETE_ALL_CONFIRM_PHRASE = 'DELETE ALL';
  const isDeleteAllPhraseMatched = deleteAllConfirmText.trim() === DELETE_ALL_CONFIRM_PHRASE;

  // ── Export mode ──────────────────────────────
  const [exportMode, setExportMode] = useState(false);
  const [exportSelectedIds, setExportSelectedIds] = useState<number[]>([]);
  const [isExporting, setIsExporting] = useState(false);

  // ── Search ───────────────────────────────────
  const [searchOpen, setSearchOpen] = useState(false);
  const [searchQuery, setSearchQuery] = useState('');
  const searchInputRef = useRef<HTMLInputElement>(null);

  // ── Dictionary association modal ─────────────
  const [dictModalOpen, setDictModalOpen] = useState(false);

  // ── Prompt template association modal ────────
  const [templateModalOpen, setTemplateModalOpen] = useState(false);

  const openSearch = () => {
    setSearchOpen(true);
    setTimeout(() => searchInputRef.current?.focus(), 50);
  };
  const closeSearch = () => {
    setSearchOpen(false);
    setSearchQuery('');
  };

  const enterDeleteMode = () => { setDeleteMode(true); setDeleteSelectedIds([]); };
  const exitDeleteMode = () => { setDeleteMode(false); setDeleteSelectedIds([]); closeDeleteAllConfirm(); };

  const toggleDeleteSelection = (id: number) => {
    setDeleteSelectedIds(prev =>
      prev.includes(id) ? prev.filter(x => x !== id) : [...prev, id]
    );
  };

  // Checked/indeterminate reflect only the CURRENT PAGE's files — a file
  // selected on another page doesn't count toward "all selected" here.
  const pageDeleteSelectedCount = serverFiles.filter(f => deleteSelectedIds.includes(f.id)).length;
  const allDeleteSelected = serverFiles.length > 0 && pageDeleteSelectedCount === serverFiles.length;
  const someDeleteSelected = pageDeleteSelectedCount > 0 && !allDeleteSelected;

  const toggleDeleteSelectAll = () => {
    const pageIds = serverFiles.map(f => f.id);
    setDeleteSelectedIds(prev => {
      if (allDeleteSelected) {
        // Unchecking here only removes THIS page's ids — selections made on
        // other pages are left untouched.
        const pageIdSet = new Set(pageIds);
        return prev.filter(id => !pageIdSet.has(id));
      }
      // Checking here adds THIS page's ids on top of whatever's already
      // selected elsewhere, without duplicating.
      const existing = new Set(prev);
      return [...prev, ...pageIds.filter(id => !existing.has(id))];
    });
  };

  const handleConfirmDelete = async () => {
    if (!deleteSelectedIds.length) return;
    setIsDeleting(true);
    try {
      const res = await api.post('/files/delete', { fileID: deleteSelectedIds, all: false });
      if ((res.data as any)?.status === 'success' || res.status === 200) {
        const deleted = [...deleteSelectedIds];
        dispatch(clearServerSelection());
        await fetchFiles();
        exitDeleteMode();
        onDeleteComplete(deleted);
      }
    } catch {
      // silently ignore — stay in delete mode so user can retry
    } finally {
      setIsDeleting(false);
    }
  };

  // ── Delete ALL — wipes every file on the account, ignores fileID ──
  const openDeleteAllConfirm = () => {
    setDeleteAllError(null);
    setDeleteAllConfirmText('');
    setDeleteAllConfirmOpen(true);
  };
  const closeDeleteAllConfirm = () => {
    setDeleteAllConfirmOpen(false);
    setDeleteAllConfirmText('');
    setDeleteAllError(null);
  };
  const handleConfirmDeleteAll = async () => {
    if (!isDeleteAllPhraseMatched) return;
    setIsDeletingAll(true);
    setDeleteAllError(null);
    try {
      const res = await api.post('/files/delete', {
        all: true,
        start_date: dateFrom,
        end_date: dateTo,
      });
      if ((res.data as any)?.status === 'success' || res.status === 200) {
        dispatch(clearServerSelection());
        closeDeleteAllConfirm();
        exitDeleteMode();
        await fetchFiles({ page: 1 });
        onDeleteComplete([], true);
      } else {
        setDeleteAllError(t('uploadInfer.filePanel.deleteAllFailed'));
      }
    } catch (err) {
      setDeleteAllError(err instanceof Error ? err.message : t('uploadInfer.filePanel.deleteAllFailed'));
    } finally {
      setIsDeletingAll(false);
    }
  };

  // ── Export handlers ───────────────────────────
  const enterExportMode = () => { setExportMode(true); setExportSelectedIds([]); setExportAllError(null); };
  const exitExportMode = () => { setExportMode(false); setExportSelectedIds([]); setExportAllError(null); };

  const toggleExportSelection = (id: number) => {
    setExportSelectedIds(prev =>
      prev.includes(id) ? prev.filter(x => x !== id) : [...prev, id]
    );
  };

  // Checked/indeterminate reflect only the CURRENT PAGE's files — a file
  // selected on another page doesn't count toward "all selected" here.
  const pageExportSelectedCount = serverFiles.filter(f => exportSelectedIds.includes(f.id)).length;
  const allExportSelected = serverFiles.length > 0 && pageExportSelectedCount === serverFiles.length;
  const someExportSelected = pageExportSelectedCount > 0 && !allExportSelected;

  const toggleExportSelectAll = () => {
    const pageIds = serverFiles.map(f => f.id);
    setExportSelectedIds(prev => {
      if (allExportSelected) {
        const pageIdSet = new Set(pageIds);
        return prev.filter(id => !pageIdSet.has(id));
      }
      const existing = new Set(prev);
      return [...prev, ...pageIds.filter(id => !existing.has(id))];
    });
  };

  const handleConfirmExport = async () => {
    if (!exportSelectedIds.length) return;
    setIsExporting(true);
    try {
      const res = await api.post(
        '/export_excel',
        { file_ids: exportSelectedIds, all: false },
        { responseType: 'blob' },
      );

      // Use the original file name (without extension) + .xlsx
      // For a single file: "CS401_Week03.vtt" → "CS401_Week03.xlsx"
      // For multiple files: "export_2026-05-07.xlsx"
      let filename: string;
      if (exportSelectedIds.length === 1) {
        const f = serverFiles.find(sf => sf.id === exportSelectedIds[0]);
        const base = f ? f.original_name.replace(/\.[^.]+$/, '') : String(exportSelectedIds[0]);
        filename = `${base}.xlsx`;
      } else {
        const today = new Date().toISOString().slice(0, 10);
        filename = `export_${today}.xlsx`;
      }

      const url = URL.createObjectURL(new Blob([res.data]));
      const a = document.createElement('a');
      a.href = url;
      a.download = filename;
      document.body.appendChild(a);
      a.click();
      a.remove();
      URL.revokeObjectURL(url);

      exitExportMode();
    } catch {
      // silently ignore — stay in export mode so user can retry
    } finally {
      setIsExporting(false);
    }
  };

  // ── Export ALL — every file in the current date range, ignores file_ids ──
  const [isExportingAll, setIsExportingAll] = useState(false);
  const [exportAllError, setExportAllError] = useState<string | null>(null);

  const handleExportAll = async () => {
    setIsExportingAll(true);
    setExportAllError(null);
    try {
      const res = await api.post(
        '/export_excel',
        { all: true, start_date: dateFrom, end_date: dateTo },
        { responseType: 'blob' },
      );

      const filename = `export_all_${dateFrom}_${dateTo}.xlsx`;
      const url = URL.createObjectURL(new Blob([res.data]));
      const a = document.createElement('a');
      a.href = url;
      a.download = filename;
      document.body.appendChild(a);
      a.click();
      a.remove();
      URL.revokeObjectURL(url);

      exitExportMode();
    } catch (err) {
      setExportAllError(err instanceof Error ? err.message : t('uploadInfer.filePanel.exportAllFailed'));
    } finally {
      setIsExportingAll(false);
    }
  };

  const fileInputRef = useRef<HTMLInputElement>(null);
  const folderInputRef = useRef<HTMLInputElement>(null);

  // ── Fetch server files (paginated, sorted, searched) ──
  // Any param not passed in `opts` falls back to the current redux value, so
  // callers only need to specify what's actually changing (e.g. just `page`).
  const fetchFiles = useCallback(async (opts?: {
    from?: string; to?: string;
    page?: number; pageSize?: number;
    sortBy?: FilesSortBy; sortOrder?: SortOrder;
    search?: string;
  }) => {
    const from = opts?.from ?? dateFrom;
    const to = opts?.to ?? dateTo;
    const page = opts?.page ?? filesPage;
    const pageSize = opts?.pageSize ?? filesPageSize;
    const sortBy = opts?.sortBy ?? filesSortBy;
    const sortOrder = opts?.sortOrder ?? filesSortOrder;
    const search = opts?.search ?? filesSearch;

    dispatch(serverFilesLoading());
    try {
      const res = await api.post('/files/by-date/', {
        start_date: from,
        end_date: to,
        page,
        page_size: pageSize,
        sort_by: sortBy,
        sort_order: sortOrder,
        ...(search ? { search } : {}),
      });
      const d = (res.data as any)?.data ?? {};
      dispatch(serverFilesSuccess({
        files: d.data ?? [],
        total: d.total ?? 0,
        page: d.page ?? page,
        pageSize: d.page_size ?? pageSize,
        totalPages: d.total_pages ?? 1,
        sortBy, sortOrder, search,
      }));
    } catch (err) {
      dispatch(serverFilesFailure(err instanceof Error ? err.message : 'Failed to load files.'));
    }
  }, [dispatch, dateFrom, dateTo, filesPage, filesPageSize, filesSortBy, filesSortOrder, filesSearch]);

  useEffect(() => { fetchFiles({ page: 1 }); }, []); // eslint-disable-line

  // Re-fetch /by-date/ whenever a batch finishes so completed statuses are up to date
  useEffect(() => {
    if (lastBatchFinishedAt !== null) {
      fetchFiles();
    }
  }, [lastBatchFinishedAt]); // eslint-disable-line

  // ── Pagination / page-size handlers ──
  const goToPage = useCallback((p: number) => {
    if (p < 1 || p > filesTotalPages || p === filesPage || filesLoading) return;
    fetchFiles({ page: p });
  }, [filesTotalPages, filesPage, filesLoading, fetchFiles]);

  const handlePageSizeChange = useCallback((size: number) => {
    dispatch(setFilesPageSize(size));
    fetchFiles({ pageSize: size, page: 1 });
  }, [dispatch, fetchFiles]);

  // ── Browse / drop ────────────────────────────
  const handleFiles = useCallback((files: File[]) => {
    const valid = filesToBrowsed(files);
    if (!valid.length) return;
    setBrowsed(valid);
    setView('preview');
  }, []);

  const onFileChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    if (e.target.files) handleFiles(Array.from(e.target.files));
    e.target.value = '';
  };

  const onDragOver = (e: React.DragEvent) => { e.preventDefault(); setIsDragOver(true); };
  const onDragLeave = () => setIsDragOver(false);
  const onDrop = (e: React.DragEvent) => {
    e.preventDefault(); setIsDragOver(false);
    const files: File[] = [];
    if (e.dataTransfer.items) {
      const traverse = (entry: FileSystemEntry) => {
        if (entry.isFile) (entry as FileSystemFileEntry).file(f => { if (isAllowed(f.name)) files.push(f); });
        else if (entry.isDirectory) { const r = (entry as FileSystemDirectoryEntry).createReader(); r.readEntries(es => es.forEach(traverse)); }
      };
      Array.from(e.dataTransfer.items).forEach(item => { const e = item.webkitGetAsEntry?.(); if (e) traverse(e); });
      setTimeout(() => { if (files.length) handleFiles(files); }, 200);
    } else handleFiles(Array.from(e.dataTransfer.files));
  };

  // ── Upload actions ───────────────────────────
  const removeFile = (id: string) => { const n = browsed.filter(f => f.id !== id); setBrowsed(n); if (!n.length) setView('dropzone'); };
  const handleCancel = () => { setBrowsed([]); setView('dropzone'); };

  const handleUpload = async () => {
    setView('uploading');
    for (const bf of browsed) {
      setBrowsed(prev => prev.map(f => f.id === bf.id ? { ...f, status: 'uploading' } : f));
      try {
        const fd = new FormData(); fd.append('file', bf.file);
        const res = await api.post('/upload', fd, { headers: { 'Content-Type': 'multipart/form-data' } });
        const ok = res.status === 200 && (res.data as any)?.status === 'Success';
        setBrowsed(prev => prev.map(f => f.id === bf.id ? { ...f, status: ok ? 'success' : 'failed', error: ok ? undefined : 'Upload failed' } : f));
      } catch (err) {
        setBrowsed(prev => prev.map(f => f.id === bf.id ? { ...f, status: 'failed', error: err instanceof Error ? err.message : 'Upload failed' } : f));
      }
    }
  };

  useEffect(() => {
    if (view !== 'uploading') return;
    if (!browsed.every(f => f.status === 'success' || f.status === 'failed')) return;
    fetchFiles({ page: 1 });
    const allOk = browsed.every(f => f.status === 'success');
    if (allOk) {
      // All files uploaded successfully — clear immediately
      setBrowsed([]);
      setView('dropzone');
      return;
    }
    // Some failed — leave the list visible briefly so the user can read the error
    const t = setTimeout(() => { setBrowsed([]); setView('dropzone'); }, 2500);
    return () => clearTimeout(t);
  }, [browsed, view]); // eslint-disable-line

  const handleApply = () => fetchFiles({ page: 1 });

  // Select-all state for uploaded files (scoped to the current page)
  // Checked/indeterminate reflect only the CURRENT PAGE's files — files
  // selected on other pages (kept in redux) don't affect this checkbox.
  const pageSelectedCount = serverFiles.filter(f => selectedServerIds.includes(f.id)).length;
  const allSelected = serverFiles.length > 0 && pageSelectedCount === serverFiles.length;
  const someSelected = pageSelectedCount > 0 && !allSelected;

  // ── Sort (server-side — maps UI key to the API's sort_by values) ──────
  const SORT_KEY_TO_API: Record<SortKey, FilesSortBy> = {
    id: 'id', name: 'original_name', date: 'inserted_at', status: 'status',
  };
  const API_TO_SORT_KEY: Record<FilesSortBy, SortKey> = {
    id: 'id', original_name: 'name', inserted_at: 'date', status: 'status',
  };
  const sort = { key: API_TO_SORT_KEY[filesSortBy], dir: filesSortOrder as SortDir };

  // ── exitSelectMode — clears selections then notifies parent ──
  const exitSelectMode = useCallback(() => {
    dispatch(clearServerSelection());
    onExitSelectMode();
  }, [dispatch, onExitSelectMode]);

  const handleSort = useCallback((key: SortKey) => {
    const apiKey = SORT_KEY_TO_API[key];
    const dir: SortDir = filesSortBy === apiKey ? (filesSortOrder === 'asc' ? 'desc' : 'asc') : 'asc';
    dispatch(setFilesSort({ sortBy: apiKey, sortOrder: dir }));
    fetchFiles({ sortBy: apiKey, sortOrder: dir, page: 1 });
  }, [filesSortBy, filesSortOrder, dispatch, fetchFiles]); // eslint-disable-line

  // ── Search (debounced, server-side) ──────────────────────────────────
  const skipNextSearchEffect = useRef(true);
  useEffect(() => {
    if (skipNextSearchEffect.current) { skipNextSearchEffect.current = false; return; }
    const q = searchQuery.trim();
    const handle = setTimeout(() => {
      dispatch(setFilesSearch(q));
      fetchFiles({ search: q, page: 1 });
    }, 400);
    return () => clearTimeout(handle);
  }, [searchQuery]); // eslint-disable-line

  const displayedFiles = serverFiles;

  const done = browsed.filter(f => f.status === 'success').length;
  const failed = browsed.filter(f => f.status === 'failed').length;
  const finished = done + failed;

  return (
    <div className={styles.panel}>
      <input ref={fileInputRef} type="file" accept=".vtt,.srt" multiple style={{ display: 'none' }} onChange={onFileChange} />
      <input ref={folderInputRef} type="file" accept=".vtt,.srt" multiple style={{ display: 'none' }} onChange={onFileChange} {...{ webkitdirectory: 'true' }} />

      {/* ══ SECTION 1: Step-1 ══ */}
      <div className={`${styles.step1} ${uploadZoneCollapsed ? styles.step1Collapsed : ''}`}>
        <div className={styles.step1Bar}>
          <span className={styles.slbl}>{t('uploadInfer.filePanel.step1Label')}</span>
          {view === 'dropzone' && (
            <span className={`${styles.badge} ${styles.bReady}`}>{t('uploadInfer.filePanel.uploaded', { count: filesTotal })}</span>
          )}
          {(view === 'preview' || view === 'uploading') && (
            <span className={`${styles.badge} ${styles.bInfo}`}>{t('uploadInfer.filePanel.selected', { count: browsed.length })}</span>
          )}
          <button className={styles.collapseBtn} onClick={() => dispatch(toggleUploadZone())}
            title={uploadZoneCollapsed ? t('uploadInfer.filePanel.showUpload') : t('uploadInfer.filePanel.hideUpload')}>
            <IconChevron up={!uploadZoneCollapsed} />
            <span>{uploadZoneCollapsed ? t('uploadInfer.filePanel.showUpload') : t('uploadInfer.filePanel.hideUpload')}</span>
          </button>
        </div>

        <div className={styles.step1Content}>
          {view === 'dropzone' && (
            <div className={`${styles.dropzone} ${isDragOver ? styles.dragOver : ''}`}
              onDragOver={onDragOver} onDragLeave={onDragLeave} onDrop={onDrop}
              onClick={() => fileInputRef.current?.click()}>
              <div className={styles.dzIc}>
                <svg viewBox="0 0 18 18" fill="none" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round" stroke="var(--blue)">
                  <path d="M9 12V4M6 7l3-3 3 3" /><path d="M2 15h14" />
                </svg>
              </div>
              <div className={styles.dzTitle}>{isDragOver ? t('uploadInfer.filePanel.dropTitle') : t('uploadInfer.filePanel.dropTitleDefault')}</div>
              <div className={styles.dzSub}>{t('uploadInfer.filePanel.dropSub')}</div>
              <div className={styles.dzActions} onClick={e => e.stopPropagation()}>
                <button className={`${styles.btn} ${styles.btnP} ${styles.btnSm}`} onClick={() => fileInputRef.current?.click()}>{t('uploadInfer.filePanel.browseFiles')}</button>
                <button className={`${styles.btn} ${styles.btnSm}`} onClick={() => folderInputRef.current?.click()}>{t('uploadInfer.filePanel.folder')}</button>
              </div>
            </div>
          )}

          {(view === 'preview' || view === 'uploading') && (
            <div className={styles.previewWrap}>
              <div className={styles.previewList}>
                {browsed.map(bf => (
                  <div key={bf.id} className={`${styles.fileCard} ${styles[bf.status]}`}>
                    <div className={`${styles.extBadge} ${styles[bf.ext]}`}>{bf.ext.toUpperCase()}</div>
                    <div className={styles.fileInfo}>
                      <div className={styles.fileName}>{bf.name}</div>
                      <div className={styles.fileMeta}>
                        <span className={styles.fileSizeChip}>{bf.size}</span>
                        {bf.status === 'failed' && bf.error && <span className={styles.fileError}>{bf.error}</span>}
                        {bf.status === 'pending' && <span className={styles.fileStatusText}>{t('uploadInfer.filePanel.fileStatus.ready')}</span>}
                        {bf.status === 'uploading' && <span className={styles.fileStatusText}>{t('uploadInfer.filePanel.fileStatus.uploading')}</span>}
                        {bf.status === 'success' && <span className={styles.fileStatusTextSuccess}>{t('uploadInfer.filePanel.fileStatus.uploaded')}</span>}
                      </div>
                    </div>
                    <div className={`${styles.statusIc} ${styles[bf.status]}`}>
                      {bf.status === 'pending' && <IconPending />}
                      {bf.status === 'uploading' && <IconUploading />}
                      {bf.status === 'success' && <IconSuccess />}
                      {bf.status === 'failed' && <IconFailed />}
                    </div>
                    {view === 'preview' && <button className={styles.removeBtn} onClick={() => removeFile(bf.id)}><IconClose /></button>}
                  </div>
                ))}
              </div>
              {view === 'preview' && (
                <div className={styles.actionBar}>
                  <button className={`${styles.btn} ${styles.btnP}`} onClick={handleUpload}>
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                      <path d="M8 11V4M5 7l3-3 3 3" /><path d="M2.5 13.5h11" />
                    </svg>
                    {t('uploadInfer.filePanel.uploadBtn', { count: browsed.length })}
                  </button>
                  <button className={`${styles.btn} ${styles.btnDanger}`} onClick={handleCancel}>{t('uploadInfer.filePanel.cancelBtn')}</button>
                </div>
              )}
              {view === 'uploading' && (
                <div className={styles.uploadSummary}>
                  <div className={styles.uploadProgressBar}>
                    <div className={styles.uploadProgressFill} style={{ width: `${browsed.length ? (finished / browsed.length) * 100 : 0}%` }} />
                  </div>
                  <div className={styles.uploadProgressLabel}>
                    {t('uploadInfer.filePanel.complete', { finished, total: browsed.length })}
                    {failed > 0 && <span className={styles.failCount}>{t('uploadInfer.filePanel.failed', { count: failed })}</span>}
                  </div>
                </div>
              )}
            </div>
          )}
        </div>
      </div>

      {/* ══ SECTION 2: Uploaded files ══ */}
      <div className={styles.section2}>

        {/* Header */}
        <div className={styles.section2Header}>

          {/* ── Title row (always visible) ── */}
          <div className={styles.section2TitleRow}>
            {/* Select-all checkbox — inference mode */}
            {selectMode && !isBatchRunning && (
              <Checkbox
                checked={allSelected}
                indeterminate={someSelected}
                onChange={() => dispatch(toggleSelectAllServerFiles())}
                disabled={filesTotal === 0}
              />
            )}

            {/* Select-all checkbox — export mode */}
            {exportMode && !isBatchRunning && (
              <Checkbox
                checked={allExportSelected}
                indeterminate={someExportSelected}
                onChange={toggleExportSelectAll}
                disabled={filesTotal === 0}
              />
            )}

            {/* Select-all checkbox — delete mode */}
            {deleteMode && !isBatchRunning && (
              <Checkbox
                checked={allDeleteSelected}
                indeterminate={someDeleteSelected}
                onChange={toggleDeleteSelectAll}
                disabled={filesTotal === 0}
              />
            )}

            <div className={styles.section2Title}>
              {t('uploadInfer.filePanel.uploadedFiles')}
              {!filesLoading && filesTotal > 0 && (
                <span className={styles.filesCount}>{filesTotal}</span>
              )}
              {selectMode && !isBatchRunning && selectedServerIds.length > 0 && (
                <span className={styles.selectedTotalHint}>
                  {t('uploadInfer.filePanel.selectedTotal', { count: selectedServerIds.length, total: filesTotal })}
                </span>
              )}
              {deleteMode && !isBatchRunning && deleteSelectedIds.length > 0 && (
                <span className={styles.selectedTotalHint}>
                  {t('uploadInfer.filePanel.selectedTotal', { count: deleteSelectedIds.length, total: filesTotal })}
                </span>
              )}
              {exportMode && !isBatchRunning && exportSelectedIds.length > 0 && (
                <span className={styles.selectedTotalHint}>
                  {t('uploadInfer.filePanel.selectedTotal', { count: exportSelectedIds.length, total: filesTotal })}
                </span>
              )}
            </div>

            {/* Search stays in title row for select mode only; export/delete have their own row */}
            {selectMode && !isBatchRunning && (
              <div className={styles.headerActions}>
                <button
                  className={`${styles.searchIconBtn} ${searchOpen ? styles.searchIconBtnActive : ''}`}
                  onClick={openSearch}
                  title={t('uploadInfer.filePanel.searchBtn')}
                >
                  <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                    <circle cx="6.5" cy="6.5" r="4" />
                    <path d="M11 11l2.5 2.5" />
                  </svg>
                </button>
              </div>
            )}
          </div>

          {/* ── Export mode action row ── */}
          {exportMode && !isBatchRunning && (
            <div className={styles.modeActionsRow}>
              <div className={styles.modeActionsLeft}>
                <button
                  className={`${styles.modeTile} ${styles.modeTileSearch} ${searchOpen ? styles.modeTileSearchActive : ''}`}
                  onClick={openSearch}
                >
                  <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                    <circle cx="6.5" cy="6.5" r="4" />
                    <path d="M11 11l2.5 2.5" />
                  </svg>
                  {t('uploadInfer.filePanel.searchBtn')}
                </button>
                <button
                  className={styles.modeTileExportAll}
                  onClick={handleExportAll}
                  disabled={filesTotal === 0 || isExportingAll || isExporting}
                  title={t('uploadInfer.filePanel.exportAllBtn')}
                >
                  {isExportingAll ? (
                    <div className={styles.miniSpinnerGreen} />
                  ) : (
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                      <path d="M9 2H4a1 1 0 00-1 1v10a1 1 0 001 1h8a1 1 0 001-1V6z" />
                      <path d="M9 2v4h4" />
                      <path d="M6 9.5l1.5 2 2.5-3" />
                    </svg>
                  )}
                  {t('uploadInfer.filePanel.exportAllBtn')}
                </button>
                {exportAllError && <span className={styles.modeActionsError}>{exportAllError}</span>}
              </div>
              <div className={styles.modeActionsRight}>
                <button
                  className={`${styles.modeTile} ${styles.modeTileConfirmExport} ${exportSelectedIds.length === 0 ? styles.modeTileDisabled : ''}`}
                  onClick={handleConfirmExport}
                  disabled={exportSelectedIds.length === 0 || isExporting}
                >
                  {isExporting ? (
                    <div className={styles.miniSpinnerGreen} />
                  ) : (
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                      <path d="M9 2H4a1 1 0 00-1 1v10a1 1 0 001 1h8a1 1 0 001-1V6z" />
                      <path d="M9 2v4h4" />
                      <path d="M6 9.5l1.5 2 2.5-3" />
                    </svg>
                  )}
                  {exportSelectedIds.length > 0
                    ? t('uploadInfer.filePanel.exportCount', { count: exportSelectedIds.length })
                    : t('uploadInfer.filePanel.exportBtn')}
                </button>
                <button
                  className={`${styles.modeTile} ${styles.modeTileCancel}`}
                  onClick={exitExportMode}
                >
                  <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                    <path d="M4 4l8 8M12 4l-8 8" />
                  </svg>
                  {t('uploadInfer.filePanel.cancelBtn')}
                </button>
              </div>
            </div>
          )}

          {/* ── Delete mode action row ── */}
          {deleteMode && !isBatchRunning && (
            <div className={styles.modeActionsRow}>
              <div className={styles.modeActionsLeft}>
                <button
                  className={`${styles.modeTile} ${styles.modeTileSearch} ${searchOpen ? styles.modeTileSearchActive : ''}`}
                  onClick={openSearch}
                >
                  <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                    <circle cx="6.5" cy="6.5" r="4" />
                    <path d="M11 11l2.5 2.5" />
                  </svg>
                  {t('uploadInfer.filePanel.searchBtn')}
                </button>
                <button
                  className={styles.modeTileDeleteAll}
                  onClick={openDeleteAllConfirm}
                  disabled={filesTotal === 0}
                  title={t('uploadInfer.filePanel.deleteAllBtn')}
                >
                  <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                    <path d="M8 1.5l6.5 11.5h-13z" />
                    <path d="M8 6v3.5M8 12v.1" />
                  </svg>
                  {t('uploadInfer.filePanel.deleteAllBtn')}
                </button>
              </div>
              <div className={styles.modeActionsRight}>
                <button
                  className={`${styles.modeTile} ${styles.modeTileConfirmDelete} ${deleteSelectedIds.length === 0 ? styles.modeTileDisabled : ''}`}
                  onClick={handleConfirmDelete}
                  disabled={deleteSelectedIds.length === 0 || isDeleting}
                >
                  {isDeleting ? (
                    <div className={styles.miniSpinner} />
                  ) : (
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round" strokeLinejoin="round">
                      <path d="M3 4h10M6 4V3h4v1" />
                      <path d="M5 4l.5 8h5l.5-8" />
                      <path d="M7 7v3M9 7v3" />
                    </svg>
                  )}
                  {deleteSelectedIds.length > 0
                    ? t('uploadInfer.filePanel.deleteCount', { count: deleteSelectedIds.length })
                    : t('uploadInfer.filePanel.deleteBtn')}
                </button>
                <button
                  className={`${styles.modeTile} ${styles.modeTileCancel}`}
                  onClick={exitDeleteMode}
                >
                  <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                    <path d="M4 4l8 8M12 4l-8 8" />
                  </svg>
                  {t('uploadInfer.filePanel.cancelBtn')}
                </button>
              </div>
            </div>
          )}

          {/* ── Action grid — 3×2 tiles, only in normal (idle) mode ── */}
          {!selectMode && !deleteMode && !exportMode && !isBatchRunning && (
            <div className={styles.actionsGrid}>
              {/* Infer */}
              <button
                className={`${styles.actionTile} ${styles.actionTileInfer}`}
                onClick={onEnterSelectMode}
                title={t('uploadInfer.filePanel.inferenceBtn')}
              >
                <svg viewBox="0 0 16 16" fill="currentColor" stroke="none">
                  <path d="M8 1l1.8 4.4L14 6.2l-3.3 2.5 1.2 4.3L8 10.8 4.1 13l1.2-4.3L2 6.2l4.2-.8z" />
                </svg>
                {t('uploadInfer.filePanel.inferenceBtn')}
              </button>

              {/* Dictionary */}
              <button
                className={`${styles.actionTile} ${styles.actionTileDictionary}`}
                onClick={() => setDictModalOpen(true)}
                disabled={filesTotal === 0}
                title={t('uploadInfer.filePanel.dictionaryBtn')}
              >
                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                  strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                  <path d="M3 2.5h7.5a1.5 1.5 0 011.5 1.5v9a.5.5 0 01-.5.5H4a1 1 0 01-1-1V2.5z" />
                  <path d="M3 11.5a1 1 0 011-1h8" />
                  <path d="M6 5.5h3" />
                </svg>
                {t('uploadInfer.filePanel.dictionaryBtn')}
              </button>

              {/* Template */}
              <button
                className={`${styles.actionTile} ${styles.actionTileTemplate}`}
                onClick={() => setTemplateModalOpen(true)}
                disabled={filesTotal === 0}
                title={t('uploadInfer.filePanel.templateBtn')}
              >
                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                  strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                  <path d="M3 2.5h10v11H3z" />
                  <path d="M5.5 5.5h5M5.5 8h5M5.5 10.5h3" />
                </svg>
                {t('uploadInfer.filePanel.templateBtn')}
              </button>

              {/* Search */}
              <button
                className={`${styles.actionTile} ${styles.actionTileSearch} ${searchOpen ? styles.actionTileActive : ''}`}
                onClick={openSearch}
                title={t('uploadInfer.filePanel.searchBtn')}
              >
                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                  <circle cx="6.5" cy="6.5" r="4" />
                  <path d="M11 11l2.5 2.5" />
                </svg>
                {t('uploadInfer.filePanel.searchBtn')}
              </button>

              {/* Export */}
              <button
                className={`${styles.actionTile} ${styles.actionTileExport}`}
                onClick={enterExportMode}
                disabled={filesTotal === 0}
                title={t('uploadInfer.filePanel.exportBtn')}
              >
                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                  strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                  <path d="M9 2H4a1 1 0 00-1 1v10a1 1 0 001 1h8a1 1 0 001-1V6z" />
                  <path d="M9 2v4h4" />
                  <path d="M6 9.5l1.5 2 2.5-3" />
                </svg>
                {t('uploadInfer.filePanel.exportBtn')}
              </button>

              {/* Delete */}
              <button
                className={`${styles.actionTile} ${styles.actionTileDelete}`}
                onClick={enterDeleteMode}
                disabled={filesTotal === 0}
                title={t('uploadInfer.filePanel.deleteBtn')}
              >
                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                  strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                  <path d="M3 4h10M6 4V3h4v1" />
                  <path d="M5 4l.5 8h5l.5-8" />
                  <path d="M7 7v3M9 7v3" />
                </svg>
                {t('uploadInfer.filePanel.deleteBtn')}
              </button>
            </div>
          )}

        </div>

        <div className={styles.dateFilter}>
          <div className={styles.dateField}>
            <label className={styles.dateLabel}>{t('uploadInfer.filePanel.dateFrom')}</label>
            <input type="date" className={styles.dateInput} value={dateFrom} max={dateTo}
              disabled={isBatchRunning}
              onChange={e => dispatch(setDateFrom(e.target.value))} />
          </div>
          <div className={styles.dateSep}>—</div>
          <div className={styles.dateField}>
            <label className={styles.dateLabel}>{t('uploadInfer.filePanel.dateTo')}</label>
            <input type="date" className={styles.dateInput} value={dateTo} min={dateFrom}
              disabled={isBatchRunning}
              onChange={e => dispatch(setDateTo(e.target.value))} />
          </div>
          <button className={`${styles.btn} ${styles.btnApply}`} onClick={handleApply} disabled={filesLoading || isBatchRunning}>{t('uploadInfer.filePanel.applyDate')}</button>
        </div>

        {/* Sort header */}
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

        {/* Search bar */}
        <div className={`${styles.searchBar} ${searchOpen ? styles.searchBarOpen : ''}`}>
          <div className={styles.searchBarRow}>
            <div className={styles.searchInner}>
              <svg className={styles.searchBarIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor"
                strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                <circle cx="6.5" cy="6.5" r="4" />
                <path d="M11 11l2.5 2.5" />
              </svg>
              <input
                ref={searchInputRef}
                type="text"
                className={styles.searchInput}
                placeholder={t('uploadInfer.filePanel.searchPlaceholder')}
                value={searchQuery}
                onChange={e => setSearchQuery(e.target.value)}
              />
              {filesSearch && (
                <span className={styles.searchCount}>
                  {t('uploadInfer.filePanel.matchCount', { count: filesTotal })}
                </span>
              )}
            </div>
            <button className={styles.searchCloseBtn} onClick={closeSearch} title={t('uploadInfer.filePanel.searchBtn')}>
              <svg viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round">
                <path d="M2 2l8 8M10 2l-8 8" />
              </svg>
            </button>
          </div>
        </div>

        {/* File list */}
        <div className={styles.uploadedBody}>
          {filesLoading && (
            <div className={styles.listState}><div className={styles.spinner} /><span>{t('uploadInfer.filePanel.loadingFiles')}</span></div>
          )}
          {!filesLoading && filesError && (
            <div className={`${styles.listState} ${styles.errorState}`}>
              <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                <circle cx="8" cy="8" r="6" /><path d="M8 5v3M8 11v.5" />
              </svg>
              {filesError}
            </div>
          )}
          {!filesLoading && !filesError && filesTotal === 0 && !filesSearch && (
            <div className={styles.listState}>
              <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                <rect x="2" y="2" width="12" height="12" rx="2" /><path d="M5 8h6M5 5.5h4M5 10.5h6" />
              </svg>
              {t('uploadInfer.filePanel.noFilesRange')}
            </div>
          )}
          {!filesLoading && !filesError && filesTotal === 0 && filesSearch && (
            <div className={styles.listState}>
              <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                <circle cx="6.5" cy="6.5" r="4" /><path d="M11 11l2.5 2.5" />
              </svg>
              {t('uploadInfer.filePanel.noFilesMatch', { query: filesSearch })}
            </div>
          )}
          {!filesLoading && !filesError && displayedFiles.map(f => {
            const ext = getExt(f.original_name);
            const isChecked = selectedServerIds.includes(f.id);
            const isDeleteChecked = deleteSelectedIds.includes(f.id);
            const isExportChecked = exportSelectedIds.includes(f.id);
            const selectable = selectMode && !isBatchRunning;
            const deletable = deleteMode && !isBatchRunning;
            const exportable = exportMode && !isBatchRunning;
            return (
              <div
                key={f.id}
                className={`${styles.hitm}
                  ${selectable ? (isChecked ? styles.active : '') : ''}
                  ${deletable ? (isDeleteChecked ? styles.activeDelete : '') : ''}
                  ${exportable ? (isExportChecked ? styles.activeExport : '') : ''}
                  ${!selectable && !deletable && !exportable ? (f.id === activeFileId ? styles.activeView : '') : ''}
                  ${styles.hitmSelectable}`}
                onClick={() => {
                  if (selectable) dispatch(toggleServerFileSelection(f.id));
                  else if (deletable) toggleDeleteSelection(f.id);
                  else if (exportable) toggleExportSelection(f.id);
                  else onFileClick(f.id);
                }}
              >
                {selectable && (
                  <Checkbox checked={isChecked} onChange={() => dispatch(toggleServerFileSelection(f.id))} />
                )}
                {deletable && (
                  <Checkbox checked={isDeleteChecked} onChange={() => toggleDeleteSelection(f.id)} />
                )}
                {exportable && (
                  <Checkbox checked={isExportChecked} onChange={() => toggleExportSelection(f.id)} />
                )}
                <div className={`${styles.ficon} ${styles[ext]}`}>{ext.toUpperCase()}</div>
                <div className={styles.hi}>
                  <div className={styles.hn}>{f.original_name}</div>
                  <div className={styles.hm}>{f.inserted_at} · #{f.id}</div>
                </div>
                <span className={`${styles.badge} ${isChecked ? styles.bSelected :
                  isDeleteChecked ? styles.bDelete :
                    isExportChecked ? styles.bExport :
                      f.status === 'completed' ? styles.bInferred : styles.bNotInferred
                  }`}>
                  {isChecked ? (
                    <svg viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" width="10" height="10">
                      <path d="M2 6l3 3 5-5" />
                    </svg>
                  ) : isDeleteChecked ? (
                    <svg viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" width="10" height="10">
                      <path d="M2 2l8 8M10 2l-8 8" />
                    </svg>
                  ) : isExportChecked ? (
                    <>
                      <svg viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" width="9" height="9">
                        <path d="M6 1v6M3.5 4.5L6 7l2.5-2.5" /><path d="M2 10h8" />
                      </svg>
                      {t('uploadInfer.filePanel.export')}
                    </>
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
        {!filesLoading && !filesError && filesTotal > 0 && (
          <div className={styles.paginationBar}>
            <div className={styles.pageSizeGroup}>
              <span className={styles.pageSizeLabel}>{t('uploadInfer.filePanel.perPage')}</span>
              <select
                className={styles.pageSizeSelect}
                value={filesPageSize}
                onChange={e => handlePageSizeChange(Number(e.target.value))}
              >
                {[100, 150, 200].map(size => (
                  <option key={size} value={size}>{size}</option>
                ))}
              </select>
            </div>

            <div className={styles.pageNav}>
              <button
                className={styles.pageNavBtn}
                onClick={() => goToPage(filesPage - 1)}
                disabled={filesPage <= 1}
                aria-label="Previous page"
              >
                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                  <path d="M10 3L6 8l4 5" />
                </svg>
              </button>

              {getPageNumbers(filesPage, filesTotalPages).map((p, i) =>
                p === '…' ? (
                  <span key={`ellipsis-${i}`} className={styles.pageEllipsis}>…</span>
                ) : (
                  <button
                    key={p}
                    className={`${styles.pageNumBtn} ${p === filesPage ? styles.pageNumBtnActive : ''}`}
                    onClick={() => goToPage(p as number)}
                  >
                    {p}
                  </button>
                )
              )}

              <button
                className={styles.pageNavBtn}
                onClick={() => goToPage(filesPage + 1)}
                disabled={filesPage >= filesTotalPages}
                aria-label="Next page"
              >
                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                  <path d="M6 3l4 5-4 5" />
                </svg>
              </button>
            </div>

            <div className={styles.pageInfo}>
              {t('uploadInfer.filePanel.pageInfo', { page: filesPage, totalPages: filesTotalPages, total: filesTotal })}
            </div>
          </div>
        )}
      </div>

      <DictionaryAssociationModal
        open={dictModalOpen}
        onClose={() => setDictModalOpen(false)}
      />

      <PromptTemplateAssociationModal
        open={templateModalOpen}
        onClose={() => setTemplateModalOpen(false)}
      />

      {/* ── Delete ALL confirmation — wipes every file on the account ── */}
      {deleteAllConfirmOpen && (
        <div className={styles.dangerOverlay} onClick={() => !isDeletingAll && closeDeleteAllConfirm()}>
          <div className={styles.dangerModal} onClick={e => e.stopPropagation()}>
            <div className={styles.dangerIcon}>
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round" strokeLinejoin="round">
                <path d="M12 2l10 18H2z" />
                <path d="M12 9v5M12 17v.1" />
              </svg>
            </div>
            <div className={styles.dangerTitle}>{t('uploadInfer.filePanel.deleteAllConfirmTitle')}</div>
            <div className={styles.dangerBody}>
              {t('uploadInfer.filePanel.deleteAllConfirmBody', { from: dateFrom, to: dateTo })}
            </div>
            <label
              className={styles.dangerLabel}
              dangerouslySetInnerHTML={{
                __html: t('uploadInfer.filePanel.deleteAllConfirmPrompt', { phrase: DELETE_ALL_CONFIRM_PHRASE }),
              }}
            />
            <input
              type="text"
              className={styles.dangerInput}
              value={deleteAllConfirmText}
              onChange={e => setDeleteAllConfirmText(e.target.value)}
              placeholder={DELETE_ALL_CONFIRM_PHRASE}
              autoFocus
              autoComplete="off"
              autoCorrect="off"
              autoCapitalize="off"
              spellCheck={false}
              disabled={isDeletingAll}
            />
            {deleteAllError && <div className={styles.dangerError}>{deleteAllError}</div>}
            <div className={styles.dangerActions}>
              <button
                className={`${styles.btn} ${styles.btnFull}`}
                onClick={closeDeleteAllConfirm}
                disabled={isDeletingAll}
              >
                {t('uploadInfer.filePanel.cancelBtn')}
              </button>
              <button
                type="button"
                className={`${styles.btn} ${styles.btnDanger} ${styles.btnFull}`}
                onClick={handleConfirmDeleteAll}
                disabled={!isDeleteAllPhraseMatched || isDeletingAll}
                aria-disabled={!isDeleteAllPhraseMatched || isDeletingAll}
              >
                {isDeletingAll ? <div className={styles.miniSpinner} /> : t('uploadInfer.filePanel.deleteAllConfirmBtn')}
              </button>
            </div>
          </div>
        </div>
      )}
    </div>
  );
};

export default FilePanel;





















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
export interface CooccurrenceData { metrix: number[][]; keywords: string[]; }
export interface GlossaryItem { term: string; definition: string; first_mention_ms: number; }
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

const UploadInfer: React.FC = () => {
  const { t } = useTranslation();
  const isBatchRunning = useAppSelector(s => s.upload.isBatchRunning);
  const serverFiles = useAppSelector(s => s.upload.serverFiles);
  const dispatch = useAppDispatch();

  const [selectMode, setSelectMode] = useState(false);
  const step2Visible = selectMode || isBatchRunning;

  const [step2Minimized, setStep2Minimized] = useState(false);
  useEffect(() => { if (step2Visible) setStep2Minimized(false); }, [step2Visible]);
  useEffect(() => { if (!isBatchRunning) setSelectMode(false); }, [isBatchRunning]);
  useEffect(() => { if (!selectMode) dispatch(clearServerSelection()); }, [selectMode]); // eslint-disable-line
  useEffect(() => { return () => { dispatch(clearServerSelection()); }; }, []); // eslint-disable-line

  const [fileResult, setFileResult] = useState<FileResult | null>(null);
  const [fileLoading, setFileLoading] = useState(false);
  const [activeFileId, setActiveFileId] = useState<number | null>(null);

  const fetchFileData = useCallback(async (fileId: number) => {
    setFileLoading(true);
    try {
      const res = await api.get(`/files/${fileId}`);
      const d = (res.data as any)?.data ?? {};
      const serverFile = serverFiles.find(f => f.id === fileId);
      const fileName = d.original_name ?? serverFile?.original_name ?? String(fileId);
      const insertedAt = d.inserted_at ?? serverFile?.inserted_at ?? '';
      setFileResult({
        fileId, fileName, insertedAt,
        summary: d.summary ?? '',
        keywords: d.keywords ?? [],
        faq: d.faq ?? '[]',
        shortAnswer: d.short_answer ?? '[]',
        trueFalse: d.true_false ?? '[]',
        // NOTE: backend key is genuinely "timstamped_summary" (missing an "e") — match it exactly.
        timestampedSummary: d.timstamped_summary ?? '[]',
        keywordInsights: d.keyword_insights ?? null,
      });
    } catch { setFileResult(null); } finally { setFileLoading(false); }
  }, [serverFiles]);

  const handleFileClick = useCallback(async (fileId: number) => {
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

  return (
    <div className={styles.page}>
      <div className={styles.ph}>
        <div className={styles.phRow}>
          <div>
            <div className={styles.phTitle}>{t('uploadInfer.pageTitle')}</div>
            <div className={styles.phSub}>{t('uploadInfer.pageSub')}</div>
          </div>
        </div>
      </div>

      <div className={styles.upbody}>
        <FilePanel
          selectMode={selectMode}
          onEnterSelectMode={() => setSelectMode(true)}
          onExitSelectMode={() => setSelectMode(false)}
          onFileClick={handleFileClick}
          onDeleteComplete={handleDeleteComplete}
          activeFileId={activeFileId}
        />

        <div className={
          `${styles.step2Wrap} ` +
          `${step2Visible ? styles.step2WrapVisible : styles.step2WrapHidden} ` +
          `${step2Visible && step2Minimized ? styles.step2WrapMinimized : ''}`
        }>
          <InferencePanel
            onClose={() => setSelectMode(false)}
            minimized={step2Minimized}
            onToggleMinimize={() => setStep2Minimized(v => !v)}
          />
        </div>

        <WorkspacePanel
          step2Visible={step2Visible}
          step2Minimized={step2Minimized}
          fileResult={fileResult}
          fileLoading={fileLoading}
          activeFileId={activeFileId}
          onResultUpdate={patch => setFileResult(prev => prev ? { ...prev, ...patch } : prev)}
        />
      </div>
    </div>
  );
};

export default UploadInfer;






















// ═══════════════════════════════════════════════
// pages/UploadInfer/WorkspacePanel.tsx
// LectureAI · Step-3 Workspace Result panel
// ═══════════════════════════════════════════════
import React, { useState, useMemo, useRef, useEffect, useCallback } from 'react';
import { useTranslation } from 'react-i18next';
import { marked } from 'marked';
import {
    ReactFlow, Background, Controls, MiniMap, Handle, Position, applyNodeChanges,
    type Node as RFNode, type Edge as RFEdge, type NodeChange,
} from '@xyflow/react';
import '@xyflow/react/dist/style.css';
import api from '../../services/api';
import type { FileResult, WordCloudData, ClustersData, FrequencyData, ImportanceComplexityData, GlossaryData, KeywordInsights } from './UploadInfer';
import styles from './WorkspacePanel.module.scss';

marked.setOptions({ breaks: false, gfm: true });

interface Props {
    step2Visible?: boolean;
    step2Minimized?: boolean;
    fileResult: FileResult | null;
    fileLoading: boolean;
    activeFileId: number | null;
    onResultUpdate?: (patch: Partial<Pick<FileResult, 'summary' | 'keywords'>>) => void;
}

// TABS labels are now driven by i18n — see tabLabels() inside WorkspacePanel
const TAB_IDS = ['summary', 'keywords', 'assessment', 'shortAnswer', 'trueFalse', 'timestampedSummary', 'keywordInsights'] as const;
type TabId = typeof TAB_IDS[number];


interface FaqItem {
    question: string;
    options: Record<string, string>;
    correct_answer: string;
    explanation: string;
}

function parseFaq(raw: string): FaqItem[] {
    if (!raw || raw === '[]') return [];
    try {
        // eslint-disable-next-line no-eval
        const result = eval(raw);
        if (Array.isArray(result)) return result as FaqItem[];
        return [];
    } catch {
        // Fallback: try direct JSON parse (if API returns valid JSON)
        try { return JSON.parse(raw) as FaqItem[]; } catch { }
        return [];
    }
}

// ── Permissive parser for the newer python-repr-style fields (short_answer,
//    true_false, timestamped_summary). Handles True/False/None literals that
//    plain eval() can't, and tries strict JSON first since that's cheaper
//    and safer whenever the backend does send valid JSON. ──
function parsePyList<T>(raw: string): T[] {
    if (!raw || raw === '[]') return [];
    try {
        const result = JSON.parse(raw);
        if (Array.isArray(result)) return result as T[];
    } catch { /* not valid JSON — fall through to python-ish eval */ }
    try {
        const normalized = raw
            .replace(/\bTrue\b/g, 'true')
            .replace(/\bFalse\b/g, 'false')
            .replace(/\bNone\b/g, 'null');
        // eslint-disable-next-line no-eval
        const result = eval('(' + normalized + ')');
        if (Array.isArray(result)) return result as T[];
    } catch { /* give up */ }
    return [];
}

interface ShortAnswerItem { question: string; answer: string; }
interface TrueFalseItem { statement: string; is_true: boolean; explanation: string; }
interface TimestampSegment { start_time: string; end_time: string; summary: string; }

const CHIP_COLORS = [
    'linear-gradient(135deg, #667eea 0%, #764ba2 100%)',
    'linear-gradient(135deg, #f093fb 0%, #f5576c 100%)',
    'linear-gradient(135deg, #4facfe 0%, #00f2fe 100%)',
    'linear-gradient(135deg, #43e97b 0%, #38f9d7 100%)',
    'linear-gradient(135deg, #fa709a 0%, #fee140 100%)',
    'linear-gradient(135deg, #30cfd0 0%, #330867 100%)',
    'linear-gradient(135deg, #a8edea 0%, #fed6e3 100%)',
    'linear-gradient(135deg, #ff9a56 0%, #ff6a88 100%)',
    'linear-gradient(135deg, #ffecd2 0%, #fcb69f 100%)',
    'linear-gradient(135deg, #a1c4fd 0%, #c2e9fb 100%)',
];

function seededColorIndex(str: string): number {
    let hash = 0;
    for (let i = 0; i < str.length; i++) hash = (hash * 31 + str.charCodeAt(i)) >>> 0;
    return hash % CHIP_COLORS.length;
}

// ── Copy / Download helpers ───────────────────────────────
function copyText(text: string) {
    if (navigator.clipboard) { navigator.clipboard.writeText(text).catch(() => fallbackCopy(text)); }
    else fallbackCopy(text);
}
function fallbackCopy(text: string) {
    const ta = document.createElement('textarea');
    ta.value = text; ta.style.cssText = 'position:fixed;top:0;left:0;opacity:0';
    document.body.appendChild(ta); ta.select();
    try { document.execCommand('copy'); } catch { }
    document.body.removeChild(ta);
}
function downloadFile(content: string, filename: string, mime: string) {
    const blob = new Blob([content], { type: mime });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url; a.download = filename;
    document.body.appendChild(a); a.click();
    document.body.removeChild(a); URL.revokeObjectURL(url);
}
function wrapHtml(body: string, title: string): string {
    return `<!DOCTYPE html><html lang="en"><head><meta charset="UTF-8"><title>${title}</title><style>body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;max-width:900px;margin:40px auto;padding:20px;background:#f8f9fa;color:#333}</style></head><body>${body}</body></html>`;
}

// ── Format helpers ────────────────────────────────────────
type Formatted = { content: string; filename: string; mime: string };

function formatSummary(summary: string, fmt: string, base = 'summary'): Formatted {
    const ts = new Date().toISOString().slice(0, 10);
    if (fmt === 'html') return { content: wrapHtml(marked.parse(summary) as string, 'Summary'), filename: `${base}_summary_${ts}.html`, mime: 'text/html' };
    if (fmt === 'json') return { content: JSON.stringify({ type: 'summary', content: summary, timestamp: new Date().toISOString() }, null, 2), filename: `${base}_summary_${ts}.json`, mime: 'application/json' };
    if (fmt === 'md') return { content: summary, filename: `${base}_summary_${ts}.md`, mime: 'text/markdown' };
    return { content: summary, filename: `${base}_summary_${ts}.txt`, mime: 'text/plain' };
}

function formatKeywords(keywords: string[], fmt: string, base = 'keywords'): Formatted {
    const ts = new Date().toISOString().slice(0, 10);
    if (fmt === 'html') return { content: wrapHtml(`<div style="display:flex;flex-wrap:wrap;gap:10px">${keywords.map(k => `<span style="padding:6px 14px;border-radius:20px;background:linear-gradient(135deg,#8b5cf6,#a78bfa);color:#fff;font-weight:600">${k}</span>`).join('')}</div>`, 'Keywords'), filename: `${base}_keywords_${ts}.html`, mime: 'text/html' };
    if (fmt === 'json') return { content: JSON.stringify({ type: 'keywords', keywords, timestamp: new Date().toISOString() }, null, 2), filename: `${base}_keywords_${ts}.json`, mime: 'application/json' };
    if (fmt === 'md') return { content: '# Keywords\n\n' + keywords.map(k => `- ${k}`).join('\n'), filename: `${base}_keywords_${ts}.md`, mime: 'text/markdown' };
    return { content: keywords.join('\n'), filename: `${base}_keywords_${ts}.txt`, mime: 'text/plain' };
}

function formatAssessment(faqRaw: string, fmt: string, base = 'assessment'): Formatted {
    const ts = new Date().toISOString().slice(0, 10);
    const items = parseFaq(faqRaw);
    if (fmt === 'json') return { content: JSON.stringify(items, null, 2), filename: `${base}_assessment_${ts}.json`, mime: 'application/json' };
    if (fmt === 'html') {
        const body = items.map((q, i) =>
            `<div style="background:#fff;border-radius:12px;padding:24px;margin-bottom:20px;box-shadow:0 2px 8px rgba(0,0,0,.08);border-left:4px solid #8b5cf6">
             <div style="font-weight:700;font-size:16px;margin-bottom:14px"><span style="background:linear-gradient(135deg,#8b5cf6,#a78bfa);color:#fff;padding:3px 10px;border-radius:6px;margin-right:10px;font-size:13px">Q${i + 1}</span>${q.question}</div>
             ${Object.entries(q.options).map(([k, v]) => `<div style="padding:10px 14px;margin:6px 0;border-radius:8px;border:1px solid ${k === q.correct_answer ? '#10b981' : '#e5e7eb'};background:${k === q.correct_answer ? 'rgba(16,185,129,.08)' : '#f9fafb'}"><b style="color:${k === q.correct_answer ? '#10b981' : '#8b5cf6'}">${k}.</b> ${v}${k === q.correct_answer ? ' <span style="background:#10b981;color:#fff;padding:1px 7px;border-radius:4px;font-size:11px">✓</span>' : ''}</div>`).join('')}
             <div style="margin-top:14px;padding:12px 16px;background:rgba(139,92,246,.06);border-left:3px solid #8b5cf6;border-radius:0 8px 8px 0;font-size:13px"><b style="color:#8b5cf6;font-size:11px;text-transform:uppercase">Explanation</b><br/>${q.explanation}</div>
             </div>`
        ).join('');
        return { content: wrapHtml(body, 'Assessment Questions'), filename: `${base}_assessment_${ts}.html`, mime: 'text/html' };
    }
    // txt
    const txt = items.map((q, i) =>
        `Q${i + 1}. ${q.question}\n` +
        Object.entries(q.options).map(([k, v]) => `  ${k}. ${v}${k === q.correct_answer ? ' ✓' : ''}`).join('\n') +
        `\n\nAnswer: ${q.correct_answer}\nExplanation: ${q.explanation}`
    ).join('\n\n' + '─'.repeat(60) + '\n\n');
    return { content: txt, filename: `${base}_assessment_${ts}.txt`, mime: 'text/plain' };
}

function formatShortAnswer(raw: string, fmt: string, base = 'short_answer'): Formatted {
    const ts = new Date().toISOString().slice(0, 10);
    const items = parsePyList<ShortAnswerItem>(raw);
    if (fmt === 'json') return { content: JSON.stringify(items, null, 2), filename: `${base}_${ts}.json`, mime: 'application/json' };
    if (fmt === 'html') {
        const body = items.map((item, i) =>
            `<div style="background:#fff;border-radius:12px;padding:20px;margin-bottom:16px;box-shadow:0 2px 8px rgba(0,0,0,.08);border-left:4px solid #4facfe">
             <div style="font-weight:700;font-size:15px;margin-bottom:10px"><span style="background:#4facfe;color:#fff;padding:3px 10px;border-radius:6px;margin-right:10px;font-size:13px">Q${i + 1}</span>${item.question}</div>
             <div style="padding:10px 14px;background:rgba(79,172,254,.08);border-radius:8px;font-size:13px"><b style="color:#4facfe">Answer:</b> ${item.answer}</div>
             </div>`
        ).join('');
        return { content: wrapHtml(body, 'Short Answer Questions'), filename: `${base}_${ts}.html`, mime: 'text/html' };
    }
    const txt = items.map((item, i) => `Q${i + 1}. ${item.question}\nA: ${item.answer}`).join('\n\n' + '─'.repeat(60) + '\n\n');
    return { content: txt, filename: `${base}_${ts}.txt`, mime: 'text/plain' };
}

function formatTrueFalse(raw: string, fmt: string, base = 'true_false'): Formatted {
    const ts = new Date().toISOString().slice(0, 10);
    const items = parsePyList<TrueFalseItem>(raw);
    if (fmt === 'json') return { content: JSON.stringify(items, null, 2), filename: `${base}_${ts}.json`, mime: 'application/json' };
    if (fmt === 'html') {
        const body = items.map((item, i) =>
            `<div style="background:#fff;border-radius:12px;padding:20px;margin-bottom:16px;box-shadow:0 2px 8px rgba(0,0,0,.08);border-left:4px solid #43e97b">
             <div style="font-weight:700;font-size:15px;margin-bottom:10px"><span style="background:#43e97b;color:#fff;padding:3px 10px;border-radius:6px;margin-right:10px;font-size:13px">${i + 1}</span>${item.statement}</div>
             <div style="padding:10px 14px;background:rgba(67,233,123,.08);border-radius:8px;font-size:13px"><b style="color:#43e97b">Answer:</b> ${item.is_true ? 'True' : 'False'}</div>
             <div style="margin-top:10px;font-size:13px;color:#666"><b>Explanation:</b> ${item.explanation}</div>
             </div>`
        ).join('');
        return { content: wrapHtml(body, 'True / False Questions'), filename: `${base}_${ts}.html`, mime: 'text/html' };
    }
    const txt = items.map((item, i) => `${i + 1}. ${item.statement}\nAnswer: ${item.is_true ? 'True' : 'False'}\nExplanation: ${item.explanation}`).join('\n\n' + '─'.repeat(60) + '\n\n');
    return { content: txt, filename: `${base}_${ts}.txt`, mime: 'text/plain' };
}

function formatTimestampedSummary(raw: string, fmt: string, base = 'timestamped_summary'): Formatted {
    const ts = new Date().toISOString().slice(0, 10);
    const items = parsePyList<TimestampSegment>(raw);
    if (fmt === 'json') return { content: JSON.stringify(items, null, 2), filename: `${base}_${ts}.json`, mime: 'application/json' };
    if (fmt === 'md') return { content: items.map(s => `### ${s.start_time} – ${s.end_time}\n\n${s.summary}`).join('\n\n'), filename: `${base}_${ts}.md`, mime: 'text/markdown' };
    const txt = items.map(s => `[${s.start_time} – ${s.end_time}]\n${s.summary}`).join('\n\n');
    return { content: txt, filename: `${base}_${ts}.txt`, mime: 'text/plain' };
}


const SUMMARY_FMTS = [{ k: 'txt', l: 'Text' }, { k: 'md', l: 'Markdown' }, { k: 'html', l: 'HTML' }, { k: 'json', l: 'JSON' }];
const KEYWORDS_FMTS = [{ k: 'txt', l: 'Text' }, { k: 'md', l: 'Markdown' }, { k: 'html', l: 'HTML' }, { k: 'json', l: 'JSON' }];
const ASSESSMENT_FMTS = [{ k: 'txt', l: 'Plain Text' }, { k: 'html', l: 'HTML' }, { k: 'json', l: 'JSON' }];
const SHORT_ANSWER_FMTS = [{ k: 'txt', l: 'Plain Text' }, { k: 'html', l: 'HTML' }, { k: 'json', l: 'JSON' }];
const TRUE_FALSE_FMTS = [{ k: 'txt', l: 'Plain Text' }, { k: 'html', l: 'HTML' }, { k: 'json', l: 'JSON' }];
const TIMESTAMPED_SUMMARY_FMTS = [{ k: 'txt', l: 'Text' }, { k: 'md', l: 'Markdown' }, { k: 'json', l: 'JSON' }];

// ── ActionBtn ─────────────────────────────────────────────
const ActionBtn: React.FC<{ title: string; onClick: () => void; active?: boolean; children: React.ReactNode }> = ({ title, onClick, active, children }) => (
    <button className={`${styles.actionBtn} ${active ? styles.actionBtnActive : ''}`} title={title} onClick={e => { e.stopPropagation(); onClick(); }}>
        {children}
    </button>
);

// ── FormatIcon ────────────────────────────────────────────
// Small colored glyph rendered next to each format option in the
// Copy / Download dropdowns. Each format has a recognisable hue.
const FORMAT_ICONS: Record<string, { color: string; bg: string; node: React.ReactNode }> = {
    txt: {
        color: '#64748b',
        bg: 'rgba(100, 116, 139, 0.12)',
        node: (
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                <path d="M3 2.5h6.5L13 6v7.5A1 1 0 0112 14.5H3a1 1 0 01-1-1v-10a1 1 0 011-1z" />
                <path d="M9 2.5V6h4" />
                <path d="M5 9h6M5 11.5h4" />
            </svg>
        ),
    },
    md: {
        color: '#3b82f6',
        bg: 'rgba(59, 130, 246, 0.14)',
        node: (
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                <rect x="1.5" y="3.5" width="13" height="9" rx="1.5" />
                <path d="M4 10.5V6l1.75 2.5L7.5 6v4.5" />
                <path d="M10.25 6v4.5M8.75 9l1.5 1.5L11.75 9" />
            </svg>
        ),
    },
    html: {
        color: '#e34f26',
        bg: 'rgba(227, 79, 38, 0.14)',
        node: (
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                <path d="M2.5 2l1 12 4.5 1.3L12.5 14l1-12z" />
                <path d="M5 5.5h6L10.6 11l-2.6.8L5.4 11l-.15-2" />
            </svg>
        ),
    },
    json: {
        color: '#f59e0b',
        bg: 'rgba(245, 158, 11, 0.14)',
        node: (
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                <path d="M6 2.5C4.5 2.5 4 3.5 4 5v1.5C4 7.5 3 8 2 8c1 0 2 .5 2 1.5V11c0 1.5.5 2.5 2 2.5" />
                <path d="M10 2.5c1.5 0 2 1 2 2.5v1.5C12 7.5 13 8 14 8c-1 0-2 .5-2 1.5V11c0 1.5-.5 2.5-2 2.5" />
            </svg>
        ),
    },
};

const FormatIcon: React.FC<{ fmt: string }> = ({ fmt }) => {
    const def = FORMAT_ICONS[fmt];
    if (!def) return null;
    return (
        <span
            className={styles.fmtIcon}
            style={{ color: def.color, background: def.bg }}
            aria-hidden="true"
        >
            {def.node}
        </span>
    );
};

// ── Dropdown ──────────────────────────────────────────────
const Dropdown: React.FC<{
    icon: React.ReactNode; title: string; label: string;
    fmts: { k: string; l: string }[];
    onSelect: (k: string) => void; active?: boolean;
}> = ({ icon, title, label, fmts, onSelect, active }) => {
    const [open, setOpen] = useState(false);
    const ref = useRef<HTMLDivElement>(null);
    useEffect(() => {
        const h = (e: MouseEvent) => { if (ref.current && !ref.current.contains(e.target as Node)) setOpen(false); };
        document.addEventListener('mousedown', h); return () => document.removeEventListener('mousedown', h);
    }, []);
    return (
        <div className={styles.dropdownWrap} ref={ref}>
            <ActionBtn title={title} onClick={() => setOpen(o => !o)} active={open || active}>{icon}</ActionBtn>
            {open && (
                <div className={styles.dropdown}>
                    <div className={styles.dropdownLabel}>{label}</div>
                    {fmts.map(f => (
                        <button
                            key={f.k}
                            className={styles.dropdownItem}
                            onClick={() => { onSelect(f.k); setOpen(false); }}
                        >
                            <FormatIcon fmt={f.k} />
                            <span className={styles.dropdownItemLabel}>{f.l}</span>
                        </button>
                    ))}
                </div>
            )}
        </div>
    );
};

// ── Icons ─────────────────────────────────────────────────
const IcoEdit = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <path d="M7 2.5H3.5A1.5 1.5 0 002 4v8.5A1.5 1.5 0 003.5 14H12a1.5 1.5 0 001.5-1.5V9" />
        <path d="M11.5 1.5a1.414 1.414 0 012 2L8 9l-2.5.5.5-2.5 5.5-5.5z" />
    </svg>
);
const IcoCopy = ({ success }: { success: boolean }) => success ? (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" className={styles.successIcon}><path d="M3 8l3.5 3.5L13 4" /></svg>
) : (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <rect x="5" y="5" width="8" height="9" rx="1.5" />
        <path d="M10 5V4a1 1 0 00-1-1H4a1.5 1.5 0 00-1.5 1.5V11a1 1 0 001 1h1" />
    </svg>
);
const IcoDownload = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <path d="M8 2v8M5 7l3 3 3-3" />
        <path d="M2 12v1.5A1.5 1.5 0 003.5 15h9a1.5 1.5 0 001.5-1.5V12" />
    </svg>
);

// ── TabToolbar: edit + copy + download ───────────────────
const TabToolbar: React.FC<{
    onEdit?: () => void;
    fmts: { k: string; l: string }[];
    onCopy: (k: string) => void;
    onDownload: (k: string) => void;
    copied: boolean;
}> = ({ onEdit, fmts, onCopy, onDownload, copied }) => {
    const { t } = useTranslation();
    return (
        <div className={styles.tabToolbar}>
            {onEdit && <ActionBtn title={t('uploadInfer.workspacePanel.edit')} onClick={onEdit}><IcoEdit /></ActionBtn>}
            <Dropdown icon={<IcoCopy success={copied} />} title={t('uploadInfer.workspacePanel.copy')} label={t('uploadInfer.workspacePanel.copyAs')} fmts={fmts} onSelect={onCopy} active={copied} />
            <Dropdown icon={<IcoDownload />} title={t('uploadInfer.workspacePanel.download')} label={t('uploadInfer.workspacePanel.downloadAs')} fmts={fmts} onSelect={onDownload} />
        </div>
    );
};

// ── Stable keyword chip ───────────────────────────────────
const KeywordChip: React.FC<{ kw: string; isNew: boolean }> = ({ kw, isNew }) => (
    <span className={`${styles.keywordPill} ${isNew ? styles.keywordPillNew : ''}`}
        style={{ background: CHIP_COLORS[seededColorIndex(kw)] }}>
        {kw}
    </span>
);

const ChipGrid: React.FC<{ kws: string[]; prevSet?: Set<string> }> = ({ kws, prevSet }) => {
    const { t } = useTranslation();
    return kws.length > 0 ? (
        <div className={styles.keywordGrid}>
            {kws.map(kw => <KeywordChip key={kw} kw={kw} isNew={prevSet ? !prevSet.has(kw) : false} />)}
        </div>
    ) : <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noKeywords')}</div>;
};

// ── Tab: Summary ──────────────────────────────────────────
const TabSummary: React.FC<{ summary: string; fileId: number; onSaved: (s: string) => void }> = ({ summary, fileId, onSaved }) => {
    const { t } = useTranslation();
    const [editing, setEditing] = useState(false);
    const [draft, setDraft] = useState(summary);
    const [saving, setSaving] = useState(false);
    const [copied, setCopied] = useState(false);

    const html = useMemo(() => marked.parse(draft) as string, [draft]);
    const viewHtml = useMemo(() => marked.parse(summary) as string, [summary]);

    const handleSave = async () => {
        setSaving(true);
        try { await api.post('/files/update', { fileID: String(fileId), summary: draft }); onSaved(draft); setEditing(false); }
        finally { setSaving(false); }
    };
    const handleCopy = (fmt: string) => { copyText(formatSummary(summary, fmt).content); setCopied(true); setTimeout(() => setCopied(false), 1500); };
    const handleDownload = (fmt: string) => { const f = formatSummary(summary, fmt); downloadFile(f.content, f.filename, f.mime); };

    if (!summary && !editing) return <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noSummary')}</div>;

    return (
        <div className={`${styles.tabContent} ${editing ? styles.editMode : ''}`}>
            {!editing ? (
                <>
                    <TabToolbar onEdit={() => { setDraft(summary); setEditing(true); }} fmts={SUMMARY_FMTS} onCopy={handleCopy} onDownload={handleDownload} copied={copied} />
                    <div className={styles.summaryMd} dangerouslySetInnerHTML={{ __html: viewHtml }} />
                </>
            ) : (
                <div className={styles.editLayout}>
                    <div className={styles.editCol}>
                        <div className={styles.editColHeader}><span className={styles.editColLabel}>{t('uploadInfer.workspacePanel.editColPreview')}</span><span className={styles.editColTag}>{t('uploadInfer.workspacePanel.markdownRenderer')}</span></div>
                        <div className={styles.editPreview}><div className={styles.summaryMd} dangerouslySetInnerHTML={{ __html: html }} /></div>
                    </div>
                    <div className={styles.editCol}>
                        <div className={styles.editColHeader}><span className={styles.editColLabel}>{t('uploadInfer.workspacePanel.editColEdit')}</span><span className={styles.editColTag}>{t('uploadInfer.workspacePanel.markdownTag')}</span></div>
                        <textarea className={styles.editTextarea} value={draft} onChange={e => setDraft(e.target.value)} spellCheck={false} />
                    </div>
                    <div className={styles.editFooter}>
                        <button className={styles.cancelBtn} onClick={() => setEditing(false)} disabled={saving}>{t('uploadInfer.workspacePanel.cancel')}</button>
                        <button className={styles.saveBtn} onClick={handleSave} disabled={saving}>
                            {saving ? <><span className={styles.inlineSpinner} />{t('uploadInfer.workspacePanel.saving')}</> : t('uploadInfer.workspacePanel.save')}
                        </button>
                    </div>
                </div>
            )}
        </div>
    );
};

// ── Tab: Keywords ─────────────────────────────────────────
const TabKeywords: React.FC<{ keywords: string[]; fileId: number; onSaved: (kws: string[]) => void }> = ({ keywords, fileId, onSaved }) => {
    const { t } = useTranslation();
    const [editing, setEditing] = useState(false);
    const [draft, setDraft] = useState(keywords.join('\n'));
    const [saving, setSaving] = useState(false);
    const [copied, setCopied] = useState(false);

    // prevSetSnapshot holds the keyword set from the PREVIOUS render of ChipGrid
    // so only truly new chips get the pop animation
    const prevSetSnapshot = useRef<Set<string>>(new Set(keywords));

    const draftKeywords = useMemo(() => draft.split('\n').map(k => k.trim()).filter(Boolean), [draft]);

    // After each render, schedule an update of the snapshot so the *next*
    // render can compare against what's currently visible
    useEffect(() => {
        const id = setTimeout(() => { prevSetSnapshot.current = new Set(draftKeywords); }, 0);
        return () => clearTimeout(id);
    }, [draftKeywords]);

    const handleSave = async () => {
        setSaving(true);
        try { await api.post('/files/update', { fileID: String(fileId), keywords: draftKeywords }); onSaved(draftKeywords); setEditing(false); }
        finally { setSaving(false); }
    };
    const handleCopy = (fmt: string) => { copyText(formatKeywords(keywords, fmt).content); setCopied(true); setTimeout(() => setCopied(false), 1500); };
    const handleDownload = (fmt: string) => { const f = formatKeywords(keywords, fmt); downloadFile(f.content, f.filename, f.mime); };

    if (!keywords.length && !editing) return <div className={styles.tabEmpty}>No keywords available.</div>;

    return (
        <div className={`${styles.tabContent} ${editing ? styles.editMode : ''}`}>
            {!editing ? (
                <>
                    <TabToolbar onEdit={() => { setDraft(keywords.join('\n')); prevSetSnapshot.current = new Set(keywords); setEditing(true); }} fmts={KEYWORDS_FMTS} onCopy={handleCopy} onDownload={handleDownload} copied={copied} />
                    <ChipGrid kws={keywords} />
                </>
            ) : (
                <div className={styles.editLayout}>
                    <div className={styles.editCol}>
                        <div className={styles.editColHeader}><span className={styles.editColLabel}>{t('uploadInfer.workspacePanel.editColPreview')}</span><span className={styles.editColTag}>{t('uploadInfer.workspacePanel.chipView')}</span></div>
                        <div className={styles.editPreview}>
                            <ChipGrid kws={draftKeywords} prevSet={prevSetSnapshot.current} />
                        </div>
                    </div>
                    <div className={styles.editCol}>
                        <div className={styles.editColHeader}><span className={styles.editColLabel}>{t('uploadInfer.workspacePanel.editColEdit')}</span><span className={styles.editColTag}>{t('uploadInfer.workspacePanel.onePerLine')}</span></div>
                        <textarea className={styles.editTextarea} value={draft} onChange={e => setDraft(e.target.value)} spellCheck={false} placeholder={t('uploadInfer.workspacePanel.keywordPlaceholder')} />
                    </div>
                    <div className={styles.editFooter}>
                        <button className={styles.cancelBtn} onClick={() => setEditing(false)} disabled={saving}>{t('uploadInfer.workspacePanel.cancel')}</button>
                        <button className={styles.saveBtn} onClick={handleSave} disabled={saving}>
                            {saving ? <><span className={styles.inlineSpinner} />{t('uploadInfer.workspacePanel.saving')}</> : t('uploadInfer.workspacePanel.save')}
                        </button>
                    </div>
                </div>
            )}
        </div>
    );
};

// ── Tab: Assessment ───────────────────────────────────────
type AssessMode = 'mcq' | 'all';

const TabAssessment: React.FC<{ faq: string }> = ({ faq }) => {
    const { t } = useTranslation();
    const items = useMemo(() => parseFaq(faq), [faq]);

    // Always default to MCQ; reset when faq changes (new file)
    const [mode, setMode] = useState<AssessMode>('mcq');
    const [current, setCurrent] = useState(0);
    const [selected, setSelected] = useState<string | null>(null);
    const [revealed, setRevealed] = useState(false);
    const [complete, setComplete] = useState(false);
    const [copied, setCopied] = useState(false);

    const prevFaq = useRef(faq);
    useEffect(() => {
        if (faq !== prevFaq.current) {
            prevFaq.current = faq;
            setMode('mcq');
            setCurrent(0); setSelected(null); setRevealed(false); setComplete(false);
        }
    }, [faq]);

    const handleCopy = (fmt: string) => { copyText(formatAssessment(faq, fmt).content); setCopied(true); setTimeout(() => setCopied(false), 1500); };
    const handleDownload = (fmt: string) => { const f = formatAssessment(faq, fmt); downloadFile(f.content, f.filename, f.mime); };

    if (!items.length) return <div className={`${styles.tabContent} ${styles.tabEmpty}`}>{t('uploadInfer.workspacePanel.noQuestions')}</div>;

    // ── MCQ helpers ───────────────────────────────────────
    const q = items[current];
    const isLast = current === items.length - 1;
    const progress = Math.round(((current + (complete ? 1 : 0)) / items.length) * 100);

    const handleSelect = (key: string) => { if (revealed) return; setSelected(key); setRevealed(true); };
    const handleNext = () => { if (isLast) setComplete(true); else { setCurrent(c => c + 1); setSelected(null); setRevealed(false); } };
    const handleRestart = () => { setCurrent(0); setSelected(null); setRevealed(false); setComplete(false); };
    const getOptClass = (key: string) => {
        if (!revealed) return selected === key ? styles.faqOptSelected : '';
        if (key === q.correct_answer && selected === key) return styles.faqOptCorrect;
        if (key === q.correct_answer) return styles.faqOptCorrectAlt;
        if (key === selected) return styles.faqOptWrong;
        return '';
    };

    return (
        <div className={styles.tabContent}>
            {/* Toolbar row: mode tabs left, copy/download right */}
            <div className={styles.assessHeader}>
                <div className={styles.assessModeTabs}>
                    <button
                        className={`${styles.assessModeTab} ${mode === 'mcq' ? styles.assessModeTabActive : ''}`}
                        onClick={() => setMode('mcq')}
                    >
                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                            <circle cx="8" cy="8" r="6" />
                            <path d="M6 8.5l1.5 1.5L10.5 6" />
                        </svg>
                        MCQ
                    </button>
                    <button
                        className={`${styles.assessModeTab} ${mode === 'all' ? styles.assessModeTabActive : ''}`}
                        onClick={() => setMode('all')}
                    >
                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                            <path d="M2 4h12M2 8h12M2 12h8" />
                        </svg>
                        View All
                    </button>
                </div>
                <TabToolbar fmts={ASSESSMENT_FMTS} onCopy={handleCopy} onDownload={handleDownload} copied={copied} />
            </div>

            {/* ── MCQ mode ── */}
            {mode === 'mcq' && (
                complete ? (
                    <div className={styles.assessComplete}>
                        <div className={styles.assessCompleteIcon}>
                            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                <path d="M9 12.75L11.25 15 15 9.75M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
                            </svg>
                        </div>
                        <div className={styles.assessCompleteTitle}>{t('uploadInfer.workspacePanel.assessCompleteTitle')}</div>
                        <div className={styles.assessCompleteDesc}>{t('uploadInfer.workspacePanel.assessCompleteDesc', { count: items.length })}</div>
                        <button className={styles.assessRestartBtn} onClick={handleRestart}>{t('uploadInfer.workspacePanel.restart')}</button>
                    </div>
                ) : (
                    <>
                        <div className={styles.assessProgress}>
                            <div className={styles.assessProgressTrack}>
                                <div className={styles.assessProgressFill} style={{ width: `${progress}%` }} />
                            </div>
                            <span className={styles.assessProgressLabel}>{current + 1} / {items.length}</span>
                        </div>
                        <div className={styles.faqCard}>
                            <div className={styles.faqQ}>
                                <span className={styles.faqNum}>Q{current + 1}</span>
                                {q.question}
                            </div>
                            <div className={styles.faqOptions}>
                                {Object.entries(q.options).map(([key, val]) => (
                                    <button key={key} className={`${styles.faqOpt} ${getOptClass(key)}`} onClick={() => handleSelect(key)} disabled={revealed}>
                                        <span className={styles.faqOptKey}>{key}</span>
                                        <span className={styles.faqOptVal}>{val}</span>
                                        {revealed && key === q.correct_answer && <svg className={styles.faqCheckIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round"><path d="M3 8l3.5 3.5L13 4" /></svg>}
                                        {revealed && key === selected && key !== q.correct_answer && <svg className={styles.faqXIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round"><path d="M4 4l8 8M12 4l-8 8" /></svg>}
                                    </button>
                                ))}
                            </div>
                            {revealed && (
                                <div className={styles.faqExplain}>
                                    <div className={styles.faqExplainLabel}>
                                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round"><circle cx="8" cy="8" r="6" /><path d="M8 7v4M8 5.5v.5" /></svg>
                                        Explanation
                                    </div>
                                    <p className={styles.faqExplainText}>{q.explanation}</p>
                                </div>
                            )}
                        </div>
                        {revealed && (
                            <button className={`${styles.assessNextBtn} ${isLast ? styles.assessDoneBtn : ''}`} onClick={handleNext}>
                                {isLast ? t('uploadInfer.workspacePanel.finish') : t('uploadInfer.workspacePanel.next')}
                                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round">
                                    {isLast ? <path d="M3 8l3.5 3.5L13 4" /> : <path d="M3 8h10M9 4l4 4-4 4" />}
                                </svg>
                            </button>
                        )}
                    </>
                )
            )}

            {/* ── View All mode ── */}
            {mode === 'all' && (
                <div className={styles.viewAllList}>
                    {items.map((item, idx) => (
                        <div key={idx} className={styles.viewAllCard}>
                            {/* Question */}
                            <div className={styles.viewAllQ}>
                                <span className={styles.faqNum}>Q{idx + 1}</span>
                                {item.question}
                            </div>

                            {/* Options */}
                            <div className={styles.faqOptions}>
                                {Object.entries(item.options).map(([key, val]) => (
                                    <div
                                        key={key}
                                        className={`${styles.faqOpt} ${key === item.correct_answer ? styles.faqOptCorrectAlt : styles.viewAllOptNeutral}`}
                                    >
                                        <span className={`${styles.faqOptKey} ${key === item.correct_answer ? styles.faqOptKeyCorrect : ''}`}>{key}</span>
                                        <span className={styles.faqOptVal}>{val}</span>
                                        {key === item.correct_answer && (
                                            <svg className={styles.faqCheckIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round">
                                                <path d="M3 8l3.5 3.5L13 4" />
                                            </svg>
                                        )}
                                    </div>
                                ))}
                            </div>

                            {/* Explanation */}
                            <div className={styles.faqExplain}>
                                <div className={styles.faqExplainLabel}>
                                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                                        <circle cx="8" cy="8" r="6" /><path d="M8 7v4M8 5.5v.5" />
                                    </svg>
                                    Explanation
                                </div>
                                <p className={styles.faqExplainText}>{item.explanation}</p>
                            </div>
                        </div>
                    ))}
                </div>
            )}
        </div>
    );
};

// ── Tab: Short Answer ──────────────────────────────────────
const TabShortAnswer: React.FC<{ raw: string }> = ({ raw }) => {
    const { t } = useTranslation();
    const items = useMemo(() => parsePyList<ShortAnswerItem>(raw), [raw]);
    const [revealedIds, setRevealedIds] = useState<Set<number>>(new Set());
    const [copied, setCopied] = useState(false);

    const prevRaw = useRef(raw);
    useEffect(() => { if (raw !== prevRaw.current) { prevRaw.current = raw; setRevealedIds(new Set()); } }, [raw]);

    const toggleReveal = (idx: number) => setRevealedIds(prev => {
        const next = new Set(prev);
        if (next.has(idx)) next.delete(idx); else next.add(idx);
        return next;
    });
    const allRevealed = items.length > 0 && revealedIds.size === items.length;
    const toggleAll = () => setRevealedIds(allRevealed ? new Set() : new Set(items.map((_, i) => i)));

    const handleCopy = (fmt: string) => { copyText(formatShortAnswer(raw, fmt).content); setCopied(true); setTimeout(() => setCopied(false), 1500); };
    const handleDownload = (fmt: string) => { const f = formatShortAnswer(raw, fmt); downloadFile(f.content, f.filename, f.mime); };

    if (!items.length) return <div className={`${styles.tabContent} ${styles.tabEmpty}`}>{t('uploadInfer.workspacePanel.noShortAnswer')}</div>;

    return (
        <div className={styles.tabContent}>
            <div className={styles.assessHeader}>
                <button className={styles.revealAllBtn} onClick={toggleAll}>
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                        <path d="M1.5 8s2.5-4.5 6.5-4.5S14.5 8 14.5 8s-2.5 4.5-6.5 4.5S1.5 8 1.5 8z" /><circle cx="8" cy="8" r="2" />
                    </svg>
                    {allRevealed ? t('uploadInfer.workspacePanel.hideAllAnswers') : t('uploadInfer.workspacePanel.showAllAnswers')}
                </button>
                <TabToolbar fmts={SHORT_ANSWER_FMTS} onCopy={handleCopy} onDownload={handleDownload} copied={copied} />
            </div>
            <div className={styles.viewAllList}>
                {items.map((item, idx) => {
                    const revealed = revealedIds.has(idx);
                    return (
                        <div key={idx} className={styles.viewAllCard}>
                            <div className={styles.viewAllQ}>
                                <span className={styles.faqNum}>Q{idx + 1}</span>
                                {item.question}
                            </div>
                            {revealed ? (
                                <div className={styles.shortAnswerBox}>
                                    <div className={styles.shortAnswerLabel}>{t('uploadInfer.workspacePanel.answer')}</div>
                                    <p className={styles.shortAnswerText}>{item.answer}</p>
                                </div>
                            ) : (
                                <button className={styles.revealBtn} onClick={() => toggleReveal(idx)}>
                                    {t('uploadInfer.workspacePanel.revealAnswer')}
                                </button>
                            )}
                        </div>
                    );
                })}
            </div>
        </div>
    );
};

// ── Tab: True / False ────────────────────────────────────────
type TfMode = 'quiz' | 'all';

const TabTrueFalse: React.FC<{ raw: string }> = ({ raw }) => {
    const { t } = useTranslation();
    const items = useMemo(() => parsePyList<TrueFalseItem>(raw), [raw]);

    const [mode, setMode] = useState<TfMode>('quiz');
    const [current, setCurrent] = useState(0);
    const [selected, setSelected] = useState<'true' | 'false' | null>(null);
    const [revealed, setRevealed] = useState(false);
    const [complete, setComplete] = useState(false);
    const [copied, setCopied] = useState(false);

    const prevRaw = useRef(raw);
    useEffect(() => {
        if (raw !== prevRaw.current) {
            prevRaw.current = raw;
            setMode('quiz');
            setCurrent(0); setSelected(null); setRevealed(false); setComplete(false);
        }
    }, [raw]);

    const handleCopy = (fmt: string) => { copyText(formatTrueFalse(raw, fmt).content); setCopied(true); setTimeout(() => setCopied(false), 1500); };
    const handleDownload = (fmt: string) => { const f = formatTrueFalse(raw, fmt); downloadFile(f.content, f.filename, f.mime); };

    if (!items.length) return <div className={`${styles.tabContent} ${styles.tabEmpty}`}>{t('uploadInfer.workspacePanel.noTrueFalse')}</div>;

    const q = items[current];
    const correctKey: 'true' | 'false' = q.is_true ? 'true' : 'false';
    const isLast = current === items.length - 1;
    const progress = Math.round(((current + (complete ? 1 : 0)) / items.length) * 100);

    const handleSelect = (key: 'true' | 'false') => { if (revealed) return; setSelected(key); setRevealed(true); };
    const handleNext = () => { if (isLast) setComplete(true); else { setCurrent(c => c + 1); setSelected(null); setRevealed(false); } };
    const handleRestart = () => { setCurrent(0); setSelected(null); setRevealed(false); setComplete(false); };
    const getOptClass = (key: 'true' | 'false') => {
        if (!revealed) return selected === key ? styles.faqOptSelected : '';
        if (key === correctKey && selected === key) return styles.faqOptCorrect;
        if (key === correctKey) return styles.faqOptCorrectAlt;
        if (key === selected) return styles.faqOptWrong;
        return '';
    };

    return (
        <div className={styles.tabContent}>
            <div className={styles.assessHeader}>
                <div className={styles.assessModeTabs}>
                    <button className={`${styles.assessModeTab} ${mode === 'quiz' ? styles.assessModeTabActive : ''}`} onClick={() => setMode('quiz')}>
                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                            <circle cx="8" cy="8" r="6" /><path d="M6 8.5l1.5 1.5L10.5 6" />
                        </svg>
                        {t('uploadInfer.workspacePanel.quizMode')}
                    </button>
                    <button className={`${styles.assessModeTab} ${mode === 'all' ? styles.assessModeTabActive : ''}`} onClick={() => setMode('all')}>
                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                            <path d="M2 4h12M2 8h12M2 12h8" />
                        </svg>
                        {t('uploadInfer.workspacePanel.viewAll')}
                    </button>
                </div>
                <TabToolbar fmts={TRUE_FALSE_FMTS} onCopy={handleCopy} onDownload={handleDownload} copied={copied} />
            </div>

            {mode === 'quiz' && (
                complete ? (
                    <div className={styles.assessComplete}>
                        <div className={styles.assessCompleteIcon}>
                            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                <path d="M9 12.75L11.25 15 15 9.75M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
                            </svg>
                        </div>
                        <div className={styles.assessCompleteTitle}>{t('uploadInfer.workspacePanel.assessCompleteTitle')}</div>
                        <div className={styles.assessCompleteDesc}>{t('uploadInfer.workspacePanel.assessCompleteDesc', { count: items.length })}</div>
                        <button className={styles.assessRestartBtn} onClick={handleRestart}>{t('uploadInfer.workspacePanel.restart')}</button>
                    </div>
                ) : (
                    <>
                        <div className={styles.assessProgress}>
                            <div className={styles.assessProgressTrack}><div className={styles.assessProgressFill} style={{ width: `${progress}%` }} /></div>
                            <span className={styles.assessProgressLabel}>{current + 1} / {items.length}</span>
                        </div>
                        <div className={styles.faqCard}>
                            <div className={styles.faqQ}>
                                <span className={styles.faqNum}>{current + 1}</span>
                                {q.statement}
                            </div>
                            <div className={styles.tfOptions}>
                                {(['true', 'false'] as const).map(key => (
                                    <button key={key} className={`${styles.tfOpt} ${getOptClass(key)}`} onClick={() => handleSelect(key)} disabled={revealed}>
                                        <span className={styles.tfOptLabel}>{key === 'true' ? t('uploadInfer.workspacePanel.true') : t('uploadInfer.workspacePanel.false')}</span>
                                        {revealed && key === correctKey && <svg className={styles.faqCheckIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round"><path d="M3 8l3.5 3.5L13 4" /></svg>}
                                        {revealed && key === selected && key !== correctKey && <svg className={styles.faqXIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round"><path d="M4 4l8 8M12 4l-8 8" /></svg>}
                                    </button>
                                ))}
                            </div>
                            {revealed && (
                                <div className={styles.faqExplain}>
                                    <div className={styles.faqExplainLabel}>
                                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round"><circle cx="8" cy="8" r="6" /><path d="M8 7v4M8 5.5v.5" /></svg>
                                        {t('uploadInfer.workspacePanel.explanation')}
                                    </div>
                                    <p className={styles.faqExplainText}>{q.explanation}</p>
                                </div>
                            )}
                        </div>
                        {revealed && (
                            <button className={`${styles.assessNextBtn} ${isLast ? styles.assessDoneBtn : ''}`} onClick={handleNext}>
                                {isLast ? t('uploadInfer.workspacePanel.finish') : t('uploadInfer.workspacePanel.next')}
                                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round">
                                    {isLast ? <path d="M3 8l3.5 3.5L13 4" /> : <path d="M3 8h10M9 4l4 4-4 4" />}
                                </svg>
                            </button>
                        )}
                    </>
                )
            )}

            {mode === 'all' && (
                <div className={styles.viewAllList}>
                    {items.map((item, idx) => {
                        const ck: 'true' | 'false' = item.is_true ? 'true' : 'false';
                        return (
                            <div key={idx} className={styles.viewAllCard}>
                                <div className={styles.viewAllQ}>
                                    <span className={styles.faqNum}>{idx + 1}</span>
                                    {item.statement}
                                </div>
                                <div className={styles.tfOptions}>
                                    {(['true', 'false'] as const).map(key => (
                                        <div key={key} className={`${styles.tfOpt} ${key === ck ? styles.faqOptCorrectAlt : styles.viewAllOptNeutral}`}>
                                            <span className={styles.tfOptLabel}>{key === 'true' ? t('uploadInfer.workspacePanel.true') : t('uploadInfer.workspacePanel.false')}</span>
                                            {key === ck && <svg className={styles.faqCheckIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round"><path d="M3 8l3.5 3.5L13 4" /></svg>}
                                        </div>
                                    ))}
                                </div>
                                <div className={styles.faqExplain}>
                                    <div className={styles.faqExplainLabel}>
                                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round"><circle cx="8" cy="8" r="6" /><path d="M8 7v4M8 5.5v.5" /></svg>
                                        {t('uploadInfer.workspacePanel.explanation')}
                                    </div>
                                    <p className={styles.faqExplainText}>{item.explanation}</p>
                                </div>
                            </div>
                        );
                    })}
                </div>
            )}
        </div>
    );
};

// ── Tab: Timestamped Summary ─────────────────────────────────
const TabTimestampedSummary: React.FC<{ raw: string }> = ({ raw }) => {
    const { t } = useTranslation();
    const items = useMemo(() => parsePyList<TimestampSegment>(raw), [raw]);
    const [copied, setCopied] = useState(false);

    const handleCopy = (fmt: string) => { copyText(formatTimestampedSummary(raw, fmt).content); setCopied(true); setTimeout(() => setCopied(false), 1500); };
    const handleDownload = (fmt: string) => { const f = formatTimestampedSummary(raw, fmt); downloadFile(f.content, f.filename, f.mime); };

    if (!items.length) return <div className={`${styles.tabContent} ${styles.tabEmpty}`}>{t('uploadInfer.workspacePanel.noTimestampedSummary')}</div>;

    return (
        <div className={styles.tabContent}>
            <TabToolbar fmts={TIMESTAMPED_SUMMARY_FMTS} onCopy={handleCopy} onDownload={handleDownload} copied={copied} />
            <div className={styles.tsList}>
                {items.map((seg, idx) => (
                    <div key={idx} className={styles.tsRow}>
                        <div className={styles.tsRail}>
                            <div className={styles.tsDot} />
                            {idx < items.length - 1 && <div className={styles.tsLine} />}
                        </div>
                        <div className={styles.tsCard}>
                            <div className={styles.tsRange}>{seg.start_time} <span className={styles.tsArrow}>→</span> {seg.end_time}</div>
                            <p className={styles.tsText}>{seg.summary}</p>
                        </div>
                    </div>
                ))}
            </div>
        </div>
    );
};


// ── Keyword Insights: shared node-link graph (used for Knowledge Graph & Prerequisites) ──
interface GNode { id: string; label: string; value?: number; }
interface GEdge { source: string; target: string; label?: string; }

function buildGraphNodes(nodes: GNode[], edges: GEdge[]): GNode[] {
    const norm = (s: string) => s.trim().toLowerCase();
    const byKey = new Map<string, GNode>();
    const byId = new Map<string, GNode>();
    nodes.forEach(n => {
        byKey.set(norm(n.id), n); byKey.set(norm(n.label), n); byId.set(n.id, n);
    });
    edges.forEach(e => {
        [e.source, e.target].forEach(ref => {
            const k = norm(ref);
            if (!byKey.has(k)) {
                const synth: GNode = { id: ref, label: ref, value: 1 };
                byKey.set(k, synth); byId.set(ref, synth);
            }
        });
    });
    return Array.from(byId.values());
}

// Custom node — a labeled circle sized by `value`, with an invisible
// handle on all 4 sides (each doubling as source+target) so edges can
// connect from whichever side actually faces the other node.
const MindMapNode: React.FC<{ data: { label: string; value?: number } }> = ({ data }) => {
    const size = 34 + Math.min(26, (data.value ?? 1) * 4);
    return (
        <div className={styles.rfNode} style={{ width: size, height: size }} title={data.label}>
            <Handle type="target" position={Position.Top} id="top-target" className={styles.rfHandle} />
            <Handle type="source" position={Position.Top} id="top-source" className={styles.rfHandle} />
            <Handle type="target" position={Position.Right} id="right-target" className={styles.rfHandle} />
            <Handle type="source" position={Position.Right} id="right-source" className={styles.rfHandle} />
            <Handle type="target" position={Position.Bottom} id="bottom-target" className={styles.rfHandle} />
            <Handle type="source" position={Position.Bottom} id="bottom-source" className={styles.rfHandle} />
            <Handle type="target" position={Position.Left} id="left-target" className={styles.rfHandle} />
            <Handle type="source" position={Position.Left} id="left-source" className={styles.rfHandle} />
            <span className={styles.rfNodeLabel}>{data.label}</span>
        </div>
    );
};
const RF_NODE_TYPES = { mindmap: MindMapNode };

type Side = 'top' | 'right' | 'bottom' | 'left';
const OPPOSITE_SIDE: Record<Side, Side> = { top: 'bottom', bottom: 'top', left: 'right', right: 'left' };
function pickHandleSide(sx: number, sy: number, tx: number, ty: number): Side {
    const deg = (Math.atan2(ty - sy, tx - sx) * 180) / Math.PI;
    if (deg > -45 && deg <= 45) return 'right';
    if (deg > 45 && deg <= 135) return 'bottom';
    if (deg > 135 || deg <= -135) return 'left';
    return 'top';
}

const NodeLinkGraph: React.FC<{ nodes: GNode[]; edges: GEdge[] }> = ({ nodes, edges }) => {
    const { t } = useTranslation();
    const allNodes = useMemo(() => buildGraphNodes(nodes, edges), [nodes, edges]);

    const initial = useMemo(() => {
        const W = 700, H = 460, cx = W / 2, cy = H / 2;
        const r = Math.min(W, H) / 2 - 70;
        const positioned = allNodes.map((n, i) => {
            const angle = (2 * Math.PI * i) / Math.max(1, allNodes.length) - Math.PI / 2;
            return { ...n, x: cx + r * Math.cos(angle), y: cy + r * Math.sin(angle) };
        });
        const posByKey = new Map<string, typeof positioned[number]>();
        positioned.forEach(p => { posByKey.set(p.id.trim().toLowerCase(), p); posByKey.set(p.label.trim().toLowerCase(), p); });

        const rfNodes: RFNode[] = positioned.map(n => ({
            id: n.id,
            type: 'mindmap',
            position: { x: n.x, y: n.y },
            data: { label: n.label, value: n.value },
        }));

        const rfEdges: RFEdge[] = [];
        edges.forEach((e, i) => {
            const s = posByKey.get(e.source.trim().toLowerCase());
            const tg = posByKey.get(e.target.trim().toLowerCase());
            if (!s || !tg) return;
            const side = pickHandleSide(s.x, s.y, tg.x, tg.y);
            rfEdges.push({
                id: `e${i}-${s.id}-${tg.id}`,
                source: s.id,
                target: tg.id,
                sourceHandle: `${side}-source`,
                targetHandle: `${OPPOSITE_SIDE[side]}-target`,
                type: 'straight',
                label: e.label,
                style: { stroke: 'var(--bdr2)' },
                labelStyle: { fill: 'var(--t2)', fontSize: 10 },
                labelBgStyle: { fill: 'var(--bg1)' },
            });
        });
        return { rfNodes, rfEdges };
    }, [allNodes, edges]);

    const [rfNodes, setRfNodes] = useState<RFNode[]>(initial.rfNodes);
    useEffect(() => { setRfNodes(initial.rfNodes); }, [initial.rfNodes]);
    const onNodesChange = useCallback((changes: NodeChange[]) => setRfNodes(nds => applyNodeChanges(changes, nds)), []);

    if (allNodes.length === 0) return <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>;

    return (
        <div className={styles.graphWrap}>
            <ReactFlow
                nodes={rfNodes}
                edges={initial.rfEdges}
                onNodesChange={onNodesChange}
                nodeTypes={RF_NODE_TYPES}
                fitView
                minZoom={0.3}
                maxZoom={2}
                proOptions={{ hideAttribution: true }}
            >
                <Background gap={16} size={1} color="var(--bdr)" />
                <Controls showInteractive={false} />
                {allNodes.length > 8 && <MiniMap pannable zoomable style={{ background: 'var(--bg0)' }} />}
            </ReactFlow>
        </div>
    );
};

// ── Keyword Insights: shared matrix/heatmap grid (Timeline, Heatmap, Co-occurrence) ──
const MatrixGrid: React.FC<{ rowLabels: string[]; colLabels: string[]; matrix: number[][] }> = ({ rowLabels, colLabels, matrix }) => {
    const { t } = useTranslation();
    if (!rowLabels.length || !colLabels.length) return <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>;
    const max = Math.max(1, ...matrix.flat().filter(v => typeof v === 'number'));
    return (
        <div className={styles.matrixScroll}>
            <div className={styles.matrixGrid} style={{ gridTemplateColumns: `140px repeat(${colLabels.length}, minmax(28px, 1fr))` }}>
                <div className={styles.matrixCorner} />
                {colLabels.map((c, i) => <div key={i} className={styles.matrixColHead} title={c}>{c}</div>)}
                {rowLabels.map((r, ri) => (
                    <React.Fragment key={ri}>
                        <div className={styles.matrixRowHead} title={r}>{r}</div>
                        {colLabels.map((c, ci) => {
                            const v = matrix[ri]?.[ci] ?? 0;
                            const alpha = v > 0 ? Math.min(1, 0.22 + (v / max) * 0.78) : 0;
                            return <div key={ci} className={styles.matrixCell} style={{ background: `rgba(91, 164, 239, ${alpha})` }} title={`${r} · ${c}: ${v}`} />;
                        })}
                    </React.Fragment>
                ))}
            </div>
        </div>
    );
};

// ── Keyword Insights: word cloud ──
const WordCloudView: React.FC<{ data: WordCloudData }> = ({ data }) => {
    const { t } = useTranslation();
    const entries = data.wordcloud ?? [];
    if (!entries.length) return <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>;
    const counts = entries.map(([, c]) => c);
    const min = Math.min(...counts), max = Math.max(...counts);
    const scale = (c: number) => (min === max ? 20 : 13 + ((c - min) / (max - min)) * 24);
    const complexityColor = (word: string) => {
        const lvl = (data.complexity_map?.[word] || '').toLowerCase();
        if (lvl === 'easy') return 'var(--green)';
        if (lvl === 'medium') return 'var(--amber)';
        if (lvl === 'hard') return 'var(--red)';
        return 'var(--blue)';
    };
    return (
        <div className={styles.wordCloud}>
            {entries.map(([word, count], i) => (
                <span key={i} className={styles.wcWord} style={{ fontSize: `${scale(count)}px`, color: complexityColor(word) }} title={`${word}: ${count}`}>
                    {word}
                </span>
            ))}
        </div>
    );
};

// ── Keyword Insights: clusters ──
const ClustersView: React.FC<{ data: ClustersData }> = ({ data }) => {
    const { t } = useTranslation();
    const entries = Object.entries(data.clusters ?? {});
    if (!entries.length) return <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>;
    return (
        <div className={styles.clusterGrid}>
            {entries.map(([key, items], i) => (
                <div key={key + i} className={styles.clusterCard}>
                    <div className={styles.clusterTitle}>{key}</div>
                    <div className={styles.clusterChips}>
                        {items.map((it, j) => <span key={j} className={styles.clusterChip} title={`value: ${it.value}`}>{it.label}</span>)}
                    </div>
                </div>
            ))}
        </div>
    );
};

// ── Keyword Insights: frequency bars ──
const FrequencyBars: React.FC<{ data: FrequencyData }> = ({ data }) => {
    const { t } = useTranslation();
    const items = data.data ?? [];
    if (!items.length) return <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>;
    const sorted = [...items].sort((a, b) => b.relative_pct - a.relative_pct);
    return (
        <div className={styles.freqList}>
            {sorted.map((it, i) => (
                <div key={i} className={styles.freqRow}>
                    <div className={styles.freqLabel} title={it.keyword}>{it.keyword}</div>
                    <div className={styles.freqBarTrack}><div className={styles.freqBarFill} style={{ width: `${Math.min(100, it.relative_pct)}%` }} /></div>
                    <div className={styles.freqMeta}>
                        <span>{it.count}×</span>
                        <span className={styles.freqDim}>{t('uploadInfer.workspacePanel.firstMentionAt', { pct: Math.round(it.first_mention_pct) })}</span>
                    </div>
                </div>
            ))}
        </div>
    );
};

// ── Keyword Insights: importance vs complexity scatter ──
const ImportanceComplexityScatter: React.FC<{ data: ImportanceComplexityData }> = ({ data }) => {
    const { t } = useTranslation();
    const items = data.data ?? [];
    if (!items.length) return <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>;

    const order = ['Easy', 'Medium', 'Hard'];
    const complexities = Array.from(new Set(items.map(it => it.complexity)))
        .sort((a, b) => {
            const ai = order.indexOf(a), bi = order.indexOf(b);
            if (ai === -1 && bi === -1) return a.localeCompare(b);
            if (ai === -1) return 1;
            if (bi === -1) return -1;
            return ai - bi;
        });

    const W = 580, H = 320, padL = 20, padR = 20, padT = 24, padB = 36;
    const maxImportance = Math.max(1, ...items.map(it => it.importance));
    const maxFreq = Math.max(1, ...items.map(it => it.frequency));
    const xFor = (c: string) => {
        const idx = complexities.indexOf(c);
        const slot = (W - padL - padR) / Math.max(1, complexities.length);
        return padL + slot * (idx + 0.5);
    };
    const yFor = (imp: number) => H - padB - (imp / maxImportance) * (H - padB - padT);

    return (
        <div className={styles.scatterWrap}>
            <svg viewBox={`0 0 ${W} ${H}`} className={styles.scatterSvg}>
                <line x1={padL} y1={H - padB} x2={W - padR} y2={H - padB} className={styles.scatterAxis} />
                <line x1={padL} y1={padT} x2={padL} y2={H - padB} className={styles.scatterAxis} />
                {complexities.map((c, i) => (
                    <text key={i} x={xFor(c)} y={H - padB + 18} textAnchor="middle" className={styles.scatterAxisLabel}>{c}</text>
                ))}
                <text x={padL} y={padT - 8} className={styles.scatterAxisLabel}>{t('uploadInfer.workspacePanel.importanceAxis')}</text>
                {items.map((it, i) => {
                    const rad = 5 + (it.frequency / maxFreq) * 9;
                    return (
                        <g key={i}>
                            <circle cx={xFor(it.complexity)} cy={yFor(it.importance)} r={rad} className={styles.scatterDot}>
                                <title>{`${it.keyword} — importance ${it.importance}, ${it.complexity}, freq ${it.frequency}`}</title>
                            </circle>
                            <text x={xFor(it.complexity)} y={yFor(it.importance) - rad - 5} textAnchor="middle" className={styles.scatterDotLabel}>{it.keyword}</text>
                        </g>
                    );
                })}
            </svg>
        </div>
    );
};

// ── Keyword Insights: glossary ──
const GlossaryView: React.FC<{ data: GlossaryData }> = ({ data }) => {
    const { t } = useTranslation();
    const items = data.glossary ?? [];
    if (!items.length) return <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>;
    const fmtTime = (ms: number) => {
        const totalSec = Math.floor(ms / 1000);
        const m = Math.floor(totalSec / 60), s = totalSec % 60;
        return `${m}:${String(s).padStart(2, '0')}`;
    };
    return (
        <div className={styles.glossaryList}>
            {items.map((it, i) => (
                <div key={i} className={styles.glossaryRow}>
                    <div className={styles.glossaryTerm}>{it.term}<span className={styles.glossaryTime}>{fmtTime(it.first_mention_ms)}</span></div>
                    <div className={styles.glossaryDef}>{it.definition}</div>
                </div>
            ))}
        </div>
    );
};

// ── Tab: Keyword Insights ────────────────────────────────────
const KI_SUBTABS = ['graph', 'wordcloud', 'timeline', 'heatmap', 'clusters', 'frequency', 'prerequisites', 'importance', 'cooccurrence', 'glossary'] as const;
type KiSubTab = typeof KI_SUBTABS[number];

const TabKeywordInsights: React.FC<{ data: KeywordInsights | null }> = ({ data }) => {
    const { t } = useTranslation();
    const [sub, setSub] = useState<KiSubTab>('graph');

    if (!data) return <div className={`${styles.tabContent} ${styles.tabEmpty}`}>{t('uploadInfer.workspacePanel.noKeywordInsights')}</div>;

    return (
        <div className={styles.tabContent}>
            <ScrollableTabRow
                ids={KI_SUBTABS}
                activeId={sub}
                onChange={setSub}
                labelFor={id => t(`uploadInfer.workspacePanel.kiTabs.${id}`)}
                itemClassName={styles.kiSubTab}
                activeClassName={styles.kiSubTabActive}
                wrapClassName={styles.kiSubNavWrap}
                trackClassName={styles.kiSubNav}
            />
            <div className={styles.kiBody}>
                {sub === 'graph' && (data.knowledge_graph
                    ? <NodeLinkGraph nodes={data.knowledge_graph.nodes} edges={data.knowledge_graph.edges.map(e => ({ source: e.source, target: e.target, label: e.type }))} />
                    : <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>)}
                {sub === 'wordcloud' && (data.word_cloud ? <WordCloudView data={data.word_cloud} /> : <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>)}
                {sub === 'timeline' && (data.timeline
                    ? <MatrixGrid rowLabels={data.timeline.datasets.map(d => d.label)} colLabels={data.timeline.labels} matrix={data.timeline.datasets.map(d => d.data)} />
                    : <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>)}
                {sub === 'heatmap' && (data.heatmap
                    ? <MatrixGrid rowLabels={data.heatmap.keywords} colLabels={data.heatmap.segments} matrix={data.heatmap.matrix} />
                    : <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>)}
                {sub === 'clusters' && (data.clusters ? <ClustersView data={data.clusters} /> : <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>)}
                {sub === 'frequency' && (data.frequency ? <FrequencyBars data={data.frequency} /> : <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>)}
                {sub === 'prerequisites' && (data.prerequisites
                    ? <NodeLinkGraph nodes={data.prerequisites.nodes} edges={data.prerequisites.edges.map(e => ({ source: e.prerequisite, target: e.enables, label: e.reason }))} />
                    : <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>)}
                {sub === 'importance' && (data.importance_complexity ? <ImportanceComplexityScatter data={data.importance_complexity} /> : <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>)}
                {sub === 'cooccurrence' && (data.cooccurrence
                    ? <MatrixGrid rowLabels={data.cooccurrence.keywords} colLabels={data.cooccurrence.keywords} matrix={data.cooccurrence.metrix} />
                    : <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>)}
                {sub === 'glossary' && (data.glossary ? <GlossaryView data={data.glossary} /> : <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>)}
            </div>
        </div>
    );
};

// ── Scrollable tab row — generic horizontal-scroll tab strip with arrow
//    buttons + edge fades, reused for both the main tab bar and any
//    sub-navigation row (e.g. Keyword Insights) that could overflow. ──
function ScrollableTabRow<T extends string>({
    ids, activeId, onChange, labelFor,
    itemClassName, activeClassName, wrapClassName, trackClassName,
}: {
    ids: readonly T[];
    activeId: T;
    onChange: (id: T) => void;
    labelFor: (id: T) => string;
    itemClassName: string;
    activeClassName: string;
    wrapClassName: string;
    trackClassName: string;
}) {
    const trackRef = useRef<HTMLDivElement>(null);
    const [canScrollLeft, setCanScrollLeft] = useState(false);
    const [canScrollRight, setCanScrollRight] = useState(false);

    const updateScrollState = () => {
        const el = trackRef.current;
        if (!el) return;
        setCanScrollLeft(el.scrollLeft > 2);
        setCanScrollRight(el.scrollLeft < el.scrollWidth - el.clientWidth - 2);
    };

    useEffect(() => {
        updateScrollState();
        const el = trackRef.current;
        if (!el) return;
        const onResize = () => updateScrollState();
        window.addEventListener('resize', onResize);
        el.addEventListener('scroll', updateScrollState);
        return () => { window.removeEventListener('resize', onResize); el.removeEventListener('scroll', updateScrollState); };
    }, []); // eslint-disable-line

    // Keep the active item in view when it changes
    useEffect(() => {
        const el = trackRef.current;
        if (!el) return;
        const activeEl = el.querySelector<HTMLElement>(`[data-tab-id="${activeId}"]`);
        activeEl?.scrollIntoView({ behavior: 'smooth', block: 'nearest', inline: 'nearest' });
        const id = setTimeout(updateScrollState, 300);
        return () => clearTimeout(id);
    }, [activeId]); // eslint-disable-line

    const scrollBy = (dx: number) => trackRef.current?.scrollBy({ left: dx, behavior: 'smooth' });

    return (
        <div className={wrapClassName}>
            {canScrollLeft && (
                <button className={`${styles.tabScrollBtn} ${styles.tabScrollBtnLeft}`} onClick={() => scrollBy(-160)} aria-label="Scroll left">
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round"><path d="M10 3L6 8l4 5" /></svg>
                </button>
            )}
            <div className={trackClassName} ref={trackRef}>
                {ids.map(id => (
                    <button
                        key={id}
                        data-tab-id={id}
                        className={`${itemClassName} ${activeId === id ? activeClassName : ''}`}
                        onClick={() => onChange(id)}
                    >
                        {labelFor(id)}
                    </button>
                ))}
            </div>
            {canScrollLeft && <div className={`${styles.tabFade} ${styles.tabFadeLeft}`} />}
            {canScrollRight && <div className={`${styles.tabFade} ${styles.tabFadeRight}`} />}
            {canScrollRight && (
                <button className={`${styles.tabScrollBtn} ${styles.tabScrollBtnRight}`} onClick={() => scrollBy(160)} aria-label="Scroll right">
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round"><path d="M6 3l4 5-4 5" /></svg>
                </button>
            )}
        </div>
    );
}

// ── Main tab bar — the fixed-width flex row silently clipped tabs once
//    there were more than ~3; ScrollableTabRow fixes that. ──
const ScrollableTabs: React.FC<{ activeTab: TabId; onChange: (id: TabId) => void }> = ({ activeTab, onChange }) => {
    const { t } = useTranslation();
    return (
        <ScrollableTabRow
            ids={TAB_IDS}
            activeId={activeTab}
            onChange={onChange}
            labelFor={id => t(`uploadInfer.workspacePanel.tabs.${id}`)}
            itemClassName={styles.tab}
            activeClassName={styles.active}
            wrapClassName={styles.wsptabsWrap}
            trackClassName={styles.wsptabs}
        />
    );
};

// ── Main panel ────────────────────────────────────────────
const WorkspacePanel: React.FC<Props> = ({ step2Visible = true, step2Minimized = false, fileResult, fileLoading, activeFileId, onResultUpdate }) => {
    const { t } = useTranslation();
    const [activeTab, setActiveTab] = useState<TabId>('summary');

    return (
        <div className={`${styles.wspanel} ${(!step2Visible || step2Minimized) ? styles.wspanelExpanded : ''} ${step2Visible ? styles.wspanelWithStep2 : ''}`}>
            <div className={styles.wspanelHead}>
                <div className={styles.headLeft}>
                    {fileResult ? (
                        <>
                            <div className={styles.wsftitle}>{fileResult.fileName}</div>
                            <div className={styles.wsmeta}>
                                <span className={styles.wsmetaId}>#{fileResult.fileId}</span>
                                {fileResult.insertedAt && (<><span className={styles.wsmetaSep}>·</span><span className={styles.wsmetaDate}>{fileResult.insertedAt}</span></>)}
                            </div>
                        </>
                    ) : (
                        <>
                            <div className={styles.wsftitleEmpty}>—</div>
                            <div className={styles.wsmeta}><span className={styles.wsmetaHint}>{t('uploadInfer.workspacePanel.clickToView')}</span></div>
                        </>
                    )}
                </div>
            </div>

            <ScrollableTabs activeTab={activeTab} onChange={setActiveTab} />

            <div className={styles.wsbody}>
                {fileLoading && (
                    <div className={styles.wsSpinner}><div className={styles.spinner} /><span>{t('uploadInfer.workspacePanel.loadingFile')}</span></div>
                )}
                {!fileLoading && !fileResult && (
                    <div className={styles.wsEmpty}>
                        <svg viewBox="0 0 48 48" fill="none" stroke="currentColor" strokeWidth="1.2" strokeLinecap="round" strokeLinejoin="round">
                            <rect x="8" y="6" width="32" height="36" rx="3" /><path d="M16 16h16M16 23h16M16 30h10" />
                        </svg>
                        <div className={styles.wsEmptyTitle}>{t('uploadInfer.workspacePanel.noFileSelected')}</div>
                        <div className={styles.wsEmptyDesc}>{t('uploadInfer.workspacePanel.noFileDesc')}</div>
                    </div>
                )}
                {!fileLoading && fileResult && (
                    <>
                        {activeTab === 'summary' && <TabSummary summary={fileResult.summary} fileId={fileResult.fileId} onSaved={s => onResultUpdate?.({ summary: s })} />}
                        {activeTab === 'keywords' && <TabKeywords keywords={fileResult.keywords} fileId={fileResult.fileId} onSaved={kw => onResultUpdate?.({ keywords: kw })} />}
                        {activeTab === 'assessment' && <TabAssessment faq={fileResult.faq} />}
                        {activeTab === 'shortAnswer' && <TabShortAnswer raw={fileResult.shortAnswer} />}
                        {activeTab === 'trueFalse' && <TabTrueFalse raw={fileResult.trueFalse} />}
                        {activeTab === 'timestampedSummary' && <TabTimestampedSummary raw={fileResult.timestampedSummary} />}
                        {activeTab === 'keywordInsights' && <TabKeywordInsights data={fileResult.keywordInsights} />}
                    </>
                )}
            </div>
        </div>
    );
};

export default WorkspacePanel;
