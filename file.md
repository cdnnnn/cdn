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
    const { serverFiles, dateFrom, dateTo } = useAppSelector(s => s.upload);

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

    // ── Associate ALL — every file in the current date range, bypasses the
    //     per-row preview entirely and hits the backend directly. ──
    const [isAssociatingAll, setIsAssociatingAll] = useState(false);
    const [associateAllError, setAssociateAllError] = useState<string | null>(null);

    useEffect(() => {
        if (!open) return;
        const init: Record<number, number> = {};
        serverFiles.forEach(f => { init[f.id] = f.dictionary_id ?? NO_DICT; });
        setSelection(init);
        initialSelectionRef.current = init;
        setBulkValue(NO_DICT);
        setDetailDictId(null); setDetailData(null); setDetailError(null);
        setAssociateAllError(null);

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
                calls.push(api.post('/dictionary/associate', { file_ids: fileIds, dictionary_id: Number(dictIdStr), all: false }));
            }
            for (const id of disassociateIds) { calls.push(api.post('/dictionary/disassociate', { file_ids: id })); }
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

    // ── Associate ALL files in [dateFrom, dateTo] with bulkValue — skips the
    //     per-row preview/save flow and applies directly on the backend. ──
    const handleAssociateAll = useCallback(async () => {
        if (bulkValue === NO_DICT || saving || isAssociatingAll) return;
        setIsAssociatingAll(true);
        setAssociateAllError(null);
        try {
            await api.post('/dictionary/associate', {
                dictionary_id: bulkValue,
                all: true,
                start_date: dateFrom,
                end_date: dateTo,
            });
            // Best-effort local patch for whatever's currently loaded — files on
            // other pages will reflect the change next time they're fetched.
            const patch: Record<number, number | null> = {};
            serverFiles.forEach(f => { patch[f.id] = bulkValue; });
            dispatch(patchFileDictionaries(patch));
            dispatch(addToast(t('uploadInfer.dictModal.associateAllSuccess'), 'success'));
            onClose();
        } catch {
            setAssociateAllError(t('uploadInfer.dictModal.associateAllFail'));
        } finally {
            setIsAssociatingAll(false);
        }
    }, [bulkValue, saving, isAssociatingAll, dateFrom, dateTo, serverFiles, dispatch, onClose, t]);

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

                {/* Bulk bar — applies to files currently loaded in this modal (preview, needs Save) */}
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

                {/* Associate ALL bar — applies immediately to every file in the date
                    range on the backend, including files not currently loaded here */}
                <div className={styles.associateAllBar}>
                    <span className={styles.bulkLabel}>
                        {t('uploadInfer.dictModal.associateAllLabel', { from: dateFrom, to: dateTo })}
                    </span>
                    <button
                        className={styles.associateAllBtn}
                        onClick={handleAssociateAll}
                        disabled={bulkValue === NO_DICT || saving || isAssociatingAll}
                    >
                        {isAssociatingAll ? <span className={styles.spinnerSm} /> : t('uploadInfer.dictModal.associateAllBtn')}
                    </button>
                    {associateAllError && <span className={styles.associateAllError}>{associateAllError}</span>}
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
// pages/UploadInfer/DictionaryAssociationModal.module.scss
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
        border-color: var(--blue-bdr);
        box-shadow: 0 0 0 2px var(--blue-dim);
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
    border: 1px solid var(--blue-bdr);
    background: var(--blue-dim);
    color: var(--blue);
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
    grid-template-columns: 1fr 220px 140px;
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
    grid-template-columns: 1fr 220px 140px;
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
    background: var(--amber-dim);

    &:hover {
        background: var(--amber-dim);
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
    color: var(--green);
    background: var(--green-dim);
    border: 1px solid var(--green-bdr);
    border-radius: 4px;
}

.colDict {
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
        border-color: var(--blue-bdr);
        box-shadow: 0 0 0 2px var(--blue-dim);
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
    border-radius: var(--r);
    border: 1px solid var(--bdr2);
    background: var(--bg2);
    color: var(--t1);
    cursor: pointer;
    display: inline-flex;
    align-items: center;
    justify-content: center;

    svg {
        width: 14px;
        height: 14px;
    }

    &:hover:not(:disabled) {
        background: var(--blue-dim);
        color: var(--blue);
        border-color: var(--blue-bdr);
    }

    &:disabled {
        opacity: 0.5;
        cursor: not-allowed;
    }
}

.disBtn {
    height: 28px;
    padding: 0 10px;
    font-size: 11.5px;
    font-weight: 500;
    border-radius: var(--r);
    border: 1px solid var(--red-bdr);
    background: transparent;
    color: var(--red);
    cursor: pointer;
    font-family: inherit;

    &:hover:not(:disabled) {
        background: var(--red-dim);
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

.termsTable {
    flex: 1;
    overflow-y: auto;
    padding: 4px 6px 8px;
}

.termsHead {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 8px;
    padding: 8px 10px;
    font-size: 11px;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    color: var(--t2);
    border-bottom: 1px solid var(--bdr);
    position: sticky;
    top: 0;
    background: var(--bg2);
}

.termRow {
    display: grid;
    grid-template-columns: 1fr auto 1fr;
    gap: 8px;
    align-items: center;
    padding: 7px 10px;
    font-size: 12.5px;
    border-bottom: 1px solid var(--bdr);
}

.termWrong {
    color: var(--red);
    font-family: var(--font-mono);
    font-size: 12px;
}

.termArrow {
    color: var(--t2);
    font-size: 11px;
}

.termRight {
    color: var(--green);
    font-family: var(--font-mono);
    font-size: 12px;
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
    border-top-color: var(--blue);
    border-radius: 50%;
    animation: spin 0.9s linear infinite;
}

.spinnerSm {
    width: 12px;
    height: 12px;
    border: 2px solid var(--bdr2);
    border-top-color: var(--blue);
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
    background: var(--blue);
    color: var(--bg0);
    border-color: var(--blue);

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
        for (const id of dirtyIds) {
            const v = selection[id];
            if (v === NO_TEMPLATE) continue;
            if (!associateGroups[v]) associateGroups[v] = [];
            associateGroups[v].push(id);
        }
        const entries = Object.entries(associateGroups);
        if (entries.length === 0) { setSaving(false); onClose(); return; }
        try {
            await Promise.all(entries.map(([templateIdStr, fileIds]) =>
                api.post('/prompt_template/associate', { file_ids: fileIds, template_id: Number(templateIdStr), all: false })
            ));
            const templatePatch: Record<number, number | null> = {};
            entries.forEach(([templateIdStr, fileIds]) => {
                const tmpl = templates.find(t => t.id === Number(templateIdStr));
                fileIds.forEach(id => { templatePatch[id] = Number(templateIdStr); });
                if (!tmpl) return;
                dispatch(updateFilePrompts({ fileIds, summaryPrompt: tmpl.summary_prompt, keywordsPrompt: tmpl.keyword_prompt, faqPrompt: tmpl.faq_prompt }));
            });
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
                dispatch(updateFilePrompts({ fileIds, summaryPrompt: tmpl.summary_prompt, keywordsPrompt: tmpl.keyword_prompt, faqPrompt: tmpl.faq_prompt }));
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
    grid-template-columns: 1fr 220px 60px;
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
    grid-template-columns: 1fr 220px 60px;
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
    border-radius: var(--r);
    border: 1px solid var(--bdr2);
    background: var(--bg2);
    color: var(--t1);
    cursor: pointer;
    display: inline-flex;
    align-items: center;
    justify-content: center;

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
