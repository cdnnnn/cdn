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
  type FilesSortBy, type SortOrder, type FileStatus, type ServerFile,
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

// ── Status badge metadata (covers every value /files/by-date/ can return) ──
const STATUS_META: Record<FileStatus, { labelKey: string; cls: string }> = {
  completed: { labelKey: 'uploadInfer.filePanel.statusCompleted', cls: 'bInferred' },
  running: { labelKey: 'uploadInfer.filePanel.statusRunning', cls: 'bRunning' },
  queued: { labelKey: 'uploadInfer.filePanel.statusQueued', cls: 'bQueued' },
  error: { labelKey: 'uploadInfer.filePanel.statusError', cls: 'bError' },
  waiting: { labelKey: 'uploadInfer.filePanel.statusWaiting', cls: 'bNotInferred' },
  tbd: { labelKey: 'uploadInfer.filePanel.notInferenced', cls: 'bNotInferred' },
};

// ── Which of the 5 content prompts are customized for this file ──
const PROMPT_FIELDS: { key: keyof ServerFile; letter: string; labelKey: string }[] = [
  { key: 'summary_prompt', letter: 'S', labelKey: 'uploadInfer.inferencePanel.summaryPrompt' },
  { key: 'keywords_prompt', letter: 'K', labelKey: 'uploadInfer.inferencePanel.keywordPrompt' },
  { key: 'faq_prompt', letter: 'Q', labelKey: 'uploadInfer.inferencePanel.questionPrompt' },
  { key: 'short_answer_prompt', letter: 'A', labelKey: 'uploadInfer.inferencePanel.shortAnswerPrompt' },
  { key: 'true_false_prompt', letter: 'T', labelKey: 'uploadInfer.inferencePanel.trueFalsePrompt' },
];

const filesToBrowsed = (files: File[]): BrowsedFile[] =>
  files.filter(f => isAllowed(f.name)).map(f => ({
    id: uid(), file: f, name: f.name, size: formatSize(f.size), ext: getExt(f.name), status: 'pending' as UploadStatus,
  }));

// Fixed 4-button sliding window around the current page — e.g. current=57,
// total=200 → [56, 57, 58, 59]. Keeps the row compact and predictable
// instead of a variable count with ellipsis gaps.
const PAGE_WINDOW = 4;
const getPageNumbers = (current: number, total: number): number[] => {
  if (total <= PAGE_WINDOW) return Array.from({ length: total }, (_, i) => i + 1);
  const start = Math.max(1, Math.min(current - 1, total - PAGE_WINDOW + 1));
  return Array.from({ length: PAGE_WINDOW }, (_, i) => start + i);
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
  // Optional — lets the host switch to the Run inference tab once files
  // are picked here. FilePanel works fine without it (just no shortcut).
  onGoToInfer?: () => void;
}

const FilePanel: React.FC<FilePanelProps> = ({ selectMode, onEnterSelectMode, onExitSelectMode, onFileClick, onDeleteComplete, activeFileId, onGoToInfer }) => {
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
        <div className={styles.step1Card}>
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
              {uploadZoneCollapsed && (
                <button
                  className={styles.reopenUploadBtn}
                  onClick={() => dispatch(toggleUploadZone())}
                  title={t('uploadInfer.filePanel.showUpload')}
                >
                  <IconChevron />
                  {t('uploadInfer.filePanel.showUpload')}
                </button>
              )}
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

          {/* ── Select mode action row (choosing files for inference) ── */}
          {selectMode && !isBatchRunning && (
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
              </div>
              <div className={styles.modeActionsRight}>
                <button
                  className={`${styles.modeTile} ${styles.modeTileConfirmExport} ${selectedServerIds.length === 0 ? styles.modeTileDisabled : ''}`}
                  onClick={onGoToInfer}
                  disabled={selectedServerIds.length === 0 || !onGoToInfer}
                >
                  <svg viewBox="0 0 16 16" fill="currentColor" stroke="none" width="12" height="12">
                    <path d="M8 1l1.8 4.4L14 6.2l-3.3 2.5 1.2 4.3L8 10.8 4.1 13l1.2-4.3L2 6.2l4.2-.8z" />
                  </svg>
                  {selectedServerIds.length > 0
                    ? t('uploadInfer.filePanel.continueToInferCount', { count: selectedServerIds.length })
                    : t('uploadInfer.filePanel.continueToInfer')}
                </button>
                <button
                  className={`${styles.modeTile} ${styles.modeTileCancel}`}
                  onClick={exitSelectMode}
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

        </div>

        <div className={styles.filterSortRow}>
          <div className={styles.dateSortLine}>
            <div className={styles.dateFilter}>
              <svg className={styles.dateIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                <rect x="2" y="3" width="12" height="11" rx="1.5" />
                <path d="M2 6.5h12M5 2v2.5M11 2v2.5" />
              </svg>
              <input type="date" className={styles.dateInput} value={dateFrom} max={dateTo}
                disabled={isBatchRunning}
                aria-label={t('uploadInfer.filePanel.dateFrom')}
                onChange={e => dispatch(setDateFrom(e.target.value))} />
              <span className={styles.dateSep}>–</span>
              <input type="date" className={styles.dateInput} value={dateTo} min={dateFrom}
                disabled={isBatchRunning}
                aria-label={t('uploadInfer.filePanel.dateTo')}
                onChange={e => dispatch(setDateTo(e.target.value))} />
              <button className={styles.applyBtn} onClick={handleApply} disabled={filesLoading || isBatchRunning}>{t('uploadInfer.filePanel.applyDate')}</button>
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
          </div>

          {/* ── Action toolbar — its own line, only in normal (idle) mode ── */}
          {!selectMode && !deleteMode && !exportMode && !isBatchRunning && (
            <>
              <div className={styles.filterSortHDivider} />
              <div className={styles.actionsGrid}>
                <span className={styles.actionsLabel}>
                  <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                    <path d="M7 1.5l1.4 3.4L12 6l-3.6 1.4L7 12.5l-1.4-4.1L2 7l3.6-1.1z" />
                  </svg>
                  {t('uploadInfer.filePanel.actionsLabel')}
                </span>

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
            </>
          )}
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
            const checked = selectable ? isChecked : deletable ? isDeleteChecked : exportable ? isExportChecked : false;
            const toggle = () => {
              if (selectable) dispatch(toggleServerFileSelection(f.id));
              else if (deletable) toggleDeleteSelection(f.id);
              else if (exportable) toggleExportSelection(f.id);
              else onFileClick(f.id);
            };
            const statusMeta = STATUS_META[f.status] ?? STATUS_META.tbd;
            const hasLinks = !!f.dictionary_id || !!f.prompt_template_id;
            return (
              <div
                key={f.id}
                className={`${styles.fcard}
                  ${selectable ? (isChecked ? styles.fcardActive : '') : ''}
                  ${deletable ? (isDeleteChecked ? styles.fcardActiveDelete : '') : ''}
                  ${exportable ? (isExportChecked ? styles.fcardActiveExport : '') : ''}
                  ${!selectable && !deletable && !exportable ? (f.id === activeFileId ? styles.fcardActiveView : '') : ''}
                `}
                onClick={toggle}
                title={f.original_name}
              >
                {(selectable || deletable || exportable) && (
                  <div className={styles.fcardCheck}>
                    <Checkbox checked={checked} onChange={toggle} />
                  </div>
                )}

                {hasLinks && (
                  <div className={styles.fcardLinks}>
                    {f.dictionary_id && (
                      <span className={styles.fcardLinkIcon} title={t('uploadInfer.filePanel.dictionaryLinked')}>
                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                          <path d="M3 2.5h7.5a1.5 1.5 0 011.5 1.5v9a.5.5 0 01-.5.5H4a1 1 0 01-1-1V2.5z" />
                          <path d="M3 11.5a1 1 0 011-1h8" />
                        </svg>
                      </span>
                    )}
                    {f.prompt_template_id && (
                      <span className={styles.fcardLinkIcon} title={t('uploadInfer.filePanel.templateLinked')}>
                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                          <path d="M3 2.5h10v11H3z" />
                          <path d="M5.5 5.5h5M5.5 8h5M5.5 10.5h3" />
                        </svg>
                      </span>
                    )}
                  </div>
                )}

                <div className={`${styles.fcardIcon} ${styles[ext]}`}>{ext.toUpperCase()}</div>
                <div className={styles.fcardName}>{f.original_name}</div>
                <div className={styles.fcardMeta}>{f.inserted_at} · #{f.id}</div>

                <div className={styles.fcardPrompts} onClick={e => e.stopPropagation()}>
                  {PROMPT_FIELDS.map(({ key, letter, labelKey }) => {
                    const set = !!(f[key] as string)?.trim();
                    return (
                      <span
                        key={key}
                        className={`${styles.fcardPromptDot} ${set ? styles.fcardPromptDotSet : ''}`}
                        title={`${t(labelKey)}${set ? ' — ' + t('uploadInfer.filePanel.promptCustomized') : ' — ' + t('uploadInfer.filePanel.promptDefault')}`}
                      >
                        {letter}
                      </span>
                    );
                  })}
                </div>

                <span className={`${styles.badge} ${styles.fcardBadge} ${isChecked ? styles.bSelected :
                  isDeleteChecked ? styles.bDelete :
                    isExportChecked ? styles.bExport :
                      styles[statusMeta.cls]
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
                  ) : (
                    <>
                      {f.status === 'completed' && (
                        <svg viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" width="9" height="9">
                          <path d="M2 6l3 3 5-5" />
                        </svg>
                      )}
                      {(f.status === 'running' || f.status === 'queued') && (
                        <span className={styles.fcardBadgeDot} />
                      )}
                      {t(statusMeta.labelKey)}
                    </>
                  )}
                </span>
              </div>
            );
          })}
        </div>

        {/* Pagination footer */}
        {!filesLoading && !filesError && filesTotal > 0 && (
          <div className={styles.paginationBar}>
            <div className={styles.paginationTopRow}>
              <div className={styles.pageSizeGroup}>
                <span className={styles.pageSizeLabel}>{t('uploadInfer.filePanel.perPage')}</span>
                <select
                  className={styles.pageSizeSelect}
                  value={filesPageSize}
                  onChange={e => handlePageSizeChange(Number(e.target.value))}
                >
                  {[50, 75, 100].map(size => (
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

                {(() => {
                  const pageWindow = getPageNumbers(filesPage, filesTotalPages);
                  const showFirst = pageWindow[0] > 1;
                  const showLast = pageWindow[pageWindow.length - 1] < filesTotalPages;
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
                          className={`${styles.pageNumBtn} ${p === filesPage ? styles.pageNumBtnActive : ''}`}
                          onClick={() => goToPage(p)}
                        >
                          {p}
                        </button>
                      ))}
                      {showLast && (
                        <>
                          {pageWindow[pageWindow.length - 1] < filesTotalPages - 1 && <span className={styles.pageEllipsis}>…</span>}
                          <button className={styles.pageNumBtn} onClick={() => goToPage(filesTotalPages)}>{filesTotalPages}</button>
                        </>
                      )}
                    </>
                  );
                })()}

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
  grid-auto-rows: 226px;
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
  gap: 7px;
  padding: 16px 14px 34px;
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

// Dictionary / prompt-template association icons — mirrors the checkbox's
// corner, opposite side, and only renders when at least one is linked.
.fcardLinks {
  position: absolute;
  top: 10px;
  right: 10px;
  z-index: 1;
  display: flex;
  gap: 4px;
}

.fcardLinkIcon {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 20px;
  height: 20px;
  border-radius: 5px;
  background: var(--violet-dim);
  color: var(--violet);

  svg { width: 11px; height: 11px; }
}

.fcardIcon {
  width: 36px;
  height: 36px;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 10px;
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
  line-height: 1.3;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
  word-break: break-word;
  flex-shrink: 0;
}

.fcardMeta {
  font-size: 9.5px;
  color: var(--t2);
  flex-shrink: 0;
  @include m.mono;
}

// Compact S·K·Q·A·T checklist — which content prompts are customized
// for this file (Summary, Keywords, Questions, short Answer, True/false).
.fcardPrompts {
  display: flex;
  align-items: center;
  gap: 3px;
  flex-shrink: 0;
  cursor: default;
}

.fcardPromptDot {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 15px;
  height: 15px;
  border-radius: 4px;
  background: var(--bg2);
  color: var(--t2);
  font-size: 8px;
  font-weight: 700;
  @include m.mono;
}

.fcardPromptDotSet {
  background: var(--blue-dim);
  color: var(--blue);
}

.fcardBadgeDot {
  width: 5px;
  height: 5px;
  border-radius: 50%;
  background: currentColor;
  animation: fcardPulse 1.4s ease-in-out infinite;
}

@keyframes fcardPulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.35; }
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

// Running — blue, with the pulsing dot from fcardBadgeDot
.bRunning {
  background: var(--blue-dim);
  color: var(--blue);
  border-color: var(--blue-bdr);
}

// Queued — amber, waiting its turn in the batch
.bQueued {
  background: rgba(240, 160, 48, 0.1);
  color: var(--amber);
  border-color: rgba(240, 160, 48, 0.3);
}

// Error — red, inference failed for this file
.bError {
  background: var(--red-dim);
  color: var(--red);
  border-color: var(--red-bdr);
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






















// ═══════════════════════════════════════════════
// store/uploadSlice.ts
// Content Analytics · Upload & Inference state
// ═══════════════════════════════════════════════
import { createSlice, type PayloadAction } from '@reduxjs/toolkit';

// ── Server file (from /files/by-date/) ──────────
// "tbd" = not yet inferenced, "completed" = inferenced.
export type FileStatus = 'waiting' | 'running' | 'completed' | 'error' | 'tbd' | 'queued';

export interface ServerFile {
  id: number;
  original_name: string;
  inserted_at: string;
  summary_prompt: string;
  keywords_prompt: string;
  faq_prompt: string;
  short_answer_prompt: string;
  true_false_prompt: string;
  progress: string | number | null;
  dictionary_id?: number | null;
  prompt_template_id?: number | null;
  status: FileStatus;
}

// ── /files/by-date/ pagination params & sort keys ──
export type FilesSortBy = 'id' | 'original_name' | 'inserted_at' | 'status';
export type SortOrder = 'asc' | 'desc';

export interface ServerFilesData {
  queued: ServerFile[];
  completed: ServerFile[];
  pending: ServerFile[];
  running: ServerFile[];
}

export interface UploadedFile {
  id: number;
  name: string;
  size: string;
  type: 'vtt' | 'srt';
  status: 'ready' | 'running' | 'done' | 'failed';
  summaryPrompt: string;
  questionPrompt: string;
}

export interface BatchFile {
  id: number;
  name: string;
  size: string;
  type: 'vtt' | 'srt';
  progress?: number;
}

export interface BatchGroup {
  id: string;
  status: 'running' | 'queued' | 'done' | 'pending';
  files: BatchFile[];
  collapsed: boolean;
}

export type TimeInterval = 5 | 10 | 15 | 20 | 30 | 45 | 60;

export interface InferenceSettings {
  generateSummary: boolean;
  generateKeywords: boolean;
  generateQuestions: boolean;
  summaryStyle: string;
  timestampInterval: string;
  languageOutput: string;
  mcq: boolean;
  trueFalse: boolean;
  shortAnswer: boolean;
  questionCount: number;
  keywordCount: number;
  summaryPromptOverride: string;
  keywordPromptOverride: string;
  questionPromptOverride: string;

  // ── New independent content types ──
  generateShortAnswer: boolean;
  generateTrueFalse: boolean;
  // Only meaningful (and only sent as true) when generateKeywords is also true.
  generateKeywordInsights: boolean;
  timestampedSummary: boolean;
  timeInterval: TimeInterval;
  shortAnswerPromptOverride: string;
  trueFalsePromptOverride: string;
}

interface UploadState {
  files: UploadedFile[];
  selectedIds: number[];
  settingsCollapsed: boolean;
  batchVisible: boolean;
  batchGroups: BatchGroup[];
  settings: InferenceSettings;
  uploadZoneCollapsed: boolean;

  // ── Server files (from /files/by-date/) ──
  // NOTE: serverFilesData is now populated exclusively by inferenceStatusSuccess
  // (the /files/by-progress/ polling endpoint), which still returns the old
  // bucketed shape. /files/by-date/ is paginated and returns a flat, per-file
  // `status` — so it no longer feeds serverFilesData.
  serverFilesData: ServerFilesData;          // raw split by status (from /files/by-progress/)
  serverFiles: ServerFile[];             // current page of /files/by-date/ results
  serverFilesLoading: boolean;
  serverFilesError: string | null;
  dateFrom: string;
  dateTo: string;

  // ── /files/by-date/ pagination, sort & search ──
  filesPage: number;
  filesPageSize: number;
  filesTotal: number;
  filesTotalPages: number;
  filesSortBy: FilesSortBy;
  filesSortOrder: SortOrder;
  filesSearch: string;

  // ── Selection on uploaded files ──
  selectedServerIds: number[];

  // ── Batch running state ──
  isBatchRunning: boolean;                  // true while queued/running not empty
  lastBatchFinishedAt: number | null;       // timestamp (Date.now()) set when batch transitions running→done

  // ── Models ──
  models: string[];
  modelsLoading: boolean;
  selectedModel: string;
}

function toDateStr(d: Date) { return d.toISOString().slice(0, 10); }
function defaultFrom() { const d = new Date(); d.setMonth(d.getMonth() - 1); return toDateStr(d); }
function defaultTo() { return toDateStr(new Date()); }

const emptyData: ServerFilesData = { queued: [], completed: [], pending: [], running: [] };

const initialState: UploadState = {
  files: [],
  selectedIds: [],
  uploadZoneCollapsed: false,
  settingsCollapsed: false,
  batchVisible: false,
  batchGroups: [],
  settings: {
    generateSummary: true, generateKeywords: true, generateQuestions: true,
    summaryStyle: 'Table of Contents', timestampInterval: '5 minute segments',
    languageOutput: 'Korean + English',
    mcq: true, trueFalse: true, shortAnswer: false,
    questionCount: 10, keywordCount: 15,
    summaryPromptOverride: '', keywordPromptOverride: '', questionPromptOverride: '',

    generateShortAnswer: false, generateTrueFalse: false,
    generateKeywordInsights: false,
    timestampedSummary: false, timeInterval: 5,
    shortAnswerPromptOverride: '', trueFalsePromptOverride: '',
  },
  serverFilesData: emptyData,
  serverFiles: [],
  serverFilesLoading: false,
  serverFilesError: null,
  dateFrom: defaultFrom(),
  dateTo: defaultTo(),

  filesPage: 1,
  filesPageSize: 50,
  filesTotal: 0,
  filesTotalPages: 1,
  filesSortBy: 'id',
  filesSortOrder: 'desc',
  filesSearch: '',

  selectedServerIds: [],
  isBatchRunning: false,
  lastBatchFinishedAt: null,
  models: [],
  modelsLoading: false,
  selectedModel: '',
};

const uploadSlice = createSlice({
  name: 'upload',
  initialState,
  reducers: {
    // ── Inference legacy ──
    toggleFileSelection(state, action: PayloadAction<number>) {
      const idx = state.selectedIds.indexOf(action.payload);
      if (idx > -1) state.selectedIds.splice(idx, 1);
      else state.selectedIds.push(action.payload);
    },
    toggleSelectAll(state) {
      const allIds = state.files.map(f => f.id);
      state.selectedIds = state.selectedIds.length === allIds.length ? [] : allIds;
    },
    toggleUploadZone(state) {
      state.uploadZoneCollapsed = !state.uploadZoneCollapsed;
    },
    toggleBatchGroup(state, action: PayloadAction<string>) {
      const g = state.batchGroups.find(g => g.id === action.payload);
      if (g) g.collapsed = !g.collapsed;
    },
    runInference(state) {
      state.batchVisible = true;
    },
    updateSummaryPrompt(state, action: PayloadAction<string>) {
      state.settings.summaryPromptOverride = action.payload;
    },
    updateKeywordPrompt(state, action: PayloadAction<string>) {
      state.settings.keywordPromptOverride = action.payload;
    },
    updateQuestionPrompt(state, action: PayloadAction<string>) {
      state.settings.questionPromptOverride = action.payload;
    },
    updateShortAnswerPrompt(state, action: PayloadAction<string>) {
      state.settings.shortAnswerPromptOverride = action.payload;
    },
    updateTrueFalsePrompt(state, action: PayloadAction<string>) {
      state.settings.trueFalsePromptOverride = action.payload;
    },
    updateSettings(state, action: PayloadAction<Partial<InferenceSettings>>) {
      state.settings = { ...state.settings, ...action.payload };
    },

    // ── Date range ──
    setDateFrom(state, action: PayloadAction<string>) { state.dateFrom = action.payload; },
    setDateTo(state, action: PayloadAction<string>) { state.dateTo = action.payload; },

    // ── Patch progress % onto running files from /files/progress/ response ──
    updateRunningProgress(state, action: PayloadAction<Record<string, number | string>>) {
      const progressMap = action.payload;
      state.serverFilesData.running = state.serverFilesData.running.map(f => {
        const raw = progressMap[String(f.id)];
        if (raw === undefined) return f;
        const pct = typeof raw === 'string' ? parseFloat(raw) : raw;
        return { ...f, progress: isNaN(pct) ? f.progress : Math.min(100, Math.max(0, pct)) };
      });
    },

    // ── Inference-only status update (does NOT touch the FilePanel file list) ──
    inferenceStatusSuccess(state, action: PayloadAction<ServerFilesData>) {
      const d = action.payload;
      // Merge any newly-completed files from the polling response into the
      // existing completed bucket. We keep the historical /by-date/ entries
      // and append/replace any IDs that just finished in the active batch,
      // so FilePanel badges flip to "Inferenced" the moment Step-2 moves a
      // file into its Completed column — without needing another API call.
      const incomingCompleted = d.completed ?? [];
      const incomingCompletedIds = new Set(incomingCompleted.map(f => f.id));
      const mergedCompleted = [
        ...state.serverFilesData.completed.filter(f => !incomingCompletedIds.has(f.id)),
        ...incomingCompleted,
      ];
      state.serverFilesData = {
        queued: d.queued ?? [],
        running: d.running ?? [],
        pending: d.pending ?? [],
        completed: mergedCompleted,
      };
      const running = (d.queued?.length ?? 0) > 0 || (d.running?.length ?? 0) > 0;
      // Clear selections the moment the batch becomes active so highlighted
      // cards stop showing the blue selected state after a successful submit.
      if (running && !state.isBatchRunning) {
        state.selectedServerIds = [];
      }
      // Auto-collapse settings when a batch starts running. (Previously derived
      // from /files/by-date/, which is now paginated and can't reliably tell us this.)
      if (running) state.settingsCollapsed = true;
      // When batch transitions running → done, stamp a timestamp so FilePanel
      // knows to re-fetch /by-date/ and refresh the completed list.
      if (!running && state.isBatchRunning) {
        state.lastBatchFinishedAt = Date.now();
      }
      state.isBatchRunning = running;
    },

    // ── Server files (paginated /files/by-date/) ──
    serverFilesLoading(state) {
      state.serverFilesLoading = true;
      state.serverFilesError = null;
    },
    serverFilesSuccess(state, action: PayloadAction<{
      files: ServerFile[];
      total: number;
      page: number;
      pageSize: number;
      totalPages: number;
      sortBy: FilesSortBy;
      sortOrder: SortOrder;
      search: string;
    }>) {
      const { files, total, page, pageSize, totalPages, sortBy, sortOrder, search } = action.payload;
      state.serverFiles = files;
      state.serverFilesLoading = false;
      state.serverFilesError = null;

      state.filesTotal = total;
      state.filesPage = page;
      state.filesPageSize = pageSize;
      state.filesTotalPages = totalPages;
      state.filesSortBy = sortBy;
      state.filesSortOrder = sortOrder;
      state.filesSearch = search;

      // NOTE: isBatchRunning is no longer derived here — /files/by-date/ is
      // paginated so a running/queued file may simply be on another page.
      // isBatchRunning is set by inferenceStatusSuccess (/files/by-progress/),
      // which always reflects the full, unpaginated set of active files.

      // NOTE: selections are intentionally NOT pruned against this page's ids —
      // a file not present here may just be on a different page, not deleted.
      // Deletion flows already clear selectedServerIds explicitly on success.
    },
    serverFilesFailure(state, action: PayloadAction<string>) {
      state.serverFilesLoading = false;
      state.serverFilesError = action.payload;
    },

    // ── Pagination / sort / search (kept in sync for UI display; the actual
    //     fetch is triggered imperatively by the component with these values) ──
    setFilesPage(state, action: PayloadAction<number>) {
      state.filesPage = action.payload;
    },
    setFilesPageSize(state, action: PayloadAction<number>) {
      state.filesPageSize = action.payload;
      state.filesPage = 1;
    },
    setFilesSort(state, action: PayloadAction<{ sortBy: FilesSortBy; sortOrder: SortOrder }>) {
      state.filesSortBy = action.payload.sortBy;
      state.filesSortOrder = action.payload.sortOrder;
      state.filesPage = 1;
    },
    setFilesSearch(state, action: PayloadAction<string>) {
      state.filesSearch = action.payload;
      state.filesPage = 1;
    },

    // ── Server file selection ──
    toggleServerFileSelection(state, action: PayloadAction<number>) {
      if (state.isBatchRunning) return;  // no selection while batch running
      const idx = state.selectedServerIds.indexOf(action.payload);
      if (idx > -1) state.selectedServerIds.splice(idx, 1);
      else state.selectedServerIds.push(action.payload);
    },
    toggleSelectAllServerFiles(state) {
      if (state.isBatchRunning) return;
      const pageIds = state.serverFiles.map(f => f.id);
      if (pageIds.length === 0) return;
      const selectedSet = new Set(state.selectedServerIds);
      const allPageSelected = pageIds.every(id => selectedSet.has(id));
      if (allPageSelected) {
        // Uncheck — remove only this page's ids, leave other pages' selections alone.
        const pageIdSet = new Set(pageIds);
        state.selectedServerIds = state.selectedServerIds.filter(id => !pageIdSet.has(id));
      } else {
        // Check — add this page's ids on top of whatever's already selected elsewhere.
        pageIds.forEach(id => { if (!selectedSet.has(id)) state.selectedServerIds.push(id); });
      }
    },
    clearServerSelection(state) {
      state.selectedServerIds = [];
    },

    // ── Patch prompt fields on saved files ──
    updateFilePrompts(state, action: PayloadAction<{
      fileIds: number[];
      summaryPrompt?: string;
      keywordsPrompt?: string;
      faqPrompt?: string;
      shortAnswerPrompt?: string;
      trueFalsePrompt?: string;
    }>) {
      const { fileIds, summaryPrompt, keywordsPrompt, faqPrompt, shortAnswerPrompt, trueFalsePrompt } = action.payload;
      const idSet = new Set(fileIds);
      state.serverFiles = state.serverFiles.map(f => {
        if (!idSet.has(f.id)) return f;
        return {
          ...f,
          ...(summaryPrompt !== undefined && { summary_prompt: summaryPrompt }),
          ...(keywordsPrompt !== undefined && { keywords_prompt: keywordsPrompt }),
          ...(faqPrompt !== undefined && { faq_prompt: faqPrompt }),
          ...(shortAnswerPrompt !== undefined && { short_answer_prompt: shortAnswerPrompt }),
          ...(trueFalsePrompt !== undefined && { true_false_prompt: trueFalsePrompt }),
        };
      });
      // Also patch inside serverFilesData buckets so the per-file preview reflects the change
      const patchBucket = (bucket: ServerFile[]) => bucket.map(f => {
        if (!idSet.has(f.id)) return f;
        return {
          ...f,
          ...(summaryPrompt !== undefined && { summary_prompt: summaryPrompt }),
          ...(keywordsPrompt !== undefined && { keywords_prompt: keywordsPrompt }),
          ...(faqPrompt !== undefined && { faq_prompt: faqPrompt }),
          ...(shortAnswerPrompt !== undefined && { short_answer_prompt: shortAnswerPrompt }),
          ...(trueFalsePrompt !== undefined && { true_false_prompt: trueFalsePrompt }),
        };
      });
      state.serverFilesData.completed = patchBucket(state.serverFilesData.completed);
      state.serverFilesData.queued = patchBucket(state.serverFilesData.queued);
      state.serverFilesData.running = patchBucket(state.serverFilesData.running);
      state.serverFilesData.pending = patchBucket(state.serverFilesData.pending);
    },


    // ── Patch dictionary_id on saved files (after associate/disassociate) ──
    patchFileDictionaries(state, action: PayloadAction<Record<number, number | null>>) {
      const map = action.payload;
      const apply = (f: ServerFile): ServerFile =>
        map[f.id] !== undefined ? { ...f, dictionary_id: map[f.id] } : f;
      state.serverFiles = state.serverFiles.map(apply);
      state.serverFilesData.completed = state.serverFilesData.completed.map(apply);
      state.serverFilesData.queued = state.serverFilesData.queued.map(apply);
      state.serverFilesData.running = state.serverFilesData.running.map(apply);
      state.serverFilesData.pending = state.serverFilesData.pending.map(apply);
    },

    // ── Patch prompt_template_id on saved files (after template association) ──
    patchFilePromptTemplate(state, action: PayloadAction<Record<number, number | null>>) {
      const map = action.payload;
      const apply = (f: ServerFile): ServerFile =>
        map[f.id] !== undefined ? { ...f, prompt_template_id: map[f.id] } : f;
      state.serverFiles = state.serverFiles.map(apply);
      state.serverFilesData.completed = state.serverFilesData.completed.map(apply);
      state.serverFilesData.queued = state.serverFilesData.queued.map(apply);
      state.serverFilesData.running = state.serverFilesData.running.map(apply);
      state.serverFilesData.pending = state.serverFilesData.pending.map(apply);
    },

    // ── Models ──
    modelsLoading(state) {
      state.modelsLoading = true;
    },
    modelsSuccess(state, action: PayloadAction<string[]>) {
      state.models = action.payload;
      state.modelsLoading = false;
      if (action.payload.length > 0 && !state.selectedModel) {
        state.selectedModel = action.payload[0];
      }
    },
    modelsFailure(state) {
      state.modelsLoading = false;
    },
    setSelectedModel(state, action: PayloadAction<string>) {
      state.selectedModel = action.payload;
    },
  },
});

export const {
  toggleFileSelection, toggleSelectAll,
  toggleUploadZone, toggleBatchGroup,
  runInference, updateSummaryPrompt, updateKeywordPrompt, updateQuestionPrompt,
  updateShortAnswerPrompt, updateTrueFalsePrompt, updateSettings,
  setDateFrom, setDateTo,
  inferenceStatusSuccess,
  updateRunningProgress,
  serverFilesLoading, serverFilesSuccess, serverFilesFailure,
  setFilesPage, setFilesPageSize, setFilesSort, setFilesSearch,
  toggleServerFileSelection, toggleSelectAllServerFiles, clearServerSelection,
  updateFilePrompts,
  patchFileDictionaries,
  patchFilePromptTemplate,
  modelsLoading, modelsSuccess, modelsFailure, setSelectedModel,
} = uploadSlice.actions;

export default uploadSlice.reducer;























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
        const sa = file.short_answer_prompt ?? '';
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

                  <div className={styles.fieldsGrid}>

                  {/* Summary */}
                  <div className={styles.fieldCell}>
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
                  </div>

                  {/* Keywords + nested Keyword Insights */}
                  <div className={styles.fieldCell}>
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
                  </div>

                  {/* Assessment Questions */}
                  <div className={styles.fieldCell}>
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
                  </div>

                  {/* Short Answer */}
                  <div className={styles.fieldCell}>
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
                                <div className={styles.perFilePrompt}>{f.short_answer_prompt || <span className={styles.perFileEmpty}>{t('uploadInfer.inferencePanel.noPromptSet')}</span>}</div>
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
                  </div>

                  {/* True/False */}
                  <div className={styles.fieldCell}>
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
                  </div>

                  {/* Timestamp Summary + nested interval dropdown */}
                  <div className={styles.fieldCell}>
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
                  </div>

                  {/* Model dropdown — moved in from the old Output settings card, now spans both columns */}
                  <div className={`${styles.fieldCell} ${styles.fieldCellWide}`}>
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
