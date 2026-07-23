// ═══════════════════════════════════════════════
// store/videoExplorerSlice.ts
// Content Analytics · Video Explorer
// ═══════════════════════════════════════════════
import { createSlice, createAsyncThunk, type PayloadAction } from '@reduxjs/toolkit';
import axios from '../utils/axiosInstance'; // adjust to your existing axios wrapper

// ─────────────────────────────────────────────
// Server file types (local copies — kept in sync with the
// /stt/files/by-date/ response shape used elsewhere in the app)
// ─────────────────────────────────────────────
export interface ServerFile {
    id: number;
    original_name: string;
    inserted_at: string;
    summary_prompt: string;
    keywords_prompt: string;
    faq_prompt: string;
    progress: string | number | null;
    dictionary_id?: number | null;
    prompt_template_id?: number | null;
}

export interface ServerFilesData {
    queued: ServerFile[];
    completed: ServerFile[];
    pending: ServerFile[];
    running: ServerFile[];
}

interface FilesByDateResponse {
    data: Partial<ServerFilesData>;
}

// Local copy of the fetch call — swap the endpoint below if it differs
// from the one used by the upload page.
async function fetchFilesByDate(startDate: string, endDate: string): Promise<ServerFilesData> {
    const { data } = await axios.post<FilesByDateResponse>('/stt/files/by-date/', {
        start_date: startDate,
        end_date: endDate,
    });
    // Defensive: fill in any missing buckets so consumers can always rely on arrays
    return {
        queued: data.data?.queued ?? [],
        running: data.data?.running ?? [],
        pending: data.data?.pending ?? [],
        completed: data.data?.completed ?? [],
    };
}

// ─────────────────────────────────────────────
// Types
// ─────────────────────────────────────────────
export type MediaKind = 'video' | 'audio';
export type LibraryStatus = 'queued' | 'running' | 'pending' | 'completed';

export interface LibraryItem extends ServerFile {
    status: LibraryStatus;
    mediaKind: MediaKind;
    mediaUrl: string;
}

export interface ChatMessage {
    id: string;
    role: 'user' | 'assistant';
    content: string;
    timestamp_seconds?: number; // optional: playback position the question refers to
    created_at: string;
}

interface VideoExplorerState {
    library: LibraryItem[];
    libraryLoading: boolean;
    libraryError: string | null;
    libraryLoaded: boolean; // guards against re-fetching after the one-time initial load

    selectedFileId: number | null;

    chatByFileId: Record<number, ChatMessage[]>;
    chatLoading: boolean;
    chatError: string | null;
}

const initialState: VideoExplorerState = {
    library: [],
    libraryLoading: false,
    libraryError: null,
    libraryLoaded: false,

    selectedFileId: null,

    chatByFileId: {},
    chatLoading: false,
    chatError: null,
};

// ─────────────────────────────────────────────
// Helpers
// ─────────────────────────────────────────────
const AUDIO_EXT = ['mp3', 'wav', 'm4a', 'aac', 'ogg', 'flac'];

const extOf = (name: string) => name.split('.').pop()?.toLowerCase() ?? '';

const mediaKindOf = (name: string): MediaKind => (AUDIO_EXT.includes(extOf(name)) ? 'audio' : 'video');

// TODO: point this at your real streaming/download endpoint for a file id.
const buildMediaUrl = (fileId: number) => `/api/stt/files/${fileId}/stream`;

const toLibraryItem = (file: ServerFile, status: LibraryStatus): LibraryItem => ({
    ...file,
    status,
    mediaKind: mediaKindOf(file.original_name),
    mediaUrl: buildMediaUrl(file.id),
});

const flattenServerFiles = (data: ServerFilesData): LibraryItem[] => [
    ...data.completed.map((f) => toLibraryItem(f, 'completed')),
    ...data.running.map((f) => toLibraryItem(f, 'running')),
    ...data.queued.map((f) => toLibraryItem(f, 'queued')),
    ...data.pending.map((f) => toLibraryItem(f, 'pending')),
];

const toDateStr = (d: Date) => d.toISOString().slice(0, 10);

// ─────────────────────────────────────────────
// Thunks
// ─────────────────────────────────────────────

// One-time initial load: last 1 year → today. Static range per current requirement.
export const loadLibrary = createAsyncThunk('videoExplorer/loadLibrary', async () => {
    const end = new Date();
    const start = new Date();
    start.setFullYear(start.getFullYear() - 1);

    const data = await fetchFilesByDate(toDateStr(start), toDateStr(end));
    return flattenServerFiles(data);
});

// NOTE: endpoint paths are placeholders — wire these up to your actual backend.
export const fetchChatHistory = createAsyncThunk(
    'videoExplorer/fetchChatHistory',
    async (fileId: number) => {
        const res = await axios.get(`/api/stt/files/${fileId}/chat`);
        return { fileId, messages: res.data as ChatMessage[] };
    },
);

export const sendChatMessage = createAsyncThunk(
    'videoExplorer/sendChatMessage',
    async (payload: { fileId: number; content: string; timestamp_seconds?: number }) => {
        const res = await axios.post(`/api/stt/files/${payload.fileId}/chat`, {
            content: payload.content,
            timestamp_seconds: payload.timestamp_seconds,
        });
        // Expected shape: { userMessage: ChatMessage, assistantMessage: ChatMessage }
        return { fileId: payload.fileId, ...res.data } as {
            fileId: number;
            userMessage: ChatMessage;
            assistantMessage: ChatMessage;
        };
    },
);

// ─────────────────────────────────────────────
// Slice
// ─────────────────────────────────────────────
const videoExplorerSlice = createSlice({
    name: 'videoExplorer',
    initialState,
    reducers: {
        selectFile(state, action: PayloadAction<number | null>) {
            state.selectedFileId = action.payload;
        },
        clearChatError(state) {
            state.chatError = null;
        },
    },
    extraReducers: (builder) => {
        builder
            // library (loaded once)
            .addCase(loadLibrary.pending, (state) => {
                state.libraryLoading = true;
                state.libraryError = null;
            })
            .addCase(loadLibrary.fulfilled, (state, action) => {
                state.libraryLoading = false;
                state.libraryLoaded = true;
                state.library = action.payload;
            })
            .addCase(loadLibrary.rejected, (state, action) => {
                state.libraryLoading = false;
                state.libraryLoaded = true; // don't hammer the endpoint on failure; user can retry explicitly
                state.libraryError = action.error.message ?? 'Could not load files.';
            })
            // chat history
            .addCase(fetchChatHistory.pending, (state) => {
                state.chatLoading = true;
                state.chatError = null;
            })
            .addCase(fetchChatHistory.fulfilled, (state, action) => {
                state.chatLoading = false;
                state.chatByFileId[action.payload.fileId] = action.payload.messages;
            })
            .addCase(fetchChatHistory.rejected, (state, action) => {
                state.chatLoading = false;
                state.chatError = action.error.message ?? 'Could not load chat history.';
            })
            // send message
            .addCase(sendChatMessage.pending, (state) => {
                state.chatLoading = true;
                state.chatError = null;
            })
            .addCase(sendChatMessage.fulfilled, (state, action) => {
                state.chatLoading = false;
                const { fileId, userMessage, assistantMessage } = action.payload;
                const existing = state.chatByFileId[fileId] ?? [];
                state.chatByFileId[fileId] = [...existing, userMessage, assistantMessage];
            })
            .addCase(sendChatMessage.rejected, (state, action) => {
                state.chatLoading = false;
                state.chatError = action.error.message ?? 'Message failed to send.';
            });
    },
});

export const { selectFile, clearChatError } = videoExplorerSlice.actions;
export default videoExplorerSlice.reducer;
