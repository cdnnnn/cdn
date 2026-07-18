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

          {variant === 'queued' && (
            <div className={styles.queuedMeta}>
              <span className={styles.queuedDot} />
              {t('uploadInfer.inferencePanel.waitingQueue')}
            </div>
          )}

          {variant === 'completed' && (
            <div className={styles.completedMeta}>{file.inserted_at}</div>
          )}

          {variant === 'running' && typeof file.progress !== 'number' && (
            <div className={styles.statusDate}>{file.inserted_at}</div>
          )}
        </div>

        {/* Stop button — queued files only */}
        {variant === 'queued' && onStop && (
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
  const canRunInference = n > 0 && !isBatchRunning && !!selectedModel;
  const pollingRef = useRef<ReturnType<typeof setInterval> | null>(null);

  // ── Fetch models ────────────────────────────
  const fetchModels = useCallback(async () => {
    dispatch(modelsLoading());
    try {
      const res = await api.get('/get_models');
      const result = (res.data as any)?.result ?? [];
      dispatch(modelsSuccess(result));
    } catch {
      dispatch(modelsFailure());
    }
  }, [dispatch]); // eslint-disable-line

  useEffect(() => { fetchModels(); }, []); // eslint-disable-line

  // ── Polling helper ───────────────────────────
  const fetchFilesForPolling = useCallback(async () => {
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
            </div>}

            {/* ── Settings — hidden while batch running or submitting ── */}
            {!isBatchRunning && !running && <div className={styles.infSettingsWrap} data-tour="infer-settings">
              <div className={styles.settingsGrid}>

                {/* Generate content card */}
                <div className={styles.card}>
                  <div className={styles.cardT}>{t('uploadInfer.inferencePanel.generateContent')}</div>

                  {/* Model dropdown — moved to the top; deliberately narrow rather than spanning the row */}
                  <div className={styles.modelFieldTop}>
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

                  <div className={styles.fieldsGrid}>

                  {/* Summary */}
                  <div className={styles.fieldCell}>
                  <div
                    className={`${styles.cr} ${settings.generateSummary ? styles.ck : ''} ${styles.mb8}`}
                    onClick={() => dispatch(updateSettings({ generateSummary: !settings.generateSummary }))}
                  >
                    <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateSummary')}</label>
                  </div>

                  {settings.generateSummary && (
                    <div className={styles.promptInline} onClick={(e) => e.stopPropagation()}>
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
                  </div>

                  {/* Keywords + nested Keyword Insights */}
                  <div className={styles.fieldCell}>
                  <div
                    className={`${styles.cr} ${settings.generateKeywords ? styles.ck : ''} ${styles.mb8}`}
                    onClick={() => dispatch(updateSettings({
                      generateKeywords: !settings.generateKeywords,
                      // Force the nested toggle off when the parent turns off,
                      // so a stale "on" state can never leak into the request.
                      ...(settings.generateKeywords ? { generateKeywordInsights: false } : {}),
                    }))}
                  >
                    <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateKeywords')}</label>
                  </div>

                  {settings.generateKeywords && (
                    <div className={styles.promptInline} onClick={(e) => e.stopPropagation()}>
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
                  {settings.generateKeywords && (
                    <div
                      className={`${styles.crNested} ${settings.generateKeywordInsights ? styles.ck : ''} ${styles.mb8}`}
                      onClick={(e) => { e.stopPropagation(); dispatch(updateSettings({ generateKeywordInsights: !settings.generateKeywordInsights })); }}
                    >
                      <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateKeywordInsights')}</label>
                    </div>
                  )}
                  </div>

                  {/* Assessment Questions */}
                  <div className={styles.fieldCell}>
                  <div
                    className={`${styles.cr} ${settings.generateQuestions ? styles.ck : ''} ${styles.mb8}`}
                    onClick={() => dispatch(updateSettings({ generateQuestions: !settings.generateQuestions }))}
                  >
                    <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateQuestions')}</label>
                  </div>

                  {settings.generateQuestions && (
                    <div className={styles.promptInline} onClick={(e) => e.stopPropagation()}>
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
                  </div>

                  {/* Short Answer */}
                  <div className={styles.fieldCell}>
                  <div
                    className={`${styles.cr} ${settings.generateShortAnswer ? styles.ck : ''} ${styles.mb8}`}
                    onClick={() => dispatch(updateSettings({ generateShortAnswer: !settings.generateShortAnswer }))}
                  >
                    <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateShortAnswer')}</label>
                  </div>

                  {settings.generateShortAnswer && (
                    <div className={styles.promptInline} onClick={(e) => e.stopPropagation()}>
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
                  </div>

                  {/* True/False */}
                  <div className={styles.fieldCell}>
                  <div
                    className={`${styles.cr} ${settings.generateTrueFalse ? styles.ck : ''} ${styles.mb8}`}
                    onClick={() => dispatch(updateSettings({ generateTrueFalse: !settings.generateTrueFalse }))}
                  >
                    <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateTrueFalse')}</label>
                  </div>

                  {settings.generateTrueFalse && (
                    <div className={styles.promptInline} onClick={(e) => e.stopPropagation()}>
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

                  {/* Timestamp Summary + nested interval dropdown */}
                  <div className={styles.fieldCell}>
                  <div
                    className={`${styles.cr} ${settings.timestampedSummary ? styles.ck : ''} ${styles.mb8}`}
                    onClick={() => dispatch(updateSettings({ timestampedSummary: !settings.timestampedSummary }))}
                  >
                    <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.timestampedSummary')}</label>
                  </div>
                  {settings.timestampedSummary && (
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
                        : batchData.running.map(f => <StatusCard key={f.id} file={f} variant="running" />)
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
    </div>
  );
};

export default InferencePanel;













// ═══════════════════════════════════════════════
// pages/UploadInfer/FileSidebar.tsx
// Content Analytics · Shared 400px file sidebar
// Used by both the Infer tab (mode="select") and the Workspace tab
// (mode="view"). Deliberately does NOT read the Upload tab's shared
// `serverFiles` — it keeps its own local list, date filter, sort,
// search, and pagination, and fetches independently every time its
// tab becomes active. The only thing still shared with the rest of
// the app is `selectedServerIds`, since InferencePanel needs that to
// run a batch.
// ═══════════════════════════════════════════════
import React, { useCallback, useEffect, useRef, useState } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import { toggleServerFileSelection, toggleSelectAllServerFiles, selectFilesSuccess, ServerFile, FilesSortBy, SortOrder, FileStatus } from '../../store/uploadSlice';
import api from '../../services/api';
import styles from './FileSidebar.module.scss';

const getExt = (n: string): 'vtt' | 'srt' => n.toLowerCase().endsWith('.srt') ? 'srt' : 'vtt';

function toDateStr(d: Date) { return d.toISOString().slice(0, 10); }
function defaultFrom() { const d = new Date(); d.setMonth(d.getMonth() - 1); return toDateStr(d); }
function defaultTo() { return toDateStr(new Date()); }

const PAGE_WINDOW = 4;
const getPageNumbers = (current: number, total: number): number[] => {
  if (total <= PAGE_WINDOW) return Array.from({ length: total }, (_, i) => i + 1);
  const start = Math.max(1, Math.min(current - 1, total - PAGE_WINDOW + 1));
  return Array.from({ length: PAGE_WINDOW }, (_, i) => start + i);
};

type SortKey = 'id' | 'name' | 'date' | 'status';
const SORT_KEY_TO_API: Record<SortKey, FilesSortBy> = {
  id: 'id', name: 'original_name', date: 'inserted_at', status: 'status',
};
const API_TO_SORT_KEY: Record<FilesSortBy, SortKey> = {
  id: 'id', original_name: 'name', inserted_at: 'date', status: 'status',
};

// ── Status → badge label/color — same 5-state mapping as the Upload tab ──
const STATUS_META: Record<FileStatus, { labelKey: string; cls: string }> = {
  waiting: { labelKey: 'uploadInfer.filePanel.statusWaiting', cls: 'bWaiting' },
  running: { labelKey: 'uploadInfer.filePanel.statusRunning', cls: 'bRunning' },
  completed: { labelKey: 'uploadInfer.filePanel.inferenced', cls: 'bInferred' },
  error: { labelKey: 'uploadInfer.filePanel.notInferenced', cls: 'bNotInferred' },
  tbd: { labelKey: 'uploadInfer.filePanel.notInferenced', cls: 'bNotInferred' },
  queued: { labelKey: 'uploadInfer.filePanel.statusQueued', cls: 'bQueued' },
};

const STATUS_FILTER_OPTIONS: { value: FileStatus | ''; labelKey: string }[] = [
  { value: '', labelKey: 'uploadInfer.filePanel.statusAll' },
  { value: 'waiting', labelKey: 'uploadInfer.filePanel.statusWaiting' },
  { value: 'queued', labelKey: 'uploadInfer.filePanel.statusQueued' },
  { value: 'running', labelKey: 'uploadInfer.filePanel.statusRunning' },
  { value: 'completed', labelKey: 'uploadInfer.filePanel.inferenced' },
  { value: 'error', labelKey: 'uploadInfer.filePanel.statusError' },
  { value: 'tbd', labelKey: 'uploadInfer.filePanel.statusTbd' },
];

interface CbProps { checked: boolean; indeterminate?: boolean; onChange: () => void; disabled?: boolean; }
const Checkbox: React.FC<CbProps> = ({ checked, indeterminate, onChange, disabled }) => (
  <div
    className={`${styles.cb} ${checked ? styles.cbChecked : ''} ${indeterminate ? styles.cbIndet : ''} ${disabled ? styles.cbDisabled : ''}`}
    onClick={e => { e.stopPropagation(); if (!disabled) onChange(); }}
    role="checkbox" aria-checked={indeterminate ? 'mixed' : checked} tabIndex={0}
    onKeyDown={e => { if (e.key === ' ' || e.key === 'Enter') { e.preventDefault(); if (!disabled) onChange(); } }}
  />
);

interface Props {
  // "select" — checkbox multi-select for building the inference batch (Infer tab)
  // "view"   — click a file to load its results (Workspace tab)
  mode: 'select' | 'view';
  // Whether this sidebar's tab is the one currently showing. A fresh
  // fetch fires each time this flips true — i.e. every time the user
  // navigates to the Infer or Result tab, not just on first mount.
  active: boolean;
  activeFileId?: number | null;
  onFileClick?: (fileId: number, fileName: string) => void;
}

const PAGE_SIZES = [25, 50, 75, 100];

const FileSidebar: React.FC<Props> = ({ mode, active, activeFileId = null, onFileClick }) => {
  const { t } = useTranslation();
  const dispatch = useAppDispatch();
  const selectedServerIds = useAppSelector(s => s.upload.selectedServerIds);
  const isBatchRunning = useAppSelector(s => s.upload.isBatchRunning);

  // ── Independent local file list ──
  const [files, setFiles] = useState<ServerFile[]>([]);
  const [total, setTotal] = useState(0);
  const [totalPages, setTotalPages] = useState(1);
  const [page, setPage] = useState(1);
  const [pageSize, setPageSize] = useState(50);
  const [sortBy, setSortBy] = useState<FilesSortBy>('id');
  const [sortOrder, setSortOrder] = useState<SortOrder>('desc');
  const [search, setSearch] = useState('');
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const [dateFrom, setDateFrom] = useState(defaultFrom());
  const [dateTo, setDateTo] = useState(defaultTo());
  const [applying, setApplying] = useState(false);
  const [statusFilter, setStatusFilter] = useState<FileStatus | ''>('');

  const fetchFiles = useCallback(async (opts?: {
    page?: number; pageSize?: number; from?: string; to?: string;
    sortBy?: FilesSortBy; sortOrder?: SortOrder; search?: string;
    statusFilter?: FileStatus | '';
  }) => {
    const p = opts?.page ?? page;
    const ps = opts?.pageSize ?? pageSize;
    const from = opts?.from ?? dateFrom;
    const to = opts?.to ?? dateTo;
    const sBy = opts?.sortBy ?? sortBy;
    const sOrd = opts?.sortOrder ?? sortOrder;
    const q = opts?.search ?? search;
    const statusF = opts?.statusFilter ?? statusFilter;
    setLoading(true);
    setError(null);
    try {
      const res = await api.post('/files/by-date/', {
        start_date: from,
        end_date: to,
        page: p,
        page_size: ps,
        sort_by: sBy,
        sort_order: sOrd,
        ...(q ? { search: q } : {}),
        ...(statusF ? { status_filter: statusF } : {}),
      });
      const d = (res.data as any)?.data ?? {};
      setFiles(d.data ?? []);
      setTotal(d.total ?? 0);
      setPage(d.page ?? p);
      setPageSize(d.page_size ?? ps);
      setTotalPages(d.total_pages ?? 1);
      setSortBy(sBy);
      setSortOrder(sOrd);
      setStatusFilter(statusF);
      // select mode = Infer tab: this is the only live source of full file
      // objects (prompt defaults included) while that tab is active, since
      // the Upload tab's shared `serverFiles` gets cleared as soon as it's
      // not the active tab. view mode (Workspace tab) doesn't publish —
      // InferencePanel only needs this for its own selection.
      if (mode === 'select') dispatch(selectFilesSuccess(d.data ?? []));
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to load files.');
    } finally {
      setLoading(false);
    }
  }, [page, pageSize, dateFrom, dateTo, sortBy, sortOrder, search, statusFilter, mode, dispatch]);

  const handleApply = useCallback(async () => {
    setApplying(true);
    await fetchFiles({ page: 1 });
    setApplying(false);
  }, [fetchFiles]);

  const handleStatusFilterChange = (value: FileStatus | '') => {
    fetchFiles({ page: 1, statusFilter: value });
  };

  const goToPage = (p: number) => {
    if (p < 1 || p > totalPages || p === page || loading) return;
    fetchFiles({ page: p });
  };

  const handlePageSizeChange = (size: number) => {
    fetchFiles({ page: 1, pageSize: size });
  };

  const handleSort = useCallback((key: SortKey) => {
    const apiKey = SORT_KEY_TO_API[key];
    const dir: SortOrder = sortBy === apiKey ? (sortOrder === 'asc' ? 'desc' : 'asc') : 'asc';
    fetchFiles({ sortBy: apiKey, sortOrder: dir, page: 1 });
  }, [sortBy, sortOrder, fetchFiles]);
  const sort = { key: API_TO_SORT_KEY[sortBy], dir: sortOrder };

  // ── Search — server-side, debounced (matches the Upload tab's search) ──
  const [searchQuery, setSearchQuery] = useState('');
  const skipNextSearchEffect = useRef(true);
  useEffect(() => {
    if (skipNextSearchEffect.current) { skipNextSearchEffect.current = false; return; }
    const q = searchQuery.trim();
    const handle = setTimeout(() => {
      setSearch(q);
      fetchFiles({ search: q, page: 1 });
    }, 400);
    return () => clearTimeout(handle);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [searchQuery]);

  // Own API call, fired every time this tab becomes active — entirely
  // separate from whatever the Upload tab is doing. When the tab goes
  // inactive (navigated away from), clear the list and reset search/
  // sort/filter/page back to defaults so a return visit shows a clean
  // loading state and a genuinely fresh fetch, instead of the previous
  // tab's stale results flashing before the new data arrives.
  useEffect(() => {
    if (active) {
      fetchFiles({ page: 1 });
    } else {
      skipNextSearchEffect.current = true; // don't let the reset below trigger a second debounced fetch
      setFiles([]);
      if (mode === 'select') dispatch(selectFilesSuccess([]));
      setTotal(0);
      setTotalPages(1);
      setPage(1);
      setSortBy('id');
      setSortOrder('desc');
      setSearch('');
      setSearchQuery('');
      setStatusFilter('');
      setError(null);
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [active]);

  const pageSelectedCount = files.filter(f => selectedServerIds.includes(f.id)).length;
  const allSelected = files.length > 0 && pageSelectedCount === files.length;
  const someSelected = pageSelectedCount > 0 && !allSelected;

  const handleRowClick = (fileId: number, fileName: string) => {
    if (mode === 'select') {
      if (isBatchRunning) return;
      dispatch(toggleServerFileSelection(fileId));
    } else {
      onFileClick?.(fileId, fileName);
    }
  };

  return (
    <div className={styles.sidebar} data-tour={mode === 'select' ? 'infer-sidebar' : 'results-sidebar'}>
      <div className={styles.dateFilter}>
        <div className={styles.dateField}>
          <label className={styles.dateLabel}>{t('uploadInfer.filePanel.dateFrom')}</label>
          <input
            type="date" className={styles.dateInput} value={dateFrom} max={dateTo}
            disabled={applying}
            onChange={e => setDateFrom(e.target.value)}
          />
        </div>
        <div className={styles.dateSep}>—</div>
        <div className={styles.dateField}>
          <label className={styles.dateLabel}>{t('uploadInfer.filePanel.dateTo')}</label>
          <input
            type="date" className={styles.dateInput} value={dateTo} min={dateFrom}
            disabled={applying}
            onChange={e => setDateTo(e.target.value)}
          />
        </div>
        <button className={styles.applyBtn} onClick={handleApply} disabled={applying || loading}>
          {applying ? <span className={styles.miniSpinner} /> : t('uploadInfer.filePanel.applyDate')}
        </button>
      </div>

      <div className={styles.statusFilterWrap}>
        <svg className={styles.statusFilterIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
          <path d="M2 4h12M4.5 8h7M7 12h2" />
        </svg>
        <select
          className={styles.statusFilterSelect}
          value={statusFilter}
          disabled={loading}
          onChange={e => handleStatusFilterChange(e.target.value as FileStatus | '')}
        >
          {STATUS_FILTER_OPTIONS.map(opt => (
            <option key={opt.value || 'all'} value={opt.value}>{t(opt.labelKey)}</option>
          ))}
        </select>
      </div>

      <div className={styles.head}>
        {mode === 'select' && (
          <Checkbox
            checked={allSelected}
            indeterminate={someSelected}
            onChange={() => dispatch(toggleSelectAllServerFiles(files.map(f => f.id)))}
            disabled={files.length === 0 || isBatchRunning}
          />
        )}
        <span className={styles.headTitle}>{t('uploadInfer.workspace.filesTitle')}</span>
        {total > 0 && <span className={styles.headCount}>{total}</span>}
        {loading && <span className={styles.headSpinner} aria-label="Loading" />}
        {mode === 'select' && selectedServerIds.length > 0 && (
          <span className={styles.headSelected}>
            {t('uploadInfer.filePanel.selectedTotal', { count: selectedServerIds.length, total })}
          </span>
        )}
      </div>

      <div className={styles.searchWrap}>
        <svg className={styles.searchIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
          <circle cx="6.5" cy="6.5" r="4" /><path d="M11 11l2.5 2.5" />
        </svg>
        <input
          type="text"
          className={styles.searchInput}
          placeholder={t('uploadInfer.workspace.searchFiles')}
          value={searchQuery}
          onChange={e => setSearchQuery(e.target.value)}
        />
      </div>

      {/* Sort header — same sort keys as the Upload tab */}
      <div className={styles.sortHeader}>
        <span className={styles.sortHeaderLabel}>
          <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
            <path d="M2 4h10M4 7h6M6 10h2" />
          </svg>
          {t('uploadInfer.filePanel.sortBy')}
        </span>
        <div className={styles.sortCols}>
          {([
            { key: 'id' as SortKey, label: t('uploadInfer.filePanel.sortId') },
            { key: 'name' as SortKey, label: t('uploadInfer.filePanel.sortName') },
            { key: 'date' as SortKey, label: t('uploadInfer.filePanel.sortDate') },
            { key: 'status' as SortKey, label: t('uploadInfer.filePanel.sortStatus') },
          ]).map(({ key, label }) => (
            <button
              key={key}
              className={`${styles.sortCol} ${sort.key === key ? styles.sortColActive : ''}`}
              onClick={() => handleSort(key)}
              disabled={loading}
            >
              {label}
              <svg viewBox="0 0 10 12" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round"
                className={sort.key === key ? (sort.dir === 'asc' ? styles.sortAsc : styles.sortDesc) : styles.sortInactive}>
                <path d="M5 1v10M2 8l3 3 3-3" />
              </svg>
            </button>
          ))}
        </div>
      </div>

      <div className={styles.rowsWrap}>
        <div className={styles.rows}>
        {error && (
          <div className={styles.empty}>{error}</div>
        )}
        {!error && loading && files.length === 0 && (
          <div className={styles.emptyLoading}>
            <span className={styles.rowsSpinner} />
            {t('uploadInfer.workspace.loadingFiles')}
          </div>
        )}
        {!error && !loading && files.length === 0 && (
          <div className={styles.empty}>
            {search ? t('uploadInfer.workspace.noFilesMatch') : t('uploadInfer.workspace.noFilesYet')}
          </div>
        )}
        {files.map(f => {
          const ext = getExt(f.original_name);
          const isChecked = mode === 'select' && selectedServerIds.includes(f.id);
          const isActiveView = mode === 'view' && f.id === activeFileId;
          const disabled = mode === 'select' && isBatchRunning;
          const statusMeta = STATUS_META[f.status] ?? STATUS_META.tbd;
          return (
            <div
              key={f.id}
              role="button"
              tabIndex={0}
              className={`${styles.hitm} ${styles.hitmSelectable}
                ${isChecked ? styles.active : ''}
                ${isActiveView ? styles.activeView : ''}
                ${disabled ? styles.rowDisabled : ''}`}
              onClick={() => !disabled && handleRowClick(f.id, f.original_name)}
              onKeyDown={e => { if (!disabled && (e.key === 'Enter' || e.key === ' ')) { e.preventDefault(); handleRowClick(f.id, f.original_name); } }}
            >
              {mode === 'select' && (
                <Checkbox checked={isChecked} onChange={() => handleRowClick(f.id, f.original_name)} disabled={isBatchRunning} />
              )}
              <div className={`${styles.ficon} ${styles[ext]}`}>{ext.toUpperCase()}</div>
              <div className={styles.hi}>
                <div className={styles.hn}>{f.original_name}</div>
                <div className={styles.hm}>{f.inserted_at} · #{f.id}</div>
              </div>
              <span className={`${styles.badge} ${isChecked ? styles.bSelected : styles[statusMeta.cls]}`}>
                {isChecked ? (
                  <svg viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" width="10" height="10">
                    <path d="M2 6l3 3 5-5" />
                  </svg>
                ) : (
                  <>
                    {f.status === 'completed' && (
                      <svg viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" width="9" height="9">
                        <path d="M2 6l3 3 5-5" />
                      </svg>
                    )}
                    {(f.status === 'running' || f.status === 'queued') && (
                      <span className={styles.badgeDot} />
                    )}
                    {t(statusMeta.labelKey)}
                  </>
                )}
              </span>
            </div>
          );
        })}
        </div>

        {/* Blocks interaction with the current list while a fetch is in
            flight (pagination, sort, search, filter) — only shown when
            there's already content on screen; the empty-state spinner
            above covers the first load. */}
        {loading && files.length > 0 && (
          <div className={styles.rowsLoadingOverlay}>
            <span className={styles.rowsSpinner} />
          </div>
        )}
      </div>

      {/* Pagination footer — same page-number layout as the Upload tab */}
      {!loading && !error && total > 0 && (
        <div className={styles.paginationBar}>
          <div className={styles.paginationTopRow}>
            <div className={styles.pageSizeGroup}>
              <span className={styles.pageSizeLabel}>{t('uploadInfer.filePanel.perPage')}</span>
              <select
                className={styles.pageSizeSelect}
                value={pageSize}
                onChange={e => handlePageSizeChange(Number(e.target.value))}
              >
                {PAGE_SIZES.map(size => (
                  <option key={size} value={size}>{size}</option>
                ))}
              </select>
            </div>

            <div className={styles.pageNav}>
              <button
                className={styles.pageNavBtn}
                onClick={() => goToPage(page - 1)}
                disabled={page <= 1}
                aria-label="Previous page"
              >
                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                  <path d="M10 3L6 8l4 5" />
                </svg>
              </button>

              {(() => {
                const pageWindow = getPageNumbers(page, totalPages);
                const showFirst = pageWindow[0] > 1;
                const showLast = pageWindow[pageWindow.length - 1] < totalPages;
                return (
                  <>
                    {showFirst && (
                      <>
                        <button className={styles.pageNumBtn} onClick={() => goToPage(1)}>1</button>
                        {pageWindow[0] > 2 && <span className={styles.pageEllipsis}>…</span>}
                      </>
                    )}
                    {pageWindow.map(p => (
                      <button
                        key={p}
                        className={`${styles.pageNumBtn} ${p === page ? styles.pageNumBtnActive : ''}`}
                        onClick={() => goToPage(p)}
                      >
                        {p}
                      </button>
                    ))}
                    {showLast && (
                      <>
                        {pageWindow[pageWindow.length - 1] < totalPages - 1 && <span className={styles.pageEllipsis}>…</span>}
                        <button className={styles.pageNumBtn} onClick={() => goToPage(totalPages)}>{totalPages}</button>
                      </>
                    )}
                  </>
                );
              })()}

              <button
                className={styles.pageNavBtn}
                onClick={() => goToPage(page + 1)}
                disabled={page >= totalPages}
                aria-label="Next page"
              >
                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                  <path d="M6 3l4 5-4 5" />
                </svg>
              </button>
            </div>
          </div>

          <div className={styles.pageInfo}>
            {t('uploadInfer.filePanel.pageInfo', { page, totalPages, total })}
          </div>
        </div>
      )}
    </div>
  );
};

export default FileSidebar;


















// ═══════════════════════════════════════════════
// store/uploadSlice.ts
// Content Analytics · Upload & Inference state
// ═══════════════════════════════════════════════
import { createSlice, type PayloadAction } from '@reduxjs/toolkit';

// ── Server file (from /files/by-date/) ──────────
// "tbd" = not yet inferenced, "completed" = inferenced.
export type FileStatus = 'waiting' | 'running' | 'completed' | 'error' | 'tbd' | 'queued';

export interface ServerFile {
  id: number;
  original_name: string;
  inserted_at: string;
  summary_prompt: string;
  keywords_prompt: string;
  faq_prompt: string;
  short_answer_prompt: string;
  true_false_prompt: string;
  progress: string | number | null;
  dictionary_id?: number | null;
  prompt_template_id?: number | null;
  status: FileStatus;
}

// ── /files/by-date/ pagination params & sort keys ──
export type FilesSortBy = 'id' | 'original_name' | 'inserted_at' | 'status';
export type SortOrder = 'asc' | 'desc';

export interface ServerFilesData {
  queued: ServerFile[];
  completed: ServerFile[];
  pending: ServerFile[];
  running: ServerFile[];
}

export interface UploadedFile {
  id: number;
  name: string;
  size: string;
  type: 'vtt' | 'srt';
  status: 'ready' | 'running' | 'done' | 'failed';
  summaryPrompt: string;
  questionPrompt: string;
}

export interface BatchFile {
  id: number;
  name: string;
  size: string;
  type: 'vtt' | 'srt';
  progress?: number;
}

export interface BatchGroup {
  id: string;
  status: 'running' | 'queued' | 'done' | 'pending';
  files: BatchFile[];
  collapsed: boolean;
}

export type TimeInterval = 5 | 10 | 15 | 20 | 30 | 45 | 60;

export interface InferenceSettings {
  generateSummary: boolean;
  generateKeywords: boolean;
  generateQuestions: boolean;
  summaryStyle: string;
  timestampInterval: string;
  languageOutput: string;
  mcq: boolean;
  trueFalse: boolean;
  shortAnswer: boolean;
  questionCount: number;
  keywordCount: number;
  summaryPromptOverride: string;
  keywordPromptOverride: string;
  questionPromptOverride: string;

  // ── New independent content types ──
  generateShortAnswer: boolean;
  generateTrueFalse: boolean;
  // Only meaningful (and only sent as true) when generateKeywords is also true.
  generateKeywordInsights: boolean;
  timestampedSummary: boolean;
  timeInterval: TimeInterval;
  shortAnswerPromptOverride: string;
  trueFalsePromptOverride: string;
}

interface UploadState {
  files: UploadedFile[];
  selectedIds: number[];
  settingsCollapsed: boolean;
  batchVisible: boolean;
  batchGroups: BatchGroup[];
  settings: InferenceSettings;
  uploadZoneCollapsed: boolean;

  // ── Server files (from /files/by-date/) ──
  // NOTE: serverFilesData is now populated exclusively by inferenceStatusSuccess
  // (the /files/by-progress/ polling endpoint), which still returns the old
  // bucketed shape. /files/by-date/ is paginated and returns a flat, per-file
  // `status` — so it no longer feeds serverFilesData.
  serverFilesData: ServerFilesData;          // raw split by status (from /files/by-progress/)
  serverFiles: ServerFile[];             // current page of /files/by-date/ results (Upload tab only)
  serverFilesLoading: boolean;
  serverFilesError: string | null;
  // Separate from serverFiles above — that one belongs to the Upload tab
  // and gets cleared whenever it goes inactive. FileSidebar (Infer tab,
  // mode="select") publishes its own fetched page here instead, so
  // InferencePanel has a live source of full file objects (prompt
  // defaults included) for whichever file(s) are currently selected.
  selectFiles: ServerFile[];
  dateFrom: string;
  dateTo: string;
  statusFilter: FileStatus | '';

  // ── /files/by-date/ pagination, sort & search ──
  filesPage: number;
  filesPageSize: number;
  filesTotal: number;
  filesTotalPages: number;
  filesSortBy: FilesSortBy;
  filesSortOrder: SortOrder;
  filesSearch: string;

  // ── Selection on uploaded files ──
  selectedServerIds: number[];

  // ── Batch running state ──
  isBatchRunning: boolean;                  // true while queued/running not empty
  lastBatchFinishedAt: number | null;       // timestamp (Date.now()) set when batch transitions running→done

  // ── Models ──
  models: string[];
  modelsLoading: boolean;
  selectedModel: string;
}

function toDateStr(d: Date) { return d.toISOString().slice(0, 10); }
function defaultFrom() { const d = new Date(); d.setMonth(d.getMonth() - 1); return toDateStr(d); }
function defaultTo() { return toDateStr(new Date()); }

const emptyData: ServerFilesData = { queued: [], completed: [], pending: [], running: [] };

const initialState: UploadState = {
  files: [],
  selectedIds: [],
  uploadZoneCollapsed: false,
  settingsCollapsed: false,
  batchVisible: false,
  batchGroups: [],
  settings: {
    generateSummary: true, generateKeywords: true, generateQuestions: true,
    summaryStyle: 'Table of Contents', timestampInterval: '5 minute segments',
    languageOutput: 'Korean + English',
    mcq: true, trueFalse: true, shortAnswer: false,
    questionCount: 10, keywordCount: 15,
    summaryPromptOverride: '', keywordPromptOverride: '', questionPromptOverride: '',

    generateShortAnswer: false, generateTrueFalse: false,
    generateKeywordInsights: false,
    timestampedSummary: false, timeInterval: 5,
    shortAnswerPromptOverride: '', trueFalsePromptOverride: '',
  },
  serverFilesData: emptyData,
  serverFiles: [],
  selectFiles: [],
  serverFilesLoading: false,
  serverFilesError: null,
  dateFrom: defaultFrom(),
  dateTo: defaultTo(),
  statusFilter: '',

  filesPage: 1,
  filesPageSize: 50,
  filesTotal: 0,
  filesTotalPages: 1,
  filesSortBy: 'id',
  filesSortOrder: 'desc',
  filesSearch: '',

  selectedServerIds: [],
  isBatchRunning: false,
  lastBatchFinishedAt: null,
  models: [],
  modelsLoading: false,
  selectedModel: '',
};

const uploadSlice = createSlice({
  name: 'upload',
  initialState,
  reducers: {
    // ── Inference legacy ──
    toggleFileSelection(state, action: PayloadAction<number>) {
      const idx = state.selectedIds.indexOf(action.payload);
      if (idx > -1) state.selectedIds.splice(idx, 1);
      else state.selectedIds.push(action.payload);
    },
    toggleSelectAll(state) {
      const allIds = state.files.map(f => f.id);
      state.selectedIds = state.selectedIds.length === allIds.length ? [] : allIds;
    },
    toggleUploadZone(state) {
      state.uploadZoneCollapsed = !state.uploadZoneCollapsed;
    },
    toggleBatchGroup(state, action: PayloadAction<string>) {
      const g = state.batchGroups.find(g => g.id === action.payload);
      if (g) g.collapsed = !g.collapsed;
    },
    runInference(state) {
      state.batchVisible = true;
    },
    updateSummaryPrompt(state, action: PayloadAction<string>) {
      state.settings.summaryPromptOverride = action.payload;
    },
    updateKeywordPrompt(state, action: PayloadAction<string>) {
      state.settings.keywordPromptOverride = action.payload;
    },
    updateQuestionPrompt(state, action: PayloadAction<string>) {
      state.settings.questionPromptOverride = action.payload;
    },
    updateShortAnswerPrompt(state, action: PayloadAction<string>) {
      state.settings.shortAnswerPromptOverride = action.payload;
    },
    updateTrueFalsePrompt(state, action: PayloadAction<string>) {
      state.settings.trueFalsePromptOverride = action.payload;
    },
    updateSettings(state, action: PayloadAction<Partial<InferenceSettings>>) {
      state.settings = { ...state.settings, ...action.payload };
    },

    // ── Date range ──
    setDateFrom(state, action: PayloadAction<string>) { state.dateFrom = action.payload; },
    setDateTo(state, action: PayloadAction<string>) { state.dateTo = action.payload; },
    setStatusFilter(state, action: PayloadAction<FileStatus | ''>) { state.statusFilter = action.payload; },

    // ── Patch progress % onto running files from /files/progress/ response ──
    updateRunningProgress(state, action: PayloadAction<Record<string, number | string>>) {
      const progressMap = action.payload;
      state.serverFilesData.running = state.serverFilesData.running.map(f => {
        const raw = progressMap[String(f.id)];
        if (raw === undefined) return f;
        const pct = typeof raw === 'string' ? parseFloat(raw) : raw;
        return { ...f, progress: isNaN(pct) ? f.progress : Math.min(100, Math.max(0, pct)) };
      });
    },

    // ── Inference-only status update (does NOT touch the FilePanel file list) ──
    inferenceStatusSuccess(state, action: PayloadAction<ServerFilesData>) {
      const d = action.payload;
      // Merge any newly-completed files from the polling response into the
      // existing completed bucket. We keep the historical /by-date/ entries
      // and append/replace any IDs that just finished in the active batch,
      // so FilePanel badges flip to "Inferenced" the moment Step-2 moves a
      // file into its Completed column — without needing another API call.
      const incomingCompleted = d.completed ?? [];
      const incomingCompletedIds = new Set(incomingCompleted.map(f => f.id));
      const mergedCompleted = [
        ...state.serverFilesData.completed.filter(f => !incomingCompletedIds.has(f.id)),
        ...incomingCompleted,
      ];
      state.serverFilesData = {
        queued: d.queued ?? [],
        running: d.running ?? [],
        pending: d.pending ?? [],
        completed: mergedCompleted,
      };
      const running = (d.queued?.length ?? 0) > 0 || (d.running?.length ?? 0) > 0;
      // Clear selections the moment the batch becomes active so highlighted
      // cards stop showing the blue selected state after a successful submit.
      if (running && !state.isBatchRunning) {
        state.selectedServerIds = [];
      }
      // Auto-collapse settings when a batch starts running. (Previously derived
      // from /files/by-date/, which is now paginated and can't reliably tell us this.)
      if (running) state.settingsCollapsed = true;
      // When batch transitions running → done, stamp a timestamp so FilePanel
      // knows to re-fetch /by-date/ and refresh the completed list.
      if (!running && state.isBatchRunning) {
        state.lastBatchFinishedAt = Date.now();
      }
      state.isBatchRunning = running;
    },

    // ── Select-mode files (Infer tab's FileSidebar, mode="select") ──
    // Deliberately minimal — just the current page of full ServerFile
    // objects, so InferencePanel can look up prompt defaults for whatever
    // is in selectedServerIds. No pagination/sort/search mirrored here;
    // FileSidebar keeps that locally, this is purely a lookup source.
    selectFilesSuccess(state, action: PayloadAction<ServerFile[]>) {
      state.selectFiles = action.payload;
    },

    // ── Server files (paginated /files/by-date/) ──
    serverFilesLoading(state) {
      state.serverFilesLoading = true;
      state.serverFilesError = null;
    },
    serverFilesSuccess(state, action: PayloadAction<{
      files: ServerFile[];
      total: number;
      page: number;
      pageSize: number;
      totalPages: number;
      sortBy: FilesSortBy;
      sortOrder: SortOrder;
      search: string;
    }>) {
      const { files, total, page, pageSize, totalPages, sortBy, sortOrder, search } = action.payload;
      state.serverFiles = files;
      state.serverFilesLoading = false;
      state.serverFilesError = null;

      state.filesTotal = total;
      state.filesPage = page;
      state.filesPageSize = pageSize;
      state.filesTotalPages = totalPages;
      state.filesSortBy = sortBy;
      state.filesSortOrder = sortOrder;
      state.filesSearch = search;

      // NOTE: isBatchRunning is no longer derived here — /files/by-date/ is
      // paginated so a running/queued file may simply be on another page.
      // isBatchRunning is set by inferenceStatusSuccess (/files/by-progress/),
      // which always reflects the full, unpaginated set of active files.

      // NOTE: selections are intentionally NOT pruned against this page's ids —
      // a file not present here may just be on a different page, not deleted.
      // Deletion flows already clear selectedServerIds explicitly on success.
    },
    serverFilesFailure(state, action: PayloadAction<string>) {
      state.serverFilesLoading = false;
      state.serverFilesError = action.payload;
    },

    // ── Pagination / sort / search (kept in sync for UI display; the actual
    //     fetch is triggered imperatively by the component with these values) ──
    setFilesPage(state, action: PayloadAction<number>) {
      state.filesPage = action.payload;
    },
    setFilesPageSize(state, action: PayloadAction<number>) {
      state.filesPageSize = action.payload;
      state.filesPage = 1;
    },
    setFilesSort(state, action: PayloadAction<{ sortBy: FilesSortBy; sortOrder: SortOrder }>) {
      state.filesSortBy = action.payload.sortBy;
      state.filesSortOrder = action.payload.sortOrder;
      state.filesPage = 1;
    },
    setFilesSearch(state, action: PayloadAction<string>) {
      state.filesSearch = action.payload;
      state.filesPage = 1;
    },

    // ── Server file selection ──
    toggleServerFileSelection(state, action: PayloadAction<number>) {
      if (state.isBatchRunning) return;  // no selection while batch running
      const idx = state.selectedServerIds.indexOf(action.payload);
      if (idx > -1) state.selectedServerIds.splice(idx, 1);
      else state.selectedServerIds.push(action.payload);
    },
    toggleSelectAllServerFiles(state, action: PayloadAction<number[]>) {
      if (state.isBatchRunning) return;
      const pageIds = action.payload;
      if (pageIds.length === 0) return;
      const selectedSet = new Set(state.selectedServerIds);
      const allPageSelected = pageIds.every(id => selectedSet.has(id));
      if (allPageSelected) {
        // Uncheck — remove only this page's ids, leave other pages' selections alone.
        const pageIdSet = new Set(pageIds);
        state.selectedServerIds = state.selectedServerIds.filter(id => !pageIdSet.has(id));
      } else {
        // Check — add this page's ids on top of whatever's already selected elsewhere.
        pageIds.forEach(id => { if (!selectedSet.has(id)) state.selectedServerIds.push(id); });
      }
    },
    clearServerSelection(state) {
      state.selectedServerIds = [];
    },

    // ── Patch prompt fields on saved files ──
    updateFilePrompts(state, action: PayloadAction<{
      fileIds: number[];
      summaryPrompt?: string;
      keywordsPrompt?: string;
      faqPrompt?: string;
      shortAnswerPrompt?: string;
      trueFalsePrompt?: string;
    }>) {
      const { fileIds, summaryPrompt, keywordsPrompt, faqPrompt, shortAnswerPrompt, trueFalsePrompt } = action.payload;
      const idSet = new Set(fileIds);
      state.serverFiles = state.serverFiles.map(f => {
        if (!idSet.has(f.id)) return f;
        return {
          ...f,
          ...(summaryPrompt !== undefined && { summary_prompt: summaryPrompt }),
          ...(keywordsPrompt !== undefined && { keywords_prompt: keywordsPrompt }),
          ...(faqPrompt !== undefined && { faq_prompt: faqPrompt }),
          ...(shortAnswerPrompt !== undefined && { short_answer_prompt: shortAnswerPrompt }),
          ...(trueFalsePrompt !== undefined && { true_false_prompt: trueFalsePrompt }),
        };
      });
      // Also patch inside serverFilesData buckets so the per-file preview reflects the change
      const patchBucket = (bucket: ServerFile[]) => bucket.map(f => {
        if (!idSet.has(f.id)) return f;
        return {
          ...f,
          ...(summaryPrompt !== undefined && { summary_prompt: summaryPrompt }),
          ...(keywordsPrompt !== undefined && { keywords_prompt: keywordsPrompt }),
          ...(faqPrompt !== undefined && { faq_prompt: faqPrompt }),
          ...(shortAnswerPrompt !== undefined && { short_answer_prompt: shortAnswerPrompt }),
          ...(trueFalsePrompt !== undefined && { true_false_prompt: trueFalsePrompt }),
        };
      });
      state.serverFilesData.completed = patchBucket(state.serverFilesData.completed);
      state.serverFilesData.queued = patchBucket(state.serverFilesData.queued);
      state.serverFilesData.running = patchBucket(state.serverFilesData.running);
      state.serverFilesData.pending = patchBucket(state.serverFilesData.pending);
    },


    // ── Patch dictionary_id on saved files (after associate/disassociate) ──
    patchFileDictionaries(state, action: PayloadAction<Record<number, number | null>>) {
      const map = action.payload;
      const apply = (f: ServerFile): ServerFile =>
        map[f.id] !== undefined ? { ...f, dictionary_id: map[f.id] } : f;
      state.serverFiles = state.serverFiles.map(apply);
      state.serverFilesData.completed = state.serverFilesData.completed.map(apply);
      state.serverFilesData.queued = state.serverFilesData.queued.map(apply);
      state.serverFilesData.running = state.serverFilesData.running.map(apply);
      state.serverFilesData.pending = state.serverFilesData.pending.map(apply);
    },

    // ── Patch prompt_template_id on saved files (after template association) ──
    patchFilePromptTemplate(state, action: PayloadAction<Record<number, number | null>>) {
      const map = action.payload;
      const apply = (f: ServerFile): ServerFile =>
        map[f.id] !== undefined ? { ...f, prompt_template_id: map[f.id] } : f;
      state.serverFiles = state.serverFiles.map(apply);
      state.serverFilesData.completed = state.serverFilesData.completed.map(apply);
      state.serverFilesData.queued = state.serverFilesData.queued.map(apply);
      state.serverFilesData.running = state.serverFilesData.running.map(apply);
      state.serverFilesData.pending = state.serverFilesData.pending.map(apply);
    },

    // ── Models ──
    modelsLoading(state) {
      state.modelsLoading = true;
    },
    modelsSuccess(state, action: PayloadAction<string[]>) {
      state.models = action.payload;
      state.modelsLoading = false;
      if (action.payload.length > 0 && !state.selectedModel) {
        state.selectedModel = action.payload[0];
      }
    },
    modelsFailure(state) {
      state.modelsLoading = false;
    },
    setSelectedModel(state, action: PayloadAction<string>) {
      state.selectedModel = action.payload;
    },
  },
});

export const {
  toggleFileSelection, toggleSelectAll,
  toggleUploadZone, toggleBatchGroup,
  runInference, updateSummaryPrompt, updateKeywordPrompt, updateQuestionPrompt,
  updateShortAnswerPrompt, updateTrueFalsePrompt, updateSettings,
  setDateFrom, setDateTo, setStatusFilter,
  inferenceStatusSuccess,
  updateRunningProgress,
  serverFilesLoading, serverFilesSuccess, serverFilesFailure,
  selectFilesSuccess,
  setFilesPage, setFilesPageSize, setFilesSort, setFilesSearch,
  toggleServerFileSelection, toggleSelectAllServerFiles, clearServerSelection,
  updateFilePrompts,
  patchFileDictionaries,
  patchFilePromptTemplate,
  modelsLoading, modelsSuccess, modelsFailure, setSelectedModel,
} = uploadSlice.actions;

export default uploadSlice.reducer;
