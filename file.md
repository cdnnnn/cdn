// ═══════════════════════════════════════════════
// pages/UploadInfer/InferencePanel.tsx
// Content Analytics · Inference configuration + batch status
// ═══════════════════════════════════════════════
import React, { useEffect, useRef, useCallback, useState } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import {
  inferenceStatusSuccess, updateRunningProgress,
  updateSummaryPrompt, updateKeywordPrompt, updateQuestionPrompt,
  updateShortAnswerPrompt, updateTrueFalsePrompt, updateSettings,
  modelsLoading, modelsSuccess, modelsFailure, setSelectedModel,
  updateFilePrompts,
  type ServerFile, type ServerFilesData, type TimeInterval,
} from '../../store/uploadSlice';
import api from '../../services/api';
import styles from './InferencePanel.module.scss';
import { addToast } from '../../store/toastSlice';
import PromptTemplateAssociationModal from './PromptTemplateAssociationModal';

// ── Batch status columns ──────────────────────────
const StatusCard: React.FC<{ file: ServerFile; variant: 'queued' | 'running' | 'completed'; onStop?: (id: number) => void; stopping?: boolean }> = ({ file, variant, onStop, stopping }) => {
  const { t } = useTranslation();
  const ext = file.original_name.toLowerCase().endsWith('.srt') ? 'srt' : 'vtt';
  return (
    <div className={`${styles.statusCardWrap} ${styles[variant + 'Wrap']}`}>
      <div className={`${styles.statusCard} ${styles[variant]}`}>

        {/* Left icon — ext badge for queued/running, green check for completed */}
        {variant === 'completed' ? (
          <div className={styles.completedCheck}>
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
              <path d="M3 8l3.5 3.5L13 5" />
            </svg>
          </div>
        ) : (
          <div className={`${styles.statusExt} ${styles[ext]}`}>{ext.toUpperCase()}</div>
        )}

        <div className={styles.statusInfo}>
          <div className={styles.statusNameRow}>
            <span className={`${styles.statusIdBadge} ${styles['statusIdBadge_' + variant]}`}>#{file.id}</span>
            <div className={styles.statusName}>{file.original_name}</div>
          </div>

          {variant === 'running' && typeof file.progress === 'number' && (
            <div className={styles.statusProgress}>
              <div className={styles.statusBar}>
                <div className={styles.statusFill} style={{ width: `${file.progress}%` }} />
              </div>
              <span className={styles.statusPct}>{file.progress}%</span>
            </div>
          )}

          {variant === 'running' && (file.elapsed_time || file.eta) && (
            <div className={styles.statusTimeMeta}>
              {file.elapsed_time && (
                <span className={styles.statusTimeItem}>
                  <svg viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="1.3" strokeLinecap="round" strokeLinejoin="round">
                    <circle cx="6" cy="6" r="4.75" /><path d="M6 3.5V6l1.8 1" />
                  </svg>
                  {t('uploadInfer.inferencePanel.elapsedTime')} {file.elapsed_time}
                </span>
              )}
              {file.eta && (
                <span className={styles.statusTimeItem}>
                  <svg viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="1.3" strokeLinecap="round" strokeLinejoin="round">
                    <path d="M6 1.5v2M6 8.5v2M1.5 6h2M8.5 6h2" /><circle cx="6" cy="6" r="3" />
                  </svg>
                  {t('uploadInfer.inferencePanel.eta')} {file.eta}
                </span>
              )}
            </div>
          )}

          {variant === 'queued' && (
            <div className={styles.queuedMeta}>
              <span className={styles.queuedDot} />
              {t('uploadInfer.inferencePanel.waitingQueue')}
              {file.elapsed_time && (
                <span className={styles.statusTimeItem}>
                  <svg viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="1.3" strokeLinecap="round" strokeLinejoin="round">
                    <circle cx="6" cy="6" r="4.75" /><path d="M6 3.5V6l1.8 1" />
                  </svg>
                  {t('uploadInfer.inferencePanel.elapsedTime')} {file.elapsed_time}
                </span>
              )}
            </div>
          )}

          {variant === 'completed' && (
            <div className={styles.completedMeta}>{file.inserted_at}</div>
          )}

          {variant === 'running' && typeof file.progress !== 'number' && (
            <div className={styles.statusDate}>{file.inserted_at}</div>
          )}
        </div>

        {/* Stop button — queued and running files */}
        {(variant === 'queued' || variant === 'running') && onStop && (
          <button
            className={styles.stopBtn}
            onClick={() => onStop(file.id)}
            disabled={stopping}
            title={t('uploadInfer.inferencePanel.stopFile')}
          >
            {stopping ? (
              <span className={styles.stopSpinner} />
            ) : (
              <svg viewBox="0 0 16 16" fill="currentColor" stroke="none">
                <rect x="4" y="4" width="8" height="8" rx="1.5" />
              </svg>
            )}
          </button>
        )}

      </div>
    </div>
  );
};

// ── InferencePanel ───────────────────────────────
interface InferencePanelProps {
  onClose?: () => void;
  minimized?: boolean;
  onToggleMinimize?: () => void;
}

const InferencePanel: React.FC<InferencePanelProps> = ({ onClose, minimized = false, onToggleMinimize }) => {
  const { t } = useTranslation();
  const dispatch = useAppDispatch();
  const {
    settings,
    selectedServerIds, isBatchRunning,
    selectFiles,
    models, modelsLoading: mlLoading, selectedModel,
    dateFrom, dateTo,
  } = useAppSelector(s => s.upload);

  const [running, setRunning] = React.useState(false);
  const [stoppingIds, setStoppingIds] = React.useState<Set<number>>(new Set());

  // ── Prompt template association modal — moved here from the Upload &
  // Manage tab, since mapping a template is something you'd do right
  // before running inference. ──
  const [templateModalOpen, setTemplateModalOpen] = useState(false);

  // ── Prompt save state ──────────────────────────
  const [promptSaving, setPromptSaving] = useState(false);
  const [promptSaveError, setPromptSaveError] = useState<string | null>(null);
  const promptSnapshot = useRef({ summary: '', keyword: '', question: '', shortAnswer: '', trueFalse: '' });
  const promptsDirty =
    settings.summaryPromptOverride !== promptSnapshot.current.summary ||
    settings.keywordPromptOverride !== promptSnapshot.current.keyword ||
    settings.questionPromptOverride !== promptSnapshot.current.question ||
    settings.shortAnswerPromptOverride !== promptSnapshot.current.shortAnswer ||
    settings.trueFalsePromptOverride !== promptSnapshot.current.trueFalse;

  const emptyBatch: ServerFilesData = { queued: [], running: [], completed: [], pending: [] };
  const [batchData, setBatchData] = React.useState<ServerFilesData>(emptyBatch);
  const [countdown, setCountdown] = React.useState(10);
  const countdownRef = useRef<ReturnType<typeof setInterval> | null>(null);
  const n = selectedServerIds.length;
  // When nothing is selected, every generation checkbox should read as
  // unchecked and be non-interactive — there's nothing to configure yet.
  // Settings themselves aren't reset (so a prior selection is restored if
  // the user re-selects the same files), only the display/interaction is.
  const noFilesSelected = n === 0;
  const isChecked = (v: boolean) => v && !noFilesSelected;
  const canRunInference = n > 0 && !isBatchRunning && !!selectedModel;
  const pollingRef = useRef<ReturnType<typeof setInterval> | null>(null);

  // ── Fetch models ────────────────────────────
  // In-flight guard: set synchronously before the await, so a second
  // overlapping call (e.g. StrictMode's intentional double mount-effect
  // invocation in dev) bails out instead of firing a second real request.
  const modelsFetchInFlightRef = useRef(false);
  const fetchModels = useCallback(async () => {
    if (modelsFetchInFlightRef.current) return;
    modelsFetchInFlightRef.current = true;
    dispatch(modelsLoading());
    try {
      const res = await api.get('/get_models');
      const result = (res.data as any)?.result ?? [];
      dispatch(modelsSuccess(result));
    } catch {
      dispatch(modelsFailure());
    } finally {
      modelsFetchInFlightRef.current = false;
    }
  }, [dispatch]); // eslint-disable-line

  useEffect(() => { fetchModels(); }, []); // eslint-disable-line

  // ── Polling helper ───────────────────────────
  // Same in-flight guard as fetchModels — this is the one that's called
  // from two different mount-time effects (the isBatchRunning effect and
  // the bootstrap effect below), so it's the most exposed to duplicate
  // overlapping calls.
  const pollFetchInFlightRef = useRef(false);
  const fetchFilesForPolling = useCallback(async () => {
    if (pollFetchInFlightRef.current) return false;
    pollFetchInFlightRef.current = true;
    try {
      const statusRes = await api.post('/files/by-progress/', { start_date: dateFrom, end_date: dateTo });
      const d = (statusRes.data as any)?.data;
      const data: ServerFilesData = {
        queued: d?.queued ?? [],
        completed: d?.completed ?? [],
        pending: d?.pending ?? [],
        running: d?.running ?? [],
      };
      dispatch(inferenceStatusSuccess(data));
      setBatchData(data);

      const stillRunning = data.running.length > 0 || data.queued.length > 0;

      if (data.running.length > 0) {
        const runningIds = data.running.map(f => f.id);
        try {
          const progressRes = await api.post('/files/progress/', { file_ids: runningIds });
          const progressMap = (progressRes.data as any)?.result ?? {};
          dispatch(updateRunningProgress(progressMap));
          setBatchData(prev => ({
            ...prev,
            running: prev.running.map(f => {
              const raw = progressMap[String(f.id)];
              if (raw === undefined) return f;
              const pct = typeof raw === 'string' ? parseFloat(raw) : raw;
              return { ...f, progress: isNaN(pct) ? f.progress : Math.min(100, Math.max(0, pct)) };
            }),
          }));
        } catch {
          // progress fetch failing is non-critical — silently ignore
        }
      }

      if (!stillRunning) setBatchData(emptyBatch);
      return stillRunning;
    } catch {
      return false;
    } finally {
      pollFetchInFlightRef.current = false;
    }
  }, [dispatch, dateFrom, dateTo]);

  const stopCountdown = useCallback(() => {
    if (countdownRef.current) {
      clearInterval(countdownRef.current);
      countdownRef.current = null;
    }
  }, []);

  const startCountdown = useCallback(() => {
    stopCountdown();
    setCountdown(10);
    countdownRef.current = setInterval(() => {
      setCountdown(prev => (prev <= 1 ? 10 : prev - 1));
    }, 1000);
  }, [stopCountdown]);

  const stopPolling = useCallback(() => {
    if (pollingRef.current) {
      clearInterval(pollingRef.current);
      pollingRef.current = null;
    }
    stopCountdown();
    setCountdown(10);
  }, [stopCountdown]);

  const startPolling = useCallback(() => {
    stopPolling();
    startCountdown();
    pollingRef.current = setInterval(async () => {
      const stillRunning = await fetchFilesForPolling();
      if (!stillRunning) stopPolling();
      else startCountdown();
    }, 10000);
  }, [fetchFilesForPolling, stopPolling, startCountdown]);

  useEffect(() => {
    if (isBatchRunning) {
      fetchFilesForPolling().then(stillRunning => {
        if (stillRunning) startPolling();
        else stopPolling();
      });
    } else {
      stopPolling();
    }
    return stopPolling;
  }, [isBatchRunning]); // eslint-disable-line

  // ── One-time bootstrap check on mount ──
  // /files/by-date/ is now paginated and no longer tells us whether a batch
  // is running (a running/queued file may simply be on another page), so we
  // can't rely on its response to seed isBatchRunning like before. Ask
  // /files/by-progress/ directly once on mount to catch a batch that was
  // already running before this page loaded (e.g. after a refresh).
  useEffect(() => {
    fetchFilesForPolling().then(stillRunning => { if (stillRunning) startPolling(); });
  }, []); // eslint-disable-line

  // ── Stop a queued file ───────────────────────
  const handleStop = useCallback(async (fileId: number) => {
    setStoppingIds(prev => new Set(prev).add(fileId));
    try {
      await api.patch('/files/stop', { fileID: [fileId] });
    } catch (err: any) {
      if (err?.response?.status === 403) {
        dispatch(addToast(t('uploadInfer.inferencePanel.stopAlready'), 'error'));
      } else {
        console.error('Stop file failed:', err);
      }
    } finally {
      await fetchFilesForPolling();
      setStoppingIds(prev => { const s = new Set(prev); s.delete(fileId); return s; });
    }
  }, [fetchFilesForPolling]);

  // ── Run inference ────────────────────────────
  const handleRun = async () => {
    if (!canRunInference) return;
    setRunning(true);
    try {
      await api.post('/batch_process', {
        all: false, // always false for now — file_ids-driven runs only
        file_ids: selectedServerIds,
        start_date: dateFrom,
        end_date: dateTo,
        model_name: selectedModel,
        summary_prompt: settings.summaryPromptOverride,
        faq_prompt: settings.questionPromptOverride,
        keywords_prompt: settings.keywordPromptOverride,
        short_answers_prompt: settings.shortAnswerPromptOverride,
        true_false_prompt: settings.trueFalsePromptOverride,
        generate_summary: settings.generateSummary,
        generate_keywords: settings.generateKeywords,
        generate_faq: settings.generateQuestions,
        generate_short_answer: settings.generateShortAnswer,
        generate_true_false: settings.generateTrueFalse,
        // Only meaningful — and only ever sent true — when Keywords itself is on.
        generate_keyword_insights: settings.generateKeywords && settings.generateKeywordInsights,
        timestamped_summary: settings.timestampedSummary,
        time_interval: settings.timeInterval,
      });
      const stillRunning = await fetchFilesForPolling();
      if (stillRunning) startPolling();
      dispatch(addToast(t('uploadInfer.inferencePanel.inferenceStarted'), 'success'));
    } catch (err) {
      console.error('Batch process failed:', err);
    } finally {
      setRunning(false);
    }
  };

  // ── Run button sub-label ─────────────────────
  const runBtnSubLabel = (() => {
    if (n === 0) return t('uploadInfer.inferencePanel.selectFilesFirst');
    if (!selectedModel) return t('uploadInfer.inferencePanel.noModelSelected');
    return `${n} file${n !== 1 ? 's' : ''} ready`;
  })();

  // ── Auto-fill prompts when exactly 1 file is selected ────────────
  // Keyed off the *actual selected file id* (or a stable "multi"/"none"
  // marker), not the selection count and not the selectedServerIds array
  // reference. Redux can hand us a brand-new array reference for the same
  // underlying selection on unrelated updates — keying off the array (or
  // count alone) made this effect refire and stomp on prompt edits the
  // user was still mid-typing, even though the selection hadn't changed.
  const singleSelectedId = n === 1 ? selectedServerIds[0] : null;
  const selectionKey = n === 1 ? `one:${singleSelectedId}` : n === 0 ? 'none' : 'multi';
  const prevSelectionKey = useRef<string | null>(null);
  useEffect(() => {
    if (selectionKey === prevSelectionKey.current) return;
    prevSelectionKey.current = selectionKey;

    if (n === 1) {
      const file = selectFiles.find(f => f.id === singleSelectedId);
      if (file) {
        const s = file.summary_prompt ?? '';
        const k = file.keywords_prompt ?? '';
        const q = file.faq_prompt ?? '';
        const sa = file.short_answer_prompt ?? '';
        const tf = file.true_false_prompt ?? '';
        dispatch(updateSummaryPrompt(s));
        dispatch(updateKeywordPrompt(k));
        dispatch(updateQuestionPrompt(q));
        dispatch(updateShortAnswerPrompt(sa));
        dispatch(updateTrueFalsePrompt(tf));
        promptSnapshot.current = { summary: s, keyword: k, question: q, shortAnswer: sa, trueFalse: tf };
      }
    } else {
      dispatch(updateSummaryPrompt(''));
      dispatch(updateKeywordPrompt(''));
      dispatch(updateQuestionPrompt(''));
      dispatch(updateShortAnswerPrompt(''));
      dispatch(updateTrueFalsePrompt(''));
      promptSnapshot.current = { summary: '', keyword: '', question: '', shortAnswer: '', trueFalse: '' };
    }
    setPromptSaveError(null);
  }, [selectionKey]); // eslint-disable-line

  // ── Per-file prompt preview expand/collapse ──
  const [summaryExpanded, setSummaryExpanded] = useState(false);
  const [keywordExpanded, setKeywordExpanded] = useState(false);
  const [questionExpanded, setQuestionExpanded] = useState(false);
  const [shortAnswerExpanded, setShortAnswerExpanded] = useState(false);
  const [trueFalseExpanded, setTrueFalseExpanded] = useState(false);

  const prevSelectedCount = useRef(selectedServerIds.length);
  useEffect(() => {
    if (selectedServerIds.length !== prevSelectedCount.current) {
      prevSelectedCount.current = selectedServerIds.length;
      setSummaryExpanded(false);
      setKeywordExpanded(false);
      setQuestionExpanded(false);
      setShortAnswerExpanded(false);
      setTrueFalseExpanded(false);
    }
  });

  const selectedFiles = selectFiles.filter(f => selectedServerIds.includes(f.id));

  const handlePromptCancel = () => {
    dispatch(updateSummaryPrompt(promptSnapshot.current.summary));
    dispatch(updateKeywordPrompt(promptSnapshot.current.keyword));
    dispatch(updateQuestionPrompt(promptSnapshot.current.question));
    dispatch(updateShortAnswerPrompt(promptSnapshot.current.shortAnswer));
    dispatch(updateTrueFalsePrompt(promptSnapshot.current.trueFalse));
    setPromptSaveError(null);
  };

  const handlePromptSave = async () => {
    setPromptSaving(true);
    setPromptSaveError(null);
    try {
      const payload: Record<string, unknown> = {
        file_ids: selectedServerIds,
      };
      if (settings.generateSummary) payload.summary_prompt = settings.summaryPromptOverride;
      if (settings.generateKeywords) payload.keywords_prompt = settings.keywordPromptOverride;
      if (settings.generateQuestions) payload.faq_prompt = settings.questionPromptOverride;
      if (settings.generateShortAnswer) payload.short_answers_prompt = settings.shortAnswerPromptOverride;
      if (settings.generateTrueFalse) payload.true_false_prompt = settings.trueFalsePromptOverride;

      await api.post('/prompt_update', payload);

      dispatch(updateFilePrompts({
        fileIds: selectedServerIds,
        ...(settings.generateSummary && { summaryPrompt: settings.summaryPromptOverride }),
        ...(settings.generateKeywords && { keywordsPrompt: settings.keywordPromptOverride }),
        ...(settings.generateQuestions && { faqPrompt: settings.questionPromptOverride }),
        ...(settings.generateShortAnswer && { shortAnswerPrompt: settings.shortAnswerPromptOverride }),
        ...(settings.generateTrueFalse && { trueFalsePrompt: settings.trueFalsePromptOverride }),
      }));

      promptSnapshot.current = {
        summary: settings.summaryPromptOverride,
        keyword: settings.keywordPromptOverride,
        question: settings.questionPromptOverride,
        shortAnswer: settings.shortAnswerPromptOverride,
        trueFalse: settings.trueFalsePromptOverride,
      };
    } catch {
      setPromptSaveError(t('uploadInfer.inferencePanel.promptSaveFail'));
    } finally {
      setPromptSaving(false);
    }
  };

  return (
    <div className={`${styles.infpanel} ${minimized ? styles.infpanelMinimized : ''}`}>

      {/* ── Minimized rail ── */}
      {minimized ? (
        <button
          type="button"
          className={styles.minRail}
          onClick={onToggleMinimize}
          title={t('uploadInfer.inferencePanel.expandStep2')}
          aria-label={t('uploadInfer.inferencePanel.expandStep2')}
        >
          <span className={styles.minRailIcon}>
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
              strokeWidth="1.6" strokeLinecap="round" strokeLinejoin="round">
              <path d="M6 4l4 4-4 4" />
            </svg>
          </span>
          <span className={styles.minRailLabel}>
            {t('uploadInfer.inferencePanel.step2Label').replace('Configuration', '').replace('2 —', '2 —')}
            {isBatchRunning && <span className={styles.minRailDot} />}
          </span>
        </button>
      ) : (
        <>

          {/* ── Header ── */}
          <div className={styles.infpanelHead}>
            <div>
              <div className={styles.slbl}>
                {isBatchRunning ? (
                  <span className={styles.inferenceRunningLabel}>{t('uploadInfer.inferencePanel.step2Running')}</span>
                ) : t('uploadInfer.inferencePanel.step2Label')}
              </div>
              <div className={styles.selSummary}>
                {t('uploadInfer.inferencePanel.filesSelected', { count: n })}
                {isBatchRunning && <span className={styles.batchRunPill}><span className={styles.batchRunDot} />{t('uploadInfer.inferencePanel.batchRunPill')}</span>}
              </div>
            </div>
            <div className={styles.headActions}>
              {!isBatchRunning && (
                <>
                  {/* ── Run Inference button — large, self-describing ── */}
                  <button
                    className={`${styles.runBtn} ${canRunInference ? styles.runBtnReady : styles.runBtnDisabled}`}
                    onClick={handleRun}
                    disabled={!canRunInference}
                    data-tour="infer-run"
                    aria-label={`Run inference on ${n} file${n !== 1 ? 's' : ''}`}
                  >
                    <span className={styles.runBtnIconWrap}>
                      <svg width="15" height="15" viewBox="0 0 16 16" aria-hidden="true">
                        <path d="M5 3l8 5-8 5V3z" fill="currentColor" />
                      </svg>
                    </span>
                    <span className={styles.runBtnText}>
                      <span className={styles.runBtnTitle}>Run inference</span>
                      <span className={styles.runBtnSub}>{runBtnSubLabel}</span>
                    </span>
                  </button>

                  {onToggleMinimize && (
                    <button
                      className={styles.minimizeStepBtn}
                      onClick={onToggleMinimize}
                      title={t('uploadInfer.inferencePanel.minimizeStep2')}
                      aria-label={t('uploadInfer.inferencePanel.minimizeStep2')}
                    >
                      <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                        strokeWidth="1.6" strokeLinecap="round">
                        <path d="M3 8h10" />
                      </svg>
                    </button>
                  )}
                  {onClose && (
                    <button
                      className={styles.closeStepBtn}
                      onClick={onClose}
                      title={t('uploadInfer.inferencePanel.closeStep2')}
                    >
                      <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                        <path d="M4 4l8 8M12 4l-8 8" />
                      </svg>
                    </button>
                  )}
                </>
              )}
              {isBatchRunning && onToggleMinimize && (
                <button
                  className={styles.minimizeStepBtn}
                  onClick={onToggleMinimize}
                  title={t('uploadInfer.inferencePanel.minimizeStep2')}
                  aria-label={t('uploadInfer.inferencePanel.minimizeStep2')}
                >
                  <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                    strokeWidth="1.6" strokeLinecap="round">
                    <path d="M3 8h10" />
                  </svg>
                </button>
              )}
            </div>
          </div>


          {/* ── Main body ── */}
          <div className={`${styles.infpanelBody} ${isBatchRunning ? styles.infpanelBodyRunning : styles.infpanelBodyConfig}`}>

            {/* ── Submitting overlay — shown while awaiting /batch_process ── */}
            {running && (
              <div className={styles.submittingOverlay}>
                <div className={styles.submittingCard}>
                  <div className={styles.submittingSpinner} />
                  <div className={styles.submittingTitle}>{t('uploadInfer.inferencePanel.submitting')}</div>
                  <div className={styles.submittingDesc}>
                    {t('uploadInfer.inferencePanel.submittingDesc', { count: n }).split('\n').map((line, i) => <React.Fragment key={i}>{line}{i === 0 && <br />}</React.Fragment>)}
                  </div>
                  <div className={styles.submittingFiles}>
                    {selectedServerIds.slice(0, 5).map((id, i) => {
                      const f = selectFiles.find(sf => sf.id === id);
                      return f ? (
                        <div key={id} className={styles.submittingFile}>
                          <span className={styles.submittingDot} style={{ animationDelay: `${i * 0.15}s` }} />
                          {f.original_name}
                        </div>
                      ) : null;
                    })}
                    {selectedServerIds.length > 5 && (
                      <div className={styles.submittingMore}>+{selectedServerIds.length - 5} more</div>
                    )}
                  </div>
                </div>
              </div>
            )}

            {/* Selection banner — hidden while batch running or submitting */}
            {!isBatchRunning && !running && <div className={styles.selBanner}>
              <svg width="12" height="12" viewBox="0 0 16 16" fill="none" stroke="var(--blue)" strokeWidth="1.5" strokeLinecap="round">
                <path d="M2 8h12M8 3l5 5-5 5" />
              </svg>
              <span className={styles.selCt}>{n} file{n !== 1 ? 's' : ''}</span>
              <span className={styles.selNm}>
                {n === 0 ? t('uploadInfer.inferencePanel.selBannerEmpty') : `${n} file${n !== 1 ? 's' : ''} selected for inference`}
              </span>
              <button
                type="button"
                data-tour="infer-action-template"
                className={styles.mapTemplateBtn}
                onClick={() => setTemplateModalOpen(true)}
                title={t('uploadInfer.filePanel.templateBtn')}
              >
                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                  strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                  <path d="M3 2.5h10v11H3z" />
                  <path d="M5.5 5.5h5M5.5 8h5M5.5 10.5h3" />
                </svg>
                {t('uploadInfer.filePanel.templateBtn')}
              </button>
            </div>}

            {/* ── Settings — hidden while batch running or submitting ── */}
            {!isBatchRunning && !running && <div className={styles.infSettingsWrap} data-tour="infer-settings">
              <div className={styles.settingsGrid}>

                {/* Generate content card */}
                <div className={styles.card}>
                  <div className={styles.cardT}>{t('uploadInfer.inferencePanel.generateContent')}</div>

                  {/* Model dropdown — moved to the top; deliberately narrow rather than spanning the row */}
                  <div className={styles.modelFieldTop} data-tour="infer-model">
                    <div className={styles.fg}>
                      <div className={styles.modelLabelRow}>
                        <label className={styles.fl}>
                          {t('uploadInfer.inferencePanel.modelLabel')} {mlLoading && <span className={styles.loadingDot}>…</span>}
                        </label>
                        <button
                          className={styles.refreshBtn}
                          onClick={fetchModels}
                          disabled={mlLoading}
                          title={t('uploadInfer.inferencePanel.refreshModels')}
                        >
                          <svg
                            viewBox="0 0 16 16" fill="none" stroke="currentColor"
                            strokeWidth="1.6" strokeLinecap="round" strokeLinejoin="round"
                            className={mlLoading ? styles.spinning : undefined}
                          >
                            <path d="M13.5 8A5.5 5.5 0 1 1 10 3.07" />
                            <path d="M10 2v3h3" />
                          </svg>
                          Refresh
                        </button>
                      </div>
                      <select className={styles.fc} value={selectedModel}
                        onChange={e => dispatch(setSelectedModel(e.target.value))}
                        disabled={mlLoading || models.length === 0}>
                        {models.length === 0
                          ? <option value="">{t('uploadInfer.inferencePanel.noModelsOption')}</option>
                          : models.map(m => <option key={m} value={m}>{m}</option>)
                        }
                      </select>
                      {!mlLoading && models.length === 0 && (
                        <div className={styles.modelWarn}>
                          <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                            <path d="M8 2L14 13H2L8 2z" /><path d="M8 7v3M8 11.5v.1" />
                          </svg>
                          {t('uploadInfer.inferencePanel.noModels')}
                        </div>
                      )}
                      {!mlLoading && models.length > 0 && (
                        <div className={styles.modelOk}>
                          <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                            <circle cx="8" cy="8" r="5.5" /><path d="M5.5 8l2 2 3-3" />
                          </svg>
                          {t('uploadInfer.inferencePanel.modelsAvailable', { count: models.length })}
                        </div>
                      )}
                    </div>
                  </div>

                  <div className={styles.genContentCols}>

                  {/* ── Left column: all checkboxes ── */}
                  <div className={styles.checkboxCol}>

                    <div data-tour="infer-check-summary"
                      className={`${styles.cr} ${isChecked(settings.generateSummary) ? styles.ck : ''} ${styles.mb8} ${noFilesSelected ? styles.crDisabled : ''}`}
                      onClick={() => { if (noFilesSelected) return; dispatch(updateSettings({ generateSummary: !settings.generateSummary })); }}
                    >
                      <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateSummary')}</label>
                    </div>

                    <div data-tour="infer-check-keywords"
                      className={`${styles.cr} ${isChecked(settings.generateKeywords) ? styles.ck : ''} ${styles.mb8} ${noFilesSelected ? styles.crDisabled : ''}`}
                      onClick={() => {
                        if (noFilesSelected) return;
                        dispatch(updateSettings({
                          generateKeywords: !settings.generateKeywords,
                          // Force the nested toggle off when the parent turns off,
                          // so a stale "on" state can never leak into the request.
                          ...(settings.generateKeywords ? { generateKeywordInsights: false } : {}),
                        }));
                      }}
                    >
                      <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateKeywords')}</label>
                    </div>
                    {isChecked(settings.generateKeywords) && (
                      <div
                        className={`${styles.crNested} ${isChecked(settings.generateKeywordInsights) ? styles.ck : ''} ${styles.mb8}`}
                        onClick={(e) => { e.stopPropagation(); if (noFilesSelected) return; dispatch(updateSettings({ generateKeywordInsights: !settings.generateKeywordInsights })); }}
                      >
                        <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateKeywordInsights')}</label>
                      </div>
                    )}

                    <div data-tour="infer-check-questions"
                      className={`${styles.cr} ${isChecked(settings.generateQuestions) ? styles.ck : ''} ${styles.mb8} ${noFilesSelected ? styles.crDisabled : ''}`}
                      onClick={() => { if (noFilesSelected) return; dispatch(updateSettings({ generateQuestions: !settings.generateQuestions })); }}
                    >
                      <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateQuestions')}</label>
                    </div>

                    <div data-tour="infer-check-shortanswer"
                      className={`${styles.cr} ${isChecked(settings.generateShortAnswer) ? styles.ck : ''} ${styles.mb8} ${noFilesSelected ? styles.crDisabled : ''}`}
                      onClick={() => { if (noFilesSelected) return; dispatch(updateSettings({ generateShortAnswer: !settings.generateShortAnswer })); }}
                    >
                      <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateShortAnswer')}</label>
                    </div>

                    <div data-tour="infer-check-truefalse"
                      className={`${styles.cr} ${isChecked(settings.generateTrueFalse) ? styles.ck : ''} ${styles.mb8} ${noFilesSelected ? styles.crDisabled : ''}`}
                      onClick={() => { if (noFilesSelected) return; dispatch(updateSettings({ generateTrueFalse: !settings.generateTrueFalse })); }}
                    >
                      <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateTrueFalse')}</label>
                    </div>

                    <div data-tour="infer-check-timestamped"
                      className={`${styles.cr} ${isChecked(settings.timestampedSummary) ? styles.ck : ''} ${styles.mb8} ${noFilesSelected ? styles.crDisabled : ''}`}
                      onClick={() => { if (noFilesSelected) return; dispatch(updateSettings({ timestampedSummary: !settings.timestampedSummary })); }}
                    >
                      <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.timestampedSummary')}</label>
                    </div>
                    {isChecked(settings.timestampedSummary) && (
                      <div className={styles.nestedField} onClick={(e) => e.stopPropagation()}>
                        <label>{t('uploadInfer.inferencePanel.timeInterval')}</label>
                        <select
                          className={styles.fc}
                          value={settings.timeInterval}
                          onChange={e => dispatch(updateSettings({ timeInterval: Number(e.target.value) as TimeInterval }))}
                        >
                          {([5, 10, 15, 20, 30, 45, 60] as const).map(min => (
                            <option key={min} value={min}>{t('uploadInfer.inferencePanel.minutesOption', { count: min })}</option>
                          ))}
                        </select>
                      </div>
                    )}

                  </div>

                  {/* ── Right column: prompt panel for whichever checkboxes are on ── */}
                  <div className={styles.promptCol}>

                    {!isChecked(settings.generateSummary) && !isChecked(settings.generateKeywords)
                      && !isChecked(settings.generateQuestions) && !isChecked(settings.generateShortAnswer)
                      && !isChecked(settings.generateTrueFalse) && (
                      <div className={styles.noPromptHint}>
                        {noFilesSelected
                          ? t('uploadInfer.inferencePanel.selBannerEmpty')
                          : t('uploadInfer.inferencePanel.noPromptHint', 'Check an option on the left to set its prompt')}
                      </div>
                    )}

                    {isChecked(settings.generateSummary) && (
                      <div className={styles.promptCard}>
                        <div className={styles.fg}>
                          <div className={styles.flRow}>
                            <label className={styles.fl}>
                              {t('uploadInfer.inferencePanel.summaryPrompt')} <span className={styles.optTag}>{t('uploadInfer.inferencePanel.optional')}</span>
                            </label>
                            {n > 1 && (
                              <button
                                className={`${styles.perFileBtn} ${summaryExpanded ? styles.perFileBtnActive : ''}`}
                                onClick={(e) => { e.stopPropagation(); setSummaryExpanded(v => !v); }}
                                title={summaryExpanded ? 'Collapse per-file prompts' : 'View per-file prompts'}
                              >
                                <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                  <rect x="1.5" y="2" width="11" height="10" rx="2" />
                                  <path d="M4 5h6M4 7.5h4" />
                                </svg>
                                {summaryExpanded ? 'Hide' : t('uploadInfer.inferencePanel.perFile')}
                                <svg className={`${styles.perFileChevron} ${summaryExpanded ? styles.perFileChevronOpen : ''}`} viewBox="0 0 10 10" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                                  <path d="M2 3.5l3 3 3-3" />
                                </svg>
                              </button>
                            )}
                          </div>
                          <div className={`${styles.perFilePanel} ${summaryExpanded ? styles.perFilePanelOpen : ''}`}>
                            <div className={styles.perFilePanelInner}>
                              {selectedFiles.map(f => (
                                <div key={f.id} className={styles.perFileRow}>
                                  <div className={styles.perFileName}>{f.original_name}</div>
                                  <div className={styles.perFilePrompt}>{f.summary_prompt || <span className={styles.perFileEmpty}>{t('uploadInfer.inferencePanel.noPromptSet')}</span>}</div>
                                </div>
                              ))}
                            </div>
                          </div>
                          <textarea className={styles.fc} rows={3}
                            value={settings.summaryPromptOverride}
                            placeholder={n === 1 ? t('uploadInfer.inferencePanel.autofillPlaceholder') : t('uploadInfer.inferencePanel.manualPlaceholder')}
                            onChange={e => dispatch(updateSummaryPrompt(e.target.value))} />
                        </div>
                      </div>
                    )}

                    {isChecked(settings.generateKeywords) && (
                      <div className={styles.promptCard}>
                        <div className={styles.fg}>
                          <div className={styles.flRow}>
                            <label className={styles.fl}>
                              {t('uploadInfer.inferencePanel.keywordPrompt')} <span className={styles.optTag}>{t('uploadInfer.inferencePanel.optional')}</span>
                            </label>
                            {n > 1 && (
                              <button
                                className={`${styles.perFileBtn} ${keywordExpanded ? styles.perFileBtnActive : ''}`}
                                onClick={(e) => { e.stopPropagation(); setKeywordExpanded(v => !v); }}
                                title={keywordExpanded ? 'Collapse per-file prompts' : 'View per-file prompts'}
                              >
                                <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                  <rect x="1.5" y="2" width="11" height="10" rx="2" />
                                  <path d="M4 5h6M4 7.5h4" />
                                </svg>
                                {keywordExpanded ? t('uploadInfer.inferencePanel.hide') : t('uploadInfer.inferencePanel.perFile')}
                                <svg className={`${styles.perFileChevron} ${keywordExpanded ? styles.perFileChevronOpen : ''}`} viewBox="0 0 10 10" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                                  <path d="M2 3.5l3 3 3-3" />
                                </svg>
                              </button>
                            )}
                          </div>
                          <div className={`${styles.perFilePanel} ${keywordExpanded ? styles.perFilePanelOpen : ''}`}>
                            <div className={styles.perFilePanelInner}>
                              {selectedFiles.map(f => (
                                <div key={f.id} className={styles.perFileRow}>
                                  <div className={styles.perFileName}>{f.original_name}</div>
                                  <div className={styles.perFilePrompt}>{f.keywords_prompt || <span className={styles.perFileEmpty}>No prompt set</span>}</div>
                                </div>
                              ))}
                            </div>
                          </div>
                          <textarea className={styles.fc} rows={3}
                            value={settings.keywordPromptOverride}
                            placeholder={n === 1 ? 'Autofilled from file — edit to override…' : 'Leave blank to use per-file prompts…'}
                            onChange={e => dispatch(updateKeywordPrompt(e.target.value))} />
                        </div>
                      </div>
                    )}

                    {isChecked(settings.generateQuestions) && (
                      <div className={styles.promptCard}>
                        <div className={styles.fg}>
                          <div className={styles.flRow}>
                            <label className={styles.fl}>
                              {t('uploadInfer.inferencePanel.questionPrompt')} <span className={styles.optTag}>{t('uploadInfer.inferencePanel.optional')}</span>
                            </label>
                            {n > 1 && (
                              <button
                                className={`${styles.perFileBtn} ${questionExpanded ? styles.perFileBtnActive : ''}`}
                                onClick={(e) => { e.stopPropagation(); setQuestionExpanded(v => !v); }}
                                title={questionExpanded ? 'Collapse per-file prompts' : 'View per-file prompts'}
                              >
                                <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                  <rect x="1.5" y="2" width="11" height="10" rx="2" />
                                  <path d="M4 5h6M4 7.5h4" />
                                </svg>
                                {questionExpanded ? t('uploadInfer.inferencePanel.hide') : t('uploadInfer.inferencePanel.perFile')}
                                <svg className={`${styles.perFileChevron} ${questionExpanded ? styles.perFileChevronOpen : ''}`} viewBox="0 0 10 10" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                                  <path d="M2 3.5l3 3 3-3" />
                                </svg>
                              </button>
                            )}
                          </div>
                          <div className={`${styles.perFilePanel} ${questionExpanded ? styles.perFilePanelOpen : ''}`}>
                            <div className={styles.perFilePanelInner}>
                              {selectedFiles.map(f => (
                                <div key={f.id} className={styles.perFileRow}>
                                  <div className={styles.perFileName}>{f.original_name}</div>
                                  <div className={styles.perFilePrompt}>{f.faq_prompt || <span className={styles.perFileEmpty}>No prompt set</span>}</div>
                                </div>
                              ))}
                            </div>
                          </div>
                          <textarea className={styles.fc} rows={3}
                            value={settings.questionPromptOverride}
                            placeholder={n === 1 ? 'Autofilled from file — edit to override…' : 'Leave blank to use per-file prompts…'}
                            onChange={e => dispatch(updateQuestionPrompt(e.target.value))} />
                        </div>
                      </div>
                    )}

                    {isChecked(settings.generateShortAnswer) && (
                      <div className={styles.promptCard}>
                        <div className={styles.fg}>
                          <div className={styles.flRow}>
                            <label className={styles.fl}>
                              {t('uploadInfer.inferencePanel.shortAnswerPrompt')} <span className={styles.optTag}>{t('uploadInfer.inferencePanel.optional')}</span>
                            </label>
                            {n > 1 && (
                              <button
                                className={`${styles.perFileBtn} ${shortAnswerExpanded ? styles.perFileBtnActive : ''}`}
                                onClick={(e) => { e.stopPropagation(); setShortAnswerExpanded(v => !v); }}
                                title={shortAnswerExpanded ? t('uploadInfer.inferencePanel.hide') : t('uploadInfer.inferencePanel.viewPerFile')}
                              >
                                <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                  <rect x="1.5" y="2" width="11" height="10" rx="2" />
                                  <path d="M4 5h6M4 7.5h4" />
                                </svg>
                                {shortAnswerExpanded ? t('uploadInfer.inferencePanel.hide') : t('uploadInfer.inferencePanel.perFile')}
                                <svg className={`${styles.perFileChevron} ${shortAnswerExpanded ? styles.perFileChevronOpen : ''}`} viewBox="0 0 10 10" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                                  <path d="M2 3.5l3 3 3-3" />
                                </svg>
                              </button>
                            )}
                          </div>
                          <div className={`${styles.perFilePanel} ${shortAnswerExpanded ? styles.perFilePanelOpen : ''}`}>
                            <div className={styles.perFilePanelInner}>
                              {selectedFiles.map(f => (
                                <div key={f.id} className={styles.perFileRow}>
                                  <div className={styles.perFileName}>{f.original_name}</div>
                                  <div className={styles.perFilePrompt}>{f.short_answer_prompt || <span className={styles.perFileEmpty}>{t('uploadInfer.inferencePanel.noPromptSet')}</span>}</div>
                                </div>
                              ))}
                            </div>
                          </div>
                          <textarea className={styles.fc} rows={3}
                            value={settings.shortAnswerPromptOverride}
                            placeholder={n === 1 ? t('uploadInfer.inferencePanel.autofillPlaceholder') : t('uploadInfer.inferencePanel.manualPlaceholder')}
                            onChange={e => dispatch(updateShortAnswerPrompt(e.target.value))} />
                        </div>
                      </div>
                    )}

                    {isChecked(settings.generateTrueFalse) && (
                      <div className={styles.promptCard}>
                        <div className={styles.fg}>
                          <div className={styles.flRow}>
                            <label className={styles.fl}>
                              {t('uploadInfer.inferencePanel.trueFalsePrompt')} <span className={styles.optTag}>{t('uploadInfer.inferencePanel.optional')}</span>
                            </label>
                            {n > 1 && (
                              <button
                                className={`${styles.perFileBtn} ${trueFalseExpanded ? styles.perFileBtnActive : ''}`}
                                onClick={(e) => { e.stopPropagation(); setTrueFalseExpanded(v => !v); }}
                                title={trueFalseExpanded ? t('uploadInfer.inferencePanel.hide') : t('uploadInfer.inferencePanel.viewPerFile')}
                              >
                                <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                  <rect x="1.5" y="2" width="11" height="10" rx="2" />
                                  <path d="M4 5h6M4 7.5h4" />
                                </svg>
                                {trueFalseExpanded ? t('uploadInfer.inferencePanel.hide') : t('uploadInfer.inferencePanel.perFile')}
                                <svg className={`${styles.perFileChevron} ${trueFalseExpanded ? styles.perFileChevronOpen : ''}`} viewBox="0 0 10 10" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                                  <path d="M2 3.5l3 3 3-3" />
                                </svg>
                              </button>
                            )}
                          </div>
                          <div className={`${styles.perFilePanel} ${trueFalseExpanded ? styles.perFilePanelOpen : ''}`}>
                            <div className={styles.perFilePanelInner}>
                              {selectedFiles.map(f => (
                                <div key={f.id} className={styles.perFileRow}>
                                  <div className={styles.perFileName}>{f.original_name}</div>
                                  <div className={styles.perFilePrompt}>{f.true_false_prompt || <span className={styles.perFileEmpty}>{t('uploadInfer.inferencePanel.noPromptSet')}</span>}</div>
                                </div>
                              ))}
                            </div>
                          </div>
                          <textarea className={styles.fc} rows={3}
                            value={settings.trueFalsePromptOverride}
                            placeholder={n === 1 ? t('uploadInfer.inferencePanel.autofillPlaceholder') : t('uploadInfer.inferencePanel.manualPlaceholder')}
                            onChange={e => dispatch(updateTrueFalsePrompt(e.target.value))} />
                        </div>
                      </div>
                    )}

                  </div>

                  </div>

                  {/* Save / Cancel bar — persists any edited prompt overrides above */}
                  {n > 0 && (settings.generateSummary || settings.generateKeywords || settings.generateQuestions
                    || settings.generateShortAnswer || settings.generateTrueFalse) && (
                    <div className={styles.promptActions}>
                      {promptSaveError && (
                        <span className={styles.promptSaveError}>{promptSaveError}</span>
                      )}
                      <button
                        className={`${styles.btn} ${styles.btnSm} ${styles.btnGhost}`}
                        onClick={handlePromptCancel}
                        disabled={!promptsDirty || promptSaving}
                      >
                        Cancel
                      </button>
                      <button
                        className={`${styles.btn} ${styles.btnSm} ${styles.btnPrimary}`}
                        onClick={handlePromptSave}
                        disabled={!promptsDirty || promptSaving}
                      >
                        {promptSaving ? (
                          <><span className={styles.btnSpinner} />{t('uploadInfer.inferencePanel.saving')}</>
                        ) : t('uploadInfer.inferencePanel.savePrompts')}
                      </button>
                    </div>
                  )}
                </div>

              </div>
            </div>}

            {/* ══ 3-Column batch status ══ */}
            {isBatchRunning && (
              <div className={styles.batchSection}>
                <div className={styles.batchSectionTitle}>
                  <span className={styles.liveLabel}>{t('uploadInfer.inferencePanel.inferenceStatus')}</span>
                  {isBatchRunning && (
                    <span className={styles.liveStatus}>
                      <span className={styles.liveDot} />
                      Live
                      <span className={styles.liveSep}>·</span>
                      {t('uploadInfer.inferencePanel.refreshingIn', { sec: countdown })}
                      {batchData.running.length > 0 && (
                        <><span className={styles.liveSep}>·</span>
                          <span className={styles.liveFiles}>
                            {t('uploadInfer.inferencePanel.filesRunning', { count: batchData.running.length })}
                          </span></>
                      )}
                    </span>
                  )}
                </div>
                <div className={styles.batchColumns}>

                  {/* Queued */}
                  <div className={styles.batchCol}>
                    <div className={`${styles.batchColHead} ${styles.headQueued}`}>
                      <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round">
                        <circle cx="7" cy="7" r="5" /><path d="M7 4v3.5l2 1.5" />
                      </svg>
                      Queued
                      <span className={styles.batchColCount}>{batchData.queued.length}</span>
                    </div>
                    <div className={styles.batchColBody}>
                      {batchData.queued.length === 0
                        ? <div className={styles.batchEmpty}>—</div>
                        : batchData.queued.map(f => (
                          <StatusCard
                            key={f.id}
                            file={f}
                            variant="queued"
                            onStop={handleStop}
                            stopping={stoppingIds.has(f.id)}
                          />
                        ))
                      }
                    </div>
                  </div>

                  {/* Running */}
                  <div className={styles.batchCol}>
                    <div className={`${styles.batchColHead} ${styles.headRunning}`}>
                      <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round">
                        <circle cx="7" cy="7" r="5" /><path d="M7 4v3M7 9.5v.2" />
                      </svg>
                      Running
                      <span className={styles.batchColCount}>{batchData.running.length}</span>
                    </div>
                    <div className={styles.batchColBody}>
                      {batchData.running.length === 0
                        ? <div className={styles.batchEmpty}>—</div>
                        : batchData.running.map(f => (
                          <StatusCard
                            key={f.id}
                            file={f}
                            variant="running"
                            onStop={handleStop}
                            stopping={stoppingIds.has(f.id)}
                          />
                        ))
                      }
                    </div>
                  </div>

                  {/* Completed */}
                  <div className={styles.batchCol}>
                    <div className={`${styles.batchColHead} ${styles.headCompleted}`}>
                      <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                        <circle cx="7" cy="7" r="5" /><path d="M4.5 7l2 2 3-3" />
                      </svg>
                      Completed
                      <span className={styles.batchColCount}>{batchData.completed.length}</span>
                    </div>
                    <div className={styles.batchColBody}>
                      {batchData.completed.length === 0
                        ? <div className={styles.batchEmpty}>—</div>
                        : batchData.completed.map(f => <StatusCard key={f.id} file={f} variant="completed" />)
                      }
                    </div>
                  </div>

                </div>
              </div>
            )}

          </div>
        </>
      )}

      <PromptTemplateAssociationModal
        open={templateModalOpen}
        onClose={() => setTemplateModalOpen(false)}
      />
    </div>
  );
};

export default InferencePanel;
