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

    // The /get_captions response now includes `is_vtt` alongside `lines`,
    // telling us which format the file's captions actually came from.
    // NOTE: cast to `any` here because this field may not yet be reflected
    // in sttSlice's CaptionsEntry type — add `is_vtt: boolean` there and
    // this can drop the cast.
    const isVtt = !!(captionsEntry as any)?.is_vtt;

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
