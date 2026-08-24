// ═══════════════════════════════════════════════
// pages/PromptTemplates/PromptTemplates.tsx
// Content Analytics · Prompt Template management
// ═══════════════════════════════════════════════
import React, { useEffect, useLayoutEffect, useRef, useState } from 'react';
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
// AutoResizeTextarea
// Grows with content up to MAX_H, then scrolls.
// Works correctly inside scrollable modal bodies
// where the native resize grip is suppressed.
// ─────────────────────────────────────────────
const MAX_TEXTAREA_H = 320;

interface AutoResizeTextareaProps extends React.TextareaHTMLAttributes<HTMLTextAreaElement> {
    className?: string;
}

const AutoResizeTextarea = React.forwardRef<HTMLTextAreaElement, AutoResizeTextareaProps>(
    ({ onChange, className, value, ...props }, forwardedRef) => {
        const innerRef = useRef<HTMLTextAreaElement>(null);
        const ref = (forwardedRef as React.RefObject<HTMLTextAreaElement>) ?? innerRef;

        const adjust = () => {
            const el = ref.current;
            if (!el) return;
            el.style.height = 'auto';
            const next = Math.min(el.scrollHeight, MAX_TEXTAREA_H);
            el.style.height = `${next}px`;
            el.style.overflowY = el.scrollHeight > MAX_TEXTAREA_H ? 'auto' : 'hidden';
        };

        // Re-measure whenever value changes (e.g. when edit modal pre-fills)
        useLayoutEffect(() => { adjust(); });

        return (
            <textarea
                ref={ref}
                value={value}
                className={className}
                onChange={(e) => { adjust(); onChange?.(e); }}
                {...props}
            />
        );
    },
);

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
                                            <AutoResizeTextarea
                                                className={`${styles.textarea} ${modalMode === 'view' ? styles.textareaReadonly : ''}`}
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

















// ═══════════════════════════════════════════════
// PromptTemplates.module.scss
// Content Analytics · Prompt Template management
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

// Tour trigger pill — matches UploadInfer pattern
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

  svg { width: 13px; height: 13px; }

  &:hover {
    background: var(--bg3);
    color: var(--t1);
    border-color: var(--bdr3);
  }
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

// ── Table ────────────────────────────────────────
.tableWrap {
  overflow-x: auto;
  border: 1px solid var(--bdr);
  border-radius: var(--rl);
  background: var(--bg1);
  @include m.scrollbar;
}

.table {
  width: 100%;
  border-collapse: collapse;
  font-size: 12px;

  th {
    padding: 10px 16px;
    text-align: left;
    font-size: 10px;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.08em;
    color: var(--t2);
    background: var(--bg0);
    border-bottom: 1px solid var(--bdr);
    white-space: nowrap;
    @include m.mono;
  }

  td {
    padding: 11px 16px;
    border-bottom: 1px solid var(--bdr);
    color: var(--t0);
    vertical-align: middle;

    &:last-child {
      text-align: right;
    }
  }

  tr:last-child td {
    border-bottom: none;
  }

  tr:hover td {
    background: var(--bg2);
  }
}

.nameCell {
  font-weight: 600;
  color: var(--t0);
}

.nameCellInner {
  display: flex;
  align-items: center;
  gap: 7px;
  flex-wrap: wrap;
}

// "Default" badge shown next to the template name
.defaultBadge {
  display: inline-flex;
  align-items: center;
  padding: 1px 7px;
  border-radius: 99px;
  font-size: 10px;
  font-weight: 600;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  background: var(--bg3);
  border: 1px solid var(--bdr2);
  color: var(--t2);
  @include m.mono;
  white-space: nowrap;
}

// Info chip shown in the actions column for default (read-only) templates
.defaultInfo {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 4px 9px;
  border-radius: var(--r);
  border: 1px solid var(--bdr);
  background: var(--bg2);
  color: var(--t2);
  font-size: 11px;
  font-weight: 500;
  cursor: default;
  white-space: nowrap;

  svg {
    width: 13px;
    height: 13px;
    flex-shrink: 0;
    color: var(--blue);
  }
}

.descCell {
  max-width: 320px;
  @include m.truncate;
}

.rowActs {
  display: inline-flex;
  gap: 6px;
}

.emptyRow {
  text-align: center !important;
  color: var(--t2);
  padding: 32px 16px !important;
  font-size: 12px;
}

.muted {
  color: var(--t2) !important;
}

.mono {
  @include m.mono;
  color: var(--t1) !important;
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

.btnSm {
  padding: 4px 10px;
  font-size: 11px;
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

.btnDanger {
  color: var(--red);
  border-color: var(--red-bdr);

  &:hover:not(:disabled) {
    background: var(--red-dim);
    color: var(--red);
    border-color: var(--red-bdr);
  }
}

// View button for default (read-only) templates
.btnView {
  color: var(--blue);
  border-color: var(--blue-bdr);
  background: var(--blue-dim);

  svg {
    width: 12px;
    height: 12px;
    flex-shrink: 0;
  }

  &:hover:not(:disabled) {
    filter: brightness(0.95);
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
.spinnerWrap {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 12px;
  padding: 80px 0;
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
  animation: ptSpin 0.7s linear infinite;
}

// ── Modal ─────────────────────────────────────────
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
  animation: ptFadeIn 0.15s ease-out;

  // Give a little more breathing room on short viewports
  @media (max-height: 700px) {
    padding: 12px;
    align-items: flex-start;
    overflow-y: auto;
    @include m.scrollbar;
  }
}

.modal {
  width: 100%;
  max-width: 560px;
  // Cap the modal at viewport height minus overlay padding
  max-height: calc(100vh - 48px);
  display: flex;
  flex-direction: column;
  // Without this the flex children can push past max-height
  min-height: 0;
  background: var(--bg1);
  border: 1px solid var(--bdr2);
  border-radius: var(--rxl);
  box-shadow: var(--shadow);
  animation: ptScaleIn 0.16s cubic-bezier(0.4, 0, 0.2, 1);

  // The <form> inside must also be a flex column so modalBody can flex-grow
  form {
    display: flex;
    flex-direction: column;
    flex: 1;
    min-height: 0;
  }
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

// Inner flex row: title + optional view-mode badge
.modalHeadInner {
  display: flex;
  align-items: center;
  gap: 10px;
  min-width: 0;
}

// "Read-only" lock badge shown next to the title in view mode
.viewModeBadge {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 2px 8px;
  border-radius: 99px;
  font-size: 10px;
  font-weight: 600;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  background: var(--bg3);
  border: 1px solid var(--bdr2);
  color: var(--t2);
  white-space: nowrap;
  @include m.mono;

  svg {
    width: 11px;
    height: 11px;
    flex-shrink: 0;
  }
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

  &:hover {
    background: var(--bg3);
    color: var(--t0);
  }
}

.modalBody {
  padding: 16px 18px;
  flex: 1;        // fill all space between header and footer
  min-height: 0;  // allows shrinking below content size so overflow-y kicks in
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 14px;
  @include m.scrollbar;
}

.modalFoot {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 8px;
  padding: 14px 18px;
  border-top: 1px solid var(--bdr);
  flex-shrink: 0;
}

.confirmText {
  font-size: 13px;
  color: var(--t1);
  line-height: 1.5;

  strong {
    color: var(--t0);
  }
}

// ── Form ──────────────────────────────────────────
.formGroup {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.label {
  font-size: 11px;
  font-weight: 600;
  color: var(--t2);
  text-transform: uppercase;
  letter-spacing: 0.06em;
  @include m.mono;
}

.required {
  color: var(--red);
}

.input,
.textarea {
  width: 100%;
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

.textarea {
  // Height and overflow are driven by AutoResizeTextarea (JS).
  // min-height sets the starting size; the component grows up to 320px
  // then switches overflow-y to auto so the modal body can scroll.
  min-height: 80px;
  line-height: 1.5;
  @include m.mono;
  font-size: 12px;
  overflow-y: hidden; // JS overrides this to 'auto' once max-height is hit
  resize: none;       // native grip disabled — auto-resize replaces it
}

// Read-only textarea shown in view mode
.textareaReadonly {
  overflow-y: auto;
  cursor: default;
  opacity: 0.75;
  background: var(--bg2) !important;
  border-color: var(--bdr) !important;
  box-shadow: none !important;
  color: var(--t1) !important;
  resize: none;

  &:focus {
    border-color: var(--bdr) !important;
    box-shadow: none !important;
  }
}

// Read-only input shown in view mode
.inputReadonly {
  cursor: default;
  opacity: 0.75;
  background: var(--bg2) !important;
  border-color: var(--bdr) !important;
  box-shadow: none !important;
  color: var(--t1) !important;

  &:focus {
    border-color: var(--bdr) !important;
    box-shadow: none !important;
  }
}

// ── Animations ────────────────────────────────────
@keyframes ptFadeIn {
  from {
    opacity: 0;
  }

  to {
    opacity: 1;
  }
}

@keyframes ptScaleIn {
  from {
    opacity: 0;
    transform: translateY(6px) scale(0.98);
  }

  to {
    opacity: 1;
    transform: translateY(0) scale(1);
  }
}

@keyframes ptSpin {
  to {
    transform: rotate(360deg);
  }
}
