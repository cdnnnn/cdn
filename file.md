// ═══════════════════════════════════════════════
// pages/GeneralSettings/GeneralSettings.tsx
// Content Analytics · General Settings
// ═══════════════════════════════════════════════
import React, { useCallback, useEffect, useMemo, useRef, useState } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import {
    fetchSettings,
    updateSettings,
    resetSettingsToDefaults,
    clearSettingsError,
    type DictionaryTerm,
} from '../../store/settingsSlice';
import { fetchPromptTemplates, type PromptTemplate } from '../../store/promptTemplateSlice';
import styles from './GeneralSettings.module.scss';
import TourGuide, { type TourStep } from './TourGuide';

type FormState = {
    subtitle_format: 'srt' | 'vtt';
    default_prompt_template_id: number | null;
    keywords_mode: 'auto' | 'custom';
    num_keywords: string;
    questions_mode: 'auto' | 'custom';
    num_assessment_questions: string;
    global_dictionary_enabled: boolean;
    global_dictionary_terms: DictionaryTerm[];
};

const DEFAULT_NUM_KEYWORDS = '20';
const DEFAULT_NUM_ASSESSMENT_QUESTIONS = '20';

const toFormState = (s: {
    subtitle_format: 'srt' | 'vtt';
    default_prompt_template_id: number | null;
    num_keywords: string;
    num_assessment_questions: string;
    global_dictionary_enabled: boolean;
    global_dictionary_terms: DictionaryTerm[];
}): FormState => ({
    subtitle_format: s.subtitle_format,
    default_prompt_template_id: s.default_prompt_template_id,
    keywords_mode: s.num_keywords === 'auto' ? 'auto' : 'custom',
    num_keywords: s.num_keywords === 'auto' ? '' : s.num_keywords,
    questions_mode: s.num_assessment_questions === 'auto' ? 'auto' : 'custom',
    num_assessment_questions: s.num_assessment_questions === 'auto' ? '' : s.num_assessment_questions,
    global_dictionary_enabled: s.global_dictionary_enabled,
    global_dictionary_terms: s.global_dictionary_terms ?? [],
});

// ── Dictionary modal ──────────────────────────────
interface DictionaryModalProps {
    terms: DictionaryTerm[];
    onSave: (terms: DictionaryTerm[]) => void;
    onClose: () => void;
    t: (key: string) => string;
}

const ROW_HEIGHT = 52; // px — must match .termRow min-height in SCSS

// ── Memoized single term row ──────────────────────
interface TermRowProps {
    term: DictionaryTerm;
    index: number;
    wrongPlaceholder: string;
    correctPlaceholder: string;
    removeAriaLabel: string;
    onChange: (index: number, field: keyof DictionaryTerm, value: string) => void;
    onRemove: (index: number) => void;
}

const TermRow: React.FC<TermRowProps> = React.memo(({ term, index, wrongPlaceholder, correctPlaceholder, removeAriaLabel, onChange, onRemove }) => (
    <div className={styles.termRow} style={{ height: ROW_HEIGHT }}>
        <input
            className={styles.input}
            value={term.wrong_term}
            onChange={(e) => onChange(index, 'wrong_term', e.target.value)}
            placeholder={wrongPlaceholder}
        />
        <svg className={styles.arrowIc} width="14" height="14" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
            <path d="M2 8h11M9 4l4 4-4 4" />
        </svg>
        <input
            className={styles.input}
            value={term.correct_term}
            onChange={(e) => onChange(index, 'correct_term', e.target.value)}
            placeholder={correctPlaceholder}
        />
        <button type="button" className={styles.removeBtn} onClick={() => onRemove(index)} aria-label={removeAriaLabel}>
            <svg width="13" height="13" viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                <path d="M2 2l10 10M12 2L2 12" />
            </svg>
        </button>
    </div>
));

// ── Virtual list — only renders visible rows ──────
const VISIBLE_ROWS = 9;
const OVERSCAN     = 3;

interface VirtualListProps {
    terms: DictionaryTerm[];
    wrongPlaceholder: string;
    correctPlaceholder: string;
    removeAriaLabel: string;
    scrollRef: React.RefObject<HTMLDivElement | null>;
    onChange: (index: number, field: keyof DictionaryTerm, value: string) => void;
    onRemove: (index: number) => void;
}

const VirtualList: React.FC<VirtualListProps> = ({ terms, wrongPlaceholder, correctPlaceholder, removeAriaLabel, scrollRef, onChange, onRemove }) => {
    const [scrollTop, setScrollTop] = useState(0);

    const totalHeight  = terms.length * ROW_HEIGHT;
    const startIdx     = Math.max(0, Math.floor(scrollTop / ROW_HEIGHT) - OVERSCAN);
    const endIdx       = Math.min(terms.length - 1, startIdx + VISIBLE_ROWS + OVERSCAN * 2);
    const visibleTerms = terms.slice(startIdx, endIdx + 1);
    const offsetY      = startIdx * ROW_HEIGHT;

    return (
        <div
            ref={scrollRef}
            className={styles.virtualScroll}
            onScroll={(e) => setScrollTop(e.currentTarget.scrollTop)}
        >
            <div style={{ height: totalHeight, position: 'relative' }}>
                <div style={{ position: 'absolute', top: offsetY, left: 0, right: 0 }}>
                    {visibleTerms.map((term, i) => (
                        <TermRow
                            key={startIdx + i}
                            index={startIdx + i}
                            term={term}
                            wrongPlaceholder={wrongPlaceholder}
                            correctPlaceholder={correctPlaceholder}
                            removeAriaLabel={removeAriaLabel}
                            onChange={onChange}
                            onRemove={onRemove}
                        />
                    ))}
                </div>
            </div>
        </div>
    );
};

// ── Dictionary modal ──────────────────────────────
const DictionaryModal: React.FC<DictionaryModalProps> = ({ terms, onSave, onClose, t }) => {
    const [localTerms, setLocalTerms] = useState<DictionaryTerm[]>(terms);
    const scrollRef = useRef<HTMLDivElement>(null);

    const handleAdd = useCallback(() => {
        setLocalTerms(prev => [...prev, { wrong_term: '', correct_term: '' }]);
        setTimeout(() => {
            if (scrollRef.current) {
                scrollRef.current.scrollTop = scrollRef.current.scrollHeight;
            }
        }, 50);
    }, []);

    const handleChange = useCallback((index: number, field: keyof DictionaryTerm, value: string) =>
        setLocalTerms(prev => prev.map((term, i) => (i === index ? { ...term, [field]: value } : term))), []);

    const handleRemove = useCallback((index: number) =>
        setLocalTerms(prev => prev.filter((_, i) => i !== index)), []);

    const handleSave = () => {
        const result = localTerms.filter(
            t => t.wrong_term.trim() !== '' || t.correct_term.trim() !== ''
        );
        onSave(result);
    };

    return (
        <div className={styles.overlay}>
            <div className={styles.modal}>
                <div className={styles.modalHead}>
                    <div className={styles.modalTitle}>{t('generalSettings.globalDict.title')}</div>
                    <button className={styles.closeBtn} onClick={onClose} type="button" aria-label={t('generalSettings.templateModal.closeAriaLabel')}>
                        <svg width="14" height="14" viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                            <path d="M2 2l10 10M12 2L2 12" />
                        </svg>
                    </button>
                </div>

                <div className={styles.modalBody}>
                    <p className={styles.dictModalDesc}>{t('generalSettings.globalDict.desc')}</p>
                    <div className={styles.termsHeader}>
                        <span>{t('generalSettings.globalDict.wrongPlaceholder')}</span>
                        <span>{t('generalSettings.globalDict.correctPlaceholder')}</span>
                    </div>
                    {localTerms.length === 0 ? (
                        <div className={styles.emptyTerms}>{t('generalSettings.globalDict.noTerms')}</div>
                    ) : (
                        <VirtualList
                            terms={localTerms}
                            wrongPlaceholder={t('generalSettings.globalDict.wrongPlaceholder')}
                            correctPlaceholder={t('generalSettings.globalDict.correctPlaceholder')}
                            removeAriaLabel={t('generalSettings.globalDict.removeTerm')}
                            scrollRef={scrollRef}
                            onChange={handleChange}
                            onRemove={handleRemove}
                        />
                    )}
                </div>

                <div className={styles.modalFoot}>
                    <button type="button" className={styles.addTermBtn} onClick={handleAdd}>
                        <svg width="12" height="12" viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                            <path d="M6 1.5v9M1.5 6h9" />
                        </svg>
                        {t('generalSettings.globalDict.addTerm')}
                    </button>
                    <div className={styles.modalFootRight}>
                        <button className={styles.btn} onClick={onClose}>{t('generalSettings.resetModal.cancel')}</button>
                        <button className={styles.btnPrimary} onClick={handleSave}>{t('generalSettings.globalDict.apply')}</button>
                    </div>
                </div>
            </div>
        </div>
    );
};


const Spinner: React.FC<{ label: string }> = ({ label }) => (
    <div className={styles.spinnerWrap}>
        <div className={styles.spinner} />
        <span>{label}</span>
    </div>
);
interface StepperProps {
    value: number;
    min: number;
    max: number;
    onChange: (val: number) => void;
}

const Stepper: React.FC<StepperProps> = ({ value, min, max, onChange }) => {
    const [inputVal, setInputVal] = useState(String(value));
    const [error, setError] = useState('');

    useEffect(() => {
        setInputVal(String(value));
        setError('');
    }, [value]);

    const commit = (raw: string) => {
        const n = parseInt(raw, 10);
        if (isNaN(n) || n < min || n > max) {
            setError(`Enter a value between ${min} and ${max}`);
            setInputVal(String(value)); // revert
        } else {
            setError('');
            onChange(n);
        }
    };

    return (
        <div className={styles.stepperWrap}>
            <div className={`${styles.stepper} ${error ? styles.stepperError : ''}`}>
                <button
                    type="button"
                    className={styles.stepBtn}
                    onClick={() => onChange(Math.max(min, value - 1))}
                    disabled={value <= min}
                    aria-label="Decrease"
                >
                    <svg width="12" height="12" viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round">
                        <path d="M2 6h8" />
                    </svg>
                </button>
                <input
                    className={styles.stepInput}
                    type="number"
                    min={min}
                    max={max}
                    value={inputVal}
                    onChange={(e) => { setInputVal(e.target.value); setError(''); }}
                    onBlur={(e) => commit(e.target.value)}
                    onKeyDown={(e) => e.key === 'Enter' && commit(inputVal)}
                />
                <button
                    type="button"
                    className={styles.stepBtn}
                    onClick={() => onChange(Math.min(max, value + 1))}
                    disabled={value >= max}
                    aria-label="Increase"
                >
                    <svg width="12" height="12" viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round">
                        <path d="M6 2v8M2 6h8" />
                    </svg>
                </button>
            </div>
            <div className={styles.stepMeta}>
                <span className={styles.stepRange}>{min} – {max}</span>
                {error && <span className={styles.stepErr}>{error}</span>}
            </div>
        </div>
    );
};

// ── Stepper ───────────────────────────────────────

const GeneralSettings: React.FC = () => {
    const dispatch = useAppDispatch();
    const { t } = useTranslation();
    const { settings, loading, saving, resetting, error } = useAppSelector(s => s.settings);
    const { templates } = useAppSelector(s => s.promptTemplate);

    const [form, setForm] = useState<FormState | null>(null);
    const [successMsg, setSuccessMsg] = useState<string | null>(null);
    const [resetConfirmOpen, setResetConfirmOpen] = useState(false);
    const [viewTemplate, setViewTemplate] = useState<PromptTemplate | null>(null);
    const [dictModalOpen, setDictModalOpen] = useState(false);
    const [tourActive, setTourActive] = useState(false);

    useEffect(() => {
        dispatch(fetchSettings());
        dispatch(fetchPromptTemplates());
    }, [dispatch]);

    useEffect(() => {
        if (settings) setForm(toFormState(settings));
    }, [settings]);

    useEffect(() => {
        if (!successMsg) return;
        const timer = setTimeout(() => setSuccessMsg(null), 3000);
        return () => clearTimeout(timer);
    }, [successMsg]);

    const isDirty = useMemo(() => {
        if (!form || !settings) return false;
        return JSON.stringify(form) !== JSON.stringify(toFormState(settings));
    }, [form, settings]);

    const selectedTemplate = useMemo(
        () => templates.find(tpl => tpl.id === form?.default_prompt_template_id) ?? null,
        [templates, form?.default_prompt_template_id],
    );

    // ── Handlers — must be defined before any early return ──
    const update = <K extends keyof FormState>(field: K, value: FormState[K]) =>
        setForm(f => (f ? { ...f, [field]: value } : f));

    if (loading || !form) {
        return (
            <div className={styles.page}>
                <div className={styles.ph}>
                    <div className={styles.phRow}>
                        <div>
                            <div className={styles.phTitle}>{t('generalSettings.pageTitle')}</div>
                            <div className={styles.phSub}>{t('generalSettings.pageSub')}</div>
                        </div>
                    </div>
                </div>
                <div className={styles.body}>
                    <div className={styles.loadingBody}>
                        <Spinner label={t('generalSettings.loading')} />
                    </div>
                </div>
            </div>
        );
    }

    const handleSave = async () => {
        dispatch(clearSettingsError());
        const payload = {
            subtitle_format: form.subtitle_format,
            default_prompt_template_id: form.default_prompt_template_id,
            num_keywords: form.keywords_mode === 'auto' ? 'auto' : (form.num_keywords || DEFAULT_NUM_KEYWORDS),
            num_assessment_questions: form.questions_mode === 'auto' ? 'auto' : (form.num_assessment_questions || DEFAULT_NUM_ASSESSMENT_QUESTIONS),
            global_dictionary_terms: form.global_dictionary_terms,
            global_dictionary_enabled: form.global_dictionary_enabled,
        };
        try {
            await dispatch(updateSettings(payload)).unwrap();
            setSuccessMsg(t('generalSettings.savedSuccess'));
        } catch { /* surfaced via error banner */ }
    };

    const handleResetConfirm = async () => {
        setResetConfirmOpen(false);
        try {
            await dispatch(resetSettingsToDefaults()).unwrap();
            setSuccessMsg(t('generalSettings.resetSuccess'));
        } catch { /* surfaced via error banner */ }
    };

    const handleDiscard = () => {
        if (settings) setForm(toFormState(settings));
        dispatch(clearSettingsError());
    };

    const tourSteps: TourStep[] = [
        {
            target: 'gs-subtitle',
            title: t('generalSettings.tour.subtitleTitle'),
            content: t('generalSettings.tour.subtitleBody'),
            placement: 'bottom',
        },
        {
            target: 'gs-template',
            title: t('generalSettings.tour.templateTitle'),
            content: t('generalSettings.tour.templateBody'),
            placement: 'bottom',
        },
        {
            target: 'gs-keywords',
            title: t('generalSettings.tour.keywordsTitle'),
            content: t('generalSettings.tour.keywordsBody'),
            placement: 'bottom',
        },
        {
            target: 'gs-questions',
            title: t('generalSettings.tour.questionsTitle'),
            content: t('generalSettings.tour.questionsBody'),
            placement: 'bottom',
        },
        {
            target: 'gs-dictionary',
            title: t('generalSettings.tour.dictTitle'),
            content: t('generalSettings.tour.dictBody'),
            placement: 'top',
        },
    ];

    return (
        <div className={styles.page}>

            {/* ── Page header ── */}
            <div className={styles.ph}>
                <div className={styles.phRow}>
                    <div>
                        <div className={styles.phTitle}>{t('generalSettings.pageTitle')}</div>
                        <div className={styles.phSub}>{t('generalSettings.pageSub')}</div>
                    </div>
                    <div className={styles.phActs}>
                        <button
                            type="button"
                            className={styles.tourTriggerBtn}
                            onClick={() => setTourActive(true)}
                            aria-label={t('generalSettings.tour.takeTour')}
                        >
                            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                <circle cx="8" cy="8" r="6.25" />
                                <path d="M6.1 6.2a1.9 1.9 0 013.6.7c0 1.3-1.7 1.5-1.7 2.7M8 11.4v.1" />
                            </svg>
                            {t('generalSettings.tour.takeTour')}
                        </button>
                        <button className={styles.btn} onClick={() => setResetConfirmOpen(true)} disabled={resetting}>
                            {resetting ? t('generalSettings.resetting') : t('generalSettings.resetToDefaults')}
                        </button>
                        {isDirty && (
                            <button className={styles.btn} onClick={handleDiscard} disabled={saving}>
                                {t('generalSettings.discard')}
                            </button>
                        )}
                        <button className={styles.btnPrimary} onClick={handleSave} disabled={saving || !isDirty}>
                            {saving ? t('generalSettings.saving') : t('generalSettings.saveChanges')}
                        </button>
                    </div>
                </div>
            </div>

            {/* ── Scrollable body ── */}
            <div className={styles.body}>
                <div className={styles.viewBody}>

                    {successMsg && <div className={styles.successBanner}>{successMsg}</div>}
                    {error && <div className={styles.errorBanner}>{error}</div>}

                    {/* ── Subtitle Format ── */}
                    <div className={styles.card} data-tour="gs-subtitle">
                        <div className={styles.cardTitle}>{t('generalSettings.subtitleFormat.title')}</div>
                        <div className={styles.cardDesc}>{t('generalSettings.subtitleFormat.desc')}</div>
                        <div className={styles.segmented}>
                            <button
                                type="button"
                                className={`${styles.segmentBtn} ${form.subtitle_format === 'srt' ? styles.segmentActive : ''}`}
                                onClick={() => update('subtitle_format', 'srt')}
                            >SRT</button>
                            <button
                                type="button"
                                className={`${styles.segmentBtn} ${form.subtitle_format === 'vtt' ? styles.segmentActive : ''}`}
                                onClick={() => update('subtitle_format', 'vtt')}
                            >VTT</button>
                        </div>
                    </div>

                    {/* ── Default Prompt Template ── */}
                    <div className={styles.card} data-tour="gs-template">
                        <div className={styles.cardTitle}>{t('generalSettings.defaultTemplate.title')}</div>
                        <div className={styles.cardDesc}>{t('generalSettings.defaultTemplate.desc')}</div>
                        <div className={styles.templateRow}>
                            <select
                                className={styles.select}
                                value={form.default_prompt_template_id ?? ''}
                                onChange={(e) =>
                                    update('default_prompt_template_id', e.target.value === '' ? null : Number(e.target.value))
                                }
                            >
                                <option value="">{t('generalSettings.defaultTemplate.none')}</option>
                                {templates.map(tpl => (
                                    <option key={tpl.id} value={tpl.id}>{tpl.name}</option>
                                ))}
                            </select>
                            <button
                                type="button"
                                className={styles.viewTplBtn}
                                onClick={() => selectedTemplate && setViewTemplate(selectedTemplate)}
                                disabled={!selectedTemplate}
                                title={selectedTemplate ? t('generalSettings.defaultTemplate.viewAriaLabel') : undefined}
                                aria-label={t('generalSettings.defaultTemplate.viewAriaLabel')}
                            >
                                <svg width="15" height="15" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                    <path d="M1 8s2.5-4.5 7-4.5S15 8 15 8s-2.5 4.5-7 4.5S1 8 1 8z" />
                                    <circle cx="8" cy="8" r="2" />
                                </svg>
                            </button>
                        </div>
                    </div>

                    {/* ── Num Keywords ── */}
                    <div className={styles.card} data-tour="gs-keywords">
                        <div className={styles.cardTitle}>{t('generalSettings.numKeywords.title')}</div>
                        <div className={styles.cardDesc}>{t('generalSettings.numKeywords.desc')}</div>
                        <div className={styles.inlineRow}>
                            <div className={styles.segmented}>
                                <button
                                    type="button"
                                    className={`${styles.segmentBtn} ${form.keywords_mode === 'auto' ? styles.segmentActive : ''}`}
                                    onClick={() => update('keywords_mode', 'auto')}
                                >{t('generalSettings.numKeywords.auto')}</button>
                                <button
                                    type="button"
                                    className={`${styles.segmentBtn} ${form.keywords_mode === 'custom' ? styles.segmentActive : ''}`}
                                    onClick={() => setForm(f => (f ? { ...f, keywords_mode: 'custom', num_keywords: f.num_keywords || DEFAULT_NUM_KEYWORDS } : f))}
                                >{t('generalSettings.numKeywords.custom')}</button>
                            </div>
                            {form.keywords_mode === 'custom' && (
                                <Stepper
                                    min={1}
                                    max={20}
                                    value={Number(form.num_keywords) || 20}
                                    onChange={(val) => update('num_keywords', String(val))}
                                />
                            )}
                        </div>
                    </div>

                    {/* ── Num Assessment Questions ── */}
                    <div className={styles.card} data-tour="gs-questions">
                        <div className={styles.cardTitle}>{t('generalSettings.numQuestions.title')}</div>
                        <div className={styles.cardDesc}>{t('generalSettings.numQuestions.desc')}</div>
                        <div className={styles.inlineRow}>
                            <div className={styles.segmented}>
                                <button
                                    type="button"
                                    className={`${styles.segmentBtn} ${form.questions_mode === 'auto' ? styles.segmentActive : ''}`}
                                    onClick={() => update('questions_mode', 'auto')}
                                >{t('generalSettings.numQuestions.auto')}</button>
                                <button
                                    type="button"
                                    className={`${styles.segmentBtn} ${form.questions_mode === 'custom' ? styles.segmentActive : ''}`}
                                    onClick={() => setForm(f => (f ? { ...f, questions_mode: 'custom', num_assessment_questions: f.num_assessment_questions || DEFAULT_NUM_ASSESSMENT_QUESTIONS } : f))}
                                >{t('generalSettings.numQuestions.custom')}</button>
                            </div>
                            {form.questions_mode === 'custom' && (
                                <Stepper
                                    min={1}
                                    max={20}
                                    value={Number(form.num_assessment_questions) || 10}
                                    onChange={(val) => update('num_assessment_questions', String(val))}
                                />
                            )}
                        </div>
                    </div>

                    {/* ── Global Dictionary ── */}
                    <div className={styles.card} data-tour="gs-dictionary">
                        <div className={styles.cardHead}>
                            <div className={styles.cardHeadText}>
                                <div className={styles.cardTitle}>{t('generalSettings.globalDict.title')}</div>
                                <div className={styles.cardDesc}>{t('generalSettings.globalDict.desc')}</div>
                            </div>
                            <label className={styles.toggle}>
                                <input
                                    type="checkbox"
                                    checked={form.global_dictionary_enabled}
                                    onChange={(e) => update('global_dictionary_enabled', e.target.checked)}
                                />
                                <span className={styles.toggleTrack}><span className={styles.toggleThumb} /></span>
                            </label>
                        </div>
                        <div className={styles.dictSummaryRow}>
                            <span className={styles.dictCount}>
                                {form.global_dictionary_terms.length > 0
                                    ? `${form.global_dictionary_terms.length} term${form.global_dictionary_terms.length === 1 ? '' : 's'} defined`
                                    : t('generalSettings.globalDict.noTerms')}
                            </span>
                            <button type="button" className={styles.btn} onClick={() => setDictModalOpen(true)}>
                                <svg width="12" height="12" viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                                    <path d="M6 1.5v9M1.5 6h9" />
                                </svg>
                                {t('generalSettings.globalDict.manage')}
                            </button>
                        </div>
                    </div>

                </div>
            </div>

            {/* ── Reset confirmation modal ── */}
            {resetConfirmOpen && (
                <div className={styles.overlay}>
                    <div className={`${styles.modal} ${styles.modalSm}`}>
                        <div className={styles.modalHead}>
                            <div className={styles.modalTitle}>{t('generalSettings.resetModal.title')}</div>
                        </div>
                        <div className={styles.modalBody}>
                            <p className={styles.confirmText}>{t('generalSettings.resetModal.body')}</p>
                        </div>
                        <div className={styles.modalFoot}>
                            <button className={styles.btn} onClick={() => setResetConfirmOpen(false)}>
                                {t('generalSettings.resetModal.cancel')}
                            </button>
                            <button className={`${styles.btnPrimary} ${styles.btnDangerSolid}`} onClick={handleResetConfirm}>
                                {t('generalSettings.resetModal.confirm')}
                            </button>
                        </div>
                    </div>
                </div>
            )}

            {/* ── Template details modal ── */}
            {viewTemplate && (
                <div className={styles.overlay}>
                    <div className={styles.modal}>
                        <div className={styles.modalHead}>
                            <div className={styles.modalTitle}>{viewTemplate.name}</div>
                            <button className={styles.closeBtn} onClick={() => setViewTemplate(null)} aria-label={t('generalSettings.templateModal.closeAriaLabel')} type="button">
                                <svg width="14" height="14" viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                                    <path d="M2 2l10 10M12 2L2 12" />
                                </svg>
                            </button>
                        </div>
                        <div className={styles.modalBody}>
                            <div className={styles.tplDetailGroup}>
                                <div className={styles.tplDetailLabel}>{t('generalSettings.templateModal.description')}</div>
                                <div className={styles.tplDetailValue}>{viewTemplate.description || '—'}</div>
                            </div>
                            <div className={styles.tplDetailGroup}>
                                <div className={styles.tplDetailLabel}>{t('generalSettings.templateModal.summaryPrompt')}</div>
                                <div className={styles.tplDetailBlock}>{viewTemplate.summary_prompt || '—'}</div>
                            </div>
                            <div className={styles.tplDetailGroup}>
                                <div className={styles.tplDetailLabel}>{t('generalSettings.templateModal.keywordPrompt')}</div>
                                <div className={styles.tplDetailBlock}>{viewTemplate.keyword_prompt || '—'}</div>
                            </div>
                            <div className={styles.tplDetailGroup}>
                                <div className={styles.tplDetailLabel}>{t('generalSettings.templateModal.faqPrompt')}</div>
                                <div className={styles.tplDetailBlock}>{viewTemplate.faq_prompt || '—'}</div>
                            </div>
                        </div>
                        <div className={styles.modalFoot}>
                            <button className={styles.btn} onClick={() => setViewTemplate(null)}>
                                {t('generalSettings.templateModal.close')}
                            </button>
                        </div>
                    </div>
                </div>
            )}
            {/* ── Tour ── */}
            <TourGuide
                steps={tourSteps}
                active={tourActive}
                onFinish={() => setTourActive(false)}
            />

            {/* ── Dictionary modal ── */}
            {dictModalOpen && (
                <DictionaryModal
                    terms={form.global_dictionary_terms}
                    onSave={(terms) => { update('global_dictionary_terms', terms); setDictModalOpen(false); }}
                    onClose={() => setDictModalOpen(false)}
                    t={t}
                />
            )}

        </div>
    );
};

export default GeneralSettings;















// ═══════════════════════════════════════════════
// GeneralSettings.module.scss
// Content Analytics · General Settings
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
    padding: 14px 24px 12px;
    background: var(--bg1);
    border-bottom: 1px solid var(--bdr);
    flex-shrink: 0;
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

.phActs {
    display: flex;
    align-items: center;
    gap: 10px;
    flex-shrink: 0;
}

// ── Scrollable body ──────────────────────────────
.body {
    flex: 1;
    overflow-y: auto;
    @include m.scrollbar;
}

.viewBody {
    padding: 20px 24px 40px;
    display: flex;
    flex-direction: column;
    gap: 14px;
    max-width: 680px;
}

// ── Banners ───────────────────────────────────────
.successBanner {
    padding: 9px 14px;
    border-radius: var(--r);
    background: var(--green-dim);
    border: 1px solid var(--green-bdr);
    color: var(--green);
    font-size: 12px;
    font-weight: 500;
}

.errorBanner {
    padding: 9px 14px;
    border-radius: var(--r);
    background: var(--red-dim);
    border: 1px solid var(--red-bdr);
    color: var(--red);
    font-size: 12px;
    font-weight: 500;
}

// ── Cards ─────────────────────────────────────────
.card {
    @include m.card;
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.cardHead {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
}

.cardHeadText {
    flex: 1;
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 4px;

    .cardDesc {
        margin-top: 0;
    }
}

.cardTitle {
    font-size: 13px;
    font-weight: 600;
    color: var(--t0);
}

.cardDesc {
    font-size: 11.5px;
    color: var(--t2);
    line-height: 1.5;
    margin-top: -6px;
}

.inlineRow {
    display: flex;
    align-items: center;
    gap: 10px;
    flex-wrap: wrap;
}

// ── Segmented control ────────────────────────────
.segmented {
    display: inline-flex;
    padding: 2px;
    background: var(--bg3);
    border: 1px solid var(--bdr2);
    border-radius: var(--r);
    width: fit-content;
}

.segmentBtn {
    padding: 6px 14px;
    border-radius: 5px;
    border: none;
    background: transparent;
    color: var(--t2);
    font-family: var(--font-ui);
    font-size: 12px;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.12s;

    &:hover {
        color: var(--t0);
    }
}

.segmentActive {
    background: var(--blue);
    color: #fff;

    &:hover {
        color: #fff;
    }
}

// ── Select / number input ────────────────────────
.select,
.numInput,
.input {
    background: var(--bg3);
    border: 1px solid var(--bdr2);
    border-radius: var(--r);
    color: var(--t0);
    font-family: var(--font-ui);
    font-size: 13px;
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

.select {
    width: 100%;
    max-width: 320px;
}

.templateRow {
    display: flex;
    align-items: center;
    gap: 8px;
}

.viewTplBtn {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 30px;
    height: 30px;
    flex-shrink: 0;
    border-radius: var(--r);
    border: 1px solid var(--bdr2);
    background: transparent;
    color: var(--t2);
    cursor: pointer;
    transition: all 0.12s;

    &:hover:not(:disabled) {
        background: var(--bg3);
        color: var(--t0);
        border-color: var(--bdr3);
    }

    &:disabled {
        opacity: 0.4;
        cursor: default;
    }
}

.tplDetailGroup {
    display: flex;
    flex-direction: column;
    gap: 4px;
}

.tplDetailLabel {
    font-size: 10px;
    font-weight: 600;
    color: var(--t2);
    text-transform: uppercase;
    letter-spacing: 0.08em;
    @include m.mono;
}

.tplDetailValue {
    font-size: 13px;
    color: var(--t0);
    line-height: 1.5;
}

.tplDetailBlock {
    font-size: 12.5px;
    color: var(--t1);
    line-height: 1.6;
    background: var(--bg3);
    border: 1px solid var(--bdr2);
    border-radius: var(--r);
    padding: 10px 12px;
    white-space: pre-wrap;
    @include m.mono;
}

// ── Stepper ───────────────────────────────────────
.stepperWrap {
    display: flex;
    align-items: center;
    gap: 8px;
}

.stepper {
    display: inline-flex;
    align-items: center;
    height: 32px;
    border: 1px solid var(--bdr2);
    border-radius: var(--r);
    overflow: hidden;
    background: var(--bg3);
    transition: border-color 0.12s, box-shadow 0.12s;

    &:focus-within {
        border-color: var(--blue-bdr);
        box-shadow: 0 0 0 2px var(--blue-dim);
    }
}

.stepperError {
    border-color: var(--red-bdr) !important;
    box-shadow: 0 0 0 2px var(--red-dim) !important;
}

.stepBtn {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 30px;
    height: 100%;
    flex-shrink: 0;
    border: none;
    background: transparent;
    color: var(--t1);
    cursor: pointer;
    transition: background 0.1s, color 0.1s;

    &:hover:not(:disabled) {
        background: var(--bg2);
        color: var(--blue);
    }

    &:disabled {
        color: var(--t2);
        opacity: 0.35;
        cursor: default;
    }

    &:first-child {
        border-right: 1px solid var(--bdr2);
    }

    &:last-child {
        border-left: 1px solid var(--bdr2);
    }
}

.stepInput {
    width: 44px;
    height: 100%;
    border: none;
    background: transparent;
    color: var(--t0);
    font-family: var(--font-ui);
    font-size: 13px;
    font-weight: 700;
    text-align: center;
    outline: none;
    padding: 0;

    &::-webkit-inner-spin-button,
    &::-webkit-outer-spin-button {
        -webkit-appearance: none;
        margin: 0;
    }
    -moz-appearance: textfield;
}

.stepMeta {
    display: flex;
    flex-direction: column;
    gap: 2px;
}

.stepRange {
    font-size: 11px;
    color: var(--t2);
    @include m.mono;
    line-height: 1;
}

.stepErr {
    font-size: 11px;
    color: var(--red);
    line-height: 1;
}

// ── Toggle switch ─────────────────────────────────
.toggle {
    position: relative;
    display: inline-flex;
    flex-shrink: 0;
    cursor: pointer;

    input {
        position: absolute;
        opacity: 0;
        width: 0;
        height: 0;

        &:checked+.toggleTrack {
            background: var(--blue);
            border-color: var(--blue-bdr);
        }

        &:checked+.toggleTrack .toggleThumb {
            transform: translateX(16px);
        }
    }
}

.toggleTrack {
    display: inline-flex;
    align-items: center;
    width: 36px;
    height: 20px;
    border-radius: 99px;
    background: var(--bg3);
    border: 1px solid var(--bdr2);
    transition: background 0.15s, border-color 0.15s;
}

.toggleThumb {
    width: 14px;
    height: 14px;
    margin-left: 2px;
    border-radius: 50%;
    background: #fff;
    transition: transform 0.15s;
}

// ── Dictionary terms ──────────────────────────────
.termsList {
    display: flex;
    flex-direction: column;
    gap: 8px;
}

.termRow {
    display: grid;
    grid-template-columns: 1fr auto 1fr auto;
    align-items: center;
    gap: 8px;
    min-height: 52px;
    padding: 4px 0;
    box-sizing: border-box;
}

.arrowIc {
    color: var(--t2);
    flex-shrink: 0;
}

.emptyTerms {
    font-size: 11.5px;
    color: var(--t2);
    padding: 10px 0;
}

.removeBtn {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 26px;
    height: 26px;
    border-radius: 7px;
    border: 1px solid var(--bdr2);
    background: transparent;
    color: var(--t2);
    cursor: pointer;
    transition: all 0.12s;
    flex-shrink: 0;

    &:hover {
        background: var(--red-dim);
        color: var(--red);
        border-color: var(--red-bdr);
    }
}

.addTermBtn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    width: fit-content;
    padding: 6px 12px;
    border-radius: var(--r);
    border: 1px dashed var(--bdr3);
    background: transparent;
    color: var(--t1);
    font-family: var(--font-ui);
    font-size: 11.5px;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.12s;

    &:hover {
        background: var(--bg3);
        color: var(--t0);
        border-color: var(--blue-bdr);
    }
}

// ── Tour trigger button ───────────────────────────
.tourTriggerBtn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 4px 10px;
    border-radius: 99px;
    border: 1px solid var(--bdr2);
    background: transparent;
    color: var(--t2);
    font-family: var(--font-ui);
    font-size: 12px;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.12s;
    white-space: nowrap;

    svg {
        width: 13px;
        height: 13px;
        flex-shrink: 0;
    }

    &:hover {
        background: var(--bg3);
        color: var(--t1);
        border-color: var(--bdr3);
    }
}

// ── Buttons ───────────────────────────────────────
.btn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 6px 13px;
    border-radius: var(--r);
    border: 1px solid var(--bdr2);
    background: transparent;
    color: var(--t1);
    font-family: var(--font-ui);
    font-size: 12px;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.12s;
    white-space: nowrap;

    &:hover:not(:disabled) {
        background: var(--bg3);
        color: var(--t0);
        border-color: var(--bdr3);
    }

    &:disabled {
        opacity: 0.5;
        cursor: default;
    }
}

.btnPrimary {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 7px 14px;
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

.btnDangerSolid {
    background: var(--red);
    border-color: var(--red-bdr);

    &:hover:not(:disabled) {
        filter: brightness(1.08);
    }
}

// ── Spinner ───────────────────────────────────────

// ── Virtual scroll container ──────────────────────
.virtualScroll {
    height: calc(9 * 52px);
    overflow-y: auto;
    padding-right: 8px;
    @include m.scrollbar;
}

// ── Dictionary summary ─────────────────────────────
.dictSummaryRow {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
    padding-top: 4px;
}

.dictCount {
    font-size: 12px;
    color: var(--t2);
    @include m.mono;
}

.dictModalDesc {
    font-size: 12.5px;
    color: var(--t1);
    line-height: 1.5;
    margin: 0 0 4px;
}

.termsHeader {
    display: grid;
    grid-template-columns: 1fr auto 1fr auto;
    gap: 8px;
    padding: 0 0 4px;
    font-size: 10px;
    font-weight: 600;
    color: var(--t2);
    text-transform: uppercase;
    letter-spacing: 0.07em;

    // only show first two columns as headers (arrow + remove have no label)
    span:nth-child(1) { grid-column: 1; }
    span:nth-child(2) { grid-column: 3; }
}
.loadingBody {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 100%;
    height: 100%;
    min-height: 320px;
}

.spinnerWrap {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 12px;
    color: var(--t2);
    font-size: 12px;
    @include m.mono;
}

.spinner {
    width: 22px;
    height: 22px;
    border: 2px solid var(--bdr2);
    border-top-color: var(--blue);
    border-radius: 50%;
    animation: gsSpin 0.7s linear infinite;
}

// ── Modal (reset confirmation) ────────────────────
.overlay {
    position: fixed;
    inset: 0;
    background: rgba(0, 0, 0, 0.55);
    backdrop-filter: blur(2px);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
    padding: 24px;
    animation: gsFadeIn 0.15s ease-out;
}

.modal {
    width: 100%;
    max-width: 560px;
    max-height: calc(100vh - 48px);
    display: flex;
    flex-direction: column;
    background: var(--bg1);
    border: 1px solid var(--bdr2);
    border-radius: var(--rxl);
    box-shadow: var(--shadow);
    animation: gsScaleIn 0.16s cubic-bezier(0.4, 0, 0.2, 1);
}

.modalSm {
    max-width: 380px;
}

.modalHead {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 16px 18px;
    border-bottom: 1px solid var(--bdr);
    flex-shrink: 0;
}

.modalTitle {
    font-size: 14px;
    font-weight: 600;
    color: var(--t0);
    font-family: var(--font-display);
}

.closeBtn {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 26px;
    height: 26px;
    border-radius: 7px;
    border: none;
    background: transparent;
    color: var(--t2);
    cursor: pointer;
    transition: background 0.12s, color 0.12s;
    flex-shrink: 0;

    &:hover {
        background: var(--bg3);
        color: var(--t0);
    }
}

.modalBody {
    padding: 16px 18px;
    overflow-y: auto;
    display: flex;
    flex-direction: column;
    gap: 14px;
    @include m.scrollbar;
}

.modalFoot {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
    padding: 14px 18px;
    border-top: 1px solid var(--bdr);
    flex-shrink: 0;
}

.modalFootRight {
    display: flex;
    align-items: center;
    gap: 8px;
}

.confirmText {
    font-size: 13px;
    color: var(--t1);
    line-height: 1.5;
}

// ── Animations ────────────────────────────────────
@keyframes gsFadeIn {
    from {
        opacity: 0;
    }

    to {
        opacity: 1;
    }
}

@keyframes gsScaleIn {
    from {
        opacity: 0;
        transform: translateY(6px) scale(0.98);
    }

    to {
        opacity: 1;
        transform: translateY(0) scale(1);
    }
}

@keyframes gsSpin {
    to {
        transform: rotate(360deg);
    }
}













// Add this "tour" block inside "generalSettings": { ... } in en.json

"tour": {
  "takeTour": "Take a tour",
  "subtitleTitle": "Subtitle Format",
  "subtitleBody": "Choose whether lecture subtitles are generated in SRT or VTT format. This default applies to all new lecture uploads.",
  "templateTitle": "Default Prompt Template",
  "templateBody": "Select a prompt template applied automatically to every new upload. Use the eye icon to preview a template before selecting it.",
  "keywordsTitle": "Number of Keywords",
  "keywordsBody": "Set to Auto to let the platform decide, or switch to Custom and pick a number between 1 and 20 keywords to extract per lecture.",
  "questionsTitle": "Number of Assessment Questions",
  "questionsBody": "Choose Auto for platform-decided counts, or Custom to set a fixed number between 1 and 20 assessment questions per lecture.",
  "dictTitle": "Global Dictionary",
  "dictBody": "Enable auto-correction of recurring transcription errors across all lectures. Click Manage to add wrong → correct term pairs applied automatically."
}






// Add this "tour" block inside "generalSettings": { ... } in ko.json

"tour": {
  "takeTour": "둘러보기",
  "subtitleTitle": "자막 형식",
  "subtitleBody": "강의 자막을 SRT 또는 VTT 형식으로 생성할지 선택하세요. 이 기본값은 새로 업로드되는 모든 강의에 적용됩니다.",
  "templateTitle": "기본 프롬프트 템플릿",
  "templateBody": "새 강의 업로드 시 자동으로 적용될 프롬프트 템플릿을 선택하세요. 눈 모양 아이콘을 클릭하면 선택 전에 템플릿 내용을 미리 볼 수 있습니다.",
  "keywordsTitle": "키워드 수",
  "keywordsBody": "자동으로 설정하면 플랫폼이 적절한 수를 결정하고, 직접 지정으로 바꾸면 강의당 추출할 키워드 수를 1~20개 사이에서 설정할 수 있습니다.",
  "questionsTitle": "평가 문항 수",
  "questionsBody": "자동으로 설정하면 플랫폼이 수를 결정하고, 직접 지정으로 바꾸면 강의당 생성할 평가 문항 수를 1~20개 사이에서 고정할 수 있습니다.",
  "dictTitle": "전역 사전",
  "dictBody": "모든 강의에서 반복되는 전사 오류를 자동으로 수정합니다. 관리 버튼을 클릭하여 잘못된 용어와 올바른 용어 쌍을 추가하면 자동으로 적용됩니다."
}
