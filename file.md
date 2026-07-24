// ═══════════════════════════════════════════════
// pages/VideoExplorer/VideoExplorer.tsx
// Content Analytics · Video Explorer
// ═══════════════════════════════════════════════
import React, { useEffect, useMemo, useRef, useState } from 'react';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import {
    loadLibrary,
    selectFile,
    fetchChatHistory,
    sendChatMessage,
    clearChatError,
    fetchMediaBlobUrl,
    type LibraryItem,
} from '../../store/videoExplorerSlice';
import styles from './VideoExplorer.module.scss';

// ─────────────────────────────────────────────
// Icons
// ─────────────────────────────────────────────
const VideoIc: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <rect x="1.5" y="3.5" width="13" height="9" rx="2" />
        <path d="M6.5 6.2v3.6l3-1.8z" fill="currentColor" stroke="none" />
    </svg>
);

const AudioIc: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <path d="M5.5 9.5V4.8L11 3.5v5.3" />
        <circle cx="4" cy="10.5" r="1.8" />
        <circle cx="9.5" cy="9.5" r="1.8" />
    </svg>
);

const STATUS_LABEL: Record<LibraryItem['status'], string> = {
    completed: 'Ready',
    running: 'Processing',
    queued: 'Queued',
    pending: 'Pending',
};

const formatDate = (iso: string) => {
    const d = new Date(iso);
    if (Number.isNaN(d.getTime())) return iso;
    return d.toLocaleDateString(undefined, { month: 'short', day: 'numeric', year: 'numeric' });
};

const formatTime = (secs: number) => {
    const m = Math.floor(secs / 60);
    const s = Math.floor(secs % 60);
    return `${m}:${s.toString().padStart(2, '0')}`;
};

// ─────────────────────────────────────────────
// File card
// ─────────────────────────────────────────────
const FileCard: React.FC<{ item: LibraryItem; active: boolean; onClick: () => void }> = ({ item, active, onClick }) => (
    <button type="button" className={`${styles.fileCard} ${active ? styles.fileCardActive : ''}`} onClick={onClick}>
        <div className={styles.fileIcWrap}>
            {item.mediaKind === 'audio' ? <AudioIc /> : <VideoIc />}
        </div>
        <div className={styles.fileMeta}>
            <div className={styles.fileTitle}>{item.original_name}</div>
            <div className={styles.fileSub}>
                {formatDate(item.inserted_at)}
                <span className={`${styles.statusTag} ${styles[`status_${item.status}`]}`}>
                    {STATUS_LABEL[item.status]}
                </span>
            </div>
        </div>
    </button>
);

// ─────────────────────────────────────────────
// Chat panel
// ─────────────────────────────────────────────
const ChatPanel: React.FC<{ fileId: number; currentTime: number }> = ({ fileId, currentTime }) => {
    const dispatch = useAppDispatch();
    const { chatByFileId, chatLoading, chatError } = useAppSelector((s) => s.videoExplorer);
    const [draft, setDraft] = useState('');
    const [tagTimestamp, setTagTimestamp] = useState(false);
    const scrollRef = useRef<HTMLDivElement>(null);

    const messages = chatByFileId[fileId] ?? [];

    useEffect(() => {
        dispatch(fetchChatHistory(fileId));
        dispatch(clearChatError());
    }, [dispatch, fileId]);

    useEffect(() => {
        scrollRef.current?.scrollTo({ top: scrollRef.current.scrollHeight, behavior: 'smooth' });
    }, [messages.length]);

    const handleSend = () => {
        const content = draft.trim();
        if (!content || chatLoading) return;
        setDraft('');
        dispatch(
            sendChatMessage({
                fileId,
                content,
                timestamp_seconds: tagTimestamp ? Math.floor(currentTime) : undefined,
            }),
        );
    };

    return (
        <div className={styles.chatPanel}>
            <div className={styles.chatHead}>
                <div className={styles.chatHeadTitle}>Ask about this file</div>
                <div className={styles.chatHeadSub}>Answers are grounded in the transcript</div>
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
                    placeholder="Ask a question…"
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
    const { library, libraryLoading, libraryError, libraryLoaded, selectedFileId } = useAppSelector(
        (s) => s.videoExplorer,
    );

    const [query, setQuery] = useState('');
    const [currentTime, setCurrentTime] = useState(0);

    const [mediaUrl, setMediaUrl] = useState('');
    const [mediaLoading, setMediaLoading] = useState(false);
    const [mediaError, setMediaError] = useState('');

    // One-time initial load — last 1 year → today (static for now).
    useEffect(() => {
        if (!libraryLoaded && !libraryLoading) {
            dispatch(loadLibrary());
        }
    }, [dispatch, libraryLoaded, libraryLoading]);

    // Client-side search — filters the already-loaded library, no re-fetch.
    const filtered = useMemo(() => {
        const q = query.trim().toLowerCase();
        if (!q) return library;
        return library.filter((f) => f.original_name.toLowerCase().includes(q));
    }, [library, query]);

    const selectedItem = useMemo(
        () => library.find((f) => f.id === selectedFileId) ?? null,
        [library, selectedFileId],
    );

    useEffect(() => {
        if (!selectedItem) {
            setMediaUrl('');
            setMediaError('');
            return undefined;
        }
        const controller = new AbortController();
        let revokedUrl: string | null = null;
        setMediaLoading(true);
        setMediaError('');
        setMediaUrl('');
        fetchMediaBlobUrl(selectedItem.id, controller.signal)
            .then((url) => {
                revokedUrl = url;
                setMediaUrl(url);
                setMediaLoading(false);
            })
            .catch((err) => {
                if (controller.signal.aborted) return;
                setMediaError(err instanceof Error ? err.message : 'Could not load media');
                setMediaLoading(false);
            });
        return () => {
            controller.abort();
            if (revokedUrl) URL.revokeObjectURL(revokedUrl);
        };
    }, [selectedItem?.id]); // eslint-disable-line react-hooks/exhaustive-deps

    const handleSelect = (item: LibraryItem) => {
        dispatch(selectFile(item.id));
        setCurrentTime(0);
    };

    return (
        <div className={styles.page}>

            {/* ── Page header ── */}
            <div className={styles.ph}>
                <div className={styles.phRow}>
                    <div>
                        <div className={styles.phTitle}>Video Explorer</div>
                        <div className={styles.phSub}>browse your files, play, and ask the assistant about any one of them</div>
                    </div>
                </div>

                <div className={styles.searchRow}>
                    <svg className={styles.searchIc} width="15" height="15" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                        <circle cx="7" cy="7" r="5" />
                        <path d="M11 11l3.5 3.5" />
                    </svg>
                    <input
                        className={styles.searchInput}
                        placeholder="Search by file name…"
                        value={query}
                        onChange={(e) => setQuery(e.target.value)}
                    />
                </div>
            </div>

            {/* ── Body: results + player/chat ── */}
            <div className={styles.body}>

                {/* Results rail */}
                <div className={styles.resultsRail}>
                    {libraryError && <div className={styles.errorBanner}>{libraryError}</div>}

                    {libraryLoading && (
                        <div className={styles.resultsSkeleton}>
                            {Array.from({ length: 5 }).map((_, i) => <div key={i} className={styles.skeletonCard} />)}
                        </div>
                    )}

                    {!libraryLoading && filtered.length === 0 && (
                        <div className={styles.resultsEmpty}>
                            {library.length === 0 ? 'No files found in the last year.' : 'No files match your search.'}
                        </div>
                    )}

                    <div className={styles.resultsList}>
                        {filtered.map((f) => (
                            <FileCard
                                key={f.id}
                                item={f}
                                active={selectedFileId === f.id}
                                onClick={() => handleSelect(f)}
                            />
                        ))}
                    </div>
                </div>

                {/* Player + chat */}
                <div className={styles.watchArea}>
                    {selectedItem ? (
                        <>
                            <div className={styles.playerWrap}>
                                <div className={`${styles.playerFrame} ${selectedItem.mediaKind === 'audio' ? styles.playerFrameAudio : ''}`}>
                                    {selectedItem.mediaKind === 'audio' ? (
                                        <div className={styles.audioStage}>
                                            <div className={styles.audioIcBig}><AudioIc /></div>
                                            <audio
                                                key={selectedItem.id}
                                                src={mediaUrl || undefined}
                                                controls
                                                onTimeUpdate={(e) => setCurrentTime(e.currentTarget.currentTime)}
                                            />
                                        </div>
                                    ) : (
                                        <video
                                            key={selectedItem.id}
                                            src={mediaUrl || undefined}
                                            controls
                                            onTimeUpdate={(e) => setCurrentTime(e.currentTarget.currentTime)}
                                        />
                                    )}
                                    {mediaLoading && (
                                        <div className={styles.mediaLoadingOverlay}>
                                            <div className={styles.spinner} />
                                            <span>Loading media…</span>
                                        </div>
                                    )}
                                    {mediaError && (
                                        <div className={styles.mediaErrorOverlay}>
                                            <div>Could not load media</div>
                                            <div className={styles.mediaErrorMsg}>{mediaError}</div>
                                        </div>
                                    )}
                                </div>
                                <div className={styles.playerTitle}>{selectedItem.original_name}</div>
                                <div className={styles.playerSub}>Uploaded {formatDate(selectedItem.inserted_at)}</div>
                            </div>
                            <ChatPanel fileId={selectedItem.id} currentTime={currentTime} />
                        </>
                    ) : (
                        <div className={styles.watchEmpty}>
                            <svg width="34" height="34" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.3" strokeLinecap="round" strokeLinejoin="round">
                                <rect x="2" y="4" width="20" height="14" rx="2.5" />
                                <path d="M10 9l5 3-5 3z" />
                            </svg>
                            <div>Select a file to start playing and chatting</div>
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
    max-width: 420px;
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

.resultsEmpty {
    color: var(--t2);
    font-size: 12px;
    padding: 20px 6px;
    line-height: 1.5;
}

.resultsSkeleton {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.skeletonCard {
    height: 52px;
    border-radius: var(--rl);
    background: linear-gradient(90deg, var(--bg2) 25%, var(--bg3) 37%, var(--bg2) 63%);
    background-size: 400% 100%;
    animation: veShimmer 1.4s ease infinite;
}

.resultsList {
    display: flex;
    flex-direction: column;
    gap: 6px;
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

// ── File card ────────────────────────────────────
.fileCard {
    display: flex;
    gap: 10px;
    align-items: center;
    padding: 9px 10px;
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

.fileCardActive {
    background: var(--bg3);
    border-color: var(--blue-bdr);
}

.fileIcWrap {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 34px;
    height: 34px;
    flex-shrink: 0;
    border-radius: 9px;
    background: var(--bg2);
    color: var(--t1);

    svg {
        width: 16px;
        height: 16px;
    }
}

.fileMeta {
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 3px;
}

.fileTitle {
    font-size: 12px;
    font-weight: 600;
    color: var(--t0);
    line-height: 1.35;
    @include m.truncate;
}

.fileSub {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 10.5px;
    color: var(--t2);
    @include m.mono;
}

.statusTag {
    display: inline-flex;
    align-items: center;
    padding: 1px 6px;
    border-radius: 99px;
    font-size: 9.5px;
    font-weight: 600;
    letter-spacing: 0.02em;
}

.status_completed {
    background: var(--green-dim);
    color: var(--green);
}

.status_running {
    background: var(--blue-dim);
    color: var(--blue);
}

.status_queued,
.status_pending {
    background: var(--amber-dim);
    color: var(--amber);
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

.playerFrameAudio {
    padding-top: 32%;
    background: var(--bg2);
}

.audioStage {
    position: absolute;
    inset: 0;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 16px;
    padding: 0 24px;
}

.audioIcBig {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 48px;
    height: 48px;
    border-radius: 12px;
    background: var(--bg3);
    color: var(--blue);

    svg {
        width: 22px;
        height: 22px;
    }
}

.audioStage audio {
    width: 100%;
    max-width: 420px;
}

.playerTitle {
    font-size: 14px;
    font-weight: 600;
    color: var(--t0);
    margin-top: 8px;
    line-height: 1.4;
    @include m.truncate;
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

.mediaLoadingOverlay,
.mediaErrorOverlay {
    position: absolute;
    inset: 0;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 8px;
    background: rgba(0, 0, 0, 0.55);
    color: var(--t1);
    font-size: 12px;
    text-align: center;
    padding: 0 20px;
}

.mediaErrorOverlay {
    color: var(--red);
}

.mediaErrorMsg {
    font-size: 10.5px;
    color: var(--t2);
    @include m.mono;
}

.spinner {
    width: 20px;
    height: 20px;
    border-radius: 50%;
    border: 2px solid var(--bdr2);
    border-top-color: var(--blue);
    animation: veSpin 0.7s linear infinite;
}

@keyframes veSpin {
    to { transform: rotate(360deg); }
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
    mediaEndpoint: string; // API path — fetch via fetchMediaBlobUrl(item.id), not usable as a direct <video>/<audio> src
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

// Endpoint for a given file id's raw media bytes. Relative — resolved
// through the axios instance (so auth headers/cookies are attached),
// NOT used directly as a <video>/<audio> src.
const buildMediaUrl = (fileId: number) => `/stt/files/load_media?fileID=${fileId}`;

// Fetches the media as a blob and returns an object URL, mirroring the
// working pattern used in TranscriptDetail.tsx (fetchMediaBlobUrl). A
// bare <video src="...api url...">  won't carry axios's auth headers/
// cookies, which is why that approach didn't actually play anything.
export async function fetchMediaBlobUrl(fileId: number, signal?: AbortSignal): Promise<string> {
    const res = await axios.get(buildMediaUrl(fileId), {
        responseType: 'blob',
        signal,
    });
    return URL.createObjectURL(res.data as Blob);
}

const toLibraryItem = (file: ServerFile, status: LibraryStatus): LibraryItem => ({
    ...file,
    status,
    mediaKind: mediaKindOf(file.original_name),
    mediaEndpoint: buildMediaUrl(file.id),
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
























// ═══════════════════════════════════════════════
// VideoExplorer.module.scss
// Content Analytics · Video Explorer
// ═══════════════════════════════════════════════
@use '../../styles/mixins' as m;

.page {
    height: 100%;
    overflow-y: auto;
    @include m.scrollbar;
}

// ── Search header ────────────────────────────────
.ph {
    position: sticky;
    top: 0;
    z-index: 2;
    padding: 18px 24px 12px;
    background: var(--bg1);
    border-bottom: 1px solid var(--bdr);
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 12px;
}

.searchRow {
    position: relative;
    width: 100%;
    max-width: 560px;
    display: flex;
    align-items: center;
    color: var(--t2);
}

.searchRow svg:first-child {
    position: absolute;
    left: 12px;
    pointer-events: none;
}

.searchInput {
    width: 100%;
    background: var(--bg3);
    border: 1px solid var(--bdr2);
    border-radius: 99px;
    color: var(--t0);
    font-family: var(--font-ui);
    font-size: 13.5px;
    padding: 10px 36px;
    outline: none;
    transition: border-color 0.12s, box-shadow 0.12s;

    &::placeholder { color: var(--t2); }

    &:focus {
        border-color: var(--blue-bdr);
        box-shadow: 0 0 0 2px var(--blue-dim);
    }
}

.searchClearBtn {
    position: absolute;
    right: 10px;
    display: flex;
    align-items: center;
    justify-content: center;
    width: 20px;
    height: 20px;
    border-radius: 50%;
    border: none;
    background: var(--bg2);
    color: var(--t2);
    cursor: pointer;

    &:hover { background: var(--bg3); color: var(--t0); }
}

// ── Chip filters ─────────────────────────────────
.chipsRow {
    display: flex;
    gap: 8px;
}

.chip {
    padding: 5px 14px;
    border-radius: 99px;
    border: 1px solid var(--bdr2);
    background: var(--bg2);
    color: var(--t1);
    font-family: var(--font-ui);
    font-size: 11.5px;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.12s;

    &:hover { border-color: var(--bdr3); }
}

.chipActive {
    background: var(--t0);
    border-color: var(--t0);
    color: var(--bg1);
}

// ── Browse body / grid ───────────────────────────
.browseBody {
    padding: 18px 24px 32px;
}

.gridLabel {
    font-size: 13px;
    font-weight: 700;
    color: var(--t0);
    margin-bottom: 12px;
}

.errorBanner {
    padding: 9px 12px;
    border-radius: var(--r);
    background: var(--red-dim);
    border: 1px solid var(--red-bdr);
    color: var(--red);
    font-size: 12px;
    font-weight: 500;
    margin-bottom: 14px;
}

.resultsEmpty {
    color: var(--t2);
    font-size: 12.5px;
    padding: 24px 0;
    text-align: center;
}

.grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
    gap: 18px 14px;
}

.skeletonCard {
    height: 190px;
    border-radius: var(--rl);
    background: linear-gradient(90deg, var(--bg2) 25%, var(--bg3) 37%, var(--bg2) 63%);
    background-size: 400% 100%;
    animation: veShimmer 1.4s ease infinite;
}

// ── Grid card ────────────────────────────────────
.card {
    display: flex;
    flex-direction: column;
    gap: 8px;
    background: transparent;
    border: none;
    padding: 0;
    cursor: pointer;
    text-align: left;

    &:hover .cardTitle { color: var(--blue); }
}

.cardThumb {
    position: relative;
    width: 100%;
    padding-top: 56.25%;
    border-radius: var(--rl);
    overflow: hidden;
}

.cardThumbIc {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    color: rgba(255, 255, 255, 0.85);

    svg { width: 34px; height: 34px; }
}

.cardStatusTag,
.cardKindTag {
    position: absolute;
    padding: 2px 7px;
    border-radius: 5px;
    font-size: 9.5px;
    font-weight: 700;
    letter-spacing: 0.02em;
}

.cardStatusTag {
    top: 8px;
    left: 8px;
}

.cardKindTag {
    bottom: 8px;
    right: 8px;
    background: rgba(0, 0, 0, 0.7);
    color: #fff;
}

.status_completed { background: var(--green-dim); color: var(--green); }
.status_running { background: var(--blue-dim); color: var(--blue); }
.status_queued,
.status_pending { background: var(--amber-dim); color: var(--amber); }

.cardMeta {
    display: flex;
    gap: 8px;
}

.cardAvatar {
    width: 26px;
    height: 26px;
    border-radius: 50%;
    flex-shrink: 0;
    margin-top: 1px;
}

.cardText {
    min-width: 0;
}

.cardTitle {
    font-size: 12.5px;
    font-weight: 600;
    color: var(--t0);
    line-height: 1.35;
    display: -webkit-box;
    -webkit-line-clamp: 2;
    -webkit-box-orient: vertical;
    overflow: hidden;
    transition: color 0.12s;
}

.cardSub {
    font-size: 10.5px;
    color: var(--t2);
    margin-top: 3px;
}

// ── Watch page ───────────────────────────────────
.watchPage {
    padding: 14px 24px 32px;
}

.watchBackBtn {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    margin-bottom: 12px;
    padding: 6px 12px 6px 8px;
    border-radius: 99px;
    border: 1px solid var(--bdr2);
    background: var(--bg2);
    color: var(--t1);
    font-family: var(--font-ui);
    font-size: 11.5px;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.12s;

    &:hover { border-color: var(--bdr3); color: var(--t0); }
}

.watchGrid {
    display: flex;
    gap: 20px;
    align-items: flex-start;
}

.watchMain {
    flex: 1.7;
    min-width: 0;
}

.watchSide {
    flex: 1;
    min-width: 320px;
    max-width: 420px;
    display: flex;
    flex-direction: column;
    gap: 16px;
}

.playerFrame {
    position: relative;
    width: 100%;
    padding-top: 56.25%;
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

.playerFrameAudio {
    padding-top: 32%;
    background: var(--bg2);
}

.audioStage {
    position: absolute;
    inset: 0;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 16px;
    padding: 0 24px;
}

.audioIcBig {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 48px;
    height: 48px;
    border-radius: 12px;
    background: var(--bg3);
    color: var(--blue);

    svg { width: 22px; height: 22px; }
}

.audioStage audio {
    width: 100%;
    max-width: 420px;
}

.mediaLoadingOverlay,
.mediaErrorOverlay {
    position: absolute;
    inset: 0;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 8px;
    background: rgba(0, 0, 0, 0.55);
    color: var(--t1);
    font-size: 12px;
    text-align: center;
    padding: 0 20px;
}

.mediaErrorOverlay { color: var(--red); }

.mediaErrorMsg {
    font-size: 10.5px;
    color: var(--t2);
    @include m.mono;
}

.spinner {
    width: 20px;
    height: 20px;
    border-radius: 50%;
    border: 2px solid var(--bdr2);
    border-top-color: var(--blue);
    animation: veSpin 0.7s linear infinite;
}

.watchTitle {
    font-size: 15px;
    font-weight: 700;
    color: var(--t0);
    margin-top: 12px;
    line-height: 1.4;
}

.watchMetaRow {
    display: flex;
    align-items: center;
    gap: 10px;
    margin-top: 6px;
    font-size: 11.5px;
    color: var(--t2);
}

// ── Up next ──────────────────────────────────────
.upNextPanel {
    background: var(--bg1);
    border: 1px solid var(--bdr);
    border-radius: var(--rl);
    overflow: hidden;
}

.upNextHead {
    padding: 10px 14px;
    font-size: 12px;
    font-weight: 700;
    color: var(--t0);
    border-bottom: 1px solid var(--bdr);
}

.upNextList {
    max-height: 420px;
    overflow-y: auto;
    padding: 8px;
    display: flex;
    flex-direction: column;
    gap: 4px;
    @include m.scrollbar;
}

.upNextRow {
    display: flex;
    gap: 9px;
    align-items: center;
    padding: 6px;
    border-radius: var(--r);
    border: 1px solid transparent;
    background: transparent;
    cursor: pointer;
    text-align: left;
    transition: all 0.12s;

    &:hover { background: var(--bg2); }
}

.upNextRowActive {
    background: var(--bg3);
    border-color: var(--blue-bdr);
}

.upNextThumb {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 68px;
    height: 42px;
    flex-shrink: 0;
    border-radius: 6px;
    color: rgba(255, 255, 255, 0.85);

    svg { width: 16px; height: 16px; }
}

.upNextMeta {
    min-width: 0;
}

.upNextTitle {
    font-size: 11.5px;
    font-weight: 600;
    color: var(--t0);
    line-height: 1.3;
    @include m.truncate;
}

.upNextSub {
    font-size: 10px;
    color: var(--t2);
    margin-top: 2px;
}

// ── Chat panel ───────────────────────────────────
.chatPanel {
    display: flex;
    flex-direction: column;
    background: var(--bg1);
    border: 1px solid var(--bdr);
    border-radius: var(--rl);
    overflow: hidden;
    max-height: 420px;
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
    min-height: 80px;
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

.msgRow { display: flex; justify-content: flex-start; }
.msgRowUser { justify-content: flex-end; }

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

    &:hover { border-color: var(--bdr3); color: var(--t1); }
}

.timestampChipActive {
    background: var(--blue-dim);
    border-color: var(--blue-bdr);
    color: var(--blue);
}

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

    &::placeholder { color: var(--t2); }

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

    &:hover:not(:disabled) { filter: brightness(1.08); }
    &:disabled { opacity: 0.5; cursor: default; }
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

@keyframes veSpin {
    to { transform: rotate(360deg); }
}





























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
    mediaEndpoint: string; // API path — fetch via fetchMediaBlobUrl(item.id), not usable as a direct <video>/<audio> src
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

// Endpoint for a given file id's raw media bytes. Relative — resolved
// through the axios instance (so auth headers/cookies are attached),
// NOT used directly as a <video>/<audio> src.
const buildMediaUrl = (fileId: number) => `/stt/files/load_media?fileID=${fileId}`;

// Fetches the media as a blob and returns an object URL, mirroring the
// working pattern used in TranscriptDetail.tsx (fetchMediaBlobUrl). A
// bare <video src="...api url...">  won't carry axios's auth headers/
// cookies, which is why that approach didn't actually play anything.
export async function fetchMediaBlobUrl(fileId: number, signal?: AbortSignal): Promise<string> {
    const res = await axios.get(buildMediaUrl(fileId), {
        responseType: 'blob',
        signal,
    });
    return URL.createObjectURL(res.data as Blob);
}

const toLibraryItem = (file: ServerFile, status: LibraryStatus): LibraryItem => ({
    ...file,
    status,
    mediaKind: mediaKindOf(file.original_name),
    mediaEndpoint: buildMediaUrl(file.id),
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
