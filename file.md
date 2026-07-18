// ═══════════════════════════════════════════════
// pages/PromptTemplates/PromptTemplates.tsx
// Content Analytics · Prompt Template management
// ═══════════════════════════════════════════════
import React, { useEffect, useRef, useState } from 'react';
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
// Form state
// ─────────────────────────────────────────────
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

const formatDate = (iso: string) =>
  new Date(iso).toLocaleDateString('en-GB', { day: '2-digit', month: 'short', year: 'numeric' });

// ─────────────────────────────────────────────
// Spinner
// ─────────────────────────────────────────────
const Spinner: React.FC = () => (
  <div className={styles.spinnerWrap}>
    <div className={styles.spinner} />
    <span>Loading templates…</span>
  </div>
);

// ═════════════════════════════════════════════
// ROOT: PromptTemplates
// ═════════════════════════════════════════════
const PromptTemplates: React.FC = () => {
  const dispatch = useAppDispatch();
  const { templates, total, loading, saving, deletingId, error } = useAppSelector(s => s.promptTemplate);

  const [modalOpen, setModalOpen] = useState(false);
  const [editingId, setEditingId] = useState<number | null>(null);
  const [form, setForm] = useState<FormState>(EMPTY_FORM);
  const [formError, setFormError] = useState<string | null>(null);
  const [deleteTarget, setDeleteTarget] = useState<PromptTemplate | null>(null);
  const [successMsg, setSuccessMsg] = useState<string | null>(null);
  const [tourActive, setTourActive] = useState(false);

  // Ref so tour step callbacks can always read the latest setter without
  // closing over a stale value.
  const setModalOpenRef = useRef(setModalOpen);
  setModalOpenRef.current = setModalOpen;

  useEffect(() => {
    dispatch(fetchPromptTemplates());
  }, [dispatch]);

  // Auto-dismiss the success banner
  useEffect(() => {
    if (!successMsg) return;
    const t = setTimeout(() => setSuccessMsg(null), 3000);
    return () => clearTimeout(t);
  }, [successMsg]);

  // ── Modal helpers ──
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
    if (!isFormValid) {
      setFormError('All fields are required.');
      return;
    }
    setFormError(null);

    try {
      if (editingId) {
        await dispatch(updatePromptTemplate({ id: editingId, ...form })).unwrap();
        setSuccessMsg('Template updated.');
      } else {
        await dispatch(createPromptTemplate(form)).unwrap();
        setSuccessMsg('Template created.');
      }
      setModalOpen(false);
    } catch (err) {
      setFormError(typeof err === 'string' ? err : 'Something went wrong. Please try again.');
    }
  };

  const confirmDelete = async () => {
    if (!deleteTarget) return;
    try {
      await dispatch(deletePromptTemplate(deleteTarget.id)).unwrap();
      setSuccessMsg('Template deleted.');
    } catch {
      // surfaced via the error banner below
    } finally {
      setDeleteTarget(null);
    }
  };

  // ── Tour ──────────────────────────────────────
  // Steps that live inside the modal use onEnter to open/close it so the
  // target elements are actually present in the DOM when TourGuide
  // tries to measure them.
  const TOUR_STEPS: TourStep[] = [
    {
      target: 'pt-header',
      title: 'Prompt Templates',
      content:
        'This page lets you manage reusable prompt templates. Each template bundles seven prompts — summary, keyword, FAQ, short-answer and true/false — so you can swap them without touching the pipeline configuration.',
      placement: 'bottom',
    },
    {
      target: 'pt-new-btn',
      title: 'Create a template',
      content:
        'Click "New Template" to open the creation form. You can have as many templates as you need and switch between them freely.',
      placement: 'bottom',
    },
    {
      target: 'pt-table',
      title: 'Template list',
      content:
        'All saved templates appear here. The table shows the name, a short description and when the template was last edited. Templates marked as default are supplied by the platform and cannot be deleted.',
      placement: 'top',
    },
    {
      target: 'pt-edit-btn',
      title: 'Edit a template',
      content:
        'Click Edit on any row to open the template in the form and update any of its prompts.',
      placement: 'left',
    },
    {
      target: 'pt-delete-btn',
      title: 'Delete a template',
      content:
        'Click Delete to remove a template permanently. A confirmation dialog will appear before anything is deleted.',
      placement: 'left',
    },
    // ── Modal field steps ──
    {
      target: 'pt-field-name',
      title: 'Name & Description',
      content:
        'Give the template a clear name and a short description so your team knows when to use it.',
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
      title: 'Summary Prompt',
      content:
        'This prompt is sent to the AI when generating a lecture summary. Tailor the tone and length requirements here.',
      placement: 'right',
    },
    {
      target: 'pt-field-keyword',
      title: 'Keyword Prompt',
      content:
        'Controls how keywords and key concepts are extracted from the transcript.',
      placement: 'right',
    },
    {
      target: 'pt-field-faq',
      title: 'FAQ Prompt',
      content:
        'Shapes the frequently-asked-questions that are generated from the lecture content.',
      placement: 'right',
    },
    {
      target: 'pt-field-short-answer',
      title: 'Short Answer Prompt',
      content:
        'Used when generating short-answer assessment questions from the lecture.',
      placement: 'right',
    },
    {
      target: 'pt-field-true-false',
      title: 'True / False Prompt',
      content:
        'Used when generating true/false questions. Describe any specific format or difficulty requirements you want the AI to follow.',
      placement: 'right',
    },
  ];

  const handleTourFinish = () => {
    setTourActive(false);
    // Close the modal if the tour opened it
    setModalOpen(false);
  };

  return (
    <div className={styles.page}>

      {/* ── Page header ── */}
      <div className={styles.ph} data-tour="pt-header">
        <div className={styles.phRow}>
          <div>
            <div className={styles.phTitle}>Prompt Templates</div>
            <div className={styles.phSub}>
              {total} template{total === 1 ? '' : 's'} · reusable prompts for summary, keyword & FAQ generation
            </div>
          </div>
          <div className={styles.phActs}>
            {/* Tour trigger */}
            <button
              className={styles.btnTour}
              onClick={() => setTourActive(true)}
              title="Take a guided tour of this page"
            >
              <svg width="13" height="13" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                <circle cx="8" cy="8" r="6.5" />
                <path d="M6.5 6.2C6.7 5.3 7.3 4.8 8 4.8s1.5.6 1.5 1.4C9.5 7.5 8 8 8 9.2" />
                <circle cx="8" cy="11.2" r=".6" fill="currentColor" />
              </svg>
              Tour
            </button>

            <button
              className={styles.btnPrimary}
              onClick={openCreate}
              data-tour="pt-new-btn"
            >
              <svg width="12" height="12" viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                <path d="M6 1.5v9M1.5 6h9" />
              </svg>
              New Template
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
            <Spinner />
          ) : (
            <div className={styles.tableWrap} data-tour="pt-table">
              <table className={styles.table}>
                <thead>
                  <tr>
                    <th>Name</th>
                    <th>Description</th>
                    <th>Updated</th>
                    <th />
                  </tr>
                </thead>
                <tbody>
                  {templates.length === 0 ? (
                    <tr>
                      <td colSpan={4} className={styles.emptyRow}>
                        No templates yet — create your first one to standardise summary, keyword and FAQ prompts.
                      </td>
                    </tr>
                  ) : templates.map((tpl, idx) => (
                    <tr key={tpl.id}>
                      <td className={styles.nameCell}>{tpl.name}</td>
                      <td className={`${styles.muted} ${styles.descCell}`}>{tpl.description || '—'}</td>
                      <td className={`${styles.muted} ${styles.mono}`}>{formatDate(tpl.updated_at)}</td>
                      <td>
                        <div className={styles.rowActs}>
                          {/* Tag only the first row's buttons for the tour */}
                          <button
                            className={`${styles.btn} ${styles.btnSm}`}
                            onClick={() => openEdit(tpl)}
                            {...(idx === 0 ? { 'data-tour': 'pt-edit-btn' } : {})}
                          >
                            Edit
                          </button>
                          <button
                            className={`${styles.btn} ${styles.btnSm} ${styles.btnDanger}`}
                            onClick={() => setDeleteTarget(tpl)}
                            disabled={deletingId === tpl.id}
                            {...(idx === 0 ? { 'data-tour': 'pt-delete-btn' } : {})}
                          >
                            {deletingId === tpl.id ? 'Deleting…' : 'Delete'}
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
              <div className={styles.modalTitle}>{editingId ? 'Edit Template' : 'New Template'}</div>
              <button className={styles.closeBtn} onClick={closeModal} aria-label="Close" type="button">
                <svg width="14" height="14" viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                  <path d="M2 2l10 10M12 2L2 12" />
                </svg>
              </button>
            </div>

            <form onSubmit={handleSubmit}>
              <div className={styles.modalBody}>
                {formError && <div className={styles.errorBanner}>{formError}</div>}

                <div className={styles.formGroup} data-tour="pt-field-name">
                  <label className={styles.label}>Name <span className={styles.required}>*</span></label>
                  <input
                    className={styles.input}
                    value={form.name}
                    onChange={handleChange('name')}
                    placeholder="e.g. Lecture Summary — Default"
                    autoFocus={!tourActive}
                    required
                  />
                </div>

                <div className={styles.formGroup}>
                  <label className={styles.label}>Description <span className={styles.required}>*</span></label>
                  <input
                    className={styles.input}
                    value={form.description}
                    onChange={handleChange('description')}
                    placeholder="Short description of when to use this template"
                    required
                  />
                </div>

                <div className={styles.formGroup} data-tour="pt-field-summary">
                  <label className={styles.label}>Summary Prompt <span className={styles.required}>*</span></label>
                  <textarea
                    className={styles.textarea}
                    value={form.summary_prompt}
                    onChange={handleChange('summary_prompt')}
                    rows={3}
                    placeholder="Instructions used to generate the summary"
                    required
                  />
                </div>

                <div className={styles.formGroup} data-tour="pt-field-keyword">
                  <label className={styles.label}>Keyword Prompt <span className={styles.required}>*</span></label>
                  <textarea
                    className={styles.textarea}
                    value={form.keyword_prompt}
                    onChange={handleChange('keyword_prompt')}
                    rows={3}
                    placeholder="Instructions used to extract keywords"
                    required
                  />
                </div>

                <div className={styles.formGroup} data-tour="pt-field-faq">
                  <label className={styles.label}>FAQ Prompt <span className={styles.required}>*</span></label>
                  <textarea
                    className={styles.textarea}
                    value={form.faq_prompt}
                    onChange={handleChange('faq_prompt')}
                    rows={3}
                    placeholder="Instructions used to generate FAQs"
                    required
                  />
                </div>

                <div className={styles.formGroup} data-tour="pt-field-short-answer">
                  <label className={styles.label}>Short Answer Prompt <span className={styles.required}>*</span></label>
                  <textarea
                    className={styles.textarea}
                    value={form.short_answer_prompt}
                    onChange={handleChange('short_answer_prompt')}
                    rows={3}
                    placeholder="Instructions used to generate short answer questions"
                    required
                  />
                </div>

                <div className={styles.formGroup} data-tour="pt-field-true-false">
                  <label className={styles.label}>True / False Prompt <span className={styles.required}>*</span></label>
                  <textarea
                    className={styles.textarea}
                    value={form.true_false_prompt}
                    onChange={handleChange('true_false_prompt')}
                    rows={3}
                    placeholder="Instructions used to generate true/false questions"
                    required
                  />
                </div>
              </div>

              <div className={styles.modalFoot}>
                <button type="button" className={styles.btn} onClick={closeModal} disabled={saving}>
                  Cancel
                </button>
                <button type="submit" className={styles.btnPrimary} disabled={saving || !isFormValid}>
                  {saving ? 'Saving…' : editingId ? 'Save Changes' : 'Create Template'}
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
              <div className={styles.modalTitle}>Delete Template</div>
            </div>
            <div className={styles.modalBody}>
              <p className={styles.confirmText}>
                Delete <strong>{deleteTarget.name}</strong>? This can't be undone.
              </p>
            </div>
            <div className={styles.modalFoot}>
              <button className={styles.btn} onClick={() => setDeleteTarget(null)}>Cancel</button>
              <button className={`${styles.btnPrimary} ${styles.btnDangerSolid}`} onClick={confirmDelete}>
                Delete
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

// Ghost "Tour" button in the page header
.btnTour {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 6px 11px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  font-family: var(--font-ui);
  font-size: 12px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.12s;

  &:hover {
    background: var(--bg3);
    color: var(--t0);
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















// ═══════════════════════════════════════════════
// pages/UploadInfer/TourGuide.tsx
// Content Analytics · Lightweight guided product tour
// No external dependency — spotlight overlay + positioned tooltip,
// driven by `data-tour="<id>"` attributes placed on real UI elements.
// ═══════════════════════════════════════════════
import React, { useEffect, useLayoutEffect, useRef, useState } from 'react';
import styles from './TourGuide.module.scss';

export interface TourStep {
  target: string;              // matches an element's data-tour="<target>"
  title: string;
  content: string;
  placement?: 'top' | 'bottom' | 'left' | 'right';
  // Fired right before this step tries to measure its target — use this
  // to switch tabs / open panels so the target actually exists in the DOM.
  onEnter?: () => void;
}

interface TourGuideProps {
  steps: TourStep[];
  active: boolean;
  onFinish: () => void;
}

interface Rect { top: number; left: number; width: number; height: number; }

const PAD = 6; // spotlight padding around the target's real bounding box

const TourGuide: React.FC<TourGuideProps> = ({ steps, active, onFinish }) => {
  const [index, setIndex] = useState(0);
  const [rect, setRect] = useState<Rect | null>(null);
  const [ready, setReady] = useState(false);
  // Real tooltip size, measured after each render — starts with a rough
  // estimate for the very first paint, then locks to the actual size.
  const [size, setSize] = useState({ w: 320, h: 190 });
  const tooltipRef = useRef<HTMLDivElement>(null);

  const step = steps[index];
  const isLast = index === steps.length - 1;

  useEffect(() => {
    if (active) { setIndex(0); setReady(false); }
  }, [active]);

  // Measure the current step's target — runs the step's onEnter first
  // (e.g. switch tabs), then waits a tick for that to paint, then locates
  // and scrolls to the element before reading its rect.
  useLayoutEffect(() => {
    if (!active || !step) return;
    setReady(false);
    step.onEnter?.();

    let cancelled = false;
    const measure = () => {
      if (cancelled) return;
      const el = document.querySelector<HTMLElement>(`[data-tour="${step.target}"]`);
      if (!el) { setRect(null); setReady(true); return; }
      el.scrollIntoView({ block: 'nearest', inline: 'nearest' });
      requestAnimationFrame(() => {
        if (cancelled) return;
        const r = el.getBoundingClientRect();
        setRect({ top: r.top, left: r.left, width: r.width, height: r.height });
        setReady(true);
      });
    };
    // Two frames: one for onEnter's state update (e.g. tab switch) to
    // render, one more for layout to settle before measuring.
    const raf1 = requestAnimationFrame(() => requestAnimationFrame(measure));

    const onResize = () => measure();
    window.addEventListener('resize', onResize);
    window.addEventListener('scroll', onResize, true);
    return () => {
      cancelled = true;
      cancelAnimationFrame(raf1);
      window.removeEventListener('resize', onResize);
      window.removeEventListener('scroll', onResize, true);
    };
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [active, index]);

  // Lock in the tooltip's real rendered size — runs after every render
  // (cheap: only updates state, and thus re-renders, on an actual change).
  useLayoutEffect(() => {
    const el = tooltipRef.current;
    if (!el) return;
    const w = el.offsetWidth, h = el.offsetHeight;
    if (w && h && (w !== size.w || h !== size.h)) setSize({ w, h });
  });

  useEffect(() => {
    if (!active) return;
    const onKey = (e: KeyboardEvent) => {
      if (e.key === 'Escape') onFinish();
      if (e.key === 'ArrowRight') go(1);
      if (e.key === 'ArrowLeft') go(-1);
    };
    window.addEventListener('keydown', onKey);
    return () => window.removeEventListener('keydown', onKey);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [active, index]);

  if (!active || !step) return null;

  const go = (delta: number) => {
    const next = index + delta;
    if (next < 0) return;
    if (next >= steps.length) { onFinish(); return; }
    setIndex(next);
  };

  // ── Placement: pick whichever side actually has room, treating the
  // step's preferred side as a preference rather than a guarantee. This
  // is what stops a "top" or "right" placement from clipping off-screen
  // when its target sits near a viewport edge. ──
  const spotRect = rect ?? { top: -9999, left: -9999, width: 0, height: 0 };
  const vw = typeof window !== 'undefined' ? window.innerWidth : 1200;
  const vh = typeof window !== 'undefined' ? window.innerHeight : 800;
  const EDGE = 12;
  const GAP = PAD + 12;

  const spaceBelow = vh - (spotRect.top + spotRect.height);
  const spaceAbove = spotRect.top;
  const spaceRight = vw - (spotRect.left + spotRect.width);
  const spaceLeft = spotRect.left;
  const fits = {
    bottom: spaceBelow >= size.h + GAP + EDGE,
    top: spaceAbove >= size.h + GAP + EDGE,
    right: spaceRight >= size.w + GAP + EDGE,
    left: spaceLeft >= size.w + GAP + EDGE,
  };
  const preferred = step.placement;
  const placement: 'top' | 'bottom' | 'left' | 'right' =
    (preferred && fits[preferred]) ? preferred
      : fits.bottom ? 'bottom'
        : fits.top ? 'top'
          : fits.right ? 'right'
            : fits.left ? 'left'
              : 'bottom'; // nothing fits cleanly — fall back and let clamping do its best

  let top = 0, left = 0;
  if (placement === 'bottom') {
    top = spotRect.top + spotRect.height + GAP;
    left = spotRect.left + spotRect.width / 2 - size.w / 2;
  } else if (placement === 'top') {
    top = spotRect.top - GAP - size.h;
    left = spotRect.left + spotRect.width / 2 - size.w / 2;
  } else if (placement === 'left') {
    top = spotRect.top + spotRect.height / 2 - size.h / 2;
    left = spotRect.left - GAP - size.w;
  } else {
    top = spotRect.top + spotRect.height / 2 - size.h / 2;
    left = spotRect.left + spotRect.width + GAP;
  }
  // Final safety clamp regardless of placement — guarantees the tooltip
  // is always fully on-screen even in edge cases nothing above caught.
  top = Math.min(Math.max(top, EDGE), vh - size.h - EDGE);
  left = Math.min(Math.max(left, EDGE), vw - size.w - EDGE);

  return (
    <div className={styles.tourRoot} aria-live="polite">
      {/* Spotlight cutout — a box-shadow "hole" over the real target */}
      {rect && (
        <div
          className={styles.spotlight}
          style={{
            top: rect.top - PAD, left: rect.left - PAD,
            width: rect.width + PAD * 2, height: rect.height + PAD * 2,
          }}
        />
      )}
      {!rect && ready && <div className={styles.dimFallback} />}

      <div
        ref={tooltipRef}
        className={`${styles.tooltip} ${styles[`place_${placement}`]}`}
        style={{
          top, left,
          maxHeight: `calc(100vh - ${EDGE * 2}px)`,
          overflowY: 'auto',
          width: 320,
        }}
      >
        <div className={styles.tooltipStep}>{index + 1} / {steps.length}</div>
        <div className={styles.tooltipTitle}>{step.title}</div>
        <div className={styles.tooltipBody}>{step.content}</div>
        <div className={styles.tooltipFooter}>
          <button className={styles.skipBtn} onClick={onFinish}>Skip tour</button>
          <div className={styles.navBtns}>
            {index > 0 && (
              <button className={styles.backBtn} onClick={() => go(-1)}>Back</button>
            )}
            <button className={styles.nextBtn} onClick={() => go(1)}>
              {isLast ? 'Done' : 'Next'}
            </button>
          </div>
        </div>
      </div>
    </div>
  );
};

export default TourGuide;
















// ═══════════════════════════════════════════════
// TourGuide.module.scss
// Content Analytics · Guided tour overlay
// ═══════════════════════════════════════════════
@use '../../styles/mixins' as m;

.tourRoot {
  position: fixed;
  inset: 0;
  z-index: 500;
  pointer-events: none;
}

// The "hole" is created by a huge box-shadow spread around a
// transparent rect exactly matching the target — everything outside
// that rect gets dimmed, the target itself stays fully visible.
.spotlight {
  position: fixed;
  border-radius: 10px;
  box-shadow: 0 0 0 9999px rgba(0, 0, 0, 0.6);
  outline: 2px solid var(--blue);
  outline-offset: 2px;
  transition: top 0.25s ease, left 0.25s ease, width 0.25s ease, height 0.25s ease;
  pointer-events: none;
}

// Fallback full dim if a step's target can't be found in the DOM
.dimFallback {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.6);
}

.tooltip {
  position: fixed;
  z-index: 501;
  pointer-events: auto;
  background: var(--bg1);
  border: 1px solid var(--bdr2);
  border-radius: var(--rl);
  box-shadow: var(--shadow);
  padding: 16px;
  transition: top 0.25s ease, left 0.25s ease;
}

.tooltipStep {
  font-size: 10.5px;
  font-weight: 600;
  color: var(--t2);
  letter-spacing: 0.06em;
  text-transform: uppercase;
  @include m.mono;
  margin-bottom: 6px;
}

.tooltipTitle {
  font-size: 14.5px;
  font-weight: 600;
  color: var(--t0);
  margin-bottom: 6px;
}

.tooltipBody {
  font-size: 13px;
  line-height: 1.5;
  color: var(--t1);
  margin-bottom: 14px;
}

.tooltipFooter {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
}

.skipBtn {
  padding: 0;
  border: none;
  background: transparent;
  color: var(--t2);
  font-size: 12px;
  cursor: pointer;

  &:hover {
    color: var(--t1);
    text-decoration: underline;
  }
}

.navBtns {
  display: flex;
  align-items: center;
  gap: 8px;
}

.backBtn {
  height: 30px;
  padding: 0 12px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t1);
  font-size: 12.5px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.12s;

  &:hover {
    background: var(--bg3);
  }
}

.nextBtn {
  height: 30px;
  padding: 0 14px;
  border-radius: var(--r);
  border: 1px solid var(--blue);
  background: var(--blue-dim);
  color: var(--blue);
  font-size: 12.5px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.12s;

  &:hover {
    background: var(--blue);
    color: #fff;
  }
}

@media (max-width: 560px) {
  .tooltip {
    width: calc(100vw - 24px) !important;
    left: 12px !important;
    right: auto !important;
  }
}
