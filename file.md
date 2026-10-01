//Rag.ts
import { apiClient } from '../client';

// File extensions accepted by the RAG upload step (POST /datasets/upload-rag).
export const RAG_UPLOAD_EXTENSIONS = [
  'pdf', 'csv', 'xlsx', 'xls', 'docx', 'pptx', 'md', 'html', 'htm', 'txt', 'json',
  'png', 'jpg', 'jpeg', 'gif', 'webp',
];

export interface RagFile {
  id: number;
  dataset_id: string;
  filename: string;
  ext: string;
  size: number;
  content_type: string;
  original_path: string;
  accessible_file_path: string;
  artifact_path: string;
  parser_engine: string;
  // Only present on entries returned inside the /datasets/status response —
  // the plain /datasets/uploaded_files list doesn't carry a processing
  // status per file.
  status?: string | null;
  error_message?: string | null;
  created_at: string;
  updated_at: string;
}

export interface RagStatusResponse {
  dataset_id: string;
  current_state: string; // "upload" | "processing" (spec gives these as examples, not an exhaustive enum)
  total: number;
  uploaded: number;
  queued: number;
  processing: number;
  ready: number;
  failed: number;
  files: RagFile[];
}

export interface CreateDatasetParams {
  name: string;
  description: string;
}

// ASSUMPTION — the request/response shape for this endpoint wasn't fully
// specified ("use same old one"), only that it's reused from the existing
// LLM/Agent create flow. Modeled here on name/description/eval_type, and
// the response is read defensively for whichever id field it actually
// returns (dataset_id or id) since that wasn't specified either.
interface CreateDatasetResponse {
  dataset_id?: string;
  id?: string;
  [key: string]: unknown;
}

export const ragApi = {
  // POST /datasets/create — same creation call used for the LLM/Agent flow,
  // just with eval_type fixed to "rag" and no file attached at this step.
  create: async ({ name, description }: CreateDatasetParams): Promise<string> => {
    const { data } = await apiClient.post<CreateDatasetResponse>('/datasets/create', {
      name,
      description,
      eval_type: 'rag',
    });
    const id = data?.dataset_id ?? data?.id;
    if (!id) throw new Error('Dataset was created, but no dataset id was returned.');
    return String(id);
  },

  // POST /datasets/upload-rag?dataset_id= — multipart, multiple files under
  // the "files" field. Can be called repeatedly for the same dataset; each
  // call is its own batch.
  uploadFiles: async (datasetId: string, files: File[]): Promise<void> => {
    const formData = new FormData();
    files.forEach((f) => formData.append('files', f));
    await apiClient.post('/datasets/upload-rag', formData, { params: { dataset_id: datasetId } });
  },

  // GET /datasets/uploaded_files — per the spec this takes no dataset_id
  // param and appears to return every uploaded file across every RAG
  // dataset, each carrying its own dataset_id. Filtered client-side to the
  // dataset in view.
  listUploadedFiles: async (datasetId: string): Promise<RagFile[]> => {
    const { data } = await apiClient.get<RagFile[]>('/datasets/uploaded_files');
    const all = Array.isArray(data) ? data : [];
    return all.filter((f) => f?.dataset_id === datasetId);
  },

  // POST /datasets/rag_processing?dataset_id=
  startProcessing: async (datasetId: string): Promise<void> => {
    await apiClient.post('/datasets/rag_processing', null, { params: { dataset_id: datasetId } });
  },

  // GET /datasets/status?dataset_id=
  fetchStatus: async (datasetId: string): Promise<RagStatusResponse> => {
    const { data } = await apiClient.get<RagStatusResponse>('/datasets/status', {
      params: { dataset_id: datasetId },
    });
    return data;
  },
};
















//Ragslice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { ragApi } from '../../api/endpoints/rag';
import type { CreateDatasetParams, RagFile, RagStatusResponse } from '../../api/endpoints/rag';

type AsyncStatus = 'idle' | 'loading' | 'succeeded' | 'failed';

interface RagState {
  createStatus: AsyncStatus;
  createError: string | null;

  uploadedFiles: RagFile[];
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

  uploadedFiles: [],
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
      return await ragApi.listUploadedFiles(datasetId);
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

// POST /datasets/rag_processing?dataset_id=
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
      return await ragApi.fetchStatus(datasetId);
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
      state.uploadedFiles = [];
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
        state.uploadedFiles = Array.isArray(action.payload) ? action.payload : [];
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
        state.status = action.payload ?? null;
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

















//Ragstudio.tsx
//RagStudio.tsx
import { useCallback, useEffect, useMemo, useRef, useState } from 'react';
import {
  X, UploadCloud, FilePlus2, Trash2, Loader2, PlayCircle, CheckCircle2, XCircle,
  Clock, AlertCircle, Database, RotateCw, File as FileIcon,
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import {
  fetchRagUploadedFiles, uploadRagFiles, startRagProcessing, fetchRagStatus,
} from '../../store/slices/ragSlice';
import { RAG_UPLOAD_EXTENSIONS } from '../../api/endpoints/rag';
import type { RagFile } from '../../api/endpoints/rag';
import styles from './RagStudio.module.scss';

const ACCEPT_ATTR = RAG_UPLOAD_EXTENSIONS.map((e) => `.${e}`).join(',');
const STATUS_POLL_MS = 4000;

interface StagedFile {
  key: string;
  file: File;
}

interface RagStudioProps {
  datasetId: string;
  datasetName: string;
  onClose: () => void;
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

export default function RagStudio({ datasetId, datasetName, onClose }: RagStudioProps) {
  const dispatch = useAppDispatch();
  const {
    uploadedFiles = [],
    filesStatus = 'idle',
    filesError = null,
    uploadStatus = 'idle',
    uploadError = null,
    processingStartStatus = 'idle',
    processingStartError = null,
    status = null,
    statusFetchStatus = 'idle',
  } = useAppSelector((s) => s.rag) ?? {};

  const [stagedFiles, setStagedFiles] = useState<StagedFile[]>([]);
  const [rejectedNote, setRejectedNote] = useState<string | null>(null);
  const [dragOver, setDragOver] = useState(false);
  const fileInputRef = useRef<HTMLInputElement>(null);

  // Initial load — always hit the API fresh rather than trusting any stale
  // local state, so this reflects reality even if the studio was closed and
  // reopened (or the whole app was reloaded) mid-processing.
  useEffect(() => {
    dispatch(fetchRagUploadedFiles(datasetId));
    dispatch(fetchRagStatus(datasetId));
  }, [dispatch, datasetId]);

  // Live status polling while the studio is open.
  useEffect(() => {
    const interval = setInterval(() => {
      dispatch(fetchRagStatus(datasetId));
    }, STATUS_POLL_MS);
    return () => clearInterval(interval);
  }, [dispatch, datasetId]);

  useEffect(() => {
    const onKey = (e: KeyboardEvent) => {
      if (e.key === 'Escape') onClose();
    };
    window.addEventListener('keydown', onKey);
    return () => window.removeEventListener('keydown', onKey);
  }, [onClose]);

  const isLocked = (status?.current_state || '').trim().toLowerCase() === 'processing';
  const totalFromStatus = status?.total ?? null;
  const knownFileCount = totalFromStatus ?? uploadedFiles.length;
  const hasAnyFiles = knownFileCount > 0;
  const canProcess = hasAnyFiles && !isLocked && processingStartStatus !== 'loading';

  // Prefer the richer per-file status list from /datasets/status once it's
  // loaded (it carries status + error_message); fall back to the plain
  // uploaded-files list before that first status fetch resolves.
  const displayFiles: RagFile[] = status?.files?.length ? status.files : uploadedFiles;

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
      dispatch(fetchRagStatus(datasetId));
    } catch {
      // processingStartError is rendered below.
    }
  };

  const onDrop = (e: React.DragEvent<HTMLDivElement>) => {
    e.preventDefault();
    setDragOver(false);
    if (isLocked) return;
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
    <div className={styles['rag-studio__overlay']} onClick={onClose}>
      <div className={styles['rag-studio']} role="dialog" aria-modal="true" aria-label="RAG dataset workspace" onClick={(e) => e.stopPropagation()}>
        {/* ---- header --------------------------------------------------- */}
        <div className={styles['rag-studio__hdr']}>
          <div className={styles['rag-studio__hdr-info']}>
            <span className={styles['rag-studio__hdr-icon']}><Database size={16} /></span>
            <div>
              <p className={styles['rag-studio__eyebrow']}>RAG Dataset</p>
              <h2>{datasetName || 'Untitled RAG dataset'}</h2>
            </div>
          </div>
          <button type="button" className={styles['rag-studio__close']} onClick={onClose} title="Close">
            <X size={17} />
          </button>
        </div>

        <div className={styles['rag-studio__body']}>
          {/* ---- upload section ----------------------------------------- */}
          <section className={styles['rag-studio__section']}>
            <div className={styles['rag-studio__section-hdr']}>
              <h3>Upload files</h3>
              <span className={styles['rag-studio__section-hint']}>
                pdf · csv · excel · docx · pptx · md · html · txt · json · images
              </span>
            </div>

            {isLocked ? (
              <div className={styles['rag-studio__locked-note']}>
                <Clock size={15} />
                <span>Processing has started — new file uploads are disabled until it finishes.</span>
              </div>
            ) : (
              <>
                <div
                  className={`${styles['rag-studio__dropzone']} ${dragOver ? styles['rag-studio__dropzone--over'] : ''}`}
                  onDragOver={(e) => { e.preventDefault(); setDragOver(true); }}
                  onDragLeave={() => setDragOver(false)}
                  onDrop={onDrop}
                  onClick={() => fileInputRef.current?.click()}
                >
                  <UploadCloud size={22} />
                  <span className={styles['rag-studio__dropzone-text']}>
                    Drag files here, or click to browse
                  </span>
                  <span className={styles['rag-studio__dropzone-hint']}>
                    Select as many as you like — upload in batches, any number of times
                  </span>
                </div>
                <input
                  ref={fileInputRef}
                  type="file"
                  multiple
                  accept={ACCEPT_ATTR}
                  className={styles['rag-studio__file-input']}
                  onChange={(e) => {
                    if (e.target.files?.length) addFiles(e.target.files);
                    e.target.value = '';
                  }}
                />

                {rejectedNote && (
                  <div className={styles['rag-studio__inline-warn']}>
                    <AlertCircle size={13} /> {rejectedNote}
                  </div>
                )}

                {stagedFiles.length > 0 && (
                  <>
                    <div className={styles['rag-studio__staged-list']}>
                      {stagedFiles.map((s) => (
                        <span key={s.key} className={styles['rag-studio__staged-chip']}>
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
                      ))}
                    </div>

                    {uploadError && (
                      <div className={styles['rag-studio__inline-error']}>
                        <AlertCircle size={13} /> {uploadError}
                      </div>
                    )}

                    <button
                      type="button"
                      className={styles['rag-studio__upload-btn']}
                      onClick={handleUploadStaged}
                      disabled={uploadStatus === 'loading'}
                    >
                      {uploadStatus === 'loading'
                        ? <Loader2 size={14} className={styles['rag-studio__spin']} />
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
          <section className={styles['rag-studio__section']}>
            <div className={styles['rag-studio__section-hdr']}>
              <h3>Processing</h3>
              {status?.current_state && (
                <span className={styles['rag-studio__state-tag']}>{capitalize(status.current_state)}</span>
              )}
            </div>

            {stats.length > 0 && (
              <div className={styles['rag-studio__stats']}>
                {stats.map((s) => (
                  <div key={s.label} className={`${styles['rag-studio__stat']} ${styles[`rag-studio__stat--${s.tier}`]}`}>
                    <span className={styles['rag-studio__stat-val']}>{s.value}</span>
                    <span className={styles['rag-studio__stat-label']}>{s.label}</span>
                  </div>
                ))}
              </div>
            )}

            {progressPct !== null && (
              <div className={styles['rag-studio__progress-track']}>
                <div className={styles['rag-studio__progress-fill']} style={{ width: `${progressPct}%` }} />
              </div>
            )}

            {processingStartError && (
              <div className={styles['rag-studio__inline-error']}>
                <AlertCircle size={13} /> {processingStartError}
              </div>
            )}

            <button
              type="button"
              className={styles['rag-studio__process-btn']}
              onClick={handleStartProcessing}
              disabled={!canProcess}
              title={
                !hasAnyFiles
                  ? 'Upload at least one file first'
                  : isLocked
                    ? 'Processing has already started'
                    : 'Start processing the uploaded files'
              }
            >
              {processingStartStatus === 'loading'
                ? <Loader2 size={14} className={styles['rag-studio__spin']} />
                : <PlayCircle size={14} />}
              {processingStartStatus === 'loading' ? 'Starting…' : 'Process Files'}
            </button>
          </section>

          {/* ---- files list ------------------------------------------------ */}
          <section className={styles['rag-studio__section']}>
            <div className={styles['rag-studio__section-hdr']}>
              <h3>Files {displayFiles.length > 0 && <em>{displayFiles.length}</em>}</h3>
            </div>

            {filesStatus === 'loading' && displayFiles.length === 0 && (
              <div className={styles['rag-studio__empty']}>
                <Loader2 size={16} className={styles['rag-studio__spin']} /> Loading files…
              </div>
            )}

            {filesStatus === 'failed' && displayFiles.length === 0 && (
              <div className={styles['rag-studio__empty']}>
                <AlertCircle size={16} /> {filesError || 'Failed to load uploaded files.'}
              </div>
            )}

            {filesStatus !== 'loading' && displayFiles.length === 0 && !filesError && (
              <div className={styles['rag-studio__empty']}>
                <FileIcon size={16} /> No files uploaded yet.
              </div>
            )}

            {displayFiles.length > 0 && (
              <div className={styles['rag-studio__file-list']}>
                {displayFiles.map((f) => {
                  const tier = statusTier(f.status);
                  return (
                    <div key={f.id} className={styles['rag-studio__file-row']}>
                      <span className={styles['rag-studio__file-icon']}><FileIcon size={14} /></span>
                      <div className={styles['rag-studio__file-info']}>
                        <span className={styles['rag-studio__file-name']} title={f.filename}>{f.filename}</span>
                        <span className={styles['rag-studio__file-meta']}>
                          {(f.ext || '').toUpperCase()} · {formatBytes(f.size)}
                        </span>
                      </div>
                      {f.status && (
                        <span
                          className={`${styles['rag-studio__file-status']} ${styles[`rag-studio__file-status--${tier}`]}`}
                          title={f.error_message || undefined}
                        >
                          {tier === 'ready' && <CheckCircle2 size={12} />}
                          {tier === 'processing' && <Loader2 size={12} className={styles['rag-studio__spin']} />}
                          {tier === 'queued' && <Clock size={12} />}
                          {tier === 'failed' && <XCircle size={12} />}
                          {capitalize(f.status)}
                        </span>
                      )}
                      {f.status && statusTier(f.status) === 'failed' && f.error_message && (
                        <span className={styles['rag-studio__file-error-icon']} title={f.error_message}>
                          <AlertCircle size={13} />
                        </span>
                      )}
                    </div>
                  );
                })}
              </div>
            )}
          </section>
        </div>

        {/* ---- footer -------------------------------------------------- */}
        <div className={styles['rag-studio__foot']}>
          <button type="button" className={styles['rag-studio__refresh']} onClick={() => { dispatch(fetchRagStatus(datasetId)); dispatch(fetchRagUploadedFiles(datasetId)); }}>
            <RotateCw size={13} /> Refresh
          </button>
          <button type="button" className={styles['rag-studio__done-btn']} onClick={onClose}>
            Done
          </button>
        </div>
      </div>
    </div>
  );
}



















//Ragstudio.module.scss
//RagStudio.module.scss
@use 'sass:color';
@use '../../styles/_variables' as *;

// ===========================================================================
// RAG Dataset Studio — a deliberately separate interface from the rest of
// Datasets. RAG datasets have a fundamentally different lifecycle (create →
// upload in batches → process → track status), so rather than bolting that
// onto the existing card/detail UI, this is its own sectioned workspace
// panel: Upload / Processing / Files, each a distinct card.
//
// Reuses the shared "ink" theme tokens from _variables.scss (already
// theme-aware via _theme.scss) so dark mode works here for free — nothing
// hardcoded except the flat brand-accent hexes the shared tokens don't
// cover (amber/queued tint).
// ===========================================================================

$queued-amber: #E08600;
$queued-amber-wash: #FDF3E3;

$soft: $shadow-2;
$lift: $shadow-3;

// Base font-size — same em-scaling convention as Datasets / History /
// Model Catalog / Sidebar / Providers.
$rag-studio-base-font: 0.8125rem;

%micro {
  font-family: $font-mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

.rag-studio {
  font-size: $rag-studio-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  width: min(720px, 100%);
  max-height: 90vh;
  display: flex;
  flex-direction: column;
  background: $card;
  border-radius: 20px;
  overflow: hidden;
  box-shadow: 0 30px 70px -20px rgba(20, 22, 27, 0.45);
  animation: rag-studio-in 0.2s cubic-bezier(0.22, 1, 0.36, 1);

  &__overlay {
    position: fixed;
    inset: 0;
    z-index: 75;
    background: rgba(20, 22, 27, 0.5);
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 24px;
    animation: rag-studio-fade 0.15s ease;
  }

  // ---- header ---------------------------------------------------------------
  &__hdr {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
    padding: 18px 20px;
    background: linear-gradient(135deg, $signal 0%, $signal-2 100%);
    color: #fff;
  }

  &__hdr-info {
    display: flex;
    align-items: center;
    gap: 12px;
    min-width: 0;

    h2 {
      font-family: $font-display;
      font-size: 1.3846em; // 1.125rem / 0.8125rem
      font-weight: 700;
      letter-spacing: -0.01em;
      color: #fff;
      margin: 2px 0 0;
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
    }
  }

  &__hdr-icon {
    flex-shrink: 0;
    width: 34px;
    height: 34px;
    border-radius: 10px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    background: rgba(255, 255, 255, 0.16);
    color: #fff;
  }

  &__eyebrow {
    font-family: $font-mono;
    font-size: 0.6923em; // 0.5625rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    color: rgba(255, 255, 255, 0.75);
  }

  &__close {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 32px;
    height: 32px;
    border-radius: 9px;
    border: 1px solid transparent;
    background: rgba(255, 255, 255, 0.12);
    color: #fff;
    cursor: pointer;
    transition: background 0.15s ease;

    &:hover { background: rgba(255, 255, 255, 0.24); }
  }

  // ---- body / sections --------------------------------------------------------
  &__body {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding: 20px;
    display: flex;
    flex-direction: column;
    gap: 16px;
    background: $paper;
  }

  &__section {
    background: $card;
    border: 1px solid $line;
    border-radius: 16px;
    padding: 18px;
  }

  &__section-hdr {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
    margin-bottom: 14px;

    h3 {
      font-family: $font-display;
      font-size: 1.0769em; // 0.875rem / 0.8125rem
      font-weight: 700;
      color: $ink;
      margin: 0;
      display: flex;
      align-items: center;
      gap: 8px;

      em {
        font-family: $font-mono;
        font-style: normal;
        font-size: 0.7143em;
        font-weight: 700;
        color: $ink-3;
        background: $paper;
        border: 1px solid $line;
        border-radius: 99px;
        padding: 1px 8px;
      }
    }
  }

  &__section-hint {
    font-family: $font-mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    color: $ink-3;
    text-align: right;
  }

  &__state-tag {
    font-family: $font-mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.05em;
    text-transform: uppercase;
    color: $signal;
    background: $wash;
    border-radius: 999px;
    padding: 3px 10px;
    white-space: nowrap;
  }

  // ---- dropzone --------------------------------------------------------------
  &__dropzone {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 6px;
    padding: 30px 16px;
    border: 1.5px dashed $line;
    border-radius: 14px;
    background: $paper;
    color: $ink-3;
    cursor: pointer;
    text-align: center;
    transition: border-color 0.15s ease, background 0.15s ease, transform 0.1s ease;

    &:hover { border-color: $signal; background: $wash; }

    &--over {
      border-color: $signal;
      background: $wash;
      transform: scale(1.005);
    }

    svg { color: $signal; margin-bottom: 2px; }
  }

  &__dropzone-text {
    font-family: $font-body;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 650;
    color: $ink;
  }

  &__dropzone-hint {
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    color: $ink-3;
  }

  &__file-input {
    display: none;
  }

  &__locked-note {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 13px 15px;
    border: 1px dashed $line;
    border-radius: 12px;
    background: $paper;
    color: $ink-2;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    line-height: 1.5;

    svg { flex-shrink: 0; color: $ink-3; }
  }

  &__inline-warn,
  &__inline-error {
    display: flex;
    align-items: center;
    gap: 7px;
    margin-top: 10px;
    padding: 8px 11px;
    border-radius: 9px;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    font-weight: 600;
    line-height: 1.5;

    svg { flex-shrink: 0; }
  }

  &__inline-warn {
    background: $queued-amber-wash;
    color: $queued-amber;
  }

  &__inline-error {
    background: $danger-wash;
    color: $danger;
  }

  // ---- staged files ------------------------------------------------------------
  &__staged-list {
    display: flex;
    flex-wrap: wrap;
    gap: 7px;
    margin-top: 14px;
  }

  &__staged-chip {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    max-width: 100%;
    padding: 5px 6px 5px 10px;
    border: 1px solid $line;
    border-radius: 999px;
    background: $paper;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    color: $ink-2;

    svg:first-child { flex-shrink: 0; color: $ink-3; }

    span {
      max-width: 180px;
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
      font-weight: 600;
      color: $ink;
    }

    em {
      flex-shrink: 0;
      font-family: $font-mono;
      font-style: normal;
      font-size: 0.85em;
      color: $ink-3;
    }

    button {
      flex-shrink: 0;
      display: inline-flex;
      align-items: center;
      justify-content: center;
      width: 18px;
      height: 18px;
      border: 0;
      border-radius: 999px;
      background: $line-2;
      color: $ink-2;
      cursor: pointer;
      transition: background 0.13s ease, color 0.13s ease;

      &:hover:not(:disabled) { background: $danger-wash; color: $danger; }
      &:disabled { opacity: 0.5; cursor: not-allowed; }
    }
  }

  &__upload-btn {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    margin-top: 14px;
    padding: 9px 16px;
    border: 1px solid transparent;
    border-radius: 10px;
    background: $signal;
    color: #fff;
    font-family: $font-body;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: background 0.15s ease;

    &:hover:not(:disabled) { background: $signal-2; }
    &:disabled { opacity: 0.6; cursor: not-allowed; }
  }

  // ---- processing stats --------------------------------------------------------
  &__stats {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(84px, 1fr));
    gap: 8px;
    margin-bottom: 14px;
  }

  &__stat {
    display: flex;
    flex-direction: column;
    gap: 2px;
    padding: 10px 12px;
    border-radius: 12px;
    background: $paper;
    border: 1px solid $line;
  }

  &__stat-val {
    font-family: $font-display;
    font-size: 1.3846em; // 1.125rem / 0.8125rem
    font-weight: 700;
    color: $ink;
    line-height: 1;
  }

  &__stat-label {
    font-family: $font-mono;
    font-size: 0.6154em; // 0.5rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ink-3;
    margin-top: 4px;
  }

  &__stat--uploaded .rag-studio__stat-val { color: $ink-2; }
  &__stat--queued .rag-studio__stat-val { color: $queued-amber; }
  &__stat--processing .rag-studio__stat-val { color: $signal; }
  &__stat--ready .rag-studio__stat-val { color: $ok; }
  &__stat--failed .rag-studio__stat-val { color: $danger; }

  &__progress-track {
    height: 8px;
    border-radius: 999px;
    background: $line-2;
    overflow: hidden;
    margin-bottom: 14px;
  }

  &__progress-fill {
    height: 100%;
    border-radius: 999px;
    background: linear-gradient(90deg, $signal, $ok);
    transition: width 0.4s ease;
  }

  &__process-btn {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    padding: 10px 20px;
    border: 1px solid transparent;
    border-radius: 10px;
    background: $ok;
    color: #fff;
    font-family: $font-body;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 700;
    cursor: pointer;
    transition: background 0.15s ease, transform 0.15s ease;

    &:hover:not(:disabled) { background: color.scale($ok, $lightness: -10%); transform: translateY(-1px); }
    &:disabled { opacity: 0.45; cursor: not-allowed; transform: none; }
  }

  // ---- file list -----------------------------------------------------------
  &__empty {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
    padding: 28px 12px;
    color: $ink-3;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    text-align: center;
  }

  &__file-list {
    display: flex;
    flex-direction: column;
    gap: 6px;
    max-height: 280px;
    overflow-y: auto;
  }

  &__file-row {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 9px 11px;
    border: 1px solid $line;
    border-radius: 10px;
    background: $paper;
  }

  &__file-icon {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 28px;
    height: 28px;
    border-radius: 8px;
    background: $card;
    border: 1px solid $line;
    color: $ink-3;
  }

  &__file-info {
    flex: 1;
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 2px;
  }

  &__file-name {
    font-size: 0.8846em; // 0.71875rem / 0.8125rem
    font-weight: 650;
    color: $ink;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__file-meta {
    font-family: $font-mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    color: $ink-3;
  }

  &__file-status {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 3px 9px;
    border-radius: 999px;
    font-family: $font-mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.03em;
    text-transform: uppercase;
    white-space: nowrap;

    &--uploaded { color: $ink-2; background: $ink-wash; }
    &--queued { color: $queued-amber; background: $queued-amber-wash; }
    &--processing { color: $signal; background: $wash; }
    &--ready { color: $ok; background: $ok-wash; }
    &--failed { color: $danger; background: $danger-wash; }
    &--other { color: $ink-3; background: $ink-wash; }
  }

  &__file-error-icon {
    flex-shrink: 0;
    display: inline-flex;
    color: $danger;
    cursor: help;
  }

  // ---- footer -----------------------------------------------------------------
  &__foot {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
    padding: 14px 20px;
    border-top: 1px solid $line;
    background: $card;
  }

  &__refresh {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 8px 13px;
    border: 1px solid $line;
    border-radius: 999px;
    background: $paper;
    color: $ink-2;
    font-family: $font-body;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $ink-3; color: $ink; }
  }

  &__done-btn {
    padding: 9px 20px;
    border: 1px solid transparent;
    border-radius: 10px;
    background: $ink;
    color: #fff;
    font-family: $font-body;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: background 0.15s ease;

    &:hover { background: color.scale($ink, $lightness: 12%); }
  }

  &__spin {
    animation: rag-studio-spin 0.8s linear infinite;
  }
}

@keyframes rag-studio-fade {
  from { opacity: 0; }
  to { opacity: 1; }
}

@keyframes rag-studio-in {
  from { opacity: 0; transform: translateY(10px) scale(0.98); }
  to { opacity: 1; transform: none; }
}

@keyframes rag-studio-spin {
  to { transform: rotate(360deg); }
}

@media (max-width: 640px) {
  .rag-studio__overlay { padding: 0; }
  .rag-studio { width: 100%; max-width: 100vw; max-height: 100vh; height: 100%; border-radius: 0; }
  .rag-studio__stats { grid-template-columns: repeat(3, 1fr); }
}

@media (prefers-reduced-motion: reduce) {
  .rag-studio__overlay,
  .rag-studio,
  .rag-studio__spin { animation: none; }
  .rag-studio__dropzone,
  .rag-studio__upload-btn,
  .rag-studio__process-btn,
  .rag-studio__close,
  .rag-studio__refresh,
  .rag-studio__done-btn { transition: none; }
}


















//Datasets.tsx
//Datasets.tsx
import { useEffect, useMemo, useState, useRef, useCallback } from 'react';
import {
  RefreshCw, Search, Layers, AlertTriangle, Database, ListFilter, X,
  Check, Boxes, ArrowRight, Filter, ChevronsUpDown, Upload, Loader2,
  FileUp, AlertCircle, CheckCircle2, Trash2, Eye, ChevronLeft, ChevronRight, HelpCircle,
  Image as ImageIcon, Download, RotateCw,
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import {
  fetchTestSuites, uploadDataset, resetUploadStatus, deleteTestSuite, resetDeleteError,
  fetchDatasetPreview, resetPreview,
} from '../../store/slices/testSuitesSlice';
import type { TestSuite, PreviewState } from '../../store/slices/testSuitesSlice';
import { testSuitesApi, SUPPORTED_UPLOAD_EXTENSIONS } from '../../api/endpoints/testSuites';
import type { EvalType } from '../../api/endpoints/testSuites';
import { createRagDataset, resetRagCreate, resetRagStudio } from '../../store/slices/ragSlice';
import RagStudio from './RagStudio';
import styles from './Datasets.module.scss';

// Deterministic color hash so the same category always gets the same pill
// color across renders, without a hardcoded lookup table. Palette matches
// the app's ink/paper/signal design tokens.
const PILL_COLORS = [
  { bg: '#ECEDFF', fg: '#2B2BF5' }, // signal
  { bg: '#FDF3E3', fg: '#C56A00' }, // amber
  { bg: '#E7F7EF', fg: '#0B8F58' }, // ok
  { bg: '#FDECEC', fg: '#C81E1E' }, // danger
  { bg: '#E6F4FB', fg: '#0369A1' }, // sky
  { bg: '#F1EDFB', fg: '#6D28D9' }, // violet
  { bg: '#EAF6EC', fg: '#3F7D20' }, // moss
];
function hashColor(label?: string | null) {
  const safe = label || '—';
  const sum = [...safe].reduce((acc, ch) => acc + ch.charCodeAt(0), 0);
  return PILL_COLORS[sum % PILL_COLORS.length];
}

// Categories/tags come back with inconsistent casing across datasets
// (e.g. "Chemistry" vs "chemistry") but should be treated as the same
// category for filtering, active-state highlighting, and color assignment.
function normalizeTag(t?: string | null): string {
  return (t || '').trim().toLowerCase();
}

// Renders a readable filename from a MinIO object key
// ("datasets/ds-001/images/image1.jpg" -> "image1.jpg").
function imageBasename(path: string): string {
  const parts = path.split('/');
  return parts[parts.length - 1] || path;
}

// Question metadata shape varies across preview responses (level/source/
// annotator on some, something else entirely on others) — rather than
// typing and rendering every possible field by hand, flatten whatever
// object comes back into displayable [label, value] pairs. Nested objects
// become dotted labels ("annotator.number_of_steps"); arrays are joined
// into a single readable string; null/undefined values are dropped.
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

// "annotator.number_of_steps" -> "Annotator · Number Of Steps"
function formatMetaLabel(label: string): string {
  return label
    .split('.')
    .map((seg) => seg.replace(/_/g, ' ').replace(/\b\w/g, (c) => c.toUpperCase()))
    .join(' · ');
}

const EVAL_TYPE_OPTIONS: { value: EvalType; label: string }[] = [
  { value: 'model', label: 'Model' },
  { value: 'agent', label: 'Agent' },
  { value: 'rag', label: 'RAG' },
];

const ACCEPT_ATTR = SUPPORTED_UPLOAD_EXTENSIONS.map((e) => `.${e}`).join(',');
const PREVIEW_PAGE_SIZE_OPTIONS = [10, 20, 50];

// Delete is only offered for user-uploaded custom datasets, not built-in
// benchmark suites.
function isCustomDataset(d: TestSuite | null | undefined): boolean {
  return (d?.dataset_type || '').toLowerCase() === 'custom';
}

export default function Datasets() {
  const dispatch = useAppDispatch();
  const {
    items,
    status = 'idle',
    error = null,
    uploadStatus = 'idle',
    uploadError = null,
    deletingId = null,
    deleteError = null,
    preview,
  } = useAppSelector((s) => s.testSuites) ?? {};
  const safeItems = items ?? [];
  const previewState = preview ?? {
    datasetId: null, questions: [], total: 0, limit: 20, offset: 0, status: 'idle' as const, error: null,
  };

  const {
    createStatus: ragCreateStatus = 'idle',
    createError: ragCreateError = null,
  } = useAppSelector((s) => s.rag) ?? {};

  const [search, setSearch] = useState('');
  const [datasetTypeFilter, setDatasetTypeFilter] = useState('All');
  const [tagFilter, setTagFilter] = useState<string[]>([]);       // active dataset_categories facets
  const [selectedId, setSelectedId] = useState<string | null>(null);
  const [uploadOpen, setUploadOpen] = useState(false);
  const [deleteTarget, setDeleteTarget] = useState<TestSuite | null>(null);
  const [previewTarget, setPreviewTarget] = useState<TestSuite | null>(null);
  const [lightboxPath, setLightboxPath] = useState<string | null>(null);
  const [ragStudioTarget, setRagStudioTarget] = useState<{ id: string; name: string } | null>(null);
  const searchRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    dispatch(fetchTestSuites());
  }, [dispatch]);

  const datasetTypes = useMemo(
    () => ['All', ...new Set(safeItems.map((d) => d?.dataset_type).filter((t): t is string => Boolean(t)))],
    [safeItems]
  );

  const filtered = useMemo(() => {
    const q = search.trim().toLowerCase();
    return safeItems.filter((d) => {
      if (!d) return false;
      if (datasetTypeFilter !== 'All' && (d.dataset_type ?? '') !== datasetTypeFilter) return false;
      const tags = d.dataset_categories ?? [];
      if (tagFilter.length && !tagFilter.some((t) => tags.some((tag) => normalizeTag(tag) === normalizeTag(t)))) return false;
      if (!q) return true;
      const name = (d.name ?? '').toLowerCase();
      const desc = (d.description ?? '').toLowerCase();
      return name.includes(q) || desc.includes(q);
    });
  }, [safeItems, search, datasetTypeFilter, tagFilter]);

  // Keep a valid selection as the filtered set changes.
  useEffect(() => {
    if (!filtered.length) return;
    if (!filtered.some((d) => d?.id && d.id === selectedId)) {
      const first = filtered[0];
      if (first?.id) setSelectedId(first.id);
    }
  }, [filtered, selectedId]);

  const selected = safeItems.find((d) => d?.id && d.id === selectedId) ?? null;

  const toggleTag = useCallback((t: string) => {
    setTagFilter((prev) => {
      const normalized = normalizeTag(t);
      const exists = prev.some((x) => normalizeTag(x) === normalized);
      return exists ? prev.filter((x) => normalizeTag(x) !== normalized) : [...prev, t];
    });
  }, []);

  // Keyboard: ↑/↓ walk the list, "/" focuses search.
  useEffect(() => {
    const onKey = (e: KeyboardEvent) => {
      const el = document.activeElement as HTMLElement | null;
      const typing = el?.tagName === 'INPUT' || el?.tagName === 'TEXTAREA';
      if (e.key === '/' && !typing) {
        e.preventDefault();
        searchRef.current?.focus();
        return;
      }
      if (typing || (e.key !== 'ArrowDown' && e.key !== 'ArrowUp')) return;
      e.preventDefault();
      const idx = filtered.findIndex((d) => d?.id && d.id === selectedId);
      if (idx === -1) {
        const first = filtered[0];
        if (first?.id) setSelectedId(first.id);
        return;
      }
      const next = e.key === 'ArrowDown'
        ? Math.min(idx + 1, filtered.length - 1)
        : Math.max(idx - 1, 0);
      const nextItem = filtered[next];
      if (nextItem?.id) setSelectedId(nextItem.id);
    };
    window.addEventListener('keydown', onKey);
    return () => window.removeEventListener('keydown', onKey);
  }, [filtered, selectedId]);

  // Closes the modal + refreshes the list once an upload succeeds. The
  // upload endpoints return a bare 200 with no dataset body, so a re-fetch
  // is the only way to pick up the new row.
  useEffect(() => {
    if (uploadStatus !== 'succeeded') return;
    dispatch(fetchTestSuites());
    setUploadOpen(false);
    const t = setTimeout(() => dispatch(resetUploadStatus()), 300);
    return () => clearTimeout(t);
  }, [uploadStatus, dispatch]);

  const closeUploadModal = () => {
    setUploadOpen(false);
    if (uploadStatus !== 'idle') dispatch(resetUploadStatus());
    if (ragCreateStatus !== 'idle') dispatch(resetRagCreate());
  };

  // Creates the (empty) RAG dataset, then hands off from the upload modal
  // straight into the dedicated RAG studio — file upload/processing for RAG
  // never happens in the upload modal itself.
  const handleCreateRagDataset = async (params: { name: string; description: string }) => {
    try {
      const newId = await dispatch(createRagDataset(params)).unwrap();
      setUploadOpen(false);
      dispatch(resetRagCreate());
      setRagStudioTarget({ id: newId, name: params.name });
      dispatch(fetchTestSuites()); // picks up the new (still-empty) dataset in the list
    } catch {
      // ragCreateError is already populated in state; UploadModal renders it
      // and stays open so the user can retry.
    }
  };

  const closeRagStudio = () => {
    setRagStudioTarget(null);
    dispatch(resetRagStudio());
    dispatch(fetchTestSuites()); // question counts / dataset_type etc. may have changed
  };

  const closeDeleteModal = () => {
    setDeleteTarget(null);
    if (deleteError) dispatch(resetDeleteError());
  };

  const confirmDelete = async () => {
    if (!deleteTarget?.id) return;
    try {
      await dispatch(deleteTestSuite(deleteTarget.id)).unwrap();
      if (selectedId === deleteTarget.id) setSelectedId(null);
      setDeleteTarget(null);
    } catch {
      // deleteError is already populated in state; the confirm modal stays
      // open and renders it.
    }
  };

  // Fetches page 1 as soon as the preview drawer opens for a dataset.
  const openPreview = (dataset: TestSuite) => {
    setPreviewTarget(dataset);
    if (dataset.id) {
      dispatch(fetchDatasetPreview({ datasetId: dataset.id, limit: previewState.limit || 20, offset: 0 }));
    }
  };

  const closePreview = () => {
    setPreviewTarget(null);
    dispatch(resetPreview());
  };

  const changePreviewOffset = (offset: number) => {
    if (!previewTarget?.id) return;
    dispatch(fetchDatasetPreview({ datasetId: previewTarget.id, limit: previewState.limit, offset }));
  };

  const changePreviewLimit = (limit: number) => {
    if (!previewTarget?.id) return;
    dispatch(fetchDatasetPreview({ datasetId: previewTarget.id, limit, offset: 0 }));
  };

  // Close the preview drawer with Escape — same pattern as History's
  // details drawer. Yields to the lightbox first: if an image is open on
  // top of the drawer, Escape should close just the image, not both.
  useEffect(() => {
    if (!previewTarget) return;
    const onKey = (e: KeyboardEvent) => {
      if (e.key !== 'Escape') return;
      if (lightboxPath) return;
      closePreview();
    };
    window.addEventListener('keydown', onKey);
    return () => window.removeEventListener('keydown', onKey);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [previewTarget, lightboxPath]);

  // Close the image lightbox with Escape.
  useEffect(() => {
    if (!lightboxPath) return;
    const onKey = (e: KeyboardEvent) => {
      if (e.key === 'Escape') setLightboxPath(null);
    };
    window.addEventListener('keydown', onKey);
    return () => window.removeEventListener('keydown', onKey);
  }, [lightboxPath]);

  return (
    <div className="page-enter pg-shell">
      {/* ---- header (unchanged, + upload button) --------------------------- */}
      <div className={styles['datasets__header']}>
        <div>
          <p className={styles['datasets__header-eyebrow']}>Datasets</p>
          <h1>Test Suite Library</h1>
          <p className={styles['datasets__header-sub']}>
            Browse every dataset available for evaluations, independent of any single wizard run.
          </p>
        </div>
        <div className={styles['datasets__header-meta']}>
          <span className={styles['datasets__header-count']}>
            <Database size={13} /> {safeItems.length} datasets available
          </span>
          <button className={styles['datasets__refresh-btn']} onClick={() => dispatch(fetchTestSuites())}>
            <RefreshCw size={14} /> Refresh
          </button>
          <button className={styles['datasets__upload-btn']} onClick={() => setUploadOpen(true)}>
            <Upload size={14} /> Upload dataset
          </button>
        </div>
      </div>

      {/* ---- toolbar (unchanged) ------------------------------------------ */}
      <div className={styles['datasets__toolbar']}>
        <div className={styles['datasets__search']}>
          <Search size={16} />
          <input
            ref={searchRef}
            placeholder="Search datasets…  (press /)"
            value={search}
            onChange={(e) => setSearch(e.target.value)}
          />
        </div>
        <div className={styles['datasets__filters']}>
          <span className={styles['datasets__toolbar-label']}>
            <ListFilter size={11} />
          </span>
          {datasetTypes.map((t) => (
            <button
              key={t}
              className={`${styles['datasets__filter-pill']} ${datasetTypeFilter === t ? styles['datasets__filter-pill--on'] : ''}`}
              onClick={() => setDatasetTypeFilter(t)}
            >
              {t}
            </button>
          ))}
        </div>
      </div>

      {/* ---- active tag facets --------------------------------------------- */}
      {tagFilter.length > 0 && (
        <div className={styles['datasets__facets']}>
          <Filter size={12} />
          <span className={styles['datasets__facets-lead']}>Showing datasets tagged</span>
          {tagFilter.map((t) => {
            const col = hashColor(normalizeTag(t));
            return (
              <button
                key={t}
                className={styles['datasets__facet']}
                style={{ background: col.bg, color: col.fg }}
                onClick={() => toggleTag(t)}
              >
                {t} <X size={11} />
              </button>
            );
          })}
          <button className={styles['datasets__facets-clear']} onClick={() => setTagFilter([])}>
            Clear all
          </button>
        </div>
      )}

      {/* ---- body: master / detail ---------------------------------------- */}
      <div className={styles['datasets__body']}>
        {status === 'failed' && (
          <div
            className={`${styles['datasets__state']} ${styles['datasets__state--error']}`}
            style={{ gridColumn: '1 / -1' }}
          >
            <AlertTriangle size={28} />
            <div>{error || 'Failed to load datasets.'}</div>
            <button className={styles['datasets__refresh-btn']} onClick={() => dispatch(fetchTestSuites())}>
              Retry
            </button>
          </div>
        )}

        {status !== 'failed' && (
          <>
            {/* LIST RAIL — bordered cards, one per dataset */}
            <aside className={styles['datasets__rail']}>
              <div className={styles['datasets__rail-head']}>
                <span>{filtered.length} of {safeItems.length}</span>
                <span className={styles['datasets__rail-hint']}>
                  <ChevronsUpDown size={11} /> ↑ ↓ to move
                </span>
              </div>
              <div className={styles['datasets__rail-scroll']}>
                {status === 'loading' &&
                  Array.from({ length: 7 }).map((_, i) => (
                    <div className={styles['datasets__skel-row']} key={i}>
                      <span className={styles['datasets__skel']} style={{ width: '55%' }} />
                      <span className={styles['datasets__skel']} style={{ width: '35%' }} />
                    </div>
                  ))}

                {status !== 'loading' && filtered.length === 0 && (
                  <div className={styles['datasets__empty-rail']}>
                    <Layers size={22} />
                    <p>No datasets match.<br />Loosen a filter to see more.</p>
                  </div>
                )}

                {status !== 'loading' &&
                  filtered.map((d, i) => {
                    if (!d) return null;
                    const rowKey = d.id ?? `row-${i}`;
                    const on = !!d.id && d.id === selectedId;
                    const dsType = d.dataset_type || 'Unknown';
                    const evalTypeLabel = d.eval_type || null;
                    const accent = hashColor(dsType);
                    const tags = d.dataset_categories ?? [];
                    const count = typeof d.question_count === 'number' ? d.question_count : 0;
                    return (
                      <button
                        key={rowKey}
                        className={`${styles['datasets__row']} ${on ? styles['datasets__row--on'] : ''}`}
                        onClick={() => d.id && setSelectedId(d.id)}
                        style={{ borderLeftColor: accent.fg }}
                      >
                        <div className={styles['datasets__row-top']}>
                          <span className={styles['datasets__row-name']}>{d.name || 'Untitled dataset'}</span>
                          <span className={styles['datasets__row-count']}>{count.toLocaleString()}</span>
                        </div>
                        <div className={styles['datasets__row-foot']}>
                          <span className={styles['datasets__row-type-group']}>
                            <span className={styles['datasets__row-type']} style={{ color: accent.fg }}>
                              {dsType}
                            </span>
                            {evalTypeLabel && (
                              <span className={styles['datasets__row-eval-tag']}>{evalTypeLabel}</span>
                            )}
                          </span>
                          <span className={styles['datasets__row-dots']}>
                            {tags.slice(0, 4).map((t, ti) => (
                              <i key={t || ti} style={{ background: hashColor(normalizeTag(t)).fg }} title={t || undefined} />
                            ))}
                          </span>
                        </div>
                      </button>
                    );
                  })}
              </div>
            </aside>

            {/* DETAIL */}
            <section className={styles['datasets__detail']}>
              {status === 'loading' ? (
                <div className={styles['datasets__detail-scroll']}>
                  <span className={styles['datasets__skel']} style={{ width: 120, height: 34, marginBottom: 22 }} />
                  <span className={styles['datasets__skel']} style={{ width: '90%', marginBottom: 8 }} />
                  <span className={styles['datasets__skel']} style={{ width: '70%' }} />
                </div>
              ) : !selected ? (
                <div className={styles['datasets__detail-empty']}>
                  <Boxes size={30} />
                  <p>Select a dataset to inspect its categories, questions, and eval type.</p>
                </div>
              ) : (
                <DetailView
                  dataset={selected}
                  tagFilter={tagFilter}
                  toggleTag={toggleTag}
                  onDeleteClick={() => setDeleteTarget(selected)}
                  onPreviewClick={() => openPreview(selected)}
                />
              )}
            </section>
          </>
        )}
      </div>

      {/* ---- upload dataset modal ------------------------------------------ */}
      {uploadOpen && (
        <UploadModal
          uploadStatus={uploadStatus}
          uploadError={uploadError}
          ragCreateStatus={ragCreateStatus}
          ragCreateError={ragCreateError}
          onClose={closeUploadModal}
          onSubmit={(params) => dispatch(uploadDataset(params))}
          onSubmitRag={handleCreateRagDataset}
        />
      )}

      {/* ---- delete dataset confirmation ------------------------------------ */}
      {deleteTarget && (
        <DeleteConfirmModal
          dataset={deleteTarget}
          isDeleting={!!deleteTarget.id && deletingId === deleteTarget.id}
          deleteError={deleteError}
          onCancel={closeDeleteModal}
          onConfirm={confirmDelete}
        />
      )}

      {/* ---- question preview slide-over drawer ----------------------------- */}
      <div
        className={`${styles['datasets__preview-overlay']} ${previewTarget ? styles['datasets__preview-overlay--open'] : ''}`}
        onClick={closePreview}
      />
      <div
        className={`${styles['datasets__preview-drawer']} ${previewTarget ? styles['datasets__preview-drawer--open'] : ''}`}
        role="dialog"
        aria-hidden={!previewTarget}
        aria-label="Dataset question preview"
      >
        {previewTarget && (
          <PreviewDrawer
            dataset={previewTarget}
            preview={previewState}
            onClose={closePreview}
            onChangeOffset={changePreviewOffset}
            onChangeLimit={changePreviewLimit}
            onOpenImage={setLightboxPath}
          />
        )}
      </div>

      {/* ---- MinIO image lightbox — mounted only while an image is open,
          fetched lazily on demand rather than preloaded with the drawer */}
      {lightboxPath && (
        <ImageLightbox imagePath={lightboxPath} onClose={() => setLightboxPath(null)} />
      )}

      {/* ---- RAG dataset studio — a fully separate workspace, opened once
          the (empty) RAG dataset has been created. Own component/stylesheet,
          nothing here shared with the rest of the Datasets UI. */}
      {ragStudioTarget && (
        <RagStudio
          datasetId={ragStudioTarget.id}
          datasetName={ragStudioTarget.name}
          onClose={closeRagStudio}
        />
      )}
    </div>
  );
}

// ---------------------------------------------------------------------------

interface DetailViewProps {
  dataset: TestSuite;
  tagFilter: string[];
  toggleTag: (t: string) => void;
  onDeleteClick: () => void;
  onPreviewClick: () => void;
}

function DetailView({ dataset: d, tagFilter, toggleTag, onDeleteClick, onPreviewClick }: DetailViewProps) {
  if (!d) return null;

  const category = d.category || 'Uncategorized';
  const datasetType = d.dataset_type || 'Unknown';
  const evalType = d.eval_type || '—';
  const questionCount = typeof d.question_count === 'number' ? d.question_count : 0;
  const accent = hashColor(category);
  const tags = d.dataset_categories ?? [];
  const deletable = isCustomDataset(d);

  return (
    <div className={styles['datasets__detail-scroll']} key={d.id ?? d.name ?? 'selected'}>
      <div className={styles['datasets__hero']}>
        <span className={styles['datasets__hero-bar']} style={{ background: accent.fg }} />
        <div className={styles['datasets__hero-top']}>
          <div>
            <span className={styles['datasets__hero-type']} style={{ color: accent.fg }}>{category}</span>
            <h2>{d.name || 'Untitled dataset'}</h2>
          </div>
          <div className={styles['datasets__hero-actions']}>
            <span className={styles['datasets__hero-badge-group']}>
              <span className={styles['datasets__hero-badge-label']}>Dataset Type</span>
              <span className={styles['datasets__source-badge']}>{datasetType}</span>
            </span>
            {d.eval_type && (
              <span className={styles['datasets__hero-badge-group']}>
                <span className={styles['datasets__hero-badge-label']}>Eval Type</span>
                <span className={styles['datasets__eval-badge']}>{d.eval_type}</span>
              </span>
            )}
            <button
              type="button"
              className={styles['datasets__preview-btn']}
              onClick={onPreviewClick}
              title="Preview questions"
            >
              <Eye size={14} /> Preview
            </button>
            {deletable && (
              <button
                type="button"
                className={styles['datasets__delete-btn']}
                onClick={onDeleteClick}
                title="Delete this dataset"
              >
                <Trash2 size={14} />
              </button>
            )}
          </div>
        </div>
        <p className={styles['datasets__hero-desc']}>{d.description || 'No description provided.'}</p>
      </div>

      <div className={styles['datasets__stats']}>
        <Stat label="Questions" value={questionCount.toLocaleString()} />
        <Stat label="Categories" value={tags.length || '—'} />
        <Stat label="Eval Type" value={evalType} mono />
        <Stat label="Source" value={datasetType} mono />
      </div>

      <div className={styles['datasets__section']}>
        <div className={styles['datasets__section-head']}>
          <h3>Categories {tags.length > 0 && <em>{tags.length}</em>}</h3>
          {tags.length > 0 && (
            <span className={styles['datasets__section-hint']}>
              click a category to see other datasets with it <ArrowRight size={11} />
            </span>
          )}
        </div>

        {tags.length === 0 ? (
          <div className={styles['datasets__single']}>
            <Boxes size={15} />
            <div>
              <strong>No categories tagged.</strong>
              <span>This dataset isn't broken down into subject areas.</span>
            </div>
          </div>
        ) : (
          <div className={styles['datasets__caps']}>
            {tags.filter(Boolean).map((t) => {
              const col = hashColor(normalizeTag(t));
              const active = tagFilter.some((x) => normalizeTag(x) === normalizeTag(t));
              return (
                <button
                  key={t}
                  className={`${styles['datasets__cap']} ${active ? styles['datasets__cap--active'] : ''}`}
                  style={active
                    ? { background: col.fg, color: '#fff', borderColor: col.fg }
                    : { background: col.bg, color: col.fg, borderColor: 'transparent' }}
                  onClick={() => toggleTag(t)}
                >
                  {t}{active && <Check size={12} />}
                </button>
              );
            })}
          </div>
        )}
      </div>
    </div>
  );
}

function Stat({ label, value, mono }: { label: string; value: string | number | null | undefined; mono?: boolean }) {
  const display = value === null || value === undefined || value === '' ? '—' : value;
  return (
    <div className={styles['datasets__stat']}>
      <span className={`${styles['datasets__stat-val']} ${mono ? styles['datasets__stat-val--mono'] : ''}`}>
        {display}
      </span>
      <span className={styles['datasets__stat-label']}>{label}</span>
    </div>
  );
}

// ---------------------------------------------------------------------------
// Upload modal
// ---------------------------------------------------------------------------

interface UploadModalProps {
  uploadStatus: 'idle' | 'loading' | 'succeeded' | 'failed';
  uploadError: string | null;
  ragCreateStatus: 'idle' | 'loading' | 'succeeded' | 'failed';
  ragCreateError: string | null;
  onClose: () => void;
  onSubmit: (params: { file: File; name: string; description: string; evalType: EvalType }) => void;
  onSubmitRag: (params: { name: string; description: string }) => void;
}

function getExtension(filename: string): string {
  const idx = filename.lastIndexOf('.');
  return idx >= 0 ? filename.slice(idx + 1).toLowerCase() : '';
}

function UploadModal({ uploadStatus, uploadError, ragCreateStatus, ragCreateError, onClose, onSubmit, onSubmitRag }: UploadModalProps) {
  const [evalType, setEvalType] = useState<EvalType>('model');
  const [name, setName] = useState('');
  const [description, setDescription] = useState('');
  const [file, setFile] = useState<File | null>(null);
  const [localError, setLocalError] = useState<string | null>(null);
  const fileInputRef = useRef<HTMLInputElement>(null);

  const isRag = evalType === 'rag';
  const isLoading = isRag ? ragCreateStatus === 'loading' : uploadStatus === 'loading';
  const ext = file ? getExtension(file.name) : '';
  const isJsonl = ext === 'jsonl';

  const pickFile = (f: File | null) => {
    setLocalError(null);
    if (!f) {
      setFile(null);
      return;
    }
    const fExt = getExtension(f.name);
    if (!SUPPORTED_UPLOAD_EXTENSIONS.includes(fExt)) {
      setFile(null);
      setLocalError(`Unsupported file type ".${fExt || '?'}". Please choose a .json, .jsonl, .arrow, or .parquet file.`);
      return;
    }
    setFile(f);
  };

  const handleSubmit = () => {
    setLocalError(null);
    if (!name.trim()) return setLocalError('Please enter a dataset name.');
    if (!description.trim()) return setLocalError('Please enter a description.');
    if (isRag) {
      // RAG datasets are created empty — files are uploaded afterward in
      // the dedicated RAG studio, not here.
      onSubmitRag({ name: name.trim(), description: description.trim() });
      return;
    }
    if (!file) return setLocalError('Please choose a file to upload.');
    onSubmit({ file, name: name.trim(), description: description.trim(), evalType });
  };

  const displayError = localError || (isRag
    ? (ragCreateStatus === 'failed' ? ragCreateError : null)
    : (uploadStatus === 'failed' ? uploadError : null));

  return (
    <div className={styles['datasets__modal-overlay']}>
      <div
        className={styles['datasets__modal']}
        role="dialog"
        aria-modal="true"
        aria-label="Upload dataset"
      >
        <div className={styles['datasets__modal-hdr']}>
          <div>
            <p className={styles['datasets__modal-eyebrow']}>New test suite</p>
            <h3>Upload dataset</h3>
          </div>
          <button className={styles['datasets__modal-close']} onClick={onClose} disabled={isLoading} aria-label="Close">
            <X size={16} />
          </button>
        </div>

        <div className={styles['datasets__modal-body']}>
          <div className={styles['datasets__field']}>
            <label className={styles['datasets__field-label']}>Eval type</label>
            <div className={styles['datasets__eval-pills']}>
              {EVAL_TYPE_OPTIONS.map((opt) => (
                <button
                  key={opt.value}
                  type="button"
                  className={`${styles['datasets__eval-pill']} ${evalType === opt.value ? styles['datasets__eval-pill--on'] : ''}`}
                  onClick={() => setEvalType(opt.value)}
                  disabled={isLoading}
                >
                  {opt.label}
                </button>
              ))}
            </div>
          </div>

          <div className={styles['datasets__field']}>
            <label className={styles['datasets__field-label']} htmlFor="ds-upload-name">Name</label>
            <input
              id="ds-upload-name"
              className={styles['datasets__field-input']}
              placeholder="e.g. Internal QA Set v2"
              value={name}
              onChange={(e) => setName(e.target.value)}
              disabled={isLoading}
            />
          </div>

          <div className={styles['datasets__field']}>
            <label className={styles['datasets__field-label']} htmlFor="ds-upload-desc">Description</label>
            <textarea
              id="ds-upload-desc"
              className={styles['datasets__field-textarea']}
              placeholder="What does this dataset cover?"
              value={description}
              onChange={(e) => setDescription(e.target.value)}
              disabled={isLoading}
              rows={3}
            />
          </div>

          {isRag ? (
            <div className={styles['datasets__field']}>
              <div className={styles['datasets__field-note']} style={{ marginTop: 0 }}>
                RAG datasets are created empty. Once created, you'll upload source files (PDFs, docs,
                spreadsheets, and more) and start processing in a dedicated workspace.
              </div>
            </div>
          ) : (
            <div className={styles['datasets__field']}>
              <label className={styles['datasets__field-label']}>File</label>
              <button
                type="button"
                className={styles['datasets__dropzone']}
                onClick={() => fileInputRef.current?.click()}
                disabled={isLoading}
              >
                <FileUp size={18} />
                <span className={styles['datasets__dropzone-text']}>
                  {file ? file.name : 'Choose a file to upload'}
                </span>
                <span className={styles['datasets__dropzone-hint']}>
                  .json &nbsp;·&nbsp; .jsonl &nbsp;·&nbsp; .arrow &nbsp;·&nbsp; .parquet
                </span>
              </button>
              <input
                ref={fileInputRef}
                type="file"
                accept={ACCEPT_ATTR}
                className={styles['datasets__file-input']}
                onChange={(e) => pickFile(e.target.files?.[0] ?? null)}
                disabled={isLoading}
              />
              {isJsonl && evalType === 'agent' && (
                <p className={styles['datasets__field-note']}>
                  JSONL uploads for Agent evals are automatically tagged with category “Agents”.
                </p>
              )}
            </div>
          )}

          {displayError && (
            <div className={styles['datasets__modal-error']}>
              <AlertCircle size={14} />
              {displayError}
            </div>
          )}

          {!isRag && uploadStatus === 'succeeded' && (
            <div className={styles['datasets__modal-success']}>
              <CheckCircle2 size={14} />
              Dataset uploaded — refreshing the library…
            </div>
          )}
        </div>

        <div className={styles['datasets__modal-foot']}>
          <button className={styles['datasets__modal-cancel']} onClick={onClose} disabled={isLoading}>
            Cancel
          </button>
          <button className={styles['datasets__modal-submit']} onClick={handleSubmit} disabled={isLoading}>
            {isLoading
              ? <Loader2 size={14} className={styles['datasets__spin']} />
              : (isRag ? <Database size={14} /> : <Upload size={14} />)}
            {isLoading ? (isRag ? 'Creating…' : 'Uploading…') : (isRag ? 'Create RAG dataset' : 'Upload dataset')}
          </button>
        </div>
      </div>
    </div>
  );
}

// ---------------------------------------------------------------------------
// Delete confirmation modal
// ---------------------------------------------------------------------------

interface DeleteConfirmModalProps {
  dataset: TestSuite;
  isDeleting: boolean;
  deleteError: string | null;
  onCancel: () => void;
  onConfirm: () => void;
}

function DeleteConfirmModal({ dataset, isDeleting, deleteError, onCancel, onConfirm }: DeleteConfirmModalProps) {
  const name = dataset.name || 'this dataset';

  return (
    <div className={styles['datasets__modal-overlay']}>
      <div
        className={styles['datasets__modal']}
        role="dialog"
        aria-modal="true"
        aria-label="Delete dataset"
      >
        <div className={styles['datasets__modal-hdr']}>
          <div>
            <p className={`${styles['datasets__modal-eyebrow']} ${styles['datasets__modal-eyebrow--danger']}`}>
              This can't be undone
            </p>
            <h3>Delete dataset</h3>
          </div>
          <button className={styles['datasets__modal-close']} onClick={onCancel} disabled={isDeleting} aria-label="Close">
            <X size={16} />
          </button>
        </div>

        <div className={styles['datasets__modal-body']}>
          <p className={styles['datasets__delete-copy']}>
            Are you sure you want to delete <strong>{name}</strong>? This will permanently remove
            the dataset and it will no longer be available for evaluations.
          </p>

          {deleteError && (
            <div className={styles['datasets__modal-error']}>
              <AlertCircle size={14} />
              {deleteError}
            </div>
          )}
        </div>

        <div className={styles['datasets__modal-foot']}>
          <button className={styles['datasets__modal-cancel']} onClick={onCancel} disabled={isDeleting}>
            Cancel
          </button>
          <button className={styles['datasets__modal-danger']} onClick={onConfirm} disabled={isDeleting}>
            {isDeleting ? <Loader2 size={14} className={styles['datasets__spin']} /> : <Trash2 size={14} />}
            {isDeleting ? 'Deleting…' : 'Delete dataset'}
          </button>
        </div>
      </div>
    </div>
  );
}

// ---------------------------------------------------------------------------
// Question preview drawer — slides in from the right, same pattern as
// History's test-detail/metric-score drawer: a separate fade-only overlay,
// a fixed panel pinned to the right edge that translates in on open, its
// own header + scrollable body, and a pagination footer.
// ---------------------------------------------------------------------------

interface PreviewDrawerProps {
  dataset: TestSuite;
  preview: PreviewState;
  onClose: () => void;
  onChangeOffset: (offset: number) => void;
  onChangeLimit: (limit: number) => void;
  onOpenImage: (imagePath: string) => void;
}

function PreviewDrawer({ dataset, preview, onClose, onChangeOffset, onChangeLimit, onOpenImage }: PreviewDrawerProps) {
  const { questions, total, limit, offset, status, error } = preview;
  const safeLimit = limit > 0 ? limit : 20;
  const totalPages = Math.max(1, Math.ceil(total / safeLimit));
  const currentPage = Math.floor(offset / safeLimit) + 1;
  const rangeStart = total === 0 ? 0 : offset + 1;
  const rangeEnd = Math.min(offset + safeLimit, total);
  const isLoading = status === 'loading';

  return (
    <>
      <div className={styles['datasets__preview-header']}>
        <div>
          <p className={styles['datasets__preview-eyebrow']}>Question preview</p>
          <h3 className={styles['datasets__preview-title']}>{dataset.name || 'Untitled dataset'}</h3>
          <div className={styles['datasets__preview-sub']}>
            {total.toLocaleString()} question{total === 1 ? '' : 's'} total
          </div>
        </div>
        <button type="button" className={styles['datasets__preview-close']} onClick={onClose} title="Close">
          <X size={16} />
        </button>
      </div>

      <div className={styles['datasets__preview-body']}>
        {isLoading && questions.length === 0 && (
          <div className={styles['datasets__preview-loading']}>
            <Loader2 size={16} className={styles['datasets__spin']} /> Loading questions…
          </div>
        )}

        {status === 'failed' && questions.length === 0 && (
          <div className={styles['datasets__preview-loading']}>
            <AlertCircle size={16} /> {error || 'Failed to load questions.'}
          </div>
        )}

        {status === 'succeeded' && questions.length === 0 && (
          <div className={styles['datasets__preview-loading']}>
            <HelpCircle size={16} /> No questions found for this dataset.
          </div>
        )}

        {questions.map((q, i) => {
          const prompt = q?.input?.prompt;
          const answer = q?.expected?.answer;
          const language = q?.input?.language;
          const images = (q?.input?.images ?? []).filter(Boolean) as string[];
          return (
            <div key={q?.id ?? i} className={styles['datasets__preview-card']}>
              <div className={styles['datasets__preview-card-hdr']}>
                <span className={styles['datasets__preview-card-num']}>Q{offset + i + 1}</span>
                <span className={styles['datasets__preview-card-tags']}>
                  {language && <span className={styles['datasets__preview-card-lang']}>{language}</span>}
                  {q?.category && <span className={styles['datasets__preview-card-cat']}>{q.category}</span>}
                </span>
              </div>
              <div className={styles['datasets__preview-field']}>
                <span className={styles['datasets__preview-field-label']}>Prompt</span>
                <div className={styles['datasets__preview-field-text']}>{prompt || '—'}</div>
              </div>

              {images.length > 0 && (
                <div className={styles['datasets__preview-field']}>
                  <span className={styles['datasets__preview-field-label']}>
                    Image{images.length === 1 ? '' : `s (${images.length})`}
                  </span>
                  <div className={styles['datasets__preview-image-list']}>
                    {images.map((imgPath, imgIdx) => (
                      <button
                        key={imgPath || imgIdx}
                        type="button"
                        className={styles['datasets__preview-image-chip']}
                        onClick={() => onOpenImage(imgPath)}
                        title={`Preview ${imageBasename(imgPath)}`}
                      >
                        <ImageIcon size={13} />
                        <span>{imageBasename(imgPath)}</span>
                      </button>
                    ))}
                  </div>
                </div>
              )}

              <div className={styles['datasets__preview-field']}>
                <span className={styles['datasets__preview-field-label']}>Expected answer</span>
                <div
                  className={`${styles['datasets__preview-field-text']} ${answer == null ? styles['datasets__preview-field-text--empty'] : ''}`}
                >
                  {answer ?? 'No expected answer'}
                </div>
              </div>

              {q?.metadata && Object.keys(q.metadata).length > 0 && (
                <div className={styles['datasets__preview-meta']}>
                  {flattenMetadata(q.metadata).map(([label, value]) => (
                    <div
                      key={label}
                      className={`${styles['datasets__preview-meta-item']} ${value.length > 40 ? styles['datasets__preview-meta-item--full'] : ''}`}
                    >
                      <span className={styles['datasets__preview-meta-label']}>{formatMetaLabel(label)}</span>
                      <span className={styles['datasets__preview-meta-val']}>{value}</span>
                    </div>
                  ))}
                </div>
              )}
            </div>
          );
        })}
      </div>

      {total > 0 && (
        <div className={styles['datasets__preview-pagination']}>
          <div className={styles['datasets__preview-pagination-info']}>
            {rangeStart}–{rangeEnd} of {total}
          </div>
          <div className={styles['datasets__preview-pagination-controls']}>
            <select
              className={styles['datasets__preview-size-select']}
              value={safeLimit}
              onChange={(e) => onChangeLimit(Number(e.target.value))}
              disabled={isLoading}
              title="Questions per page"
            >
              {PREVIEW_PAGE_SIZE_OPTIONS.map((n) => (
                <option key={n} value={n}>{n} / page</option>
              ))}
            </select>
            <button
              type="button"
              className={styles['datasets__preview-nav-btn']}
              onClick={() => onChangeOffset(Math.max(0, offset - safeLimit))}
              disabled={offset <= 0 || isLoading}
              title="Previous page"
            >
              <ChevronLeft size={14} />
            </button>
            <span className={styles['datasets__preview-page-label']}>{currentPage} / {totalPages}</span>
            <button
              type="button"
              className={styles['datasets__preview-nav-btn']}
              onClick={() => onChangeOffset(offset + safeLimit)}
              disabled={currentPage >= totalPages || isLoading}
              title="Next page"
            >
              <ChevronRight size={14} />
            </button>
          </div>
        </div>
      )}
    </>
  );
}

// ---------------------------------------------------------------------------
// MinIO image lightbox — mirrors the MinIO console's own object-preview
// experience: a near-black backdrop, the image centered and scaled to fit
// the viewport, a slim header bar with the filename and a download action,
// and the image itself only fetched once this is actually opened (never
// preloaded with the question list).
// ---------------------------------------------------------------------------

interface ImageLightboxProps {
  imagePath: string;
  onClose: () => void;
}

function ImageLightbox({ imagePath, onClose }: ImageLightboxProps) {
  const [blobUrl, setBlobUrl] = useState<string | null>(null);
  const [status, setStatus] = useState<'loading' | 'succeeded' | 'failed'>('loading');
  const [error, setError] = useState<string | null>(null);
  const [reloadKey, setReloadKey] = useState(0);

  useEffect(() => {
    let cancelled = false;
    let objectUrl: string | null = null;

    setStatus('loading');
    setError(null);

    testSuitesApi
      .getImageBlobUrl(imagePath)
      .then((url) => {
        if (cancelled) {
          URL.revokeObjectURL(url);
          return;
        }
        objectUrl = url;
        setBlobUrl(url);
        setStatus('succeeded');
      })
      .catch((err) => {
        if (cancelled) return;
        setError(err instanceof Error ? err.message : 'Failed to load image.');
        setStatus('failed');
      });

    return () => {
      cancelled = true;
      if (objectUrl) URL.revokeObjectURL(objectUrl);
    };
  }, [imagePath, reloadKey]);

  const filename = imageBasename(imagePath);

  return (
    <div className={styles['datasets__lightbox-overlay']} onClick={onClose}>
      <div className={styles['datasets__lightbox']} onClick={(e) => e.stopPropagation()}>
        <div className={styles['datasets__lightbox-hdr']}>
          <div className={styles['datasets__lightbox-hdr-info']}>
            <ImageIcon size={14} />
            <span className={styles['datasets__lightbox-filename']} title={imagePath}>{filename}</span>
          </div>
          <div className={styles['datasets__lightbox-hdr-actions']}>
            {blobUrl && (
              <a
                className={styles['datasets__lightbox-action']}
                href={blobUrl}
                download={filename}
                title="Download image"
              >
                <Download size={15} />
              </a>
            )}
            <button
              type="button"
              className={styles['datasets__lightbox-action']}
              onClick={onClose}
              title="Close"
            >
              <X size={16} />
            </button>
          </div>
        </div>

        <div className={styles['datasets__lightbox-stage']}>
          {status === 'loading' && (
            <div className={styles['datasets__lightbox-state']}>
              <Loader2 size={22} className={styles['datasets__spin']} />
              <span>Loading image…</span>
            </div>
          )}

          {status === 'failed' && (
            <div className={styles['datasets__lightbox-state']}>
              <AlertCircle size={22} />
              <span>{error || "Couldn't load this image."}</span>
              <button
                type="button"
                className={styles['datasets__lightbox-retry']}
                onClick={() => setReloadKey((k) => k + 1)}
              >
                <RotateCw size={13} /> Retry
              </button>
            </div>
          )}

          {status === 'succeeded' && blobUrl && (
            <img className={styles['datasets__lightbox-img']} src={blobUrl} alt={filename} />
          )}
        </div>
      </div>
    </div>
  );
}
