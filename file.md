// ═══════════════════════════════════════════════
// pages/UploadInfer/PromptTemplateAssociationModal.tsx
// Content Analytics · Bulk file ↔ prompt-template association
// ═══════════════════════════════════════════════
import React, { useCallback, useEffect, useMemo, useRef, useState } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import { updateFilePrompts, patchFilePromptTemplate, type ServerFile } from '../../store/uploadSlice';
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
    // dateFrom/dateTo are the app's shared date-range filter (used below for
    // the "Associate ALL in range" bar). The file *list* itself, though, is
    // fetched independently here rather than read from state.upload.serverFiles
    // — that list is only populated while the Upload & Manage tab is the
    // active one (FilePanel clears it to [] whenever it isn't), so relying
    // on it left this modal empty whenever it was opened from anywhere else,
    // like the Inference tab's "Map prompt template" button.
    const { dateFrom, dateTo } = useAppSelector(s => s.upload);

    const [serverFiles, setServerFiles] = useState<ServerFile[]>([]);
    const [filesLoading, setFilesLoading] = useState(false);
    const [filesError, setFilesError] = useState<string | null>(null);

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

        (async () => {
            setFilesLoading(true); setFilesError(null);
            try {
                const res = await api.post('/files/by-date/', {
                    start_date: dateFrom,
                    end_date: dateTo,
                    page: 1,
                    page_size: 500,
                    sort_by: 'original_name',
                    sort_order: 'asc',
                });
                const d = (res.data as any)?.data ?? {};
                const list: ServerFile[] = d.data ?? [];
                if (cancelled) return;
                setServerFiles(list);
                const init: Record<number, number> = {};
                list.forEach(f => { init[f.id] = f.prompt_template_id ?? NO_TEMPLATE; });
                setSelection(init);
                initialSelectionRef.current = init;
            } catch {
                if (!cancelled) setFilesError(t('uploadInfer.templateModal.loadFail'));
            } finally { if (!cancelled) setFilesLoading(false); }
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
                        disabled={saving || templatesLoading || filesLoading || serverFiles.length === 0}>
                        <option value={NO_TEMPLATE}>{t('uploadInfer.templateModal.noneOption')}</option>
                        {templates.map(tmpl => <option key={tmpl.id} value={tmpl.id}>{tmpl.name}</option>)}
                    </select>
                    <button className={styles.bulkApplyBtn} onClick={handleApplyToAll} disabled={saving || filesLoading || serverFiles.length === 0}>
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

                        {(templatesLoading || filesLoading) && (
                            <div className={styles.listState}>
                                <div className={styles.spinner} /><span>{t('uploadInfer.templateModal.loadingTemplates')}</span>
                            </div>
                        )}
                        {templatesError && <div className={`${styles.listState} ${styles.errorState}`}>{templatesError}</div>}
                        {filesError && <div className={`${styles.listState} ${styles.errorState}`}>{filesError}</div>}
                        {!templatesLoading && !filesLoading && !filesError && serverFiles.length === 0 && (
                            <div className={styles.listState}>{t('uploadInfer.templateModal.noFiles')}</div>
                        )}

                        {!templatesLoading && !filesLoading && !templatesError && !filesError && serverFiles.map(f => {
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
