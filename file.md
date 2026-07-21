// ═══════════════════════════════════════════════
// store/sttSlice.ts
// LectureAI · STT Transcription state
//
// Mirrors the shape of uploadSlice but stays separate so the two
// modules don't collide. Re-uses the ServerFile / ServerFilesData
// types from uploadSlice since they describe the same backend shape.
// ═══════════════════════════════════════════════
import { createSlice, createAsyncThunk, type PayloadAction } from '@reduxjs/toolkit';
import type { ServerFile, ServerFilesData } from './uploadSlice';
import * as sttApi from '../services/sttApi';
import type { CaptionLine } from '../services/sttApi';

// ── Date defaults: last 30 days ──────────────────
function toDateStr(d: Date) { return d.toISOString().slice(0, 10); }
function defaultFrom() { const d = new Date(); d.setMonth(d.getMonth() - 1); return toDateStr(d); }
function defaultTo() { return toDateStr(new Date()); }

const emptyData: ServerFilesData = { queued: [], completed: [], pending: [], running: [] };

interface CaptionCacheEntry {
    loadedAt: number;
    lines: CaptionLine[];
    is_vtt: boolean;
}

// ── Background upload task ──
// Each in-flight upload gets a client-generated id (taskId, not the
// server-side file id which we don't know until upload completes).
// Concurrent uploads are supported — each task tracks itself.
export type UploadTaskStatus = 'uploading' | 'done' | 'failed' | 'canceled';

export interface UploadTask {
    taskId: string;            // client-generated; only ever exists in the UI
    fileName: string;
    fileSize: number;
    status: UploadTaskStatus;
    progress: number;          // 0–100; meaningful while uploading
    indeterminate: boolean;    // true when server didn't report Content-Length
    error?: string;
    serverFileId?: number;     // populated on success
    startedAt: number;
    completedAt?: number;
}

interface SttState {
    // ── File list (server) ──
    filesData: ServerFilesData;
    filesLoading: boolean;
    filesError: string | null;

    // ── Date range (controls /stt/files/by-date) ──
    dateFrom: string;
    dateTo: string;

    // ── Models ──
    models: string[];
    modelsLoading: boolean;
    modelsError: string | null;

    // ── Upload tasks (background, possibly concurrent) ──
    // Keyed by client-generated taskId. Persists for the lifetime of the
    // session — completed tasks auto-disappear from the UI after a short
    // delay (handled by the component) but stay in state so a refresh
    // doesn't lose them mid-fade.
    uploadTasks: Record<string, UploadTask>;

    // ── Start-transcription bookkeeping ──
    // Files we've successfully POSTed /stt/process for but haven't yet seen
    // in the server's `running` bucket via /by-date. Treated as "in flight"
    // for the purposes of disabling the Generate buttons. Cleared when
    // /by-date confirms the file moved to running (or anywhere non-pending).
    pendingStartIds: number[];
    // The single file whose Start request is currently being POSTed.
    // Drives the modal's "Starting…" state.
    startingId: number | null;
    startError: string | null;


    // ── Local error flags per file ──
    // Set when /stt/process fails for an upload that already succeeded,
    // or when polling repeatedly errors out. Keyed by fileId.
    failedIds: number[];

    // ── Captions cache ──
    // fileId -> caption lines, loaded on demand from /stt/get_captions.
    captionsCache: Record<number, CaptionCacheEntry>;
    captionsLoadingId: number | null;
    captionsError: string | null;

    // ── Inference (batch process) ──
    inferenceOpen: boolean;           // is the 3rd column visible?
    inferenceRunning: boolean;        // batch is in flight
    inferenceSubmitting: boolean;     // POST /stt/batch_process in flight
    inferenceError: string | null;

    // Files currently being tracked via /stt/by-progress
    inferenceQueued: Array<{ id: number; original_name: string }>;
    inferenceRunningFiles: Array<{ id: number; original_name: string }>;
    inferenceCompleted: Array<{ id: number; original_name: string }>;
    inferencePending: Array<{ id: number; original_name: string }>;

    // Progress map from /stt/progress_list — fileId -> 0..100 | -1
    inferenceProgressMap: Record<number, number>;
}

const initialState: SttState = {
    filesData: emptyData,
    filesLoading: false,
    filesError: null,
    dateFrom: defaultFrom(),
    dateTo: defaultTo(),
    models: [],
    modelsLoading: false,
    modelsError: null,
    uploadTasks: {},
    pendingStartIds: [],
    startingId: null,
    startError: null,
    failedIds: [],
    captionsCache: {},
    captionsLoadingId: null,
    captionsError: null,

    inferenceOpen: false,
    inferenceRunning: false,
    inferenceSubmitting: false,
    inferenceError: null,
    inferenceQueued: [],
    inferenceRunningFiles: [],
    inferenceCompleted: [],
    inferencePending: [],
    inferenceProgressMap: {},
};

// ═══════════════════════════════════════════════
// THUNKS
// ═══════════════════════════════════════════════

// ── Fetch file list by date ──
export const fetchSttFiles = createAsyncThunk<
    ServerFilesData,
    { from?: string; to?: string } | undefined,
    { state: { stt: SttState }; rejectValue: string }
>('stt/fetchFiles', async (arg, { getState, rejectWithValue }) => {
    const s = getState().stt;
    const from = arg?.from ?? s.dateFrom;
    const to = arg?.to ?? s.dateTo;
    try {
        return await sttApi.fetchFilesByDate(from, to);
    } catch (err) {
        const msg = err instanceof Error ? err.message : 'Failed to load files';
        return rejectWithValue(msg);
    }
});

// ── Fetch model list ──
export const fetchSttModels = createAsyncThunk<
    string[],
    void,
    { rejectValue: string }
>('stt/fetchModels', async (_, { rejectWithValue }) => {
    try {
        return await sttApi.fetchAudioModels();
    } catch (err) {
        const msg = err instanceof Error ? err.message : 'Failed to load models';
        return rejectWithValue(msg);
    }
});

// ── Upload a file in the background ──
// Returns immediately-usable taskId via fulfilled.meta so the caller can
// drop a row into the UI before the promise resolves. Concurrent calls
// are fully independent — each owns its taskId and progress.
//
// The thunk dispatches uploadProgress() actions during the request so the
// progress bar updates live. On success we also fire fetchSttFiles() so
// the new file appears in the library buckets.
export const uploadOnly = createAsyncThunk<
    { taskId: string; serverFileId: number },
    { file: File; taskId: string },
    {
        state: { stt: SttState };
        rejectValue: { taskId: string; message: string };
    }
>('stt/upload', async ({ file, taskId }, { dispatch, rejectWithValue }) => {
    try {
        const serverFileId = await sttApi.uploadFile(file, {
            onProgress: (pct) => {
                // Cap at 90% while the bytes are in flight.
                // The remaining 10% jumps to 100 once the server responds (see fulfilled handler).
                dispatch(uploadProgress({ taskId, progress: Math.min(90, pct) }));
            },
        });
        // Kick off a /by-date refresh on success so the new file appears.
        // Clear stale data first (inline since clearSttFiles action creator is defined later in file)
        dispatch({ type: 'stt/clearSttFiles' });
        dispatch(fetchSttFiles());
        return { taskId, serverFileId };
    } catch (err) {
        const message = err instanceof Error ? err.message : 'Upload failed';
        return rejectWithValue({ taskId, message });
    }
});

// ── Start a transcription for an existing file ──
// Used for: first-time generation on a pending file, re-generation on a
// completed file, retry on a failed file. The slice marks the file as
// "pending start" until /by-date confirms it moved to the running bucket;
// this avoids a brief UI flicker where the button re-enables before the
// server has reflected the new status.
export const startTranscription = createAsyncThunk<
    number,
    { fileId: number; modelName: string; language: string; chunk: string },
    { rejectValue: { fileId: number; message: string } }
>('stt/startTranscription', async (
    { fileId, modelName, language, chunk },
    { rejectWithValue },
) => {
    try {
        await sttApi.startProcessing(fileId, modelName, language, chunk);
        return fileId;
    } catch (err) {
        const message = err instanceof Error ? err.message : 'Could not start transcription';
        return rejectWithValue({ fileId, message });
    }
});

// ── Single progress check ──
// Returns 100 to signal "done — caller should refresh the list".
// Errors are swallowed (returns previous value) so transient network blips
// don't tear down the polling loop.
export const fetchSttProgress = createAsyncThunk<
    { fileId: number; progress: number },
    number,
    { rejectValue: string }
>('stt/fetchProgress', async (fileId, { rejectWithValue }) => {
    try {
        const progress = await sttApi.fetchProgress(fileId);
        return { fileId, progress };
    } catch (err) {
        const msg = err instanceof Error ? err.message : 'Progress check failed';
        return rejectWithValue(msg);
    }
});

// ── Load captions for a file (cached) ──
export const fetchSttCaptions = createAsyncThunk<
    { fileId: number; lines: CaptionLine[]; is_vtt: boolean },
    number,
    { state: { stt: SttState }; rejectValue: string }
>('stt/fetchCaptions', async (fileId, { getState, rejectWithValue }) => {
    // In-memory cache hit — skip the call.
    const cached = getState().stt.captionsCache[fileId];
    if (cached) return { fileId, lines: cached.lines, is_vtt: cached.is_vtt };
    try {
        const { lines, is_vtt } = await sttApi.fetchCaptions(fileId);
        return { fileId, lines, is_vtt };
    } catch (err) {
        const msg = err instanceof Error ? err.message : 'Failed to load captions';
        return rejectWithValue(msg);
    }
});

// ── Save a single edited caption to the server ──
// Returns the server-canonicalized text so the caller can reconcile if the
// backend changes anything (whitespace, punctuation, etc.).
// ── Batch process (inference) ──
export const submitBatchInference = createAsyncThunk<
    { fileIds: number[] },
    { fileIds: number[]; modelName: string; language: string; chunkSize: number },
    { rejectValue: string }
>('stt/submitBatchInference', async ({ fileIds, modelName, language, chunkSize }, { rejectWithValue }) => {
    try {
        await sttApi.batchProcess(fileIds, modelName, language, chunkSize);
        return { fileIds };
    } catch (err) {
        const msg = err instanceof Error ? err.message : 'Failed to submit inference';
        return rejectWithValue(msg);
    }
});

// ── Poll /stt/by-progress ──
export const fetchInferenceProgress = createAsyncThunk<
    { queued: Array<{id:number;original_name:string}>; running: Array<{id:number;original_name:string}>; completed: Array<{id:number;original_name:string}>; pending: Array<{id:number;original_name:string}> },
    { from: string; to: string },
    { rejectValue: string }
>('stt/fetchInferenceProgress', async ({ from, to }, { rejectWithValue }) => {
    try {
        return await sttApi.fetchByProgress(from, to);
    } catch (err) {
        const msg = err instanceof Error ? err.message : 'Failed to fetch inference progress';
        return rejectWithValue(msg);
    }
});

// ── Poll /stt/progress_list for running file progress ──
export const fetchInferenceProgressList = createAsyncThunk<
    Record<string, number>,
    number[],
    { rejectValue: string }
>('stt/fetchInferenceProgressList', async (fileIds, { rejectWithValue }) => {
    try {
        return await sttApi.fetchProgressList(fileIds);
    } catch (err) {
        const msg = err instanceof Error ? err.message : 'Failed to fetch progress list';
        return rejectWithValue(msg);
    }
});

export const updateSttCaption = createAsyncThunk<
    { fileId: number; captionId: number; text: string },
    { fileId: number; captionId: number; text: string },
    { rejectValue: { captionId: number; message: string } }
>('stt/updateCaption', async ({ fileId, captionId, text }, { rejectWithValue }) => {
    try {
        const updated = await sttApi.updateCaption(fileId, captionId, text);
        return { fileId, captionId, text: updated.text };
    } catch (err) {
        const message = err instanceof Error ? err.message : 'Could not save caption';
        return rejectWithValue({ captionId, message });
    }
});

// ═══════════════════════════════════════════════
// SLICE
// ═══════════════════════════════════════════════
const sttSlice = createSlice({
    name: 'stt',
    initialState,
    reducers: {
        setSttDateFrom(state, action: PayloadAction<string>) {
            state.dateFrom = action.payload;
        },
        setSttDateTo(state, action: PayloadAction<string>) {
            state.dateTo = action.payload;
        },
        clearSttFiles(state) {
            state.filesData = emptyData;
            state.filesLoading = true;
            state.filesError = null;
        },
        // Clear a local 'failed' flag — used when the user retries.
        clearSttFileFailure(state, action: PayloadAction<number>) {
            state.failedIds = state.failedIds.filter(id => id !== action.payload);
        },
        // Invalidate a captions cache entry (e.g. after the user edits them
        // server-side and wants a fresh fetch).
        invalidateSttCaptions(state, action: PayloadAction<number>) {
            delete state.captionsCache[action.payload];
        },

        // ── Upload-task reducers ──────────────────────
        // `uploadStart` is fired by the UI immediately before dispatching the
        // upload thunk, so the task pill appears with `uploading` status from
        // the very first frame. The thunk doesn't fire its own "pending" case
        // because we'd lose the file name + size that the UI generated.
        uploadStart(state, action: PayloadAction<{
            taskId: string;
            fileName: string;
            fileSize: number;
        }>) {
            const { taskId, fileName, fileSize } = action.payload;
            state.uploadTasks[taskId] = {
                taskId,
                fileName,
                fileSize,
                status: 'uploading',
                progress: 0,
                indeterminate: true,  // until we get the first onProgress event
                startedAt: Date.now(),
            };
        },
        // Streamed during the upload by the thunk.
        uploadProgress(state, action: PayloadAction<{ taskId: string; progress: number }>) {
            const t = state.uploadTasks[action.payload.taskId];
            if (!t) return;
            t.progress = action.payload.progress;
            t.indeterminate = false;
        },
        // Manual dismiss — UI calls this after the auto-fade timer fires on a
        // successful task, or when the user X's a failed task.
        dismissUploadTask(state, action: PayloadAction<string>) {
            delete state.uploadTasks[action.payload];
        },
        // Clear every settled task in one shot (e.g. a "Clear all" button).
        // ── Inference column open/close ──
        openInference(state) {
            state.inferenceOpen = true;
        },
        closeInference(state) {
            state.inferenceOpen = false;
        },
        clearInferenceError(state) {
            state.inferenceError = null;
        },

        clearCompletedUploads(state) {
            for (const id of Object.keys(state.uploadTasks)) {
                const s = state.uploadTasks[id].status;
                if (s === 'done' || s === 'failed' || s === 'canceled') {
                    delete state.uploadTasks[id];
                }
            }
        },
    },
    extraReducers: (builder) => {
        builder
            // ── fetchSttFiles ──
            .addCase(fetchSttFiles.pending, (state) => {
                state.filesLoading = true;
                state.filesError = null;
            })
            .addCase(fetchSttFiles.fulfilled, (state, action) => {
                state.filesLoading = false;
                state.filesData = action.payload;
                state.failedIds = state.failedIds.filter(id => {
                    const liveIds = new Set([
                        ...action.payload.queued.map((f: any) => f.id),
                        ...action.payload.running.map((f: any) => f.id),
                        ...action.payload.pending.map((f: any) => f.id),
                        ...action.payload.completed.map((f: any) => f.id),
                    ]);
                    return liveIds.has(id);
                });

                // Clear our optimistic "pending start" flag for any file that has
                // moved out of the `pending` bucket (i.e. the server picked it up
                // and put it in queued/running/completed). This is the handoff from
                // local-optimistic-state to server-truth.
                //
                // Keep the flag only while the file is still in `pending` (server
                // hasn't acknowledged the start request yet).
                const stillPending = new Set(action.payload.pending.map(f => f.id));
                state.pendingStartIds = state.pendingStartIds.filter(id => stillPending.has(id));
            })
            .addCase(fetchSttFiles.rejected, (state, action) => {
                state.filesLoading = false;
                state.filesError = action.payload ?? 'Failed to load files';
            })

            // ── fetchSttModels ──
            .addCase(fetchSttModels.pending, (state) => {
                state.modelsLoading = true;
                state.modelsError = null;
            })
            .addCase(fetchSttModels.fulfilled, (state, action) => {
                state.modelsLoading = false;
                state.models = action.payload;
            })
            .addCase(fetchSttModels.rejected, (state, action) => {
                state.modelsLoading = false;
                state.modelsError = action.payload ?? 'Failed to load models';
            })

            // ── uploadOnly (background) ──
            // No `pending` handler — the UI dispatches uploadStart() before the
            // thunk runs, since it's the only place that knows the file name.
            .addCase(uploadOnly.fulfilled, (state, action) => {
                const t = state.uploadTasks[action.payload.taskId];
                if (!t) return;
                t.status = 'done';
                t.progress = 100;
                t.indeterminate = false;
                t.serverFileId = action.payload.serverFileId;
                t.completedAt = Date.now();
            })
            .addCase(uploadOnly.rejected, (state, action) => {
                const taskId = action.payload?.taskId;
                if (!taskId) return;
                const t = state.uploadTasks[taskId];
                if (!t) return;
                t.status = 'failed';
                t.error = action.payload?.message ?? 'Upload failed';
                t.completedAt = Date.now();
            })

            // ── startTranscription ──
            .addCase(startTranscription.pending, (state, action) => {
                state.startingId = action.meta.arg.fileId;
                state.startError = null;
            })
            .addCase(startTranscription.fulfilled, (state, action) => {
                state.startingId = null;
                // Mark optimistically as in-flight so the "any running?" check picks
                // this file up even before /by-date reports it.
                const id = action.payload;
                if (!state.pendingStartIds.includes(id)) state.pendingStartIds.push(id);
                // If this file had been marked locally failed, clear that flag.
                state.failedIds = state.failedIds.filter(x => x !== id);
            })
            .addCase(startTranscription.rejected, (state, action) => {
                state.startingId = null;
                state.startError = action.payload?.message ?? 'Could not start transcription';
                // Mark the file as failed locally so the UI can show a retry hint.
                if (action.payload?.fileId !== undefined) {
                    const fid = action.payload.fileId;
                    if (!state.failedIds.includes(fid)) state.failedIds.push(fid);
                }
            })

            // ── fetchSttCaptions ──
            .addCase(fetchSttCaptions.pending, (state, action) => {
                state.captionsLoadingId = action.meta.arg;
                state.captionsError = null;
            })
            .addCase(fetchSttCaptions.fulfilled, (state, action) => {
                state.captionsLoadingId = null;
                state.captionsCache[action.payload.fileId] = {
                    loadedAt: Date.now(),
                    lines: action.payload.lines,
                    is_vtt: action.payload.is_vtt,
                };
            })
            .addCase(fetchSttCaptions.rejected, (state, action) => {
                state.captionsLoadingId = null;
                state.captionsError = action.payload ?? 'Failed to load captions';
            })

            // ── submitBatchInference ──
            .addCase(submitBatchInference.pending, (state) => {
                state.inferenceSubmitting = true;
                state.inferenceError = null;
            })
            .addCase(submitBatchInference.fulfilled, (state, action) => {
                state.inferenceSubmitting = false;
                state.inferenceRunning = true;

                // Optimistically move submitted files from their current bucket → queued
                // so the Library column immediately shows them as "Queued" without waiting
                // for the first /by-progress poll.
                const submittedIds = new Set(action.payload.fileIds);
                const bucketsToSearch: (keyof typeof state.filesData)[] = [
                    'completed', 'pending', 'running',
                ];
                for (const bucket of bucketsToSearch) {
                    const moved = (state.filesData[bucket] as any[]).filter((f: any) => submittedIds.has(f.id));
                    if (moved.length > 0) {
                        (state.filesData[bucket] as any[]) = (state.filesData[bucket] as any[]).filter((f: any) => !submittedIds.has(f.id));
                        // Avoid duplicates in queued
                        const existingQueuedIds = new Set(state.filesData.queued.map(f => f.id));
                        for (const f of moved) {
                            if (!existingQueuedIds.has(f.id)) {
                                state.filesData.queued.push(f);
                                existingQueuedIds.add(f.id);
                            }
                        }
                    }
                }
            })
            .addCase(submitBatchInference.rejected, (state, action) => {
                state.inferenceSubmitting = false;
                state.inferenceError = action.payload ?? 'Submission failed';
            })

            // ── fetchInferenceProgress ──
            .addCase(fetchInferenceProgress.fulfilled, (state, action) => {
                const { queued, running, completed, pending } = action.payload;
                state.inferenceQueued = queued;
                state.inferenceRunningFiles = running;
                state.inferenceCompleted = completed;
                state.inferencePending = pending;

                // Sync filesData buckets with the real-time /by-progress response.
                // Strategy: build a lookup of all known files across all buckets,
                // then re-assign each bucket exactly as the server reports.
                const runningIds   = new Set(running.map(f => f.id));
                const queuedIds    = new Set(queued.map(f => f.id));
                const completedIds = new Set(completed.map(f => f.id));

                // Build a map of id → full file object from all current buckets
                const knownFiles = new Map<number, typeof state.filesData.completed[0]>();
                for (const bucket of ['queued', 'running', 'completed', 'pending'] as const) {
                    for (const f of state.filesData[bucket]) {
                        if (!knownFiles.has(f.id)) knownFiles.set(f.id, f);
                    }
                }

                // Re-assign running bucket: only files the server says are running
                state.filesData.running = running.map(
                    f => knownFiles.get(f.id) ?? (f as unknown as typeof state.filesData.running[0])
                );

                // Re-assign queued bucket: only files the server says are queued
                state.filesData.queued = queued.map(
                    f => knownFiles.get(f.id) ?? (f as unknown as typeof state.filesData.queued[0])
                );

                // Move newly completed files into the completed bucket (preserve existing completed)
                const existingCompletedIds = new Set(state.filesData.completed.map(f => f.id));
                for (const f of completed) {
                    if (!existingCompletedIds.has(f.id)) {
                        state.filesData.completed.push(
                            knownFiles.get(f.id) ?? (f as unknown as typeof state.filesData.completed[0])
                        );
                        existingCompletedIds.add(f.id);
                    }
                }

                // Remove from completed bucket files that moved back to running/queued (edge case)
                state.filesData.completed = state.filesData.completed.filter(
                    f => !runningIds.has(f.id) && !queuedIds.has(f.id)
                );

                // If nothing is queued or running, inference is done
                if (queued.length === 0 && running.length === 0) {
                    state.inferenceRunning = false;
                } else {
                    state.inferenceRunning = true;
                }
            })

            // ── fetchInferenceProgressList ──
            .addCase(fetchInferenceProgressList.fulfilled, (state, action) => {
                const result = action.payload;
                for (const [id, progress] of Object.entries(result)) {
                    state.inferenceProgressMap[Number(id)] = progress;
                }
                // If all are done (100 or -1), mark inference complete
                const vals = Object.values(result);
                if (vals.length > 0 && vals.every(v => v === 100 || v === -1)) {
                    state.inferenceRunning = false;
                }
            })

            // ── updateSttCaption ──
            // Patch the cached line in-place so reopening the file reflects the
            // server-confirmed text without a refetch.
            .addCase(updateSttCaption.fulfilled, (state, action) => {
                const { fileId, captionId, text } = action.payload;
                const entry = state.captionsCache[fileId];
                if (!entry) return;
                const idx = entry.lines.findIndex(l => l.id === captionId);
                if (idx >= 0) {
                    entry.lines[idx] = { ...entry.lines[idx], text };
                }
            });
    },
});

// ── Selectors ────────────────────────────────────
// Flattened, status-tagged view of the current buckets for the UI to render.
// Each file gets a synthetic `status` field derived from which bucket it sat in.
export type SttFileStatus = 'queued' | 'pending' | 'running' | 'completed' | 'failed';
export interface SttFileView extends ServerFile {
    status: SttFileStatus;
    add_to_lecture_pipeline?: 0 | 1;  // 1 = already in lecture pipeline
}

export const selectSttFileViews = (state: { stt: SttState }): SttFileView[] => {
    const { filesData, failedIds } = state.stt;
    const failedSet = new Set(failedIds);
    const flatten = (bucket: ServerFile[], status: SttFileStatus): SttFileView[] =>
        bucket.map(f => ({
            ...f,
            status: failedSet.has(f.id) ? 'failed' : status,
        }));
    return [
        ...flatten(filesData.running, 'running'),
        ...flatten(filesData.queued, 'queued'),
        ...flatten(filesData.pending, 'pending'),
        ...flatten(filesData.completed, 'completed'),
    ];
};

// ── Is any transcription in flight, anywhere? ──
// Drives the "Generate Transcription" button across all cards: while this is
// true, no new transcription can be started. We consider a job in flight if:
//   - the server reports it in `queued` or `running` (the truth-source),
//   - OR we just POSTed /stt/process for it and the server hasn't yet
//     reflected the move out of `pending` (the optimistic flag),
//   - OR the user is mid-click and the POST itself is still pending.
export const selectIsAnyRunning = (state: { stt: SttState }): boolean => {
    const { filesData, pendingStartIds, startingId } = state.stt;
    if (filesData.running.length > 0) return true;
    if (filesData.queued.length > 0) return true;
    if (pendingStartIds.length > 0) return true;
    if (startingId !== null) return true;
    return false;
};

// Sorted list of upload tasks for the UI to render. Newest first.
export const selectUploadTasks = (state: { stt: SttState }): UploadTask[] => {
    const arr = Object.values(state.stt.uploadTasks);
    arr.sort((a, b) => b.startedAt - a.startedAt);
    return arr;
};

export const {
    setSttDateFrom,
    setSttDateTo,
    clearSttFiles,
    clearSttFileFailure,
    invalidateSttCaptions,
    uploadStart,
    uploadProgress,
    dismissUploadTask,
    clearCompletedUploads,
    openInference,
    closeInference,
    clearInferenceError,
} = sttSlice.actions;

// Inference selectors
export const selectInferenceIsOpen    = (s: { stt: ReturnType<typeof sttSlice.getInitialState> }) => s.stt.inferenceOpen;
export const selectInferenceRunning   = (s: { stt: ReturnType<typeof sttSlice.getInitialState> }) => s.stt.inferenceRunning;
export const selectInferenceSubmitting = (s: { stt: ReturnType<typeof sttSlice.getInitialState> }) => s.stt.inferenceSubmitting;

export default sttSlice.reducer;




















// ═══════════════════════════════════════════════
// pages/SttTranscription/components/TranscriptDetail.tsx
//
// Player + synced transcript editor for a single completed file.
// On mount, dispatches fetchSttCaptions(id); when the cache fills,
// renders the editor. Edits save to the server (POST /stt/update_captions),
// debounced per-cue so rapid typing doesn't hammer the endpoint.
// ═══════════════════════════════════════════════
import React, {
    useRef,
    useState,
    useCallback,
    useEffect,
    useMemo,
} from 'react';
import { useAppDispatch, useAppSelector } from '../../../store/hooks';
import { fetchSttCaptions, updateSttCaption, type SttFileView } from '../../../store/sttSlice';
import { fetchMediaBlobUrl } from '../../../services/sttApi';
import CueTimeline from './CueTimeline';
import type { Cue, SaveStatus } from '../utils/types';
import {
    captionsToCues,
    cuesToSrt,
    formatShort,
} from '../utils/srt';
import styles from '../SttTranscription.module.scss';

interface Props {
    file: SttFileView;
    // Controls the Results tab's file-list sidebar — collapsing it gives the
    // media player more room, which is the point of the toggle button below.
    sidebarCollapsed: boolean;
    onToggleSidebar: () => void;
}

// ── Icons ────────────────────────────────────────
const IconMic: React.FC = () => (
    <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <rect x="9" y="2" width="6" height="11" rx="3" />
        <path d="M5 10a7 7 0 0014 0" />
        <path d="M12 21v-4M8 21h8" />
    </svg>
);
const IconDownload: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <path d="M8 3v8M5 8l3 3 3-3" /><path d="M2.5 13.5h11" />
    </svg>
);
const IconReset: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <path d="M2.5 8a5.5 5.5 0 109-4M2.5 3v3.5h3.5" />
    </svg>
);
const IconSidebarToggle: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <rect x="2" y="3" width="12" height="10" rx="1.5" />
        <path d="M6.5 3v10" />
        <path d="M10 6.2L8.4 8l1.6 1.8" />
    </svg>
);

const IconPencil: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <path d="M2.5 13.5l3-.5 7-7-2.5-2.5-7 7-.5 3z" />
        <path d="M9.5 4l2.5 2.5" />
    </svg>
);

const IconCheck: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.8" strokeLinecap="round" strokeLinejoin="round">
        <path d="M3 8.5l3.5 3.5 7-7.5" />
    </svg>
);

const IconX: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
        <path d="M4 4l8 8M12 4l-8 8" />
    </svg>
);

// ── File-type detection from server filename ──
const isVideoName = (n: string) =>
    /\.(mp4|avi|mov|mkv|webm)$/i.test(n);

// WebVTT formatter — mirrors cuesToSrt's shape/ordering but with the
// WEBVTT header, dot-decimal timestamps, and no numeric cue index.
// NOTE: this is a self-contained implementation here rather than added to
// utils/srt.ts, since that file wasn't available to edit directly in this
// change — consider moving it there alongside cuesToSrt for consistency.
const vttTimestamp = (seconds: number): string => {
    const s = Math.max(0, seconds);
    const h = Math.floor(s / 3600);
    const m = Math.floor((s % 3600) / 60);
    const sec = Math.floor(s % 60);
    const ms = Math.round((s - Math.floor(s)) * 1000);
    const pad = (n: number, len = 2) => String(n).padStart(len, '0');
    return `${pad(h)}:${pad(m)}:${pad(sec)}.${pad(ms, 3)}`;
};
const cuesToVtt = (cues: Cue[]): string => {
    const body = cues
        .map(c => `${vttTimestamp(c.start)} --> ${vttTimestamp(c.end)}\n${c.text}`)
        .join('\n\n');
    return `WEBVTT\n\n${body}\n`;
};

const TranscriptDetail: React.FC<Props> = ({ file, sidebarCollapsed, onToggleSidebar }) => {
    const dispatch = useAppDispatch();
    const captionsEntry = useAppSelector(s => s.stt.captionsCache[file.id]);
    const captionsLoadingId = useAppSelector(s => s.stt.captionsLoadingId);
    const captionsError = useAppSelector(s => s.stt.captionsError);

    const [cues, setCues] = useState<Cue[]>([]);
    const [activeIndex, setActiveIndex] = useState<number>(-1);
    const [saveStatus, setSaveStatus] = useState<SaveStatus>('idle');
    const [toastMsg, setToastMsg] = useState<string>('');

    // ── Media (audio/video) loading state ──
    // The media endpoint requires auth, so we fetch the bytes through axios
    // and hand the player an object URL. See fetchMediaBlobUrl for the why.
    const [mediaUrl, setMediaUrl] = useState<string>('');
    const [mediaError, setMediaError] = useState<string>('');
    const [mediaLoading, setMediaLoading] = useState(false);

    const mediaRef = useRef<HTMLVideoElement | null>(null);
    const cueListRef = useRef<HTMLDivElement>(null);
    const toastTimerRef = useRef<ReturnType<typeof setTimeout> | null>(null);
    const userScrolledRef = useRef(false);
    const scrollResetTimerRef = useRef<ReturnType<typeof setTimeout> | null>(null);

    // ── Edit mode ──
    // Only one cue is ever in edit mode at a time. editingIdx is the cue's
    // index in the cues array; editingText holds the textarea's working copy
    // so we can cancel without dispatching anything.
    const [editingIdx, setEditingIdx] = useState<number | null>(null);
    const [editingText, setEditingText] = useState<string>('');
    const [savingIdx, setSavingIdx] = useState<number | null>(null);

    // ── Media time tracking for the scrub bar ──
    const [mediaCurrentTime, setMediaCurrentTime] = useState(0);
    const [mediaDuration, setMediaDuration] = useState(0);

    const video = isVideoName(file.original_name);
    const isLoading = captionsLoadingId === file.id || (!captionsEntry && !captionsError);

    // The /get_captions response includes `is_vtt` alongside `lines`,
    // stored in sttSlice's captionsCache entry.
    const isVtt = !!captionsEntry?.is_vtt;

    // ── Trigger fetch on mount / file change ──
    useEffect(() => {
        if (!captionsEntry) {
            dispatch(fetchSttCaptions(file.id));
        }
        // eslint-disable-next-line react-hooks/exhaustive-deps
    }, [file.id]);

    // Switching files while a cue is mid-edit shouldn't carry that edit
    // over to the new file's transcript — close it out immediately.
    useEffect(() => {
        setEditingIdx(null);
        setEditingText('');
        setSavingIdx(null);
    }, [file.id]);

    // ── Load the media blob ──
    // Runs whenever file.id changes. Aborts an in-flight fetch if the user
    // navigates away mid-download, and revokes the previous object URL so we
    // don't leak blobs.
    useEffect(() => {
        const controller = new AbortController();
        let revokedUrl: string | null = null;

        setMediaLoading(true);
        setMediaError('');
        setMediaUrl('');

        fetchMediaBlobUrl(file.id, controller.signal)
            .then(url => {
                revokedUrl = url;
                setMediaUrl(url);
                setMediaLoading(false);
            })
            .catch(err => {
                // AbortError is expected on unmount/file change — ignore it.
                if (controller.signal.aborted) return;
                const msg = err instanceof Error ? err.message : 'Could not load media';
                setMediaError(msg);
                setMediaLoading(false);
            });

        return () => {
            controller.abort();
            if (revokedUrl) URL.revokeObjectURL(revokedUrl);
        };
    }, [file.id]);

    // ── Toast helper ──
    const flashToast = useCallback((msg: string) => {
        setToastMsg(msg);
        if (toastTimerRef.current) clearTimeout(toastTimerRef.current);
        toastTimerRef.current = setTimeout(() => setToastMsg(''), 1800);
    }, []);

    // ── Edit-mode handlers ──
    // beginEdit/cancelEdit/saveEdit work on the cue index, since that's what
    // the row knows. saveEdit dispatches a single POST per intentional save —
    // no debouncing, no auto-save, no surprise writes.
    const beginEdit = useCallback((idx: number) => {
        const c = cues[idx];
        if (!c) return;
        setEditingIdx(idx);
        setEditingText(c.text);
        setSaveStatus('idle');
    }, [cues]);

    const cancelEdit = useCallback(() => {
        setEditingIdx(null);
        setEditingText('');
    }, []);

    const saveEdit = useCallback(async () => {
        if (editingIdx === null) return;
        const c = cues[editingIdx];
        if (!c || c.captionId === null) {
            cancelEdit();
            return;
        }
        const next = editingText;
        // No-op short-circuit — if nothing changed, just exit edit mode.
        if (next === c.text) {
            cancelEdit();
            return;
        }
        setSavingIdx(editingIdx);
        setSaveStatus('editing');
        const result = await dispatch(updateSttCaption({
            fileId: file.id,
            captionId: c.captionId,
            text: next,
        }));
        setSavingIdx(null);

        if (updateSttCaption.fulfilled.match(result)) {
            // Reflect the server-confirmed text in our local copy.
            const confirmed = result.payload.text;
            setCues(prev => prev.map((cc, i) => (i === editingIdx ? { ...cc, text: confirmed } : cc)));
            setSaveStatus('saved');
            flashToast('Saved');
            setEditingIdx(null);
            setEditingText('');
        } else {
            setSaveStatus('failed');
            flashToast('Save failed');
            // Stay in edit mode so the user can retry or cancel — their text isn't lost.
        }
    }, [editingIdx, editingText, cues, dispatch, file.id, cancelEdit, flashToast]);

    // ── When captions arrive from API, build cues ──
    useEffect(() => {
        if (!captionsEntry) return;
        const parsed = captionsToCues(captionsEntry.lines);
        if (parsed.length === 0) {
            flashToast('No captions available');
            setCues([]);
            return;
        }
        setCues(parsed);
        setActiveIndex(-1);
        setSaveStatus('saved');
    }, [captionsEntry, flashToast]);

    // ── Time tracking ──
    const findActiveAt = useCallback((t: number): number => {
        for (let i = 0; i < cues.length; i++) {
            if (t >= cues[i].start && t < cues[i].end) return i;
        }
        return -1;
    }, [cues]);

    useEffect(() => {
        const m = mediaRef.current;
        if (!m) return;
        const onTime = () => {
            // Two things track the play position:
            //   1. activeIndex (which cue row to highlight)
            //   2. mediaCurrentTime (where the orange playhead dot sits on the scrub bar)
            const idx = findActiveAt(m.currentTime);
            setActiveIndex(prev => (prev === idx ? prev : idx));
            setMediaCurrentTime(m.currentTime);
        };
        const onMeta = () => {
            // Use the media element's duration if known. Fall back to the last cue's
            // end time, which is at least a sensible bound for the scrub bar even
            // when the media file isn't loaded yet (or has no duration metadata).
            if (isFinite(m.duration) && m.duration > 0) {
                setMediaDuration(m.duration);
            }
        };
        m.addEventListener('timeupdate', onTime);
        m.addEventListener('seeked', onTime);
        m.addEventListener('loadedmetadata', onMeta);
        return () => {
            m.removeEventListener('timeupdate', onTime);
            m.removeEventListener('seeked', onTime);
            m.removeEventListener('loadedmetadata', onMeta);
        };
    }, [findActiveAt]);

    // Fallback duration: the latest cue end time. Lets the scrub bar render
    // sensibly even before the media element reports its own duration.
    const scrubDuration = useMemo(() => {
        if (mediaDuration > 0) return mediaDuration;
        if (cues.length === 0) return 0;
        return cues[cues.length - 1].end;
    }, [mediaDuration, cues]);

    // Programmatic seek used by the scrub bar and by clicking a cue row.
    const seekTo = useCallback((seconds: number) => {
        const m = mediaRef.current;
        if (!m) return;
        m.currentTime = Math.max(0, seconds + 0.001);
        setMediaCurrentTime(m.currentTime);
    }, []);

    // ── Auto-scroll active cue ──
    useEffect(() => {
        if (activeIndex < 0 || userScrolledRef.current) return;
        const list = cueListRef.current;
        if (!list) return;
        const el = list.querySelector<HTMLElement>(`[data-cue-idx="${activeIndex}"]`);
        if (el) el.scrollIntoView({ behavior: 'smooth', block: 'center' });
    }, [activeIndex]);

    const onCueListScroll = () => {
        userScrolledRef.current = true;
        if (scrollResetTimerRef.current) clearTimeout(scrollResetTimerRef.current);
        scrollResetTimerRef.current = setTimeout(() => {
            userScrolledRef.current = false;
        }, 3000);
    };

    // ── Click a cue row → seek to its start ──
    const jumpTo = (idx: number) => {
        const cue = cues[idx];
        if (!cue) return;
        seekTo(cue.start);
        setActiveIndex(idx);
        const m = mediaRef.current;
        if (m && m.paused) m.play().catch(() => { /* gesture required */ });
    };

    // ── Download — format follows whatever the captions actually are (VTT
    // or SRT), per the /get_captions response's is_vtt flag ──
    const handleDownload = () => {
        if (cues.length === 0) return;
        const ext = isVtt ? 'vtt' : 'srt';
        const mime = isVtt ? 'text/vtt;charset=utf-8' : 'text/plain;charset=utf-8';
        const out = isVtt ? cuesToVtt(cues) : cuesToSrt(cues);
        const blob = new Blob([out], { type: mime });
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = (file.original_name.replace(/\.[^.]+$/, '') || 'transcript') + '.' + ext;
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
        URL.revokeObjectURL(url);
        flashToast('Downloaded');
    };

    // Status text for the liveBox indicator. With explicit edit mode there's
    // no per-cue dirty state to track — at any moment, either a save is in
    // flight or the displayed text matches the server's.
    const statusText = useMemo(() => {
        switch (saveStatus) {
            case 'editing': return 'Saving…';
            case 'saved': return 'Saved';
            case 'failed': return 'Save failed';
            default: return '—';
        }
    }, [saveStatus]);

    useEffect(() => () => {
        if (toastTimerRef.current) clearTimeout(toastTimerRef.current);
        if (scrollResetTimerRef.current) clearTimeout(scrollResetTimerRef.current);
    }, []);

    // ── Render ──
    return (
        <div className={styles.page}>
            {/* ── Page header ── */}
            <div className={styles.ph}>
                <div className={styles.phRow}>
                    <div className={styles.phTitleWrap}>
                        <button
                            className={`${styles.backBtn} ${sidebarCollapsed ? styles.backBtnFlipped : ''}`}
                            onClick={onToggleSidebar}
                            aria-label={sidebarCollapsed ? 'Expand file list' : 'Collapse file list'}
                            title={sidebarCollapsed ? 'Expand file list' : 'Collapse file list'}
                        >
                            <IconSidebarToggle />
                        </button>
                        <div>
                            <div className={styles.phTitle}>{file.original_name}</div>
                            <div className={styles.phSub}>
                                File #{file.id} · {cues.length} cues
                            </div>
                        </div>
                    </div>
                    <div className={styles.phActs}>
                        <button className={`${styles.btn} ${styles.btnBlue}`} onClick={handleDownload} disabled={cues.length === 0}>
                            <IconDownload /> {isVtt ? 'Download VTT' : 'Download SRT'}
                        </button>
                    </div>
                </div>
            </div>

            {captionsError && (
                <div className={styles.errorBanner}>
                    Could not load captions: {captionsError}
                    <button className={styles.btn} onClick={() => dispatch(fetchSttCaptions(file.id))}>
                        Retry
                    </button>
                </div>
            )}

            <div className={styles.playerBody}>
                {/* ── LEFT: media player ── */}
                <div className={styles.mediaPane}>
                    <div className={styles.mediaFrame}>
                        {video ? (
                            <video
                                ref={mediaRef as React.RefObject<HTMLVideoElement>}
                                src={mediaUrl || undefined}
                                controls
                                className={styles.mediaEl}
                            />
                        ) : (
                            <div className={styles.audioStage}>
                                <div className={styles.audioStageIc}><IconMic /></div>
                                <div className={styles.audioStageName}>{file.original_name}</div>
                                <audio
                                    ref={mediaRef as React.RefObject<HTMLAudioElement>}
                                    src={mediaUrl || undefined}
                                    controls
                                    className={styles.audioEl}
                                />
                            </div>
                        )}

                        {/* Loading overlay — shown while the blob downloads. Doesn't block
                the user from reading the transcript pane on the right. */}
                        {mediaLoading && (
                            <div className={styles.mediaLoading}>
                                <div className={styles.spinner} />
                                <span>Loading media…</span>
                            </div>
                        )}
                        {mediaError && (
                            <div className={styles.mediaError}>
                                <div>Could not load media</div>
                                <div className={styles.mediaErrorMsg}>{mediaError}</div>
                            </div>
                        )}
                    </div>
                </div>

                {/* ── RIGHT: live cue spotlight (top) + transcript editor (bottom) ── */}
                <div className={styles.transcriptPane}>
                    <div className={`${styles.liveBox} ${saveStatus === 'editing' ? styles.liveBoxEditing : ''}`}>
                        <div className={styles.liveBoxHeader}>
                            <span className={styles.liveBoxHeaderTitle}>Now Playing</span>
                            <span className={`${styles.liveBoxStatus} ${saveStatus === 'editing' ? styles.liveBoxStatusEditing :
                                saveStatus === 'saved' ? styles.liveBoxStatusSaved :
                                    saveStatus === 'failed' ? styles.liveBoxStatusFailed : ''
                                }`}>
                                {statusText}
                            </span>
                        </div>
                        <div className={styles.liveBoxBody}>
                            <div className={styles.liveBoxMeta}>
                                <span className={styles.liveBoxInfo}>
                                    {activeIndex >= 0
                                        ? `Cue ${activeIndex + 1} of ${cues.length} · ${formatShort(cues[activeIndex].start)} → ${formatShort(cues[activeIndex].end)}`
                                        : (cues.length > 0 ? `${cues.length} cues · play to begin` : (isLoading ? 'Loading captions…' : 'No transcript'))}
                                </span>
                            </div>
                            <div className={styles.liveBoxText}>
                                {activeIndex >= 0
                                    ? cues[activeIndex].text
                                    : (isLoading ? '' : 'Press play to follow along')}
                            </div>
                        </div>
                    </div>

                    <div className={styles.transcriptScrollPane}>
                        <div className={styles.transcriptHeader}>
                            <div className={styles.transcriptTitle}>
                                Transcript
                                {cues.length > 0 && <span className={styles.jobCount}>{cues.length}</span>}
                            </div>
                            <div className={styles.transcriptHint}>Click a row to jump · click pencil to edit</div>
                        </div>

                        {/* ── Scrub bar ── */}
                        {/* Sits between the header and the rows so it's always visible
                  regardless of how the row list scrolls. */}
                        {cues.length > 0 && scrubDuration > 0 && (
                            <div className={styles.timelineWrap}>
                                <CueTimeline
                                    duration={scrubDuration}
                                    currentTime={mediaCurrentTime}
                                    onSeek={seekTo}
                                />
                            </div>
                        )}


                    <div className={styles.cueList} ref={cueListRef} onScroll={onCueListScroll}>
                        {isLoading && cues.length === 0 && (
                            <div className={styles.cueEmpty}>
                                <div className={styles.spinner} />
                                <span>Loading captions…</span>
                            </div>
                        )}
                        {!isLoading && cues.length === 0 && (
                            <div className={styles.cueEmpty}>
                                <span>No captions available for this file</span>
                            </div>
                        )}
                        {cues.map((c, i) => {
                            const isActive = i === activeIndex;
                            const isEditing = i === editingIdx;
                            const isSavingThis = savingIdx === i;
                            const timeLabel = `${formatShort(c.start)} – ${formatShort(c.end)}`;

                            return (
                                <div
                                    key={i}
                                    data-cue-idx={i}
                                    className={`${styles.cueCard} ${isActive ? styles.cueCardActive : ''} ${isEditing ? styles.cueCardEditing : ''}`}
                                >
                                    {/* ── Time pill row ── */}
                                    <div className={styles.cueCardTimeRow}>
                                        <button
                                            type="button"
                                            className={styles.cueCardTime}
                                            onClick={() => !isEditing && jumpTo(i)}
                                            disabled={isEditing}
                                            title={isEditing ? '' : 'Jump to this cue'}
                                        >
                                            {timeLabel}
                                        </button>
                                        {isActive && !isEditing && (
                                            <span className={styles.cueCardCurrentTag}>· Current</span>
                                        )}
                                        <div className={styles.cueCardSpacer} />
                                        {!isEditing ? (
                                            <button
                                                type="button"
                                                className={styles.cueCardAction}
                                                onClick={() => beginEdit(i)}
                                                title="Edit caption"
                                                aria-label="Edit caption"
                                            >
                                                <IconPencil />
                                            </button>
                                        ) : (
                                            <div className={styles.cueCardEditActions}>
                                                <button
                                                    type="button"
                                                    className={styles.cueCardActionCancel}
                                                    onClick={cancelEdit}
                                                    disabled={isSavingThis}
                                                    title="Cancel"
                                                    aria-label="Cancel edit"
                                                >
                                                    <IconX />
                                                </button>
                                                <button
                                                    type="button"
                                                    className={styles.cueCardActionSave}
                                                    onClick={saveEdit}
                                                    disabled={isSavingThis || editingText === c.text}
                                                    title="Save"
                                                    aria-label="Save edit"
                                                >
                                                    {isSavingThis ? <div className={styles.spinnerSmall} /> : <IconCheck />}
                                                </button>
                                            </div>
                                        )}
                                    </div>

                                    {/* ── Body: text (read) or textarea (edit) ── */}
                                    {!isEditing ? (
                                        <div className={styles.cueCardText} title={c.text}>
                                            {c.text}
                                        </div>
                                    ) : (
                                        <textarea
                                            className={styles.cueCardTextarea}
                                            value={editingText}
                                            onChange={e => setEditingText(e.target.value)}
                                            onKeyDown={e => {
                                                // Cmd/Ctrl+Enter saves, Esc cancels — keyboard niceties
                                                // that match the explicit-mode pattern (Enter inserts a
                                                // newline, as users expect inside a textarea).
                                                if (e.key === 'Escape') {
                                                    e.preventDefault();
                                                    cancelEdit();
                                                } else if (e.key === 'Enter' && (e.metaKey || e.ctrlKey)) {
                                                    e.preventDefault();
                                                    saveEdit();
                                                }
                                            }}
                                            autoFocus
                                            rows={3}
                                        />
                                    )}
                                </div>
                            );
                        })}
                    </div>
                    </div>
                </div>
            </div>

            {toastMsg && <div className={styles.toast}>{toastMsg}</div>}
        </div>
    );
};

export default TranscriptDetail;
