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































//RagPanel.module.scss
@use 'sass:color';
@use '../../styles/_variables' as *;

// ===========================================================================
// RAG upload + processing panel — embedded as the second column of the
// dataset detail view for RAG datasets (replaces the old RagStudio modal).
// Sections (Upload / Processing / Files) are the same cards as before; only
// the modal chrome (overlay, gradient header, footer) is gone.
//
// Reuses the shared "ink" theme tokens from _variables.scss so dark mode
// keeps working without anything hardcoded beyond the amber queued tint.
// ===========================================================================

$queued-amber: #E08600;
$queued-amber-wash: #FDF3E3;

// Base font-size — same em-scaling convention as Datasets.
$rag-panel-base-font: 0.8125rem;

.rag-panel {
  font-size: $rag-panel-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  display: flex;
  flex-direction: column;
  gap: 16px;

  // ---- header ---------------------------------------------------------------
  &__hdr {
    display: flex;
    align-items: center;
    justify-content: space-between;
    flex-wrap: wrap;
    gap: 10px 12px;
  }

  &__hdr-actions {
    display: flex;
    align-items: center;
    gap: 12px;
    margin-left: auto;
    // Room for the rotated stamp so it isn't clipped by the scroll container.
    padding: 4px 4px 4px 0;
  }

  &__hdr-info {
    display: flex;
    align-items: center;
    gap: 10px;
    min-width: 0;

    h2 {
      font-family: $font-display;
      font-size: 1.2308em; // 1rem / 0.8125rem
      font-weight: 700;
      letter-spacing: -0.01em;
      color: $ink;
      margin: 2px 0 0;
    }
  }

  &__hdr-icon {
    flex-shrink: 0;
    width: 32px;
    height: 32px;
    border-radius: 10px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    background: $wash;
    color: $signal;
  }

  &__eyebrow {
    font-family: $font-mono;
    font-size: 0.6923em; // 0.5625rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    color: $ink-3;
    margin: 0;
  }

  &__refresh {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 8px 13px;
    border: 1px solid $line;
    border-radius: 999px;
    background: $card;
    color: $ink-2;
    font-family: $font-body;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $ink-3; color: $ink; }
  }

  // ---- phase marker (header) ---------------------------------------------------
  &__phase {
    position: relative;
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-family: $font-mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 800;
    letter-spacing: 0.14em;
    text-transform: uppercase;
    white-space: nowrap;
    user-select: none;
  }

  // Upload — calm, dashed "action needed" tag in the app's signal color.
  &__phase--upload {
    padding: 6px 12px;
    border: 1.5px dashed $signal;
    border-radius: 999px;
    color: $signal;
    background: $wash;
    animation: rag-panel-phase-in 0.3s ease both;
  }

  // Processing — solid highlighter with a light sweep passing across it.
  &__phase--processing {
    padding: 6px 13px;
    border-radius: 999px;
    color: #fff;
    background: $signal;
    overflow: hidden;
    box-shadow: 0 0 0 3px $wash, 0 4px 12px rgba(43, 43, 245, 0.25);
    animation: rag-panel-phase-in 0.3s ease both;

    &::after {
      content: '';
      position: absolute;
      inset: 0;
      width: 50%;
      background: linear-gradient(100deg, transparent, rgba(255, 255, 255, 0.45), transparent);
      transform: translateX(-120%);
      animation: rag-panel-sweep 1.8s ease-in-out infinite;
      pointer-events: none;
    }
  }

  // Completed — rubber-stamp look: double border, slight tilt, "stamped"
  // onto the page with a quick scale-down on entry.
  &__phase--completed {
    padding: 6px 14px 6px 11px;
    color: $ok;
    background: rgba($ok, 0.06);
    border: 2px solid $ok;
    border-radius: 7px;
    font-size: 0.8462em;
    letter-spacing: 0.18em;
    transform: rotate(-6deg);
    // inner hairline = the second ring of the stamp
    box-shadow: inset 0 0 0 2px $card, inset 0 0 0 3px $ok;
    animation: rag-panel-stamp 0.45s cubic-bezier(0.2, 0.9, 0.3, 1.25) both;
  }

  // ---- sections -------------------------------------------------------------
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

  &__stat--uploaded .rag-panel__stat-val { color: $ink-2; }
  &__stat--queued .rag-panel__stat-val { color: $queued-amber; }
  &__stat--processing .rag-panel__stat-val { color: $signal; }
  &__stat--ready .rag-panel__stat-val { color: $ok; }
  &__stat--failed .rag-panel__stat-val { color: $danger; }

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

  &__spin {
    animation: rag-panel-spin 0.8s linear infinite;
  }
}

@keyframes rag-panel-spin {
  to { transform: rotate(360deg); }
}

@keyframes rag-panel-phase-in {
  from { opacity: 0; transform: translateY(-4px); }
  to { opacity: 1; transform: translateY(0); }
}

@keyframes rag-panel-sweep {
  0% { transform: translateX(-120%); }
  60%, 100% { transform: translateX(320%); }
}

@keyframes rag-panel-stamp {
  0% { opacity: 0; transform: rotate(-16deg) scale(1.9); }
  60% { opacity: 1; transform: rotate(-4deg) scale(0.94); }
  100% { opacity: 1; transform: rotate(-6deg) scale(1); }
}

@media (prefers-reduced-motion: reduce) {
  .rag-panel__spin,
  .rag-panel__phase,
  .rag-panel__phase--processing::after { animation: none; }
  .rag-panel__phase--processing::after { display: none; }
  .rag-panel__dropzone,
  .rag-panel__upload-btn,
  .rag-panel__process-btn,
  .rag-panel__refresh { transition: none; }
}
