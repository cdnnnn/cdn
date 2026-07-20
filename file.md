// ═══════════════════════════════════════════════
// pages/SttTranscription/components/InferencePanel.tsx
// ═══════════════════════════════════════════════
import React, { useEffect, useRef, useState } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppDispatch, useAppSelector } from '../../../store/hooks';
import {
    submitBatchInference,
    fetchInferenceProgress,
    fetchInferenceProgressList,
    closeInference as _unused,
    fetchSttFiles,
    clearSttFiles,
    fetchSttModels,
    clearInferenceError,
} from '../../../store/sttSlice';
import styles from '../SttTranscription.module.scss';

interface Props {
    selectedIds: Set<number>;
    onSubmitted: () => void;
    onCompleted: () => void; // called when inference finishes — clears selection, no panel to close
}

const IconX: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
        <path d="M4 4l8 8M12 4l-8 8" />
    </svg>
);
const IconSpark: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <path d="M8 1.5l1.2 3 3 1.2-3 1.2L8 10l-1.2-3.1-3-1.2 3-1.2z" />
        <path d="M12.5 10l.7 1.8 1.8.7-1.8.7-.7 1.8-.7-1.8-1.8-.7 1.8-.7z" />
    </svg>
);

const CHUNK_OPTIONS = [
    { value: 'auto', labelKey: 'stt.inference.chunkAuto' },
    { value: '5',    labelKey: '5s'  },
    { value: '10',   labelKey: '10s' },
    { value: '15',   labelKey: '15s' },
    { value: '20',   labelKey: '20s' },
];
const POLL_INTERVAL = 10_000;

const InferencePanel: React.FC<Props> = ({ selectedIds, onSubmitted, onCompleted }) => {
    const dispatch = useAppDispatch();
    const { t } = useTranslation();

    const models        = useAppSelector(s => s.stt.models);
    const modelsLoading = useAppSelector(s => s.stt.modelsLoading);
    const dateFrom      = useAppSelector(s => s.stt.dateFrom);
    const dateTo        = useAppSelector(s => s.stt.dateTo);
    const running       = useAppSelector(s => s.stt.inferenceRunning);
    const submitting    = useAppSelector(s => s.stt.inferenceSubmitting);
    const error         = useAppSelector(s => s.stt.inferenceError);
    const queued        = useAppSelector(s => s.stt.inferenceQueued);
    const runningFiles  = useAppSelector(s => s.stt.inferenceRunningFiles);
    const completed     = useAppSelector(s => s.stt.inferenceCompleted);
    const pending       = useAppSelector(s => s.stt.inferencePending);
    const progressMap   = useAppSelector(s => s.stt.inferenceProgressMap);

    const [model, setModel]   = useState('');
    const [language, setLang] = useState('ko');
    const [chunk, setChunk]   = useState('auto');

    const pollRef      = useRef<ReturnType<typeof setInterval> | null>(null);
    const countRef     = useRef<ReturnType<typeof setInterval> | null>(null);
    const [countdown, setCountdown] = useState(0);

    useEffect(() => {
        if (models.length === 0) dispatch(fetchSttModels());
    }, [dispatch, models.length]);

    useEffect(() => {
        if (!model && models.length > 0) setModel(models[0]);
    }, [models, model]);

    // Poll /by-progress while running.
    // Stop when queued:[] AND running:[] — inference complete.
    // Then call /by-date once to refresh the library and close col-3.
    useEffect(() => {
        const tick = async () => {
            const result = await dispatch(fetchInferenceProgress({ from: dateFrom, to: dateTo }));
            if (!fetchInferenceProgress.fulfilled.match(result)) return;

            const { queued, running: runList } = result.payload;

            // Poll per-file progress for currently running files
            if (runList.length > 0) {
                dispatch(fetchInferenceProgressList(runList.map((f: any) => f.id)));
            }

            // Both empty → inference finished
            if (queued.length === 0 && runList.length === 0) {
                if (pollRef.current) { clearInterval(pollRef.current); pollRef.current = null; }
                if (countRef.current) { clearInterval(countRef.current); countRef.current = null; }
                dispatch(clearSttFiles());
                await dispatch(fetchSttFiles({ from: dateFrom, to: dateTo }));
                onCompleted();
            }
        };

        if (running) {
            setCountdown(POLL_INTERVAL / 1000);
            tick();
            pollRef.current = setInterval(() => { setCountdown(POLL_INTERVAL / 1000); tick(); }, POLL_INTERVAL);
            countRef.current = setInterval(() => setCountdown(prev => Math.max(0, prev - 1)), 1000);
        } else {
            if (pollRef.current)  { clearInterval(pollRef.current);  pollRef.current  = null; }
            if (countRef.current) { clearInterval(countRef.current); countRef.current = null; }
            setCountdown(0);
        }
        return () => {
            if (pollRef.current)  { clearInterval(pollRef.current);  pollRef.current  = null; }
            if (countRef.current) { clearInterval(countRef.current); countRef.current = null; }
        };
    }, [running, dispatch, dateFrom, dateTo]);

    const handleSubmit = async () => {
        if (selectedIds.size === 0 || !model || submitting) return;
        dispatch(clearInferenceError());
        const result = await dispatch(submitBatchInference({
            fileIds: Array.from(selectedIds),
            modelName: model,
            language,
            chunkSize: chunk === 'auto' ? 0 : Number(chunk),
        }));
        if (submitBatchInference.fulfilled.match(result)) {
            onSubmitted(); // clear selection in parent
        } else {
            // Non-200 from /batch_process — fire /by-progress once (no await, fire-and-forget)
            dispatch(fetchInferenceProgress({ from: dateFrom, to: dateTo }));
        }
    };

    const progressOf = (id: number) => progressMap[id] ?? 0;
    const statusLabel = (id: number) => {
        const p = progressOf(id);
        if (p === 100) return 'Done';
        if (p === -1)  return 'Failed';
        if (p > 0)     return `${p}%`;
        return 'Starting…';
    };

    return (
        <div className={styles.inferencePanel}>
            {/* Heading — countdown badge only; no close button since this is a persistent tab, not a col-3 panel */}
            {running && countdown > 0 && (
                <div className={styles.colHeading}>
                    <span className={styles.colHeadingIc}><IconSpark /></span>
                    <span className={styles.colHeadingText}>{t('stt.inference.title')}</span>
                    <span className={styles.infCountdownBadge}>⟳ next check in {countdown}s</span>
                </div>
            )}

            {/* CONFIG VIEW */}
            {!running && (
                <div className={styles.infColBody}>
                    {/* Selected files count */}
                    <div className={styles.infSelectedHint} data-tour="infer-settings">
                        {selectedIds.size === 0
                            ? 'Select files from the sidebar to run inference on.'
                            : `${selectedIds.size} file${selectedIds.size === 1 ? '' : 's'} selected`}
                    </div>

                    <div className={styles.infSection} data-tour="infer-model">
                        <div className={styles.infSectionTitle}>Settings</div>

                        <div className={styles.infField}>
                            <label className={styles.infLabel}>Model</label>
                            {modelsLoading
                                ? <div className={styles.infHint}>Loading…</div>
                                : <select className={styles.infSelect} value={model} onChange={e => setModel(e.target.value)}>
                                    {models.map(m => <option key={m} value={m}>{m}</option>)}
                                  </select>
                            }
                        </div>

                        <div className={styles.infField}>
                            <label className={styles.infLabel}>Language</label>
                            <select className={styles.infSelect} value={language} onChange={e => setLang(e.target.value)}>
                                <option value="ko">Korean</option>
                                <option value="en">English</option>
                            </select>
                        </div>

                        <div className={styles.infField}>
                            <label className={styles.infLabel}>Chunk size</label>
                            <select className={styles.infSelect} value={chunk} onChange={e => setChunk(e.target.value)}>
                                {CHUNK_OPTIONS.map(c => <option key={c.value} value={c.value}>{t(c.labelKey)}</option>)}
                            </select>
                        </div>
                    </div>

                    {error && <div className={styles.infError}>{error}</div>}

                    <div className={styles.infActions} data-tour="infer-run">
                        <button
                            className={`${styles.btn} ${styles.btnBlue}`}
                            disabled={selectedIds.size === 0 || !model || submitting}
                            onClick={handleSubmit}
                        >
                            {submitting ? 'Submitting…' : `Run Inference${selectedIds.size > 0 ? ` (${selectedIds.size})` : ''}`}
                        </button>
                    </div>
                </div>
            )}

            {/* PROGRESS VIEW */}
            {running && (
                <div className={styles.infColBody}>
                    <div className={styles.infProgressGrid}>

                        {/* ── Queued group ── */}
                        <div className={`${styles.infProgressCol} ${styles.infColPending}`}>
                            <div className={styles.infProgressColHead}>
                                <div className={styles.infProgressColIcon}>
                                    <svg viewBox="0 0 16 16" fill="none" strokeWidth="1.5" strokeLinecap="round" stroke="var(--t2)">
                                        <circle cx="8" cy="8" r="5.5" /><path d="M8 5v3.5l2 1.5" />
                                    </svg>
                                </div>
                                <span>Queued</span>
                                <span className={styles.infProgressCount}>{queued.length} file{queued.length !== 1 ? 's' : ''}</span>
                                <span className={`${styles.infProgressBadge} ${styles.infBadgePending}`}>Waiting</span>
                            </div>
                            <div className={styles.infProgressBody}>
                                {queued.length === 0
                                    ? <div className={styles.infProgressEmpty}>No queued files</div>
                                    : queued.map(f => (
                                        <div key={f.id} className={styles.infProgressCard}>
                                            <div className={styles.infProgressIcon}>STT</div>
                                            <div className={styles.infProgressName} title={f.original_name}>{f.original_name}</div>
                                            <span className={`${styles.infProgressBadge} ${styles.infBadgePending}`} style={{ marginLeft: 'auto' }}>In queue</span>
                                        </div>
                                    ))
                                }
                            </div>
                        </div>

                        {/* ── Running group ── */}
                        <div className={`${styles.infProgressCol} ${styles.infColRunning}`}>
                            <div className={styles.infProgressColHead}>
                                <div className={styles.infProgressColIcon}>
                                    <svg viewBox="0 0 16 16" fill="none" strokeWidth="1.5" strokeLinecap="round" stroke="var(--amber)">
                                        <circle cx="8" cy="8" r="6" /><path d="M8 5v4M8 11v.1" />
                                    </svg>
                                </div>
                                <span>Running</span>
                                <span className={styles.infProgressCount}>{runningFiles.length} file{runningFiles.length !== 1 ? 's' : ''}</span>
                                <span className={`${styles.infProgressBadge} ${styles.infBadgeRunning}`}>
                                    <span className={styles.infBadgeDot} />Active
                                </span>
                            </div>
                            <div className={styles.infProgressBody}>
                                {runningFiles.length === 0
                                    ? <div className={styles.infProgressEmpty}>No active files</div>
                                    : runningFiles.map(f => {
                                        const pct = progressOf(f.id);
                                        return (
                                            <div key={f.id} className={`${styles.infProgressCard} ${styles.infProgressCardRunning}`}>
                                                <div className={styles.infProgressIcon}>STT</div>
                                                <div className={styles.infProgressName} title={f.original_name}>{f.original_name}</div>
                                                <div className={styles.infProgressBar}>
                                                    <div className={styles.infProgressBarTrack}>
                                                        <div className={styles.infProgressFill} style={{ width: `${Math.max(3, pct)}%` }} />
                                                    </div>
                                                </div>
                                                <div className={styles.infProgressPct}>{statusLabel(f.id)}</div>
                                            </div>
                                        );
                                    })
                                }
                            </div>
                        </div>

                        {/* ── Done group — mirrors .gDone ── */}
                        <div className={`${styles.infProgressCol} ${styles.infColDone}`}>
                            <div className={styles.infProgressColHead}>
                                <div className={styles.infProgressColIcon}>
                                    <svg viewBox="0 0 16 16" fill="none" strokeWidth="1.5" strokeLinecap="round" stroke="var(--green)">
                                        <circle cx="8" cy="8" r="5.5" /><path d="M5.5 8l2 2 3-3" />
                                    </svg>
                                </div>
                                <span>Done</span>
                                <span className={styles.infProgressCount}>{completed.length} file{completed.length !== 1 ? 's' : ''}</span>
                                <span className={`${styles.infProgressBadge} ${styles.infBadgeDone}`}>
                                    <span className={styles.infBadgeDot} />
                                    Done
                                </span>
                            </div>
                            <div className={styles.infProgressBody}>
                                {completed.length === 0
                                    ? <div className={styles.infProgressEmpty}>No completed files yet</div>
                                    : completed.map(f => (
                                        <div key={f.id} className={`${styles.infProgressCard} ${styles.infProgressCardDone}`}>
                                            <div className={styles.infProgressIcon}>STT</div>
                                            <div className={styles.infProgressName} title={f.original_name}>{f.original_name}</div>
                                            <span className={`${styles.infProgressBadge} ${styles.infBadgeDone}`} style={{ marginLeft: 'auto' }}>
                                                <span className={styles.infBadgeDot} />Done
                                            </span>
                                        </div>
                                    ))
                                }
                            </div>
                        </div>

                    </div>
                </div>
            )}
        </div>
    );
};

export default InferencePanel;



















{
  "_comment": "Add these keys inside the existing 'stt' object in en.json",
  "stt": {
    "inference": {
      "chunkAuto": "Auto"
    }
  }
}












{
  "_comment": "Add these keys inside the existing 'stt' object in ko.json",
  "stt": {
    "inference": {
      "chunkAuto": "자동"
    }
  }
}
