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
import { RAG_UPLOAD_EXTENSIONS, getRagPhase } from '../../api/endpoints/rag';
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

  // Phase driven by current_state from /datasets/status:
  //   upload     — can upload files and start processing
  //   processing — uploads locked, progress only
  //   completed  — read-only view
  const phase = getRagPhase(status?.current_state);
  const canUpload = phase === 'upload';
  const isProcessing = phase === 'processing';
  const isCompleted = phase === 'completed';

  // Live status polling while the studio is open. Stops once the dataset is
  // completed since nothing further can change (Refresh still works manually).
  useEffect(() => {
    if (phase === 'completed') return;
    const interval = setInterval(() => {
      dispatch(fetchRagStatus(datasetId));
    }, STATUS_POLL_MS);
    return () => clearInterval(interval);
  }, [dispatch, datasetId, phase]);

  useEffect(() => {
    const onKey = (e: KeyboardEvent) => {
      if (e.key === 'Escape') onClose();
    };
    window.addEventListener('keydown', onKey);
    return () => window.removeEventListener('keydown', onKey);
  }, [onClose]);

  const totalFromStatus = status?.total ?? null;
  const knownFileCount = totalFromStatus ?? uploadedFiles.length;
  const hasAnyFiles = knownFileCount > 0;
  const canProcess = canUpload && hasAnyFiles && processingStartStatus !== 'loading';

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
              {canUpload && (
                <span className={styles['rag-studio__section-hint']}>
                  pdf · csv · excel · docx · pptx · md · html · txt · json · images
                </span>
              )}
            </div>

            {isCompleted ? (
              <div className={styles['rag-studio__locked-note']}>
                <CheckCircle2 size={15} />
                <span>Processing is complete — this dataset is now read-only.</span>
              </div>
            ) : isProcessing ? (
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

            {!canUpload && progressPct !== null && (
              <div className={styles['rag-studio__progress-track']}>
                <div className={styles['rag-studio__progress-fill']} style={{ width: `${progressPct}%` }} />
              </div>
            )}

            {canUpload && processingStartError && (
              <div className={styles['rag-studio__inline-error']}>
                <AlertCircle size={13} /> {processingStartError}
              </div>
            )}

            {canUpload && (
              <button
                type="button"
                className={styles['rag-studio__process-btn']}
                onClick={handleStartProcessing}
                disabled={!canProcess}
                title={
                  !hasAnyFiles
                    ? 'Upload at least one file first'
                    : 'Start processing the uploaded files'
                }
              >
                {processingStartStatus === 'loading'
                  ? <Loader2 size={14} className={styles['rag-studio__spin']} />
                  : <PlayCircle size={14} />}
                {processingStartStatus === 'loading' ? 'Starting…' : 'Process Files'}
              </button>
            )}
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
















//rag.ts
import { apiClient } from '../client';

// File extensions accepted by the RAG upload step (POST /datasets/upload-rag)
// — source documents for retrieval.
export const RAG_UPLOAD_EXTENSIONS = [
  'pdf', 'csv', 'xlsx', 'xls', 'docx', 'pptx', 'md', 'html', 'htm', 'txt', 'json',
  'png', 'jpg', 'jpeg', 'gif', 'webp',
];

// File extensions accepted for the step-1 questions file — same as the
// LLM/Agent create flow: JSON or JSONL, in any supported question format
// (SQuAD, BoolQ, standard, simple flat).
export const RAG_QUESTIONS_FILE_EXTENSIONS = ['json', 'jsonl'];

// Overall state of a RAG dataset, as reported by GET /datasets/status.
//   upload     — user can still upload files and start processing
//   processing — uploads locked, progress is shown
//   completed  — everything is done; the dataset is read-only
export type RagPhase = 'upload' | 'processing' | 'completed';

// Normalises the backend's current_state. Unknown/missing values fall back
// to 'upload' (the most permissive phase).
export function getRagPhase(raw?: string | null): RagPhase {
  const v = (raw || '').trim().toLowerCase();
  if (v === 'processing') return 'processing';
  if (['completed', 'complete', 'done'].includes(v)) return 'completed';
  return 'upload';
}

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
  current_state: string; // "upload" | "processing" | "completed" — normalise with getRagPhase()
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
  // Step-1 questions file — required, same as the LLM/Agent create flow.
  // JSON or JSONL, in any supported question format (SQuAD, BoolQ,
  // standard, simple flat); the backend is responsible for detecting which.
  questionsFile: File;
}

// ASSUMPTION — the exact request/response shape for this endpoint wasn't
// fully specified ("use same old one"), only that it's reused from the
// existing LLM/Agent create flow and now also takes a questions file.
// Sent as multipart (name/description/eval_type as fields, file as
// "questions_file") since a file is involved; the response is read
// defensively for whichever id field it actually returns (dataset_id or
// id) since that wasn't specified either.
interface CreateDatasetResponse {
  dataset_id?: string;
  id?: string;
  [key: string]: unknown;
}

export const ragApi = {
  // POST /datasets/create — same creation call used for the LLM/Agent flow,
  // eval_type fixed to "rag", plus the step-1 questions file.
  create: async ({ name, description, questionsFile }: CreateDatasetParams): Promise<string> => {
    const formData = new FormData();
    formData.append('name', name);
    formData.append('description', description);
    formData.append('eval_type', 'rag');
    formData.append('questions_file', questionsFile);

    const { data } = await apiClient.post<CreateDatasetResponse>('/datasets/create', formData);
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

  // GET /datasets/rag_processing?dataset_id=
  startProcessing: async (datasetId: string): Promise<void> => {
    await apiClient.get('/datasets/rag_processing', { params: { dataset_id: datasetId } });
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
