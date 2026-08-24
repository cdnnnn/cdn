// ═══════════════════════════════════════════════
// pages/PromptTemplates/PromptTemplates.tsx
// Content Analytics · Prompt Template management
// ═══════════════════════════════════════════════
import React, { useEffect, useRef, useState } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import {
    fetchPromptTemplates,
    createPromptTemplate,
    updatePromptTemplate,
    deletePromptTemplate,
    clearPromptTemplateError,
    type PromptTemplate,
} from '../../store/promptTemplateSlice';
import TourGuide, { type TourStep } from './TourGuide';
import styles from './PromptTemplates.module.scss';

// ─────────────────────────────────────────────
// ResizableTextarea
// Custom drag handle that works inside any
// overflow container (native CSS resize breaks
// inside overflow:auto/scroll — this bypasses
// that restriction via JS mouse events).
// ─────────────────────────────────────────────
const MIN_H = 80;
const MAX_H = 400;

interface ResizableTextareaProps extends React.TextareaHTMLAttributes<HTMLTextAreaElement> {
    textareaClassName?: string;
    wrapClassName?: string;
}

const ResizableTextarea: React.FC<ResizableTextareaProps> = ({
    textareaClassName,
    wrapClassName,
    readOnly,
    ...props
}) => {
    const textareaRef = useRef<HTMLTextAreaElement>(null);
    const startY = useRef(0);
    const startH = useRef(0);

    const onMouseDown = (e: React.MouseEvent) => {
        e.preventDefault();
        startY.current = e.clientY;
        startH.current = textareaRef.current?.offsetHeight ?? MIN_H;

        const onMouseMove = (ev: MouseEvent) => {
            const delta = ev.clientY - startY.current;
            const next = Math.max(MIN_H, Math.min(startH.current + delta, MAX_H));
            if (textareaRef.current) {
                textareaRef.current.style.height = `${next}px`;
            }
        };

        const onMouseUp = () => {
            window.removeEventListener('mousemove', onMouseMove);
            window.removeEventListener('mouseup', onMouseUp);
        };

        window.addEventListener('mousemove', onMouseMove);
        window.addEventListener('mouseup', onMouseUp);
    };

    return (
        <div className={`${styles.textareaWrap} ${wrapClassName ?? ''}`}>
            <textarea
                ref={textareaRef}
                className={textareaClassName}
                readOnly={readOnly}
                {...props}
            />
            <div
                className={styles.resizeHandle}
                onMouseDown={onMouseDown}
                title="Drag to resize"
                aria-hidden="true"
            >
                <svg width="18" height="6" viewBox="0 0 18 6" fill="none">
                    <line x1="2" y1="1" x2="16" y2="1" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" />
                    <line x1="2" y1="3.5" x2="16" y2="3.5" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" />
                    <line x1="2" y1="6" x2="16" y2="6" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" />
                </svg>
            </div>
        </div>
    );
};

type FormState = {
    name: string;
    description: string;
    summary_prompt: string;
    keyword_prompt: string;
    faq_prompt: string;
    short_answer_prompt: string;
    true_false_prompt: string;
};

const EMPTY_FORM: FormState = {
    name: '',
    description: '',
    summary_prompt: '',
    keyword_prompt: '',
    faq_prompt: '',
    short_answer_prompt: '',
    true_false_prompt: '',
};

// Maps each field to its i18n label key and tour target id
const FIELD_CONFIG = [
    { field: 'name'               as const, inputType: 'input'    as const, labelKey: 'name',            placeholderKey: 'namePlaceholder',     tourTarget: 'pt-field-name'         },
    { field: 'description'        as const, inputType: 'input'    as const, labelKey: 'description',     placeholderKey: 'descPlaceholder',     tourTarget: undefined                },
    { field: 'summary_prompt'     as const, inputType: 'textarea' as const, labelKey: 'summaryPrompt',   placeholderKey: 'summaryPlaceholder',  tourTarget: 'pt-field-summary'      },
    { field: 'keyword_prompt'     as const, inputType: 'textarea' as const, labelKey: 'keywordPrompt',   placeholderKey: 'keywordPlaceholder',  tourTarget: 'pt-field-keyword'      },
    { field: 'faq_prompt'         as const, inputType: 'textarea' as const, labelKey: 'faqPrompt',       placeholderKey: 'faqPlaceholder',      tourTarget: 'pt-field-faq'          },
    { field: 'short_answer_prompt'as const, inputType: 'textarea' as const, labelKey: 'shortAnswerPrompt', placeholderKey: 'shortAnswerPlaceholder', tourTarget: 'pt-field-short-answer' },
    { field: 'true_false_prompt'  as const, inputType: 'textarea' as const, labelKey: 'trueFalsePrompt', placeholderKey: 'trueFalsePlaceholder', tourTarget: 'pt-field-true-false'  },
] as const;

const formatDate = (iso: string) =>
    new Date(iso).toLocaleDateString('en-GB', { day: '2-digit', month: 'short', year: 'numeric' });

const Spinner: React.FC<{ label: string }> = ({ label }) => (
    <div className={styles.spinnerWrap}>
        <div className={styles.spinner} />
        <span>{label}</span>
    </div>
);

const PromptTemplates: React.FC = () => {
    const dispatch = useAppDispatch();
    const { t } = useTranslation();
    const { templates, total, loading, saving, deletingId, error } = useAppSelector(s => s.promptTemplate);

    const [modalOpen, setModalOpen] = useState(false);
    const [modalMode, setModalMode] = useState<'create' | 'edit' | 'view'>('create');
    const [editingId, setEditingId] = useState<number | null>(null);
    const [form, setForm] = useState<FormState>(EMPTY_FORM);
    const [formError, setFormError] = useState<string | null>(null);
    const [deleteTarget, setDeleteTarget] = useState<PromptTemplate | null>(null);
    const [successMsg, setSuccessMsg] = useState<string | null>(null);
    const [tourActive, setTourActive] = useState(false);

    // Stable ref so tour onEnter callbacks never close over a stale setter
    const setModalOpenRef = useRef(setModalOpen);
    setModalOpenRef.current = setModalOpen;

    useEffect(() => {
        dispatch(fetchPromptTemplates());
    }, [dispatch]);

    useEffect(() => {
        if (!successMsg) return;
        const timer = setTimeout(() => setSuccessMsg(null), 3000);
        return () => clearTimeout(timer);
    }, [successMsg]);

    const openCreate = () => {
        setModalMode('create');
        setEditingId(null);
        setForm(EMPTY_FORM);
        setFormError(null);
        dispatch(clearPromptTemplateError());
        setModalOpen(true);
    };

    const openEdit = (tpl: PromptTemplate) => {
        setModalMode('edit');
        setEditingId(tpl.id);
        setForm({
            name: tpl.name,
            description: tpl.description,
            summary_prompt: tpl.summary_prompt,
            keyword_prompt: tpl.keyword_prompt,
            faq_prompt: tpl.faq_prompt,
            short_answer_prompt: tpl.short_answer_prompt,
            true_false_prompt: tpl.true_false_prompt,
        });
        setFormError(null);
        dispatch(clearPromptTemplateError());
        setModalOpen(true);
    };

    const openView = (tpl: PromptTemplate) => {
        setModalMode('view');
        setEditingId(tpl.id);
        setForm({
            name: tpl.name,
            description: tpl.description,
            summary_prompt: tpl.summary_prompt,
            keyword_prompt: tpl.keyword_prompt,
            faq_prompt: tpl.faq_prompt,
            short_answer_prompt: tpl.short_answer_prompt,
            true_false_prompt: tpl.true_false_prompt,
        });
        setFormError(null);
        setModalOpen(true);
    };

    const closeModal = () => {
        if (saving) return;
        setModalOpen(false);
    };

    const handleChange = (field: keyof FormState) => (
        e: React.ChangeEvent<HTMLInputElement | HTMLTextAreaElement>,
    ) => setForm(f => ({ ...f, [field]: e.target.value }));

    const isFormValid =
        form.name.trim() !== '' &&
        form.description.trim() !== '' &&
        form.summary_prompt.trim() !== '' &&
        form.keyword_prompt.trim() !== '' &&
        form.faq_prompt.trim() !== '' &&
        form.short_answer_prompt.trim() !== '' &&
        form.true_false_prompt.trim() !== '';

    const handleSubmit = async (e: React.FormEvent) => {
        e.preventDefault();
        if (!isFormValid) { setFormError(t('promptTemplates.allFieldsRequired')); return; }
        setFormError(null);
        try {
            if (editingId) {
                await dispatch(updatePromptTemplate({ id: editingId, ...form })).unwrap();
                setSuccessMsg(t('promptTemplates.updatedSuccess'));
            } else {
                await dispatch(createPromptTemplate(form)).unwrap();
                setSuccessMsg(t('promptTemplates.createdSuccess'));
            }
            setModalOpen(false);
        } catch (err) {
            setFormError(typeof err === 'string' ? err : t('promptTemplates.genericError'));
        }
    };

    const confirmDelete = async () => {
        if (!deleteTarget) return;
        try {
            await dispatch(deletePromptTemplate(deleteTarget.id)).unwrap();
            setSuccessMsg(t('promptTemplates.deletedSuccess'));
        } catch { /* surfaced via error banner */ } finally {
            setDeleteTarget(null);
        }
    };

    // ── Tour steps ────────────────────────────────
    // Steps 6-13 target form fields inside the modal.
    // onEnter on step 6 opens the modal so the fields exist in the DOM;
    // handleTourFinish closes it again when the tour ends/is skipped.
    const TOUR_STEPS: TourStep[] = [
        {
            target: 'pt-header',
            title: t('promptTemplates.tour.headerTitle'),
            content: t('promptTemplates.tour.headerContent'),
            placement: 'bottom',
        },
        {
            target: 'pt-new-btn',
            title: t('promptTemplates.tour.newBtnTitle'),
            content: t('promptTemplates.tour.newBtnContent'),
            placement: 'bottom',
        },
        {
            target: 'pt-table',
            title: t('promptTemplates.tour.tableTitle'),
            content: t('promptTemplates.tour.tableContent'),
            placement: 'top',
        },
        {
            target: 'pt-edit-btn',
            title: t('promptTemplates.tour.editBtnTitle'),
            content: t('promptTemplates.tour.editBtnContent'),
            placement: 'left',
        },
        {
            target: 'pt-delete-btn',
            title: t('promptTemplates.tour.deleteBtnTitle'),
            content: t('promptTemplates.tour.deleteBtnContent'),
            placement: 'left',
        },
        {
            target: 'pt-field-name',
            title: t('promptTemplates.tour.fieldNameTitle'),
            content: t('promptTemplates.tour.fieldNameContent'),
            placement: 'bottom',
            onEnter: () => {
                setModalMode('create');
                setEditingId(null);
                setForm(EMPTY_FORM);
                setFormError(null);
                setModalOpenRef.current(true);
            },
        },
        {
            target: 'pt-field-summary',
            title: t('promptTemplates.tour.fieldSummaryTitle'),
            content: t('promptTemplates.tour.fieldSummaryContent'),
            placement: 'right',
        },
        {
            target: 'pt-field-keyword',
            title: t('promptTemplates.tour.fieldKeywordTitle'),
            content: t('promptTemplates.tour.fieldKeywordContent'),
            placement: 'right',
        },
        {
            target: 'pt-field-faq',
            title: t('promptTemplates.tour.fieldFaqTitle'),
            content: t('promptTemplates.tour.fieldFaqContent'),
            placement: 'right',
        },
        {
            target: 'pt-field-short-answer',
            title: t('promptTemplates.tour.fieldShortAnswerTitle'),
            content: t('promptTemplates.tour.fieldShortAnswerContent'),
            placement: 'right',
        },
        {
            target: 'pt-field-true-false',
            title: t('promptTemplates.tour.fieldTrueFalseTitle'),
            content: t('promptTemplates.tour.fieldTrueFalseContent'),
            placement: 'right',
        },
    ];

    const handleTourFinish = () => {
        setTourActive(false);
        setModalOpen(false);
    };

    return (
        <div className={styles.page}>

            {/* ── Page header ── */}
            <div className={styles.ph} data-tour="pt-header">
                <div className={styles.phRow}>
                    <div>
                        <div className={styles.phTitle}>{t('promptTemplates.pageTitle')}</div>
                        <div className={styles.phSub}>
                            {t('promptTemplates.pageSub', { count: total })}
                        </div>
                    </div>
                    <div className={styles.phActs}>
                        <button
                            type="button"
                            className={styles.tourTriggerBtn}
                            onClick={() => setTourActive(true)}
                            title={t('promptTemplates.tour.triggerTitle')}
                        >
                            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                <circle cx="8" cy="8" r="6.25" />
                                <path d="M6.1 6.2a1.9 1.9 0 013.6.7c0 1.3-1.7 1.5-1.7 2.7M8 11.4v.1" />
                            </svg>
                            {t('promptTemplates.tour.triggerLabel')}
                        </button>
                        <button
                            className={styles.btnPrimary}
                            onClick={openCreate}
                            data-tour="pt-new-btn"
                        >
                            <svg width="12" height="12" viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                                <path d="M6 1.5v9M1.5 6h9" />
                            </svg>
                            {t('promptTemplates.newTemplate')}
                        </button>
                    </div>
                </div>
            </div>

            {/* ── Scrollable body ── */}
            <div className={styles.body}>
                <div className={styles.viewBody}>

                    {successMsg && <div className={styles.successBanner}>{successMsg}</div>}
                    {error && !modalOpen && <div className={styles.errorBanner}>{error}</div>}

                    {loading ? (
                        <Spinner label={t('promptTemplates.loading')} />
                    ) : (
                        <div className={styles.tableWrap} data-tour="pt-table">
                            <table className={styles.table}>
                                <thead>
                                    <tr>
                                        <th>{t('promptTemplates.table.name')}</th>
                                        <th>{t('promptTemplates.table.description')}</th>
                                        <th>{t('promptTemplates.table.updated')}</th>
                                        <th />
                                    </tr>
                                </thead>
                                <tbody>
                                    {templates.length === 0 ? (
                                        <tr>
                                            <td colSpan={4} className={styles.emptyRow}>
                                                {t('promptTemplates.table.empty')}
                                            </td>
                                        </tr>
                                    ) : templates.map((tpl, idx) => (
                                        <tr key={tpl.id}>
                                            <td className={styles.nameCell}>
                                                <div className={styles.nameCellInner}>
                                                    {tpl.name}
                                                    {tpl.is_default && (
                                                        <span className={styles.defaultBadge}>
                                                            {t('promptTemplates.table.defaultBadge')}
                                                        </span>
                                                    )}
                                                </div>
                                            </td>
                                            <td className={`${styles.muted} ${styles.descCell}`}>{tpl.description || '—'}</td>
                                            <td className={`${styles.muted} ${styles.mono}`}>{formatDate(tpl.updated_at)}</td>
                                            <td>
                                                <div className={styles.rowActs}>
                                                    {tpl.is_default ? (
                                                        // Default templates are read-only — show a View button
                                                        <button
                                                            className={`${styles.btn} ${styles.btnSm} ${styles.btnView}`}
                                                            onClick={() => openView(tpl)}
                                                            title={t('promptTemplates.table.defaultTooltip')}
                                                            {...(idx === 0 ? { 'data-tour': 'pt-edit-btn' } : {})}
                                                        >
                                                            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                                                <path d="M1.5 8S4 3 8 3s6.5 5 6.5 5S12 13 8 13 1.5 8 1.5 8z" />
                                                                <circle cx="8" cy="8" r="2" />
                                                            </svg>
                                                            {t('promptTemplates.table.view')}
                                                        </button>
                                                    ) : (
                                                        <>
                                                            <button
                                                                className={`${styles.btn} ${styles.btnSm}`}
                                                                onClick={() => openEdit(tpl)}
                                                                {...(idx === 0 ? { 'data-tour': 'pt-edit-btn' } : {})}
                                                            >
                                                                {t('promptTemplates.table.edit')}
                                                            </button>
                                                            <button
                                                                className={`${styles.btn} ${styles.btnSm} ${styles.btnDanger}`}
                                                                onClick={() => setDeleteTarget(tpl)}
                                                                disabled={deletingId === tpl.id}
                                                                {...(idx === 0 ? { 'data-tour': 'pt-delete-btn' } : {})}
                                                            >
                                                                {deletingId === tpl.id
                                                                    ? t('promptTemplates.table.deleting')
                                                                    : t('promptTemplates.table.delete')}
                                                            </button>
                                                        </>
                                                    )}
                                                </div>
                                            </td>
                                        </tr>
                                    ))}
                                </tbody>
                            </table>
                        </div>
                    )}
                </div>
            </div>

            {/* ── Create / Edit / View modal ── */}
            {modalOpen && (
                <div className={styles.overlay}>
                    <div className={styles.modal}>
                        <div className={styles.modalHead}>
                            <div className={styles.modalHeadInner}>
                                <div className={styles.modalTitle}>
                                    {modalMode === 'view'
                                        ? t('promptTemplates.modal.viewTitle')
                                        : modalMode === 'edit'
                                            ? t('promptTemplates.modal.editTitle')
                                            : t('promptTemplates.modal.createTitle')}
                                </div>
                                {modalMode === 'view' && (
                                    <span className={styles.viewModeBadge}>
                                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                            <rect x="2.5" y="6" width="11" height="8" rx="1.5" />
                                            <path d="M5 6V4.5a3 3 0 016 0V6" />
                                        </svg>
                                        {t('promptTemplates.modal.viewModeBadge')}
                                    </span>
                                )}
                            </div>
                            <button className={styles.closeBtn} onClick={closeModal} aria-label={t('promptTemplates.modal.closeAriaLabel')} type="button">
                                <svg width="14" height="14" viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                                    <path d="M2 2l10 10M12 2L2 12" />
                                </svg>
                            </button>
                        </div>

                        <form onSubmit={handleSubmit}>
                            <div className={styles.modalBody}>
                                {formError && <div className={styles.errorBanner}>{formError}</div>}

                                {FIELD_CONFIG.map(({ field, inputType, labelKey, placeholderKey, tourTarget }) => (
                                    <div
                                        className={styles.formGroup}
                                        key={field}
                                        {...(tourTarget ? { 'data-tour': tourTarget } : {})}
                                    >
                                        <label className={styles.label}>
                                            {t(`promptTemplates.modal.${labelKey}`)}
                                            {modalMode !== 'view' && (
                                                <>{' '}<span className={styles.required}>{t('promptTemplates.modal.required')}</span></>
                                            )}
                                        </label>
                                        {inputType === 'input' ? (
                                            <input
                                                className={`${styles.input} ${modalMode === 'view' ? styles.inputReadonly : ''}`}
                                                value={form[field]}
                                                onChange={handleChange(field)}
                                                placeholder={modalMode !== 'view' ? t(`promptTemplates.modal.${placeholderKey}`) : undefined}
                                                autoFocus={field === 'name' && !tourActive && modalMode !== 'view'}
                                                readOnly={modalMode === 'view'}
                                                required={modalMode !== 'view'}
                                            />
                                        ) : (
                                            <ResizableTextarea
                                                textareaClassName={`${styles.textarea} ${modalMode === 'view' ? styles.textareaReadonly : ''}`}
                                                wrapClassName={modalMode === 'view' ? styles.textareaWrapReadonly : ''}
                                                value={form[field]}
                                                onChange={handleChange(field)}
                                                placeholder={modalMode !== 'view' ? t(`promptTemplates.modal.${placeholderKey}`) : undefined}
                                                readOnly={modalMode === 'view'}
                                                required={modalMode !== 'view'}
                                            />
                                        )}
                                    </div>
                                ))}
                            </div>

                            <div className={styles.modalFoot}>
                                {modalMode === 'view' ? (
                                    <button type="button" className={styles.btnPrimary} onClick={closeModal}>
                                        {t('promptTemplates.modal.close')}
                                    </button>
                                ) : (
                                    <>
                                        <button type="button" className={styles.btn} onClick={closeModal} disabled={saving}>
                                            {t('promptTemplates.modal.cancel')}
                                        </button>
                                        <button type="submit" className={styles.btnPrimary} disabled={saving || !isFormValid}>
                                            {saving
                                                ? t('promptTemplates.modal.saving')
                                                : modalMode === 'edit'
                                                    ? t('promptTemplates.modal.saveChanges')
                                                    : t('promptTemplates.modal.createBtn')}
                                        </button>
                                    </>
                                )}
                            </div>
                        </form>
                    </div>
                </div>
            )}

            {/* ── Delete confirmation ── */}
            {deleteTarget && (
                <div className={styles.overlay}>
                    <div className={`${styles.modal} ${styles.modalSm}`}>
                        <div className={styles.modalHead}>
                            <div className={styles.modalTitle}>{t('promptTemplates.deleteModal.title')}</div>
                        </div>
                        <div className={styles.modalBody}>
                            <p
                                className={styles.confirmText}
                                dangerouslySetInnerHTML={{
                                    __html: t('promptTemplates.deleteModal.body', { name: deleteTarget.name }),
                                }}
                            />
                        </div>
                        <div className={styles.modalFoot}>
                            <button className={styles.btn} onClick={() => setDeleteTarget(null)}>
                                {t('promptTemplates.deleteModal.cancel')}
                            </button>
                            <button className={`${styles.btnPrimary} ${styles.btnDangerSolid}`} onClick={confirmDelete}>
                                {t('promptTemplates.deleteModal.confirm')}
                            </button>
                        </div>
                    </div>
                </div>
            )}

            {/* ── Tour ── */}
            <TourGuide
                steps={TOUR_STEPS}
                active={tourActive}
                onFinish={handleTourFinish}
            />
        </div>
    );
};

export default PromptTemplates;
