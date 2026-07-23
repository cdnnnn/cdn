// ═══════════════════════════════════════════════
// pages/VideoExplorer/VideoExplorer.tsx
// Content Analytics · Video Explorer
// ═══════════════════════════════════════════════
import React, { useEffect, useRef, useState } from 'react';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import {
    searchVideos,
    selectVideo,
    setQuery,
    fetchChatHistory,
    sendChatMessage,
    clearSearchError,
    clearChatError,
    type VideoResult,
} from '../../store/videoExplorerSlice';
import styles from './VideoExplorer.module.scss';

// ─────────────────────────────────────────────
// Video result card
// ─────────────────────────────────────────────
const VideoCard: React.FC<{ video: VideoResult; active: boolean; onClick: () => void }> = ({
    video,
    active,
    onClick,
}) => (
    <button type="button" className={`${styles.videoCard} ${active ? styles.videoCardActive : ''}`} onClick={onClick}>
        <div className={styles.thumbWrap}>
            <img src={video.thumbnail_url} alt={video.title} className={styles.thumb} />
            <span className={styles.duration}>{video.duration}</span>
        </div>
        <div className={styles.videoMeta}>
            <div className={styles.videoTitle}>{video.title}</div>
            <div className={styles.videoSub}>{video.channel}</div>
        </div>
    </button>
);

// ─────────────────────────────────────────────
// Chat panel
// ─────────────────────────────────────────────
const ChatPanel: React.FC<{ videoId: string }> = ({ videoId }) => {
    const dispatch = useAppDispatch();
    const { chatByVideoId, chatLoading, chatError } = useAppSelector((s) => s.videoExplorer);
    const [draft, setDraft] = useState('');
    const scrollRef = useRef<HTMLDivElement>(null);

    const messages = chatByVideoId[videoId] ?? [];

    useEffect(() => {
        dispatch(fetchChatHistory(videoId));
        dispatch(clearChatError());
    }, [dispatch, videoId]);

    useEffect(() => {
        scrollRef.current?.scrollTo({ top: scrollRef.current.scrollHeight, behavior: 'smooth' });
    }, [messages.length]);

    const handleSend = () => {
        const content = draft.trim();
        if (!content || chatLoading) return;
        setDraft('');
        dispatch(sendChatMessage({ videoId, content }));
    };

    return (
        <div className={styles.chatPanel}>
            <div className={styles.chatHead}>
                <div className={styles.chatHeadTitle}>Ask about this video</div>
                <div className={styles.chatHeadSub}>Answers are grounded in the video's transcript</div>
            </div>

            <div className={styles.chatBody} ref={scrollRef}>
                {chatError && <div className={styles.chatErrorBanner}>{chatError}</div>}

                {messages.length === 0 && !chatLoading ? (
                    <div className={styles.chatEmpty}>
                        Ask a question — e.g. "Summarize the first 5 minutes" or "What does the speaker say about X?"
                    </div>
                ) : (
                    messages.map((m) => (
                        <div key={m.id} className={`${styles.msgRow} ${m.role === 'user' ? styles.msgRowUser : ''}`}>
                            <div className={`${styles.bubble} ${m.role === 'user' ? styles.bubbleUser : styles.bubbleAssistant}`}>
                                {m.content}
                            </div>
                        </div>
                    ))
                )}

                {chatLoading && (
                    <div className={styles.msgRow}>
                        <div className={`${styles.bubble} ${styles.bubbleAssistant} ${styles.bubbleTyping}`}>
                            <span className={styles.typingDot} />
                            <span className={styles.typingDot} />
                            <span className={styles.typingDot} />
                        </div>
                    </div>
                )}
            </div>

            <div className={styles.chatInputRow}>
                <textarea
                    className={styles.chatInput}
                    placeholder="Ask a question about this video…"
                    value={draft}
                    onChange={(e) => setDraft(e.target.value)}
                    onKeyDown={(e) => {
                        if (e.key === 'Enter' && !e.shiftKey) {
                            e.preventDefault();
                            handleSend();
                        }
                    }}
                    rows={2}
                />
                <button
                    type="button"
                    className={styles.sendBtn}
                    onClick={handleSend}
                    disabled={!draft.trim() || chatLoading}
                    aria-label="Send"
                >
                    <svg width="15" height="15" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                        <path d="M14.5 1.5L7.5 8.5M14.5 1.5L10 14.5l-2.5-6-6-2.5z" />
                    </svg>
                </button>
            </div>
        </div>
    );
};

// ═════════════════════════════════════════════
// ROOT: VideoExplorer
// ═════════════════════════════════════════════
const VideoExplorer: React.FC = () => {
    const dispatch = useAppDispatch();
    const { query, results, searching, searchError, selectedVideo } = useAppSelector((s) => s.videoExplorer);
    const [inputValue, setInputValue] = useState(query);

    const handleSearch = (e?: React.FormEvent) => {
        e?.preventDefault();
        const q = inputValue.trim();
        if (!q) return;
        dispatch(setQuery(q));
        dispatch(clearSearchError());
        dispatch(searchVideos(q));
    };

    const handleSelect = (video: VideoResult) => {
        dispatch(selectVideo(video));
    };

    return (
        <div className={styles.page}>

            {/* ── Page header ── */}
            <div className={styles.ph}>
                <div className={styles.phRow}>
                    <div>
                        <div className={styles.phTitle}>Video Explorer</div>
                        <div className={styles.phSub}>search, watch, and ask the assistant about any video</div>
                    </div>
                </div>

                <form className={styles.searchRow} onSubmit={handleSearch}>
                    <svg className={styles.searchIc} width="15" height="15" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                        <circle cx="7" cy="7" r="5" />
                        <path d="M11 11l3.5 3.5" />
                    </svg>
                    <input
                        className={styles.searchInput}
                        placeholder="Search videos…"
                        value={inputValue}
                        onChange={(e) => setInputValue(e.target.value)}
                    />
                    <button type="submit" className={styles.btnPrimary} disabled={searching || !inputValue.trim()}>
                        {searching ? 'Searching…' : 'Search'}
                    </button>
                </form>
            </div>

            {/* ── Body: results + player/chat ── */}
            <div className={styles.body}>

                {/* Results rail */}
                <div className={styles.resultsRail}>
                    {searchError && <div className={styles.errorBanner}>{searchError}</div>}

                    {searching && results.length === 0 && (
                        <div className={styles.resultsSkeleton}>
                            {Array.from({ length: 4 }).map((_, i) => <div key={i} className={styles.skeletonCard} />)}
                        </div>
                    )}

                    {!searching && results.length === 0 && !searchError && (
                        <div className={styles.resultsEmpty}>Search for a topic to find videos.</div>
                    )}

                    <div className={styles.resultsList}>
                        {results.map((v) => (
                            <VideoCard
                                key={v.id}
                                video={v}
                                active={selectedVideo?.id === v.id}
                                onClick={() => handleSelect(v)}
                            />
                        ))}
                    </div>
                </div>

                {/* Player + chat */}
                <div className={styles.watchArea}>
                    {selectedVideo ? (
                        <>
                            <div className={styles.playerWrap}>
                                <div className={styles.playerFrame}>
                                    <iframe
                                        key={selectedVideo.id}
                                        src={`https://www.youtube.com/embed/${selectedVideo.id}`}
                                        title={selectedVideo.title}
                                        allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
                                        allowFullScreen
                                    />
                                </div>
                                <div className={styles.playerTitle}>{selectedVideo.title}</div>
                                <div className={styles.playerSub}>{selectedVideo.channel}</div>
                            </div>
                            <ChatPanel videoId={selectedVideo.id} />
                        </>
                    ) : (
                        <div className={styles.watchEmpty}>
                            <svg width="34" height="34" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.3" strokeLinecap="round" strokeLinejoin="round">
                                <rect x="2" y="4" width="20" height="14" rx="2.5" />
                                <path d="M10 9l5 3-5 3z" />
                            </svg>
                            <div>Select a video to start watching and chatting</div>
                        </div>
                    )}
                </div>
            </div>
        </div>
    );
};

export default VideoExplorer;















// ═══════════════════════════════════════════════
// VideoExplorer.module.scss
// Content Analytics · Video Explorer
// ═══════════════════════════════════════════════
@use '../../styles/mixins' as m;

// ── Page shell ──────────────────────────────────
.page {
    display: flex;
    flex-direction: column;
    height: 100%;
    overflow: hidden;
}

// ── Page header ─────────────────────────────────
.ph {
    padding: 14px 24px 14px;
    background: var(--bg1);
    border-bottom: 1px solid var(--bdr);
    flex-shrink: 0;
    display: flex;
    flex-direction: column;
    gap: 12px;
}

.phRow {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
}

.phTitle {
    font-size: 18px;
    font-weight: 600;
    color: var(--t0);
    letter-spacing: -0.3px;
    font-family: var(--font-display);
}

.phSub {
    font-size: 11px;
    color: var(--t2);
    margin-top: 3px;
    @include m.mono;
}

// ── Search row ───────────────────────────────────
.searchRow {
    display: flex;
    align-items: center;
    gap: 10px;
    max-width: 640px;
}

.searchIc {
    position: relative;
    left: 28px;
    color: var(--t2);
    pointer-events: none;
}

.searchInput {
    flex: 1;
    background: var(--bg3);
    border: 1px solid var(--bdr2);
    border-radius: var(--r);
    color: var(--t0);
    font-family: var(--font-ui);
    font-size: 13px;
    padding: 9px 12px 9px 32px;
    margin-left: -28px;
    outline: none;
    transition: border-color 0.12s, box-shadow 0.12s;

    &::placeholder {
        color: var(--t2);
    }

    &:focus {
        border-color: var(--blue-bdr);
        box-shadow: 0 0 0 2px var(--blue-dim);
    }
}

.btnPrimary {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 8px 16px;
    border-radius: var(--r);
    border: 1px solid var(--blue-bdr);
    background: var(--blue);
    color: #fff;
    font-family: var(--font-ui);
    font-size: 12px;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.12s;
    white-space: nowrap;

    &:hover:not(:disabled) {
        filter: brightness(1.08);
    }

    &:disabled {
        opacity: 0.6;
        cursor: default;
    }
}

// ── Body: results rail + watch area ─────────────
.body {
    flex: 1;
    display: flex;
    overflow: hidden;
}

.resultsRail {
    width: 300px;
    flex-shrink: 0;
    border-right: 1px solid var(--bdr);
    overflow-y: auto;
    padding: 14px;
    display: flex;
    flex-direction: column;
    gap: 10px;
    @include m.scrollbar;
}

.resultsEmpty,
.resultsSkeleton .skeletonCard {
    color: var(--t2);
    font-size: 12px;
}

.resultsEmpty {
    padding: 20px 6px;
    line-height: 1.5;
}

.resultsSkeleton {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.skeletonCard {
    height: 76px;
    border-radius: var(--rl);
    background: linear-gradient(90deg, var(--bg2) 25%, var(--bg3) 37%, var(--bg2) 63%);
    background-size: 400% 100%;
    animation: veShimmer 1.4s ease infinite;
}

.resultsList {
    display: flex;
    flex-direction: column;
    gap: 8px;
}

.errorBanner {
    padding: 9px 12px;
    border-radius: var(--r);
    background: var(--red-dim);
    border: 1px solid var(--red-bdr);
    color: var(--red);
    font-size: 12px;
    font-weight: 500;
}

// ── Video card ───────────────────────────────────
.videoCard {
    display: flex;
    gap: 10px;
    align-items: flex-start;
    padding: 8px;
    border-radius: var(--rl);
    border: 1px solid transparent;
    background: transparent;
    cursor: pointer;
    text-align: left;
    transition: all 0.12s;

    &:hover {
        background: var(--bg2);
    }
}

.videoCardActive {
    background: var(--bg3);
    border-color: var(--blue-bdr);
}

.thumbWrap {
    position: relative;
    width: 96px;
    height: 60px;
    flex-shrink: 0;
    border-radius: 6px;
    overflow: hidden;
    background: var(--bg2);
}

.thumb {
    width: 100%;
    height: 100%;
    object-fit: cover;
    display: block;
}

.duration {
    position: absolute;
    right: 4px;
    bottom: 4px;
    background: rgba(0, 0, 0, 0.75);
    color: #fff;
    font-size: 9.5px;
    font-weight: 600;
    padding: 1px 4px;
    border-radius: 3px;
    @include m.mono;
}

.videoMeta {
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 3px;
}

.videoTitle {
    font-size: 12px;
    font-weight: 600;
    color: var(--t0);
    line-height: 1.35;
    display: -webkit-box;
    -webkit-line-clamp: 2;
    -webkit-box-orient: vertical;
    overflow: hidden;
}

.videoSub {
    font-size: 10.5px;
    color: var(--t2);
    @include m.truncate;
}

// ── Watch area ───────────────────────────────────
.watchArea {
    flex: 1;
    display: flex;
    gap: 16px;
    padding: 16px;
    overflow: hidden;
}

.watchEmpty {
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 10px;
    color: var(--t2);
    font-size: 12.5px;
}

.playerWrap {
    flex: 1.4;
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 6px;
    overflow-y: auto;
    @include m.scrollbar;
}

.playerFrame {
    position: relative;
    width: 100%;
    padding-top: 56.25%; // 16:9
    border-radius: var(--rl);
    overflow: hidden;
    background: #000;
    border: 1px solid var(--bdr2);

    iframe {
        position: absolute;
        inset: 0;
        width: 100%;
        height: 100%;
        border: none;
    }
}

.playerTitle {
    font-size: 14px;
    font-weight: 600;
    color: var(--t0);
    margin-top: 8px;
    line-height: 1.4;
}

.playerSub {
    font-size: 11.5px;
    color: var(--t2);
}

// ── Chat panel ───────────────────────────────────
.chatPanel {
    flex: 1;
    min-width: 300px;
    max-width: 420px;
    display: flex;
    flex-direction: column;
    background: var(--bg1);
    border: 1px solid var(--bdr);
    border-radius: var(--rl);
    overflow: hidden;
}

.chatHead {
    padding: 12px 14px;
    border-bottom: 1px solid var(--bdr);
    flex-shrink: 0;
}

.chatHeadTitle {
    font-size: 13px;
    font-weight: 600;
    color: var(--t0);
}

.chatHeadSub {
    font-size: 10.5px;
    color: var(--t2);
    margin-top: 2px;
    @include m.mono;
}

.chatBody {
    flex: 1;
    overflow-y: auto;
    padding: 12px 14px;
    display: flex;
    flex-direction: column;
    gap: 10px;
    @include m.scrollbar;
}

.chatEmpty {
    color: var(--t2);
    font-size: 12px;
    line-height: 1.6;
    padding: 8px 2px;
}

.chatErrorBanner {
    padding: 8px 10px;
    border-radius: var(--r);
    background: var(--red-dim);
    border: 1px solid var(--red-bdr);
    color: var(--red);
    font-size: 11.5px;
    font-weight: 500;
}

.msgRow {
    display: flex;
    justify-content: flex-start;
}

.msgRowUser {
    justify-content: flex-end;
}

.bubble {
    max-width: 85%;
    padding: 8px 11px;
    border-radius: 11px;
    font-size: 12.5px;
    line-height: 1.5;
    white-space: pre-wrap;
}

.bubbleAssistant {
    background: var(--bg3);
    color: var(--t0);
    border: 1px solid var(--bdr2);
    border-bottom-left-radius: 4px;
}

.bubbleUser {
    background: var(--blue);
    color: #fff;
    border-bottom-right-radius: 4px;
}

.bubbleTyping {
    display: flex;
    align-items: center;
    gap: 4px;
    padding: 10px 12px;
}

.typingDot {
    width: 5px;
    height: 5px;
    border-radius: 50%;
    background: var(--t2);
    animation: veTypingBounce 1s infinite ease-in-out;

    &:nth-child(2) { animation-delay: 0.15s; }
    &:nth-child(3) { animation-delay: 0.3s; }
}

// ── Chat input ───────────────────────────────────
.chatInputRow {
    display: flex;
    align-items: flex-end;
    gap: 8px;
    padding: 10px 12px;
    border-top: 1px solid var(--bdr);
    flex-shrink: 0;
}

.chatInput {
    flex: 1;
    resize: none;
    background: var(--bg3);
    border: 1px solid var(--bdr2);
    border-radius: var(--r);
    color: var(--t0);
    font-family: var(--font-ui);
    font-size: 12.5px;
    padding: 8px 10px;
    outline: none;
    transition: border-color 0.12s, box-shadow 0.12s;

    &::placeholder {
        color: var(--t2);
    }

    &:focus {
        border-color: var(--blue-bdr);
        box-shadow: 0 0 0 2px var(--blue-dim);
    }
}

.sendBtn {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 34px;
    height: 34px;
    flex-shrink: 0;
    border-radius: var(--r);
    border: 1px solid var(--blue-bdr);
    background: var(--blue);
    color: #fff;
    cursor: pointer;
    transition: all 0.12s;

    &:hover:not(:disabled) {
        filter: brightness(1.08);
    }

    &:disabled {
        opacity: 0.5;
        cursor: default;
    }
}

// ── Animations ────────────────────────────────────
@keyframes veShimmer {
    0% { background-position: 100% 0; }
    100% { background-position: 0 0; }
}

@keyframes veTypingBounce {
    0%, 60%, 100% { transform: translateY(0); opacity: 0.5; }
    30% { transform: translateY(-3px); opacity: 1; }
}
















// ═══════════════════════════════════════════════
// store/videoExplorerSlice.ts
// Content Analytics · Video Explorer
// ═══════════════════════════════════════════════
import { createSlice, createAsyncThunk, type PayloadAction } from '@reduxjs/toolkit';
import axios from '../utils/axiosInstance'; // adjust to your existing axios wrapper

// ─────────────────────────────────────────────
// Types
// ─────────────────────────────────────────────
export interface VideoResult {
    id: string;              // youtube video id (or internal id)
    title: string;
    channel: string;
    thumbnail_url: string;
    duration: string;        // "12:34"
    published_at: string;    // ISO date
}

export interface ChatMessage {
    id: string;
    role: 'user' | 'assistant';
    content: string;
    timestamp_seconds?: number; // optional: video timestamp the question refers to
    created_at: string;
}

interface VideoExplorerState {
    query: string;
    results: VideoResult[];
    searching: boolean;
    searchError: string | null;

    selectedVideo: VideoResult | null;

    chatByVideoId: Record<string, ChatMessage[]>;
    chatLoading: boolean;
    chatError: string | null;
}

const initialState: VideoExplorerState = {
    query: '',
    results: [],
    searching: false,
    searchError: null,

    selectedVideo: null,

    chatByVideoId: {},
    chatLoading: false,
    chatError: null,
};

// ─────────────────────────────────────────────
// Thunks
// NOTE: endpoint paths are placeholders — wire these up to your
// actual backend (e.g. a YouTube Data API proxy + LLM chat service).
// ─────────────────────────────────────────────
export const searchVideos = createAsyncThunk(
    'videoExplorer/searchVideos',
    async (query: string) => {
        const res = await axios.get('/api/videos/search', { params: { q: query } });
        return res.data as VideoResult[];
    },
);

export const fetchChatHistory = createAsyncThunk(
    'videoExplorer/fetchChatHistory',
    async (videoId: string) => {
        const res = await axios.get(`/api/videos/${videoId}/chat`);
        return { videoId, messages: res.data as ChatMessage[] };
    },
);

export const sendChatMessage = createAsyncThunk(
    'videoExplorer/sendChatMessage',
    async (payload: { videoId: string; content: string; timestamp_seconds?: number }) => {
        const res = await axios.post(`/api/videos/${payload.videoId}/chat`, {
            content: payload.content,
            timestamp_seconds: payload.timestamp_seconds,
        });
        // Expected shape: { userMessage: ChatMessage, assistantMessage: ChatMessage }
        return { videoId: payload.videoId, ...res.data } as {
            videoId: string;
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
        setQuery(state, action: PayloadAction<string>) {
            state.query = action.payload;
        },
        selectVideo(state, action: PayloadAction<VideoResult | null>) {
            state.selectedVideo = action.payload;
        },
        clearSearchError(state) {
            state.searchError = null;
        },
        clearChatError(state) {
            state.chatError = null;
        },
    },
    extraReducers: (builder) => {
        builder
            // search
            .addCase(searchVideos.pending, (state) => {
                state.searching = true;
                state.searchError = null;
            })
            .addCase(searchVideos.fulfilled, (state, action) => {
                state.searching = false;
                state.results = action.payload;
            })
            .addCase(searchVideos.rejected, (state, action) => {
                state.searching = false;
                state.searchError = action.error.message ?? 'Search failed.';
            })
            // chat history
            .addCase(fetchChatHistory.pending, (state) => {
                state.chatLoading = true;
                state.chatError = null;
            })
            .addCase(fetchChatHistory.fulfilled, (state, action) => {
                state.chatLoading = false;
                state.chatByVideoId[action.payload.videoId] = action.payload.messages;
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
                const { videoId, userMessage, assistantMessage } = action.payload;
                const existing = state.chatByVideoId[videoId] ?? [];
                state.chatByVideoId[videoId] = [...existing, userMessage, assistantMessage];
            })
            .addCase(sendChatMessage.rejected, (state, action) => {
                state.chatLoading = false;
                state.chatError = action.error.message ?? 'Message failed to send.';
            });
    },
});

export const { setQuery, selectVideo, clearSearchError, clearChatError } = videoExplorerSlice.actions;
export default videoExplorerSlice.reducer;












// ═══════════════════════════════════════════════
// components/Sidebar/Sidebar.tsx
// Content Analytics · Sidebar navigation
// ═══════════════════════════════════════════════
import React from 'react';
import { NavLink } from 'react-router-dom';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import { toggleSidebar } from '../../store/uiSlice';
import styles from './Sidebar.module.scss';

interface NavItemProps {
  to: string;
  label: string;
  badge?: string;
  icon: React.ReactNode;
  collapsed: boolean;
}

interface DisabledItemProps {
  label: string;
  icon: React.ReactNode;
  soon?: boolean;
  collapsed: boolean;
}

const NavItem: React.FC<NavItemProps> = ({ to, label, badge, icon, collapsed }) => (
  <NavLink
    to={to}
    title={collapsed ? label : undefined}
    className={({ isActive }) =>
      `${styles.navItem} ${isActive ? styles.active : ''} ${collapsed ? styles.navItemCollapsed : ''}`
    }
  >
    <span className={styles.navIc}>{icon}</span>
    {!collapsed && <span className={styles.navLabel}>{label}</span>}
    {!collapsed && badge && <span className={styles.navBadge}>{badge}</span>}
    {collapsed && badge && <span className={styles.navBadgeDot} />}
  </NavLink>
);

const DisabledItem: React.FC<DisabledItemProps> = ({ label, icon, soon, collapsed }) => (
  <span
    className={`${styles.navItemDisabled} ${collapsed ? styles.navItemCollapsed : ''}`}
    title={collapsed ? label : undefined}
  >
    <span className={styles.navIc}>{icon}</span>
    {!collapsed && <span className={styles.navLabel}>{label}</span>}
    {!collapsed && soon && <span className={styles.soonTag}>Soon</span>}
    {collapsed && soon && <span className={styles.soonDot} />}
  </span>
);

const Sidebar: React.FC = () => {
  const dispatch = useAppDispatch();
  const collapsed = useAppSelector(s => s.ui.sidebarCollapsed);

  return (
    <nav className={`${styles.sidebar} ${collapsed ? styles.sidebarCollapsed : ''}`}>

      {/* ── Toggle button ── */}
      <button
        className={styles.collapseBtn}
        onClick={() => dispatch(toggleSidebar())}
        title={collapsed ? 'Expand sidebar' : 'Collapse sidebar'}
        aria-label={collapsed ? 'Expand sidebar' : 'Collapse sidebar'}
      >
        {!collapsed && <span className={styles.collapseBtnLabel}>Menu</span>}
        <svg
          className={`${styles.collapseBtnIc} ${collapsed ? styles.collapseBtnIcFlipped : ''}`}
          viewBox="0 0 16 16"
          fill="none"
          stroke="currentColor"
          strokeWidth="1.5"
          strokeLinecap="round"
          strokeLinejoin="round"
        >
          <path d="M9.5 4.5L6 8l3.5 3.5" />
          <path d="M6.5 4.5L3 8l3.5 3.5" />
        </svg>
      </button>

      <div className={styles.sidebarNav}>

        {/* ── Main ── */}
        <div className={styles.group}>
          {!collapsed && <div className={styles.groupLabel}>Main</div>}
          <NavItem collapsed={collapsed} to="/upload" label="Lecture Pipeline" icon={
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
              <path d="M8 1.5l1.2 3.8 3.8 1.2-3.8 1.2L8 11.5l-1.2-3.8L3 6.5l3.8-1.2z" />
              <path d="M12.5 10l.7 2 2 .7-2 .7-.7 2-.7-2-2-.7 2-.7z" />
            </svg>
          } />
          <NavItem collapsed={collapsed} to="/history" label="History & Workspace" badge="3" icon={
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
              <rect x="2" y="2" width="12" height="12" rx="2.5" /><path d="M5 8h6M5 5.5h4M5 10.5h6" />
            </svg>
          } />
        </div>

        {/* ── Analysis ── */}
        <div className={styles.group}>
          {!collapsed && <div className={styles.groupLabel}>Analysis</div>}
          <DisabledItem collapsed={collapsed} label="Keyword Insights" soon icon={
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
              <path d="M2 10h3l2-6 2 10 2-6h3" />
            </svg>
          } />
          <DisabledItem collapsed={collapsed} label="Assessment" soon icon={
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
              <circle cx="8" cy="8" r="5.5" />
              <path d="M6.5 6.2C6.7 5.3 7.3 4.8 8 4.8s1.5.6 1.5 1.4C9.5 7.5 8 8 8 9.2" />
              <circle cx="8" cy="11.2" r=".6" fill="currentColor" />
            </svg>
          } />
        </div>

        {/* ── Tools ── */}
        <div className={styles.group}>
          {!collapsed && <div className={styles.groupLabel}>Tools</div>}
          <NavItem collapsed={collapsed} to="/stt" label="STT Transcription" icon={
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
              <rect x="5.5" y="1.5" width="5" height="8" rx="2.5" />
              <path d="M2.5 9.5A5.5 5.5 0 008 15M8 15a5.5 5.5 0 005.5-5.5M8 15v-2.5" />
            </svg>
          } />
          <NavItem collapsed={collapsed} to="/video-explorer" label="Video Explorer" icon={
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
              <rect x="1.5" y="3.5" width="13" height="9" rx="2" />
              <path d="M6.5 6.2v3.6l3-1.8z" fill="currentColor" stroke="none" />
            </svg>
          } />
          <NavItem collapsed={collapsed} to="/prompt-templates" label="Prompt Templates" icon={
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
              <path d="M4 1.5h6l2.5 2.5V13a1 1 0 01-1 1H4a1 1 0 01-1-1V2.5a1 1 0 011-1z" />
              <path d="M10 1.5V4h2.5" />
              <path d="M5.5 7.5h5M5.5 10h3.5" />
            </svg>
          } />
          <NavItem collapsed={collapsed} to="/general-settings" label="General Settings" icon={
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
              <circle cx="8" cy="8" r="2" />
              <path d="M8 1v2M8 13v2M1 8h2M13 8h2M3.2 3.2l1.4 1.4M11.4 11.4l1.4 1.4M11.4 3.2l-1.4 1.4M3.2 11.4l1.4 1.4" />
            </svg>
          } />
          <NavItem collapsed={collapsed} to="/settings" label="Settings" icon={
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
              <rect x="2" y="2" width="12" height="12" rx="2.5" />
              <path d="M5 5.5h6M5 8h6M5 10.5h3.5" />
            </svg>
          } />
          <NavItem collapsed={collapsed} to="/admin" label="Admin Dashboard" icon={
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
              <rect x="2" y="9" width="3" height="5" rx="1" />
              <rect x="6.5" y="6" width="3" height="8" rx="1" />
              <rect x="11" y="3" width="3" height="11" rx="1" />
            </svg>
          } />
        </div>

        {/* ── Voice of Customer ── */}
        <div className={styles.group}>
          {!collapsed && <div className={styles.groupLabel}>Feedback</div>}

          <a href="https://your-voc-form-url.com"
            target="_blank"
            rel="noopener noreferrer"
            title={collapsed ? 'Voice of Customer' : undefined}
            className={`${styles.navItem} ${collapsed ? styles.navItemCollapsed : ''}`}
          >
            <span className={styles.navIc}>
              <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                <path d="M2 3.5h12v7H6.5L3 13.5v-3H2z" />
              </svg>
            </span>
            {!collapsed && <span className={styles.navLabel}>Voice of Customer</span>}
          </a>
        </div>

      </div>
    </nav>
  );
};

export default Sidebar;
















// ═══════════════════════════════════════════════
// store/index.ts — Redux store configuration
// Content Analytics · Lecture Intelligence Platform
// ═══════════════════════════════════════════════
import { configureStore } from '@reduxjs/toolkit';
import authReducer from './authSlice';
import uploadReducer from './uploadSlice';
import historyReducer from './historySlice';
import uiReducer from './uiSlice';
import toastReducer from './toastSlice';
import sttReducer from './sttSlice';
import dictionaryReducer from './dictionarySlice';
import promptTemplateReducer from './promptTemplateSlice';
import settingsReducer from './settingsSlice';
import videoExplorerReducer from './videoExplorerSlice';

export const store = configureStore({
  reducer: {
    auth: authReducer,
    upload: uploadReducer,
    history: historyReducer,
    ui: uiReducer,
    toast: toastReducer,
    stt: sttReducer,
    dictionary: dictionaryReducer,
    promptTemplate: promptTemplateReducer,
    settings: settingsReducer,
    videoExplorer: videoExplorerReducer,
  },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;

















// ═══════════════════════════════════════════════
// pages/VideoExplorer/VideoExplorer.tsx
// Content Analytics · Video Explorer
// ═══════════════════════════════════════════════
import React, { useEffect, useRef, useState } from 'react';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import {
    searchVideos,
    selectVideo,
    setQuery,
    fetchChatHistory,
    sendChatMessage,
    clearSearchError,
    clearChatError,
    type VideoResult,
} from '../../store/videoExplorerSlice';
import styles from './VideoExplorer.module.scss';

// ─────────────────────────────────────────────
// Video result card
// ─────────────────────────────────────────────
const VideoCard: React.FC<{ video: VideoResult; active: boolean; onClick: () => void }> = ({
    video,
    active,
    onClick,
}) => (
    <button type="button" className={`${styles.videoCard} ${active ? styles.videoCardActive : ''}`} onClick={onClick}>
        <div className={styles.thumbWrap}>
            <img src={video.thumbnail_url} alt={video.title} className={styles.thumb} />
            <span className={styles.duration}>{video.duration}</span>
        </div>
        <div className={styles.videoMeta}>
            <div className={styles.videoTitle}>{video.title}</div>
            <div className={styles.videoSub}>{video.owner}</div>
        </div>
    </button>
);

// ─────────────────────────────────────────────
// Chat panel
// ─────────────────────────────────────────────
const formatTime = (secs: number) => {
    const m = Math.floor(secs / 60);
    const s = Math.floor(secs % 60);
    return `${m}:${s.toString().padStart(2, '0')}`;
};

const ChatPanel: React.FC<{ videoId: string; currentTime: number }> = ({ videoId, currentTime }) => {
    const dispatch = useAppDispatch();
    const { chatByVideoId, chatLoading, chatError } = useAppSelector((s) => s.videoExplorer);
    const [draft, setDraft] = useState('');
    const [tagTimestamp, setTagTimestamp] = useState(false);
    const scrollRef = useRef<HTMLDivElement>(null);

    const messages = chatByVideoId[videoId] ?? [];

    useEffect(() => {
        dispatch(fetchChatHistory(videoId));
        dispatch(clearChatError());
    }, [dispatch, videoId]);

    useEffect(() => {
        scrollRef.current?.scrollTo({ top: scrollRef.current.scrollHeight, behavior: 'smooth' });
    }, [messages.length]);

    const handleSend = () => {
        const content = draft.trim();
        if (!content || chatLoading) return;
        setDraft('');
        dispatch(
            sendChatMessage({
                videoId,
                content,
                timestamp_seconds: tagTimestamp ? Math.floor(currentTime) : undefined,
            }),
        );
    };

    return (
        <div className={styles.chatPanel}>
            <div className={styles.chatHead}>
                <div className={styles.chatHeadTitle}>Ask about this video</div>
                <div className={styles.chatHeadSub}>Answers are grounded in the video's transcript</div>
            </div>

            <div className={styles.chatBody} ref={scrollRef}>
                {chatError && <div className={styles.chatErrorBanner}>{chatError}</div>}

                {messages.length === 0 && !chatLoading ? (
                    <div className={styles.chatEmpty}>
                        Ask a question — e.g. "Summarize the first 5 minutes" or "What does the speaker say about X?"
                    </div>
                ) : (
                    messages.map((m) => (
                        <div key={m.id} className={`${styles.msgRow} ${m.role === 'user' ? styles.msgRowUser : ''}`}>
                            <div className={`${styles.bubble} ${m.role === 'user' ? styles.bubbleUser : styles.bubbleAssistant}`}>
                                {typeof m.timestamp_seconds === 'number' && (
                                    <span className={styles.msgTimestamp}>@ {formatTime(m.timestamp_seconds)}</span>
                                )}
                                {m.content}
                            </div>
                        </div>
                    ))
                )}

                {chatLoading && (
                    <div className={styles.msgRow}>
                        <div className={`${styles.bubble} ${styles.bubbleAssistant} ${styles.bubbleTyping}`}>
                            <span className={styles.typingDot} />
                            <span className={styles.typingDot} />
                            <span className={styles.typingDot} />
                        </div>
                    </div>
                )}
            </div>

            <div className={styles.chatToolsRow}>
                <button
                    type="button"
                    className={`${styles.timestampChip} ${tagTimestamp ? styles.timestampChipActive : ''}`}
                    onClick={() => setTagTimestamp((v) => !v)}
                >
                    <svg width="11" height="11" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round" strokeLinejoin="round">
                        <circle cx="8" cy="8" r="6" />
                        <path d="M8 4.5V8l2.5 1.5" />
                    </svg>
                    {tagTimestamp ? `Referencing ${formatTime(currentTime)}` : 'Reference current moment'}
                </button>
            </div>

            <div className={styles.chatInputRow}>
                <textarea
                    className={styles.chatInput}
                    placeholder="Ask a question about this video…"
                    value={draft}
                    onChange={(e) => setDraft(e.target.value)}
                    onKeyDown={(e) => {
                        if (e.key === 'Enter' && !e.shiftKey) {
                            e.preventDefault();
                            handleSend();
                        }
                    }}
                    rows={2}
                />
                <button
                    type="button"
                    className={styles.sendBtn}
                    onClick={handleSend}
                    disabled={!draft.trim() || chatLoading}
                    aria-label="Send"
                >
                    <svg width="15" height="15" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                        <path d="M14.5 1.5L7.5 8.5M14.5 1.5L10 14.5l-2.5-6-6-2.5z" />
                    </svg>
                </button>
            </div>
        </div>
    );
};

// ═════════════════════════════════════════════
// ROOT: VideoExplorer
// ═════════════════════════════════════════════
const VideoExplorer: React.FC = () => {
    const dispatch = useAppDispatch();
    const { query, results, searching, searchError, selectedVideo } = useAppSelector((s) => s.videoExplorer);
    const [inputValue, setInputValue] = useState(query);
    const [currentTime, setCurrentTime] = useState(0);
    const videoRef = useRef<HTMLVideoElement>(null);

    const handleSearch = (e?: React.FormEvent) => {
        e?.preventDefault();
        const q = inputValue.trim();
        if (!q) return;
        dispatch(setQuery(q));
        dispatch(clearSearchError());
        dispatch(searchVideos(q));
    };

    const handleSelect = (video: VideoResult) => {
        dispatch(selectVideo(video));
        setCurrentTime(0);
    };

    return (
        <div className={styles.page}>

            {/* ── Page header ── */}
            <div className={styles.ph}>
                <div className={styles.phRow}>
                    <div>
                        <div className={styles.phTitle}>Video Explorer</div>
                        <div className={styles.phSub}>search, watch, and ask the assistant about any video</div>
                    </div>
                </div>

                <form className={styles.searchRow} onSubmit={handleSearch}>
                    <svg className={styles.searchIc} width="15" height="15" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                        <circle cx="7" cy="7" r="5" />
                        <path d="M11 11l3.5 3.5" />
                    </svg>
                    <input
                        className={styles.searchInput}
                        placeholder="Search videos…"
                        value={inputValue}
                        onChange={(e) => setInputValue(e.target.value)}
                    />
                    <button type="submit" className={styles.btnPrimary} disabled={searching || !inputValue.trim()}>
                        {searching ? 'Searching…' : 'Search'}
                    </button>
                </form>
            </div>

            {/* ── Body: results + player/chat ── */}
            <div className={styles.body}>

                {/* Results rail */}
                <div className={styles.resultsRail}>
                    {searchError && <div className={styles.errorBanner}>{searchError}</div>}

                    {searching && results.length === 0 && (
                        <div className={styles.resultsSkeleton}>
                            {Array.from({ length: 4 }).map((_, i) => <div key={i} className={styles.skeletonCard} />)}
                        </div>
                    )}

                    {!searching && results.length === 0 && !searchError && (
                        <div className={styles.resultsEmpty}>Search for a topic to find videos.</div>
                    )}

                    <div className={styles.resultsList}>
                        {results.map((v) => (
                            <VideoCard
                                key={v.id}
                                video={v}
                                active={selectedVideo?.id === v.id}
                                onClick={() => handleSelect(v)}
                            />
                        ))}
                    </div>
                </div>

                {/* Player + chat */}
                <div className={styles.watchArea}>
                    {selectedVideo ? (
                        <>
                            <div className={styles.playerWrap}>
                                <div className={styles.playerFrame}>
                                    <video
                                        key={selectedVideo.id}
                                        ref={videoRef}
                                        src={selectedVideo.video_url}
                                        poster={selectedVideo.thumbnail_url}
                                        controls
                                        onTimeUpdate={(e) => setCurrentTime(e.currentTarget.currentTime)}
                                    />
                                </div>
                                <div className={styles.playerTitle}>{selectedVideo.title}</div>
                                <div className={styles.playerSub}>{selectedVideo.owner}</div>
                            </div>
                            <ChatPanel videoId={selectedVideo.id} currentTime={currentTime} />
                        </>
                    ) : (
                        <div className={styles.watchEmpty}>
                            <svg width="34" height="34" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.3" strokeLinecap="round" strokeLinejoin="round">
                                <rect x="2" y="4" width="20" height="14" rx="2.5" />
                                <path d="M10 9l5 3-5 3z" />
                            </svg>
                            <div>Select a video to start watching and chatting</div>
                        </div>
                    )}
                </div>
            </div>
        </div>
    );
};

export default VideoExplorer;

















// ═══════════════════════════════════════════════
// VideoExplorer.module.scss
// Content Analytics · Video Explorer
// ═══════════════════════════════════════════════
@use '../../styles/mixins' as m;

// ── Page shell ──────────────────────────────────
.page {
    display: flex;
    flex-direction: column;
    height: 100%;
    overflow: hidden;
}

// ── Page header ─────────────────────────────────
.ph {
    padding: 14px 24px 14px;
    background: var(--bg1);
    border-bottom: 1px solid var(--bdr);
    flex-shrink: 0;
    display: flex;
    flex-direction: column;
    gap: 12px;
}

.phRow {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
}

.phTitle {
    font-size: 18px;
    font-weight: 600;
    color: var(--t0);
    letter-spacing: -0.3px;
    font-family: var(--font-display);
}

.phSub {
    font-size: 11px;
    color: var(--t2);
    margin-top: 3px;
    @include m.mono;
}

// ── Search row ───────────────────────────────────
.searchRow {
    display: flex;
    align-items: center;
    gap: 10px;
    max-width: 640px;
}

.searchIc {
    position: relative;
    left: 28px;
    color: var(--t2);
    pointer-events: none;
}

.searchInput {
    flex: 1;
    background: var(--bg3);
    border: 1px solid var(--bdr2);
    border-radius: var(--r);
    color: var(--t0);
    font-family: var(--font-ui);
    font-size: 13px;
    padding: 9px 12px 9px 32px;
    margin-left: -28px;
    outline: none;
    transition: border-color 0.12s, box-shadow 0.12s;

    &::placeholder {
        color: var(--t2);
    }

    &:focus {
        border-color: var(--blue-bdr);
        box-shadow: 0 0 0 2px var(--blue-dim);
    }
}

.btnPrimary {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 8px 16px;
    border-radius: var(--r);
    border: 1px solid var(--blue-bdr);
    background: var(--blue);
    color: #fff;
    font-family: var(--font-ui);
    font-size: 12px;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.12s;
    white-space: nowrap;

    &:hover:not(:disabled) {
        filter: brightness(1.08);
    }

    &:disabled {
        opacity: 0.6;
        cursor: default;
    }
}

// ── Body: results rail + watch area ─────────────
.body {
    flex: 1;
    display: flex;
    overflow: hidden;
}

.resultsRail {
    width: 300px;
    flex-shrink: 0;
    border-right: 1px solid var(--bdr);
    overflow-y: auto;
    padding: 14px;
    display: flex;
    flex-direction: column;
    gap: 10px;
    @include m.scrollbar;
}

.resultsEmpty,
.resultsSkeleton .skeletonCard {
    color: var(--t2);
    font-size: 12px;
}

.resultsEmpty {
    padding: 20px 6px;
    line-height: 1.5;
}

.resultsSkeleton {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.skeletonCard {
    height: 76px;
    border-radius: var(--rl);
    background: linear-gradient(90deg, var(--bg2) 25%, var(--bg3) 37%, var(--bg2) 63%);
    background-size: 400% 100%;
    animation: veShimmer 1.4s ease infinite;
}

.resultsList {
    display: flex;
    flex-direction: column;
    gap: 8px;
}

.errorBanner {
    padding: 9px 12px;
    border-radius: var(--r);
    background: var(--red-dim);
    border: 1px solid var(--red-bdr);
    color: var(--red);
    font-size: 12px;
    font-weight: 500;
}

// ── Video card ───────────────────────────────────
.videoCard {
    display: flex;
    gap: 10px;
    align-items: flex-start;
    padding: 8px;
    border-radius: var(--rl);
    border: 1px solid transparent;
    background: transparent;
    cursor: pointer;
    text-align: left;
    transition: all 0.12s;

    &:hover {
        background: var(--bg2);
    }
}

.videoCardActive {
    background: var(--bg3);
    border-color: var(--blue-bdr);
}

.thumbWrap {
    position: relative;
    width: 96px;
    height: 60px;
    flex-shrink: 0;
    border-radius: 6px;
    overflow: hidden;
    background: var(--bg2);
}

.thumb {
    width: 100%;
    height: 100%;
    object-fit: cover;
    display: block;
}

.duration {
    position: absolute;
    right: 4px;
    bottom: 4px;
    background: rgba(0, 0, 0, 0.75);
    color: #fff;
    font-size: 9.5px;
    font-weight: 600;
    padding: 1px 4px;
    border-radius: 3px;
    @include m.mono;
}

.videoMeta {
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 3px;
}

.videoTitle {
    font-size: 12px;
    font-weight: 600;
    color: var(--t0);
    line-height: 1.35;
    display: -webkit-box;
    -webkit-line-clamp: 2;
    -webkit-box-orient: vertical;
    overflow: hidden;
}

.videoSub {
    font-size: 10.5px;
    color: var(--t2);
    @include m.truncate;
}

// ── Watch area ───────────────────────────────────
.watchArea {
    flex: 1;
    display: flex;
    gap: 16px;
    padding: 16px;
    overflow: hidden;
}

.watchEmpty {
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 10px;
    color: var(--t2);
    font-size: 12.5px;
}

.playerWrap {
    flex: 1.4;
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 6px;
    overflow-y: auto;
    @include m.scrollbar;
}

.playerFrame {
    position: relative;
    width: 100%;
    padding-top: 56.25%; // 16:9
    border-radius: var(--rl);
    overflow: hidden;
    background: #000;
    border: 1px solid var(--bdr2);

    video {
        position: absolute;
        inset: 0;
        width: 100%;
        height: 100%;
        border: none;
        background: #000;
        object-fit: contain;
    }
}

.playerTitle {
    font-size: 14px;
    font-weight: 600;
    color: var(--t0);
    margin-top: 8px;
    line-height: 1.4;
}

.playerSub {
    font-size: 11.5px;
    color: var(--t2);
}

// ── Chat panel ───────────────────────────────────
.chatPanel {
    flex: 1;
    min-width: 300px;
    max-width: 420px;
    display: flex;
    flex-direction: column;
    background: var(--bg1);
    border: 1px solid var(--bdr);
    border-radius: var(--rl);
    overflow: hidden;
}

.chatHead {
    padding: 12px 14px;
    border-bottom: 1px solid var(--bdr);
    flex-shrink: 0;
}

.chatHeadTitle {
    font-size: 13px;
    font-weight: 600;
    color: var(--t0);
}

.chatHeadSub {
    font-size: 10.5px;
    color: var(--t2);
    margin-top: 2px;
    @include m.mono;
}

.chatBody {
    flex: 1;
    overflow-y: auto;
    padding: 12px 14px;
    display: flex;
    flex-direction: column;
    gap: 10px;
    @include m.scrollbar;
}

.chatEmpty {
    color: var(--t2);
    font-size: 12px;
    line-height: 1.6;
    padding: 8px 2px;
}

.chatErrorBanner {
    padding: 8px 10px;
    border-radius: var(--r);
    background: var(--red-dim);
    border: 1px solid var(--red-bdr);
    color: var(--red);
    font-size: 11.5px;
    font-weight: 500;
}

.msgRow {
    display: flex;
    justify-content: flex-start;
}

.msgRowUser {
    justify-content: flex-end;
}

.bubble {
    max-width: 85%;
    padding: 8px 11px;
    border-radius: 11px;
    font-size: 12.5px;
    line-height: 1.5;
    white-space: pre-wrap;
}

.bubbleAssistant {
    background: var(--bg3);
    color: var(--t0);
    border: 1px solid var(--bdr2);
    border-bottom-left-radius: 4px;
}

.bubbleUser {
    background: var(--blue);
    color: #fff;
    border-bottom-right-radius: 4px;
}

.bubbleTyping {
    display: flex;
    align-items: center;
    gap: 4px;
    padding: 10px 12px;
}

.typingDot {
    width: 5px;
    height: 5px;
    border-radius: 50%;
    background: var(--t2);
    animation: veTypingBounce 1s infinite ease-in-out;

    &:nth-child(2) { animation-delay: 0.15s; }
    &:nth-child(3) { animation-delay: 0.3s; }
}

.msgTimestamp {
    display: inline-block;
    font-size: 10px;
    font-weight: 600;
    opacity: 0.75;
    margin-right: 6px;
    @include m.mono;
}

// ── Chat tools row (timestamp toggle) ────────────
.chatToolsRow {
    display: flex;
    padding: 0 12px 8px;
    flex-shrink: 0;
}

.timestampChip {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 4px 9px;
    border-radius: 99px;
    border: 1px solid var(--bdr2);
    background: transparent;
    color: var(--t2);
    font-family: var(--font-ui);
    font-size: 10.5px;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.12s;

    &:hover {
        border-color: var(--bdr3);
        color: var(--t1);
    }
}

.timestampChipActive {
    background: var(--blue-dim);
    border-color: var(--blue-bdr);
    color: var(--blue);
}

// ── Chat input ───────────────────────────────────
.chatInputRow {
    display: flex;
    align-items: flex-end;
    gap: 8px;
    padding: 10px 12px;
    border-top: 1px solid var(--bdr);
    flex-shrink: 0;
}

.chatInput {
    flex: 1;
    resize: none;
    background: var(--bg3);
    border: 1px solid var(--bdr2);
    border-radius: var(--r);
    color: var(--t0);
    font-family: var(--font-ui);
    font-size: 12.5px;
    padding: 8px 10px;
    outline: none;
    transition: border-color 0.12s, box-shadow 0.12s;

    &::placeholder {
        color: var(--t2);
    }

    &:focus {
        border-color: var(--blue-bdr);
        box-shadow: 0 0 0 2px var(--blue-dim);
    }
}

.sendBtn {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 34px;
    height: 34px;
    flex-shrink: 0;
    border-radius: var(--r);
    border: 1px solid var(--blue-bdr);
    background: var(--blue);
    color: #fff;
    cursor: pointer;
    transition: all 0.12s;

    &:hover:not(:disabled) {
        filter: brightness(1.08);
    }

    &:disabled {
        opacity: 0.5;
        cursor: default;
    }
}

// ── Animations ────────────────────────────────────
@keyframes veShimmer {
    0% { background-position: 100% 0; }
    100% { background-position: 0 0; }
}

@keyframes veTypingBounce {
    0%, 60%, 100% { transform: translateY(0); opacity: 0.5; }
    30% { transform: translateY(-3px); opacity: 1; }
}


























// ═══════════════════════════════════════════════
// store/videoExplorerSlice.ts
// Content Analytics · Video Explorer
// ═══════════════════════════════════════════════
import { createSlice, createAsyncThunk, type PayloadAction } from '@reduxjs/toolkit';
import axios from '../utils/axiosInstance'; // adjust to your existing axios wrapper

// ─────────────────────────────────────────────
// Types
// ─────────────────────────────────────────────
export interface VideoResult {
    id: string;               // internal video/lecture id
    title: string;
    owner: string;            // uploader / instructor / source name
    thumbnail_url: string;
    video_url: string;        // direct/streamable URL to our own hosted file (mp4, HLS, etc.)
    duration: string;         // "12:34"
    created_at: string;       // ISO date
}

export interface ChatMessage {
    id: string;
    role: 'user' | 'assistant';
    content: string;
    timestamp_seconds?: number; // optional: video timestamp the question refers to
    created_at: string;
}

interface VideoExplorerState {
    query: string;
    results: VideoResult[];
    searching: boolean;
    searchError: string | null;

    selectedVideo: VideoResult | null;

    chatByVideoId: Record<string, ChatMessage[]>;
    chatLoading: boolean;
    chatError: string | null;
}

const initialState: VideoExplorerState = {
    query: '',
    results: [],
    searching: false,
    searchError: null,

    selectedVideo: null,

    chatByVideoId: {},
    chatLoading: false,
    chatError: null,
};

// ─────────────────────────────────────────────
// Thunks
// NOTE: endpoint paths are placeholders — wire these up to your
// actual backend. `searchVideos` should query your own video/lecture
// library (e.g. title + transcript full-text search), not an external API.
// ─────────────────────────────────────────────
export const searchVideos = createAsyncThunk(
    'videoExplorer/searchVideos',
    async (query: string) => {
        const res = await axios.get('/api/videos/search', { params: { q: query } });
        return res.data as VideoResult[];
    },
);

export const fetchChatHistory = createAsyncThunk(
    'videoExplorer/fetchChatHistory',
    async (videoId: string) => {
        const res = await axios.get(`/api/videos/${videoId}/chat`);
        return { videoId, messages: res.data as ChatMessage[] };
    },
);

export const sendChatMessage = createAsyncThunk(
    'videoExplorer/sendChatMessage',
    async (payload: { videoId: string; content: string; timestamp_seconds?: number }) => {
        const res = await axios.post(`/api/videos/${payload.videoId}/chat`, {
            content: payload.content,
            timestamp_seconds: payload.timestamp_seconds,
        });
        // Expected shape: { userMessage: ChatMessage, assistantMessage: ChatMessage }
        return { videoId: payload.videoId, ...res.data } as {
            videoId: string;
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
        setQuery(state, action: PayloadAction<string>) {
            state.query = action.payload;
        },
        selectVideo(state, action: PayloadAction<VideoResult | null>) {
            state.selectedVideo = action.payload;
        },
        clearSearchError(state) {
            state.searchError = null;
        },
        clearChatError(state) {
            state.chatError = null;
        },
    },
    extraReducers: (builder) => {
        builder
            // search
            .addCase(searchVideos.pending, (state) => {
                state.searching = true;
                state.searchError = null;
            })
            .addCase(searchVideos.fulfilled, (state, action) => {
                state.searching = false;
                state.results = action.payload;
            })
            .addCase(searchVideos.rejected, (state, action) => {
                state.searching = false;
                state.searchError = action.error.message ?? 'Search failed.';
            })
            // chat history
            .addCase(fetchChatHistory.pending, (state) => {
                state.chatLoading = true;
                state.chatError = null;
            })
            .addCase(fetchChatHistory.fulfilled, (state, action) => {
                state.chatLoading = false;
                state.chatByVideoId[action.payload.videoId] = action.payload.messages;
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
                const { videoId, userMessage, assistantMessage } = action.payload;
                const existing = state.chatByVideoId[videoId] ?? [];
                state.chatByVideoId[videoId] = [...existing, userMessage, assistantMessage];
            })
            .addCase(sendChatMessage.rejected, (state, action) => {
                state.chatLoading = false;
                state.chatError = action.error.message ?? 'Message failed to send.';
            });
    },
});

export const { setQuery, selectVideo, clearSearchError, clearChatError } = videoExplorerSlice.actions;
export default videoExplorerSlice.reducer;
