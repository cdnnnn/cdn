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
    <svg width="14" height="14" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round" strokeLinejoin="round">
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
const PlayIc: React.FC = () => (
    <svg width="15" height="15" viewBox="0 0 16 16" fill="currentColor"><path d="M4 2.5l10 5.5-10 5.5z" /></svg>
);
const PauseIc: React.FC = () => (
    <svg width="15" height="15" viewBox="0 0 16 16" fill="currentColor">
        <rect x="3.5" y="2.5" width="3.2" height="11" rx="1" /><rect x="9.3" y="2.5" width="3.2" height="11" rx="1" />
    </svg>
);
const VolumeIc: React.FC = () => (
    <svg width="15" height="15" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <path d="M2 6h2.5L8 3v10L4.5 10H2z" fill="currentColor" stroke="none" />
        <path d="M10.5 5.5a4 4 0 010 5.6" /><path d="M12.3 3.7a7 7 0 010 8.6" />
    </svg>
);
const MuteIc: React.FC = () => (
    <svg width="15" height="15" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <path d="M2 6h2.5L8 3v10L4.5 10H2z" fill="currentColor" stroke="none" />
        <path d="M10.5 6l3.5 4M14 6l-3.5 4" />
    </svg>
);
const FullscreenIc: React.FC = () => (
    <svg width="14" height="14" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <path d="M2 5.5V2h3.5M14 5.5V2h-3.5M2 10.5V14h3.5M14 10.5V14h-3.5" />
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
// Chapter/segment generation (dev placeholder)
// No chapter data exists yet — for now we generate deterministic,
// evenly-jittered segments from the file's real duration so the UI/UX
// can be built and wired up. Swap `buildChapters` for real chapter data
// (e.g. transcript-derived topic boundaries) once the backend has it.
// ─────────────────────────────────────────────
export interface Chapter {
    start: number;
    end: number;
    title: string;
}

const CHAPTER_TITLE_POOL = [
    'Introduction', 'Overview', 'Key Concepts', 'Chapter 1', 'Deep Dive',
    'Chapter 2', 'Case Study', 'Discussion', 'Chapter 3', 'Q & A',
    'Recap', 'Conclusion',
];

function seededRandom(seed: number) {
    let s = seed % 2147483647;
    if (s <= 0) s += 2147483646;
    return () => {
        s = (s * 16807) % 2147483647;
        return (s - 1) / 2147483646;
    };
}

function buildChapters(durationSec: number, seed: number): Chapter[] {
    if (!durationSec || durationSec < 30) return [];
    const rand = seededRandom(seed || 1);
    const count = Math.max(3, Math.min(7, Math.round(3 + rand() * 4)));

    // Split duration into `count` roughly-even parts with some jitter,
    // enforcing a minimum segment length so tiny slivers don't show up.
    const minGap = Math.max(15, durationSec / (count * 3));
    let points = Array.from({ length: count - 1 }, () => rand() * durationSec).sort((a, b) => a - b);
    points = points.reduce<number[]>((acc, p) => {
        const prev = acc.length ? acc[acc.length - 1] : 0;
        return p - prev >= minGap ? [...acc, p] : acc;
    }, []);

    const boundaries = [0, ...points, durationSec];
    return boundaries.slice(0, -1).map((start, i) => ({
        start,
        end: boundaries[i + 1],
        title: CHAPTER_TITLE_POOL[i % CHAPTER_TITLE_POOL.length],
    }));
}

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
// ─────────────────────────────────────────────
// Chat panel — redesigned as the sole, full-height sidebar
// ─────────────────────────────────────────────
const SparkleIc: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="currentColor">
        <path d="M8 1.5l1.1 3.4L12.5 6l-3.4 1.1L8 10.5 6.9 7.1 3.5 6l3.4-1.1L8 1.5z" />
        <path d="M13 9.5l.55 1.7 1.7.55-1.7.55-.55 1.7-.55-1.7-1.7-.55 1.7-.55.55-1.7z" opacity="0.7" />
    </svg>
);

const SUGGESTED_PROMPTS = ['Summarize this file', 'What are the key points?', 'List any action items'];

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

    const send = (content: string) => {
        const trimmed = content.trim();
        if (!trimmed || chatLoading) return;
        setDraft('');
        dispatch(sendChatMessage({ fileId, content: trimmed, timestamp_seconds: tagTimestamp ? Math.floor(currentTime) : undefined }));
    };

    return (
        <div className={styles.chatPanel}>
            <div className={styles.chatHead}>
                <div className={styles.chatHeadIc}><SparkleIc /></div>
                <div>
                    <div className={styles.chatHeadTitle}>AI Assistant</div>
                    <div className={styles.chatHeadSub}>
                        Grounded in the transcript
                        <span className={styles.chatHeadBetaTag}>Soon</span>
                    </div>
                </div>
            </div>

            <div className={styles.chatBody}>
                {chatError && <div className={styles.chatErrorBanner}>{chatError}</div>}

                {messages.length === 0 && !chatLoading ? (
                    <div className={styles.chatEmptyState}>
                        <div className={styles.chatEmptyIc}><SparkleIc /></div>
                        <div className={styles.chatEmptyTitle}>Ask anything about this file</div>
                        <div className={styles.chatEmptyText}>Try one of these to get started</div>
                        <div className={styles.suggestedPrompts}>
                            {SUGGESTED_PROMPTS.map((p) => (
                                <button key={p} type="button" className={styles.suggestedChip} onClick={() => send(p)}>
                                    {p}
                                </button>
                            ))}
                        </div>
                    </div>
                ) : (
                    messages.map((m) => (
                        <div key={m.id} className={`${styles.msgRow} ${m.role === 'user' ? styles.msgRowUser : ''}`}>
                            {m.role === 'assistant' && <div className={styles.msgAvatar}><SparkleIc /></div>}
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
                        <div className={styles.msgAvatar}><SparkleIc /></div>
                        <div className={`${styles.bubble} ${styles.bubbleAssistant} ${styles.bubbleTyping}`}>
                            <span className={styles.typingDot} /><span className={styles.typingDot} /><span className={styles.typingDot} />
                        </div>
                    </div>
                )}
            </div>

            <div className={styles.chatFooter}>
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
                        onKeyDown={(e) => { if (e.key === 'Enter' && !e.shiftKey) { e.preventDefault(); send(draft); } }}
                        rows={1}
                    />
                    <button type="button" className={styles.sendBtn} onClick={() => send(draft)} disabled={!draft.trim() || chatLoading} aria-label="Send">
                        <svg width="14" height="14" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                            <path d="M14.5 1.5L7.5 8.5M14.5 1.5L10 14.5l-2.5-6-6-2.5z" />
                        </svg>
                    </button>
                </div>
            </div>
        </div>
    );
};

// ─────────────────────────────────────────────
// Watch page
// ─────────────────────────────────────────────
// ─────────────────────────────────────────────
// Custom control bar — replaces native controls so chapter
// boundaries can be embedded directly in the scrub track, YouTube-style.
// ─────────────────────────────────────────────
const MediaControls: React.FC<{
    mediaRef: React.RefObject<HTMLVideoElement | HTMLAudioElement>;
    duration: number;
    currentTime: number;
    chapters: Chapter[];
    isVideo: boolean;
    fullscreenTargetRef?: React.RefObject<HTMLElement>;
}> = ({ mediaRef, duration, currentTime, chapters, isVideo, fullscreenTargetRef }) => {
    const [isPlaying, setIsPlaying] = useState(false);
    const [isMuted, setIsMuted] = useState(false);
    const [hoverPct, setHoverPct] = useState<number | null>(null);
    const trackRef = useRef<HTMLDivElement>(null);

    useEffect(() => {
        const el = mediaRef.current;
        if (!el) return undefined;
        const onPlay = () => setIsPlaying(true);
        const onPause = () => setIsPlaying(false);
        const onVolume = () => setIsMuted(el.muted);
        el.addEventListener('play', onPlay);
        el.addEventListener('pause', onPause);
        el.addEventListener('volumechange', onVolume);
        return () => {
            el.removeEventListener('play', onPlay);
            el.removeEventListener('pause', onPause);
            el.removeEventListener('volumechange', onVolume);
        };
    }, [mediaRef]);

    const togglePlay = () => {
        const el = mediaRef.current; if (!el) return;
        if (el.paused) el.play().catch(() => {}); else el.pause();
    };
    const toggleMute = () => {
        const el = mediaRef.current; if (!el) return;
        el.muted = !el.muted;
    };
    const toggleFullscreen = () => {
        const target = fullscreenTargetRef?.current;
        if (!target) return;
        if (document.fullscreenElement) document.exitFullscreen();
        else target.requestFullscreen?.();
    };

    const pctFromEvent = (clientX: number) => {
        const track = trackRef.current;
        if (!track || duration <= 0) return 0;
        const rect = track.getBoundingClientRect();
        return Math.min(1, Math.max(0, (clientX - rect.left) / rect.width));
    };

    const seekToClientX = (clientX: number) => {
        const el = mediaRef.current; if (!el || duration <= 0) return;
        el.currentTime = pctFromEvent(clientX) * duration;
    };

    const onTrackPointerDown = (e: React.PointerEvent<HTMLDivElement>) => {
        (e.target as HTMLElement).setPointerCapture(e.pointerId);
        seekToClientX(e.clientX);
    };
    const onTrackPointerMove = (e: React.PointerEvent<HTMLDivElement>) => {
        setHoverPct(pctFromEvent(e.clientX));
        if (e.buttons === 1) seekToClientX(e.clientX);
    };

    const progressPct = duration > 0 ? (currentTime / duration) * 100 : 0;
    const hoverTime = hoverPct !== null ? hoverPct * duration : null;
    const hoverChapter = hoverTime !== null ? chapters.find((c) => hoverTime >= c.start && hoverTime < c.end) : null;

    return (
        <div className={styles.mediaControls}>
            <div
                ref={trackRef}
                className={styles.scrubTrack}
                onPointerDown={onTrackPointerDown}
                onPointerMove={onTrackPointerMove}
                onPointerLeave={() => setHoverPct(null)}
            >
                {/* Chapter divider ticks, embedded directly in the track */}
                {chapters.slice(1).map((c, i) => (
                    <span key={i} className={styles.scrubChapterTick} style={{ left: `${(c.start / duration) * 100}%` }} />
                ))}
                <div className={styles.scrubFill} style={{ width: `${progressPct}%` }} />
                <div className={styles.scrubHandle} style={{ left: `${progressPct}%` }} />

                {hoverPct !== null && (
                    <div className={styles.scrubHoverTooltip} style={{ left: `${hoverPct * 100}%` }}>
                        {hoverChapter && <div className={styles.scrubHoverChapter}>{hoverChapter.title}</div>}
                        <div className={styles.scrubHoverTime}>{formatTime(hoverTime ?? 0)}</div>
                    </div>
                )}
            </div>

            <div className={styles.controlsRow}>
                <button type="button" className={styles.controlsBtn} onClick={togglePlay} aria-label={isPlaying ? 'Pause' : 'Play'}>
                    {isPlaying ? <PauseIc /> : <PlayIc />}
                </button>
                <span className={styles.controlsTime}>{formatTime(currentTime)} / {formatTime(duration)}</span>
                <div className={styles.controlsSpacer} />
                <button type="button" className={styles.controlsBtn} onClick={toggleMute} aria-label={isMuted ? 'Unmute' : 'Mute'}>
                    {isMuted ? <MuteIc /> : <VolumeIc />}
                </button>
                {isVideo && (
                    <button type="button" className={styles.controlsBtn} onClick={toggleFullscreen} aria-label="Fullscreen">
                        <FullscreenIc />
                    </button>
                )}
            </div>
        </div>
    );
};

const WatchPage: React.FC<{ item: LibraryItem; onBack: () => void }> = ({ item, onBack }) => {
    const [mediaUrl, setMediaUrl] = useState('');
    const [mediaLoading, setMediaLoading] = useState(false);
    const [mediaError, setMediaError] = useState('');
    const [currentTime, setCurrentTime] = useState(0);
    const [duration, setDuration] = useState(0);
    const mediaRef = useRef<HTMLVideoElement | HTMLAudioElement>(null);
    const playerFrameRef = useRef<HTMLDivElement>(null);

    useEffect(() => {
        const controller = new AbortController();
        let revokedUrl: string | null = null;
        setMediaLoading(true); setMediaError(''); setMediaUrl(''); setCurrentTime(0); setDuration(0);
        fetchMediaBlobUrl(item.id, controller.signal)
            .then((url) => { revokedUrl = url; setMediaUrl(url); setMediaLoading(false); })
            .catch((err) => {
                if (controller.signal.aborted) return;
                setMediaError(err instanceof Error ? err.message : 'Could not load media');
                setMediaLoading(false);
            });
        return () => { controller.abort(); if (revokedUrl) URL.revokeObjectURL(revokedUrl); };
    }, [item.id]);

    const chapters = useMemo(() => buildChapters(duration, item.id), [duration, item.id]);

    return (
        <div className={styles.watchPage}>
            <button type="button" className={styles.watchBackBtn} onClick={onBack}>
                <BackIc /> Back to browse
            </button>

            <div className={styles.watchGrid}>
                {/* Main column — scrolls independently if content overflows */}
                <div className={styles.watchMain}>
                    <div
                        ref={playerFrameRef}
                        className={`${styles.playerFrame} ${item.mediaKind === 'audio' ? styles.playerFrameAudio : ''}`}
                    >
                        {item.mediaKind === 'audio' ? (
                            <div className={styles.audioStage}>
                                <div className={styles.audioIcBig}><AudioIc /></div>
                                <audio
                                    key={item.id}
                                    ref={mediaRef as React.RefObject<HTMLAudioElement>}
                                    src={mediaUrl || undefined}
                                    onTimeUpdate={(e) => setCurrentTime(e.currentTarget.currentTime)}
                                    onLoadedMetadata={(e) => setDuration(e.currentTarget.duration || 0)}
                                />
                            </div>
                        ) : (
                            <video
                                key={item.id}
                                ref={mediaRef as React.RefObject<HTMLVideoElement>}
                                src={mediaUrl || undefined}
                                onTimeUpdate={(e) => setCurrentTime(e.currentTarget.currentTime)}
                                onLoadedMetadata={(e) => setDuration(e.currentTarget.duration || 0)}
                                onClick={() => (mediaRef.current?.paused ? mediaRef.current.play().catch(() => {}) : mediaRef.current?.pause())}
                            />
                        )}
                        {mediaLoading && <div className={styles.mediaLoadingOverlay}><div className={styles.spinner} /><span>Loading media…</span></div>}
                        {mediaError && <div className={styles.mediaErrorOverlay}><div>Could not load media</div><div className={styles.mediaErrorMsg}>{mediaError}</div></div>}

                        {!mediaLoading && !mediaError && (
                            <MediaControls
                                mediaRef={mediaRef}
                                duration={duration}
                                currentTime={currentTime}
                                chapters={chapters}
                                isVideo={item.mediaKind === 'video'}
                                fullscreenTargetRef={playerFrameRef}
                            />
                        )}
                    </div>

                    <div className={styles.watchTitle}>{item.original_name}</div>
                    <div className={styles.watchMetaRow}>
                        <span>Uploaded {formatDate(item.inserted_at)}</span>
                        <span className={styles.cardKindTag}>{item.mediaKind === 'audio' ? 'Audio' : 'Video'}</span>
                    </div>
                </div>

                {/* Sidebar: chat only, fills the column height, no page scroll */}
                <div className={styles.watchSide}>
                    <ChatPanel fileId={item.id} currentTime={currentTime} />
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

    useEffect(() => {
        if (!libraryLoaded && !libraryLoading) dispatch(loadLibrary());
    }, [dispatch, libraryLoaded, libraryLoading]);

    const isSearching = query.trim().length > 0;

    const searchResults = useMemo(() => {
        const q = query.trim().toLowerCase();
        if (!q) return [];
        // TODO: replace with the real search algorithm/backend later.
        // For now: client-side match over the already-loaded library.
        return library.filter((f) => f.original_name.toLowerCase().includes(q));
    }, [library, query]);

    const trending = useMemo(() => pickTrending(library, 24), [library]);

    const selectedItem = useMemo(() => library.find((f) => f.id === selectedFileId) ?? null, [library, selectedFileId]);

    const openItem = (item: LibraryItem) => dispatch(selectFile(item.id));
    const backToBrowse = () => dispatch(selectFile(null));

    if (selectedItem) {
        return (
            <div className={`${styles.page} ${styles.pageWatch}`}>
                <WatchPage item={selectedItem} onBack={backToBrowse} />
            </div>
        );
    }

    const gridItems = isSearching ? searchResults : trending;
    const gridLabel = isSearching ? `Search results for "${query.trim()}"` : 'Recommended';

    return (
        <div className={styles.page}>

            {/* ── Header ── */}
            <div className={styles.ph}>
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

.pageWatch {
    overflow: hidden;
    display: flex;
    flex-direction: column;
}

// ── Header ───────────────────────────────────────
.ph {
    position: sticky;
    top: 0;
    z-index: 2;
    padding: 20px 24px;
    background: var(--bg1);
    border-bottom: 1px solid var(--bdr);
    display: flex;
    justify-content: center;
}

.searchRow {
    position: relative;
    width: 100%;
    max-width: 520px;
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
    flex: 1;
    min-height: 0;
    display: flex;
    flex-direction: column;
    padding: 14px 24px 16px;
}

.watchBackBtn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    margin-bottom: 14px;
    padding: 7px 14px 7px 10px;
    border-radius: 99px;
    border: 1px solid var(--bdr2);
    background: var(--bg2);
    color: var(--t1);
    font-family: var(--font-ui);
    font-size: 12px;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.12s;
    flex-shrink: 0;
    align-self: flex-start;

    svg {
        width: 14px;
        height: 14px;
        flex-shrink: 0;
    }

    &:hover {
        border-color: var(--bdr3);
        color: var(--t0);
        background: var(--bg3);
    }
}

.watchGrid {
    flex: 1;
    min-height: 0;
    display: flex;
    gap: 20px;
    align-items: stretch;
}

.watchMain {
    flex: 1.7;
    min-width: 0;
    overflow-y: auto;
    padding-right: 4px;
    @include m.scrollbar;
}

.watchSide {
    flex: 1;
    min-width: 340px;
    max-width: 440px;
    min-height: 0;
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
    padding-top: 40%;
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
    padding: 0 24px 56px;
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

// ── Custom media control bar (overlaid on the player, YouTube-style) ──
.mediaControls {
    position: absolute;
    left: 0;
    right: 0;
    bottom: 0;
    z-index: 3;
    padding: 22px 12px 8px;
    background: linear-gradient(to top, rgba(0, 0, 0, 0.82) 0%, rgba(0, 0, 0, 0.5) 55%, rgba(0, 0, 0, 0) 100%);
}

.scrubTrack {
    position: relative;
    height: 4px;
    border-radius: 3px;
    background: rgba(255, 255, 255, 0.28);
    cursor: pointer;
    touch-action: none;
    transition: height 0.1s;

    &:hover { height: 6px; }
}

.scrubChapterTick {
    position: absolute;
    top: 0;
    width: 2px;
    height: 100%;
    background: rgba(0, 0, 0, 0.5);
    transform: translateX(-1px);
    pointer-events: none;
}

.scrubFill {
    position: absolute;
    inset: 0 auto 0 0;
    border-radius: 3px;
    background: var(--blue);
    pointer-events: none;
}

.scrubHandle {
    position: absolute;
    top: 50%;
    width: 12px;
    height: 12px;
    border-radius: 50%;
    background: var(--blue);
    border: 2px solid #fff;
    box-shadow: 0 1px 4px rgba(0, 0, 0, 0.4);
    transform: translate(-50%, -50%);
    pointer-events: none;
}

.scrubHoverTooltip {
    position: absolute;
    bottom: 16px;
    transform: translateX(-50%);
    background: rgba(20, 20, 20, 0.95);
    color: #fff;
    padding: 5px 9px;
    border-radius: 7px;
    font-size: 10.5px;
    white-space: nowrap;
    pointer-events: none;
    text-align: center;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.35);
}

.scrubHoverChapter {
    font-weight: 700;
}

.scrubHoverTime {
    @include m.mono;
    opacity: 0.75;
    font-size: 9.5px;
}

.controlsRow {
    display: flex;
    align-items: center;
    gap: 2px;
    margin-top: 6px;
}

.controlsBtn {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 30px;
    height: 30px;
    border-radius: 7px;
    border: none;
    background: transparent;
    color: #fff;
    cursor: pointer;
    transition: all 0.12s;

    &:hover { background: rgba(255, 255, 255, 0.15); }
}

.controlsTime {
    font-size: 11px;
    color: rgba(255, 255, 255, 0.85);
    margin-left: 4px;
    @include m.mono;
}

.controlsSpacer {
    flex: 1;
}

// ── Chat panel — the whole sidebar now ───────────
.chatPanel {
    display: flex;
    flex-direction: column;
    height: 100%;
    background: var(--bg1);
    border: 1px solid var(--bdr);
    border-radius: 16px;
    overflow: hidden;
    box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
}

.chatHead {
    display: flex;
    align-items: center;
    gap: 11px;
    padding: 16px 18px;
    border-bottom: 1px solid var(--bdr);
    flex-shrink: 0;
    background: linear-gradient(180deg, var(--bg2), var(--bg1));
}

.chatHeadIc {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 34px;
    height: 34px;
    border-radius: 10px;
    flex-shrink: 0;
    background: linear-gradient(135deg, var(--blue), #7c5cff);
    color: #fff;
    box-shadow: 0 3px 12px rgba(76, 120, 255, 0.35);

    svg { width: 17px; height: 17px; }
}

.chatHeadTitle {
    font-size: 14px;
    font-weight: 700;
    color: var(--t0);
    letter-spacing: -0.1px;
}

.chatHeadSub {
    display: flex;
    align-items: center;
    gap: 5px;
    font-size: 10.5px;
    color: var(--t2);
    margin-top: 2px;
}

.chatHeadBetaTag {
    display: inline-flex;
    align-items: center;
    padding: 1px 6px;
    border-radius: 5px;
    font-size: 9px;
    font-weight: 700;
    letter-spacing: 0.03em;
    background: var(--amber-dim);
    color: var(--amber);
}

.chatBody {
    flex: 1;
    overflow-y: auto;
    padding: 16px 16px 8px;
    display: flex;
    flex-direction: column;
    gap: 14px;
    min-height: 80px;
    @include m.scrollbar;
}

.chatEmptyState {
    height: 100%;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    text-align: center;
    gap: 4px;
    padding: 12px;
}

.chatEmptyIc {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 46px;
    height: 46px;
    border-radius: 50%;
    background: linear-gradient(135deg, var(--blue-dim), rgba(124, 92, 255, 0.14));
    color: var(--blue);
    margin-bottom: 8px;

    svg { width: 20px; height: 20px; }
}

.chatEmptyTitle {
    font-size: 13.5px;
    font-weight: 700;
    color: var(--t0);
}

.chatEmptyText {
    font-size: 11.5px;
    color: var(--t2);
    margin-top: 2px;
}

.suggestedPrompts {
    display: flex;
    flex-direction: column;
    gap: 7px;
    width: 100%;
    max-width: 270px;
    margin-top: 18px;
}

.suggestedChip {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 10px 13px;
    border-radius: 10px;
    border: 1px solid var(--bdr2);
    background: var(--bg2);
    color: var(--t1);
    font-family: var(--font-ui);
    font-size: 11.5px;
    font-weight: 500;
    text-align: left;
    cursor: pointer;
    transition: all 0.14s;

    &:hover {
        border-color: var(--blue-bdr);
        background: var(--blue-dim);
        color: var(--blue);
        transform: translateX(2px);
    }
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

.msgRow { display: flex; justify-content: flex-start; gap: 8px; }
.msgRowUser { justify-content: flex-end; }

.msgAvatar {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 24px;
    height: 24px;
    border-radius: 50%;
    flex-shrink: 0;
    background: linear-gradient(135deg, var(--blue), #7c5cff);
    color: #fff;
    margin-top: 2px;
    box-shadow: 0 2px 6px rgba(76, 120, 255, 0.3);

    svg { width: 12px; height: 12px; }
}

.bubble {
    max-width: 80%;
    padding: 10px 13px;
    border-radius: 14px;
    font-size: 12.5px;
    line-height: 1.55;
    white-space: pre-wrap;
}

.bubbleAssistant {
    background: var(--bg2);
    color: var(--t0);
    border: 1px solid var(--bdr2);
    border-bottom-left-radius: 4px;
}

.bubbleUser {
    background: linear-gradient(135deg, var(--blue), #4f6bff);
    color: #fff;
    border-bottom-right-radius: 4px;
    box-shadow: 0 2px 8px rgba(76, 120, 255, 0.25);
}

.bubbleTyping {
    display: flex;
    align-items: center;
    gap: 4px;
    padding: 11px 13px;
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

.chatFooter {
    flex-shrink: 0;
    padding: 10px 14px 14px;
    border-top: 1px solid var(--bdr);
    background: var(--bg1);
}

.chatToolsRow {
    display: flex;
    padding-bottom: 8px;
}

.timestampChip {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 4px 10px;
    border-radius: 99px;
    border: 1px solid var(--bdr2);
    background: var(--bg2);
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
    padding: 6px 6px 6px 14px;
    border-radius: 14px;
    background: var(--bg2);
    border: 1px solid var(--bdr2);
    transition: border-color 0.14s, box-shadow 0.14s;

    &:focus-within {
        border-color: var(--blue-bdr);
        box-shadow: 0 0 0 3px var(--blue-dim);
    }
}

.chatInput {
    flex: 1;
    resize: none;
    background: transparent;
    border: none;
    color: var(--t0);
    font-family: var(--font-ui);
    font-size: 12.5px;
    padding: 8px 0;
    outline: none;
    line-height: 1.4;

    &::placeholder { color: var(--t2); }
}

.sendBtn {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 32px;
    height: 32px;
    flex-shrink: 0;
    margin-bottom: 3px;
    border-radius: 50%;
    border: none;
    background: linear-gradient(135deg, var(--blue), #4f6bff);
    color: #fff;
    cursor: pointer;
    transition: all 0.14s;
    box-shadow: 0 2px 8px rgba(76, 120, 255, 0.3);

    &:hover:not(:disabled) { transform: scale(1.06); }
    &:disabled { opacity: 0.4; cursor: default; box-shadow: none; }
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
