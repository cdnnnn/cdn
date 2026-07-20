// ═══════════════════════════════════════════════
// pages/UploadInfer/PromptTemplateAssociationModal.tsx
// Content Analytics · Bulk file ↔ prompt-template association
// ═══════════════════════════════════════════════
import React, { useCallback, useEffect, useMemo, useRef, useState } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import { updateFilePrompts, patchFilePromptTemplate } from '../../store/uploadSlice';
import { addToast } from '../../store/toastSlice';
import api from '../../services/api';
import styles from './PromptTemplateAssociationModal.module.scss';

interface PromptTemplateListItem {
    id: number; ms_user_id: number; name: string; description: string;
    summary_prompt: string; keywords_prompt: string; faq_prompt: string;
    short_answers_prompt: string; true_false_prompt: string;
    inserted_at: string; updated_at: string; is_default?: boolean;
}
interface PromptTemplateDetail {
    id: number; ms_user_id: number; name: string; description: string;
    summary_prompt: string; keyword_prompt?: string; keywords_prompt?: string;
    faq_prompt: string; short_answers_prompt?: string; true_false_prompt?: string;
    inserted_at: string; updated_at: string; is_default?: boolean;
}
interface Props { open: boolean; onClose: () => void; }

const NO_TEMPLATE = -1;

const PromptTemplateAssociationModal: React.FC<Props> = ({ open, onClose }) => {
    const dispatch = useAppDispatch();
    const { t } = useTranslation();
    const { serverFiles, dateFrom, dateTo } = useAppSelector(s => s.upload);

    const [templates, setTemplates] = useState<PromptTemplateListItem[]>([]);
    const [templatesLoading, setTemplatesLoading] = useState(false);
    const [templatesError, setTemplatesError] = useState<string | null>(null);
    const [selection, setSelection] = useState<Record<number, number>>({});
    const initialSelectionRef = useRef<Record<number, number>>({});
    const [bulkValue, setBulkValue] = useState<number>(NO_TEMPLATE);
    const [saving, setSaving] = useState(false);
    const [detailTemplateId, setDetailTemplateId] = useState<number | null>(null);
    const [detailData, setDetailData] = useState<PromptTemplateDetail | null>(null);
    const [detailLoading, setDetailLoading] = useState(false);
    const [detailError, setDetailError] = useState<string | null>(null);

    // ── Associate ALL — every file in the current date range, bypasses the
    //     per-row preview entirely and hits the backend directly. ──
    const [isAssociatingAll, setIsAssociatingAll] = useState(false);
    const [associateAllError, setAssociateAllError] = useState<string | null>(null);

    useEffect(() => {
        if (!open) return;
        const init: Record<number, number> = {};
        serverFiles.forEach(f => { init[f.id] = f.prompt_template_id ?? NO_TEMPLATE; });
        setSelection(init);
        initialSelectionRef.current = init;
        setBulkValue(NO_TEMPLATE);
        setDetailTemplateId(null); setDetailData(null); setDetailError(null);
        setAssociateAllError(null);

        let cancelled = false;
        (async () => {
            setTemplatesLoading(true); setTemplatesError(null);
            try {
                const res = await api.get('/prompt_template/list_template');
                const list: PromptTemplateListItem[] = (res.data as any)?.templates ?? [];
                if (!cancelled) setTemplates(Array.isArray(list) ? list : []);
            } catch {
                if (!cancelled) setTemplatesError(t('uploadInfer.templateModal.loadFail'));
            } finally { if (!cancelled) setTemplatesLoading(false); }
        })();
        return () => { cancelled = true; };
        // eslint-disable-next-line react-hooks/exhaustive-deps
    }, [open]);

    useEffect(() => {
        if (!open) return;
        const onKey = (e: KeyboardEvent) => { if (e.key === 'Escape') { e.preventDefault(); e.stopPropagation(); } };
        window.addEventListener('keydown', onKey, true);
        return () => window.removeEventListener('keydown', onKey, true);
    }, [open]);

    const dirtyIds = useMemo(() => {
        const init = initialSelectionRef.current;
        return Object.keys(selection).map(Number).filter(id => selection[id] !== init[id]);
    }, [selection]);
    const isDirty = dirtyIds.length > 0;

    const handleRowChange = useCallback((fileId: number, templateId: number) => {
        setSelection(prev => ({ ...prev, [fileId]: templateId }));
    }, []);

    const handleApplyToAll = useCallback(() => {
        setSelection(prev => {
            const next = { ...prev };
            Object.keys(next).forEach(k => { next[Number(k)] = bulkValue; });
            return next;
        });
    }, [bulkValue]);

    const handleViewDetails = useCallback(async (templateId: number) => {
        if (detailTemplateId === templateId) { setDetailTemplateId(null); setDetailData(null); setDetailError(null); return; }
        setDetailTemplateId(templateId); setDetailData(null); setDetailError(null); setDetailLoading(true);
        try {
            const res = await api.get(`/prompt_template/${templateId}`);
            const d: PromptTemplateDetail | undefined = (res.data as any);
            if (d && d.id !== undefined) setDetailData(d);
            else setDetailError(t('uploadInfer.templateModal.noDetailsReturned'));
        } catch { setDetailError(t('uploadInfer.templateModal.detailLoadFail')); }
        finally { setDetailLoading(false); }
    }, [detailTemplateId, t]);

    const handleSave = useCallback(async () => {
        if (!isDirty || saving) return;
        setSaving(true);
        const associateGroups: Record<number, number[]> = {};
        const disassociateIds: number[] = [];
        for (const id of dirtyIds) {
            const v = selection[id];
            if (v === NO_TEMPLATE) { disassociateIds.push(id); continue; }
            if (!associateGroups[v]) associateGroups[v] = [];
            associateGroups[v].push(id);
        }
        const entries = Object.entries(associateGroups);
        if (entries.length === 0 && disassociateIds.length === 0) { setSaving(false); onClose(); return; }
        try {
            const calls: Promise<unknown>[] = entries.map(([templateIdStr, fileIds]) =>
                api.post('/prompt_template/associate', { file_ids: fileIds, template_id: Number(templateIdStr), all: false })
            );
            if (disassociateIds.length > 0) {
                calls.push(api.post('/prompt_template/disassociate', { all: false, file_ids: disassociateIds }));
            }
            await Promise.all(calls);
            const templatePatch: Record<number, number | null> = {};
            entries.forEach(([templateIdStr, fileIds]) => {
                const tmpl = templates.find(t => t.id === Number(templateIdStr));
                fileIds.forEach(id => { templatePatch[id] = Number(templateIdStr); });
                if (!tmpl) return;
                dispatch(updateFilePrompts({
                    fileIds,
                    summaryPrompt: tmpl.summary_prompt,
                    keywordsPrompt: tmpl.keywords_prompt,
                    faqPrompt: tmpl.faq_prompt,
                    shortAnswerPrompt: tmpl.short_answers_prompt,
                    trueFalsePrompt: tmpl.true_false_prompt,
                }));
            });
            if (disassociateIds.length > 0) {
                disassociateIds.forEach(id => { templatePatch[id] = null; });
                dispatch(updateFilePrompts({
                    fileIds: disassociateIds,
                    summaryPrompt: '', keywordsPrompt: '', faqPrompt: '',
                    shortAnswerPrompt: '', trueFalsePrompt: '',
                }));
            }
            dispatch(patchFilePromptTemplate(templatePatch));
            dispatch(addToast(t('uploadInfer.templateModal.saveSuccess'), 'success'));
            onClose();
        } catch {
            dispatch(addToast(t('uploadInfer.templateModal.saveFail'), 'error'));
        } finally { setSaving(false); }
    }, [isDirty, saving, dirtyIds, selection, templates, dispatch, onClose, t]);

    // ── Associate ALL files in [dateFrom, dateTo] with bulkValue — skips the
    //     per-row preview/save flow and applies directly on the backend. ──
    const handleAssociateAll = useCallback(async () => {
        if (bulkValue === NO_TEMPLATE || saving || isAssociatingAll) return;
        setIsAssociatingAll(true);
        setAssociateAllError(null);
        try {
            await api.post('/prompt_template/associate', {
                template_id: bulkValue,
                all: true,
                start_date: dateFrom,
                end_date: dateTo,
            });
            const tmpl = templates.find(x => x.id === bulkValue);
            // Best-effort local patch for whatever's currently loaded — files on
            // other pages will reflect the change next time they're fetched.
            const templatePatch: Record<number, number | null> = {};
            const fileIds = serverFiles.map(f => f.id);
            fileIds.forEach(id => { templatePatch[id] = bulkValue; });
            dispatch(patchFilePromptTemplate(templatePatch));
            if (tmpl) {
                dispatch(updateFilePrompts({
                    fileIds,
                    summaryPrompt: tmpl.summary_prompt,
                    keywordsPrompt: tmpl.keywords_prompt,
                    faqPrompt: tmpl.faq_prompt,
                    shortAnswerPrompt: tmpl.short_answers_prompt,
                    trueFalsePrompt: tmpl.true_false_prompt,
                }));
            }
            dispatch(addToast(t('uploadInfer.templateModal.associateAllSuccess'), 'success'));
            onClose();
        } catch {
            setAssociateAllError(t('uploadInfer.templateModal.associateAllFail'));
        } finally {
            setIsAssociatingAll(false);
        }
    }, [bulkValue, saving, isAssociatingAll, dateFrom, dateTo, serverFiles, templates, dispatch, onClose, t]);

    const handleCloseClick = useCallback(() => { if (saving) return; onClose(); }, [saving, onClose]);
    const handleBackdropClick = useCallback((e: React.MouseEvent) => { e.stopPropagation(); }, []);
    const handleDialogClick = useCallback((e: React.MouseEvent) => { e.stopPropagation(); }, []);

    if (!open) return null;

    const templateName = (id: number) => {
        if (id === NO_TEMPLATE) return t('uploadInfer.templateModal.noneRow');
        const tmpl = templates.find(x => x.id === id);
        return tmpl ? tmpl.name : `Template #${id}`;
    };

    return (
        <div className={styles.backdrop} onMouseDown={handleBackdropClick} role="presentation">
            <div className={styles.dialog} onMouseDown={handleDialogClick} role="dialog" aria-modal="true" aria-labelledby="template-modal-title">

                {/* Header */}
                <div className={styles.header}>
                    <div className={styles.headerText}>
                        <div className={styles.title} id="template-modal-title">{t('uploadInfer.templateModal.title')}</div>
                        <div className={styles.subtitle}>{t('uploadInfer.templateModal.subtitle')}</div>
                    </div>
                    <button className={styles.closeBtn} onClick={handleCloseClick} disabled={saving}
                        title={saving ? t('uploadInfer.templateModal.closeSaving') : t('uploadInfer.templateModal.close')}
                        aria-label={t('uploadInfer.templateModal.close')}>
                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                            <path d="M4 4l8 8M12 4l-8 8" />
                        </svg>
                    </button>
                </div>

                {/* Bulk bar — applies to files currently loaded in this modal (preview, needs Save) */}
                <div className={styles.bulkBar}>
                    <span className={styles.bulkLabel}>{t('uploadInfer.templateModal.applyToAll')}</span>
                    <select className={styles.bulkSelect} value={bulkValue}
                        onChange={e => setBulkValue(Number(e.target.value))}
                        disabled={saving || templatesLoading || serverFiles.length === 0}>
                        <option value={NO_TEMPLATE}>{t('uploadInfer.templateModal.noneOption')}</option>
                        {templates.map(tmpl => <option key={tmpl.id} value={tmpl.id}>{tmpl.name}</option>)}
                    </select>
                    <button className={styles.bulkApplyBtn} onClick={handleApplyToAll} disabled={saving || serverFiles.length === 0}>
                        {t('uploadInfer.templateModal.applyBtn')}
                    </button>
                    <span className={styles.bulkHint}>
                        {isDirty
                            ? t('uploadInfer.templateModal.changes', { count: dirtyIds.length })
                            : t('uploadInfer.templateModal.noChanges')}
                    </span>
                </div>

                {/* Associate ALL bar — applies immediately to every file in the date
                    range on the backend, including files not currently loaded here */}
                <div className={styles.associateAllBar}>
                    <span className={styles.bulkLabel}>
                        {t('uploadInfer.templateModal.associateAllLabel', { from: dateFrom, to: dateTo })}
                    </span>
                    <button
                        className={styles.associateAllBtn}
                        onClick={handleAssociateAll}
                        disabled={bulkValue === NO_TEMPLATE || saving || isAssociatingAll}
                    >
                        {isAssociatingAll ? <span className={styles.spinnerSm} /> : t('uploadInfer.templateModal.associateAllBtn')}
                    </button>
                    {associateAllError && <span className={styles.associateAllError}>{associateAllError}</span>}
                </div>

                {/* Body */}
                <div className={styles.body}>
                    <div className={`${styles.listPane} ${detailTemplateId !== null ? styles.listPaneNarrow : ''}`}>
                        <div className={styles.listHead}>
                            <div className={styles.colFile}>{t('uploadInfer.templateModal.colFile')}</div>
                            <div className={styles.colTemplate}>{t('uploadInfer.templateModal.colTemplate')}</div>
                            <div className={styles.colActions} />
                        </div>

                        {templatesLoading && (
                            <div className={styles.listState}>
                                <div className={styles.spinner} /><span>{t('uploadInfer.templateModal.loadingTemplates')}</span>
                            </div>
                        )}
                        {templatesError && <div className={`${styles.listState} ${styles.errorState}`}>{templatesError}</div>}
                        {!templatesLoading && serverFiles.length === 0 && (
                            <div className={styles.listState}>{t('uploadInfer.templateModal.noFiles')}</div>
                        )}

                        {!templatesLoading && !templatesError && serverFiles.map(f => {
                            const current = selection[f.id] ?? NO_TEMPLATE;
                            const initial = initialSelectionRef.current[f.id] ?? NO_TEMPLATE;
                            const rowDirty = current !== initial;
                            const hasTemplate = current !== NO_TEMPLATE;
                            return (
                                <div key={f.id} className={`${styles.row} ${rowDirty ? styles.rowDirty : ''}`}>
                                    <div className={styles.colFile} title={f.original_name}>
                                        <span className={styles.fileName}>{f.original_name}</span>
                                        {initial !== NO_TEMPLATE && (
                                            <span className={styles.currentBadge} title={t('uploadInfer.templateModal.mapped')}>
                                                {t('uploadInfer.templateModal.mapped')}
                                            </span>
                                        )}
                                    </div>
                                    <div className={styles.colTemplate}>
                                        <select className={styles.rowSelect} value={current}
                                            onChange={e => handleRowChange(f.id, Number(e.target.value))} disabled={saving}>
                                            <option value={NO_TEMPLATE}>{t('uploadInfer.templateModal.noneRow')}</option>
                                            {templates.map(tmpl => <option key={tmpl.id} value={tmpl.id}>{tmpl.name}</option>)}
                                        </select>
                                    </div>
                                    <div className={styles.colActions}>
                                        {hasTemplate && (
                                            <button className={styles.viewBtn} onClick={() => handleViewDetails(current)}
                                                disabled={saving} title={t('uploadInfer.templateModal.viewDetails')}>
                                                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                                    <path d="M1.5 8s2.5-4.5 6.5-4.5S14.5 8 14.5 8s-2.5 4.5-6.5 4.5S1.5 8 1.5 8z" />
                                                    <circle cx="8" cy="8" r="2" />
                                                </svg>
                                            </button>
                                        )}
                                        {initial !== NO_TEMPLATE && current !== NO_TEMPLATE && (
                                            <button className={styles.disBtn} onClick={() => handleRowChange(f.id, NO_TEMPLATE)}
                                                disabled={saving} title={t('uploadInfer.templateModal.disassociate')}
                                                aria-label={t('uploadInfer.templateModal.disassociate')}>
                                                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                                    <path d="M6.3 9.7L4.5 11.5a2 2 0 01-2.83-2.83L3.5 6.8M9.7 6.3l1.8-1.8a2 2 0 012.83 2.83L12.5 9.2M6 10l4-4" />
                                                </svg>
                                            </button>
                                        )}
                                    </div>
                                </div>
                            );
                        })}
                    </div>

                    {/* Detail drawer */}
                    {detailTemplateId !== null && (
                        <div className={styles.detailPane}>
                            <div className={styles.detailHead}>
                                <div className={styles.detailTitle}>{detailData?.name ?? templateName(detailTemplateId)}</div>
                                <button className={styles.detailClose}
                                    onClick={() => { setDetailTemplateId(null); setDetailData(null); }}
                                    title={t('uploadInfer.templateModal.detailClose')}
                                    aria-label={t('uploadInfer.templateModal.detailClose')}>
                                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                                        <path d="M4 4l8 8M12 4l-8 8" />
                                    </svg>
                                </button>
                            </div>
                            {detailLoading && (
                                <div className={styles.listState}>
                                    <div className={styles.spinner} /><span>{t('uploadInfer.templateModal.loadingDetails')}</span>
                                </div>
                            )}
                            {detailError && <div className={`${styles.listState} ${styles.errorState}`}>{detailError}</div>}
                            {detailData && (
                                <>
                                    <div className={styles.detailMeta}>
                                        <div><span className={styles.metaLabel}>{t('uploadInfer.templateModal.metaId')}</span> {detailData.id}</div>
                                        <div><span className={styles.metaLabel}>{t('uploadInfer.templateModal.metaUpdated')}</span> {detailData.updated_at}</div>
                                        {detailData.description && (
                                            <div><span className={styles.metaLabel}>{t('uploadInfer.templateModal.metaDescription')}</span> {detailData.description}</div>
                                        )}
                                    </div>
                                    <div className={styles.promptBlocks}>
                                        {[
                                            { label: t('uploadInfer.templateModal.summaryPrompt'), text: detailData.summary_prompt },
                                            { label: t('uploadInfer.templateModal.keywordPrompt'), text: detailData.keywords_prompt ?? detailData.keyword_prompt ?? '—' },
                                            { label: t('uploadInfer.templateModal.faqPrompt'), text: detailData.faq_prompt },
                                            { label: t('uploadInfer.templateModal.shortAnswerPrompt'), text: detailData.short_answers_prompt ?? '—' },
                                            { label: t('uploadInfer.templateModal.trueFalsePrompt'), text: detailData.true_false_prompt ?? '—' },
                                        ].map(block => (
                                            <div key={block.label} className={styles.promptBlock}>
                                                <div className={styles.promptLabel}>{block.label}</div>
                                                <div className={styles.promptText}>{block.text || '—'}</div>
                                            </div>
                                        ))}
                                    </div>
                                </>
                            )}
                        </div>
                    )}
                </div>

                {/* Footer */}
                <div className={styles.footer}>
                    {saving && (
                        <span className={styles.savingHint}>
                            <span className={styles.spinnerSm} />{t('uploadInfer.templateModal.savingHint')}
                        </span>
                    )}
                    <button className={`${styles.btn} ${styles.btnGhost}`} onClick={handleCloseClick} disabled={saving}>
                        {t('uploadInfer.templateModal.cancel')}
                    </button>
                    <button className={`${styles.btn} ${styles.btnPrimary}`} onClick={handleSave} disabled={!isDirty || saving}>
                        {saving ? t('uploadInfer.templateModal.saving') : t('uploadInfer.templateModal.save')}
                    </button>
                </div>
            </div>
        </div>
    );
};

export default PromptTemplateAssociationModal;











// ═══════════════════════════════════════════════
// pages/UploadInfer/PromptTemplateAssociationModal.module.scss
// ═══════════════════════════════════════════════

.backdrop {
    position: fixed;
    inset: 0;
    background: rgba(5, 10, 40, 0.62);
    backdrop-filter: blur(2px);
    z-index: 9000;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 32px;
}

.dialog {
    background: var(--bg1);
    border: 1px solid var(--bdr2);
    border-radius: var(--rxl);
    box-shadow: var(--shadow);
    width: 100%;
    max-width: 900px;
    max-height: 90vh;
    display: flex;
    flex-direction: column;
    overflow: hidden;
    font-family: var(--font-ui);
    color: var(--t0);
}

// ── Header ─────────────────────────────────────
.header {
    display: flex;
    align-items: flex-start;
    gap: 12px;
    padding: 18px 20px 14px;
    border-bottom: 1px solid var(--bdr);
}

.headerText {
    flex: 1;
    min-width: 0;
}

.title {
    font-size: 16px;
    font-weight: 600;
    color: var(--t0);
    letter-spacing: 0.1px;
}

.subtitle {
    margin-top: 3px;
    font-size: 12.5px;
    color: var(--t2);
}

.closeBtn {
    width: 30px;
    height: 30px;
    border-radius: var(--r);
    border: 1px solid var(--bdr2);
    background: var(--bg2);
    color: var(--t1);
    cursor: pointer;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    transition: background 0.15s, color 0.15s, border-color 0.15s;

    svg {
        width: 14px;
        height: 14px;
    }

    &:hover:not(:disabled) {
        background: var(--bg3);
        color: var(--t0);
        border-color: var(--bdr3);
    }

    &:disabled {
        opacity: 0.4;
        cursor: not-allowed;
    }
}

// ── Bulk bar ───────────────────────────────────
.bulkBar {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 12px 20px;
    background: var(--bg2);
    border-bottom: 1px solid var(--bdr);
    flex-wrap: wrap;
}

.bulkLabel {
    font-size: 12.5px;
    color: var(--t1);
    font-weight: 500;
}

.bulkSelect {
    flex: 0 1 280px;
    min-width: 200px;
    height: 32px;
    padding: 0 28px 0 10px;
    border-radius: var(--r);
    border: 1px solid var(--bdr2);
    background: var(--bg1);
    color: var(--t0);
    font-size: 13px;
    font-family: inherit;
    cursor: pointer;
    appearance: none;
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 12 12' fill='none' stroke='%237a90cc' stroke-width='1.5' stroke-linecap='round'%3E%3Cpath d='M3 5l3 3 3-3'/%3E%3C/svg%3E");
    background-repeat: no-repeat;
    background-position: right 10px center;
    background-size: 12px;

    &:focus-visible {
        outline: none;
        border-color: var(--teal-bdr);
        box-shadow: 0 0 0 2px var(--teal-dim);
    }

    &:disabled {
        opacity: 0.5;
        cursor: not-allowed;
    }
}

.bulkApplyBtn {
    height: 32px;
    padding: 0 14px;
    border-radius: var(--r);
    border: 1px solid var(--teal-bdr);
    background: var(--teal-dim);
    color: var(--teal);
    font-size: 12.5px;
    font-weight: 600;
    font-family: inherit;
    cursor: pointer;
    transition: background 0.15s;

    &:hover:not(:disabled) {
        background: var(--bg3);
    }

    &:disabled {
        opacity: 0.5;
        cursor: not-allowed;
    }
}

.bulkHint {
    margin-left: auto;
    font-size: 12px;
    color: var(--t2);
    font-style: italic;
}

// ── Associate ALL bar — distinct amber accent since this applies
//    immediately on the backend rather than previewing in the row list ──
.associateAllBar {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 10px 20px;
    background: rgba(240, 160, 48, 0.06);
    border-bottom: 1px solid var(--bdr);
    flex-wrap: wrap;
}

.associateAllBtn {
    height: 30px;
    padding: 0 14px;
    border-radius: var(--r);
    border: 1px solid rgba(240, 160, 48, 0.4);
    background: rgba(240, 160, 48, 0.12);
    color: var(--amber);
    font-size: 12.5px;
    font-weight: 600;
    font-family: inherit;
    cursor: pointer;
    transition: all 0.15s;
    white-space: nowrap;

    &:hover:not(:disabled) {
        background: var(--amber);
        border-color: var(--amber);
        color: #fff;
    }

    &:disabled {
        opacity: 0.4;
        cursor: not-allowed;
    }
}

.associateAllError {
    font-size: 12px;
    color: #ef4444;
}

// ── Body ───────────────────────────────────────
.body {
    flex: 1;
    min-height: 0;
    display: flex;
    overflow: hidden;
}

.listPane {
    flex: 1;
    min-width: 0;
    overflow-y: auto;
    padding: 6px 8px 12px;
    transition: flex-basis 0.2s;
}

.listPaneNarrow {
    flex: 1.1;
}

.listHead {
    display: grid;
    grid-template-columns: 1fr 220px 68px;
    gap: 10px;
    padding: 8px 12px;
    font-size: 11.5px;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    color: var(--t2);
    border-bottom: 1px solid var(--bdr);
    position: sticky;
    top: 0;
    background: var(--bg1);
    z-index: 1;
}

.row {
    display: grid;
    grid-template-columns: 1fr 220px 68px;
    gap: 10px;
    align-items: center;
    padding: 9px 12px;
    border-bottom: 1px solid var(--bdr);
    transition: background 0.12s;

    &:hover {
        background: var(--bg2);
    }
}

.rowDirty {
    background: var(--teal-dim);

    &:hover {
        background: var(--teal-dim);
    }
}

.colFile {
    display: flex;
    align-items: center;
    gap: 8px;
    min-width: 0;
}

.fileName {
    font-size: 13px;
    color: var(--t0);
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}

.currentBadge {
    flex-shrink: 0;
    padding: 2px 7px;
    font-size: 10.5px;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.4px;
    color: var(--teal);
    background: var(--teal-dim);
    border: 1px solid var(--teal-bdr);
    border-radius: 4px;
}

.colTemplate {
    display: flex;
    align-items: center;
}

.rowSelect {
    width: 100%;
    height: 30px;
    padding: 0 26px 0 9px;
    border-radius: var(--r);
    border: 1px solid var(--bdr2);
    background: var(--bg2);
    color: var(--t0);
    font-size: 12.5px;
    font-family: inherit;
    cursor: pointer;
    appearance: none;
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 12 12' fill='none' stroke='%237a90cc' stroke-width='1.5' stroke-linecap='round'%3E%3Cpath d='M3 5l3 3 3-3'/%3E%3C/svg%3E");
    background-repeat: no-repeat;
    background-position: right 8px center;
    background-size: 11px;

    &:focus-visible {
        outline: none;
        border-color: var(--teal-bdr);
        box-shadow: 0 0 0 2px var(--teal-dim);
    }

    &:disabled {
        opacity: 0.5;
        cursor: not-allowed;
    }
}

.colActions {
    display: flex;
    align-items: center;
    justify-content: flex-end;
    gap: 6px;
}

.viewBtn {
    width: 28px;
    height: 28px;
    flex-shrink: 0;
    border-radius: var(--r);
    border: 1px solid var(--bdr2);
    background: var(--bg2);
    color: var(--t1);
    cursor: pointer;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    padding: 0;
    transition: background 0.15s, color 0.15s, border-color 0.15s;

    svg {
        width: 14px;
        height: 14px;
    }

    &:hover:not(:disabled) {
        background: var(--teal-dim);
        color: var(--teal);
        border-color: var(--teal-bdr);
    }

    &:disabled {
        opacity: 0.5;
        cursor: not-allowed;
    }
}

// Icon-only, same 28×28 footprint as .viewBtn so the two sit flush inside
// the narrow actions column — the previous plain-text "Disassociate"
// button had no rule defined at all (fell back to unstyled browser
// button chrome) and would have overflowed this column anyway. Danger
// (red) hover distinguishes it from the neutral view action.
.disBtn {
    width: 28px;
    height: 28px;
    flex-shrink: 0;
    border-radius: var(--r);
    border: 1px solid var(--bdr2);
    background: var(--bg2);
    color: var(--t1);
    cursor: pointer;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    padding: 0;
    transition: background 0.15s, color 0.15s, border-color 0.15s;

    svg {
        width: 14px;
        height: 14px;
    }

    &:hover:not(:disabled) {
        background: rgba(239, 68, 68, 0.12);
        color: var(--red);
        border-color: rgba(239, 68, 68, 0.4);
    }

    &:disabled {
        opacity: 0.5;
        cursor: not-allowed;
    }
}

// ── Detail drawer ──────────────────────────────
.detailPane {
    width: 360px;
    flex-shrink: 0;
    border-left: 1px solid var(--bdr);
    background: var(--bg2);
    display: flex;
    flex-direction: column;
    overflow: hidden;
}

.detailHead {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 12px 14px;
    border-bottom: 1px solid var(--bdr);
}

.detailTitle {
    flex: 1;
    font-size: 13.5px;
    font-weight: 600;
    color: var(--t0);
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}

.detailClose {
    width: 26px;
    height: 26px;
    border-radius: 5px;
    border: 1px solid var(--bdr2);
    background: var(--bg1);
    color: var(--t1);
    cursor: pointer;
    display: inline-flex;
    align-items: center;
    justify-content: center;

    svg {
        width: 12px;
        height: 12px;
    }

    &:hover {
        background: var(--bg3);
        color: var(--t0);
    }
}

.detailMeta {
    padding: 10px 14px;
    display: flex;
    flex-direction: column;
    gap: 4px;
    font-size: 12px;
    color: var(--t1);
    border-bottom: 1px solid var(--bdr);
}

.metaLabel {
    color: var(--t2);
    margin-right: 6px;
}

// ── Prompt preview blocks (summary / keyword / faq) ──
.promptBlocks {
    flex: 1;
    overflow-y: auto;
    padding: 10px 14px 14px;
    display: flex;
    flex-direction: column;
    gap: 12px;
}

.promptBlock {
    display: flex;
    flex-direction: column;
    gap: 5px;
}

.promptLabel {
    font-size: 11px;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    color: var(--t2);
}

.promptText {
    font-size: 12.5px;
    line-height: 1.5;
    color: var(--t0);
    background: var(--bg1);
    border: 1px solid var(--bdr);
    border-radius: var(--r);
    padding: 9px 10px;
    white-space: pre-wrap;
    word-break: break-word;
}

// ── List states (loading/error/empty) ──────────
.listState {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 10px;
    padding: 28px 16px;
    font-size: 13px;
    color: var(--t2);
}

.errorState {
    color: var(--red);
}

.spinner {
    width: 16px;
    height: 16px;
    border: 2px solid var(--bdr2);
    border-top-color: var(--teal);
    border-radius: 50%;
    animation: spin 0.9s linear infinite;
}

.spinnerSm {
    width: 12px;
    height: 12px;
    border: 2px solid var(--bdr2);
    border-top-color: var(--teal);
    border-radius: 50%;
    animation: spin 0.9s linear infinite;
    display: inline-block;
    vertical-align: middle;
    margin-right: 6px;
}

@keyframes spin {
    to {
        transform: rotate(360deg);
    }
}

// ── Footer ─────────────────────────────────────
.footer {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 12px 20px;
    border-top: 1px solid var(--bdr);
    background: var(--bg1);
}

.savingHint {
    margin-right: auto;
    font-size: 12px;
    color: var(--amber);
    display: inline-flex;
    align-items: center;
}

.btn {
    height: 32px;
    padding: 0 16px;
    font-size: 13px;
    font-weight: 600;
    border-radius: var(--r);
    cursor: pointer;
    font-family: inherit;
    transition: background 0.15s, border-color 0.15s, color 0.15s;
    border: 1px solid transparent;

    &:disabled {
        opacity: 0.5;
        cursor: not-allowed;
    }
}

.btnGhost {
    background: transparent;
    color: var(--t1);
    border-color: var(--bdr2);
    margin-left: auto;

    &:hover:not(:disabled) {
        background: var(--bg2);
        color: var(--t0);
        border-color: var(--bdr3);
    }
}

.btnPrimary {
    background: var(--teal);
    color: var(--bg0);
    border-color: var(--teal);

    &:hover:not(:disabled) {
        filter: brightness(1.08);
    }
}

// ── Light theme tweaks ─────────────────────────
:global(html.light) {
    .backdrop {
        background: rgba(20, 20, 30, 0.45);
    }

    .btnPrimary {
        color: #fff;
    }
}














// ═══════════════════════════════════════════════
// FilePanel.module.scss
// Content Analytics · Upload panel — two sections
// ═══════════════════════════════════════════════
@use '../../styles/mixins' as m;

// ── Outer panel shell ─────────────────────────
.panel {
  width: 100%;
  flex: 1;
  border-right: none;
  display: flex;
  flex-direction: row;
  align-items: stretch;
  overflow: hidden;
  background: var(--bg0);
  position: relative;

  &::after {
    content: '';
    position: absolute;
    top: 0;
    right: 0;
    bottom: 0;
    width: 1px;
    background: linear-gradient(180deg,
        rgba(139, 92, 246, 0.0) 0%,
        rgba(139, 92, 246, 0.6) 20%,
        rgba(56, 196, 186, 0.7) 55%,
        rgba(240, 160, 48, 0.6) 85%,
        rgba(240, 160, 48, 0.0) 100%);
    pointer-events: none;
    z-index: 1;
  }
}

// ══════════════════════════════════════
// SECTION 1 — Step header + upload zone
// ══════════════════════════════════════
.step1 {
  width: 400px;
  flex-shrink: 0;
  border-bottom: none;
  border-right: 1px solid var(--bdr);
  background: var(--bg0);
  display: flex;
  flex-direction: column;
  overflow: hidden;
  position: relative;

  @media (max-width: 1499px) {
    width: 350px;
  }
}

// ── The upload zone itself, as a distinct card sitting in the sidebar ──
.step1Card {
  flex: 1;
  min-height: 0;
  display: flex;
  flex-direction: column;
  background: var(--bg1);
  overflow: hidden;
  box-shadow: var(--shadow-sm);
}

// Reopen affordance shown in the file-list header once the upload
// column has been collapsed to width: 0 (and its own header bar with it).
.reopenUploadBtn {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  margin-right: 8px;
  padding: 3px 9px;
  border-radius: 99px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  font-size: 12px;
  @include m.mono;
  cursor: pointer;
  transition: all 0.12s;

  svg { width: 10px; height: 10px; }

  &:hover {
    background: var(--bg3);
    color: var(--t1);
    border-color: var(--bdr3);
  }
}

// ── Step-1 header bar (original design) ──────
.step1Bar {
  padding: 10px 14px;
  display: flex;
  align-items: center;
  gap: 9px;
  border-bottom: none;
  background: var(--bg1);
  flex-shrink: 0;
  white-space: nowrap;
  position: relative;

  &::after {
    content: '';
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    height: 1px;
    background: linear-gradient(90deg,
        rgba(139, 92, 246, 0.0) 0%,
        rgba(139, 92, 246, 0.6) 20%,
        rgba(56, 196, 186, 0.7) 50%,
        rgba(240, 160, 48, 0.6) 80%,
        rgba(240, 160, 48, 0.0) 100%);
    pointer-events: none;
  }
}

.slbl {
  font-size: 12px;
  font-weight: 600;
  color: var(--t2);
  text-transform: uppercase;
  letter-spacing: 0.1em;
  @include m.mono;
}

// Collapse toggle button — matches original
.collapseBtn {
  display: flex;
  align-items: center;
  gap: 5px;
  margin-left: auto;
  padding: 3px 8px;
  border-radius: 99px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  font-size: 12px;
  @include m.mono;
  cursor: pointer;
  transition: all 0.12s;
  user-select: none;
  flex-shrink: 0;

  svg {
    width: 10px;
    height: 10px;
  }

  &:hover {
    background: var(--bg3);
    color: var(--t1);
    border-color: var(--bdr3);
  }
}

// ── Collapsible content area — fills the rest of the column, scrolls
// internally so a long browsed/uploading list doesn't blow out the height ──
.step1Content {
  flex: 1;
  min-height: 0;
  overflow-y: auto;
  @include m.scrollbar;
}

// ── Drop zone (original style) ────────────────
.dropzone {
  border: 1.5px dashed var(--bdr2);
  border-radius: var(--rxl);
  padding: 24px 20px;
  margin: 12px 14px;
  text-align: center;
  background: var(--bg1);
  cursor: pointer;
  transition: all 0.18s;
  position: relative;
  overflow: hidden;

  &::before {
    content: '';
    position: absolute;
    inset: 0;
    background: radial-gradient(ellipse at 50% 0%, rgba(139, 92, 246, 0.04), transparent 65%);
    pointer-events: none;
  }

  &:hover,
  &.dragOver {
    border-color: var(--blue);
    background: var(--bg2);
  }

  &.dragOver {
    background: var(--blue-dim);
  }
}

.dzIc {
  width: 38px;
  height: 38px;
  border-radius: 10px;
  background: var(--blue-dim);
  border: 1px solid var(--blue-bdr);
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 0 auto 11px;

  svg {
    width: 18px;
    height: 18px;
  }
}

.dzTitle {
  font-size: 14px;
  font-weight: 500;
  color: var(--t0);
  margin-bottom: 4px;
}

.dzSub {
  font-size: 13px;
  color: var(--t2);
  @include m.mono;
}

.chips {
  display: flex;
  justify-content: center;
  gap: 6px;
  flex-wrap: wrap;
  margin-top: 10px;
}

.chip {
  font-size: 12px;
  padding: 2px 8px;
  border-radius: 99px;
  background: var(--bg3);
  border: 1px solid var(--bdr);
  color: var(--t2);
  @include m.mono;
}

.dzActions {
  display: flex;
  gap: 8px;
  justify-content: center;
  margin-top: 11px;
  position: relative;
  z-index: 1;
}

// ── Preview / uploading ───────────────────────
.previewWrap {
  display: flex;
  flex-direction: column;
  animation: fadeSlide 0.18s ease;
}

.previewList {
  overflow-y: auto;
  padding: 8px 10px;
  display: flex;
  flex-direction: column;
  gap: 5px;
  @include m.scrollbar;
}

// Browsed file card
.fileCard {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 11px;
  background: var(--bg2);
  border: 1px solid var(--bdr);
  border-radius: var(--rxl);
  transition: border-color 0.15s, background 0.15s, box-shadow 0.15s;
  position: relative;
  overflow: hidden;

  // Left accent bar
  &::after {
    content: '';
    position: absolute;
    left: 0;
    top: 8px;
    bottom: 8px;
    width: 3px;
    border-radius: 0 3px 3px 0;
    background: var(--bdr2);
    transition: background 0.2s;
  }

  &.uploading {
    border-color: rgba(139, 92, 246, 0.35);
    background: rgba(139, 92, 246, 0.05);
    box-shadow: 0 0 0 1px rgba(139, 92, 246, 0.12) inset;

    &::after {
      background: linear-gradient(180deg, var(--blue), #a78bfa);
    }
  }

  &.success {
    border-color: var(--green-bdr);
    background: var(--green-dim);

    &::after {
      background: var(--green);
    }
  }

  &.failed {
    border-color: var(--red-bdr);
    background: var(--red-dim);

    &::after {
      background: var(--red);
    }
  }
}

// Extension badge (shared between browsed + uploaded cards)
.extBadge {
  width: 34px;
  height: 34px;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 10px;
  font-weight: 800;
  @include m.mono;
  flex-shrink: 0;
  letter-spacing: 0.03em;
  position: relative;
  z-index: 1;

  &.vtt {
    background: linear-gradient(135deg, rgba(91, 164, 239, 0.18) 0%, rgba(139, 92, 246, 0.14) 100%);
    color: var(--blue);
    border: 1px solid rgba(91, 164, 239, 0.3);
    box-shadow: 0 1px 4px rgba(91, 164, 239, 0.12);
  }

  &.srt {
    background: linear-gradient(135deg, rgba(52, 211, 153, 0.18) 0%, rgba(56, 196, 186, 0.14) 100%);
    color: var(--green);
    border: 1px solid rgba(52, 211, 153, 0.3);
    box-shadow: 0 1px 4px rgba(52, 211, 153, 0.12);
  }
}

.fileInfo {
  flex: 1;
  min-width: 0;
}

.fileNameRow {
  display: flex;
  align-items: center;
  gap: 5px;
  min-width: 0;
}

.fileName {
  font-size: 13px;
  font-weight: 500;
  color: var(--t0);
  @include m.truncate;
  margin-bottom: 1px;
  flex: 1;
  min-width: 0;
}

.fileId {
  font-size: 10px;
  color: var(--t2);
  @include m.mono;
  flex-shrink: 0;
  opacity: 0.65;
}

// ── File ID badge (shown before ext badge in card) ──
.fileIdBadge {
  font-size: 10px;
  font-weight: 700;
  font-family: var(--font-mono);
  color: #a78bfa;
  background: rgba(167, 139, 250, 0.12);
  border: 1px solid rgba(167, 139, 250, 0.3);
  border-radius: 4px;
  padding: 1px 5px;
  flex-shrink: 0;
  white-space: nowrap;
  letter-spacing: 0.02em;
}

.fileMeta {
  font-size: 12px;
  color: var(--t2);
  @include m.mono;
  display: flex;
  align-items: center;
  gap: 6px;
  margin-top: 2px;
  flex-wrap: wrap;
}

.fileSizeChip {
  font-size: 11px;
  padding: 1px 6px;
  border-radius: 99px;
  background: var(--bg3);
  border: 1px solid var(--bdr);
  color: var(--t2);
}

.fileStatusText {
  font-size: 11px;
  color: var(--t2);
  opacity: 0.7;
}

.fileStatusTextSuccess {
  font-size: 11px;
  color: var(--green);
  font-weight: 600;
}

.fileError {
  color: var(--red);
}

// Status icon (upload progress)
.statusIc {
  width: 18px;
  height: 18px;
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;

  svg {
    width: 15px;
    height: 15px;
  }

  &.pending {
    color: var(--t2);
  }

  &.uploading {
    color: var(--blue);
    animation: spin 0.9s linear infinite;
  }

  &.success {
    color: var(--green);
  }

  &.failed {
    color: var(--red);
  }
}

.removeBtn {
  width: 22px;
  height: 22px;
  border-radius: 5px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  padding: 0;
  transition: all 0.12s;

  svg {
    width: 10px;
    height: 10px;
  }

  &:hover {
    background: var(--red-dim);
    border-color: var(--red-bdr);
    color: var(--red);
  }
}

// Action bar (Upload / Cancel)
.step1Footer {
  flex-shrink: 0;
  display: flex;
  gap: 7px;
  padding: 12px 14px;
  border-top: 1px solid var(--bdr);
  background: var(--bg1);
}

.step1FooterBtn {
  flex: 1;
}

// Upload progress bar
.uploadSummary {
  width: 100%;
}

.uploadProgressBar {
  height: 3px;
  background: var(--bg3);
  border-radius: 99px;
  overflow: hidden;
  margin-bottom: 6px;
}

.uploadProgressFill {
  height: 100%;
  background: linear-gradient(90deg, var(--blue), #a78bfa);
  border-radius: 99px;
  transition: width 0.4s ease;
}

.uploadProgressLabel {
  font-size: 12px;
  color: var(--t2);
  @include m.mono;
}

.failCount {
  color: var(--red);
}

// ══════════════════════════════════════
// SECTION 2 — Uploaded files list
// ══════════════════════════════════════
.section2 {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  background: var(--bg0);
}

// Section 2 header — stacked: title row + action row/grid below
.section2Header {
  display: flex;
  flex-direction: column;
  width: 100%;
  box-sizing: border-box;
  padding: 0;
  border-bottom: none;
  background: var(--bg1);
  flex-shrink: 0;
  position: relative;
}

// Title row — checkbox + title + optional search icon
.section2TitleRow {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 14px 8px;
  width: 100%;
  box-sizing: border-box;
}

.section2Title {
  font-size: 12px;
  font-weight: 600;
  color: var(--t2);
  text-transform: uppercase;
  letter-spacing: 0.1em;
  @include m.mono;
  display: flex;
  align-items: center;
  gap: 7px;
}

.filesCount {
  font-size: 10px;
  font-weight: 700;
  color: var(--blue);
  background: var(--blue-dim);
  border: 1px solid var(--blue-bdr);
  padding: 1px 6px;
  border-radius: 99px;
}

// Cross-page selection total — shown next to filesCount while in
// select/delete/export mode so it's clear selections persist across pages.
.selectedTotalHint {
  font-size: 10px;
  font-weight: 700;
  color: var(--green);
  background: var(--green-dim);
  border: 1px solid var(--green-bdr);
  padding: 1px 6px;
  border-radius: 99px;
  margin-left: 4px;
}



// ── Delete icon button (used inside mode-bar) ─────────────────
.deleteIconBtn {
  width: 28px;
  height: 28px;
  border-radius: 7px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  flex-shrink: 0;
  transition: all 0.15s;

  svg {
    width: 14px;
    height: 14px;
  }

  &:hover:not(:disabled) {
    background: rgba(239, 68, 68, 0.1);
    border-color: rgba(239, 68, 68, 0.35);
    color: #ef4444;
  }

  &:disabled {
    opacity: 0.3;
    cursor: default;
  }
}

// Confirm-delete state — always red, solid
.deleteIconBtnConfirm {
  border-color: rgba(239, 68, 68, 0.45);
  background: rgba(239, 68, 68, 0.12);
  color: #ef4444;

  &:hover:not(:disabled) {
    background: rgba(239, 68, 68, 0.22);
    border-color: rgba(239, 68, 68, 0.7);
    box-shadow: 0 0 8px rgba(239, 68, 68, 0.25);
  }
}

.deleteIconBtnDisabled {
  opacity: 0.4 !important;
  cursor: default !important;
}

// ── Export icon button (used inside mode-bar) ─────────────────
.exportIconBtn {
  width: 28px;
  height: 28px;
  border-radius: 7px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  flex-shrink: 0;
  transition: all 0.15s;

  svg {
    width: 14px;
    height: 14px;
  }

  &:hover:not(:disabled) {
    background: rgba(78, 200, 122, 0.10);
    border-color: rgba(78, 200, 122, 0.35);
    color: var(--green);
  }

  &:disabled {
    opacity: 0.3;
    cursor: default;
  }
}

// Confirm-export state — always green, solid
.exportIconBtnConfirm {
  border-color: rgba(78, 200, 122, 0.45);
  background: rgba(78, 200, 122, 0.10);
  color: var(--green);

  &:hover:not(:disabled) {
    background: rgba(78, 200, 122, 0.20);
    border-color: rgba(78, 200, 122, 0.70);
    box-shadow: 0 0 8px rgba(78, 200, 122, 0.22);
  }
}

.exportIconBtnDisabled {
  opacity: 0.4 !important;
  cursor: default !important;
}

.miniSpinner {
  width: 12px;
  height: 12px;
  border: 1.5px solid rgba(239, 68, 68, 0.3);
  border-top-color: #ef4444;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
  flex-shrink: 0;
}

.miniSpinnerGreen {
  width: 12px;
  height: 12px;
  border: 1.5px solid rgba(78, 200, 122, 0.3);
  border-top-color: #4ec87a;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
  flex-shrink: 0;
}


// ── Cancel icon button (used inside mode-bar) ─────────────────
.cancelIconBtn {
  width: 28px;
  height: 28px;
  border-radius: 7px;
  border: 1px solid rgba(239, 68, 68, 0.35);
  background: rgba(239, 68, 68, 0.08);
  color: #ef4444;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  flex-shrink: 0;
  transition: all 0.15s;

  svg {
    width: 13px;
    height: 13px;
  }

  &:hover {
    background: rgba(239, 68, 68, 0.16);
    border-color: rgba(239, 68, 68, 0.6);
    box-shadow: 0 0 6px rgba(239, 68, 68, 0.2);
  }
}

// Cards get pointer cursor only when in selection mode
.uploadedCardWrapSelectable {
  cursor: pointer;
}

// Date range filter — sits above section2Header
.filterSortRow {
  display: flex;
  flex-direction: column;
  gap: 10px;
  padding: 10px 12px;
  background: var(--bg1);
  flex-shrink: 0;
  position: relative;

  &::after {
    content: '';
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    height: 1px;
    background: linear-gradient(90deg,
        rgba(139, 92, 246, 0.0) 0%,
        rgba(139, 92, 246, 0.6) 20%,
        rgba(56, 196, 186, 0.7) 50%,
        rgba(240, 160, 48, 0.6) 80%,
        rgba(240, 160, 48, 0.0) 100%);
    pointer-events: none;
  }
}

.sortHeader {
  margin-left: auto;
}

// Row 2 — date filter + status filter on the left, actions trigger on the right
.filterBarRow {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 10px;
  width: 100%;
}

// Groups the status filter and the actions trigger together, pushed to
// the right edge of the row, with a visible gap between the two.
.filterBarRight {
  display: flex;
  align-items: center;
  gap: 14px;
  margin-left: auto;
  flex-shrink: 0;
}

// Small caption shown inside the date/status/actions pills so their
// purpose is legible at a glance, not just implied by an icon.
.filterBarLabel {
  font-size: 9.5px;
  font-weight: 600;
  color: var(--t2);
  text-transform: uppercase;
  letter-spacing: 0.06em;
  white-space: nowrap;
  @include m.mono;
}

// ── Actions — collapsed into a single trigger button + popover ──
.actionsPopoverWrap {
  position: relative;
  flex-shrink: 0;
}

.actionsTrigger {
  display: flex;
  align-items: center;
  gap: 6px;
  height: 32px;
  padding: 0 10px;
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  background: var(--bg2);
  color: var(--t1);
  cursor: pointer;
  transition: all 0.12s;

  svg {
    width: 13px;
    height: 13px;
    flex-shrink: 0;
  }

  &:hover {
    background: var(--bg3);
    border-color: var(--bdr3);
  }
}

.actionsTriggerOpen {
  background: var(--blue-dim);
  border-color: var(--blue-bdr);
  color: var(--blue);
}

.actionsTriggerChevron {
  transition: transform 0.15s;

  .actionsTriggerOpen & {
    transform: rotate(180deg);
  }
}

.actionsPopover {
  position: absolute;
  top: calc(100% + 6px);
  right: 0;
  z-index: 20;
  display: flex;
  flex-direction: column;
  gap: 4px;
  padding: 8px;
  min-width: 168px;
  border: 1px solid var(--bdr2);
  border-radius: var(--rl);
  background: var(--bg1);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.18);
  animation: fadeSlide 0.14s ease;
}

.actionsPopover .actionTile {
  width: 100%;
  justify-content: flex-start;
}

.actionTile {
  display: flex;
  flex-direction: row;
  align-items: center;
  justify-content: center;
  width: 100%;
  min-width: 0;
  height: 28px;
  gap: 5px;
  padding: 0 9px;
  border-radius: var(--r);
  border: 1px solid transparent;
  background: var(--bg3);
  color: var(--t2);
  font-size: 12px;
  font-weight: 500;
  font-family: var(--font-ui);
  letter-spacing: 0.01em;
  cursor: pointer;
  transition: all 0.13s;
  user-select: none;

  svg {
    width: 12px;
    height: 12px;
    flex-shrink: 0;
  }

  &:active:not(:disabled) {
    transform: scale(0.97);
  }

  &:disabled {
    opacity: 0.3;
    cursor: not-allowed;
  }
}

.actionTileLabel {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  min-width: 0;
}

.actionTileDictionary {
  background: var(--violet-dim);
  color: var(--violet);
  border-color: var(--violet-dim);

  &:hover:not(:disabled) { background: var(--violet-dim); border-color: var(--violet); }
}

.actionTileTemplate {
  background: var(--teal-dim);
  color: var(--teal);
  border-color: var(--teal-bdr);

  &:hover:not(:disabled) { background: var(--teal-dim); border-color: var(--teal); }
}

.actionTileSearch {
  background: var(--amber-dim);
  color: var(--amber);
  border-color: var(--amber-bdr);

  &:hover:not(:disabled) { background: var(--amber-dim); border-color: var(--amber); }
}

.actionTileExport {
  background: var(--green-dim);
  color: var(--green);
  border-color: var(--green-bdr);

  &:hover:not(:disabled) { background: var(--green-dim); border-color: var(--green); }
}

.actionTileDelete {
  background: var(--red-dim);
  color: var(--red);
  border-color: var(--red-bdr);

  &:hover:not(:disabled) { background: var(--red-dim); border-color: var(--red); }
}

.actionTileActive {
  outline: 2px solid var(--blue-bdr);
  outline-offset: -1px;
}

// Single-line, icon-led date filter — same 32px height as the rest of the row
.dateFilter {
  display: flex;
  align-items: center;
  height: 32px;
  gap: 6px;
  flex-shrink: 0;
}

.dateIcon {
  width: 14px;
  height: 14px;
  color: var(--t2);
  flex-shrink: 0;
}

.dateField {
  display: flex;
  flex-direction: column;
  gap: 3px;
  width: 118px;
  flex-shrink: 0;
}

.dateLabel {
  font-size: 10px;
  font-weight: 600;
  color: var(--t2);
  text-transform: uppercase;
  letter-spacing: 0.1em;
  @include m.mono;
}

.dateInput {
  width: 115px;
  height: 32px;
  padding: 0 8px;
  background: var(--bg2);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 12.5px;
  outline: none;
  appearance: none;
  transition: border-color 0.12s;
  cursor: pointer;

  &:focus {
    border-color: var(--blue);
    box-shadow: 0 0 0 2px var(--blue-dim);
  }

  &::-webkit-calendar-picker-indicator {
    opacity: 0.7;
    cursor: pointer;
    filter: var(--date-icon-filter);
  }
}

.dateSep {
  font-size: 12px;
  color: var(--t2);
  flex-shrink: 0;
}

// ── Status filter ──────────────────────────────
.statusFilterWrap {
  display: flex;
  align-items: center;
  height: 32px;
  gap: 6px;
  flex-shrink: 0;
}

.statusFilterSelect {
  height: 32px;
  padding: 0 8px;
  background: var(--bg2);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 12.5px;
  outline: none;
  cursor: pointer;
  transition: border-color 0.12s;

  &:focus { border-color: var(--blue); }
  &:disabled { opacity: 0.5; cursor: default; }
}

.applyBtn {
  height: 32px;
  padding: 0 12px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t1);
  font-family: var(--font-ui);
  font-size: 12.5px;
  font-weight: 500;
  cursor: pointer;
  flex-shrink: 0;
  transition: all 0.12s;

  &:hover:not(:disabled) {
    background: var(--bg3);
    border-color: var(--bdr3);
  }

  &:disabled {
    opacity: 0.5;
    cursor: default;
  }
}

// Uploaded file cards
.uploadedBody {
  flex: 1;
  overflow-y: auto;
  padding: 10px;
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
  grid-auto-rows: 240px;
  align-content: start;
  gap: 12px;
  @include m.scrollbar;
}

// listState/errorState (loading, error, empty) span every column so they
// don't get squeezed into a single grid cell
.listState {
  grid-column: 1 / -1;
}

// ── Square file cards ─────────────────────────
.fcard {
  position: relative;
  height: 100%;
  min-width: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: flex-start;
  padding: 50px 16px 40px;
  border-radius: var(--rl);
  border: 1px solid var(--bdr3);
  background: var(--bg1);
  cursor: default;
  text-align: center;
  overflow: hidden;
  transition: all 0.12s;
  user-select: none;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.06);
}

.fcardStatic {
  cursor: default;
}

.fcard:not(.fcardStatic) {
  cursor: pointer;

  &:hover {
    background: var(--bg2);
    border-color: var(--bdr4, var(--bdr3));
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
    transform: translateY(-1px);
  }
}

.fcardActive,
.fcardActiveView {
  background: rgba(91, 164, 239, 0.06);
  border-color: var(--blue-bdr);
}

.fcardActiveDelete {
  background: var(--red-dim);
  border-color: var(--red-bdr);
}

.fcardActiveExport {
  background: var(--green-dim);
  border-color: var(--green-bdr);
}

.fcardCheck {
  position: absolute;
  top: 10px;
  left: 10px;
  z-index: 1;
}

// Dictionary / prompt-template association icons — mirrors the checkbox's
// corner, opposite side, and only renders when at least one is linked.
.fcardLinks {
  position: absolute;
  top: 10px;
  right: 10px;
  z-index: 1;
  display: flex;
  gap: 4px;
}

.fcardLinkIcon {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 22px;
  height: 22px;
  border-radius: 6px;

  svg { width: 12px; height: 12px; }
}

.fcardLinkIconDict {
  background: var(--amber-dim);
  color: var(--amber);
}

.fcardLinkIconTemplate {
  background: var(--violet-dim);
  color: var(--violet);
}

.fcardIcon {
  position: absolute;
  top: 10px;
  left: 10px;
  z-index: 1;
  width: 36px;
  height: 36px;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 10.5px;
  font-weight: 700;
  transition: top 0.12s;

  &.vtt { background: var(--blue-dim); color: var(--blue); }
  &.srt { background: var(--green-dim); color: var(--green); }
}

// When the selection checkbox is also occupying the top-left corner,
// nudge the ext badge down below it instead of overlapping.
.fcardIconShifted {
  top: 42px;
}

.fcardName {
  width: 100%;
  max-width: 100%;
  min-height: 38px;
  margin-bottom: 8px;
  font-size: 13.5px;
  font-weight: 500;
  color: var(--t0);
  line-height: 1.35;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
  word-break: break-word;
  flex-shrink: 0;
}

.fcardMeta {
  font-size: 10.5px;
  color: var(--t2);
  flex-shrink: 0;
  margin-bottom: 14px;
  @include m.mono;
}

// Compact toolbar of "which content prompts are customized" buttons
// (Summary, Keywords, Questions, short Answer, True/false). Bordered as a
// group + labeled, so it reads as an interactive control rather than a
// row of static status dots.
.fcardPrompts {
  display: flex;
  flex-direction: column;
  gap: 4px;
  flex-shrink: 0;
  margin-top: auto;
  margin-bottom: 10px;
  cursor: default;
}

.fcardPromptsLabel {
  font-size: 9px;
  font-weight: 600;
  color: var(--t3, var(--t2));
  text-transform: uppercase;
  letter-spacing: 0.08em;
}

.fcardPromptsRow {
  display: flex;
  align-items: center;
  gap: 4px;
  padding: 4px;
  border: 1px solid var(--bdr2);
  border-radius: 8px;
  background: var(--bg2);
  width: fit-content;
}

.fcardPromptDot {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 25px;
  height: 25px;
  border-radius: 6px;
  border: 1px solid transparent;
  background: var(--bg1);
  color: var(--t2);
  cursor: pointer;
  transition: all 0.12s;

  svg { width: 13px; height: 13px; }

  &:hover {
    background: var(--bg3);
    border-color: var(--bdr3);
    color: var(--t0);
    transform: translateY(-1px);
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.12);
  }
}

.fcardPromptDotSet {
  background: var(--blue-dim);
  color: var(--blue);
  border-color: var(--blue-bdr);

  &:hover {
    background: var(--blue-dim);
    color: var(--blue);
    border-color: var(--blue);
    opacity: 0.9;
  }
}

.fcardBadgeDot {
  width: 5px;
  height: 5px;
  border-radius: 50%;
  background: currentColor;
  animation: fcardPulse 1.4s ease-in-out infinite;
}

@keyframes fcardPulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.35; }
}

.fcardBadge {
  position: absolute;
  bottom: 12px;
  left: 50%;
  transform: translateX(-50%);
  font-size: 10px;
  padding: 3px 8px;
  max-width: calc(100% - 16px);
  overflow: hidden;
  text-overflow: ellipsis;
}

// ── Uploaded card wrap — carries the static green gradient border ──
// ── Uploaded file card — mirrors HistoryPanel .hitm ──────────────
.hitm {
  position: relative;
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 9px;
  border-radius: var(--r);
  border: 1px solid transparent;
  transition: all 0.12s;
  margin-bottom: 3px;
  user-select: none;

  &:hover {
    background: var(--bg2);
    border-color: var(--bdr);
  }

  &.active {
    background: rgba(91, 164, 239, 0.06);
    border-color: var(--blue-bdr);

    &::before {
      content: '';
      position: absolute;
      left: 0;
      top: 6px;
      bottom: 6px;
      width: 3px;
      border-radius: 0 3px 3px 0;
      background: linear-gradient(180deg, var(--blue), #a78bfa);
    }
  }

  // Normal-mode file view highlight — distinct purple/amber accent
  &.activeView {
    background: rgba(167, 139, 250, 0.06);
    border-color: rgba(167, 139, 250, 0.3);

    &::before {
      content: '';
      position: absolute;
      left: 0;
      top: 6px;
      bottom: 6px;
      width: 3px;
      border-radius: 0 3px 3px 0;
      background: linear-gradient(180deg, #a78bfa, var(--amber));
    }
  }

  // Delete mode selected — red accent
  &.activeDelete {
    background: rgba(239, 68, 68, 0.05);
    border-color: rgba(239, 68, 68, 0.3);

    &::before {
      content: '';
      position: absolute;
      left: 0;
      top: 6px;
      bottom: 6px;
      width: 3px;
      border-radius: 0 3px 3px 0;
      background: linear-gradient(180deg, #ef4444, #f97316);
    }
  }

  // Export mode selected — green accent
  &.activeExport {
    background: rgba(78, 200, 122, 0.05);
    border-color: rgba(78, 200, 122, 0.28);

    &::before {
      content: '';
      position: absolute;
      left: 0;
      top: 6px;
      bottom: 6px;
      width: 3px;
      border-radius: 0 3px 3px 0;
      background: linear-gradient(180deg, #4ec87a, #38c4ba);
    }
  }
}

.hitmSelectable {
  cursor: pointer;
}

// ── Ext icon ──
.ficon {
  width: 26px;
  height: 26px;
  border-radius: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 10px;
  font-weight: 700;
  @include m.mono;
  flex-shrink: 0;

  &.vtt {
    background: var(--blue-dim);
    color: var(--blue);
  }

  &.srt {
    background: var(--green-dim);
    color: var(--green);
  }
}

// ── File info ──
.hi {
  flex: 1;
  min-width: 0;
}

.hn {
  font-size: 13px;
  font-weight: 500;
  color: var(--t0);
  @include m.truncate;
}

.hm {
  font-size: 12px;
  color: var(--t2);
  margin-top: 2px;
  @include m.mono;
}

// Empty / loading / error state
.listState {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 10px;
  padding: 32px 16px;
  font-size: 13px;
  color: var(--t2);
  @include m.mono;
  text-align: center;

  svg {
    width: 18px;
    height: 18px;
    opacity: 0.5;
  }
}

.errorState {
  color: var(--red);

  svg {
    opacity: 0.7;
  }
}

.spinner {
  width: 18px;
  height: 18px;
  border: 2px solid var(--bdr2);
  border-top-color: var(--blue);
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

// ── Search icon button ──────────────────────────
.searchIconBtn {
  width: 28px;
  height: 28px;
  border-radius: 7px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  flex-shrink: 0;
  transition: all 0.15s;

  svg {
    width: 13px;
    height: 13px;
  }

  &:hover {
    background: var(--bg3);
    border-color: var(--bdr3);
    color: var(--t0);
  }
}

.searchIconBtnActive {
  border-color: var(--blue-bdr);
  background: var(--blue-dim);
  color: var(--blue);

  &:hover {
    background: var(--blue-dim);
    border-color: var(--blue);
    color: var(--blue);
  }
}

// ── Search bar (slides in below sortHeader) ──────
.searchBar {
  display: grid;
  grid-template-rows: 0fr;
  opacity: 0;
  transition:
    grid-template-rows 0.22s cubic-bezier(0.4, 0, 0.2, 1),
    opacity 0.18s ease;
  overflow: hidden;
  padding: 0 10px;
}

.searchBarOpen {
  grid-template-rows: 1fr;
  opacity: 1;
  padding: 6px 10px 4px;
}

// Flex row: input box + close button — right-aligned, capped width so
// it doesn't stretch across the whole panel.
.searchBarRow {
  min-height: 0;
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 6px;
}

.searchInner {
  flex: 0 1 320px;
  min-width: 0;
  min-height: 0;
  display: flex;
  align-items: center;
  gap: 6px;
  background: var(--bg2);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  padding: 0 8px;
  transition: border-color 0.15s, box-shadow 0.15s;

  &:focus-within {
    border-color: var(--blue);
    box-shadow: 0 0 0 2px var(--blue-dim);
  }
}

.searchBarIcon {
  width: 12px;
  height: 12px;
  flex-shrink: 0;
  color: var(--t2);
  opacity: 0.6;
}

.searchInput {
  flex: 1;
  min-width: 0;
  padding: 6px 0;
  background: transparent;
  border: none;
  outline: none;
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 13px;

  &::placeholder {
    color: var(--t2);
    opacity: 0.55;
  }
}

.searchCloseBtn {
  width: 20px;
  height: 20px;
  border-radius: 5px;
  border: 1px solid rgba(239, 68, 68, 0.3);
  background: rgba(239, 68, 68, 0.08);
  color: #ef4444;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  flex-shrink: 0;
  transition: all 0.12s;

  svg {
    width: 8px;
    height: 8px;
  }

  &:hover {
    background: rgba(239, 68, 68, 0.18);
    border-color: rgba(239, 68, 68, 0.6);
    box-shadow: 0 0 6px rgba(239, 68, 68, 0.2);
  }
}

.searchCount {
  font-size: 11px;
  color: var(--t2);
  font-family: var(--font-mono);
  white-space: nowrap;
  flex-shrink: 0;
}


.badge {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-size: 12px;
  padding: 2px 8px;
  border-radius: 99px;
  font-weight: 500;
  border: 1px solid transparent;
  white-space: nowrap;
  @include m.mono;
}

.bReady {
  background: var(--green-dim);
  color: var(--green);
  border-color: var(--green-bdr);
}

// Inferenced — green (file has completed inference)
.bInferred {
  background: var(--green-dim);
  color: var(--green);
  border-color: var(--green-bdr);
}

// Not Inferenced — amber (file uploaded but not yet processed)
.bNotInferred {
  background: rgba(240, 160, 48, 0.1);
  color: var(--amber);
  border-color: rgba(240, 160, 48, 0.3);
}

// Running — blue, with the pulsing dot from fcardBadgeDot
.bRunning {
  background: var(--blue-dim);
  color: var(--blue);
  border-color: var(--blue-bdr);
}

// Queued — amber, waiting its turn in the batch
.bQueued {
  background: rgba(240, 160, 48, 0.1);
  color: var(--amber);
  border-color: rgba(240, 160, 48, 0.3);
}

// Error — red, inference failed for this file
.bError {
  background: var(--red-dim);
  color: var(--red);
  border-color: var(--red-bdr);
}

// Waiting — neutral gray, distinct from "not inferenced" (amber)
.bWaiting {
  background: var(--bg2);
  color: var(--t2);
  border-color: var(--bdr2);
}

.bSelected {
  background: var(--blue-dim);
  color: var(--blue);
  border-color: var(--blue-bdr);
  font-weight: 600;
}

.bDelete {
  background: rgba(239, 68, 68, 0.1);
  color: #ef4444;
  border-color: rgba(239, 68, 68, 0.3);
  font-weight: 600;
}

.bExport {
  background: var(--green-dim);
  color: var(--green);
  border-color: var(--green-bdr);
  font-weight: 600;
  gap: 4px;
}

.bInfo {
  background: var(--blue-dim);
  color: var(--blue);
  border-color: var(--blue-bdr);
}

// ── Buttons ───────────────────────────────────
.btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 5px;
  padding: 6px 13px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t1);
  font-family: var(--font-ui);
  font-size: 13px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.12s;
  white-space: nowrap;
  user-select: none;

  svg {
    width: 11px;
    height: 11px;
  }

  &:hover {
    background: var(--bg3);
    color: var(--t0);
    border-color: var(--bdr3);
  }

  &:disabled {
    opacity: 0.45;
    cursor: default;
  }
}

.btnP {
  background: var(--blue);
  color: #fff;
  border-color: var(--blue);
  font-weight: 600;

  &:hover {
    background: #a78bfa;
    border-color: #a78bfa;
    color: #fff;
  }
}

.btnSm {
  padding: 4px 10px;
  font-size: 13px;
}

.btnFull {
  flex: 1;
}

.btnDanger {
  color: var(--red);
  border-color: var(--red-bdr);

  &:hover {
    background: var(--red-dim);
    border-color: var(--red);
  }

  &:disabled {
    color: var(--t2);
    border-color: var(--bdr2);
    background: transparent;
    opacity: 0.4;
    cursor: not-allowed;
    pointer-events: none;

    &:hover {
      background: transparent;
      border-color: var(--bdr2);
    }
  }
}

// ── Animations ───────────────────────────────
@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

@keyframes fadeSlide {
  from {
    opacity: 0;
    transform: translateY(4px);
  }

  to {
    opacity: 1;
    transform: translateY(0);
  }
}

// ── Checkbox component ──────────────────────────
.cb {
  width: 14px;
  height: 14px;
  border: 1.5px solid var(--cb-bdr);
  border-radius: 4px;
  flex-shrink: 0;
  cursor: pointer;
  transition: all 0.12s;
  position: relative;
  outline: none;

  &:hover:not(.cbDisabled) {
    border-color: var(--blue);
  }

  &:focus-visible {
    box-shadow: 0 0 0 2px var(--blue-dim);
  }

  &.cbChecked {
    background: var(--blue);
    border-color: var(--blue);

    &::after {
      content: '';
      position: absolute;
      left: 2px;
      top: 5px;
      width: 7px;
      height: 4px;
      border-left: 1.5px solid #fff;
      border-bottom: 1.5px solid #fff;
      transform: rotate(-45deg) translate(0, -1px);
    }
  }

  &.cbIndet {
    background: var(--blue);
    border-color: var(--blue);

    &::after {
      content: '';
      position: absolute;
      left: 2px;
      top: 5px;
      width: 8px;
      height: 1.5px;
      background: #fff;
    }
  }

  &.cbDisabled {
    opacity: 0.35;
    cursor: not-allowed;
    pointer-events: none;
  }
}

// Grouped action buttons — used inside mode-bar (title row)
.headerActions {
  display: flex;
  align-items: center;
  gap: 4px;
  margin-left: auto;
  flex-shrink: 0;
}

// ── Mode action row: search LEFT, confirm+cancel RIGHT ────────
.modeActionsRow {
  display: flex;
  align-items: center;
  box-sizing: border-box;
  width: 100%;
  padding: 0 10px 8px;
  min-width: 0;
}

.modeActionsRight {
  display: flex;
  align-items: center;
  gap: 4px;
  margin-left: auto;
  flex-shrink: 0;
}

.modeActionsLeft {
  display: flex;
  align-items: center;
  gap: 4px;
  min-width: 0;
  flex-wrap: wrap;
}

// Base tile — icon + label side by side
.modeTile {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 4px 9px;
  border-radius: 6px;
  border: 1px solid var(--bdr);
  background: var(--bg3);
  color: var(--t2);
  font-size: 11px;
  font-weight: 500;
  font-family: var(--font-ui);
  cursor: pointer;
  transition: all 0.13s;
  user-select: none;
  white-space: nowrap;

  svg {
    width: 12px;
    height: 12px;
    flex-shrink: 0;
  }

  &:active:not(:disabled) {
    transform: scale(0.97);
  }

  &:disabled {
    opacity: 0.3;
    cursor: not-allowed;
  }
}

// Search tile — amber tint (left side)
.modeTileSearch {
  color: var(--amber);
  border-color: rgba(240, 160, 48, 0.25);
  background: rgba(240, 160, 48, 0.08);

  &:hover {
    background: rgba(240, 160, 48, 0.16);
    border-color: rgba(240, 160, 48, 0.45);
  }
}

.modeTileSearchActive {
  background: rgba(240, 160, 48, 0.15);
  border-color: rgba(240, 160, 48, 0.45);
}

// Confirm export — green
.modeTileConfirmExport {
  color: var(--green);
  border-color: var(--green-bdr);
  background: var(--green-dim);

  &:hover:not(:disabled) {
    background: var(--green);
    border-color: var(--green);
    color: #fff;
  }
}

// Confirm delete — red
.modeTileConfirmDelete {
  color: #ef4444;
  border-color: rgba(239, 68, 68, 0.25);
  background: rgba(239, 68, 68, 0.08);

  &:hover:not(:disabled) {
    background: rgba(239, 68, 68, 0.18);
    border-color: rgba(239, 68, 68, 0.5);
  }
}

// Cancel
.modeTileCancel {
  color: var(--t2);
  border-color: var(--bdr2);
  background: transparent;

  &:hover {
    background: var(--bg2);
    border-color: var(--bdr3);
    color: var(--t0);
  }
}

// Disabled state
.modeTileDisabled {
  opacity: 0.35 !important;
  cursor: not-allowed !important;
}

// "Export all files" trigger — mirrors modeTileDeleteAll but green/non-destructive
.modeTileExportAll {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 4px 9px;
  border-radius: 6px;
  border: 1px solid var(--green-bdr);
  background: var(--green-dim);
  color: var(--green);
  font-size: 11px;
  font-weight: 600;
  font-family: var(--font-ui);
  cursor: pointer;
  transition: all 0.13s;
  user-select: none;
  white-space: nowrap;

  svg {
    width: 12px;
    height: 12px;
    flex-shrink: 0;
  }

  &:hover:not(:disabled) {
    background: var(--green);
    border-color: var(--green);
    color: #fff;
  }

  &:disabled {
    opacity: 0.35;
    cursor: not-allowed;
  }
}

.modeActionsError {
  font-size: 11px;
  color: #ef4444;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 160px;
}

// "Delete all files" trigger — deliberately louder/more solid than the
// per-row delete tile since it's an account-wide destructive action.
.modeTileDeleteAll {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 4px 9px;
  border-radius: 6px;
  border: 1px solid rgba(239, 68, 68, 0.5);
  background: rgba(239, 68, 68, 0.14);
  color: #ef4444;
  font-size: 11px;
  font-weight: 600;
  font-family: var(--font-ui);
  cursor: pointer;
  transition: all 0.13s;
  user-select: none;
  white-space: nowrap;

  svg {
    width: 12px;
    height: 12px;
    flex-shrink: 0;
  }

  &:hover:not(:disabled) {
    background: #ef4444;
    border-color: #ef4444;
    color: #fff;
  }

  &:disabled {
    opacity: 0.3;
    cursor: not-allowed;
  }
}

// ── Delete-ALL confirmation (danger zone) ───────────────────────────
.dangerOverlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.55);
  backdrop-filter: blur(2px);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 200;
  padding: 16px;
}

.dangerModal {
  width: 100%;
  max-width: 360px;
  background: var(--bg1);
  border: 1px solid rgba(239, 68, 68, 0.4);
  border-radius: 10px;
  padding: 20px;
  box-shadow: 0 12px 40px rgba(0, 0, 0, 0.4);
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  gap: 4px;
}

.dangerIcon {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background: rgba(239, 68, 68, 0.14);
  color: #ef4444;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 6px;

  svg {
    width: 22px;
    height: 22px;
  }
}

.dangerTitle {
  font-size: 15px;
  font-weight: 700;
  color: var(--t0);
  font-family: var(--font-ui);
}

.dangerBody {
  font-size: 13px;
  color: var(--t2);
  line-height: 1.5;
  margin-bottom: 8px;
}

.dangerLabel {
  align-self: flex-start;
  font-size: 12px;
  color: var(--t1);
  margin-top: 4px;
}

.dangerInput {
  width: 100%;
  padding: 7px 10px;
  background: var(--bg0);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 13px;
  outline: none;
  margin-top: 4px;
  margin-bottom: 6px;
  text-align: center;
  transition: border-color 0.12s;

  &:focus {
    border-color: #ef4444;
    box-shadow: 0 0 0 2px rgba(239, 68, 68, 0.15);
  }

  &:disabled {
    opacity: 0.6;
  }
}

.dangerError {
  font-size: 12px;
  color: #ef4444;
  margin-bottom: 6px;
}

.dangerActions {
  display: flex;
  gap: 8px;
  width: 100%;
  margin-top: 6px;
}

// .uploadedCardSel replaced by .hitm.active

// Uploaded card disabled (batch running)
.uploadedCardDisabled {
  cursor: default !important;
  opacity: 0.65;
  pointer-events: none;
}

// ── Sort header ─────────────────────────────────────
.sortHeader {
  display: flex;
  align-items: center;
  height: 32px;
  background: var(--bg2);
  border: 1px solid var(--bdr);
  border-radius: var(--r);
  padding: 0 6px 0 10px;
  gap: 6px;
  flex-shrink: 0;
  overflow: hidden;
}

.sortHeaderLabel {
  display: flex;
  align-items: center;
  gap: 5px;
  font-size: 10px;
  font-weight: 600;
  color: var(--t2);
  font-family: var(--font-mono);
  text-transform: uppercase;
  letter-spacing: 0.06em;
  white-space: nowrap;
  flex-shrink: 0;

  svg {
    width: 11px;
    height: 11px;
    opacity: 0.6;
  }
}

.sortCols {
  display: flex;
  align-items: center;
  gap: 2px;
}

.sortCol {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 4px;
  height: 24px;
  padding: 0 8px;
  background: transparent;
  border: none;
  border-radius: 5px;
  color: var(--t2);
  font-size: 12px;
  font-family: var(--font-mono);
  cursor: pointer;
  transition: all 0.12s;
  white-space: nowrap;

  svg {
    width: 8px;
    height: 10px;
    flex-shrink: 0;
    transition: transform 0.2s;
  }

  &:hover {
    background: var(--bg3);
    color: var(--t1);
  }
}

.sortColActive {
  background: var(--blue-dim);
  color: var(--blue);
  font-weight: 600;

  &:hover {
    background: var(--blue-dim);
  }
}

.sortInactive {
  opacity: 0.25;
}

.sortAsc {
  transform: rotate(180deg);
}

// arrow up
.sortDesc {
  transform: rotate(0deg);
}

// arrow down (default path direction)
// ── Small screen overrides (height < 1000px) ─────────────────
@media (max-height: 999px) {

  // Step-1 bar — tighter
  .step1Bar {
    padding: 7px 14px;
  }

  // Dropzone — much more compact
  .dropzone {
    padding: 10px 16px;
    margin: 6px 10px;
  }

  .dzIc {
    width: 28px;
    height: 28px;
    margin-bottom: 6px;

    svg {
      width: 14px;
      height: 14px;
    }
  }

  .dzTitle {
    font-size: 12px;
    margin-bottom: 2px;
  }

  .dzSub {
    font-size: 11px;
  }

  .dzActions {
    margin-top: 7px;
    gap: 6px;
  }

  // Section2 title row
  .section2TitleRow {
    padding: 7px 14px 6px;
  }

  // Mode action row
  .modeActionsRow {
    padding: 0 8px 6px;
  }

  // Date filter — tighter
  .dateFilter {
    padding: 5px 10px 5px;
    gap: 5px;
  }

  .dateInput {
    padding: 3px 6px;
    font-size: 12px;
  }

  // Sort header — tighter
  .sortHeader {
    margin-bottom: 4px;
  }

  .sortCol {
    padding: 4px 4px;
    font-size: 11px;
  }

  .sortHeaderLabel {
    padding: 4px 0;
    padding-right: 6px;
    font-size: 9px;
  }

  // File list — tighter items, ensure it can grow
  .uploadedBody {
    padding: 4px 8px;
    gap: 2px;
  }

  .hitm {
    padding: 5px 8px;
    margin-bottom: 1px;
  }

  .ficon {
    width: 22px;
    height: 22px;
    font-size: 9px;
  }

  .hn {
    font-size: 12px;
  }

  .hm {
    font-size: 11px;
    margin-top: 1px;
  }

  .badge {
    font-size: 10px;
    padding: 1px 6px;
  }
}

// ── Pagination footer (bottom of the uploaded-files list) ──────────
.paginationBar {
  display: flex;
  flex-direction: column;
  gap: 6px;
  padding: 8px 10px;
  border-top: 1px solid var(--bdr2);
  background: var(--bg1);
  flex-shrink: 0;
}

.paginationTopRow {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  flex-wrap: nowrap;
  min-width: 0;
}

.pageSizeGroup {
  display: flex;
  align-items: center;
  gap: 6px;
  flex-shrink: 0;
}

.pageSizeLabel {
  font-size: 11px;
  color: var(--t2);
  white-space: nowrap;
}

.pageSizeSelect {
  padding: 3px 6px;
  background: var(--bg0);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 12px;
  outline: none;
  cursor: pointer;
  transition: border-color 0.12s;

  &:focus {
    border-color: var(--blue);
    box-shadow: 0 0 0 2px var(--blue-dim);
  }
}

// Page-number row sits to the right of the page-size selector. It never
// wraps to its own line — if there isn't room for every number button,
// this row scrolls horizontally within itself instead.
.pageNav {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 2px;
  flex-wrap: nowrap;
  flex-shrink: 1;
  min-width: 0;
  overflow-x: auto;
  scrollbar-width: none;
  -ms-overflow-style: none;

  &::-webkit-scrollbar {
    display: none;
  }
}

.pageNavBtn,
.pageNumBtn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 24px;
  height: 24px;
  padding: 0 6px;
  border-radius: var(--r);
  border: 1px solid transparent;
  background: transparent;
  color: var(--t1);
  font-family: var(--font-ui);
  font-size: 12px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.12s;
  user-select: none;
  flex-shrink: 0;

  svg {
    width: 11px;
    height: 11px;
  }

  &:hover:not(:disabled) {
    background: var(--bg3);
    color: var(--t0);
    border-color: var(--bdr3);
  }

  &:disabled {
    opacity: 0.35;
    cursor: default;
  }
}

.pageNumBtnActive {
  background: var(--blue-dim);
  border-color: var(--blue-bdr);
  color: var(--blue);
  font-weight: 700;

  &:hover:not(:disabled) {
    background: var(--blue-dim);
    color: var(--blue);
  }
}

.pageEllipsis {
  color: var(--t2);
  font-size: 12px;
  padding: 0 2px;
  user-select: none;
  flex-shrink: 0;
}

.pageInfo {
  font-size: 11px;
  color: var(--t2);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  text-align: right;
  @include m.mono;
}

@media (min-width: 1920px) {
  .section1Title {
    font-size: 14px;
  }

  .section2Title {
    font-size: 14px;
  }

  .uploadHint {
    font-size: 13px;
  }

  .uploadZoneText {
    font-size: 14px;
  }

  .sortHeaderLabel {
    font-size: 12px;
  }

  .sortBtn {
    font-size: 13px;
  }

  .fileName {
    font-size: 14px;
  }

  .fileMeta {
    font-size: 12px;
  }

  .fileDate {
    font-size: 12px;
  }

  .badge {
    font-size: 13px;
  }

  .btn {
    font-size: 13px;
  }

  .listState {
    font-size: 14px;
  }

  .searchInput {
    font-size: 14px;
  }

  .searchCount {
    font-size: 12px;
  }

  .dateLabel {
    font-size: 12px;
  }

  .dateInput {
    font-size: 13px;
  }

  .emptyState {
    font-size: 14px;
  }
}

// ── Per-file prompt viewer popup ─────────────────
.promptViewerModal {
  width: 100%;
  max-width: 480px;
  max-height: 70vh;
  display: flex;
  flex-direction: column;
  background: var(--bg1);
  border: 1px solid var(--bdr2);
  border-radius: var(--rl);
  box-shadow: var(--shadow);
  overflow: hidden;
}

.promptViewerHead {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 10px;
  padding: 14px 16px;
  border-bottom: 1px solid var(--bdr);
  flex-shrink: 0;
}

.promptViewerTitle {
  font-size: 13.5px;
  font-weight: 600;
  color: var(--t0);
}

.promptViewerFile {
  margin-top: 2px;
  font-size: 12px;
  color: var(--t2);
  @include m.truncate;
}

.promptViewerClose {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 26px;
  height: 26px;
  flex-shrink: 0;
  border-radius: 6px;
  border: none;
  background: transparent;
  color: var(--t2);
  cursor: pointer;

  svg { width: 13px; height: 13px; }

  &:hover {
    background: var(--bg3);
    color: var(--t0);
  }
}

.promptViewerBody {
  padding: 16px;
  overflow-y: auto;
  font-size: 13px;
  line-height: 1.6;
  color: var(--t1);
  white-space: pre-wrap;
  word-break: break-word;
  @include m.scrollbar;
}

.promptViewerEmpty {
  color: var(--t2);
  font-style: italic;
}
