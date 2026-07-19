// ═══════════════════════════════════════════════
// pages/UploadInfer/FilePanel.tsx
// Content Analytics · Step-1 upload + uploaded files list
// ═══════════════════════════════════════════════
import React, { useRef, useState, useCallback, useEffect, forwardRef, useImperativeHandle } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import {
  setDateFrom, setDateTo, setStatusFilter,
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
  waiting: { labelKey: 'uploadInfer.filePanel.statusWaiting', cls: 'bWaiting' },
  running: { labelKey: 'uploadInfer.filePanel.statusRunning', cls: 'bRunning' },
  completed: { labelKey: 'uploadInfer.filePanel.inferenced', cls: 'bInferred' },
  error: { labelKey: 'uploadInfer.filePanel.statusErrorBadge', cls: 'bError' },
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

// ── Which of the 5 content prompts are customized for this file ──
const PROMPT_FIELDS: { key: keyof ServerFile; letter: string; labelKey: string; icon: string; tourId: string }[] = [
  { key: 'summary_prompt', letter: 'S', labelKey: 'uploadInfer.inferencePanel.summaryPrompt', icon: 'M3 2.5h7.5a1.5 1.5 0 011.5 1.5v9a.5.5 0 01-.5.5H4a1 1 0 01-1-1V2.5zM5.5 5.5h5M5.5 8h5M5.5 10.5h3', tourId: 'upload-card-icon-summary' },
  { key: 'keywords_prompt', letter: 'K', labelKey: 'uploadInfer.inferencePanel.keywordPrompt', icon: 'M2 8l4.5-4.5H12v5.5L7.5 13.5 2 8z M9.5 5.5h.01', tourId: 'upload-card-icon-keywords' },
  { key: 'faq_prompt', letter: 'Q', labelKey: 'uploadInfer.inferencePanel.questionPrompt', icon: 'M8 2.5a5.5 5.5 0 100 11 5.5 5.5 0 000-11zM6.2 6.3a1.8 1.8 0 013.4.7c0 1.2-1.6 1.4-1.6 2.6 M8 11.3v.1', tourId: 'upload-card-icon-questions' },
  // Speech-bubble-with-lines — reads as "a short written reply", distinct from the summary document icon above.
  { key: 'short_answer_prompt', letter: 'A', labelKey: 'uploadInfer.inferencePanel.shortAnswerPrompt', icon: 'M2.5 3.5h11a1 1 0 011 1v6a1 1 0 01-1 1H6.5L3.5 14.5V11.5h-1a1 1 0 01-1-1v-6a1 1 0 011-1z M5 7h6M5 9h3.5', tourId: 'upload-card-icon-answer' },
  // Toggle switch — a direct visual metaphor for a binary True/False choice, distinct from a generic "done" checkmark.
  { key: 'true_false_prompt', letter: 'T', labelKey: 'uploadInfer.inferencePanel.trueFalsePrompt', icon: 'M5.5 4h5a4 4 0 010 8h-5a4 4 0 010-8z M10.6 8a1.25 1.25 0 102.5 0 1.25 1.25 0 00-2.5 0z', tourId: 'upload-card-icon-truefalse' },
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
  onDeleteComplete: (deletedIds: number[], all?: boolean) => void;
  // Whether the Upload tab is the one currently showing — triggers a
  // fresh fetch each time it becomes active, and clears stale state
  // when navigating away so a return visit doesn't flash old data.
  active: boolean;
  // Optional — lets the host switch to the Run inference tab once files
  // are picked here. FilePanel works fine without it (just no shortcut).
  onGoToInfer?: () => void;
}

// Exposed to the parent so the guided tour can programmatically open the
// actions popover for its "here's everything in here" step, without
// lifting actionsOpen into Redux just for that one use.
export interface FilePanelHandle {
  openActionsMenu: () => void;
  closeActionsMenu: () => void;
}

const FilePanel = forwardRef<FilePanelHandle, FilePanelProps>(({ selectMode, onEnterSelectMode, onExitSelectMode, onDeleteComplete, active, onGoToInfer }, ref) => {
  const { t } = useTranslation();
  const dispatch = useAppDispatch();
  const {
    serverFiles,
    serverFilesLoading: filesLoading, serverFilesError: filesError,
    selectedServerIds, isBatchRunning,
    dateFrom, dateTo, statusFilter,
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

  // ── Actions popover (dictionary / template / search / export / delete) ──
  const [actionsOpen, setActionsOpen] = useState(false);
  const actionsPopoverRef = useRef<HTMLDivElement>(null);
  useEffect(() => {
    if (!actionsOpen) return;
    const onPointerDown = (e: PointerEvent) => {
      if (actionsPopoverRef.current && !actionsPopoverRef.current.contains(e.target as Node)) {
        setActionsOpen(false);
      }
    };
    const onKey = (e: KeyboardEvent) => { if (e.key === 'Escape') setActionsOpen(false); };
    document.addEventListener('pointerdown', onPointerDown);
    document.addEventListener('keydown', onKey);
    return () => {
      document.removeEventListener('pointerdown', onPointerDown);
      document.removeEventListener('keydown', onKey);
    };
  }, [actionsOpen]);

  useImperativeHandle(ref, () => ({
    openActionsMenu: () => setActionsOpen(true),
    closeActionsMenu: () => setActionsOpen(false),
  }), []);

  // ── Dictionary association modal ─────────────
  const [dictModalOpen, setDictModalOpen] = useState(false);

  // ── Prompt template association modal ────────
  const [templateModalOpen, setTemplateModalOpen] = useState(false);

  // ── Per-file prompt viewer popup (opened from a card's prompt icons) ──
  const [promptViewer, setPromptViewer] = useState<{ file: ServerFile; fieldIdx: number } | null>(null);

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
    statusFilter?: FileStatus | '';
  }) => {
    const from = opts?.from ?? dateFrom;
    const to = opts?.to ?? dateTo;
    const page = opts?.page ?? filesPage;
    const pageSize = opts?.pageSize ?? filesPageSize;
    const sortBy = opts?.sortBy ?? filesSortBy;
    const sortOrder = opts?.sortOrder ?? filesSortOrder;
    const search = opts?.search ?? filesSearch;
    const statusF = opts?.statusFilter ?? statusFilter;

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
        ...(statusF ? { status_filter: statusF } : {}),
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
  }, [dispatch, dateFrom, dateTo, filesPage, filesPageSize, filesSortBy, filesSortOrder, filesSearch, statusFilter]);

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
  const handleStatusFilterChange = (value: FileStatus | '') => {
    dispatch(setStatusFilter(value));
    fetchFiles({ page: 1, statusFilter: value });
  };

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

  // Own fresh fetch every time the Upload tab becomes active — and when
  // it goes inactive (navigated away from), clear the list and reset
  // search/sort/status-filter/page so a return visit shows a clean
  // loading state instead of the previous stale results.
  useEffect(() => {
    if (active) {
      fetchFiles({ page: 1 });
    } else {
      skipNextSearchEffect.current = true;
      setSearchQuery('');
      dispatch(setStatusFilter(''));
      dispatch(serverFilesSuccess({
        files: [], total: 0, page: 1, pageSize: filesPageSize, totalPages: 1,
        sortBy: 'id', sortOrder: 'desc', search: '',
      }));
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [active]);

  const displayedFiles = serverFiles;

  const done = browsed.filter(f => f.status === 'success').length;
  const failed = browsed.filter(f => f.status === 'failed').length;
  const finished = done + failed;

  return (
    <div className={styles.panel}>
      <input ref={fileInputRef} type="file" accept=".vtt,.srt" multiple style={{ display: 'none' }} onChange={onFileChange} />
      <input ref={folderInputRef} type="file" accept=".vtt,.srt" multiple style={{ display: 'none' }} onChange={onFileChange} {...{ webkitdirectory: 'true' }} />

      {/* ══ SECTION 1: Step-1 ══ */}
      <div className={styles.step1} data-tour="upload-dropzone">
        <div className={styles.step1Card}>
        <div className={styles.step1Bar}>
          <span className={styles.slbl}>{t('uploadInfer.filePanel.step1Label')}</span>
          {(view === 'preview' || view === 'uploading') && (
            <span className={`${styles.badge} ${styles.bInfo}`}>{t('uploadInfer.filePanel.selected', { count: browsed.length })}</span>
          )}
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
            </div>
          )}
        </div>

        {/* ── Fixed footer — stays put once files are browsed, doesn't scroll away with the list ── */}
        {view === 'preview' && (
          <div className={styles.step1Footer}>
            <button className={`${styles.btn} ${styles.btnP} ${styles.step1FooterBtn}`} onClick={handleUpload}>
              <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                <path d="M8 11V4M5 7l3-3 3 3" /><path d="M2.5 13.5h11" />
              </svg>
              {t('uploadInfer.filePanel.uploadBtn', { count: browsed.length })}
            </button>
            <button className={`${styles.btn} ${styles.btnDanger}`} onClick={handleCancel}>{t('uploadInfer.filePanel.cancelBtn')}</button>
          </div>
        )}
        {view === 'uploading' && (
          <div className={styles.step1Footer}>
            <div className={styles.uploadSummary}>
              <div className={styles.uploadProgressBar}>
                <div className={styles.uploadProgressFill} style={{ width: `${browsed.length ? (finished / browsed.length) * 100 : 0}%` }} />
              </div>
              <div className={styles.uploadProgressLabel}>
                {t('uploadInfer.filePanel.complete', { finished, total: browsed.length })}
                {failed > 0 && <span className={styles.failCount}>{t('uploadInfer.filePanel.failed', { count: failed })}</span>}
              </div>
            </div>
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
                onChange={() => dispatch(toggleSelectAllServerFiles(serverFiles.map(f => f.id)))}
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

            <div className={styles.sortHeader} data-tour="upload-sort">
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
          <div className={styles.filterBarRow}>
            {/* ── Date filter ── */}
            <div className={styles.dateFilter} data-tour="upload-date-filter">
              <svg className={styles.dateIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                <rect x="2" y="3" width="12" height="11" rx="1.5" />
                <path d="M2 6.5h12M5 2v2.5M11 2v2.5" />
              </svg>
              <span className={styles.filterBarLabel}>{t('uploadInfer.filePanel.dateRangeLabel', 'Date Range')}</span>
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

            {/* ── Status filter + Actions — grouped on the right ── */}
            <div className={styles.filterBarRight}>
              <div className={styles.statusFilterWrap} data-tour="upload-status-filter">
                <svg className={styles.dateIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                  <path d="M2 4h12M4.5 8h7M7 12h2" />
                </svg>
                <span className={styles.filterBarLabel}>{t('uploadInfer.filePanel.statusLabel', 'Status')}</span>
                <select
                  className={styles.statusFilterSelect}
                  value={statusFilter}
                  disabled={filesLoading || isBatchRunning}
                  onChange={e => handleStatusFilterChange(e.target.value as FileStatus | '')}
                >
                  {STATUS_FILTER_OPTIONS.map(opt => (
                    <option key={opt.value || 'all'} value={opt.value}>{t(opt.labelKey)}</option>
                  ))}
                </select>
              </div>

              {/* ── Actions — collapsed into a single trigger + popover, only in normal (idle) mode ── */}
              {!selectMode && !deleteMode && !exportMode && !isBatchRunning && (
                <div className={styles.actionsPopoverWrap} ref={actionsPopoverRef}>
                  <button
                    type="button"
                  className={`${styles.actionsTrigger} ${actionsOpen ? styles.actionsTriggerOpen : ''}`}
                  onClick={() => setActionsOpen(o => !o)}
                  aria-expanded={actionsOpen}
                  aria-haspopup="true"
                >
                  <span className={styles.filterBarLabel}>{t('uploadInfer.filePanel.actionsLabel')}</span>
                  <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                    <path d="M7 1.5l1.4 3.4L12 6l-3.6 1.4L7 12.5l-1.4-4.1L2 7l3.6-1.1z" />
                  </svg>
                  <svg className={styles.actionsTriggerChevron} viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                    <path d="M3 4.5l3 3 3-3" />
                  </svg>
                </button>

                {actionsOpen && (
                  <div className={styles.actionsPopover} role="menu" data-tour="upload-actions">
                    {/* Dictionary */}
                    <button
                      data-tour="upload-action-dictionary"
                      className={`${styles.actionTile} ${styles.actionTileDictionary}`}
                      onClick={() => { setDictModalOpen(true); setActionsOpen(false); }}
                      disabled={filesTotal === 0}
                      title={t('uploadInfer.filePanel.dictionaryBtn')}
                    >
                      <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                        strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                        <path d="M3 2.5h7.5a1.5 1.5 0 011.5 1.5v9a.5.5 0 01-.5.5H4a1 1 0 01-1-1V2.5z" />
                        <path d="M3 11.5a1 1 0 011-1h8" />
                        <path d="M6 5.5h3" />
                      </svg>
                      <span className={styles.actionTileLabel}>{t('uploadInfer.filePanel.dictionaryBtn')}</span>
                    </button>

                    {/* Template */}
                    <button
                      data-tour="upload-action-template"
                      className={`${styles.actionTile} ${styles.actionTileTemplate}`}
                      onClick={() => { setTemplateModalOpen(true); setActionsOpen(false); }}
                      disabled={filesTotal === 0}
                      title={t('uploadInfer.filePanel.templateBtn')}
                    >
                      <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                        strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                        <path d="M3 2.5h10v11H3z" />
                        <path d="M5.5 5.5h5M5.5 8h5M5.5 10.5h3" />
                      </svg>
                      <span className={styles.actionTileLabel}>{t('uploadInfer.filePanel.templateBtn')}</span>
                    </button>

                    {/* Search */}
                    <button
                      data-tour="upload-action-search"
                      className={`${styles.actionTile} ${styles.actionTileSearch} ${searchOpen ? styles.actionTileActive : ''}`}
                      onClick={() => { openSearch(); setActionsOpen(false); }}
                      title={t('uploadInfer.filePanel.searchBtn')}
                    >
                      <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                        <circle cx="6.5" cy="6.5" r="4" />
                        <path d="M11 11l2.5 2.5" />
                      </svg>
                      <span className={styles.actionTileLabel}>{t('uploadInfer.filePanel.searchBtn')}</span>
                    </button>

                    {/* Export */}
                    <button
                      data-tour="upload-action-export"
                      className={`${styles.actionTile} ${styles.actionTileExport}`}
                      onClick={() => { enterExportMode(); setActionsOpen(false); }}
                      disabled={filesTotal === 0}
                      title={t('uploadInfer.filePanel.exportBtn')}
                    >
                      <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                        strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                        <path d="M9 2H4a1 1 0 00-1 1v10a1 1 0 001 1h8a1 1 0 001-1V6z" />
                        <path d="M9 2v4h4" />
                        <path d="M6 9.5l1.5 2 2.5-3" />
                      </svg>
                      <span className={styles.actionTileLabel}>{t('uploadInfer.filePanel.exportBtn')}</span>
                    </button>

                    {/* Delete */}
                    <button
                      data-tour="upload-action-delete"
                      className={`${styles.actionTile} ${styles.actionTileDelete}`}
                      onClick={() => { enterDeleteMode(); setActionsOpen(false); }}
                      disabled={filesTotal === 0}
                      title={t('uploadInfer.filePanel.deleteBtn')}
                    >
                      <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                        strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                        <path d="M3 4h10M6 4V3h4v1" />
                        <path d="M5 4l.5 8h5l.5-8" />
                        <path d="M7 7v3M9 7v3" />
                      </svg>
                      <span className={styles.actionTileLabel}>{t('uploadInfer.filePanel.deleteBtn')}</span>
                    </button>
                  </div>
                )}
              </div>
            )}
            </div>
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
        <div className={styles.uploadedBody} data-tour={filesTotal === 0 ? 'upload-cards' : undefined}>
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
          {!filesLoading && !filesError && displayedFiles.map((f, idx) => {
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
              // No mode active — cards are no longer clickable to view results;
              // that navigation now only happens from the Workspace tab's own list.
            };
            const statusMeta = STATUS_META[f.status] ?? STATUS_META.tbd;
            const isFirstCard = idx === 0;
            const hasLinks = !!f.dictionary_id || !!f.prompt_template_id;
            return (
              <div
                key={f.id}
                data-tour={idx === 0 ? 'upload-cards' : undefined}
                className={`${styles.fcard}
                  ${selectable ? (isChecked ? styles.fcardActive : '') : ''}
                  ${deletable ? (isDeleteChecked ? styles.fcardActiveDelete : '') : ''}
                  ${exportable ? (isExportChecked ? styles.fcardActiveExport : '') : ''}
                  ${!selectable && !deletable && !exportable ? styles.fcardStatic : ''}
                `}
                onClick={toggle}
                title={f.original_name}
              >
                {(selectable || deletable || exportable) && (
                  <div className={styles.fcardCheck}>
                    <Checkbox checked={checked} onChange={toggle} />
                  </div>
                )}

                <div className={`${styles.fcardIcon} ${styles[ext]} ${(selectable || deletable || exportable) ? styles.fcardIconShifted : ''}`}>
                  {ext.toUpperCase()}
                </div>

                {hasLinks && (
                  <div className={styles.fcardLinks}>
                    {f.dictionary_id && (
                      <span className={`${styles.fcardLinkIcon} ${styles.fcardLinkIconDict}`} title={t('uploadInfer.filePanel.dictionaryLinked')}>
                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                          <path d="M8 3.7c-1.2-.9-2.9-1.2-4.7-.9v9c1.8-.3 3.5-.1 4.7.9 1.2-.9 2.9-1.2 4.7-.9v-9c-1.8-.3-3.5-.1-4.7.9z" />
                          <path d="M8 3.7v9" />
                        </svg>
                      </span>
                    )}
                    {f.prompt_template_id && (
                      <span className={`${styles.fcardLinkIcon} ${styles.fcardLinkIconTemplate}`} title={t('uploadInfer.filePanel.templateLinked')}>
                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                          <path d="M5.5 2.5h5v2h-5z" />
                          <path d="M4.5 4h7a1 1 0 011 1v8a1 1 0 01-1 1h-7a1 1 0 01-1-1V5a1 1 0 011-1z" />
                          <path d="M6 8.3l1.3 1.3L10.2 7" />
                        </svg>
                      </span>
                    )}
                  </div>
                )}

                <div className={styles.fcardName}>{f.original_name}</div>
                <div className={styles.fcardMeta}>{f.inserted_at} · #{f.id}</div>

                <div className={styles.fcardPrompts} onClick={e => e.stopPropagation()}>
                  <span className={styles.fcardPromptsLabel}>{t('uploadInfer.filePanel.promptsLabel', 'Prompts')}</span>
                  <div className={styles.fcardPromptsRow}>
                    {PROMPT_FIELDS.map(({ key, labelKey, icon, tourId }, fieldIdx) => {
                      const set = !!(f[key] as string)?.trim();
                      return (
                        <button
                          key={key}
                          type="button"
                          data-tour={isFirstCard ? tourId : undefined}
                          className={`${styles.fcardPromptDot} ${set ? styles.fcardPromptDotSet : ''}`}
                          title={`${t(labelKey)}${set ? ' — ' + t('uploadInfer.filePanel.promptCustomized') : ' — ' + t('uploadInfer.filePanel.promptDefault')}`}
                          onClick={() => setPromptViewer({ file: f, fieldIdx })}
                        >
                          <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                            <path d={icon} />
                          </svg>
                        </button>
                      );
                    })}
                  </div>
                </div>

                <span
                  className={`${styles.badge} ${styles.fcardBadge} ${isChecked ? styles.bSelected :
                    isDeleteChecked ? styles.bDelete :
                      isExportChecked ? styles.bExport :
                        styles[statusMeta.cls]
                    }`}
                  title={!isChecked && !isDeleteChecked && !isExportChecked ? t(statusMeta.labelKey) : undefined}
                >
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

      {/* ── Per-file prompt viewer — opened from a card's prompt icons ── */}
      {promptViewer && (
        <div className={styles.dangerOverlay} onClick={() => setPromptViewer(null)}>
          <div className={styles.promptViewerModal} onClick={e => e.stopPropagation()}>
            <div className={styles.promptViewerHead}>
              <div>
                <div className={styles.promptViewerTitle}>{t(PROMPT_FIELDS[promptViewer.fieldIdx].labelKey)}</div>
                <div className={styles.promptViewerFile}>{promptViewer.file.original_name}</div>
              </div>
              <button className={styles.promptViewerClose} onClick={() => setPromptViewer(null)} aria-label="Close">
                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                  <path d="M4 4l8 8M12 4l-8 8" />
                </svg>
              </button>
            </div>
            <div className={styles.promptViewerBody}>
              {(promptViewer.file[PROMPT_FIELDS[promptViewer.fieldIdx].key] as string)?.trim()
                ? (promptViewer.file[PROMPT_FIELDS[promptViewer.fieldIdx].key] as string)
                : <span className={styles.promptViewerEmpty}>{t('uploadInfer.filePanel.promptDefaultFull')}</span>}
            </div>
          </div>
        </div>
      )}

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
});

FilePanel.displayName = 'FilePanel';

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
import FilePanel, { FilePanelHandle } from './FilePanel';
import InferencePanel from './InferencePanel';
import WorkspacePanel from './WorkspacePanel';
import FileSidebar from './FileSidebar';
import TourGuide, { TourStep } from './TourGuide';
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
  const dispatch = useAppDispatch();

  const [activeTab, setActiveTab] = useState<TabId>('upload');
  const filePanelRef = useRef<FilePanelHandle>(null);
  // Whether the Upload tab currently has any files loaded — lets the tour
  // adapt its "your uploaded files" step to either point at a real card
  // or describe what will appear once something's uploaded.
  const hasUploadedFiles = useAppSelector(s => s.upload.serverFiles.length > 0);

  // ── Guided tour ──
  const TOUR_STORAGE_KEY = 'uploadInfer.tourSeen';
  const [tourActive, setTourActive] = useState(false);
  useEffect(() => {
    try {
      if (!localStorage.getItem(TOUR_STORAGE_KEY)) setTourActive(true);
    } catch { /* localStorage unavailable — just skip auto-start */ }
  }, []);
  const finishTour = useCallback(() => {
    setTourActive(false);
    filePanelRef.current?.closeActionsMenu();
    try { localStorage.setItem(TOUR_STORAGE_KEY, '1'); } catch { /* ignore */ }
  }, []);
  const startTour = useCallback(() => { setActiveTab('upload'); setTourActive(true); }, []);

  const allTourSteps: TourStep[] = [
    {
      target: 'tabbar',
      title: t('uploadInfer.tour.tabsTitle', 'Three steps, always available'),
      content: t('uploadInfer.tour.tabsBody', 'Upload & Manage, Inference, and View Results — you can jump between them any time, nothing is locked behind finishing a previous step.'),
      onEnter: () => setActiveTab('upload'),
    },
    {
      target: 'upload-dropzone',
      title: t('uploadInfer.tour.uploadTitle', 'Add your files here'),
      content: t('uploadInfer.tour.uploadBody', 'Drag in a lecture recording\u2019s captions (.vtt or .srt), or browse for one. You can drop in several files at once.'),
      onEnter: () => setActiveTab('upload'),
    },
    {
      target: 'upload-cards',
      title: t('uploadInfer.tour.cardsTitle', 'Your uploaded files'),
      content: hasUploadedFiles
        ? t('uploadInfer.tour.cardsBody', 'Every file shows up here as a card. The colored badge in the corner shows the format (VTT/SRT); the small icons below the filename open the exact prompt used for each content type (Summary, Keywords, Questions, Answer, True/False); and the bottom badge shows the file\u2019s current status. If a dictionary or prompt template is linked to the file, you\u2019ll see a small book or checklist icon too.')
        : t('uploadInfer.tour.cardsBodyEmpty', 'You haven\u2019t uploaded anything yet, so this area is empty for now. Once you do, each file becomes a card showing: a format badge (VTT/SRT), the filename, small icons for each content type\u2019s prompt (Summary, Keywords, Questions, Answer, True/False), a book/checklist icon if a dictionary or prompt template is linked, and a status badge at the bottom.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeActionsMenu(); },
    },
    // ── Icon-by-icon walkthrough — only shown when a real card exists to point at ──
    {
      target: 'upload-card-icon-summary',
      title: t('uploadInfer.tour.iconSummaryTitle', 'Summary icon'),
      content: t('uploadInfer.tour.iconSummaryBody', 'Opens the prompt used to generate this file\u2019s summary. Blue means it\u2019s been customized for this file; gray means it\u2019s still using the default.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeActionsMenu(); },
    },
    {
      target: 'upload-card-icon-keywords',
      title: t('uploadInfer.tour.iconKeywordsTitle', 'Keywords icon'),
      content: t('uploadInfer.tour.iconKeywordsBody', 'Opens the prompt used to extract this file\u2019s keywords \u2014 the tag-shaped icon.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeActionsMenu(); },
    },
    {
      target: 'upload-card-icon-questions',
      title: t('uploadInfer.tour.iconQuestionsTitle', 'Assessment Questions icon'),
      content: t('uploadInfer.tour.iconQuestionsBody', 'Opens the prompt used to generate quiz-style assessment questions \u2014 the question-mark icon.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeActionsMenu(); },
    },
    {
      target: 'upload-card-icon-answer',
      title: t('uploadInfer.tour.iconAnswerTitle', 'Short Answer icon'),
      content: t('uploadInfer.tour.iconAnswerBody', 'Opens the prompt used to generate short written answers \u2014 the speech-bubble icon.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeActionsMenu(); },
    },
    {
      target: 'upload-card-icon-truefalse',
      title: t('uploadInfer.tour.iconTrueFalseTitle', 'True/False icon'),
      content: t('uploadInfer.tour.iconTrueFalseBody', 'Opens the prompt used to generate True/False questions \u2014 the toggle-switch icon.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeActionsMenu(); },
    },
    {
      target: 'upload-sort',
      title: t('uploadInfer.tour.sortTitle', 'Sort the list'),
      content: t('uploadInfer.tour.sortBody', 'Sort your files by ID, Name, Date, or Status \u2014 click a column again to flip between ascending and descending.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeActionsMenu(); },
    },
    {
      target: 'upload-date-filter',
      title: t('uploadInfer.tour.dateFilterTitle', 'Filter by date range'),
      content: t('uploadInfer.tour.dateFilterBody', 'Narrow the file list down to a date range, then hit Apply. This affects only what\u2019s shown here on the Upload tab.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeActionsMenu(); },
    },
    {
      target: 'upload-status-filter',
      title: t('uploadInfer.tour.statusFilterTitle', 'Filter by status'),
      content: t('uploadInfer.tour.statusFilterBody', 'Show only files in one status: All, Waiting, Queued, Running, Inferenced, Error, or Pending \u2014 handy once you have a lot of files in flight.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeActionsMenu(); },
    },
    {
      target: 'upload-actions',
      title: t('uploadInfer.tour.actionsTitle', 'Bulk actions, one click away'),
      content: t('uploadInfer.tour.actionsBody', 'Here are all five bulk actions: Dictionary (link a term dictionary to your files), Prompt Template (link a saved prompt set), Search (find files by name), Export (download files), and Delete (remove files) \u2014 each applies to many files at once.'),
      placement: 'left',
      // Actually opens the menu so every action is visible while this step is shown.
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.openActionsMenu(); },
    },
    // ── One step per action button, menu kept open throughout ──
    {
      target: 'upload-action-dictionary',
      title: t('uploadInfer.tour.actionDictionaryTitle', 'Dictionary'),
      content: t('uploadInfer.tour.actionDictionaryBody', 'Link a term dictionary to one or more files, so inference uses the right spellings and terminology for your subject.'),
      placement: 'left',
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.openActionsMenu(); },
    },
    {
      target: 'upload-action-template',
      title: t('uploadInfer.tour.actionTemplateTitle', 'Prompt Template'),
      content: t('uploadInfer.tour.actionTemplateBody', 'Apply a saved set of prompts to one or more files in one go, instead of customizing each file\u2019s prompts by hand.'),
      placement: 'left',
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.openActionsMenu(); },
    },
    {
      target: 'upload-action-search',
      title: t('uploadInfer.tour.actionSearchTitle', 'Search'),
      content: t('uploadInfer.tour.actionSearchBody', 'Find a file by name \u2014 opens a search box for the Upload tab\u2019s file list.'),
      placement: 'left',
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.openActionsMenu(); },
    },
    {
      target: 'upload-action-export',
      title: t('uploadInfer.tour.actionExportTitle', 'Export'),
      content: t('uploadInfer.tour.actionExportBody', 'Select files and download them \u2014 useful for backing up or sharing results outside the app.'),
      placement: 'left',
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.openActionsMenu(); },
    },
    {
      target: 'upload-action-delete',
      title: t('uploadInfer.tour.actionDeleteTitle', 'Delete'),
      content: t('uploadInfer.tour.actionDeleteBody', 'Select files and remove them permanently \u2014 use with care, this can\u2019t be undone.'),
      placement: 'left',
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.openActionsMenu(); },
    },
    {
      target: 'infer-sidebar',
      title: t('uploadInfer.tour.inferSidebarTitle', 'Pick files to analyze'),
      content: t('uploadInfer.tour.inferSidebarBody', 'Select one or more files here \u2014 this list has its own date range, status filter, search, and sort, all independent of the Upload tab.'),
      placement: 'right',
      onEnter: () => { setActiveTab('infer'); filePanelRef.current?.closeActionsMenu(); },
    },
    {
      target: 'infer-settings',
      title: t('uploadInfer.tour.inferSettingsTitle', 'Choose what to generate'),
      content: t('uploadInfer.tour.inferSettingsBody', 'Turn on Summary, Keywords (with optional Keyword Insights), Assessment Questions, Short Answer, and True/False \u2014 each has its own customizable prompt. These options stay unchecked and disabled until you\u2019ve selected at least one file.'),
      onEnter: () => setActiveTab('infer'),
    },
    {
      target: 'infer-run',
      title: t('uploadInfer.tour.inferRunTitle', 'Run it'),
      content: t('uploadInfer.tour.inferRunBody', 'Once you\u2019ve picked files and settings, run the batch here. You can watch progress or switch tabs \u2014 it keeps running either way.'),
      placement: 'left',
      onEnter: () => setActiveTab('infer'),
    },
    {
      target: 'results-sidebar',
      title: t('uploadInfer.tour.resultsSidebarTitle', 'Find a finished file'),
      content: t('uploadInfer.tour.resultsSidebarBody', 'Once a file\u2019s done, click it here to open its results. This list also has its own date range, status filter, search, and sort.'),
      placement: 'right',
      onEnter: () => setActiveTab('results'),
    },
    {
      target: 'results-content',
      title: t('uploadInfer.tour.resultsContentTitle', 'Your notes, ready to use'),
      content: t('uploadInfer.tour.resultsContentBody', 'Summary, keywords, quiz questions, and more \u2014 all generated from the file you selected. You can edit and re-generate individual sections without re-running the whole batch.'),
      placement: 'left',
      onEnter: () => setActiveTab('results'),
    },
  ];

  // The five per-icon steps only make sense once there's a real card to
  // spotlight; skip them when the Upload tab is empty rather than showing
  // five dim, un-anchored tooltips in a row.
  const tourSteps: TourStep[] = hasUploadedFiles
    ? allTourSteps
    : allTourSteps.filter(s => !s.target.startsWith('upload-card-icon-'));

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

  const fetchFileData = useCallback(async (fileId: number, knownFileName?: string) => {
    setFileLoading(true);
    try {
      const res = await api.get(`/files/${fileId}`);
      const d = (res.data as any)?.data ?? {};
      setFileResult(prev => ({
        fileId,
        fileName: d.original_name ?? knownFileName ?? prev?.fileName ?? String(fileId),
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
  // `fileName` is already known from the row that was clicked, so the
  // title can show it immediately/reliably rather than depending on the
  // detail endpoint's field naming.
  const handleFileClick = useCallback(async (fileId: number, fileName?: string) => {
    setActiveTab('results');
    if (fileId === activeFileId) return;
    setActiveFileId(fileId);
    setFileResult(fileName ? {
      fileId, fileName, insertedAt: '', summary: '', keywords: [], faq: '[]',
      shortAnswer: '[]', trueFalse: '[]', timestampedSummary: '[]', keywordInsights: null,
    } : null);
    await fetchFileData(fileId, fileName);
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

  const tabs: { id: TabId; label: string; desc: string }[] = [
    { id: 'upload', label: t('uploadInfer.tabs.upload'), desc: 'Add & browse your files' },
    { id: 'infer', label: t('uploadInfer.tabs.infer'), desc: 'Configure & run analysis' },
    { id: 'results', label: t('uploadInfer.tabs.results'), desc: 'View summaries & quizzes' },
  ];

  return (
    <div className={styles.page}>
      {/* ── Header — title and self-explanatory tab cards share one row ── */}
      <div className={styles.headerBar}>
        <div className={styles.phTitleRow}>
          <div className={styles.phTitle}>{t('uploadInfer.pageTitle')}</div>
          <button type="button" className={styles.tourTriggerBtn} onClick={startTour}>
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
              <circle cx="8" cy="8" r="6.25" />
              <path d="M6.1 6.2a1.9 1.9 0 013.6.7c0 1.3-1.7 1.5-1.7 2.7M8 11.4v.1" />
            </svg>
            {t('uploadInfer.tour.takeTour', 'Take a tour')}
          </button>
        </div>

        <div className={styles.tabbar} role="tablist" data-tour="tabbar">
          {tabs.map(tab => (
            <button
              key={tab.id}
              type="button"
              role="tab"
              aria-selected={activeTab === tab.id}
              className={`${styles.tabBtn} ${activeTab === tab.id ? styles.tabBtnActive : ''}`}
              onClick={() => setActiveTab(tab.id)}
            >
              <span className={styles.tabLabel}>{tab.label}</span>
              <span className={styles.tabDesc}>{tab.desc}</span>
            </button>
          ))}
        </div>
      </div>

      <div className={styles.upbody}>
        <div className={styles.tabPane} style={{ display: activeTab === 'upload' ? 'flex' : 'none' }}>
          <FilePanel
            ref={filePanelRef}
            selectMode={selectMode}
            onEnterSelectMode={enterInferSelection}
            onExitSelectMode={exitInferSelection}
            onDeleteComplete={handleDeleteComplete}
            active={activeTab === 'upload'}
            onGoToInfer={() => setActiveTab('infer')}
          />
        </div>

        <div className={styles.tabPane} style={{ display: activeTab === 'infer' ? 'flex' : 'none' }}>
          <FileSidebar mode="select" active={activeTab === 'infer'} />
          <InferencePanel />
        </div>

        <div className={styles.tabPane} style={{ display: activeTab === 'results' ? 'flex' : 'none' }}>
          <FileSidebar mode="view" active={activeTab === 'results'} activeFileId={activeFileId} onFileClick={handleFileClick} />
          <div data-tour="results-content" style={{ display: 'contents' }}>
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

      <TourGuide steps={tourSteps} active={tourActive} onFinish={finishTour} />
    </div>
  );
};

export default UploadInfer;




















{
  "header": {
    "logoText": "Content",
    "logoTextSpan": "Analytics",
    "sso": "Knox SSO",
    "theme": {
      "label": "Choose theme",
      "dark": "Dark",
      "light": "Light",
      "navy": "Navy"
    },
    "userMenu": {
      "label": "User menu",
      "administrator": "Administrator",
      "logout": "Log out"
    }
  },
  "footer": {
    "copyright": "© {{year}} Knox University. All rights reserved."
  },
  "sidebar": {
    "expand": "Expand sidebar",
    "collapse": "Collapse sidebar",
    "menu": "Menu",
    "soon": "Soon",
    "groups": {
      "main": "Main",
      "analysis": "Analysis",
      "tools": "Tools",
      "feedback": "Feedback",
      "settings": "Settings",
      "usage": "Usage"
    },
    "nav": {
      "lecturePipeline": "Lecture Pipeline",
      "historyWorkspace": "History & Workspace",
      "keywordInsights": "Keyword Insights",
      "assessment": "Assessment",
      "sttTranscription": "STT Transcription",
      "promptTemplates": "Prompt Templates",
      "generalSettings": "General Settings",
      "settings": "Settings",
      "adminDashboard": "Admin Dashboard",
      "voiceOfCustomer": "Voice of Customer",
      "userDictionary": "User Dictionary",
      "dashboard": "Dashboard"
    }
  },
  "ssoLogin": {
    "title": "Content",
    "titleSpan": "Analytics",
    "subtitle": "Lecture Intelligence Platform",
    "signedOut": "You've been signed out successfully.",
    "signInBtn": "Sign in with Knox SSO",
    "hint": "Authentication is handled automatically via your institution's Knox SSO service."
  },
  "adminDashboard": {
    "pageTitle": "Dashboard",
    "userViewSub": "Your usage statistics and file activity",
    "adminViewSub": "Platform-wide usage, user management, and system health",
    "myStats": "My Stats",
    "admin": "Admin",
    "administratorBadge": "Administrator",
    "readOnlyBadge": "Read-only — admin access required for edits",
    "loading": "Loading dashboard data…",
    "loadError": "Failed to load dashboard data. Please try again.",
    "adminLoadError": "Failed to load admin dashboard data. Please try again.",
    "userStats": {
      "totalUploaded": "Total files uploaded",
      "filesInferenced": "Files inferenced",
      "aiGenerations": "AI generations run",
      "dictionariesCreated": "Dictionaries created",
      "audioVideo": "{{audio}} audio · {{video}} video",
      "ofUploads": "{{pct}}% of uploads",
      "kwSummaryFaq": "KW · Summary · FAQ",
      "promptUpdates": "{{count}} prompt updates"
    },
    "adminStats": {
      "totalUploaded": "Total files uploaded",
      "totalUsers": "Total users",
      "aiGenerations": "AI generations",
      "dictionariesCreated": "Dictionaries created",
      "activeUsers": "{{count}} active users",
      "activeOf": "{{count}} active",
      "kwSummaryFaq": "KW · Summary · FAQ",
      "promptUpdates": "{{count}} prompt updates"
    },
    "tabs": {
      "overview": "Overview",
      "files": "Per-file stats",
      "dictionary": "Dictionary usage",
      "uploadInference": "Upload vs Inference",
      "users": "User activity",
      "perUser": "Per-user detail",
      "upload_inference": "Upload vs Inference",
      "per_user": "Per-user detail"
    },
    "charts": {
      "uploadVsInference": "Upload vs Inference by Type",
      "uploadVsInferenceSub": "{{uploaded}} total uploads · {{inferenced}} inferenced",
      "aiBreakdown": "AI Generation Breakdown",
      "aiBreakdownSub": "{{total}} total generations",
      "filesByType": "Files by Type",
      "filesByTypeSub": "{{total}} total files uploaded",
      "sttCompletions": "STT Completions",
      "sttCompletionsSub": "Speech-to-text jobs finished",
      "audioStt": "Audio STT",
      "videoStt": "Video STT",
      "ofAudioUploads": "{{pct}}% of audio uploads",
      "ofVideoUploads": "{{pct}}% of video uploads",
      "uploadedLabel": "Uploaded",
      "inferencedLabel": "Inferenced",
      "dictUsageFreq": "Dictionary Usage Frequency",
      "dictUsageFreqSub": "{{count}} dictionaries created",
      "dailyUploadTrend": "Daily Upload Trend",
      "dailyUploadTrendSub": "Files uploaded per day",
      "dailyGenTrend": "Daily Generation Trend",
      "dailyGenTrendSub": "AI generations per day by type",
      "uploadsByType": "Uploads by File Type",
      "uploadsByCategoryTitle": "Upload vs Inference by Category",
      "uploadsByCategorySub": "{{uploaded}} uploaded · {{inferenced}} inferenced · {{start}} → {{end}}",
      "dailyUviTrend": "Daily Upload vs Inference Trend",
      "dailyUviTrendSub": "Uploaded and inferenced per day",
      "usagePerDict": "Usage per Dictionary",
      "usageByDept": "Usage by Department",
      "usageByDeptSub": "Dictionary count and usage per department",
      "totalDictionaries": "Total dictionaries",
      "totalUsageCount": "Total usage count",
      "keywordsLabel": "Keywords",
      "summaryLabel": "Summary",
      "faqLabel": "FAQ",
      "dictionariesLabel": "Dictionaries",
      "usageLabel": "Usage"
    },
    "table": {
      "hash": "#",
      "fileName": "File name",
      "type": "Type",
      "keywords": "Keywords",
      "summaries": "Summaries",
      "faqs": "FAQs",
      "promptUpdates": "Prompt updates",
      "noFileStats": "No file stats available",
      "dictName": "Dictionary name",
      "timesUsed": "Times used",
      "created": "Created",
      "noDicts": "No dictionaries yet",
      "username": "Username",
      "department": "Department",
      "totalFiles": "Total files",
      "generations": "Generations",
      "dictionaries": "Dictionaries",
      "lastActive": "Last active",
      "noUserActivity": "No user activity data",
      "owner": "Owner",
      "category": "Category",
      "uploaded": "Uploaded",
      "inferenced": "Inferenced",
      "coverage": "Coverage",
      "noPerUserData": "No per-user data available",
      "noAdminDicts": "No dictionaries yet",
      "deptSummary": "Department summary",
      "totalUsage": "Total usage"
    },
    "perUser": {
      "filesUploaded": "Files uploaded",
      "filesInferenced": "Files inferenced",
      "aiGenerations": "AI generations",
      "dictionaries": "Dictionaries",
      "lastActive": "Last active: {{date}}",
      "promptUpdates": "{{count}} prompt updates"
    }
  },
  "generalSettings": {
    "pageTitle": "General Settings",
    "pageSub": "defaults applied across lecture processing",
    "loading": "Loading settings…",
    "resetToDefaults": "Reset to Defaults",
    "resetting": "Resetting…",
    "discard": "Discard",
    "saveChanges": "Save Changes",
    "saving": "Saving…",
    "savedSuccess": "Settings updated successfully.",
    "resetSuccess": "Settings reset to defaults.",
    "subtitleFormat": {
      "title": "Subtitle Format",
      "desc": "Default file format used when generating subtitles."
    },
    "defaultTemplate": {
      "title": "Default Prompt Template",
      "desc": "Template applied automatically to new lecture uploads.",
      "none": "None",
      "viewAriaLabel": "View template details"
    },
    "numKeywords": {
      "title": "Number of Keywords",
      "desc": "How many keywords to extract per lecture.",
      "auto": "Auto",
      "custom": "Custom"
    },
    "numQuestions": {
      "title": "Number of Assessment Questions",
      "desc": "How many assessment questions to generate per lecture.",
      "auto": "Auto",
      "custom": "Custom"
    },
    "globalDict": {
      "title": "Global Dictionary",
      "desc": "Auto-correct recurring transcription errors across all lectures.",
      "noTerms": "No dictionary terms yet.",
      "wrongPlaceholder": "Wrong term (e.g. teh)",
      "correctPlaceholder": "Correct term (e.g. the)",
      "addTerm": "Add Term",
      "removeTerm": "Remove term",
      "apply": "Apply",
      "manage": "Manage Dictionary",
      "termsDefined_one": "{{count}} term defined",
      "termsDefined_other": "{{count}} terms defined"
    },
    "resetModal": {
      "title": "Reset to Defaults",
      "body": "This will reset subtitle format, prompt template, keyword/question counts and the global dictionary back to their default values. This can't be undone.",
      "cancel": "Cancel",
      "confirm": "Reset"
    },
    "templateModal": {
      "description": "Description",
      "summaryPrompt": "Summary Prompt",
      "keywordPrompt": "Keyword Prompt",
      "faqPrompt": "FAQ Prompt",
      "close": "Close",
      "closeAriaLabel": "Close"
    },
    "stepper": {
      "rangeError": "Enter a value between {{min}} and {{max}}",
      "decreaseAriaLabel": "Decrease",
      "increaseAriaLabel": "Increase"
    }
  },
  "promptTemplates": {
    "pageTitle": "Prompt Templates",
    "pageSub_one": "{{count}} template · reusable prompts for summary, keyword & FAQ generation",
    "pageSub_other": "{{count}} templates · reusable prompts for summary, keyword & FAQ generation",
    "newTemplate": "New Template",
    "loading": "Loading templates…",
    "createdSuccess": "Template created.",
    "updatedSuccess": "Template updated.",
    "deletedSuccess": "Template deleted.",
    "genericError": "Something went wrong. Please try again.",
    "allFieldsRequired": "All fields are required.",
    "table": {
      "name": "Name",
      "description": "Description",
      "updated": "Updated",
      "empty": "No templates yet — create your first one to standardise summary, keyword and FAQ prompts.",
      "edit": "Edit",
      "delete": "Delete",
      "deleting": "Deleting…"
    },
    "modal": {
      "createTitle": "New Template",
      "editTitle": "Edit Template",
      "name": "Name",
      "namePlaceholder": "e.g. Lecture Summary — Default",
      "description": "Description",
      "descPlaceholder": "Short description of when to use this template",
      "summaryPrompt": "Summary Prompt",
      "summaryPlaceholder": "Instructions used to generate the summary",
      "keywordPrompt": "Keyword Prompt",
      "keywordPlaceholder": "Instructions used to extract keywords",
      "faqPrompt": "FAQ Prompt",
      "faqPlaceholder": "Instructions used to generate FAQs",
      "required": "*",
      "cancel": "Cancel",
      "saving": "Saving…",
      "saveChanges": "Save Changes",
      "createBtn": "Create Template",
      "closeAriaLabel": "Close"
    },
    "deleteModal": {
      "title": "Delete Template",
      "body": "Delete <strong>{{name}}</strong>? This can't be undone.",
      "cancel": "Cancel",
      "confirm": "Delete"
    }
  },
  "settings": {
    "pageTitle": "User Dictionary",
    "pageSub": "Manage custom term replacements applied during transcription and inference",
    "dictionaries": "Dictionaries",
    "newDictTooltip": "New dictionary",
    "loading": "Loading…",
    "noDict": "No dictionaries yet.\nClick + to create one.",
    "unsaved": "Unsaved",
    "newDict": "New dictionary",
    "selectOrCreate": "Select a dictionary, or create a new one.",
    "dictNameLabel": "Dictionary name",
    "dictNamePlaceholder": "Enter a dictionary name…",
    "updated": "Updated {{date}}",
    "cancel": "Cancel",
    "saving": "Saving…",
    "saved": "Saved",
    "create": "Create",
    "saveChanges": "Save changes",
    "discardConfirm": "Discard unsaved changes?",
    "nameRequired": "Dictionary name is required",
    "termRequired": "Add at least one term with both values filled",
    "searchPlaceholder": "Search terms…",
    "term_one": "{{count}} term",
    "term_other": "{{count}} terms",
    "addTerm": "Add term",
    "wrongTerm": "Wrong term",
    "correctTerm": "Correct term",
    "wrongPlaceholder": "e.g. teh",
    "correctPlaceholder": "e.g. the",
    "removeTerm": "Remove term",
    "noSearchMatch": "No terms match your search",
    "noTermsYet": "No terms yet — click \"Add term\" to create one",
    "dictInfo": "Dictionaries are applied during STT transcription and inference. Wrong terms found in transcripts will be replaced with their correct counterparts."
  },
  "landing": {
    "badge": "Lecture Intelligence Platform · v2.4.1",
    "hero": {
      "title1": "Turn lecture transcripts",
      "title2": "into structured",
      "titleAccent": "knowledge",
      "title3": "",
      "desc": "Content Analytics ingests VTT and SRT files, runs AI inference, and delivers summaries, keyword maps, and assessment questions — all in under a minute per lecture.",
      "ctaPrimary": "Lecture Pipeline",
      "ctaSecondary": "View workspace"
    },
    "stats": {
      "fasterReview": "Faster content review",
      "groundingRate": "Question grounding rate",
      "maxAudio": "Max audio per STT job",
      "filesPerBatch": "Files per batch"
    },
    "features": {
      "sectionLabel": "What it does",
      "sectionTitle": "Every step of the pipeline,\nhandled automatically",
      "batchUpload": {
        "title": "Batch Upload",
        "desc": "Drag-and-drop VTT and SRT transcript files in bulk. Supports up to 100 files, 50 MB each, with folder-level ingestion."
      },
      "aiInference": {
        "title": "AI Inference",
        "desc": "Powered by large language models — generates structured summaries, timestamped content outlines, and keyword frequency maps."
      },
      "smartSummaries": {
        "title": "Smart Summaries",
        "desc": "Three summary modes — Table of Contents, Timestamped Core Content, and Comprehensive Overview — across Korean and English."
      },
      "assessmentGen": {
        "title": "Assessment Generation",
        "desc": "Auto-generates MCQ, True/False, and short-answer questions with anti-hallucination validation — 91%+ content grounding."
      },
      "keywordInsights": {
        "title": "Keyword Insights",
        "desc": "Frequency analysis, knowledge graphs, and word clouds reveal dominant concepts across single files or entire lecture sets."
      },
      "workspaceHistory": {
        "title": "Workspace History",
        "desc": "Full audit trail of every inference run — re-open results, re-generate outputs, and export to Excel with one click."
      }
    },
    "workflow": {
      "steps": {
        "upload": {
          "label": "Upload",
          "sub": "Drop VTT / SRT files"
        },
        "configure": {
          "label": "Configure",
          "sub": "Set prompts & outputs"
        },
        "infer": {
          "label": "Infer",
          "sub": "Run AI pipeline"
        },
        "review": {
          "label": "Review",
          "sub": "Open workspace"
        },
        "export": {
          "label": "Export",
          "sub": "Download Excel"
        }
      }
    },
    "ctaSection": {
      "title": "Ready to analyse your lectures?",
      "desc": "Upload your first transcript and get a full AI-generated summary, keyword map, and question set in under 60 seconds.",
      "button": "Lecture Pipeline"
    },
    "mockCard": {
      "tabs": {
        "summary": "Summary",
        "keywords": "Keywords",
        "questions": "Questions"
      },
      "status": "Done",
      "block1Title": "1. Definition & Classification of Algorithms",
      "block2Title": "2. Comparative Analysis of Sorting Algorithms",
      "kw1": "Algorithm 18%",
      "kw2": "Time Complexity 14%",
      "kw3": "Big-O 8%",
      "kw4": "Recursion 7%"
    }
  },
  "uploadInfer": {
    "pageTitle": "Lecture Pipeline",
    "pageSub": "Step 1 — upload VTT/SRT files · Step 2 — inference · Step 3 — workspace result",
    "tour": {
      "takeTour": "Take a tour",
      "tabsTitle": "Three steps, always available",
      "tabsBody": "Upload & Manage, Inference, and View Results — you can jump between them any time, nothing is locked behind finishing a previous step.",
      "uploadTitle": "Add your files here",
      "uploadBody": "Drag in a lecture recording's captions (.vtt or .srt), or browse for one. You can drop in several files at once.",
      "cardsTitle": "Your uploaded files",
      "cardsBody": "Every file shows up here as a card. The colored badge in the corner shows the format (VTT/SRT); the small icons below the filename open the exact prompt used for each content type (Summary, Keywords, Questions, Answer, True/False); and the bottom badge shows the file's current status. If a dictionary or prompt template is linked to the file, you'll see a small book or checklist icon too.",
      "cardsBodyEmpty": "You haven't uploaded anything yet, so this area is empty for now. Once you do, each file becomes a card showing: a format badge (VTT/SRT), the filename, small icons for each content type's prompt (Summary, Keywords, Questions, Answer, True/False), a book/checklist icon if a dictionary or prompt template is linked, and a status badge at the bottom.",
      "iconSummaryTitle": "Summary icon",
      "iconSummaryBody": "Opens the prompt used to generate this file's summary. Blue means it's been customized for this file; gray means it's still using the default.",
      "iconKeywordsTitle": "Keywords icon",
      "iconKeywordsBody": "Opens the prompt used to extract this file's keywords — the tag-shaped icon.",
      "iconQuestionsTitle": "Assessment Questions icon",
      "iconQuestionsBody": "Opens the prompt used to generate quiz-style assessment questions — the question-mark icon.",
      "iconAnswerTitle": "Short Answer icon",
      "iconAnswerBody": "Opens the prompt used to generate short written answers — the speech-bubble icon.",
      "iconTrueFalseTitle": "True/False icon",
      "iconTrueFalseBody": "Opens the prompt used to generate True/False questions — the toggle-switch icon.",
      "sortTitle": "Sort the list",
      "sortBody": "Sort your files by ID, Name, Date, or Status — click a column again to flip between ascending and descending.",
      "dateFilterTitle": "Filter by date range",
      "dateFilterBody": "Narrow the file list down to a date range, then hit Apply. This affects only what's shown here on the Upload tab.",
      "statusFilterTitle": "Filter by status",
      "statusFilterBody": "Show only files in one status: All, Waiting, Queued, Running, Inferenced, Error, or Pending — handy once you have a lot of files in flight.",
      "actionsTitle": "Bulk actions, one click away",
      "actionsBody": "Here are all five bulk actions: Dictionary (link a term dictionary to your files), Prompt Template (link a saved prompt set), Search (find files by name), Export (download files), and Delete (remove files) — each applies to many files at once.",
      "actionDictionaryTitle": "Dictionary",
      "actionDictionaryBody": "Link a term dictionary to one or more files, so inference uses the right spellings and terminology for your subject.",
      "actionTemplateTitle": "Prompt Template",
      "actionTemplateBody": "Apply a saved set of prompts to one or more files in one go, instead of customizing each file's prompts by hand.",
      "actionSearchTitle": "Search",
      "actionSearchBody": "Find a file by name — opens a search box for the Upload tab's file list.",
      "actionExportTitle": "Export",
      "actionExportBody": "Select files and download them — useful for backing up or sharing results outside the app.",
      "actionDeleteTitle": "Delete",
      "actionDeleteBody": "Select files and remove them permanently — use with care, this can't be undone.",
      "inferSidebarTitle": "Pick files to analyze",
      "inferSidebarBody": "Select one or more files here — this list has its own date range, status filter, search, and sort, all independent of the Upload tab.",
      "inferSettingsTitle": "Choose what to generate",
      "inferSettingsBody": "Turn on Summary, Keywords (with optional Keyword Insights), Assessment Questions, Short Answer, and True/False — each has its own customizable prompt. These options stay unchecked and disabled until you've selected at least one file.",
      "inferRunTitle": "Run it",
      "inferRunBody": "Once you've picked files and settings, run the batch here. You can watch progress or switch tabs — it keeps running either way.",
      "resultsSidebarTitle": "Find a finished file",
      "resultsSidebarBody": "Once a file's done, click it here to open its results. This list also has its own date range, status filter, search, and sort.",
      "resultsContentTitle": "Your notes, ready to use",
      "resultsContentBody": "Summary, keywords, quiz questions, and more — all generated from the file you selected. You can edit and re-generate individual sections without re-running the whole batch."
    },
    "batchGroups": {
      "batchTitle": "Batch #{{id}}",
      "done": "{{done}} / {{total}} done",
      "running": "Running",
      "statusTitles": {
        "running": "Running",
        "queued": "Queued",
        "done": "Completed",
        "pending": "Not selected"
      },
      "badgeLabels": {
        "running": "Active",
        "queued": "Waiting",
        "done": "Done",
        "pending": "Skipped"
      },
      "file_one": "{{count}} file",
      "file_other": "{{count}} files",
      "inQueue": "In queue",
      "emptyMsg": "No files were excluded from this batch.",
      "view": "View"
    },
    "filePanel": {
      "step1Label": "Step 1 — Files",
      "uploaded_one": "{{count}} uploaded",
      "uploaded_other": "{{count}} uploaded",
      "selected_one": "{{count}} selected",
      "selected_other": "{{count}} selected",
      "showUpload": "Show upload",
      "hideUpload": "Hide upload",
      "dropTitle": "Drop files here",
      "dropTitleDefault": "Drop .vtt / .srt here",
      "dropSub": "or click to browse · folder upload supported",
      "browseFiles": "Browse files",
      "folder": "Folder",
      "uploadBtn_one": "Upload {{count}} file",
      "uploadBtn_other": "Upload {{count}} files",
      "cancelBtn": "Cancel",
      "complete": "{{finished}} / {{total}} complete",
      "failed_one": "· {{count}} failed",
      "failed_other": "· {{count}} failed",
      "fileStatus": {
        "ready": "Ready to upload",
        "uploading": "Uploading…",
        "uploaded": "Uploaded"
      },
      "uploadedFiles": "Uploaded Files",
      "inferenceBtn": "Select files for inference",
      "dictionaryBtn": "Associate / Disassociate dictionary",
      "templateBtn": "Map prompt template",
      "searchBtn": "Search files",
      "exportBtn": "Export files to Excel",
      "deleteBtn": "Delete files",
      "dateFrom": "From",
      "dateTo": "To",
      "applyDate": "Apply",
      "sortBy": "Sort by",
      "sortId": "ID",
      "sortName": "Name",
      "sortDate": "Date",
      "sortStatus": "Status",
      "searchPlaceholder": "Search by filename or ID…",
      "loadingFiles": "Loading files…",
      "noFilesRange": "No files in this date range",
      "noFilesMatch": "No files match \"{{query}}\"",
      "inferenced": "Inferenced",
      "notInferenced": "Not Inferenced",
      "export": "Export",
      "exportToExcelTitle": "Export {{count}} file(s) to Excel",
      "exportSelectFirst": "Select files to export",
      "selectFilesToExport": "Select files to export",
      "exportCountTitle": "Export {{count}} file(s) to Excel",
      "selectFilesToDelete": "Select files to delete",
      "deleteCountTitle": "Delete {{count}} file(s)",
      "exportCount": "Export {{count}} file(s)",
      "deleteCount": "Delete {{count}} file(s)",
      "cancelTitle": "Cancel",
      "uploaded": "Uploaded ({{count}})",
      "selected": "{{count}} selected",
      "uploadBtn": "Upload {{count}} file(s)",
      "failed": "· {{count}} failed",
      "exportAllBtn": "Export All",
      "exportAllFailed": "Export failed. Please try again.",
      "deleteAllBtn": "Delete All",
      "deleteAllConfirmTitle": "Delete All Files",
      "deleteAllConfirmBody": "This will permanently delete all files uploaded between {{from}} and {{to}}. This action cannot be undone.",
      "deleteAllConfirmPrompt": "Type <strong>{{phrase}}</strong> to confirm:",
      "deleteAllConfirmBtn": "Delete All Files",
      "deleteAllFailed": "Delete failed. Please try again.",
      "matchCount": "{{count}} files",
      "pageInfo": "Page {{page}} of {{totalPages}} · {{total}} files",
      "perPage": "Per page",
      "selectedTotal": "{{count}} of {{total}} selected",
      "selectAllFiles": "Select all",
      "clearAllFiles": "Clear all",
      "inferenceMode": "Select for Inference",
      "deleteMode": "Delete Files",
      "exportMode": "Export Files",
      "exitMode": "Exit",
      "noFilesSelected": "No files selected",
      "runInferenceBtn": "Run Inference",
      "confirmDeleteBtn": "Confirm Delete",
      "confirmExportBtn": "Confirm Export",
      "paginationPrev": "Previous",
      "paginationNext": "Next",
      "actionsLabel": "Actions",
      "continueToInfer": "Continue to Inference",
      "continueToInferCount": "Continue to Inference ({{count}})",
      "dictionaryLinked": "Dictionary linked",
      "templateLinked": "Prompt template linked",
      "promptCustomized": "Customized",
      "promptDefault": "Default",
      "promptDefaultFull": "Using default prompt",
      "perPageShort": "{{count}} / page",
      "statusAll": "All",
      "statusWaiting": "Waiting",
      "statusQueued": "Queued",
      "statusRunning": "Running",
      "statusError": "Error",
      "statusErrorBadge": "Error (Inference again)",
      "statusTbd": "Pending"
    },
    "inferencePanel": {
      "step2Label": "Step 2 — Inference Configuration",
      "step2Running": "Step 2 — Inference Running",
      "expandStep2": "Expand Step 2",
      "minimizeStep2": "Minimize Step 2",
      "closeStep2": "Close Step 2",
      "filesSelected_one": "{{count}} file selected for inference",
      "filesSelected_other": "{{count}} files selected for inference",
      "batchRunning": "Batch running",
      "runInference": "Run inference",
      "selectFilesFirst": "Select files first",
      "noModelSelected": "No model selected",
      "submitting": "Submitting batch…",
      "submittingDesc": "Sending {{count}} file(s) to the inference engine.\nThis may take a moment — please wait.",
      "submittingMore": "+{{count}} more",
      "selBanner": "{{count}} file(s)",
      "selBannerEmpty": "Select files from the left panel",
      "selBannerFilled": "{{count}} file(s) selected for inference",
      "generateContent": "Generate content",
      "generateSummary": "Summary",
      "generateKeywords": "Keywords",
      "generateQuestions": "Assessment questions",
      "outputSettings": "Output settings",
      "modelLabel": "Model",
      "refreshModels": "Refresh models",
      "noModels": "No models available — inference cannot be started. Try refreshing or check your connection.",
      "modelsAvailable_one": "{{count}} model available",
      "modelsAvailable_other": "{{count}} models available",
      "noModelsOption": "— no models found —",
      "promptOverrides": "Prompt overrides",
      "promptModeSingle": "Single file — autofilled · edit to override.",
      "promptModeMulti": "Multiple files — enter here to override all.",
      "summaryPrompt": "Summary prompt",
      "keywordPrompt": "Keyword prompt",
      "questionPrompt": "Assessment question prompt",
      "optional": "optional",
      "perFile": "Per-file",
      "hide": "Hide",
      "noPromptSet": "No prompt set",
      "autofillPlaceholder": "Autofilled from file — edit to override…",
      "manualPlaceholder": "Leave blank to use per-file prompts…",
      "noContentHint": "Enable at least one content type to configure its prompt.",
      "cancelPrompt": "Cancel",
      "savePrompts": "Save prompts",
      "saving": "Saving…",
      "promptSaveFail": "Failed to save prompts. Please try again.",
      "inferenceStatus": "Inference status",
      "live": "Live",
      "refreshingIn": "refreshing in {{sec}}s",
      "filesRunning_one": "{{count}} file running",
      "filesRunning_other": "{{count}} files running",
      "queued": "Queued",
      "runningCol": "Running",
      "completed": "Completed",
      "waitingQueue": "Waiting in queue",
      "stopFile": "Stop this file",
      "stopAlready": "This file has already been processed and cannot be stopped.",
      "inferenceStarted": "Inference started successfully",
      "batchRunPill": "Batch running",
      "collapsePerFile": "Collapse per-file prompts",
      "viewPerFile": "View per-file prompts",
      "filesSelected": "{{count}} file(s) selected for inference",
      "filesRunning": "{{count}} file(s) running",
      "modelsAvailable": "{{count}} model(s) available",
      "generateKeywordInsights": "Keyword Insights",
      "generateShortAnswer": "Short Answer Questions",
      "generateTrueFalse": "True / False Questions",
      "timestampedSummary": "Timestamped Summary",
      "timeInterval": "Time interval",
      "minutesOption": "{{count}} min",
      "shortAnswerPrompt": "Short answer prompt",
      "trueFalsePrompt": "True / False prompt"
    },
    "dictModal": {
      "title": "Dictionary Association",
      "subtitle": "Assign a dictionary to each file, or apply one to all files at once.",
      "closeSaving": "Saving — please wait",
      "close": "Close",
      "applyToAll": "Apply to all files:",
      "applyBtn": "Apply to all",
      "noneOption": "— None (clear) —",
      "noneRow": "— None —",
      "changes_one": "{{count}} change pending",
      "changes_other": "{{count}} changes pending",
      "noChanges": "No changes yet",
      "colFile": "File",
      "colDict": "Dictionary",
      "loadingDicts": "Loading dictionaries…",
      "noFiles": "No files available.",
      "loadFail": "Failed to load dictionaries. Please try again.",
      "associated": "Associated",
      "viewDetails": "View dictionary details",
      "disassociate": "Disassociate",
      "detailClose": "Close details",
      "metaId": "ID:",
      "metaUpdated": "Updated:",
      "metaTerms": "Terms:",
      "loadingDetails": "Loading details…",
      "noTerms": "No terms in this dictionary.",
      "detailLoadFail": "Failed to load dictionary details.",
      "noDetailsReturned": "No details returned for this dictionary.",
      "wrongTerm": "Wrong term",
      "correctTerm": "Correct term",
      "savingHint": "Saving — please don't close this window…",
      "cancel": "Cancel",
      "save": "Save",
      "saving": "Saving…",
      "saveSuccess": "Dictionary settings saved successfully",
      "saveFail": "Failed to save dictionary settings. Please try again.",
      "associateAllLabel": "Apply dictionary to all files ({{from}} – {{to}})",
      "associateAllBtn": "Apply to All Files",
      "associateAllSuccess": "Dictionary applied to all files successfully.",
      "associateAllFail": "Failed to apply dictionary to all files. Please try again.",
      "changes": "{{count}} change(s) pending"
    },
    "templateModal": {
      "title": "Prompt Template Association",
      "subtitle": "Map a prompt template to each file to update its summary, keyword, and FAQ prompts.",
      "closeSaving": "Saving — please wait",
      "close": "Close",
      "applyToAll": "Apply to all files:",
      "applyBtn": "Apply to all",
      "noneOption": "— None —",
      "noneRow": "— None —",
      "changes_one": "{{count}} change pending",
      "changes_other": "{{count}} changes pending",
      "noChanges": "No changes yet",
      "colFile": "File",
      "colTemplate": "Prompt Template",
      "loadingTemplates": "Loading prompt templates…",
      "loadFail": "Failed to load prompt templates. Please try again.",
      "noFiles": "No files available.",
      "mapped": "Mapped",
      "viewDetails": "View template details",
      "detailClose": "Close details",
      "metaId": "ID:",
      "metaUpdated": "Updated:",
      "metaDescription": "Description:",
      "loadingDetails": "Loading details…",
      "noDetailsReturned": "No details returned for this template.",
      "detailLoadFail": "Failed to load template details.",
      "summaryPrompt": "Summary Prompt",
      "keywordPrompt": "Keyword Prompt",
      "faqPrompt": "FAQ Prompt",
      "savingHint": "Saving — please don't close this window…",
      "cancel": "Cancel",
      "save": "Save",
      "saving": "Saving…",
      "saveSuccess": "Prompt template applied successfully",
      "saveFail": "Failed to apply prompt template. Please try again.",
      "associateAllLabel": "Apply template to all files ({{from}} – {{to}})",
      "associateAllBtn": "Apply to All Files",
      "associateAllSuccess": "Template applied to all files successfully.",
      "associateAllFail": "Failed to apply template to all files. Please try again.",
      "changes": "{{count}} change(s) pending",
      "shortAnswerPrompt": "Short Answer Prompt",
      "trueFalsePrompt": "True / False Prompt"
    },
    "workspacePanel": {
      "clickToView": "Click a file to view results",
      "noFileSelected": "No file selected",
      "noFileDesc": "Click any uploaded file to view its inference results here.",
      "loadingFile": "Loading file data…",
      "tabs": {
        "summary": "Summary",
        "keywords": "Keywords",
        "assessment": "Assessment Questions",
        "shortAnswer": "Short Answer",
        "trueFalse": "True / False",
        "timestampedSummary": "Timestamped Summary",
        "keywordInsights": "Keyword Insights",
        "mcq": "MCQ"
      },
      "noSummary": "No summary available.",
      "noKeywords": "No keywords available.",
      "noQuestions": "No questions available.",
      "edit": "Edit",
      "copy": "Copy",
      "download": "Download",
      "copyAs": "Copy as",
      "downloadAs": "Download as",
      "editColPreview": "Preview",
      "editColEdit": "Edit Content",
      "markdownRenderer": "Markdown Renderer",
      "markdownTag": "Markdown",
      "chipView": "Chip View",
      "onePerLine": "One per line",
      "cancel": "Cancel",
      "save": "Save",
      "saving": "Saving",
      "autofillPlaceholder": "Autofilled from file — edit to override…",
      "keywordPlaceholder": "One keyword per line…",
      "mcq": "MCQ",
      "viewAll": "View All",
      "explanation": "Explanation",
      "complete": "Complete!",
      "assessCompleteTitle": "Assessment Complete!",
      "assessCompleteDesc": "You answered all {{count}} question(s).",
      "restart": "Restart",
      "next": "Next Question",
      "finish": "Finish",
      "question": "Q{{n}}",
      "answer": "Answer",
      "true": "True",
      "false": "False",
      "revealAnswer": "Reveal answer",
      "showAllAnswers": "Show all answers",
      "hideAllAnswers": "Hide all answers",
      "quizMode": "Quiz mode",
      "noShortAnswer": "No short answer questions available.",
      "noTrueFalse": "No true / false questions available.",
      "noTimestampedSummary": "No timestamped summary available.",
      "noKeywordInsights": "No keyword insights available.",
      "noData": "No data available.",
      "firstMentionAt": "First mention at {{pct}}%",
      "importanceAxis": "Importance",
      "kiTabs": {
        "graph": "Knowledge Graph",
        "wordcloud": "Word Cloud",
        "timeline": "Timeline",
        "heatmap": "Heatmap",
        "clusters": "Clusters",
        "frequency": "Frequency",
        "prerequisites": "Prerequisites",
        "importance": "Importance",
        "cooccurrence": "Co-occurrence",
        "glossary": "Glossary"
      },
      "scrollLeft": "Scroll tabs left",
      "scrollRight": "Scroll tabs right",
      "complexityAxis": "Complexity",
      "glossaryTerm": "Term",
      "glossaryDefinition": "Definition",
      "reason": "Reason",
      "cluster": "Cluster",
      "count": "Count",
      "copied": "Copied!",
      "score": "Score",
      "period": "Period",
      "frequency": "Frequency",
      "reGenerate": "Re-generate",
      "reGenerating": "Re-generating…",
      "reGenerateSuccess": "Re-generated successfully.",
      "reGenerateFail": "Re-generation failed. Please try again."
    },
    "tabs": {
      "upload": "Upload & Manage",
      "infer": "Inference",
      "results": "View Results",
      "fileCount": "{{count}} files",
      "selectedCount": "{{count}} selected",
      "runningCount": "{{count}} running",
      "running": "Running",
      "resultsHint": "Select a file to view results"
    },
    "workspace": {
      "filesTitle": "Files",
      "searchFiles": "Search files…",
      "loadingFiles": "Loading files…",
      "noFilesYet": "No files yet",
      "noFilesMatch": "No files match your search"
    }
  },
  "stt": {
    "pageTitle": "STT Pipeline",
    "loading": "Loading…",
    "file_one": "{{count}} file",
    "file_other": "{{count}} files",
    "processing_one": " · {{count}} processing",
    "processing_other": " · {{count}} processing",
    "dateError": "Start date cannot be after end date.",
    "apply": "Apply",
    "refreshTitle": "Refresh",
    "refreshDisabled": "Disabled during inference",
    "library": {
      "title": "Library",
      "filterAll": "All",
      "searchPlaceholder": "Search…",
      "closeInference": "Close inference",
      "runInference": "Run inference",
      "cancelDelete": "Cancel delete",
      "deleteFiles": "Delete Files",
      "cancelMove": "Cancel move",
      "movePipeline": "Move to Lecture Pipeline",
      "selected_one": "{{count}} selected",
      "selected_other": "{{count}} selected",
      "noneSelected": "Click cards to select",
      "noPipelineFiles": "No completed files to select",
      "selectCompleted": "Select completed files only",
      "selectAll": "Select all",
      "clear": "Clear",
      "inferenceSelected_one": "{{count}} selected",
      "inferenceSelected_other": "{{count}} selected",
      "inferenceNone": "Click cards to select",
      "loading": "Loading files…",
      "emptyTitle": "No files in this range",
      "emptyHint": "Try widening the date filter, or upload a new file.",
      "noMatchTitle": "No files match \"{{query}}\"",
      "noMatchHint": "Try searching by file ID or a different name.",
      "loadError": "Could not load files: {{error}}",
      "retry": "Retry",
      "deleting_one": "Deleting… ({{count}})",
      "deleting_other": "Deleting… ({{count}})",
      "delete_one": "Delete ({{count}})",
      "delete_other": "Delete ({{count}})",
      "moving_one": "Moving… ({{count}})",
      "moving_other": "Moving… ({{count}})",
      "move_one": "Move to Lecture ({{count}})",
      "move_other": "Move to Lecture ({{count}})",
      "deletedSuccess_one": "{{count}} file deleted successfully.",
      "deletedSuccess_other": "{{count}} files deleted successfully.",
      "deleteFail": "Delete failed. Please try again.",
      "movedSuccess_one": "{{count}} file moved to Lecture Pipeline.",
      "movedSuccess_other": "{{count}} files moved to Lecture Pipeline.",
      "moveFail": "Move failed. Please try again.",
      "selectForInference": "Select Files for Inference"
    },
    "fileCard": {
      "status": {
        "completed": "Completed",
        "queued": "Queued",
        "pending": "Pending",
        "running": "Running",
        "failed": "Failed"
      },
      "type": {
        "video": "Video",
        "audio": "Audio"
      },
      "generate": "Generate",
      "reGenerate": "Re-generate",
      "retry": "Retry",
      "inPipelineTitle": "Added to Lecture Pipeline"
    },
    "inference": {
      "title": "Inference",
      "nextCheck": "⟳ next check in {{sec}}s",
      "selectHint": "Select files from the Library to run inference on.",
      "selectedHint_one": "{{count}} file selected from Library",
      "selectedHint_other": "{{count}} files selected from Library",
      "settingsTitle": "Settings",
      "modelLabel": "Model",
      "modelLoading": "Loading…",
      "languageLabel": "Language",
      "korean": "Korean",
      "english": "English",
      "chunkLabel": "Chunk size",
      "chunkAuto": "Auto",
      "submitting": "Submitting…",
      "runBtn": "Run Inference",
      "runBtnCount": "Run Inference ({{count}})",
      "queued": "Queued",
      "file_one": "{{count}} file",
      "file_other": "{{count}} files",
      "waiting": "Waiting",
      "noQueued": "No queued files",
      "inQueue": "In queue",
      "running": "Running",
      "active": "Active",
      "noRunning": "No active files",
      "done": "Done",
      "noDone": "No completed files yet",
      "statusDone": "Done",
      "statusFailed": "Failed",
      "statusStarting": "Starting…"
    },
    "timeline": {
      "ariaLabel": "Transcript timeline",
      "seekTo": "Seek to {{time}}"
    },
    "unknownDate": "Unknown",
    "filterLabel": "Filter: {{filter}}"
  },
  "modal": {
    "ariaLabel": "Dialog"
  },
  "transcriptDetail": {
    "backAriaLabel": "Back",
    "fileSub": "File #{{id}} · {{count}} cues",
    "downloadSrt": "Download SRT",
    "captionsError": "Could not load captions: {{error}}",
    "retry": "Retry",
    "loadingMedia": "Loading media…",
    "mediaError": "Could not load media",
    "nowPlaying": "Now Playing",
    "saveStatus": {
      "editing": "Saving…",
      "saved": "Saved",
      "failed": "Save failed"
    },
    "cueInfo": "Cue {{current}} / {{total}} · {{start}} → {{end}}",
    "cueCountHint": "{{count}} cues · press play",
    "loading": "Loading…",
    "noTranscript": "No transcript",
    "followAlong": "Press play to follow along",
    "transcriptTitle": "Transcript",
    "transcriptHint": "Click to jump · pencil to edit",
    "loadingCaptions": "Loading captions…",
    "noCaptions": "No captions available",
    "noCaptionsToast": "No captions available",
    "nowTag": "▶ Now",
    "editTitle": "Edit",
    "emptyWarn": "⚠ Save with empty text? This will clear the caption.",
    "keepEditing": "Keep editing",
    "saveEmpty": "Save empty",
    "savedToast": "Saved",
    "saveFailedToast": "Save failed",
    "downloadedToast": "Downloaded"
  },
  "transcribePanel": {
    "title": "Generate Transcription",
    "closeAriaLabel": "Close panel",
    "fileLabel": "File",
    "modelLabel": "Model",
    "modelsLoading": "Loading models…",
    "modelsError": "Could not load models.",
    "modelsRetry": "Retry",
    "noModels": "No models available",
    "languageLabel": "Language",
    "korean": "Korean",
    "english": "English",
    "chunkLabel": "Chunk",
    "chunk5": "5 sec",
    "chunk10": "10 sec",
    "chunk15": "15 sec",
    "chunk20": "20 sec",
    "cancel": "Cancel",
    "starting": "Starting…",
    "start": "Start transcription"
  },
  "uploadInline": {
    "title": "Upload Files",
    "limitBadge": "{{active}}/{{max}} uploading",
    "limitReached": "Upload limit reached",
    "dropActive": "Drop to upload",
    "dropDefault": "Drag files or a folder here",
    "dropHint": "MP3 · WAV · AAC · MP4 · MOV · max {{max}} files",
    "limitHint": "Max {{max}} files at a time",
    "browseFiles": "Browse Files",
    "browseFolder": "Browse Folder",
    "unsupportedFormat": "Unsupported format. Accepted: MP3, WAV, AAC, FLAC, M4A, OGG, MP4, AVI, MOV, MKV, WEBM.",
    "allSlotsBusy": "All {{max}} upload slots are busy. Wait for a file to finish, then try again.",
    "partialQueue": "{{queued}} file(s) queued. {{ignored}} file(s) ignored — only {{max}} uploads run at once. You can upload more as each one completes.",
    "uploading": "Uploading…",
    "uploaded": "Uploaded",
    "failed": "Failed",
    "retryBtn": "Retry",
    "dismissAriaLabel": "Dismiss"
  },
  "uploadPanel": {
    "title": "Upload File",
    "closeAriaLabel": "Close upload panel",
    "fileLabel": "File",
    "dropActive": "Drop to attach",
    "dropDefault": "Drop a file or click to browse",
    "dropHint": "MP3 WAV AAC FLAC M4A OGG · MP4 AVI MOV MKV WEBM",
    "unsupportedFormat": "Unsupported file format",
    "removeAriaLabel": "Remove file",
    "cancel": "Cancel",
    "uploading": "Uploading…",
    "upload": "Upload"
  },
  "history": {
    "pageTitle": "History & Workspace",
    "pageSub": "Browse past inference runs and review results",
    "loading": "Loading history…",
    "loadError": "Failed to load history. Please try again.",
    "retry": "Retry",
    "empty": {
      "title": "No files yet",
      "hint": "Upload files and run inference to see results here."
    },
    "searchPlaceholder": "Search by filename or ID…",
    "filterAll": "All",
    "filterInferenced": "Inferenced",
    "filterPending": "Not inferenced",
    "sortLabel": "Sort",
    "sortNewest": "Newest first",
    "sortOldest": "Oldest first",
    "sortName": "Name",
    "fileCount": "{{count}} files",
    "selectedCount": "{{count}} selected",
    "selectAll": "Select all",
    "clearSelection": "Clear",
    "openWorkspace": "Open Workspace",
    "deleteSelected": "Delete",
    "deleting": "Deleting…",
    "deleteSuccess": "{{count}} file(s) deleted successfully.",
    "deleteFail": "Delete failed. Please try again.",
    "deleteConfirm": {
      "title": "Delete Files",
      "body": "Are you sure you want to delete {{count}} file(s)? This action cannot be undone.",
      "cancel": "Cancel",
      "confirm": "Delete"
    },
    "workspace": {
      "title": "Workspace",
      "noFile": "No file selected",
      "noFileHint": "Select a file from the list to view its inference results here.",
      "tabs": {
        "summary": "Summary",
        "keywords": "Keywords",
        "assessment": "Assessment"
      },
      "noSummary": "No summary available.",
      "noKeywords": "No keywords available.",
      "noAssessment": "No assessment questions available.",
      "copy": "Copy",
      "copied": "Copied!",
      "download": "Download",
      "edit": "Edit",
      "save": "Save",
      "saving": "Saving…",
      "cancel": "Cancel",
      "reGenerate": "Re-generate",
      "reGenerating": "Re-generating…",
      "reGenerateFail": "Re-generation failed. Please try again.",
      "reGenerateSuccess": "Re-generated successfully.",
      "keywordPlaceholder": "One keyword per line…",
      "summaryPlaceholder": "Autofilled from file — edit to override…",
      "editColPreview": "Preview",
      "editColEdit": "Edit Content",
      "markdownTag": "Markdown",
      "chipView": "Chip View",
      "onePerLine": "One per line",
      "mcq": "MCQ",
      "viewAll": "View All",
      "explanation": "Explanation",
      "assessComplete": "Assessment Complete!",
      "assessCompleteDesc": "You answered all {{count}} question(s).",
      "restart": "Restart",
      "next": "Next Question",
      "finish": "Finish",
      "question": "Q{{n}}"
    }
  }
}





















{
  "header": {
    "logoText": "콘텐츠",
    "logoTextSpan": "애널리틱스",
    "sso": "Knox SSO",
    "theme": {
      "label": "테마 선택",
      "dark": "다크",
      "light": "라이트",
      "navy": "네이비"
    },
    "userMenu": {
      "label": "사용자 메뉴",
      "administrator": "관리자",
      "logout": "로그아웃"
    }
  },
  "footer": {
    "copyright": "© {{year}} Knox University. All rights reserved."
  },
  "sidebar": {
    "expand": "사이드바 펼치기",
    "collapse": "사이드바 접기",
    "menu": "메뉴",
    "soon": "출시 예정",
    "groups": {
      "main": "메인",
      "analysis": "분석",
      "tools": "도구",
      "feedback": "피드백",
      "settings": "설정",
      "usage": "사용 현황"
    },
    "nav": {
      "lecturePipeline": "강의 파이프라인",
      "historyWorkspace": "이력 및 워크스페이스",
      "keywordInsights": "키워드 인사이트",
      "assessment": "평가 문항",
      "sttTranscription": "STT 전사",
      "promptTemplates": "프롬프트 템플릿",
      "generalSettings": "일반 설정",
      "settings": "설정",
      "adminDashboard": "관리자 대시보드",
      "voiceOfCustomer": "고객의 소리",
      "userDictionary": "사용자 사전",
      "dashboard": "대시보드"
    }
  },
  "ssoLogin": {
    "title": "콘텐츠",
    "titleSpan": "애널리틱스",
    "subtitle": "강의 인텔리전스 플랫폼",
    "signedOut": "성공적으로 로그아웃되었습니다.",
    "signInBtn": "Knox SSO로 로그인",
    "hint": "인증은 기관의 Knox SSO 서비스를 통해 자동으로 처리됩니다."
  },
  "adminDashboard": {
    "pageTitle": "대시보드",
    "userViewSub": "내 사용 통계 및 파일 활동",
    "adminViewSub": "플랫폼 전체 사용 현황, 사용자 관리 및 시스템 상태",
    "myStats": "내 통계",
    "admin": "관리자",
    "administratorBadge": "관리자",
    "readOnlyBadge": "읽기 전용 — 수정은 관리자 권한 필요",
    "loading": "대시보드 데이터 불러오는 중…",
    "loadError": "대시보드 데이터를 불러오지 못했습니다. 다시 시도해 주세요.",
    "adminLoadError": "관리자 대시보드 데이터를 불러오지 못했습니다. 다시 시도해 주세요.",
    "userStats": {
      "totalUploaded": "총 업로드 파일",
      "filesInferenced": "추론된 파일",
      "aiGenerations": "AI 생성 실행",
      "dictionariesCreated": "생성된 사전",
      "audioVideo": "{{audio}} 오디오 · {{video}} 동영상",
      "ofUploads": "업로드의 {{pct}}%",
      "kwSummaryFaq": "키워드 · 요약 · FAQ",
      "promptUpdates": "{{count}}개 프롬프트 업데이트"
    },
    "adminStats": {
      "totalUploaded": "총 업로드 파일",
      "totalUsers": "전체 사용자",
      "aiGenerations": "AI 생성",
      "dictionariesCreated": "생성된 사전",
      "activeUsers": "{{count}}명 활성 사용자",
      "activeOf": "{{count}}명 활성",
      "kwSummaryFaq": "키워드 · 요약 · FAQ",
      "promptUpdates": "{{count}}개 프롬프트 업데이트"
    },
    "tabs": {
      "overview": "개요",
      "files": "파일별 통계",
      "dictionary": "사전 사용",
      "uploadInference": "업로드 대 추론",
      "users": "사용자 활동",
      "perUser": "사용자별 상세",
      "upload_inference": "업로드 대 추론",
      "per_user": "사용자별 상세"
    },
    "charts": {
      "uploadVsInference": "유형별 업로드 대 추론",
      "uploadVsInferenceSub": "총 {{uploaded}}개 업로드 · {{inferenced}}개 추론",
      "aiBreakdown": "AI 생성 분류",
      "aiBreakdownSub": "총 {{total}}개 생성",
      "filesByType": "유형별 파일",
      "filesByTypeSub": "총 {{total}}개 업로드",
      "sttCompletions": "STT 완료",
      "sttCompletionsSub": "완료된 음성 인식 작업",
      "audioStt": "오디오 STT",
      "videoStt": "동영상 STT",
      "ofAudioUploads": "오디오 업로드의 {{pct}}%",
      "ofVideoUploads": "동영상 업로드의 {{pct}}%",
      "uploadedLabel": "업로드",
      "inferencedLabel": "추론",
      "dictUsageFreq": "사전 사용 빈도",
      "dictUsageFreqSub": "{{count}}개 사전 생성",
      "dailyUploadTrend": "일별 업로드 추이",
      "dailyUploadTrendSub": "일별 업로드 파일 수",
      "dailyGenTrend": "일별 생성 추이",
      "dailyGenTrendSub": "유형별 일별 AI 생성",
      "uploadsByType": "유형별 업로드",
      "uploadsByCategoryTitle": "카테고리별 업로드 대 추론",
      "uploadsByCategorySub": "{{uploaded}}개 업로드 · {{inferenced}}개 추론 · {{start}} → {{end}}",
      "dailyUviTrend": "일별 업로드 대 추론 추이",
      "dailyUviTrendSub": "일별 업로드 및 추론",
      "usagePerDict": "사전별 사용 횟수",
      "usageByDept": "부서별 사용 현황",
      "usageByDeptSub": "부서별 사전 수 및 사용량",
      "totalDictionaries": "전체 사전",
      "totalUsageCount": "총 사용 횟수",
      "keywordsLabel": "키워드",
      "summaryLabel": "요약",
      "faqLabel": "FAQ",
      "dictionariesLabel": "사전",
      "usageLabel": "사용"
    },
    "table": {
      "hash": "#",
      "fileName": "파일명",
      "type": "유형",
      "keywords": "키워드",
      "summaries": "요약",
      "faqs": "FAQ",
      "promptUpdates": "프롬프트 업데이트",
      "noFileStats": "파일 통계 없음",
      "dictName": "사전명",
      "timesUsed": "사용 횟수",
      "created": "생성일",
      "noDicts": "사전 없음",
      "username": "사용자명",
      "department": "부서",
      "totalFiles": "총 파일",
      "generations": "생성",
      "dictionaries": "사전",
      "lastActive": "마지막 활동",
      "noUserActivity": "사용자 활동 데이터 없음",
      "owner": "소유자",
      "category": "카테고리",
      "uploaded": "업로드",
      "inferenced": "추론",
      "coverage": "처리율",
      "noPerUserData": "사용자별 데이터 없음",
      "noAdminDicts": "사전 없음",
      "deptSummary": "부서 요약",
      "totalUsage": "총 사용"
    },
    "perUser": {
      "filesUploaded": "업로드 파일",
      "filesInferenced": "추론 파일",
      "aiGenerations": "AI 생성",
      "dictionaries": "사전",
      "lastActive": "마지막 활동: {{date}}",
      "promptUpdates": "{{count}}개 프롬프트 업데이트"
    }
  },
  "generalSettings": {
    "pageTitle": "일반 설정",
    "pageSub": "강의 처리에 적용되는 기본값",
    "loading": "설정 불러오는 중…",
    "resetToDefaults": "기본값으로 초기화",
    "resetting": "초기화 중…",
    "discard": "취소",
    "saveChanges": "변경사항 저장",
    "saving": "저장 중…",
    "savedSuccess": "설정이 성공적으로 업데이트되었습니다.",
    "resetSuccess": "설정이 기본값으로 초기화되었습니다.",
    "subtitleFormat": {
      "title": "자막 형식",
      "desc": "자막 생성 시 사용되는 기본 파일 형식."
    },
    "defaultTemplate": {
      "title": "기본 프롬프트 템플릿",
      "desc": "새 강의 업로드 시 자동으로 적용되는 템플릿.",
      "none": "없음",
      "viewAriaLabel": "템플릿 상세 보기"
    },
    "numKeywords": {
      "title": "키워드 수",
      "desc": "강의당 추출할 키워드 수.",
      "auto": "자동",
      "custom": "직접 지정"
    },
    "numQuestions": {
      "title": "평가 문항 수",
      "desc": "강의당 생성할 평가 문항 수.",
      "auto": "자동",
      "custom": "직접 지정"
    },
    "globalDict": {
      "title": "전역 사전",
      "desc": "모든 강의에서 반복되는 전사 오류를 자동으로 수정합니다.",
      "noTerms": "사전 항목이 없습니다.",
      "wrongPlaceholder": "잘못된 용어 (예: teh)",
      "correctPlaceholder": "올바른 용어 (예: the)",
      "addTerm": "항목 추가",
      "removeTerm": "항목 삭제",
      "apply": "적용",
      "manage": "사전 관리",
      "termsDefined_one": "{{count}}개 항목 정의됨",
      "termsDefined_other": "{{count}}개 항목 정의됨"
    },
    "resetModal": {
      "title": "기본값으로 초기화",
      "body": "자막 형식, 프롬프트 템플릿, 키워드/문항 수 및 전역 사전이 기본값으로 초기화됩니다. 이 작업은 취소할 수 없습니다.",
      "cancel": "취소",
      "confirm": "초기화"
    },
    "templateModal": {
      "description": "설명",
      "summaryPrompt": "요약 프롬프트",
      "keywordPrompt": "키워드 프롬프트",
      "faqPrompt": "FAQ 프롬프트",
      "close": "닫기",
      "closeAriaLabel": "닫기"
    },
    "stepper": {
      "rangeError": "{{min}}에서 {{max}} 사이의 값을 입력하세요",
      "decreaseAriaLabel": "감소",
      "increaseAriaLabel": "증가"
    }
  },
  "promptTemplates": {
    "pageTitle": "프롬프트 템플릿",
    "pageSub_one": "{{count}}개 템플릿 · 요약, 키워드 및 FAQ 생성을 위한 재사용 가능한 프롬프트",
    "pageSub_other": "{{count}}개 템플릿 · 요약, 키워드 및 FAQ 생성을 위한 재사용 가능한 프롬프트",
    "newTemplate": "새 템플릿",
    "loading": "템플릿 불러오는 중…",
    "createdSuccess": "템플릿이 생성되었습니다.",
    "updatedSuccess": "템플릿이 수정되었습니다.",
    "deletedSuccess": "템플릿이 삭제되었습니다.",
    "genericError": "오류가 발생했습니다. 다시 시도해 주세요.",
    "allFieldsRequired": "모든 필드를 입력해야 합니다.",
    "table": {
      "name": "이름",
      "description": "설명",
      "updated": "수정일",
      "empty": "아직 템플릿이 없습니다 — 첫 번째 템플릿을 생성하여 요약, 키워드 및 FAQ 프롬프트를 표준화하세요.",
      "edit": "수정",
      "delete": "삭제",
      "deleting": "삭제 중…"
    },
    "modal": {
      "createTitle": "새 템플릿",
      "editTitle": "템플릿 수정",
      "name": "이름",
      "namePlaceholder": "예: 강의 요약 — 기본",
      "description": "설명",
      "descPlaceholder": "이 템플릿을 사용하는 상황에 대한 짧은 설명",
      "summaryPrompt": "요약 프롬프트",
      "summaryPlaceholder": "요약 생성에 사용되는 지시사항",
      "keywordPrompt": "키워드 프롬프트",
      "keywordPlaceholder": "키워드 추출에 사용되는 지시사항",
      "faqPrompt": "FAQ 프롬프트",
      "faqPlaceholder": "FAQ 생성에 사용되는 지시사항",
      "required": "*",
      "cancel": "취소",
      "saving": "저장 중…",
      "saveChanges": "변경사항 저장",
      "createBtn": "템플릿 생성",
      "closeAriaLabel": "닫기"
    },
    "deleteModal": {
      "title": "템플릿 삭제",
      "body": "<strong>{{name}}</strong>을(를) 삭제하시겠습니까? 이 작업은 취소할 수 없습니다.",
      "cancel": "취소",
      "confirm": "삭제"
    }
  },
  "settings": {
    "pageTitle": "사용자 사전",
    "pageSub": "전사 및 추론 중 적용되는 사용자 정의 용어 교체 관리",
    "dictionaries": "사전",
    "newDictTooltip": "새 사전",
    "loading": "불러오는 중…",
    "noDict": "사전이 없습니다.\n+ 버튼을 클릭하여 생성하세요.",
    "unsaved": "저장되지 않음",
    "newDict": "새 사전",
    "selectOrCreate": "사전을 선택하거나 새로 만드세요.",
    "dictNameLabel": "사전 이름",
    "dictNamePlaceholder": "사전 이름을 입력하세요…",
    "updated": "{{date}} 업데이트됨",
    "cancel": "취소",
    "saving": "저장 중…",
    "saved": "저장됨",
    "create": "생성",
    "saveChanges": "변경사항 저장",
    "discardConfirm": "저장하지 않은 변경사항을 버리시겠습니까?",
    "nameRequired": "사전 이름은 필수입니다",
    "termRequired": "두 값이 모두 입력된 항목이 최소 하나 있어야 합니다",
    "searchPlaceholder": "용어 검색…",
    "term_one": "{{count}}개 항목",
    "term_other": "{{count}}개 항목",
    "addTerm": "항목 추가",
    "wrongTerm": "잘못된 용어",
    "correctTerm": "올바른 용어",
    "wrongPlaceholder": "예: teh",
    "correctPlaceholder": "예: the",
    "removeTerm": "항목 삭제",
    "noSearchMatch": "검색 결과가 없습니다",
    "noTermsYet": "아직 항목이 없습니다 — \"항목 추가\"를 클릭하여 생성하세요",
    "dictInfo": "사전은 STT 전사 및 추론 중 적용됩니다. 전사본에서 발견된 잘못된 용어는 올바른 용어로 교체됩니다."
  },
  "landing": {
    "badge": "강의 인텔리전스 플랫폼 · v2.4.1",
    "hero": {
      "title1": "강의 스크립트를",
      "title2": "체계적인",
      "titleAccent": "지식",
      "title3": "으로 변환하세요",
      "desc": "Content Analytics는 VTT 및 SRT 파일을 수집하고 AI 추론을 실행하여 강의당 1분 이내에 요약, 키워드 맵, 평가 문항을 제공합니다.",
      "ctaPrimary": "강의 파이프라인",
      "ctaSecondary": "워크스페이스 보기"
    },
    "stats": {
      "fasterReview": "빠른 콘텐츠 검토",
      "groundingRate": "문항 근거 비율",
      "maxAudio": "STT 최대 오디오",
      "filesPerBatch": "배치당 파일 수"
    },
    "features": {
      "sectionLabel": "주요 기능",
      "sectionTitle": "파이프라인의 모든 단계를,\n자동으로 처리합니다",
      "batchUpload": {
        "title": "일괄 업로드",
        "desc": "VTT 및 SRT 스크립트 파일을 드래그 앤 드롭으로 대량 업로드. 파일당 최대 50MB, 최대 100개 파일, 폴더 단위 수집 지원."
      },
      "aiInference": {
        "title": "AI 추론",
        "desc": "대형 언어 모델 기반 — 구조화된 요약, 타임스탬프 콘텐츠 아웃라인, 키워드 빈도 맵을 생성합니다."
      },
      "smartSummaries": {
        "title": "스마트 요약",
        "desc": "목차, 타임스탬프 핵심 내용, 종합 개요 — 세 가지 요약 모드를 한국어와 영어로 제공합니다."
      },
      "assessmentGen": {
        "title": "평가 문항 생성",
        "desc": "반환각(hallucination) 검증 기반의 객관식, 진위형, 단답형 문항을 자동 생성 — 91% 이상의 콘텐츠 근거율."
      },
      "keywordInsights": {
        "title": "키워드 인사이트",
        "desc": "빈도 분석, 지식 그래프, 워드 클라우드로 단일 파일 또는 전체 강의 세트의 핵심 개념을 파악합니다."
      },
      "workspaceHistory": {
        "title": "워크스페이스 이력",
        "desc": "모든 추론 실행의 완전한 감사 추적 — 결과를 다시 열고, 출력을 재생성하고, 클릭 한 번으로 엑셀로 내보내기."
      }
    },
    "workflow": {
      "steps": {
        "upload": {
          "label": "업로드",
          "sub": "VTT / SRT 파일 드롭"
        },
        "configure": {
          "label": "설정",
          "sub": "프롬프트 및 출력 설정"
        },
        "infer": {
          "label": "추론",
          "sub": "AI 파이프라인 실행"
        },
        "review": {
          "label": "검토",
          "sub": "워크스페이스 열기"
        },
        "export": {
          "label": "내보내기",
          "sub": "엑셀 다운로드"
        }
      }
    },
    "ctaSection": {
      "title": "강의 분석을 시작할 준비가 되셨나요?",
      "desc": "첫 번째 스크립트를 업로드하면 60초 이내에 AI 생성 요약, 키워드 맵, 문항 세트를 받아보세요.",
      "button": "강의 파이프라인"
    },
    "mockCard": {
      "tabs": {
        "summary": "요약",
        "keywords": "키워드",
        "questions": "문항"
      },
      "status": "완료",
      "block1Title": "1. 알고리즘의 정의와 분류",
      "block2Title": "2. 정렬 알고리즘 비교 분석",
      "kw1": "알고리즘 18%",
      "kw2": "시간복잡도 14%",
      "kw3": "Big-O 8%",
      "kw4": "재귀 7%"
    }
  },
  "uploadInfer": {
    "pageTitle": "강의 파이프라인",
    "pageSub": "1단계 — VTT/SRT 파일 업로드 · 2단계 — 추론 · 3단계 — 워크스페이스 결과",
    "tour": {
      "takeTour": "둘러보기",
      "tabsTitle": "언제든 이동 가능한 3단계",
      "tabsBody": "업로드 및 관리, 추론, 결과 보기 — 언제든지 자유롭게 탭을 이동할 수 있으며, 이전 단계를 완료해야만 다음으로 넘어갈 수 있는 제약은 없습니다.",
      "uploadTitle": "여기에 파일을 추가하세요",
      "uploadBody": "강의 녹화본의 자막 파일(.vtt 또는 .srt)을 끌어다 놓거나 찾아보기로 선택하세요. 여러 파일을 한 번에 넣을 수도 있습니다.",
      "cardsTitle": "업로드한 파일 목록",
      "cardsBody": "업로드한 파일은 이렇게 카드 형태로 표시됩니다. 모서리의 색상 배지는 파일 형식(VTT/SRT)을 나타내고, 파일명 아래의 작은 아이콘들은 각 콘텐츠 유형(요약, 키워드, 질문, 답변, 참/거짓)에 사용된 프롬프트를 보여줍니다. 하단 배지는 파일의 현재 상태를 나타내며, 사전이나 프롬프트 템플릿이 연결된 파일에는 작은 책 또는 체크리스트 아이콘이 함께 표시됩니다.",
      "cardsBodyEmpty": "아직 업로드한 파일이 없어서 이 영역은 비어 있습니다. 파일을 업로드하면 각 파일이 카드로 표시되며, 형식 배지(VTT/SRT), 파일명, 콘텐츠 유형별 프롬프트 아이콘(요약, 키워드, 질문, 답변, 참/거짓), 사전이나 프롬프트 템플릿이 연결된 경우 책/체크리스트 아이콘, 그리고 하단의 상태 배지를 확인할 수 있습니다.",
      "iconSummaryTitle": "요약 아이콘",
      "iconSummaryBody": "이 파일의 요약을 생성할 때 사용된 프롬프트를 엽니다. 파란색이면 이 파일에 맞게 커스터마이징된 것이고, 회색이면 기본값을 그대로 사용 중이라는 뜻입니다.",
      "iconKeywordsTitle": "키워드 아이콘",
      "iconKeywordsBody": "이 파일의 키워드를 추출할 때 사용된 프롬프트를 엽니다 — 태그 모양의 아이콘입니다.",
      "iconQuestionsTitle": "평가 질문 아이콘",
      "iconQuestionsBody": "퀴즈 형식의 평가 질문을 생성할 때 사용된 프롬프트를 엽니다 — 물음표 모양의 아이콘입니다.",
      "iconAnswerTitle": "단답형 답변 아이콘",
      "iconAnswerBody": "짧은 서술형 답변을 생성할 때 사용된 프롬프트를 엽니다 — 말풍선 모양의 아이콘입니다.",
      "iconTrueFalseTitle": "참/거짓 아이콘",
      "iconTrueFalseBody": "참/거짓 질문을 생성할 때 사용된 프롬프트를 엽니다 — 토글 스위치 모양의 아이콘입니다.",
      "sortTitle": "목록 정렬하기",
      "sortBody": "파일을 ID, 이름, 날짜, 상태 기준으로 정렬할 수 있습니다 — 같은 항목을 다시 클릭하면 오름차순/내림차순이 바뀝니다.",
      "dateFilterTitle": "날짜 범위로 필터링",
      "dateFilterBody": "파일 목록을 특정 날짜 범위로 좁힌 뒤 적용 버튼을 누르세요. 이 필터는 업로드 탭에서 보이는 목록에만 적용됩니다.",
      "statusFilterTitle": "상태로 필터링",
      "statusFilterBody": "전체, 대기, 대기열, 실행 중, 추론 완료, 오류, 보류 중 하나의 상태로 파일을 필터링할 수 있습니다 — 처리 중인 파일이 많을 때 유용합니다.",
      "actionsTitle": "한 번의 클릭으로 실행하는 일괄 작업",
      "actionsBody": "이제 5가지 일괄 작업을 모두 확인할 수 있습니다: 사전(파일에 용어 사전 연결), 프롬프트 템플릿(저장된 프롬프트 세트 연결), 검색(파일명으로 찾기), 내보내기(파일 다운로드), 삭제(파일 제거) — 모두 여러 파일에 한 번에 적용됩니다.",
      "actionDictionaryTitle": "사전",
      "actionDictionaryBody": "하나 이상의 파일에 용어 사전을 연결하여, 추론 시 해당 분야에 맞는 정확한 철자와 용어를 사용하도록 합니다.",
      "actionTemplateTitle": "프롬프트 템플릿",
      "actionTemplateBody": "저장해 둔 프롬프트 세트를 하나 이상의 파일에 한 번에 적용합니다 — 파일마다 프롬프트를 일일이 수정할 필요가 없습니다.",
      "actionSearchTitle": "검색",
      "actionSearchBody": "파일명으로 파일을 찾습니다 — 업로드 탭의 파일 목록에 대한 검색창이 열립니다.",
      "actionExportTitle": "내보내기",
      "actionExportBody": "파일을 선택해 다운로드합니다 — 앱 외부에서 백업하거나 결과를 공유할 때 유용합니다.",
      "actionDeleteTitle": "삭제",
      "actionDeleteBody": "파일을 선택해 영구적으로 삭제합니다 — 되돌릴 수 없으니 신중하게 사용하세요.",
      "inferSidebarTitle": "분석할 파일 선택",
      "inferSidebarBody": "여기서 하나 이상의 파일을 선택하세요 — 이 목록은 업로드 탭과 별개로 자체 날짜 범위, 상태 필터, 검색, 정렬 기능을 가지고 있습니다.",
      "inferSettingsTitle": "생성할 항목 선택",
      "inferSettingsBody": "요약, 키워드(키워드 인사이트 옵션 포함), 평가 질문, 단답형 답변, 참/거짓 중에서 선택하세요 — 각 항목은 개별적으로 프롬프트를 커스터마이징할 수 있습니다. 파일을 하나 이상 선택하기 전까지는 이 옵션들이 선택 해제 상태로 비활성화되어 있습니다.",
      "inferRunTitle": "실행하기",
      "inferRunBody": "파일과 설정을 모두 선택했다면 여기서 배치를 실행하세요. 진행 상황을 지켜보거나 다른 탭으로 이동해도 작업은 계속 진행됩니다.",
      "resultsSidebarTitle": "완료된 파일 찾기",
      "resultsSidebarBody": "파일 처리가 끝나면 여기서 클릭해 결과를 확인하세요. 이 목록에도 자체 날짜 범위, 상태 필터, 검색, 정렬 기능이 있습니다.",
      "resultsContentTitle": "바로 활용할 수 있는 결과물",
      "resultsContentBody": "요약, 키워드, 퀴즈 질문 등 선택한 파일에서 생성된 모든 콘텐츠를 확인할 수 있습니다. 전체 배치를 다시 실행하지 않고도 개별 항목을 수정하거나 재생성할 수 있습니다."
    },
    "batchGroups": {
      "batchTitle": "배치 #{{id}}",
      "done": "{{done}} / {{total}} 완료",
      "running": "실행 중",
      "statusTitles": {
        "running": "실행 중",
        "queued": "대기 중",
        "done": "완료",
        "pending": "선택 안 됨"
      },
      "badgeLabels": {
        "running": "활성",
        "queued": "대기",
        "done": "완료",
        "pending": "건너뜀"
      },
      "file_one": "{{count}}개 파일",
      "file_other": "{{count}}개 파일",
      "inQueue": "대기열에 있음",
      "emptyMsg": "이 배치에서 제외된 파일이 없습니다.",
      "view": "보기"
    },
    "filePanel": {
      "step1Label": "1단계 — 파일",
      "uploaded_one": "{{count}}개 업로드됨",
      "uploaded_other": "{{count}}개 업로드됨",
      "selected_one": "{{count}}개 선택됨",
      "selected_other": "{{count}}개 선택됨",
      "showUpload": "업로드 표시",
      "hideUpload": "업로드 숨기기",
      "dropTitle": "여기에 파일을 드롭하세요",
      "dropTitleDefault": ".vtt / .srt 파일을 여기에 드롭하세요",
      "dropSub": "또는 클릭하여 탐색 · 폴더 업로드 지원",
      "browseFiles": "파일 탐색",
      "folder": "폴더",
      "uploadBtn_one": "{{count}}개 파일 업로드",
      "uploadBtn_other": "{{count}}개 파일 업로드",
      "cancelBtn": "취소",
      "complete": "{{finished}} / {{total}} 완료",
      "failed_one": "· {{count}}개 실패",
      "failed_other": "· {{count}}개 실패",
      "fileStatus": {
        "ready": "업로드 준비 완료",
        "uploading": "업로드 중…",
        "uploaded": "업로드됨"
      },
      "uploadedFiles": "업로드된 파일",
      "inferenceBtn": "추론을 위한 파일 선택",
      "dictionaryBtn": "사전 연결 / 해제",
      "templateBtn": "프롬프트 템플릿 매핑",
      "searchBtn": "파일 검색",
      "exportBtn": "엑셀로 내보내기",
      "deleteBtn": "파일 삭제",
      "dateFrom": "시작일",
      "dateTo": "종료일",
      "applyDate": "적용",
      "sortBy": "정렬 기준",
      "sortId": "ID",
      "sortName": "이름",
      "sortDate": "날짜",
      "sortStatus": "상태",
      "searchPlaceholder": "파일명 또는 ID로 검색…",
      "loadingFiles": "파일 불러오는 중…",
      "noFilesRange": "이 날짜 범위에 파일이 없습니다",
      "noFilesMatch": "{{query}}와 일치하는 파일이 없습니다",
      "inferenced": "추론됨",
      "notInferenced": "추론 안 됨",
      "export": "내보내기",
      "exportToExcelTitle": "{{count}}개 파일을 Excel로 내보내기",
      "exportSelectFirst": "내보낼 파일을 선택하세요",
      "selectFilesToExport": "내보낼 파일을 선택하세요",
      "exportCountTitle": "{{count}}개 파일을 Excel로 내보내기",
      "selectFilesToDelete": "삭제할 파일을 선택하세요",
      "deleteCountTitle": "{{count}}개 파일 삭제",
      "exportCount": "{{count}}개 내보내기",
      "deleteCount": "{{count}}개 삭제",
      "cancelTitle": "취소",
      "uploaded": "업로드됨 ({{count}})",
      "selected": "{{count}}개 선택됨",
      "uploadBtn": "{{count}}개 파일 업로드",
      "failed": "· {{count}}개 실패",
      "exportAllBtn": "전체 내보내기",
      "exportAllFailed": "내보내기에 실패했습니다. 다시 시도해 주세요.",
      "deleteAllBtn": "전체 삭제",
      "deleteAllConfirmTitle": "모든 파일 삭제",
      "deleteAllConfirmBody": "{{from}}부터 {{to}}까지 업로드된 모든 파일이 영구적으로 삭제됩니다. 이 작업은 취소할 수 없습니다.",
      "deleteAllConfirmPrompt": "확인을 위해 <strong>{{phrase}}</strong>을(를) 입력하세요:",
      "deleteAllConfirmBtn": "모든 파일 삭제",
      "deleteAllFailed": "삭제에 실패했습니다. 다시 시도해 주세요.",
      "matchCount": "파일 {{count}}개",
      "pageInfo": "{{page}} / {{totalPages}} 페이지 · 총 {{total}}개",
      "perPage": "페이지당",
      "selectedTotal": "{{total}}개 중 {{count}}개 선택됨",
      "selectAllFiles": "전체 선택",
      "clearAllFiles": "전체 해제",
      "inferenceMode": "추론을 위해 선택",
      "deleteMode": "파일 삭제",
      "exportMode": "파일 내보내기",
      "exitMode": "나가기",
      "noFilesSelected": "선택된 파일 없음",
      "runInferenceBtn": "추론 실행",
      "confirmDeleteBtn": "삭제 확인",
      "confirmExportBtn": "내보내기 확인",
      "paginationPrev": "이전",
      "paginationNext": "다음",
      "actionsLabel": "작업",
      "continueToInfer": "추론으로 계속",
      "continueToInferCount": "추론으로 계속 ({{count}})",
      "dictionaryLinked": "사전 연결됨",
      "templateLinked": "프롬프트 템플릿 연결됨",
      "promptCustomized": "사용자 지정",
      "promptDefault": "기본값",
      "promptDefaultFull": "기본 프롬프트 사용 중",
      "perPageShort": "{{count}}개 / 페이지",
      "statusAll": "전체",
      "statusWaiting": "대기 중",
      "statusQueued": "대기열",
      "statusRunning": "실행 중",
      "statusError": "오류",
      "statusErrorBadge": "오류 (다시 추론)",
      "statusTbd": "미정"
    },
    "inferencePanel": {
      "step2Label": "2단계 — 추론 설정",
      "step2Running": "2단계 — 추론 실행 중",
      "expandStep2": "Step 2 펼치기",
      "minimizeStep2": "Step 2 최소화",
      "closeStep2": "Step 2 닫기",
      "filesSelected_one": "추론을 위해 {{count}}개 파일 선택됨",
      "filesSelected_other": "추론을 위해 {{count}}개 파일 선택됨",
      "batchRunning": "배치 실행 중",
      "runInference": "추론 실행",
      "selectFilesFirst": "먼저 파일을 선택하세요",
      "noModelSelected": "모델이 선택되지 않음",
      "submitting": "배치 제출 중…",
      "submittingDesc": "추론 엔진으로 {{count}}개 파일을 전송 중입니다.\n잠시 기다려 주세요.",
      "submittingMore": "+{{count}}개 더",
      "selBanner": "{{count}}개 파일",
      "selBannerEmpty": "왼쪽 패널에서 파일을 선택하세요",
      "selBannerFilled": "추론을 위해 {{count}}개 파일이 선택됨",
      "generateContent": "콘텐츠 생성",
      "generateSummary": "요약",
      "generateKeywords": "키워드",
      "generateQuestions": "평가 문항",
      "outputSettings": "출력 설정",
      "modelLabel": "모델",
      "refreshModels": "모델 새로고침",
      "noModels": "사용 가능한 모델 없음 — 추론을 시작할 수 없습니다. 새로고침하거나 연결을 확인하세요.",
      "modelsAvailable_one": "{{count}}개 모델 사용 가능",
      "modelsAvailable_other": "{{count}}개 모델 사용 가능",
      "noModelsOption": "— 모델을 찾을 수 없음 —",
      "promptOverrides": "프롬프트 덮어쓰기",
      "promptModeSingle": "단일 파일 — 자동 입력됨 · 변경하여 덮어쓰기.",
      "promptModeMulti": "여러 파일 — 여기에 입력하면 모두 덮어씁니다.",
      "summaryPrompt": "요약 프롬프트",
      "keywordPrompt": "키워드 프롬프트",
      "questionPrompt": "평가 문항 프롬프트",
      "optional": "선택사항",
      "perFile": "파일별",
      "hide": "숨기기",
      "noPromptSet": "프롬프트 없음",
      "autofillPlaceholder": "파일에서 자동 입력됨 — 변경하여 덮어쓰기…",
      "manualPlaceholder": "파일별 프롬프트를 사용하려면 비워두세요…",
      "noContentHint": "프롬프트를 설정하려면 콘텐츠 유형을 하나 이상 활성화하세요.",
      "cancelPrompt": "취소",
      "savePrompts": "프롬프트 저장",
      "saving": "저장 중…",
      "promptSaveFail": "프롬프트 저장에 실패했습니다. 다시 시도해 주세요.",
      "inferenceStatus": "추론 상태",
      "live": "라이브",
      "refreshingIn": "{{sec}}초 후 새로고침",
      "filesRunning_one": "{{count}}개 파일 실행 중",
      "filesRunning_other": "{{count}}개 파일 실행 중",
      "queued": "대기 중",
      "runningCol": "실행 중",
      "completed": "완료",
      "waitingQueue": "대기열에 있음",
      "stopFile": "이 파일 중지",
      "stopAlready": "이 파일은 이미 처리되어 중지할 수 없습니다.",
      "inferenceStarted": "추론이 시작되었습니다",
      "batchRunPill": "배치 실행 중",
      "collapsePerFile": "파일별 프롬프트 접기",
      "viewPerFile": "파일별 프롬프트 보기",
      "filesSelected": "추론을 위해 {{count}}개 파일 선택됨",
      "filesRunning": "{{count}}개 파일 실행 중",
      "modelsAvailable": "{{count}}개 모델 사용 가능",
      "generateKeywordInsights": "키워드 인사이트",
      "generateShortAnswer": "단답형 문항",
      "generateTrueFalse": "진위형 문항",
      "timestampedSummary": "타임스탬프 요약",
      "timeInterval": "시간 간격",
      "minutesOption": "{{count}}분",
      "shortAnswerPrompt": "단답형 프롬프트",
      "trueFalsePrompt": "진위형 프롬프트"
    },
    "dictModal": {
      "title": "사전 연결",
      "subtitle": "각 파일에 사전을 할당하거나, 모든 파일에 한 번에 적용하세요.",
      "closeSaving": "저장 중 — 잠시 기다려 주세요",
      "close": "닫기",
      "applyToAll": "모든 파일에 적용:",
      "applyBtn": "모두에 적용",
      "noneOption": "— 없음 (초기화) —",
      "noneRow": "— 없음 —",
      "changes_one": "{{count}}개 변경 사항 대기 중",
      "changes_other": "{{count}}개 변경 사항 대기 중",
      "noChanges": "변경 사항 없음",
      "colFile": "파일",
      "colDict": "사전",
      "loadingDicts": "사전 불러오는 중…",
      "noFiles": "파일이 없습니다.",
      "loadFail": "사전을 불러오지 못했습니다. 다시 시도해 주세요.",
      "associated": "연결됨",
      "viewDetails": "사전 상세 보기",
      "disassociate": "연결 해제",
      "detailClose": "상세 닫기",
      "metaId": "ID:",
      "metaUpdated": "수정일:",
      "metaTerms": "항목 수:",
      "loadingDetails": "상세 정보 불러오는 중…",
      "noTerms": "이 사전에 항목이 없습니다.",
      "detailLoadFail": "사전 상세 정보를 불러오지 못했습니다.",
      "noDetailsReturned": "이 사전의 상세 정보가 반환되지 않았습니다.",
      "wrongTerm": "잘못된 용어",
      "correctTerm": "올바른 용어",
      "savingHint": "저장 중 — 이 창을 닫지 마세요…",
      "cancel": "취소",
      "save": "저장",
      "saving": "저장 중…",
      "saveSuccess": "사전 설정이 성공적으로 저장되었습니다",
      "saveFail": "사전 설정 저장에 실패했습니다. 다시 시도해 주세요.",
      "associateAllLabel": "모든 파일에 사전 적용 ({{from}} – {{to}})",
      "associateAllBtn": "모든 파일에 적용",
      "associateAllSuccess": "모든 파일에 사전이 성공적으로 적용되었습니다.",
      "associateAllFail": "모든 파일에 사전 적용에 실패했습니다. 다시 시도해 주세요.",
      "changes": "{{count}}개 변경 사항 대기 중"
    },
    "templateModal": {
      "title": "프롬프트 템플릿 연결",
      "subtitle": "각 파일에 프롬프트 템플릿을 매핑하여 요약, 키워드, FAQ 프롬프트를 업데이트하세요.",
      "closeSaving": "저장 중 — 잠시 기다려 주세요",
      "close": "닫기",
      "applyToAll": "모든 파일에 적용:",
      "applyBtn": "모두에 적용",
      "noneOption": "— 없음 —",
      "noneRow": "— 없음 —",
      "changes_one": "{{count}}개 변경 사항 대기 중",
      "changes_other": "{{count}}개 변경 사항 대기 중",
      "noChanges": "변경 사항 없음",
      "colFile": "파일",
      "colTemplate": "프롬프트 템플릿",
      "loadingTemplates": "프롬프트 템플릿 불러오는 중…",
      "loadFail": "프롬프트 템플릿을 불러오지 못했습니다. 다시 시도해 주세요.",
      "noFiles": "파일이 없습니다.",
      "mapped": "매핑됨",
      "viewDetails": "템플릿 상세 보기",
      "detailClose": "상세 닫기",
      "metaId": "ID:",
      "metaUpdated": "수정일:",
      "metaDescription": "설명:",
      "loadingDetails": "상세 정보 불러오는 중…",
      "noDetailsReturned": "이 템플릿의 상세 정보가 반환되지 않았습니다.",
      "detailLoadFail": "템플릿 상세 정보를 불러오지 못했습니다.",
      "summaryPrompt": "요약 프롬프트",
      "keywordPrompt": "키워드 프롬프트",
      "faqPrompt": "FAQ 프롬프트",
      "savingHint": "저장 중 — 이 창을 닫지 마세요…",
      "cancel": "취소",
      "save": "저장",
      "saving": "저장 중…",
      "saveSuccess": "프롬프트 템플릿이 성공적으로 적용되었습니다",
      "saveFail": "프롬프트 템플릿 적용에 실패했습니다. 다시 시도해 주세요.",
      "associateAllLabel": "모든 파일에 템플릿 적용 ({{from}} – {{to}})",
      "associateAllBtn": "모든 파일에 적용",
      "associateAllSuccess": "모든 파일에 템플릿이 성공적으로 적용되었습니다.",
      "associateAllFail": "모든 파일에 템플릿 적용에 실패했습니다. 다시 시도해 주세요.",
      "changes": "{{count}}개 변경 사항 대기 중",
      "shortAnswerPrompt": "단답형 프롬프트",
      "trueFalsePrompt": "진위형 프롬프트"
    },
    "workspacePanel": {
      "clickToView": "결과를 보려면 파일을 클릭하세요",
      "noFileSelected": "파일이 선택되지 않음",
      "noFileDesc": "업로드된 파일을 클릭하면 추론 결과가 여기에 표시됩니다.",
      "loadingFile": "파일 데이터 불러오는 중…",
      "tabs": {
        "summary": "요약",
        "keywords": "키워드",
        "assessment": "평가 문항",
        "shortAnswer": "단답형",
        "trueFalse": "진위형",
        "timestampedSummary": "타임스탬프 요약",
        "keywordInsights": "키워드 인사이트",
        "mcq": "객관식"
      },
      "noSummary": "요약을 사용할 수 없습니다.",
      "noKeywords": "키워드를 사용할 수 없습니다.",
      "noQuestions": "문항을 사용할 수 없습니다.",
      "edit": "편집",
      "copy": "복사",
      "download": "다운로드",
      "copyAs": "형식으로 복사",
      "downloadAs": "형식으로 다운로드",
      "editColPreview": "미리보기",
      "editColEdit": "내용 편집",
      "markdownRenderer": "Markdown 렌더러",
      "markdownTag": "Markdown",
      "chipView": "칩 뷰",
      "onePerLine": "한 줄에 하나씩",
      "cancel": "취소",
      "save": "저장",
      "saving": "저장 중",
      "autofillPlaceholder": "파일에서 자동 입력됨 — 편집하여 덮어쓰기…",
      "keywordPlaceholder": "키워드를 한 줄에 하나씩…",
      "mcq": "MCQ",
      "viewAll": "전체 보기",
      "explanation": "설명",
      "complete": "완료!",
      "assessCompleteTitle": "평가 완료!",
      "assessCompleteDesc": "{{count}}개의 문항을 모두 완료했습니다.",
      "restart": "다시 시작",
      "next": "다음 문항",
      "finish": "완료",
      "question": "Q{{n}}",
      "answer": "정답",
      "true": "참",
      "false": "거짓",
      "revealAnswer": "정답 보기",
      "showAllAnswers": "전체 정답 보기",
      "hideAllAnswers": "전체 정답 숨기기",
      "quizMode": "퀴즈 모드",
      "noShortAnswer": "단답형 문항이 없습니다.",
      "noTrueFalse": "진위형 문항이 없습니다.",
      "noTimestampedSummary": "타임스탬프 요약이 없습니다.",
      "noKeywordInsights": "키워드 인사이트가 없습니다.",
      "noData": "데이터가 없습니다.",
      "firstMentionAt": "첫 언급 {{pct}}%",
      "importanceAxis": "중요도",
      "kiTabs": {
        "graph": "지식 그래프",
        "wordcloud": "워드 클라우드",
        "timeline": "타임라인",
        "heatmap": "히트맵",
        "clusters": "클러스터",
        "frequency": "빈도",
        "prerequisites": "선수 지식",
        "importance": "중요도",
        "cooccurrence": "동시 출현",
        "glossary": "용어집"
      },
      "scrollLeft": "탭 왼쪽으로 스크롤",
      "scrollRight": "탭 오른쪽으로 스크롤",
      "complexityAxis": "복잡도",
      "glossaryTerm": "용어",
      "glossaryDefinition": "정의",
      "reason": "이유",
      "cluster": "클러스터",
      "count": "횟수",
      "copied": "복사됨!",
      "score": "점수",
      "period": "기간",
      "frequency": "빈도",
      "reGenerate": "재생성",
      "reGenerating": "재생성 중…",
      "reGenerateSuccess": "재생성이 완료되었습니다.",
      "reGenerateFail": "재생성에 실패했습니다. 다시 시도해 주세요."
    },
    "tabs": {
      "upload": "업로드 및 관리",
      "infer": "추론",
      "results": "결과 보기",
      "fileCount": "파일 {{count}}개",
      "selectedCount": "{{count}}개 선택됨",
      "runningCount": "{{count}}개 실행 중",
      "running": "실행 중",
      "resultsHint": "파일을 선택하여 결과 보기"
    },
    "workspace": {
      "filesTitle": "파일",
      "searchFiles": "파일 검색…",
      "loadingFiles": "파일 불러오는 중…",
      "noFilesYet": "파일이 없습니다",
      "noFilesMatch": "검색 결과가 없습니다"
    }
  },
  "stt": {
    "pageTitle": "STT 파이프라인",
    "loading": "불러오는 중…",
    "file_one": "{{count}}개 파일",
    "file_other": "{{count}}개 파일",
    "processing_one": " · {{count}}개 처리 중",
    "processing_other": " · {{count}}개 처리 중",
    "dateError": "시작일이 종료일보다 늦을 수 없습니다.",
    "apply": "적용",
    "refreshTitle": "새로고침",
    "refreshDisabled": "추론 중 비활성화됨",
    "library": {
      "title": "라이브러리",
      "filterAll": "전체",
      "searchPlaceholder": "검색…",
      "closeInference": "추론 닫기",
      "runInference": "추론 실행",
      "cancelDelete": "삭제 취소",
      "deleteFiles": "파일 삭제",
      "cancelMove": "이동 취소",
      "movePipeline": "강의 파이프라인으로 이동",
      "selected_one": "{{count}}개 선택됨",
      "selected_other": "{{count}}개 선택됨",
      "noneSelected": "클릭하여 선택",
      "noPipelineFiles": "선택 가능한 완료 파일 없음",
      "selectCompleted": "완료된 파일만 선택 가능",
      "selectAll": "전체 선택",
      "clear": "초기화",
      "inferenceSelected_one": "{{count}}개 선택됨",
      "inferenceSelected_other": "{{count}}개 선택됨",
      "inferenceNone": "클릭하여 선택",
      "loading": "파일 불러오는 중…",
      "emptyTitle": "이 기간에 파일이 없습니다",
      "emptyHint": "날짜 필터를 넓히거나 새 파일을 업로드하세요.",
      "noMatchTitle": "\"{{query}}\"와 일치하는 파일이 없습니다",
      "noMatchHint": "파일 ID 또는 다른 이름으로 검색해 보세요.",
      "loadError": "파일을 불러오지 못했습니다: {{error}}",
      "retry": "재시도",
      "deleting_one": "삭제 중… ({{count}}개)",
      "deleting_other": "삭제 중… ({{count}}개)",
      "delete_one": "삭제 ({{count}})",
      "delete_other": "삭제 ({{count}})",
      "moving_one": "이동 중… ({{count}}개)",
      "moving_other": "이동 중… ({{count}}개)",
      "move_one": "강의로 이동 ({{count}})",
      "move_other": "강의로 이동 ({{count}})",
      "deletedSuccess_one": "{{count}}개 파일이 삭제되었습니다.",
      "deletedSuccess_other": "{{count}}개 파일이 삭제되었습니다.",
      "deleteFail": "삭제에 실패했습니다. 다시 시도해 주세요.",
      "movedSuccess_one": "{{count}}개 파일이 강의 파이프라인으로 이동되었습니다.",
      "movedSuccess_other": "{{count}}개 파일이 강의 파이프라인으로 이동되었습니다.",
      "moveFail": "이동에 실패했습니다. 다시 시도해 주세요.",
      "selectForInference": "추론을 위한 파일 선택"
    },
    "fileCard": {
      "status": {
        "completed": "완료",
        "queued": "대기 중",
        "pending": "대기",
        "running": "실행 중",
        "failed": "실패"
      },
      "type": {
        "video": "비디오",
        "audio": "오디오"
      },
      "generate": "생성",
      "reGenerate": "재생성",
      "retry": "재시도",
      "inPipelineTitle": "강의 파이프라인에 추가됨"
    },
    "inference": {
      "title": "추론",
      "nextCheck": "⟳ {{sec}}초 후 확인",
      "selectHint": "추론을 실행할 파일을 라이브러리에서 선택하세요.",
      "selectedHint_one": "라이브러리에서 {{count}}개 파일 선택됨",
      "selectedHint_other": "라이브러리에서 {{count}}개 파일 선택됨",
      "settingsTitle": "설정",
      "modelLabel": "모델",
      "modelLoading": "불러오는 중…",
      "languageLabel": "언어",
      "korean": "한국어",
      "english": "영어",
      "chunkLabel": "청크 크기",
      "chunkAuto": "자동",
      "submitting": "제출 중…",
      "runBtn": "추론 실행",
      "runBtnCount": "추론 실행 ({{count}})",
      "queued": "대기 중",
      "file_one": "{{count}}개 파일",
      "file_other": "{{count}}개 파일",
      "waiting": "대기 중",
      "noQueued": "대기 중인 파일 없음",
      "inQueue": "대기열에 있음",
      "running": "실행 중",
      "active": "활성",
      "noRunning": "활성 파일 없음",
      "done": "완료",
      "noDone": "아직 완료된 파일 없음",
      "statusDone": "완료",
      "statusFailed": "실패",
      "statusStarting": "시작 중…"
    },
    "timeline": {
      "ariaLabel": "전사 타임라인",
      "seekTo": "{{time}}(으)로 이동"
    },
    "unknownDate": "알 수 없음",
    "filterLabel": "필터: {{filter}}"
  },
  "modal": {
    "ariaLabel": "대화 상자"
  },
  "transcriptDetail": {
    "backAriaLabel": "뒤로",
    "fileSub": "파일 #{{id}} · {{count}}개 큐",
    "downloadSrt": "SRT 다운로드",
    "captionsError": "자막을 불러오지 못했습니다: {{error}}",
    "retry": "재시도",
    "loadingMedia": "미디어 불러오는 중…",
    "mediaError": "미디어를 불러오지 못했습니다",
    "nowPlaying": "현재 재생 중",
    "saveStatus": {
      "editing": "저장 중…",
      "saved": "저장됨",
      "failed": "저장 실패"
    },
    "cueInfo": "큐 {{current}} / {{total}} · {{start}} → {{end}}",
    "cueCountHint": "{{count}}개 큐 · 재생을 눌러 진행하세요",
    "loading": "불러오는 중…",
    "noTranscript": "전사 없음",
    "followAlong": "재생을 눌러 따라가세요",
    "transcriptTitle": "전사",
    "transcriptHint": "클릭하여 이동 · 연필 아이콘으로 편집",
    "loadingCaptions": "자막 불러오는 중…",
    "noCaptions": "사용 가능한 자막 없음",
    "noCaptionsToast": "사용 가능한 자막이 없습니다",
    "nowTag": "▶ 지금",
    "editTitle": "편집",
    "emptyWarn": "⚠ 빈 텍스트로 저장하시겠습니까? 자막이 지워집니다.",
    "keepEditing": "계속 편집",
    "saveEmpty": "빈 상태로 저장",
    "savedToast": "저장됨",
    "saveFailedToast": "저장 실패",
    "downloadedToast": "다운로드됨"
  },
  "transcribePanel": {
    "title": "전사 생성",
    "closeAriaLabel": "패널 닫기",
    "fileLabel": "파일",
    "modelLabel": "모델",
    "modelsLoading": "모델 불러오는 중…",
    "modelsError": "모델을 불러오지 못했습니다.",
    "modelsRetry": "재시도",
    "noModels": "사용 가능한 모델 없음",
    "languageLabel": "언어",
    "korean": "한국어",
    "english": "영어",
    "chunkLabel": "청크",
    "chunk5": "5초",
    "chunk10": "10초",
    "chunk15": "15초",
    "chunk20": "20초",
    "cancel": "취소",
    "starting": "시작 중…",
    "start": "전사 시작"
  },
  "uploadInline": {
    "title": "파일 업로드",
    "limitBadge": "{{active}}/{{max}} 업로드 중",
    "limitReached": "업로드 한도 도달",
    "dropActive": "드롭하여 업로드",
    "dropDefault": "파일 또는 폴더를 여기에 드래그하세요",
    "dropHint": "MP3 · WAV · AAC · MP4 · MOV · 최대 {{max}}개 파일",
    "limitHint": "한 번에 최대 {{max}}개 파일",
    "browseFiles": "파일 찾아보기",
    "browseFolder": "폴더 찾아보기",
    "unsupportedFormat": "지원하지 않는 형식입니다. 허용 형식: MP3, WAV, AAC, FLAC, M4A, OGG, MP4, AVI, MOV, MKV, WEBM.",
    "allSlotsBusy": "{{max}}개 업로드 슬롯이 모두 사용 중입니다. 파일 업로드가 완료되면 다시 시도하세요.",
    "partialQueue": "{{queued}}개 파일 대기 중. {{ignored}}개 파일 무시됨 — 한 번에 {{max}}개까지만 업로드 가능합니다. 각 파일이 완료되면 더 업로드할 수 있습니다.",
    "uploading": "업로드 중…",
    "uploaded": "업로드됨",
    "failed": "실패",
    "retryBtn": "재시도",
    "dismissAriaLabel": "닫기"
  },
  "uploadPanel": {
    "title": "파일 업로드",
    "closeAriaLabel": "업로드 패널 닫기",
    "fileLabel": "파일",
    "dropActive": "드롭하여 첨부",
    "dropDefault": "파일을 드롭하거나 클릭하여 찾아보기",
    "dropHint": "MP3 WAV AAC FLAC M4A OGG · MP4 AVI MOV MKV WEBM",
    "unsupportedFormat": "지원하지 않는 파일 형식입니다",
    "removeAriaLabel": "파일 제거",
    "cancel": "취소",
    "uploading": "업로드 중…",
    "upload": "업로드"
  },
  "history": {
    "pageTitle": "이력 및 워크스페이스",
    "pageSub": "과거 추론 실행을 탐색하고 결과를 검토하세요",
    "loading": "이력 불러오는 중…",
    "loadError": "이력을 불러오지 못했습니다. 다시 시도해 주세요.",
    "retry": "재시도",
    "empty": {
      "title": "파일이 없습니다",
      "hint": "파일을 업로드하고 추론을 실행하면 여기에 결과가 표시됩니다."
    },
    "searchPlaceholder": "파일명 또는 ID로 검색…",
    "filterAll": "전체",
    "filterInferenced": "추론됨",
    "filterPending": "추론 안 됨",
    "sortLabel": "정렬",
    "sortNewest": "최신순",
    "sortOldest": "오래된순",
    "sortName": "이름순",
    "fileCount": "파일 {{count}}개",
    "selectedCount": "{{count}}개 선택됨",
    "selectAll": "전체 선택",
    "clearSelection": "초기화",
    "openWorkspace": "워크스페이스 열기",
    "deleteSelected": "삭제",
    "deleting": "삭제 중…",
    "deleteSuccess": "{{count}}개 파일이 성공적으로 삭제되었습니다.",
    "deleteFail": "삭제에 실패했습니다. 다시 시도해 주세요.",
    "deleteConfirm": {
      "title": "파일 삭제",
      "body": "{{count}}개 파일을 삭제하시겠습니까? 이 작업은 취소할 수 없습니다.",
      "cancel": "취소",
      "confirm": "삭제"
    },
    "workspace": {
      "title": "워크스페이스",
      "noFile": "파일이 선택되지 않음",
      "noFileHint": "왼쪽 목록에서 파일을 선택하면 추론 결과가 여기에 표시됩니다.",
      "tabs": {
        "summary": "요약",
        "keywords": "키워드",
        "assessment": "평가 문항"
      },
      "noSummary": "요약을 사용할 수 없습니다.",
      "noKeywords": "키워드를 사용할 수 없습니다.",
      "noAssessment": "평가 문항을 사용할 수 없습니다.",
      "copy": "복사",
      "copied": "복사됨!",
      "download": "다운로드",
      "edit": "편집",
      "save": "저장",
      "saving": "저장 중…",
      "cancel": "취소",
      "reGenerate": "재생성",
      "reGenerating": "재생성 중…",
      "reGenerateFail": "재생성에 실패했습니다. 다시 시도해 주세요.",
      "reGenerateSuccess": "재생성이 완료되었습니다.",
      "keywordPlaceholder": "키워드를 한 줄에 하나씩…",
      "summaryPlaceholder": "파일에서 자동 입력됨 — 편집하여 덮어쓰기…",
      "editColPreview": "미리보기",
      "editColEdit": "내용 편집",
      "markdownTag": "Markdown",
      "chipView": "칩 뷰",
      "onePerLine": "한 줄에 하나씩",
      "mcq": "MCQ",
      "viewAll": "전체 보기",
      "explanation": "설명",
      "assessComplete": "평가 완료!",
      "assessCompleteDesc": "{{count}}개의 문항을 모두 완료했습니다.",
      "restart": "다시 시작",
      "next": "다음 문항",
      "finish": "완료",
      "question": "Q{{n}}"
    }
  }
}
