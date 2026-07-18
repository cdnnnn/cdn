// ═══════════════════════════════════════════════
// pages/UploadInfer/DictionaryAssociationModal.tsx
// Content Analytics · Bulk file ↔ dictionary association
// ═══════════════════════════════════════════════
import React, { useCallback, useEffect, useMemo, useRef, useState } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import { patchFileDictionaries } from '../../store/uploadSlice';
import { addToast } from '../../store/toastSlice';
import api from '../../services/api';
import styles from './DictionaryAssociationModal.module.scss';

interface DictionaryListItem { dictionary_id: number; dictionary_name: string; }
interface DictionaryTerm { wrong_term: string; correct_term: string; }
interface DictionaryDetail {
    dictionary_id: number; dictionary_name: string;
    dictionary_terms: DictionaryTerm[]; created_at: string; updated_at: string;
}
interface Props { open: boolean; onClose: () => void; }

const NO_DICT = -1;

const DictionaryAssociationModal: React.FC<Props> = ({ open, onClose }) => {
    const dispatch = useAppDispatch();
    const { t } = useTranslation();
    const serverFiles = useAppSelector(s => s.upload.serverFiles);

    const [dicts, setDicts] = useState<DictionaryListItem[]>([]);
    const [dictsLoading, setDictsLoading] = useState(false);
    const [dictsError, setDictsError] = useState<string | null>(null);
    const [selection, setSelection] = useState<Record<number, number>>({});
    const initialSelectionRef = useRef<Record<number, number>>({});
    const [bulkValue, setBulkValue] = useState<number>(NO_DICT);
    const [saving, setSaving] = useState(false);
    const [detailDictId, setDetailDictId] = useState<number | null>(null);
    const [detailData, setDetailData] = useState<DictionaryDetail | null>(null);
    const [detailLoading, setDetailLoading] = useState(false);
    const [detailError, setDetailError] = useState<string | null>(null);

    useEffect(() => {
        if (!open) return;
        const init: Record<number, number> = {};
        serverFiles.forEach(f => { init[f.id] = f.dictionary_id ?? NO_DICT; });
        setSelection(init);
        initialSelectionRef.current = init;
        setBulkValue(NO_DICT);
        setDetailDictId(null); setDetailData(null); setDetailError(null);

        let cancelled = false;
        (async () => {
            setDictsLoading(true); setDictsError(null);
            try {
                const res = await api.get('/dictionary/list');
                const list: DictionaryListItem[] = (res.data as any)?.data ?? (res.data as any)?.result ?? [];
                if (!cancelled) setDicts(Array.isArray(list) ? list : []);
            } catch {
                if (!cancelled) setDictsError(t('uploadInfer.dictModal.loadFail'));
            } finally { if (!cancelled) setDictsLoading(false); }
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

    const handleRowChange = useCallback((fileId: number, dictId: number) => {
        setSelection(prev => ({ ...prev, [fileId]: dictId }));
    }, []);

    const handleApplyToAll = useCallback(() => {
        setSelection(prev => {
            const next = { ...prev };
            Object.keys(next).forEach(k => { next[Number(k)] = bulkValue; });
            return next;
        });
    }, [bulkValue]);

    const handleViewDetails = useCallback(async (dictId: number) => {
        if (detailDictId === dictId) { setDetailDictId(null); setDetailData(null); setDetailError(null); return; }
        setDetailDictId(dictId); setDetailData(null); setDetailError(null); setDetailLoading(true);
        try {
            const res = await api.get(`/dictionary/${dictId}`);
            const d: DictionaryDetail | undefined = (res.data as any)?.data;
            if (d) setDetailData(d);
            else setDetailError(t('uploadInfer.dictModal.noDetailsReturned'));
        } catch { setDetailError(t('uploadInfer.dictModal.detailLoadFail')); }
        finally { setDetailLoading(false); }
    }, [detailDictId, t]);

    const handleSave = useCallback(async () => {
        if (!isDirty || saving) return;
        setSaving(true);
        const associateGroups: Record<number, number[]> = {};
        const disassociateIds: number[] = [];
        for (const id of dirtyIds) {
            const v = selection[id];
            if (v === NO_DICT) disassociateIds.push(id);
            else { if (!associateGroups[v]) associateGroups[v] = []; associateGroups[v].push(id); }
        }
        try {
            const calls: Promise<unknown>[] = [];
            for (const [dictIdStr, fileIds] of Object.entries(associateGroups)) {
                calls.push(api.post('/dictionary/associate', { file_ids: fileIds, dictionary_id: Number(dictIdStr) }));
            }
            if (disassociateIds.length > 0) {
                calls.push(api.post('/dictionary/disassociate', { all: false, file_ids: disassociateIds }));
            }
            await Promise.all(calls);
            const patch: Record<number, number | null> = {};
            for (const [dictIdStr, fileIds] of Object.entries(associateGroups)) { fileIds.forEach(id => { patch[id] = Number(dictIdStr); }); }
            disassociateIds.forEach(id => { patch[id] = null; });
            dispatch(patchFileDictionaries(patch));
            dispatch(addToast(t('uploadInfer.dictModal.saveSuccess'), 'success'));
            onClose();
        } catch {
            dispatch(addToast(t('uploadInfer.dictModal.saveFail'), 'error'));
        } finally { setSaving(false); }
    }, [isDirty, saving, dirtyIds, selection, dispatch, onClose, t]);

    const handleCloseClick = useCallback(() => { if (saving) return; onClose(); }, [saving, onClose]);
    const handleBackdropClick = useCallback((e: React.MouseEvent) => { e.stopPropagation(); }, []);
    const handleDialogClick = useCallback((e: React.MouseEvent) => { e.stopPropagation(); }, []);

    if (!open) return null;

    const dictName = (id: number) => {
        if (id === NO_DICT) return t('uploadInfer.dictModal.noneRow');
        const d = dicts.find(x => x.dictionary_id === id);
        return d ? d.dictionary_name : `Dictionary #${id}`;
    };

    return (
        <div className={styles.backdrop} onMouseDown={handleBackdropClick} role="presentation">
            <div className={styles.dialog} onMouseDown={handleDialogClick} role="dialog" aria-modal="true" aria-labelledby="dict-modal-title">

                {/* Header */}
                <div className={styles.header}>
                    <div className={styles.headerText}>
                        <div className={styles.title} id="dict-modal-title">{t('uploadInfer.dictModal.title')}</div>
                        <div className={styles.subtitle}>{t('uploadInfer.dictModal.subtitle')}</div>
                    </div>
                    <button className={styles.closeBtn} onClick={handleCloseClick} disabled={saving}
                        title={saving ? t('uploadInfer.dictModal.closeSaving') : t('uploadInfer.dictModal.close')}
                        aria-label={t('uploadInfer.dictModal.close')}>
                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                            <path d="M4 4l8 8M12 4l-8 8" />
                        </svg>
                    </button>
                </div>

                {/* Bulk bar */}
                <div className={styles.bulkBar}>
                    <span className={styles.bulkLabel}>{t('uploadInfer.dictModal.applyToAll')}</span>
                    <select className={styles.bulkSelect} value={bulkValue}
                        onChange={e => setBulkValue(Number(e.target.value))}
                        disabled={saving || dictsLoading || serverFiles.length === 0}>
                        <option value={NO_DICT}>{t('uploadInfer.dictModal.noneOption')}</option>
                        {dicts.map(d => <option key={d.dictionary_id} value={d.dictionary_id}>{d.dictionary_name}</option>)}
                    </select>
                    <button className={styles.bulkApplyBtn} onClick={handleApplyToAll} disabled={saving || serverFiles.length === 0}>
                        {t('uploadInfer.dictModal.applyBtn')}
                    </button>
                    <span className={styles.bulkHint}>
                        {isDirty
                            ? t('uploadInfer.dictModal.changes', { count: dirtyIds.length })
                            : t('uploadInfer.dictModal.noChanges')}
                    </span>
                </div>

                {/* Body */}
                <div className={styles.body}>
                    <div className={`${styles.listPane} ${detailDictId !== null ? styles.listPaneNarrow : ''}`}>
                        <div className={styles.listHead}>
                            <div className={styles.colFile}>{t('uploadInfer.dictModal.colFile')}</div>
                            <div className={styles.colDict}>{t('uploadInfer.dictModal.colDict')}</div>
                            <div className={styles.colActions} />
                        </div>

                        {dictsLoading && (
                            <div className={styles.listState}>
                                <div className={styles.spinner} /><span>{t('uploadInfer.dictModal.loadingDicts')}</span>
                            </div>
                        )}
                        {dictsError && <div className={`${styles.listState} ${styles.errorState}`}>{dictsError}</div>}
                        {!dictsLoading && serverFiles.length === 0 && (
                            <div className={styles.listState}>{t('uploadInfer.dictModal.noFiles')}</div>
                        )}

                        {!dictsLoading && !dictsError && serverFiles.map(f => {
                            const current = selection[f.id] ?? NO_DICT;
                            const initial = initialSelectionRef.current[f.id] ?? NO_DICT;
                            const rowDirty = current !== initial;
                            const hasDict = current !== NO_DICT;
                            return (
                                <div key={f.id} className={`${styles.row} ${rowDirty ? styles.rowDirty : ''}`}>
                                    <div className={styles.colFile} title={f.original_name}>
                                        <span className={styles.fileName}>{f.original_name}</span>
                                        {initial !== NO_DICT && (
                                            <span className={styles.currentBadge} title={t('uploadInfer.dictModal.associated')}>
                                                {t('uploadInfer.dictModal.associated')}
                                            </span>
                                        )}
                                    </div>
                                    <div className={styles.colDict}>
                                        <select className={styles.rowSelect} value={current}
                                            onChange={e => handleRowChange(f.id, Number(e.target.value))} disabled={saving}>
                                            <option value={NO_DICT}>{t('uploadInfer.dictModal.noneRow')}</option>
                                            {dicts.map(d => <option key={d.dictionary_id} value={d.dictionary_id}>{d.dictionary_name}</option>)}
                                        </select>
                                    </div>
                                    <div className={styles.colActions}>
                                        {hasDict && (
                                            <button className={styles.viewBtn} onClick={() => handleViewDetails(current)}
                                                disabled={saving} title={t('uploadInfer.dictModal.viewDetails')}>
                                                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                                    <path d="M1.5 8s2.5-4.5 6.5-4.5S14.5 8 14.5 8s-2.5 4.5-6.5 4.5S1.5 8 1.5 8z" />
                                                    <circle cx="8" cy="8" r="2" />
                                                </svg>
                                            </button>
                                        )}
                                        {initial !== NO_DICT && current !== NO_DICT && (
                                            <button className={styles.disBtn} onClick={() => handleRowChange(f.id, NO_DICT)}
                                                disabled={saving} title={t('uploadInfer.dictModal.disassociate')}>
                                                {t('uploadInfer.dictModal.disassociate')}
                                            </button>
                                        )}
                                    </div>
                                </div>
                            );
                        })}
                    </div>

                    {/* Detail drawer */}
                    {detailDictId !== null && (
                        <div className={styles.detailPane}>
                            <div className={styles.detailHead}>
                                <div className={styles.detailTitle}>{detailData?.dictionary_name ?? dictName(detailDictId)}</div>
                                <button className={styles.detailClose}
                                    onClick={() => { setDetailDictId(null); setDetailData(null); }}
                                    title={t('uploadInfer.dictModal.detailClose')}
                                    aria-label={t('uploadInfer.dictModal.detailClose')}>
                                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                                        <path d="M4 4l8 8M12 4l-8 8" />
                                    </svg>
                                </button>
                            </div>
                            {detailLoading && (
                                <div className={styles.listState}>
                                    <div className={styles.spinner} /><span>{t('uploadInfer.dictModal.loadingDetails')}</span>
                                </div>
                            )}
                            {detailError && <div className={`${styles.listState} ${styles.errorState}`}>{detailError}</div>}
                            {detailData && (
                                <>
                                    <div className={styles.detailMeta}>
                                        <div><span className={styles.metaLabel}>{t('uploadInfer.dictModal.metaId')}</span> {detailData.dictionary_id}</div>
                                        <div><span className={styles.metaLabel}>{t('uploadInfer.dictModal.metaUpdated')}</span> {detailData.updated_at}</div>
                                        <div><span className={styles.metaLabel}>{t('uploadInfer.dictModal.metaTerms')}</span> {detailData.dictionary_terms.length}</div>
                                    </div>
                                    <div className={styles.termsTable}>
                                        <div className={styles.termsHead}>
                                            <span>{t('uploadInfer.dictModal.wrongTerm')}</span>
                                            <span>{t('uploadInfer.dictModal.correctTerm')}</span>
                                        </div>
                                        {detailData.dictionary_terms.length === 0 ? (
                                            <div className={styles.listState}>{t('uploadInfer.dictModal.noTerms')}</div>
                                        ) : detailData.dictionary_terms.map((term, i) => (
                                            <div key={i} className={styles.termRow}>
                                                <span className={styles.termWrong}>{term.wrong_term}</span>
                                                <span className={styles.termArrow}>→</span>
                                                <span className={styles.termRight}>{term.correct_term}</span>
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
                            <span className={styles.spinnerSm} />{t('uploadInfer.dictModal.savingHint')}
                        </span>
                    )}
                    <button className={`${styles.btn} ${styles.btnGhost}`} onClick={handleCloseClick} disabled={saving}>
                        {t('uploadInfer.dictModal.cancel')}
                    </button>
                    <button className={`${styles.btn} ${styles.btnPrimary}`} onClick={handleSave} disabled={!isDirty || saving}>
                        {saving ? t('uploadInfer.dictModal.saving') : t('uploadInfer.dictModal.save')}
                    </button>
                </div>
            </div>
        </div>
    );
};

export default DictionaryAssociationModal;























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
    summary_prompt: string; keyword_prompt: string; faq_prompt: string;
    inserted_at: string; updated_at: string;
}
interface PromptTemplateDetail {
    id: number; ms_user_id: number; name: string; description: string;
    summary_prompt: string; keyword_prompt?: string; keywords_prompt?: string;
    faq_prompt: string; inserted_at: string; updated_at: string;
}
interface Props { open: boolean; onClose: () => void; }

const NO_TEMPLATE = -1;

const PromptTemplateAssociationModal: React.FC<Props> = ({ open, onClose }) => {
    const dispatch = useAppDispatch();
    const { t } = useTranslation();
    const serverFiles = useAppSelector(s => s.upload.serverFiles);

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

    useEffect(() => {
        if (!open) return;
        const init: Record<number, number> = {};
        serverFiles.forEach(f => { init[f.id] = f.prompt_template_id ?? NO_TEMPLATE; });
        setSelection(init);
        initialSelectionRef.current = init;
        setBulkValue(NO_TEMPLATE);
        setDetailTemplateId(null); setDetailData(null); setDetailError(null);

        let cancelled = false;
        (async () => {
            setTemplatesLoading(true); setTemplatesError(null);
            try {
                const res = await api.get('/prompt_template/list_templates');
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
                api.post('/prompt_template/associate', { file_ids: fileIds, template_id: Number(templateIdStr) })
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
                dispatch(updateFilePrompts({ fileIds, summaryPrompt: tmpl.summary_prompt, keywordsPrompt: tmpl.keyword_prompt, faqPrompt: tmpl.faq_prompt }));
            });
            if (disassociateIds.length > 0) {
                disassociateIds.forEach(id => { templatePatch[id] = null; });
                dispatch(updateFilePrompts({ fileIds: disassociateIds, summaryPrompt: '', keywordsPrompt: '', faqPrompt: '' }));
            }
            dispatch(patchFilePromptTemplate(templatePatch));
            dispatch(addToast(t('uploadInfer.templateModal.saveSuccess'), 'success'));
            onClose();
        } catch {
            dispatch(addToast(t('uploadInfer.templateModal.saveFail'), 'error'));
        } finally { setSaving(false); }
    }, [isDirty, saving, dirtyIds, selection, templates, dispatch, onClose, t]);

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

                {/* Bulk bar */}
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
