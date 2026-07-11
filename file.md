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
  short_answers_prompt: string;
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
  filesPageSize: 100,
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
          ...(shortAnswerPrompt !== undefined && { short_answers_prompt: shortAnswerPrompt }),
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
          ...(shortAnswerPrompt !== undefined && { short_answers_prompt: shortAnswerPrompt }),
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
  onClose: () => void;
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

  const promptMode = n === 1 ? 'Single file — autofilled · edit to override.' : 'Multiple files — enter here to override all.';
  const promptModeColor = n === 1 ? 'var(--blue)' : 'var(--amber)';

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
                  <button
                    className={styles.closeStepBtn}
                    onClick={onClose}
                    title={t('uploadInfer.inferencePanel.closeStep2')}
                  >
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                      <path d="M4 4l8 8M12 4l-8 8" />
                    </svg>
                  </button>
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

                  {/* Short Answer */}
                  <div
                    className={`${styles.cr} ${settings.generateShortAnswer ? styles.ck : ''} ${styles.mb8}`}
                    onClick={() => dispatch(updateSettings({ generateShortAnswer: !settings.generateShortAnswer }))}
                  >
                    <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateShortAnswer')}</label>
                  </div>

                  {/* True/False */}
                  <div
                    className={`${styles.cr} ${settings.generateTrueFalse ? styles.ck : ''} ${styles.mb8}`}
                    onClick={() => dispatch(updateSettings({ generateTrueFalse: !settings.generateTrueFalse }))}
                  >
                    <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateTrueFalse')}</label>
                  </div>

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

              {/* Prompt overrides */}
              <div className={`${styles.card} ${styles.mb12}`}>
                <div className={styles.cardT}>
                  Prompt overrides{' '}
                  {n > 0 && <span className={styles.promptMode} style={{ color: promptModeColor }}>{promptMode}</span>}
                </div>
                <div className={styles.promptStack}>

                  {settings.generateSummary && (
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
                  )}

                  {settings.generateKeywords && (
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
                  )}

                  {settings.generateQuestions && (
                    <div className={`${styles.fg} ${styles.mb0}`}>
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
                  )}

                  {settings.generateShortAnswer && (
                    <div className={`${styles.fg} ${styles.mb0}`}>
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
                  )}

                  {settings.generateTrueFalse && (
                    <div className={`${styles.fg} ${styles.mb0}`}>
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
                  )}

                  {!settings.generateSummary && !settings.generateKeywords && !settings.generateQuestions
                    && !settings.generateShortAnswer && !settings.generateTrueFalse && (
                    <div className={styles.noPromptHint}>{t('uploadInfer.inferencePanel.noContentHint')}</div>
                  )}

                </div>

                {/* Save / Cancel bar */}
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
// pages/UploadInfer/PromptTemplateAssociationModal.tsx
// Content Analytics · Bulk file ↔ prompt-template association
// ═══════════════════════════════════════════════
import React, { useCallback, useEffect, useMemo, useRef, useState } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import { updateFilePrompts, patchFilePromptTemplate } from '../../store/uploadSlice';
import { addToast } from '../../store/toastSlice';
import api from '../../services/api';
import styles from './PromptTemplateAssociationModal.module.scss';

interface PromptTemplateListItem {
    id: number; ms_user_id: number; name: string; description: string;
    summary_prompt: string; keywords_prompt: string; faq_prompt: string;
    short_answers_prompt: string; true_false_prompt: string;
    inserted_at: string; updated_at: string; is_default?: boolean;
}
interface PromptTemplateDetail {
    id: number; ms_user_id: number; name: string; description: string;
    summary_prompt: string; keyword_prompt?: string; keywords_prompt?: string;
    faq_prompt: string; short_answers_prompt?: string; true_false_prompt?: string;
    inserted_at: string; updated_at: string; is_default?: boolean;
}
interface Props { open: boolean; onClose: () => void; }

const NO_TEMPLATE = -1;

const PromptTemplateAssociationModal: React.FC<Props> = ({ open, onClose }) => {
    const dispatch = useAppDispatch();
    const { t } = useTranslation();
    const { serverFiles, dateFrom, dateTo } = useAppSelector(s => s.upload);

    const [templates, setTemplates] = useState<PromptTemplateListItem[]>([]);
    const [templatesLoading, setTemplatesLoading] = useState(false);
    const [templatesError, setTemplatesError] = useState<string | null>(null);
    const [selection, setSelection] = useState<Record<number, number>>({});
    const initialSelectionRef = useRef<Record<number, number>>({});
    const [bulkValue, setBulkValue] = useState<number>(NO_TEMPLATE);
    const [saving, setSaving] = useState(false);
    const [detailTemplateId, setDetailTemplateId] = useState<number | null>(null);
    const [detailData, setDetailData] = useState<PromptTemplateDetail | null>(null);
    const [detailLoading, setDetailLoading] = useState(false);
    const [detailError, setDetailError] = useState<string | null>(null);

    // ── Associate ALL — every file in the current date range, bypasses the
    //     per-row preview entirely and hits the backend directly. ──
    const [isAssociatingAll, setIsAssociatingAll] = useState(false);
    const [associateAllError, setAssociateAllError] = useState<string | null>(null);

    useEffect(() => {
        if (!open) return;
        const init: Record<number, number> = {};
        serverFiles.forEach(f => { init[f.id] = f.prompt_template_id ?? NO_TEMPLATE; });
        setSelection(init);
        initialSelectionRef.current = init;
        setBulkValue(NO_TEMPLATE);
        setDetailTemplateId(null); setDetailData(null); setDetailError(null);
        setAssociateAllError(null);

        let cancelled = false;
        (async () => {
            setTemplatesLoading(true); setTemplatesError(null);
            try {
                const res = await api.get('/prompt_template/list_template');
                const list: PromptTemplateListItem[] = (res.data as any)?.templates ?? [];
                if (!cancelled) setTemplates(Array.isArray(list) ? list : []);
            } catch {
                if (!cancelled) setTemplatesError(t('uploadInfer.templateModal.loadFail'));
            } finally { if (!cancelled) setTemplatesLoading(false); }
        })();
        return () => { cancelled = true; };
        // eslint-disable-next-line react-hooks/exhaustive-deps
    }, [open]);

    useEffect(() => {
        if (!open) return;
        const onKey = (e: KeyboardEvent) => { if (e.key === 'Escape') { e.preventDefault(); e.stopPropagation(); } };
        window.addEventListener('keydown', onKey, true);
        return () => window.removeEventListener('keydown', onKey, true);
    }, [open]);

    const dirtyIds = useMemo(() => {
        const init = initialSelectionRef.current;
        return Object.keys(selection).map(Number).filter(id => selection[id] !== init[id]);
    }, [selection]);
    const isDirty = dirtyIds.length > 0;

    const handleRowChange = useCallback((fileId: number, templateId: number) => {
        setSelection(prev => ({ ...prev, [fileId]: templateId }));
    }, []);

    const handleApplyToAll = useCallback(() => {
        setSelection(prev => {
            const next = { ...prev };
            Object.keys(next).forEach(k => { next[Number(k)] = bulkValue; });
            return next;
        });
    }, [bulkValue]);

    const handleViewDetails = useCallback(async (templateId: number) => {
        if (detailTemplateId === templateId) { setDetailTemplateId(null); setDetailData(null); setDetailError(null); return; }
        setDetailTemplateId(templateId); setDetailData(null); setDetailError(null); setDetailLoading(true);
        try {
            const res = await api.get(`/prompt_template/${templateId}`);
            const d: PromptTemplateDetail | undefined = (res.data as any);
            if (d && d.id !== undefined) setDetailData(d);
            else setDetailError(t('uploadInfer.templateModal.noDetailsReturned'));
        } catch { setDetailError(t('uploadInfer.templateModal.detailLoadFail')); }
        finally { setDetailLoading(false); }
    }, [detailTemplateId, t]);

    const handleSave = useCallback(async () => {
        if (!isDirty || saving) return;
        setSaving(true);
        const associateGroups: Record<number, number[]> = {};
        for (const id of dirtyIds) {
            const v = selection[id];
            if (v === NO_TEMPLATE) continue;
            if (!associateGroups[v]) associateGroups[v] = [];
            associateGroups[v].push(id);
        }
        const entries = Object.entries(associateGroups);
        if (entries.length === 0) { setSaving(false); onClose(); return; }
        try {
            await Promise.all(entries.map(([templateIdStr, fileIds]) =>
                api.post('/prompt_template/associate', { file_ids: fileIds, template_id: Number(templateIdStr), all: false })
            ));
            const templatePatch: Record<number, number | null> = {};
            entries.forEach(([templateIdStr, fileIds]) => {
                const tmpl = templates.find(t => t.id === Number(templateIdStr));
                fileIds.forEach(id => { templatePatch[id] = Number(templateIdStr); });
                if (!tmpl) return;
                dispatch(updateFilePrompts({
                    fileIds,
                    summaryPrompt: tmpl.summary_prompt,
                    keywordsPrompt: tmpl.keywords_prompt,
                    faqPrompt: tmpl.faq_prompt,
                    shortAnswerPrompt: tmpl.short_answers_prompt,
                    trueFalsePrompt: tmpl.true_false_prompt,
                }));
            });
            dispatch(patchFilePromptTemplate(templatePatch));
            dispatch(addToast(t('uploadInfer.templateModal.saveSuccess'), 'success'));
            onClose();
        } catch {
            dispatch(addToast(t('uploadInfer.templateModal.saveFail'), 'error'));
        } finally { setSaving(false); }
    }, [isDirty, saving, dirtyIds, selection, templates, dispatch, onClose, t]);

    // ── Associate ALL files in [dateFrom, dateTo] with bulkValue — skips the
    //     per-row preview/save flow and applies directly on the backend. ──
    const handleAssociateAll = useCallback(async () => {
        if (bulkValue === NO_TEMPLATE || saving || isAssociatingAll) return;
        setIsAssociatingAll(true);
        setAssociateAllError(null);
        try {
            await api.post('/prompt_template/associate', {
                template_id: bulkValue,
                all: true,
                start_date: dateFrom,
                end_date: dateTo,
            });
            const tmpl = templates.find(x => x.id === bulkValue);
            // Best-effort local patch for whatever's currently loaded — files on
            // other pages will reflect the change next time they're fetched.
            const templatePatch: Record<number, number | null> = {};
            const fileIds = serverFiles.map(f => f.id);
            fileIds.forEach(id => { templatePatch[id] = bulkValue; });
            dispatch(patchFilePromptTemplate(templatePatch));
            if (tmpl) {
                dispatch(updateFilePrompts({
                    fileIds,
                    summaryPrompt: tmpl.summary_prompt,
                    keywordsPrompt: tmpl.keywords_prompt,
                    faqPrompt: tmpl.faq_prompt,
                    shortAnswerPrompt: tmpl.short_answers_prompt,
                    trueFalsePrompt: tmpl.true_false_prompt,
                }));
            }
            dispatch(addToast(t('uploadInfer.templateModal.associateAllSuccess'), 'success'));
            onClose();
        } catch {
            setAssociateAllError(t('uploadInfer.templateModal.associateAllFail'));
        } finally {
            setIsAssociatingAll(false);
        }
    }, [bulkValue, saving, isAssociatingAll, dateFrom, dateTo, serverFiles, templates, dispatch, onClose, t]);

    const handleCloseClick = useCallback(() => { if (saving) return; onClose(); }, [saving, onClose]);
    const handleBackdropClick = useCallback((e: React.MouseEvent) => { e.stopPropagation(); }, []);
    const handleDialogClick = useCallback((e: React.MouseEvent) => { e.stopPropagation(); }, []);

    if (!open) return null;

    const templateName = (id: number) => {
        if (id === NO_TEMPLATE) return t('uploadInfer.templateModal.noneRow');
        const tmpl = templates.find(x => x.id === id);
        return tmpl ? tmpl.name : `Template #${id}`;
    };

    return (
        <div className={styles.backdrop} onMouseDown={handleBackdropClick} role="presentation">
            <div className={styles.dialog} onMouseDown={handleDialogClick} role="dialog" aria-modal="true" aria-labelledby="template-modal-title">

                {/* Header */}
                <div className={styles.header}>
                    <div className={styles.headerText}>
                        <div className={styles.title} id="template-modal-title">{t('uploadInfer.templateModal.title')}</div>
                        <div className={styles.subtitle}>{t('uploadInfer.templateModal.subtitle')}</div>
                    </div>
                    <button className={styles.closeBtn} onClick={handleCloseClick} disabled={saving}
                        title={saving ? t('uploadInfer.templateModal.closeSaving') : t('uploadInfer.templateModal.close')}
                        aria-label={t('uploadInfer.templateModal.close')}>
                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                            <path d="M4 4l8 8M12 4l-8 8" />
                        </svg>
                    </button>
                </div>

                {/* Bulk bar — applies to files currently loaded in this modal (preview, needs Save) */}
                <div className={styles.bulkBar}>
                    <span className={styles.bulkLabel}>{t('uploadInfer.templateModal.applyToAll')}</span>
                    <select className={styles.bulkSelect} value={bulkValue}
                        onChange={e => setBulkValue(Number(e.target.value))}
                        disabled={saving || templatesLoading || serverFiles.length === 0}>
                        <option value={NO_TEMPLATE}>{t('uploadInfer.templateModal.noneOption')}</option>
                        {templates.map(tmpl => <option key={tmpl.id} value={tmpl.id}>{tmpl.name}</option>)}
                    </select>
                    <button className={styles.bulkApplyBtn} onClick={handleApplyToAll} disabled={saving || serverFiles.length === 0}>
                        {t('uploadInfer.templateModal.applyBtn')}
                    </button>
                    <span className={styles.bulkHint}>
                        {isDirty
                            ? t('uploadInfer.templateModal.changes', { count: dirtyIds.length })
                            : t('uploadInfer.templateModal.noChanges')}
                    </span>
                </div>

                {/* Associate ALL bar — applies immediately to every file in the date
                    range on the backend, including files not currently loaded here */}
                <div className={styles.associateAllBar}>
                    <span className={styles.bulkLabel}>
                        {t('uploadInfer.templateModal.associateAllLabel', { from: dateFrom, to: dateTo })}
                    </span>
                    <button
                        className={styles.associateAllBtn}
                        onClick={handleAssociateAll}
                        disabled={bulkValue === NO_TEMPLATE || saving || isAssociatingAll}
                    >
                        {isAssociatingAll ? <span className={styles.spinnerSm} /> : t('uploadInfer.templateModal.associateAllBtn')}
                    </button>
                    {associateAllError && <span className={styles.associateAllError}>{associateAllError}</span>}
                </div>

                {/* Body */}
                <div className={styles.body}>
                    <div className={`${styles.listPane} ${detailTemplateId !== null ? styles.listPaneNarrow : ''}`}>
                        <div className={styles.listHead}>
                            <div className={styles.colFile}>{t('uploadInfer.templateModal.colFile')}</div>
                            <div className={styles.colTemplate}>{t('uploadInfer.templateModal.colTemplate')}</div>
                            <div className={styles.colActions} />
                        </div>

                        {templatesLoading && (
                            <div className={styles.listState}>
                                <div className={styles.spinner} /><span>{t('uploadInfer.templateModal.loadingTemplates')}</span>
                            </div>
                        )}
                        {templatesError && <div className={`${styles.listState} ${styles.errorState}`}>{templatesError}</div>}
                        {!templatesLoading && serverFiles.length === 0 && (
                            <div className={styles.listState}>{t('uploadInfer.templateModal.noFiles')}</div>
                        )}

                        {!templatesLoading && !templatesError && serverFiles.map(f => {
                            const current = selection[f.id] ?? NO_TEMPLATE;
                            const initial = initialSelectionRef.current[f.id] ?? NO_TEMPLATE;
                            const rowDirty = current !== initial;
                            const hasTemplate = current !== NO_TEMPLATE;
                            return (
                                <div key={f.id} className={`${styles.row} ${rowDirty ? styles.rowDirty : ''}`}>
                                    <div className={styles.colFile} title={f.original_name}>
                                        <span className={styles.fileName}>{f.original_name}</span>
                                        {initial !== NO_TEMPLATE && (
                                            <span className={styles.currentBadge} title={t('uploadInfer.templateModal.mapped')}>
                                                {t('uploadInfer.templateModal.mapped')}
                                            </span>
                                        )}
                                    </div>
                                    <div className={styles.colTemplate}>
                                        <select className={styles.rowSelect} value={current}
                                            onChange={e => handleRowChange(f.id, Number(e.target.value))} disabled={saving}>
                                            <option value={NO_TEMPLATE}>{t('uploadInfer.templateModal.noneRow')}</option>
                                            {templates.map(tmpl => <option key={tmpl.id} value={tmpl.id}>{tmpl.name}</option>)}
                                        </select>
                                    </div>
                                    <div className={styles.colActions}>
                                        {hasTemplate && (
                                            <button className={styles.viewBtn} onClick={() => handleViewDetails(current)}
                                                disabled={saving} title={t('uploadInfer.templateModal.viewDetails')}>
                                                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                                    <path d="M1.5 8s2.5-4.5 6.5-4.5S14.5 8 14.5 8s-2.5 4.5-6.5 4.5S1.5 8 1.5 8z" />
                                                    <circle cx="8" cy="8" r="2" />
                                                </svg>
                                            </button>
                                        )}
                                    </div>
                                </div>
                            );
                        })}
                    </div>

                    {/* Detail drawer */}
                    {detailTemplateId !== null && (
                        <div className={styles.detailPane}>
                            <div className={styles.detailHead}>
                                <div className={styles.detailTitle}>{detailData?.name ?? templateName(detailTemplateId)}</div>
                                <button className={styles.detailClose}
                                    onClick={() => { setDetailTemplateId(null); setDetailData(null); }}
                                    title={t('uploadInfer.templateModal.detailClose')}
                                    aria-label={t('uploadInfer.templateModal.detailClose')}>
                                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                                        <path d="M4 4l8 8M12 4l-8 8" />
                                    </svg>
                                </button>
                            </div>
                            {detailLoading && (
                                <div className={styles.listState}>
                                    <div className={styles.spinner} /><span>{t('uploadInfer.templateModal.loadingDetails')}</span>
                                </div>
                            )}
                            {detailError && <div className={`${styles.listState} ${styles.errorState}`}>{detailError}</div>}
                            {detailData && (
                                <>
                                    <div className={styles.detailMeta}>
                                        <div><span className={styles.metaLabel}>{t('uploadInfer.templateModal.metaId')}</span> {detailData.id}</div>
                                        <div><span className={styles.metaLabel}>{t('uploadInfer.templateModal.metaUpdated')}</span> {detailData.updated_at}</div>
                                        {detailData.description && (
                                            <div><span className={styles.metaLabel}>{t('uploadInfer.templateModal.metaDescription')}</span> {detailData.description}</div>
                                        )}
                                    </div>
                                    <div className={styles.promptBlocks}>
                                        {[
                                            { label: t('uploadInfer.templateModal.summaryPrompt'), text: detailData.summary_prompt },
                                            { label: t('uploadInfer.templateModal.keywordPrompt'), text: detailData.keywords_prompt ?? detailData.keyword_prompt ?? '—' },
                                            { label: t('uploadInfer.templateModal.faqPrompt'), text: detailData.faq_prompt },
                                            { label: t('uploadInfer.templateModal.shortAnswerPrompt'), text: detailData.short_answers_prompt ?? '—' },
                                            { label: t('uploadInfer.templateModal.trueFalsePrompt'), text: detailData.true_false_prompt ?? '—' },
                                        ].map(block => (
                                            <div key={block.label} className={styles.promptBlock}>
                                                <div className={styles.promptLabel}>{block.label}</div>
                                                <div className={styles.promptText}>{block.text || '—'}</div>
                                            </div>
                                        ))}
                                    </div>
                                </>
                            )}
                        </div>
                    )}
                </div>

                {/* Footer */}
                <div className={styles.footer}>
                    {saving && (
                        <span className={styles.savingHint}>
                            <span className={styles.spinnerSm} />{t('uploadInfer.templateModal.savingHint')}
                        </span>
                    )}
                    <button className={`${styles.btn} ${styles.btnGhost}`} onClick={handleCloseClick} disabled={saving}>
                        {t('uploadInfer.templateModal.cancel')}
                    </button>
                    <button className={`${styles.btn} ${styles.btnPrimary}`} onClick={handleSave} disabled={!isDirty || saving}>
                        {saving ? t('uploadInfer.templateModal.saving') : t('uploadInfer.templateModal.save')}
                    </button>
                </div>
            </div>
        </div>
    );
};

export default PromptTemplateAssociationModal;
