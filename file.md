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
const BackIc: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round" strokeLinejoin="round">
        <path d="M10 4L6 8l4 4" />
    </svg>
);
const SearchIc: React.FC = () => (
    <svg width="15" height="15" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <circle cx="7" cy="7" r="5" /><path d="M11 11l3.5 3.5" />
    </svg>
);
const ClearIc: React.FC = () => (
    <svg width="12" height="12" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.8" strokeLinecap="round">
        <path d="M4 4l8 8M12 4l-8 8" />
    </svg>
);

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

// Deterministic shuffle-ish "trending" order, stable across re-renders for a given library.
const pickTrending = (items: LibraryItem[], count: number) => {
    const withScore = items.map((item) => ({ item, score: (item.id * 2654435761) % 100000 }));
    withScore.sort((a, b) => a.score - b.score);
    return withScore.slice(0, count).map((w) => w.item);
};

// ─────────────────────────────────────────────
// Client-side video thumbnail generator
// No thumbnail_url exists on ServerFile, so for video files we grab a
// real frame ourselves: fetch the media blob, seek a hidden <video> to
// ~1s in, draw it to a canvas, cache the resulting dataURL by file id.
// Audio files skip this entirely (nothing visual to grab).
// ─────────────────────────────────────────────
const thumbCache = new Map<number, string>();

function useVideoThumbnail(item: LibraryItem): string | null {
    const [thumb, setThumb] = useState<string | null>(thumbCache.get(item.id) ?? null);

    useEffect(() => {
        if (item.mediaKind !== 'video' || thumbCache.has(item.id)) return undefined;
        let cancelled = false;
        let objectUrl: string | null = null;
        const video = document.createElement('video');
        video.muted = true;
        video.playsInline = true;
        video.preload = 'metadata';

        fetchMediaBlobUrl(item.id)
            .then((url) => {
                if (cancelled) return;
                objectUrl = url;
                video.src = url;
            })
            .catch(() => {});

        const onLoadedMetadata = () => {
            video.currentTime = Math.min(1, (video.duration || 2) / 2);
        };
        const onSeeked = () => {
            try {
                const canvas = document.createElement('canvas');
                canvas.width = video.videoWidth || 320;
                canvas.height = video.videoHeight || 180;
                const ctx = canvas.getContext('2d');
                if (ctx) {
                    ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
                    const dataUrl = canvas.toDataURL('image/jpeg', 0.7);
                    thumbCache.set(item.id, dataUrl);
                    if (!cancelled) setThumb(dataUrl);
                }
            } catch {
                // canvas draw can fail on some codecs/CORS setups — silently fall back to icon tile
            }
        };
        video.addEventListener('loadedmetadata', onLoadedMetadata);
        video.addEventListener('seeked', onSeeked);

        return () => {
            cancelled = true;
            video.removeEventListener('loadedmetadata', onLoadedMetadata);
            video.removeEventListener('seeked', onSeeked);
            if (objectUrl) URL.revokeObjectURL(objectUrl);
        };
    }, [item.id, item.mediaKind]);

    return thumb;
}

// ─────────────────────────────────────────────
// Video/audio card (grid tile — home + search results)
// ─────────────────────────────────────────────
// Kept alive for the session (not revoked) so hover-preview playback can
// reuse the same blob without refetching every time — fine at dev/local
// scale; revisit if the library gets large (see WIRING_NOTES).
const previewUrlCache = new Map<number, string>();

function useHoverPreview(item: LibraryItem) {
    const [hovering, setHovering] = useState(false);
    const [previewUrl, setPreviewUrl] = useState<string | null>(previewUrlCache.get(item.id) ?? null);
    const videoRef = useRef<HTMLVideoElement>(null);

    const onEnter = () => {
        if (item.mediaKind !== 'video') return;
        setHovering(true);
        if (!previewUrlCache.has(item.id)) {
            fetchMediaBlobUrl(item.id)
                .then((url) => {
                    previewUrlCache.set(item.id, url);
                    setPreviewUrl(url);
                })
                .catch(() => {});
        }
    };

    const onLeave = () => {
        setHovering(false);
        if (videoRef.current) {
            videoRef.current.pause();
            videoRef.current.currentTime = 0;
        }
    };

    useEffect(() => {
        if (hovering && previewUrl && videoRef.current) {
            videoRef.current.play().catch(() => {});
        }
    }, [hovering, previewUrl]);

    return { hovering, previewUrl, videoRef, onEnter, onLeave };
}

const AudioCover: React.FC = () => (
    <div className={styles.audioCover}>
        <AudioIc />
    </div>
);

const MediaCard: React.FC<{ item: LibraryItem; onClick: () => void }> = ({ item, onClick }) => {
    const thumb = useVideoThumbnail(item);
    const { hovering, previewUrl, videoRef, onEnter, onLeave } = useHoverPreview(item);
    return (
        <button type="button" className={styles.card} onClick={onClick} onMouseEnter={onEnter} onMouseLeave={onLeave}>
            <div className={styles.cardThumb}>
                {item.mediaKind === 'audio' ? (
                    <AudioCover />
                ) : (
                    <>
                        {thumb ? <img className={styles.cardThumbImg} src={thumb} alt="" /> : <div className={styles.cardThumbIc}><VideoIc /></div>}
                        {hovering && previewUrl && (
                            <video
                                ref={videoRef}
                                className={styles.cardThumbPreview}
                                src={previewUrl}
                                muted
                                loop
                                playsInline
                            />
                        )}
                    </>
                )}
                <span className={styles.cardKindTag}>{item.mediaKind === 'audio' ? 'Audio' : 'Video'}</span>
            </div>
            <div className={styles.cardMeta}>
                <div className={styles.cardAvatar}>{item.mediaKind === 'audio' ? <AudioIc /> : <VideoIc />}</div>
                <div className={styles.cardText}>
                    <div className={styles.cardTitle}>{item.original_name}</div>
                    <div className={styles.cardSub}>Uploaded {formatDate(item.inserted_at)}</div>
                </div>
            </div>
        </button>
    );
};

// ─────────────────────────────────────────────
// Up-next row (compact horizontal card, watch page sidebar)
// ─────────────────────────────────────────────
const UpNextRow: React.FC<{ item: LibraryItem; active: boolean; onClick: () => void }> = ({ item, active, onClick }) => {
    const thumb = useVideoThumbnail(item);
    return (
        <button type="button" className={`${styles.upNextRow} ${active ? styles.upNextRowActive : ''}`} onClick={onClick}>
            <div className={styles.upNextThumb}>
                {item.mediaKind === 'audio' ? (
                    <div className={styles.upNextAudioCover}><AudioIc /></div>
                ) : thumb ? (
                    <img className={styles.upNextThumbImg} src={thumb} alt="" />
                ) : (
                    <VideoIc />
                )}
            </div>
            <div className={styles.upNextMeta}>
                <div className={styles.upNextTitle}>{item.original_name}</div>
                <div className={styles.upNextSub}>{formatDate(item.inserted_at)}</div>
            </div>
        </button>
    );
};

// ─────────────────────────────────────────────
// Chat panel
// ─────────────────────────────────────────────
const ChatPanel: React.FC<{ fileId: number; currentTime: number }> = ({ fileId, currentTime }) => {
    const dispatch = useAppDispatch();
    const { chatByFileId, chatLoading, chatError } = useAppSelector((s) => s.videoExplorer);
    const [draft, setDraft] = useState('');
    const [tagTimestamp, setTagTimestamp] = useState(false);

    const messages = chatByFileId[fileId] ?? [];

    useEffect(() => {
        dispatch(fetchChatHistory(fileId));
        dispatch(clearChatError());
    }, [dispatch, fileId]);

    const handleSend = () => {
        const content = draft.trim();
        if (!content || chatLoading) return;
        setDraft('');
        dispatch(sendChatMessage({ fileId, content, timestamp_seconds: tagTimestamp ? Math.floor(currentTime) : undefined }));
    };

    return (
        <div className={styles.chatPanel}>
            <div className={styles.chatHead}>
                <div className={styles.chatHeadTitle}>Ask about this file</div>
                <div className={styles.chatHeadSub}>LLM integration coming next — UI only for now</div>
            </div>

            <div className={styles.chatBody}>
                {chatError && <div className={styles.chatErrorBanner}>{chatError}</div>}
                {messages.length === 0 && !chatLoading ? (
                    <div className={styles.chatEmpty}>Ask a question — e.g. "Summarize this" or "What's said about X?"</div>
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
                            <span className={styles.typingDot} /><span className={styles.typingDot} /><span className={styles.typingDot} />
                        </div>
                    </div>
                )}
            </div>

            <div className={styles.chatToolsRow}>
                <button type="button" className={`${styles.timestampChip} ${tagTimestamp ? styles.timestampChipActive : ''}`} onClick={() => setTagTimestamp((v) => !v)}>
                    <svg width="11" height="11" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round" strokeLinejoin="round">
                        <circle cx="8" cy="8" r="6" /><path d="M8 4.5V8l2.5 1.5" />
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
                    onKeyDown={(e) => { if (e.key === 'Enter' && !e.shiftKey) { e.preventDefault(); handleSend(); } }}
                    rows={2}
                />
                <button type="button" className={styles.sendBtn} onClick={handleSend} disabled={!draft.trim() || chatLoading} aria-label="Send">
                    <svg width="15" height="15" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                        <path d="M14.5 1.5L7.5 8.5M14.5 1.5L10 14.5l-2.5-6-6-2.5z" />
                    </svg>
                </button>
            </div>
        </div>
    );
};

// ─────────────────────────────────────────────
// Watch page
// ─────────────────────────────────────────────
const WatchPage: React.FC<{ item: LibraryItem; all: LibraryItem[]; onBack: () => void; onSelect: (i: LibraryItem) => void }> = ({
    item, all, onBack, onSelect,
}) => {
    const [mediaUrl, setMediaUrl] = useState('');
    const [mediaLoading, setMediaLoading] = useState(false);
    const [mediaError, setMediaError] = useState('');
    const [currentTime, setCurrentTime] = useState(0);

    useEffect(() => {
        const controller = new AbortController();
        let revokedUrl: string | null = null;
        setMediaLoading(true); setMediaError(''); setMediaUrl(''); setCurrentTime(0);
        fetchMediaBlobUrl(item.id, controller.signal)
            .then((url) => { revokedUrl = url; setMediaUrl(url); setMediaLoading(false); })
            .catch((err) => {
                if (controller.signal.aborted) return;
                setMediaError(err instanceof Error ? err.message : 'Could not load media');
                setMediaLoading(false);
            });
        return () => { controller.abort(); if (revokedUrl) URL.revokeObjectURL(revokedUrl); };
    }, [item.id]);

    const upNext = useMemo(() => all.filter((f) => f.id !== item.id), [all, item.id]);

    return (
        <div className={styles.watchPage}>
            <button type="button" className={styles.watchBackBtn} onClick={onBack}>
                <BackIc /> Back to browse
            </button>

            <div className={styles.watchGrid}>
                {/* Main column */}
                <div className={styles.watchMain}>
                    <div className={`${styles.playerFrame} ${item.mediaKind === 'audio' ? styles.playerFrameAudio : ''}`}>
                        {item.mediaKind === 'audio' ? (
                            <div className={styles.audioStage}>
                                <div className={styles.audioIcBig}><AudioIc /></div>
                                <audio key={item.id} src={mediaUrl || undefined} controls onTimeUpdate={(e) => setCurrentTime(e.currentTarget.currentTime)} />
                            </div>
                        ) : (
                            <video key={item.id} src={mediaUrl || undefined} controls onTimeUpdate={(e) => setCurrentTime(e.currentTarget.currentTime)} />
                        )}
                        {mediaLoading && <div className={styles.mediaLoadingOverlay}><div className={styles.spinner} /><span>Loading media…</span></div>}
                        {mediaError && <div className={styles.mediaErrorOverlay}><div>Could not load media</div><div className={styles.mediaErrorMsg}>{mediaError}</div></div>}
                    </div>

                    <div className={styles.watchTitle}>{item.original_name}</div>
                    <div className={styles.watchMetaRow}>
                        <span>Uploaded {formatDate(item.inserted_at)}</span>
                        <span className={styles.cardKindTag}>{item.mediaKind === 'audio' ? 'Audio' : 'Video'}</span>
                    </div>
                </div>

                {/* Sidebar: chat + up next */}
                <div className={styles.watchSide}>
                    <ChatPanel fileId={item.id} currentTime={currentTime} />

                    <div className={styles.upNextPanel}>
                        <div className={styles.upNextHead}>Up next</div>
                        <div className={styles.upNextList}>
                            {upNext.length === 0 && <div className={styles.resultsEmpty}>No other files.</div>}
                            {upNext.map((f) => (
                                <UpNextRow key={f.id} item={f} active={false} onClick={() => onSelect(f)} />
                            ))}
                        </div>
                    </div>
                </div>
            </div>
        </div>
    );
};

// ═════════════════════════════════════════════
// ROOT: VideoExplorer
// ═════════════════════════════════════════════
const VideoExplorer: React.FC = () => {
    const dispatch = useAppDispatch();
    const { library, libraryLoading, libraryError, libraryLoaded, selectedFileId } = useAppSelector((s) => s.videoExplorer);

    const [query, setQuery] = useState('');
    const [kindFilter, setKindFilter] = useState<'all' | 'video' | 'audio'>('all');

    useEffect(() => {
        if (!libraryLoaded && !libraryLoading) dispatch(loadLibrary());
    }, [dispatch, libraryLoaded, libraryLoading]);

    const kindFiltered = useMemo(
        () => (kindFilter === 'all' ? library : library.filter((f) => f.mediaKind === kindFilter)),
        [library, kindFilter],
    );

    const isSearching = query.trim().length > 0;

    const searchResults = useMemo(() => {
        const q = query.trim().toLowerCase();
        if (!q) return [];
        // TODO: replace with the real search algorithm/backend later.
        // For now: client-side match over the already-loaded library.
        return kindFiltered.filter((f) => f.original_name.toLowerCase().includes(q));
    }, [kindFiltered, query]);

    const trending = useMemo(() => pickTrending(kindFiltered, 24), [kindFiltered]);

    const selectedItem = useMemo(() => library.find((f) => f.id === selectedFileId) ?? null, [library, selectedFileId]);

    const openItem = (item: LibraryItem) => dispatch(selectFile(item.id));
    const backToBrowse = () => dispatch(selectFile(null));

    if (selectedItem) {
        return (
            <div className={styles.page}>
                <WatchPage item={selectedItem} all={library} onBack={backToBrowse} onSelect={openItem} />
            </div>
        );
    }

    const gridItems = isSearching ? searchResults : trending;
    const gridLabel = isSearching ? `Search results for "${query.trim()}"` : 'Recommended';

    return (
        <div className={styles.page}>

            {/* ── Header ── */}
            <div className={styles.ph}>
                <div className={styles.phTitleRow}>
                    <div className={styles.phTitle}>Video Explorer</div>
                    <div className={styles.phSub}>Browse, search, and chat with your video & audio library</div>
                </div>

                <div className={styles.phToolbar}>
                    <div className={styles.searchRow}>
                        <SearchIc />
                        <input
                            className={styles.searchInput}
                            placeholder="Search video and audio files…"
                            value={query}
                            onChange={(e) => setQuery(e.target.value)}
                        />
                        {query && (
                            <button type="button" className={styles.searchClearBtn} onClick={() => setQuery('')} aria-label="Clear search">
                                <ClearIc />
                            </button>
                        )}
                    </div>

                    <div className={styles.chipsRow}>
                        {(['all', 'video', 'audio'] as const).map((k) => (
                            <button
                                key={k}
                                type="button"
                                className={`${styles.chip} ${kindFilter === k ? styles.chipActive : ''}`}
                                onClick={() => setKindFilter(k)}
                            >
                                {k === 'all' ? 'All' : k === 'video' ? 'Video' : 'Audio'}
                            </button>
                        ))}
                    </div>
                </div>
            </div>

            {/* ── Body ── */}
            <div className={styles.browseBody}>
                {libraryError && <div className={styles.errorBanner}>{libraryError}</div>}

                <div className={styles.gridLabel}>{gridLabel}</div>

                {libraryLoading ? (
                    <div className={styles.grid}>
                        {Array.from({ length: 10 }).map((_, i) => <div key={i} className={styles.skeletonCard} />)}
                    </div>
                ) : gridItems.length === 0 ? (
                    <div className={styles.resultsEmpty}>
                        {isSearching ? 'No files match your search.' : 'No files found in the last year.'}
                    </div>
                ) : (
                    <div className={styles.grid}>
                        {gridItems.map((item) => (
                            <MediaCard key={item.id} item={item} onClick={() => openItem(item)} />
                        ))}
                    </div>
                )}
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

.page {
    height: 100%;
    overflow-y: auto;
    @include m.scrollbar;
}

// ── Header ───────────────────────────────────────
.ph {
    position: sticky;
    top: 0;
    z-index: 2;
    padding: 18px 24px 14px;
    background: var(--bg1);
    border-bottom: 1px solid var(--bdr);
    display: flex;
    flex-direction: column;
    gap: 14px;
}

.phTitleRow {
    display: flex;
    align-items: baseline;
    gap: 10px;
    flex-wrap: wrap;
}

.phTitle {
    font-size: 18px;
    font-weight: 700;
    color: var(--t0);
    letter-spacing: -0.3px;
    font-family: var(--font-display);
}

.phSub {
    font-size: 11.5px;
    color: var(--t2);
}

.phToolbar {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 16px;
    flex-wrap: wrap;
}

.searchRow {
    position: relative;
    flex: 1;
    min-width: 220px;
    max-width: 440px;
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
    flex-shrink: 0;
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
    grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
    gap: 26px 20px;
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
    &:hover .cardThumb { border-color: var(--bdr3); }
}

.cardThumb {
    position: relative;
    width: 100%;
    padding-top: 56.25%;
    border-radius: var(--rl);
    overflow: hidden;
    background: var(--bg3);
    border: 1px solid var(--bdr2);
}

.cardThumbImg {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    object-fit: cover;
}

.cardThumbPreview {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    object-fit: cover;
    background: #000;
    animation: veFadeIn 0.15s ease;
}

.cardThumbIc {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    color: var(--t2);

    svg { width: 30px; height: 30px; }
}

.cardKindTag {
    position: absolute;
    top: 8px;
    right: 8px;
    padding: 2px 7px;
    border-radius: 5px;
    font-size: 9.5px;
    font-weight: 700;
    letter-spacing: 0.02em;
    background: rgba(0, 0, 0, 0.7);
    color: #fff;
}

// ── Audio cover art ──────────────────────────────
.audioCover {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    background: var(--bg3);
    color: var(--t2);

    svg {
        width: 28px;
        height: 28px;
    }
}

.cardMeta {
    display: flex;
    gap: 10px;
}

.cardAvatar {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 30px;
    height: 30px;
    border-radius: 50%;
    flex-shrink: 0;
    margin-top: 1px;
    background: var(--bg3);
    border: 1px solid var(--bdr2);
    color: var(--t2);

    svg { width: 14px; height: 14px; }
}

.cardText {
    min-width: 0;
}

.cardTitle {
    font-size: 13.5px;
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
    font-size: 11px;
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
    position: relative;
    display: flex;
    align-items: center;
    justify-content: center;
    width: 68px;
    height: 42px;
    flex-shrink: 0;
    border-radius: 6px;
    background: var(--bg3);
    border: 1px solid var(--bdr2);
    color: var(--t2);
    overflow: hidden;

    svg { width: 16px; height: 16px; }
}

.upNextThumbImg {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    object-fit: cover;
}

.upNextAudioCover {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    background: var(--bg3);
    color: var(--t2);
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

@keyframes veFadeIn {
    from { opacity: 0; }
    to { opacity: 1; }
}
