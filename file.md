//Ragslice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { ragApi } from '../../api/endpoints/rag';
import type { CreateDatasetParams, RagFile, RagStatusResponse } from '../../api/endpoints/rag';

// ---------------------------------------------------------------------------
// File rows live OUTSIDE Redux.
//
// A dataset can have thousands of files (8k+). Keeping them in the store means
// every poll pushes them through immer (deep freeze), RTK's dev-only
// immutability/serializability middleware (walks the whole state on every
// action) and a full re-render — enough to freeze the main thread and stall
// scrolling. So the rows are kept in this plain module-level cache; Redux only
// holds the small status summary plus `filesVersion`, a counter that bumps
// whenever the cached rows actually changed so components know to re-read.
// ---------------------------------------------------------------------------
const statusFilesCache = new Map<string, RagFile[]>();
const uploadedFilesCache = new Map<string, RagFile[]>();
const EMPTY_FILES: RagFile[] = [];

export const getCachedStatusFiles = (datasetId: string): RagFile[] =>
  statusFilesCache.get(datasetId) ?? EMPTY_FILES;
export const getCachedUploadedFiles = (datasetId: string): RagFile[] =>
  uploadedFilesCache.get(datasetId) ?? EMPTY_FILES;

// Reuse the previous object for every file whose visible fields are
// unchanged, and the previous ARRAY when nothing changed at all, so memoized
// rows (React.memo) skip re-rendering. Returns whether anything changed.
function storeFiles(cache: Map<string, RagFile[]>, datasetId: string, next: RagFile[]): boolean {
  const prev = cache.get(datasetId);
  if (!prev || prev.length === 0) {
    cache.set(datasetId, next);
    return next.length > 0 || !!prev;
  }
  const byId = new Map<number, RagFile>();
  for (const f of prev) byId.set(f.id, f);

  let identical = prev.length === next.length;
  const merged = next.map((f, i) => {
    const p = byId.get(f.id);
    const unchanged =
      !!p &&
      p.status === f.status &&
      p.error_message === f.error_message &&
      p.updated_at === f.updated_at &&
      p.filename === f.filename &&
      p.size === f.size;
    if (!unchanged || prev[i] !== p) identical = false;
    return unchanged ? p! : f;
  });
  if (identical) return false;
  cache.set(datasetId, merged);
  return true;
}

type AsyncStatus = 'idle' | 'loading' | 'succeeded' | 'failed';

interface RagState {
  createStatus: AsyncStatus;
  createError: string | null;

  // Bumps when the cached file rows change (see getCachedStatusFiles /
  // getCachedUploadedFiles above) — the rows themselves are not in the store.
  filesVersion: number;
  filesStatus: AsyncStatus;
  filesError: string | null;

  uploadStatus: AsyncStatus;
  uploadError: string | null;

  processingStartStatus: AsyncStatus;
  processingStartError: string | null;

  status: RagStatusResponse | null;
  statusFetchStatus: AsyncStatus;
  statusFetchError: string | null;
}

const initialState: RagState = {
  createStatus: 'idle',
  createError: null,

  filesVersion: 0,
  filesStatus: 'idle',
  filesError: null,

  uploadStatus: 'idle',
  uploadError: null,

  processingStartStatus: 'idle',
  processingStartError: null,

  status: null,
  statusFetchStatus: 'idle',
  statusFetchError: null,
};

// POST /datasets/create (eval_type: rag) — returns the new dataset id.
export const createRagDataset = createAsyncThunk(
  'rag/create',
  async (params: CreateDatasetParams, { rejectWithValue }) => {
    try {
      return await ragApi.create(params);
    } catch (err) {
      const message = err instanceof Error ? err.message : 'Failed to create dataset';
      return rejectWithValue(message);
    }
  }
);

// GET /datasets/uploaded_files, filtered to this dataset.
export const fetchRagUploadedFiles = createAsyncThunk(
  'rag/fetchUploadedFiles',
  async (datasetId: string, { rejectWithValue }) => {
    try {
      const files = await ragApi.listUploadedFiles(datasetId);
      return { changed: storeFiles(uploadedFilesCache, datasetId, Array.isArray(files) ? files : []) };
    } catch (err) {
      const message = err instanceof Error ? err.message : 'Failed to load uploaded files';
      return rejectWithValue(message);
    }
  }
);

// POST /datasets/upload-rag?dataset_id= — one batch of files.
export const uploadRagFiles = createAsyncThunk(
  'rag/uploadFiles',
  async ({ datasetId, files }: { datasetId: string; files: File[] }, { rejectWithValue }) => {
    try {
      await ragApi.uploadFiles(datasetId, files);
    } catch (err) {
      const message = err instanceof Error ? err.message : 'Failed to upload files';
      return rejectWithValue(message);
    }
  }
);

// GET /datasets/rag_processing?dataset_id=
export const startRagProcessing = createAsyncThunk(
  'rag/startProcessing',
  async (datasetId: string, { rejectWithValue }) => {
    try {
      await ragApi.startProcessing(datasetId);
    } catch (err) {
      const message = err instanceof Error ? err.message : 'Failed to start processing';
      return rejectWithValue(message);
    }
  }
);

// GET /datasets/status?dataset_id= — polled while the studio is open.
export const fetchRagStatus = createAsyncThunk(
  'rag/fetchStatus',
  async (datasetId: string, { rejectWithValue }) => {
    try {
      const full = await ragApi.fetchStatus(datasetId);
      const changed = storeFiles(statusFilesCache, datasetId, Array.isArray(full?.files) ? full.files : []);
      // Strip the rows from what goes into the store (they're in the cache).
      return { status: { ...full, files: [] as RagFile[] }, filesChanged: changed };
    } catch (err) {
      const message = err instanceof Error ? err.message : 'Failed to load processing status';
      return rejectWithValue(message);
    }
  }
);

const ragSlice = createSlice({
  name: 'rag',
  initialState,
  reducers: {
    resetRagCreate: (state) => {
      state.createStatus = 'idle';
      state.createError = null;
    },
    // Called when the studio closes — wipes everything scoped to the
    // dataset that was open so a later session starts clean.
    resetRagStudio: (state) => {
      statusFilesCache.clear();
      uploadedFilesCache.clear();
      state.filesVersion += 1;
      state.filesStatus = 'idle';
      state.filesError = null;
      state.uploadStatus = 'idle';
      state.uploadError = null;
      state.processingStartStatus = 'idle';
      state.processingStartError = null;
      state.status = null;
      state.statusFetchStatus = 'idle';
      state.statusFetchError = null;
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(createRagDataset.pending, (state) => {
        state.createStatus = 'loading';
        state.createError = null;
      })
      .addCase(createRagDataset.fulfilled, (state) => {
        state.createStatus = 'succeeded';
      })
      .addCase(createRagDataset.rejected, (state, action) => {
        state.createStatus = 'failed';
        state.createError = (action.payload as string) || action.error.message || 'Failed to create dataset';
      })

      .addCase(fetchRagUploadedFiles.pending, (state) => {
        state.filesStatus = 'loading';
        state.filesError = null;
      })
      .addCase(fetchRagUploadedFiles.fulfilled, (state, action) => {
        state.filesStatus = 'succeeded';
        if (action.payload?.changed) state.filesVersion += 1;
      })
      .addCase(fetchRagUploadedFiles.rejected, (state, action) => {
        state.filesStatus = 'failed';
        state.filesError = (action.payload as string) || action.error.message || 'Failed to load uploaded files';
      })

      .addCase(uploadRagFiles.pending, (state) => {
        state.uploadStatus = 'loading';
        state.uploadError = null;
      })
      .addCase(uploadRagFiles.fulfilled, (state) => {
        state.uploadStatus = 'succeeded';
      })
      .addCase(uploadRagFiles.rejected, (state, action) => {
        state.uploadStatus = 'failed';
        state.uploadError = (action.payload as string) || action.error.message || 'Failed to upload files';
      })

      .addCase(startRagProcessing.pending, (state) => {
        state.processingStartStatus = 'loading';
        state.processingStartError = null;
      })
      .addCase(startRagProcessing.fulfilled, (state) => {
        state.processingStartStatus = 'succeeded';
      })
      .addCase(startRagProcessing.rejected, (state, action) => {
        state.processingStartStatus = 'failed';
        state.processingStartError = (action.payload as string) || action.error.message || 'Failed to start processing';
      })

      .addCase(fetchRagStatus.pending, (state) => {
        // Deliberately doesn't flip statusFetchStatus to 'loading' on every
        // poll tick — only the very first fetch should show a loading
        // state; background polls shouldn't disrupt what's on screen.
        if (state.statusFetchStatus === 'idle') state.statusFetchStatus = 'loading';
      })
      .addCase(fetchRagStatus.fulfilled, (state, action) => {
        state.statusFetchStatus = 'succeeded';
        state.statusFetchError = null;
        state.status = action.payload?.status ?? null;
        if (action.payload?.filesChanged) state.filesVersion += 1;
      })
      .addCase(fetchRagStatus.rejected, (state, action) => {
        // Same reasoning — don't blow away a working status view just
        // because one background poll failed transiently.
        if (!state.status) {
          state.statusFetchStatus = 'failed';
          state.statusFetchError = (action.payload as string) || action.error.message || 'Failed to load status';
        }
      });
  },
});

export const { resetRagCreate, resetRagStudio } = ragSlice.actions;
export default ragSlice.reducer;






















//Ragpanel.tsx
//RagPanel.tsx
//
// Embedded RAG upload + processing workspace. Rendered as the second column
// of the dataset detail view (see DetailView in Datasets.tsx) for RAG
// datasets — replaces the old RagStudio modal.
//
// Behaviour is driven by `current_state` from GET /datasets/status:
//   upload     — user can add files and start processing
//   processing — uploads locked, progress only
//   completed  — read-only
//
// /datasets/status is polled (every 10s) only while the dataset is
// processing; upload and completed states are fetched on demand only.
import { memo, useCallback, useEffect, useMemo, useRef, useState } from 'react';
import {
  X, UploadCloud, FilePlus2, Loader2, PlayCircle, CheckCircle2, XCircle,
  Clock, AlertCircle, Database, RotateCw, File as FileIcon,
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import {
  fetchRagUploadedFiles, uploadRagFiles, startRagProcessing, fetchRagStatus, resetRagStudio,
  getCachedStatusFiles, getCachedUploadedFiles,
} from '../../store/slices/ragSlice';
import { RAG_UPLOAD_EXTENSIONS, getRagPhase } from '../../api/endpoints/rag';
import type { RagFile, RagPhase } from '../../api/endpoints/rag';
import VirtualList from './VirtualList';
import styles from './RagPanel.module.scss';

const ACCEPT_ATTR = RAG_UPLOAD_EXTENSIONS.map((e) => `.${e}`).join(',');
// Fixed row geometry for the virtualized lists (px).
const FILE_ROW_H = 54;      // 48 row + 6 gap
const FILE_ROW_GAP = 6;
const FILE_LIST_MAX_H = 360;
const STAGED_ROW_H = 38;    // 32 row + 6 gap
const STAGED_ROW_GAP = 6;
const STAGED_LIST_MAX_H = 200;

const STATUS_POLL_MS = 10000; // only polled while the dataset is in the processing state

interface StagedFile {
  key: string;
  file: File;
}

interface RagPanelProps {
  datasetId: string;
  // Fired when the dataset moves between phases after the first load
  // (e.g. processing -> completed) so the parent can refresh its list.
  onPhaseChange?: (phase: RagPhase) => void;
}

function formatBytes(bytes: number): string {
  if (!bytes || bytes <= 0) return '0 B';
  const units = ['B', 'KB', 'MB', 'GB', 'TB'];
  const i = Math.min(units.length - 1, Math.floor(Math.log(bytes) / Math.log(1024)));
  const val = bytes / Math.pow(1024, i);
  return `${val.toFixed(i === 0 ? 0 : 1)} ${units[i]}`;
}

function extOf(filename: string): string {
  const idx = filename.lastIndexOf('.');
  return idx >= 0 ? filename.slice(idx + 1).toLowerCase() : '';
}

type StatusTier = 'ready' | 'processing' | 'queued' | 'failed' | 'uploaded' | 'other';

function statusTier(raw?: string | null): StatusTier {
  const v = (raw || '').trim().toLowerCase();
  if (['ready', 'done', 'completed', 'success', 'succeeded'].includes(v)) return 'ready';
  if (['processing', 'in_progress', 'running'].includes(v)) return 'processing';
  if (['queued', 'pending'].includes(v)) return 'queued';
  if (['failed', 'error'].includes(v)) return 'failed';
  if (v === 'uploaded') return 'uploaded';
  return 'other';
}

function capitalize(s: string): string {
  return s ? s.charAt(0).toUpperCase() + s.slice(1) : s;
}

// Header marker for the dataset's phase, readable at a glance:
//   upload     — dashed "call to action" tag
//   processing — solid, shimmering highlighter with a spinner
//   completed  — rotated, double-bordered "stamp"
function PhaseMarker({ phase }: { phase: RagPhase }) {
  const config = {
    upload: { label: 'Upload', icon: <UploadCloud size={13} />, hint: 'Ready for files' },
    processing: { label: 'Processing', icon: <Loader2 size={13} className={styles['rag-panel__spin']} />, hint: 'Processing in progress' },
    completed: { label: 'Completed', icon: <CheckCircle2 size={14} />, hint: 'Processing completed' },
  }[phase];

  return (
    // key={phase} remounts the marker on a phase change so the entry
    // animation (the "stamp down" in particular) plays again.
    <span
      key={phase}
      role="status"
      aria-label={config.hint}
      title={config.hint}
      className={`${styles['rag-panel__phase']} ${styles[`rag-panel__phase--${phase}`]}`}
    >
      {config.icon}
      <span className={styles['rag-panel__phase-text']}>{config.label}</span>
    </span>
  );
}

// Memoized so a 10s status poll that returns identical rows doesn't
// re-render every visible row.
const FileRow = memo(function FileRow({ f }: { f: RagFile }) {
  const tier = statusTier(f.status);
  return (
    <div className={styles['rag-panel__file-row']}>
      <span className={styles['rag-panel__file-icon']}><FileIcon size={14} /></span>
      <div className={styles['rag-panel__file-info']}>
        <span className={styles['rag-panel__file-name']} title={f.filename}>{f.filename}</span>
        <span className={styles['rag-panel__file-meta']}>
          {(f.ext || '').toUpperCase()} · {formatBytes(f.size)}
        </span>
      </div>
      {f.status && (
        <span
          className={`${styles['rag-panel__file-status']} ${styles[`rag-panel__file-status--${tier}`]}`}
          title={f.error_message || undefined}
        >
          {tier === 'ready' && <CheckCircle2 size={12} />}
          {tier === 'processing' && <Loader2 size={12} className={styles['rag-panel__spin']} />}
          {tier === 'queued' && <Clock size={12} />}
          {tier === 'failed' && <XCircle size={12} />}
          {capitalize(f.status)}
        </span>
      )}
      {f.status && tier === 'failed' && f.error_message && (
        <span className={styles['rag-panel__file-error-icon']} title={f.error_message}>
          <AlertCircle size={13} />
        </span>
      )}
    </div>
  );
});

export default function RagPanel({ datasetId, onPhaseChange }: RagPanelProps) {
  const dispatch = useAppDispatch();
  const {
    filesVersion = 0,
    filesStatus = 'idle',
    filesError = null,
    uploadStatus = 'idle',
    uploadError = null,
    processingStartStatus = 'idle',
    processingStartError = null,
    status: rawStatus = null,
    statusFetchStatus = 'idle',
    statusFetchError = null,
  } = useAppSelector((s) => s.rag) ?? {};

  // The rag slice holds a single "current dataset" worth of state. Since the
  // user can click between datasets quickly, a response for the previously
  // selected dataset may land after this panel has mounted for a new one —
  // ignore anything that doesn't belong to the dataset in view.
  const status = rawStatus && rawStatus.dataset_id === datasetId ? rawStatus : null;
  // File rows are kept outside Redux (see ragSlice) — re-read them only when
  // filesVersion says they changed.
  const uploadedFiles = useMemo(
    () => getCachedUploadedFiles(datasetId),
    // eslint-disable-next-line react-hooks/exhaustive-deps
    [filesVersion, datasetId]
  );
  const statusFiles = useMemo(
    () => getCachedStatusFiles(datasetId),
    // eslint-disable-next-line react-hooks/exhaustive-deps
    [filesVersion, datasetId]
  );

  const [stagedFiles, setStagedFiles] = useState<StagedFile[]>([]);
  const [rejectedNote, setRejectedNote] = useState<string | null>(null);
  const [dragOver, setDragOver] = useState(false);
  const fileInputRef = useRef<HTMLInputElement>(null);

  // Initial load — always hit the API fresh so this reflects reality even if
  // the app was reloaded mid-processing. The cleanup wipes the slice when the
  // user switches dataset or leaves a RAG dataset, so nothing leaks across.
  useEffect(() => {
    dispatch(fetchRagUploadedFiles(datasetId));
    dispatch(fetchRagStatus(datasetId));
    return () => {
      dispatch(resetRagStudio());
    };
  }, [dispatch, datasetId]);

  // Phase driven by current_state from /datasets/status.
  const phase = getRagPhase(status?.current_state);
  const canUpload = phase === 'upload';
  const isProcessing = phase === 'processing';
  const isCompleted = phase === 'completed';

  // Set right after "Process Files" succeeds and cleared as soon as the
  // backend reports a phase other than "upload". Covers the short gap where
  // processing has been started but /datasets/status hasn't flipped to
  // "processing" yet — without it, polling wouldn't begin and the UI could
  // sit on a stale "upload" view.
  const [awaitingProcessing, setAwaitingProcessing] = useState(false);
  useEffect(() => {
    if (phase !== 'upload') setAwaitingProcessing(false);
  }, [phase]);
  useEffect(() => {
    setAwaitingProcessing(false);
  }, [datasetId]);

  // Status polling — every 10s, and only while processing. In the upload and
  // completed states the status is fetched on open, after uploads / starting
  // processing, and via Refresh, but never on a timer.
  const shouldPoll = isProcessing || awaitingProcessing;

  // A poll with thousands of files means a large JSON parse. Don't start one
  // while the user is mid-scroll — wait until they pause, so scrolling never
  // competes with it.
  const lastScrollAtRef = useRef(0);
  const markScrolling = useCallback(() => { lastScrollAtRef.current = Date.now(); }, []);

  useEffect(() => {
    if (!shouldPoll) return;
    let retry: ReturnType<typeof setTimeout> | null = null;
    const tick = () => {
      if (retry) { clearTimeout(retry); retry = null; }
      if (Date.now() - lastScrollAtRef.current < 600) {
        retry = setTimeout(tick, 600);
        return;
      }
      dispatch(fetchRagStatus(datasetId));
    };
    const interval = setInterval(tick, STATUS_POLL_MS);
    return () => {
      clearInterval(interval);
      if (retry) clearTimeout(retry);
    };
  }, [dispatch, datasetId, shouldPoll]);

  // Tell the parent when the phase changes after the first load.
  const prevPhaseRef = useRef<RagPhase | null>(null);
  useEffect(() => {
    if (!status) return;
    if (prevPhaseRef.current && prevPhaseRef.current !== phase) onPhaseChange?.(phase);
    prevPhaseRef.current = phase;
  }, [status, phase, onPhaseChange]);

  const initialLoading = !status && statusFetchStatus !== 'failed';
  const statusLoadFailed = !status && statusFetchStatus === 'failed';

  const totalFromStatus = status?.total ?? null;
  const knownFileCount = totalFromStatus ?? uploadedFiles.length;
  const hasAnyFiles = knownFileCount > 0;
  const canProcess = canUpload && hasAnyFiles && processingStartStatus !== 'loading';

  // Prefer the richer per-file status list from /datasets/status once it's
  // loaded (it carries status + error_message); fall back to the plain
  // uploaded-files list before that.
  const displayFiles: RagFile[] = statusFiles.length ? statusFiles : uploadedFiles;

  const refreshAll = () => {
    dispatch(fetchRagStatus(datasetId));
    dispatch(fetchRagUploadedFiles(datasetId));
  };

  const addFiles = useCallback((incoming: FileList | File[]) => {
    const list = Array.from(incoming);
    const accepted: StagedFile[] = [];
    const rejected: string[] = [];

    for (const f of list) {
      const ext = extOf(f.name);
      if (!RAG_UPLOAD_EXTENSIONS.includes(ext)) {
        rejected.push(f.name);
        continue;
      }
      accepted.push({ key: `${f.name}-${f.size}-${f.lastModified}`, file: f });
    }

    setStagedFiles((prev) => {
      const existingKeys = new Set(prev.map((s) => s.key));
      const deduped = accepted.filter((s) => !existingKeys.has(s.key));
      return [...prev, ...deduped];
    });

    if (rejected.length > 0) {
      const shown = rejected.slice(0, 3).join(', ');
      const more = rejected.length > 3 ? ` and ${rejected.length - 3} more` : '';
      setRejectedNote(`Skipped unsupported file${rejected.length === 1 ? '' : 's'}: ${shown}${more}`);
    } else {
      setRejectedNote(null);
    }
  }, []);

  const removeStaged = (key: string) => {
    setStagedFiles((prev) => prev.filter((s) => s.key !== key));
  };

  const handleUploadStaged = async () => {
    if (stagedFiles.length === 0) return;
    try {
      await dispatch(uploadRagFiles({ datasetId, files: stagedFiles.map((s) => s.file) })).unwrap();
      setStagedFiles([]);
      dispatch(fetchRagUploadedFiles(datasetId));
      dispatch(fetchRagStatus(datasetId));
    } catch {
      // uploadError is already in state and rendered below; staged files
      // are kept so the user can retry without re-selecting everything.
    }
  };

  const handleStartProcessing = async () => {
    try {
      await dispatch(startRagProcessing(datasetId)).unwrap();
      setAwaitingProcessing(true);
      dispatch(fetchRagStatus(datasetId));
    } catch {
      // processingStartError is rendered below.
    }
  };

  const onDrop = (e: React.DragEvent<HTMLDivElement>) => {
    e.preventDefault();
    setDragOver(false);
    if (!canUpload) return;
    if (e.dataTransfer.files?.length) addFiles(e.dataTransfer.files);
  };

  const stats = useMemo(() => ([
    { label: 'Total', value: status?.total ?? knownFileCount, tier: 'other' as StatusTier },
    { label: 'Uploaded', value: status?.uploaded ?? null, tier: 'uploaded' as StatusTier },
    { label: 'Queued', value: status?.queued ?? null, tier: 'queued' as StatusTier },
    { label: 'Processing', value: status?.processing ?? null, tier: 'processing' as StatusTier },
    { label: 'Ready', value: status?.ready ?? null, tier: 'ready' as StatusTier },
    { label: 'Failed', value: status?.failed ?? null, tier: 'failed' as StatusTier },
  ].filter((s) => s.value !== null)), [status, knownFileCount]);

  const progressPct = status && status.total > 0
    ? Math.round(((status.ready + status.failed) / status.total) * 100)
    : null;

  return (
    <div className={styles['rag-panel']}>
      {/* ---- header --------------------------------------------------- */}
      <div className={styles['rag-panel__hdr']}>
        <div className={styles['rag-panel__hdr-info']}>
          <span className={styles['rag-panel__hdr-icon']}><Database size={15} /></span>
          <div>
            <p className={styles['rag-panel__eyebrow']}>RAG Dataset</p>
            <h2>Upload &amp; Processing</h2>
          </div>
        </div>
        <div className={styles['rag-panel__hdr-actions']}>
          {status && <PhaseMarker phase={phase} />}
          <button type="button" className={styles['rag-panel__refresh']} onClick={refreshAll} title="Refresh status and files">
            <RotateCw size={13} /> Refresh
          </button>
        </div>
      </div>

      {statusLoadFailed && (
        <div className={styles['rag-panel__inline-error']} style={{ marginTop: 0 }}>
          <AlertCircle size={13} />
          <span>{statusFetchError || 'Couldn’t load processing status.'}</span>
        </div>
      )}

      {initialLoading ? (
        <section className={styles['rag-panel__section']}>
          <div className={styles['rag-panel__empty']}>
            <Loader2 size={16} className={styles['rag-panel__spin']} /> Loading workspace…
          </div>
        </section>
      ) : (
        <>
          {/* ---- upload section ----------------------------------------- */}
          <section className={styles['rag-panel__section']}>
            <div className={styles['rag-panel__section-hdr']}>
              <h3>Upload files</h3>
              {canUpload && (
                <span className={styles['rag-panel__section-hint']}>
                  pdf · csv · excel · docx · pptx · md · html · txt · json · images
                </span>
              )}
            </div>

            {isCompleted ? (
              <div className={styles['rag-panel__locked-note']}>
                <CheckCircle2 size={15} />
                <span>Processing is complete — this dataset is now read-only.</span>
              </div>
            ) : isProcessing ? (
              <div className={styles['rag-panel__locked-note']}>
                <Clock size={15} />
                <span>Processing has started — new file uploads are disabled until it finishes.</span>
              </div>
            ) : (
              <>
                <div
                  className={`${styles['rag-panel__dropzone']} ${dragOver ? styles['rag-panel__dropzone--over'] : ''}`}
                  onDragOver={(e) => { e.preventDefault(); setDragOver(true); }}
                  onDragLeave={() => setDragOver(false)}
                  onDrop={onDrop}
                  onClick={() => fileInputRef.current?.click()}
                >
                  <UploadCloud size={22} />
                  <span className={styles['rag-panel__dropzone-text']}>
                    Drag files here, or click to browse
                  </span>
                  <span className={styles['rag-panel__dropzone-hint']}>
                    Select as many as you like — upload in batches, any number of times
                  </span>
                </div>
                <input
                  ref={fileInputRef}
                  type="file"
                  multiple
                  accept={ACCEPT_ATTR}
                  className={styles['rag-panel__file-input']}
                  onChange={(e) => {
                    if (e.target.files?.length) addFiles(e.target.files);
                    e.target.value = '';
                  }}
                />

                {rejectedNote && (
                  <div className={styles['rag-panel__inline-warn']}>
                    <AlertCircle size={13} /> {rejectedNote}
                  </div>
                )}

                {stagedFiles.length > 0 && (
                  <>
                    <div className={styles['rag-panel__staged-list']}>
                      <VirtualList
                        items={stagedFiles}
                        rowHeight={STAGED_ROW_H}
                        gap={STAGED_ROW_GAP}
                        maxHeight={STAGED_LIST_MAX_H}
                        getKey={(s) => s.key}
                        renderRow={(s) => (
                          <span className={styles['rag-panel__staged-chip']}>
                            <FileIcon size={12} />
                            <span>{s.file.name}</span>
                            <em>{formatBytes(s.file.size)}</em>
                            <button
                              type="button"
                              onClick={() => removeStaged(s.key)}
                              title="Remove"
                              disabled={uploadStatus === 'loading'}
                            >
                              <X size={11} />
                            </button>
                          </span>
                        )}
                      />
                    </div>

                    {uploadError && (
                      <div className={styles['rag-panel__inline-error']}>
                        <AlertCircle size={13} /> {uploadError}
                      </div>
                    )}

                    <button
                      type="button"
                      className={styles['rag-panel__upload-btn']}
                      onClick={handleUploadStaged}
                      disabled={uploadStatus === 'loading'}
                    >
                      {uploadStatus === 'loading'
                        ? <Loader2 size={14} className={styles['rag-panel__spin']} />
                        : <FilePlus2 size={14} />}
                      {uploadStatus === 'loading'
                        ? 'Uploading…'
                        : `Upload ${stagedFiles.length} file${stagedFiles.length === 1 ? '' : 's'}`}
                    </button>
                  </>
                )}
              </>
            )}
          </section>

          {/* ---- processing section --------------------------------------- */}
          <section className={styles['rag-panel__section']}>
            <div className={styles['rag-panel__section-hdr']}>
              <h3>Processing</h3>
              {status?.current_state && (
                <span className={styles['rag-panel__state-tag']}>{capitalize(status.current_state)}</span>
              )}
            </div>

            {stats.length > 0 && (
              <div className={styles['rag-panel__stats']}>
                {stats.map((s) => (
                  <div key={s.label} className={`${styles['rag-panel__stat']} ${styles[`rag-panel__stat--${s.tier}`]}`}>
                    <span className={styles['rag-panel__stat-val']}>{s.value}</span>
                    <span className={styles['rag-panel__stat-label']}>{s.label}</span>
                  </div>
                ))}
              </div>
            )}

            {!canUpload && progressPct !== null && (
              <div className={styles['rag-panel__progress-track']}>
                <div className={styles['rag-panel__progress-fill']} style={{ width: `${progressPct}%` }} />
              </div>
            )}

            {canUpload && processingStartError && (
              <div className={styles['rag-panel__inline-error']}>
                <AlertCircle size={13} /> {processingStartError}
              </div>
            )}

            {canUpload && (
              <button
                type="button"
                className={styles['rag-panel__process-btn']}
                onClick={handleStartProcessing}
                disabled={!canProcess}
                title={
                  !hasAnyFiles
                    ? 'Upload at least one file first'
                    : 'Start processing the uploaded files'
                }
              >
                {processingStartStatus === 'loading'
                  ? <Loader2 size={14} className={styles['rag-panel__spin']} />
                  : <PlayCircle size={14} />}
                {processingStartStatus === 'loading' ? 'Starting…' : 'Process Files'}
              </button>
            )}
          </section>

          {/* ---- files list ------------------------------------------------ */}
          <section className={styles['rag-panel__section']}>
            <div className={styles['rag-panel__section-hdr']}>
              <h3>Files {displayFiles.length > 0 && <em>{displayFiles.length}</em>}</h3>
            </div>

            {filesStatus === 'loading' && displayFiles.length === 0 && (
              <div className={styles['rag-panel__empty']}>
                <Loader2 size={16} className={styles['rag-panel__spin']} /> Loading files…
              </div>
            )}

            {filesStatus === 'failed' && displayFiles.length === 0 && (
              <div className={styles['rag-panel__empty']}>
                <AlertCircle size={16} /> {filesError || 'Failed to load uploaded files.'}
              </div>
            )}

            {filesStatus !== 'loading' && filesStatus !== 'failed' && displayFiles.length === 0 && (
              <div className={styles['rag-panel__empty']}>
                <FileIcon size={16} /> No files uploaded yet.
              </div>
            )}

            {displayFiles.length > 0 && (
              <VirtualList
                className={styles['rag-panel__file-list']}
                items={displayFiles}
                rowHeight={FILE_ROW_H}
                gap={FILE_ROW_GAP}
                maxHeight={FILE_LIST_MAX_H}
                getKey={(f) => f.id}
                onScrollActivity={markScrolling}
                renderRow={(f) => <FileRow f={f} />}
              />
            )}
          </section>
        </>
      )}
    </div>
  );
}























//Virtuallist.tsx
// VirtualList.tsx
//
// Minimal fixed-row-height windowing, tuned for very large lists (10k+ rows).
//
//  - Only rows in/near the viewport are mounted.
//  - Scroll position lives in a ref; React re-renders only when the scroll
//    crosses a "block" boundary (BLOCK rows), not on every scroll frame. The
//    overscan is larger than a block, so the viewport is always covered
//    between re-renders — scrolling itself is handled by the browser.
//  - Rows are positioned with transform (compositor-friendly) and isolated
//    with CSS containment so one row's layout can't invalidate the others.
//  - overscroll-behavior: contain stops the scroll from chaining into the
//    outer page column when the list hits its top/bottom.
import { useCallback, useEffect, useReducer, useRef } from 'react';
import type { ReactNode } from 'react';

const BLOCK = 5; // rows per re-render step

interface VirtualListProps<T> {
  items: T[];
  // Height of one row in px, INCLUDING the gap below it.
  rowHeight: number;
  // Gap between rows in px (subtracted from the rendered row height).
  gap?: number;
  // The list grows with its content up to this height, then scrolls.
  maxHeight: number;
  // Extra rows mounted above/below the viewport. Keep >= BLOCK.
  overscan?: number;
  // Called on every scroll event (cheap) — lets the parent hold off heavy
  // work, like applying a background poll, while the user is scrolling.
  onScrollActivity?: () => void;
  getKey: (item: T, index: number) => string | number;
  renderRow: (item: T, index: number) => ReactNode;
  className?: string;
}

export default function VirtualList<T>({
  items, rowHeight, gap = 0, maxHeight, overscan = 10, onScrollActivity, getKey, renderRow, className,
}: VirtualListProps<T>) {
  const ref = useRef<HTMLDivElement>(null);
  const rafRef = useRef<number | null>(null);
  const scrollTopRef = useRef(0);
  const blockRef = useRef(0);
  const [, rerender] = useReducer((n: number) => n + 1, 0);

  const totalHeight = Math.max(0, items.length * rowHeight - gap);
  const viewportHeight = Math.min(totalHeight, maxHeight);
  const visibleRows = Math.ceil(viewportHeight / rowHeight) + 1;

  const onScroll = useCallback(() => {
    onScrollActivity?.();
    if (rafRef.current !== null) return;
    rafRef.current = requestAnimationFrame(() => {
      rafRef.current = null;
      const el = ref.current;
      if (!el) return;
      scrollTopRef.current = el.scrollTop;
      const block = Math.floor(el.scrollTop / (rowHeight * BLOCK));
      if (block !== blockRef.current) {
        blockRef.current = block;
        rerender();
      }
    });
  }, [rowHeight, onScrollActivity]);

  useEffect(() => () => {
    if (rafRef.current !== null) cancelAnimationFrame(rafRef.current);
  }, []);

  // If the list shrinks, keep the scroll position valid.
  useEffect(() => {
    const el = ref.current;
    const max = Math.max(0, totalHeight - viewportHeight);
    if (el && el.scrollTop > max) {
      el.scrollTop = max;
      scrollTopRef.current = max;
      blockRef.current = Math.floor(max / (rowHeight * BLOCK));
      rerender();
    }
  }, [totalHeight, viewportHeight, rowHeight]);

  const startBlockRow = Math.floor(scrollTopRef.current / (rowHeight * BLOCK)) * BLOCK;
  const start = Math.max(0, startBlockRow - overscan);
  const end = Math.min(items.length, startBlockRow + BLOCK + visibleRows + overscan);

  const rows: ReactNode[] = [];
  for (let i = start; i < end; i++) {
    rows.push(
      <div
        key={getKey(items[i], i)}
        style={{
          position: 'absolute',
          top: 0,
          left: 0,
          right: 0,
          height: rowHeight - gap,
          transform: `translateY(${i * rowHeight}px)`,
          contain: 'layout paint style',
        }}
      >
        {renderRow(items[i], i)}
      </div>
    );
  }

  return (
    <div
      ref={ref}
      className={className}
      onScroll={onScroll}
      style={{
        height: viewportHeight,
        overflowY: 'auto',
        position: 'relative',
        overscrollBehavior: 'contain',
        contain: 'layout paint',
      }}
    >
      <div style={{ height: totalHeight, position: 'relative' }}>{rows}</div>
    </div>
  );
}
