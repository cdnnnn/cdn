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
import { useCallback, useEffect, useMemo, useRef, useState } from 'react';
import {
  X, UploadCloud, FilePlus2, Loader2, PlayCircle, CheckCircle2, XCircle,
  Clock, AlertCircle, Database, RotateCw, File as FileIcon,
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import {
  fetchRagUploadedFiles, uploadRagFiles, startRagProcessing, fetchRagStatus, resetRagStudio,
} from '../../store/slices/ragSlice';
import { RAG_UPLOAD_EXTENSIONS, getRagPhase } from '../../api/endpoints/rag';
import type { RagFile, RagPhase } from '../../api/endpoints/rag';
import styles from './RagPanel.module.scss';

const ACCEPT_ATTR = RAG_UPLOAD_EXTENSIONS.map((e) => `.${e}`).join(',');
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

export default function RagPanel({ datasetId, onPhaseChange }: RagPanelProps) {
  const dispatch = useAppDispatch();
  const {
    uploadedFiles: rawUploadedFiles = [],
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
  const uploadedFiles = useMemo(
    () => rawUploadedFiles.filter((f) => f?.dataset_id === datasetId),
    [rawUploadedFiles, datasetId]
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
  useEffect(() => {
    if (!shouldPoll) return;
    const interval = setInterval(() => {
      dispatch(fetchRagStatus(datasetId));
    }, STATUS_POLL_MS);
    return () => clearInterval(interval);
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
  const displayFiles: RagFile[] = status?.files?.length ? status.files : uploadedFiles;

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
        <button type="button" className={styles['rag-panel__refresh']} onClick={refreshAll} title="Refresh status and files">
          <RotateCw size={13} /> Refresh
        </button>
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
                      {stagedFiles.map((s) => (
                        <span key={s.key} className={styles['rag-panel__staged-chip']}>
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
              <div className={styles['rag-panel__file-list']}>
                {displayFiles.map((f) => {
                  const tier = statusTier(f.status);
                  return (
                    <div key={f.id} className={styles['rag-panel__file-row']}>
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
                })}
              </div>
            )}
          </section>
        </>
      )}
    </div>
  );
}
