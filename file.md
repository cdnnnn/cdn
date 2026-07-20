// ═══════════════════════════════════════════════
// pages/SttTranscription/components/UploadInline.tsx
// ═══════════════════════════════════════════════
import React, { useEffect, useRef, useState } from 'react';
import { useAppDispatch, useAppSelector } from '../../../store/hooks';
import {
    uploadOnly,
    uploadStart,
    dismissUploadTask,
    selectUploadTasks,
    type UploadTask,
} from '../../../store/sttSlice';
import { ALLOWED_EXTS, isAllowed, formatBytes } from '../utils/srt';
import styles from '../SttTranscription.module.scss';

const MAX_FILES = 5;

const IconUpload: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <path d="M8 11V3M5 6l3-3 3 3" />
        <path d="M3 13h10" />
    </svg>
);

const IconClose: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <path d="M4 4l8 8M12 4l-8 8" />
    </svg>
);

const IconCheck: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
        <path d="M3 8.5l3.5 3.5 7-7.5" />
    </svg>
);

const IconAlert: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <circle cx="8" cy="8" r="6" />
        <path d="M8 5v3.5M8 11h.01" />
    </svg>
);

const AUTO_DISMISS_MS = 2500;

const UploadInline: React.FC = () => {
    const dispatch = useAppDispatch();
    const tasks = useAppSelector(selectUploadTasks);
    const inputRef = useRef<HTMLInputElement>(null);
    const folderRef = useRef<HTMLInputElement>(null);
    const [isDragOver, setIsDragOver] = useState(false);
    const [pickError, setPickError] = useState<string>('');
    const dismissTimersRef = useRef<Map<string, ReturnType<typeof setTimeout>>>(new Map());

    // Count only active (uploading) tasks — completed/failed don't block new uploads
    const activeCount = tasks.filter(t => t.status === 'uploading').length;
    const atLimit = activeCount >= MAX_FILES;

    // The "N queued / N ignored" notice only makes sense while uploads are
    // still in flight — once every active upload has settled (done/failed),
    // clear it so it doesn't linger on screen.
    useEffect(() => {
        if (activeCount === 0) setPickError('');
    }, [activeCount]);

    useEffect(() => {
        const timers = dismissTimersRef.current;
        for (const t of tasks) {
            if (t.status === 'done' && !timers.has(t.taskId)) {
                const handle = setTimeout(() => {
                    dispatch(dismissUploadTask(t.taskId));
                    timers.delete(t.taskId);
                }, AUTO_DISMISS_MS);
                timers.set(t.taskId, handle);
            }
        }
        const liveIds = new Set(tasks.map(t => t.taskId));
        for (const [id, h] of timers.entries()) {
            if (!liveIds.has(id)) { clearTimeout(h); timers.delete(id); }
        }
    }, [tasks, dispatch]);

    useEffect(() => () => {
        for (const h of dismissTimersRef.current.values()) clearTimeout(h);
        dismissTimersRef.current.clear();
    }, []);

    const startUpload = (file: File) => {
        setPickError('');
        const taskId = `task_${Date.now()}_${Math.random().toString(36).slice(2, 8)}`;
        dispatch(uploadStart({ taskId, fileName: file.name, fileSize: file.size }));
        dispatch(uploadOnly({ file, taskId }));
    };

    const handlePick = (files: FileList | null) => {
        if (!files || files.length === 0) return;
        const all = Array.from(files).filter(f => isAllowed(f.name));

        // Warn about unsupported formats
        const unsupported = Array.from(files).length - all.length;
        if (unsupported > 0 && all.length === 0) {
            setPickError(`Unsupported format. Accepted: MP3, WAV, AAC, FLAC, M4A, OGG, MP4, AVI, MOV, MKV, WEBM.`);
            return;
        }

        const available = MAX_FILES - activeCount;
        if (available <= 0) {
            setPickError(`All ${MAX_FILES} upload slots are busy. Wait for a file to finish, then try again.`);
            return;
        }

        const toUpload = all.slice(0, available);
        const ignored  = all.length - toUpload.length;

        setPickError('');
        for (const f of toUpload) startUpload(f);

        if (ignored > 0) {
            setPickError(
                `${toUpload.length} file${toUpload.length !== 1 ? 's' : ''} queued. ` +
                `${ignored} file${ignored !== 1 ? 's' : ''} ignored — only ${MAX_FILES} uploads run at once. ` +
                `You can upload more as each one completes.`
            );
        }
    };

    const handleRetry = (task: UploadTask) => {
        dispatch(dismissUploadTask(task.taskId));
        inputRef.current?.click();
    };

    return (
        <div className={styles.uploadCol} data-tour="stt-upload">
            {/* ── Fixed heading ── */}
            <div className={styles.colHeading}>
                <span className={styles.colHeadingIc}><IconUpload /></span>
                <span className={styles.colHeadingText}>Upload Files</span>
                {atLimit && (
                    <span className={styles.uploadLimitBadge}>
                        {activeCount}/{MAX_FILES} uploading
                    </span>
                )}
            </div>

            {/* ── Scrollable content ── */}
            <div className={styles.colBody}>
                {/* Hidden inputs */}
                <input
                    ref={inputRef}
                    type="file"
                    accept={ALLOWED_EXTS.join(',')}
                    multiple
                    style={{ display: 'none' }}
                    onChange={e => { handlePick(e.target.files); e.target.value = ''; }}
                />
                <input
                    ref={folderRef}
                    type="file"
                    // @ts-ignore — webkitdirectory is not in React types
                    webkitdirectory=""
                    mozdirectory=""
                    multiple
                    style={{ display: 'none' }}
                    onChange={e => { handlePick(e.target.files); e.target.value = ''; }}
                />

                {/* Drop zone — drag onto this area */}
                <div
                    className={`${styles.uploadInlineDrop} ${isDragOver && !atLimit ? styles.uploadInlineDropActive : ''} ${atLimit ? styles.uploadInlineDropDisabled : ''}`}
                    onDragOver={e => { e.preventDefault(); if (!atLimit) setIsDragOver(true); }}
                    onDragLeave={() => setIsDragOver(false)}
                    onDrop={async e => {
                        e.preventDefault();
                        setIsDragOver(false);
                        if (atLimit) return;

                        // Use FileSystemEntry API to recursively read folder contents
                        const items = Array.from(e.dataTransfer.items ?? []);
                        const files: File[] = [];

                        const readEntry = (entry: FileSystemEntry): Promise<void> => {
                            if (entry.isFile) {
                                return new Promise(resolve => {
                                    (entry as FileSystemFileEntry).file(
                                        f => { files.push(f); resolve(); },
                                        () => resolve()
                                    );
                                });
                            }
                            if (entry.isDirectory) {
                                const reader = (entry as FileSystemDirectoryEntry).createReader();
                                return new Promise(resolve => {
                                    const readAll = () => {
                                        reader.readEntries(async entries => {
                                            if (entries.length === 0) { resolve(); return; }
                                            await Promise.all(entries.map(readEntry));
                                            readAll();
                                        }, () => resolve());
                                    };
                                    readAll();
                                });
                            }
                            return Promise.resolve();
                        };

                        await Promise.all(
                            items
                                .map(item => item.webkitGetAsEntry?.())
                                .filter(Boolean)
                                .map(entry => readEntry(entry as FileSystemEntry))
                        );

                        if (files.length > 0) {
                            // Convert array to a FileList-like iterable handlePick accepts
                            const dt = new DataTransfer();
                            for (const f of files) dt.items.add(f);
                            handlePick(dt.files);
                        } else {
                            // Fallback for browsers without FileSystemEntry support
                            handlePick(e.dataTransfer.files);
                        }
                    }}
                >
                    <span className={styles.uploadInlineDropIc}><IconUpload /></span>
                    <div className={styles.uploadInlineDropMain}>
                        <div className={styles.uploadInlineDropText}>
                            {atLimit ? 'Upload limit reached' : isDragOver ? 'Drop to upload' : 'Drag files or a folder here'}
                        </div>
                        <div className={styles.uploadInlineDropHint}>
                            {atLimit ? `Max ${MAX_FILES} files at a time` : `MP3 · WAV · AAC · MP4 · MOV · max ${MAX_FILES} files`}
                        </div>
                    </div>

                    {/* Browse buttons row */}
                    {!atLimit && (
                        <div className={styles.uploadBrowseRow}>
                            <button
                                type="button"
                                className={styles.uploadBrowseBtn}
                                onClick={e => { e.stopPropagation(); inputRef.current?.click(); }}
                            >
                                Browse Files
                            </button>
                            <span className={styles.uploadBrowseSep}>or</span>
                            <button
                                type="button"
                                className={styles.uploadBrowseBtn}
                                onClick={e => { e.stopPropagation(); folderRef.current?.click(); }}
                            >
                                Browse Folder
                            </button>
                        </div>
                    )}
                </div>

                {pickError && <div className={styles.uploadError}>{pickError}</div>}

                {/* Task list */}
                {tasks.length > 0 && (
                    <div className={styles.uploadTaskList}>
                        {tasks.map(t => (
                            <UploadTaskRow
                                key={t.taskId}
                                task={t}
                                onDismiss={() => dispatch(dismissUploadTask(t.taskId))}
                                onRetry={() => handleRetry(t)}
                            />
                        ))}
                    </div>
                )}
            </div>
        </div>
    );
};

interface RowProps { task: UploadTask; onDismiss: () => void; onRetry: () => void; }

const UploadTaskRow: React.FC<RowProps> = ({ task, onDismiss, onRetry }) => {
    const isUploading = task.status === 'uploading';
    const isDone      = task.status === 'done';
    const isFailed    = task.status === 'failed';

    let stateClass = styles.uploadTaskUploading;
    if (isDone)   stateClass = styles.uploadTaskDone;
    if (isFailed) stateClass = styles.uploadTaskFailed;

    return (
        <div className={`${styles.uploadTask} ${stateClass}`}>
            <span className={styles.uploadTaskIc}>
                {isDone      && <IconCheck />}
                {isFailed    && <IconAlert />}
                {isUploading && <div className={styles.spinnerSmallDark} />}
            </span>
            <div className={styles.uploadTaskInfo}>
                <div className={styles.uploadTaskName} title={task.fileName}>{task.fileName}</div>
                <div className={styles.uploadTaskMeta}>
                    {formatBytes(task.fileSize)}
                    {isUploading && <><span className={styles.cardMetaSep}>·</span><span>{task.indeterminate ? 'Uploading…' : `${task.progress}%`}</span></>}
                    {isDone      && <><span className={styles.cardMetaSep}>·</span><span className={styles.uploadTaskDoneText}>Uploaded</span></>}
                    {isFailed    && <><span className={styles.cardMetaSep}>·</span><span className={styles.uploadTaskFailText}>{task.error ?? 'Failed'}</span></>}
                </div>
                {isUploading && (
                    <div className={styles.uploadTaskProgress}>
                        {task.indeterminate
                            ? <div className={styles.uploadTaskProgressIndeterminate} />
                            : <div className={styles.uploadTaskProgressFill} style={{ width: `${Math.max(2, task.progress)}%` }} />
                        }
                    </div>
                )}
            </div>
            <div className={styles.uploadTaskActions}>
                {isFailed    && <button type="button" className={styles.uploadTaskRetry} onClick={onRetry}>Retry</button>}
                {!isUploading && <button type="button" className={styles.uploadTaskDismiss} onClick={onDismiss} aria-label="Dismiss"><IconClose /></button>}
            </div>
        </div>
    );
};

export default UploadInline;
