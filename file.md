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
        setEditingId(null);
        setForm(EMPTY_FORM);
        setFormError(null);
        dispatch(clearPromptTemplateError());
        setModalOpen(true);
    };

    const openEdit = (tpl: PromptTemplate) => {
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
                                            <td className={styles.nameCell}>{tpl.name}</td>
                                            <td className={`${styles.muted} ${styles.descCell}`}>{tpl.description || '—'}</td>
                                            <td className={`${styles.muted} ${styles.mono}`}>{formatDate(tpl.updated_at)}</td>
                                            <td>
                                                <div className={styles.rowActs}>
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

            {/* ── Create / Edit modal ── */}
            {modalOpen && (
                <div className={styles.overlay}>
                    <div className={styles.modal}>
                        <div className={styles.modalHead}>
                            <div className={styles.modalTitle}>
                                {editingId ? t('promptTemplates.modal.editTitle') : t('promptTemplates.modal.createTitle')}
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
                                            {' '}<span className={styles.required}>{t('promptTemplates.modal.required')}</span>
                                        </label>
                                        {inputType === 'input' ? (
                                            <input
                                                className={styles.input}
                                                value={form[field]}
                                                onChange={handleChange(field)}
                                                placeholder={t(`promptTemplates.modal.${placeholderKey}`)}
                                                autoFocus={field === 'name' && !tourActive}
                                                required
                                            />
                                        ) : (
                                            <textarea
                                                className={styles.textarea}
                                                value={form[field]}
                                                onChange={handleChange(field)}
                                                rows={3}
                                                placeholder={t(`promptTemplates.modal.${placeholderKey}`)}
                                                required
                                            />
                                        )}
                                    </div>
                                ))}
                            </div>

                            <div className={styles.modalFoot}>
                                <button type="button" className={styles.btn} onClick={closeModal} disabled={saving}>
                                    {t('promptTemplates.modal.cancel')}
                                </button>
                                <button type="submit" className={styles.btnPrimary} disabled={saving || !isFormValid}>
                                    {saving
                                        ? t('promptTemplates.modal.saving')
                                        : editingId
                                            ? t('promptTemplates.modal.saveChanges')
                                            : t('promptTemplates.modal.createBtn')}
                                </button>
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
  resize: vertical;
  min-height: 64px;
  line-height: 1.5;
  @include m.mono;
  font-size: 12px;
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













{
  "promptTemplates": {
    "pageTitle": "Prompt Templates",
    "pageSub_one": "{{count}} template · reusable prompts for summary, keyword & FAQ generation",
    "pageSub_other": "{{count}} templates · reusable prompts for summary, keyword & FAQ generation",
    "newTemplate": "New Template",
    "loading": "Loading templates…",
    "createdSuccess": "Template created.",
    "updatedSuccess": "Template updated.",
    "deletedSuccess": "Template deleted.",
    "genericError": "Something went wrong. Please try again.",
    "allFieldsRequired": "All fields are required.",
    "table": {
      "name": "Name",
      "description": "Description",
      "updated": "Updated",
      "empty": "No templates yet — create your first one to standardise summary, keyword and FAQ prompts.",
      "edit": "Edit",
      "delete": "Delete",
      "deleting": "Deleting…"
    },
    "modal": {
      "createTitle": "New Template",
      "editTitle": "Edit Template",
      "name": "Name",
      "namePlaceholder": "e.g. Lecture Summary — Default",
      "description": "Description",
      "descPlaceholder": "Short description of when to use this template",
      "summaryPrompt": "Summary Prompt",
      "summaryPlaceholder": "Instructions used to generate the summary",
      "keywordPrompt": "Keyword Prompt",
      "keywordPlaceholder": "Instructions used to extract keywords",
      "faqPrompt": "FAQ Prompt",
      "faqPlaceholder": "Instructions used to generate FAQs",
      "shortAnswerPrompt": "Short Answer Prompt",
      "shortAnswerPlaceholder": "Instructions used to generate short answer questions",
      "trueFalsePrompt": "True / False Prompt",
      "trueFalsePlaceholder": "Instructions used to generate true/false questions",
      "required": "*",
      "cancel": "Cancel",
      "saving": "Saving…",
      "saveChanges": "Save Changes",
      "createBtn": "Create Template",
      "closeAriaLabel": "Close"
    },
    "deleteModal": {
      "title": "Delete Template",
      "body": "Delete <strong>{{name}}</strong>? This can't be undone.",
      "cancel": "Cancel",
      "confirm": "Delete"
    },
    "tour": {
      "triggerLabel": "Take a tour",
      "triggerTitle": "Take a guided tour of this page",
      "headerTitle": "Prompt Templates",
      "headerContent": "This page lets you manage reusable prompt templates. Each template bundles seven prompts — summary, keyword, FAQ, short-answer and true/false — so you can swap them without touching the pipeline configuration.",
      "newBtnTitle": "Create a template",
      "newBtnContent": "Click \"New Template\" to open the creation form. You can have as many templates as you need and switch between them freely.",
      "tableTitle": "Template list",
      "tableContent": "All saved templates appear here. The table shows the name, a short description and when the template was last edited.",
      "editBtnTitle": "Edit a template",
      "editBtnContent": "Click Edit on any row to open the template in the form and update any of its prompts.",
      "deleteBtnTitle": "Delete a template",
      "deleteBtnContent": "Click Delete to remove a template permanently. A confirmation dialog will appear before anything is deleted.",
      "fieldNameTitle": "Name & Description",
      "fieldNameContent": "Give the template a clear name and a short description so your team knows when to use it.",
      "fieldSummaryTitle": "Summary Prompt",
      "fieldSummaryContent": "This prompt is sent to the AI when generating a lecture summary. Tailor the tone and length requirements here.",
      "fieldKeywordTitle": "Keyword Prompt",
      "fieldKeywordContent": "Controls how keywords and key concepts are extracted from the transcript.",
      "fieldFaqTitle": "FAQ Prompt",
      "fieldFaqContent": "Shapes the frequently-asked-questions that are generated from the lecture content.",
      "fieldShortAnswerTitle": "Short Answer Prompt",
      "fieldShortAnswerContent": "Used when generating short-answer assessment questions from the lecture.",
      "fieldTrueFalseTitle": "True / False Prompt",
      "fieldTrueFalseContent": "Used when generating true/false questions. Describe any specific format or difficulty requirements you want the AI to follow."
    }
  }
}













{
  "promptTemplates": {
    "pageTitle": "프롬프트 템플릿",
    "pageSub_one": "{{count}}개 템플릿 · 요약, 키워드 및 FAQ 생성을 위한 재사용 가능한 프롬프트",
    "pageSub_other": "{{count}}개 템플릿 · 요약, 키워드 및 FAQ 생성을 위한 재사용 가능한 프롬프트",
    "newTemplate": "새 템플릿",
    "loading": "템플릿 불러오는 중…",
    "createdSuccess": "템플릿이 생성되었습니다.",
    "updatedSuccess": "템플릿이 수정되었습니다.",
    "deletedSuccess": "템플릿이 삭제되었습니다.",
    "genericError": "오류가 발생했습니다. 다시 시도해 주세요.",
    "allFieldsRequired": "모든 필드를 입력해야 합니다.",
    "table": {
      "name": "이름",
      "description": "설명",
      "updated": "수정일",
      "empty": "아직 템플릿이 없습니다 — 첫 번째 템플릿을 생성하여 요약, 키워드 및 FAQ 프롬프트를 표준화하세요.",
      "edit": "수정",
      "delete": "삭제",
      "deleting": "삭제 중…"
    },
    "modal": {
      "createTitle": "새 템플릿",
      "editTitle": "템플릿 수정",
      "name": "이름",
      "namePlaceholder": "예: 강의 요약 — 기본",
      "description": "설명",
      "descPlaceholder": "이 템플릿을 사용하는 상황에 대한 짧은 설명",
      "summaryPrompt": "요약 프롬프트",
      "summaryPlaceholder": "요약 생성에 사용되는 지시사항",
      "keywordPrompt": "키워드 프롬프트",
      "keywordPlaceholder": "키워드 추출에 사용되는 지시사항",
      "faqPrompt": "FAQ 프롬프트",
      "faqPlaceholder": "FAQ 생성에 사용되는 지시사항",
      "shortAnswerPrompt": "단답형 프롬프트",
      "shortAnswerPlaceholder": "단답형 문항 생성에 사용되는 지시사항",
      "trueFalsePrompt": "진위형 프롬프트",
      "trueFalsePlaceholder": "진위형 문항 생성에 사용되는 지시사항",
      "required": "*",
      "cancel": "취소",
      "saving": "저장 중…",
      "saveChanges": "변경사항 저장",
      "createBtn": "템플릿 생성",
      "closeAriaLabel": "닫기"
    },
    "deleteModal": {
      "title": "템플릿 삭제",
      "body": "<strong>{{name}}</strong>을(를) 삭제하시겠습니까? 이 작업은 취소할 수 없습니다.",
      "cancel": "취소",
      "confirm": "삭제"
    },
    "tour": {
      "triggerLabel": "둘러보기",
      "triggerTitle": "이 페이지의 가이드 투어 시작",
      "headerTitle": "프롬프트 템플릿",
      "headerContent": "이 페이지에서 재사용 가능한 프롬프트 템플릿을 관리할 수 있습니다. 각 템플릿에는 요약, 키워드, FAQ, 단답형, 진위형 등 7가지 프롬프트가 포함되어 있어, 파이프라인 설정을 변경하지 않고도 자유롭게 교체할 수 있습니다.",
      "newBtnTitle": "템플릿 생성",
      "newBtnContent": "\"새 템플릿\"을 클릭하면 생성 폼이 열립니다. 필요한 만큼 템플릿을 만들고 자유롭게 전환할 수 있습니다.",
      "tableTitle": "템플릿 목록",
      "tableContent": "저장된 모든 템플릿이 여기에 표시됩니다. 이름, 짧은 설명, 마지막 수정 날짜를 확인할 수 있습니다.",
      "editBtnTitle": "템플릿 수정",
      "editBtnContent": "행의 수정 버튼을 클릭하면 해당 템플릿이 폼에 열려 프롬프트를 수정할 수 있습니다.",
      "deleteBtnTitle": "템플릿 삭제",
      "deleteBtnContent": "삭제 버튼을 클릭하면 템플릿이 영구적으로 제거됩니다. 삭제 전에 확인 대화상자가 표시됩니다.",
      "fieldNameTitle": "이름 및 설명",
      "fieldNameContent": "팀원들이 언제 사용해야 할지 알 수 있도록 명확한 이름과 짧은 설명을 입력하세요.",
      "fieldSummaryTitle": "요약 프롬프트",
      "fieldSummaryContent": "강의 요약을 생성할 때 AI에 전달되는 프롬프트입니다. 여기에서 톤과 길이 요구사항을 조정하세요.",
      "fieldKeywordTitle": "키워드 프롬프트",
      "fieldKeywordContent": "전사본에서 키워드와 핵심 개념을 추출하는 방식을 제어합니다.",
      "fieldFaqTitle": "FAQ 프롬프트",
      "fieldFaqContent": "강의 내용에서 생성되는 자주 묻는 질문의 형태를 결정합니다.",
      "fieldShortAnswerTitle": "단답형 프롬프트",
      "fieldShortAnswerContent": "강의에서 단답형 평가 문항을 생성할 때 사용됩니다.",
      "fieldTrueFalseTitle": "진위형 프롬프트",
      "fieldTrueFalseContent": "진위형 문항을 생성할 때 사용됩니다. AI가 따라야 할 특정 형식이나 난이도 요구사항을 기술하세요."
    }
  }
}
