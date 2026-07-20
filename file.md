// ═══════════════════════════════════════════════
// pages/SttTranscription/components/FileLibrary.tsx
// ═══════════════════════════════════════════════
import React, { useEffect, useMemo, useState, useCallback } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppDispatch, useAppSelector } from '../../../store/hooks';
import {
    fetchSttFiles,
    setSttDateFrom,
    setSttDateTo,
    clearSttFiles,
    selectSttFileViews,
    selectIsAnyRunning,
    openInference,
    closeInference,
    fetchInferenceProgress,
    selectInferenceRunning,
    type SttFileView,
} from '../../../store/sttSlice';
import { addToast } from '../../../store/toastSlice';
import FileCard from './FileCard';
import UploadInline from './UploadInline';
import InferencePanel from './InferencePanel';
import TourGuide, { type TourStep } from './TourGuide';
import styles from '../SttTranscription.module.scss';
import sttApi from '../../../services/sttApi';

interface Props { onOpen: (id: number) => void; }

// Panel modes — only one can be active at a time
type PanelMode = 'none' | 'inference' | 'delete' | 'pipeline';

// ── Icons ──────────────────────────────────────
const IconCalendar: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <rect x="2" y="3.5" width="12" height="10" rx="1.5" /><path d="M2 6h12M5.5 2v3M10.5 2v3" />
    </svg>
);
const IconRefresh: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <path d="M13.5 8a5.5 5.5 0 11-1.6-3.9" /><path d="M13.5 2.5v3h-3" />
    </svg>
);
const IconSearch: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <circle cx="6.5" cy="6.5" r="4" /><path d="M11 11l2.5 2.5" />
    </svg>
);
const IconClose: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
        <path d="M4 4l8 8M12 4l-8 8" />
    </svg>
);
const IconInference: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <path d="M8 1.5l1.2 3 3 1.2-3 1.2L8 10l-1.2-3.1-3-1.2 3-1.2z" />
        <path d="M12.5 10l.7 1.8 1.8.7-1.8.7-.7 1.8-.7-1.8-1.8-.7 1.8-.7z" />
    </svg>
);
const IconLibrary: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <rect x="2" y="3" width="4" height="10" rx="1" />
        <rect x="7" y="3" width="4" height="10" rx="1" />
        <rect x="12" y="5" width="2" height="8" rx="1" />
    </svg>
);
const IconDelete: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <path d="M3 4h10M6 4V2.5h4V4M5 4l.5 9h5L11 4" />
    </svg>
);
const IconPipeline: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <path d="M8 10V3M5 6l3-3 3 3" />
        <path d="M3 11v2h10v-2" />
    </svg>
);
// ── Date helpers ───────────────────────────────
const toInputDate = (d: Date) => {
    const pad = (n: number) => (n < 10 ? `0${n}` : String(n));
    return `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())}`;
};
const defaultDateRange = () => {
    const to = new Date(); const from = new Date();
    from.setDate(to.getDate() - 30);
    return { from: toInputDate(from), to: toInputDate(to) };
};

const groupByDate = (files: SttFileView[], unknownLabel: string): { label: string; files: SttFileView[] }[] => {
    const dateLabel = (iso: string): string => {
        const d = new Date(iso);
        if (isNaN(d.getTime())) return unknownLabel;
        return d.toLocaleDateString(undefined, { month: 'short', day: 'numeric', year: 'numeric' });
    };
    const map = new Map<string, SttFileView[]>();
    for (const f of files) {
        const lbl = dateLabel(f.inserted_at);
        if (!map.has(lbl)) map.set(lbl, []);
        map.get(lbl)!.push(f);
    }
    return Array.from(map.entries()).map(([label, files]) => ({ label, files }));
};

const DATE_LABEL_COLORS: string[] = [
    '#818cf8','#38bdf8','#34d399','#fb923c','#f472b6',
    '#a78bfa','#22d3ee','#4ade80','#fbbf24','#f87171',
    '#60a5fa','#2dd4bf','#86efac','#fcd34d','#f9a8d4',
    '#c084fc','#67e8f9','#6ee7b7','#fde68a','#fca5a5',
    '#93c5fd','#5eead4','#a7f3d0','#fef08a','#fecaca',
    '#7dd3fc','#99f6e4','#bbf7d0','#fed7aa','#fbcfe8',
];

const FileLibrary: React.FC<Props> = ({ onOpen }) => {
    const dispatch = useAppDispatch();
    const { t } = useTranslation();
    const files        = useAppSelector(selectSttFileViews);
    const filesLoading = useAppSelector(s => s.stt.filesLoading);
    const filesError   = useAppSelector(s => s.stt.filesError);
    const [initializing, setInitializing] = useState(true);
    const dateFrom     = useAppSelector(s => s.stt.dateFrom);
    const dateTo       = useAppSelector(s => s.stt.dateTo);
    const inferenceRunning = useAppSelector(selectInferenceRunning);

    // ── Panel mode ────────────────────────────────
    const [panelMode, setPanelMode] = useState<PanelMode>('none');
    const inferenceOpen = panelMode === 'inference';
    useEffect(() => {
        if (inferenceOpen) dispatch(openInference());
        else dispatch(closeInference());
    }, [inferenceOpen, dispatch]);

    const openPanel  = (mode: PanelMode) => setPanelMode(prev => prev === mode ? 'none' : mode);
    const closePanel = () => { setPanelMode('none'); setSelectedIds(new Set()); };

    // ── Tour ──────────────────────────────────────
    const [tourActive, setTourActive] = useState(false);
    const tourSteps: TourStep[] = [
        { target: 'stt-title',            title: t('stt.tour.titleTitle'),            content: t('stt.tour.titleBody'),            placement: 'bottom' },
        { target: 'stt-date-filter',      title: t('stt.tour.dateFilterTitle'),       content: t('stt.tour.dateFilterBody'),       placement: 'bottom' },
        { target: 'stt-upload',           title: t('stt.tour.uploadTitle'),           content: t('stt.tour.uploadBody'),           placement: 'right' },
        { target: 'stt-library-heading',  title: t('stt.tour.libraryTitle'),          content: t('stt.tour.libraryBody'),          placement: 'bottom' },
        { target: 'stt-status-chips',     title: t('stt.tour.statusChipsTitle'),      content: t('stt.tour.statusChipsBody'),      placement: 'bottom' },
        {
            target: 'stt-action-inference',
            title: t('stt.tour.actionInferenceTitle'),
            content: t('stt.tour.actionInferenceBody'),
            placement: 'bottom',
            // Ensure no other panel is open so this button is visible
            onEnter: () => setPanelMode('none'),
        },
        {
            target: 'stt-action-delete',
            title: t('stt.tour.actionDeleteTitle'),
            content: t('stt.tour.actionDeleteBody'),
            placement: 'bottom',
            onEnter: () => setPanelMode('none'),
        },
        {
            target: 'stt-action-pipeline',
            title: t('stt.tour.actionPipelineTitle'),
            content: t('stt.tour.actionPipelineBody'),
            placement: 'bottom',
            onEnter: () => setPanelMode('none'),
        },
        { target: 'stt-library-body',     title: t('stt.tour.fileCardsTitle'),        content: t('stt.tour.fileCardsBody'),        placement: 'top' },
    ];

    // ── Selection ──────────────────────────────
    const [selectedIds, setSelectedIds] = useState<Set<number>>(new Set());
    const toggleFileSelect = useCallback((id: number) => {
        setSelectedIds(prev => {
            const next = new Set(prev);
            next.has(id) ? next.delete(id) : next.add(id);
            return next;
        });
    }, []);
    useEffect(() => { if (panelMode === 'none') setSelectedIds(new Set()); }, [panelMode]);

    // ── Search ──────────────────────────────────
    const [searchOpen, setSearchOpen]   = useState(false);
    const [searchQuery, setSearchQuery] = useState('');

    // ── Status filter ────────────────────────────
    type StatusFilter = 'all' | 'completed' | 'pending' | 'queued' | 'running';
    const [statusFilter, setStatusFilter] = useState<StatusFilter>('all');

    // ── Action states ───────────────────────────
    const [deleting, setDeleting]       = useState(false);
    const [deleteError, setDeleteError] = useState('');
    const [moving, setMoving]           = useState(false);
    const [moveError, setMoveError]     = useState('');

    // ── Date state ──────────────────────────────
    const freshRange = defaultDateRange();
    const [draftFrom, setDraftFrom] = useState(freshRange.from);
    const [draftTo, setDraftTo]     = useState(freshRange.to);
    const [dateError, setDateError] = useState('');

    const refreshLibrary = useCallback(() => {
        dispatch(clearSttFiles());
        dispatch(fetchSttFiles({ from: dateFrom, to: dateTo }));
    }, [dispatch, dateFrom, dateTo]);

    useEffect(() => {
        const def = defaultDateRange();
        setDraftFrom(def.from); setDraftTo(def.to);
        dispatch(setSttDateFrom(def.from)); dispatch(setSttDateTo(def.to));
        dispatch(clearSttFiles());
        dispatch(fetchInferenceProgress({ from: def.from, to: def.to }))
            .then((progressResult: any) => {
                if (fetchInferenceProgress.fulfilled.match(progressResult)) {
                    const { queued, running } = progressResult.payload;
                    if (queued.length > 0 || running.length > 0) setPanelMode('inference');
                }
            })
            .finally(() => {
                dispatch(fetchSttFiles({ from: def.from, to: def.to }))
                    .finally(() => setInitializing(false));
            });
        // eslint-disable-next-line react-hooks/exhaustive-deps
    }, []);

    const isDirty    = draftFrom !== dateFrom || draftTo !== dateTo;
    const isDateValid = !!draftFrom && !!draftTo && draftFrom <= draftTo;

    const validateDates = (from: string, to: string): boolean => {
        if (!from || !to) return true;
        if (from > to) { setDateError(t('stt.dateError')); return false; }
        setDateError(''); return true;
    };
    const handleFromChange = (val: string) => { setDraftFrom(val); validateDates(val, draftTo); };
    const handleToChange   = (val: string) => { setDraftTo(val);   validateDates(draftFrom, val); };
    const handleApply = () => {
        if (!validateDates(draftFrom, draftTo) || !isDirty) return;
        dispatch(setSttDateFrom(draftFrom)); dispatch(setSttDateTo(draftTo));
        dispatch(clearSttFiles());
        dispatch(fetchSttFiles({ from: draftFrom, to: draftTo }));
    };
    const handleRefresh = () => { dispatch(clearSttFiles()); dispatch(fetchSttFiles()); };

    // ── Delete ──────────────────────────────────
    const handleDelete = async () => {
        if (selectedIds.size === 0 || deleting) return;
        setDeleting(true); setDeleteError('');
        try {
            const res = await sttApi.deleteFiles(Array.from(selectedIds));
            if (res.status === 'success') {
                dispatch(addToast(t('stt.library.deletedSuccess', { count: selectedIds.size }), 'success'));
                closePanel(); refreshLibrary();
            } else {
                const msg = t('stt.library.deleteFail');
                setDeleteError(msg); dispatch(addToast(msg, 'error'));
            }
        } catch {
            const msg = t('stt.library.deleteFail');
            setDeleteError(msg); dispatch(addToast(msg, 'error'));
        } finally { setDeleting(false); }
    };

    // ── Move to lecture pipeline ─────────────────
    const handleMovePipeline = async () => {
        if (selectedIds.size === 0 || moving) return;
        setMoving(true); setMoveError('');
        try {
            const res = await sttApi.addToLecturePipeline(Array.from(selectedIds));
            if (res.status === 'success') {
                dispatch(addToast(t('stt.library.movedSuccess', { count: selectedIds.size }), 'success'));
                closePanel(); refreshLibrary();
            } else {
                const msg = t('stt.library.moveFail');
                setMoveError(msg); dispatch(addToast(msg, 'error'));
            }
        } catch {
            const msg = t('stt.library.moveFail');
            setMoveError(msg); dispatch(addToast(msg, 'error'));
        } finally { setMoving(false); }
    };

    // ── Derived data ─────────────────────────────
    const normalizedQuery = searchQuery.trim().toLowerCase();
    const matchesSearch = (f: SttFileView) => {
        if (!normalizedQuery) return true;
        return String(f.id).includes(normalizedQuery) || f.original_name.toLowerCase().includes(normalizedQuery);
    };
    const matchesStatus = (f: SttFileView) => statusFilter === 'all' || f.status === statusFilter;

    const { processing, library } = useMemo(() => {
        const byDate = (a: SttFileView, b: SttFileView) =>
            new Date(b.inserted_at).getTime() - new Date(a.inserted_at).getTime();
        const processing = files.filter(f => f.status === 'queued' || f.status === 'running');
        const library    = [...files].sort(byDate);
        return { processing, library };
    }, [files]);

    const completedFiles = library.filter(f => f.status === 'completed');
    const visibleLibrary = library.filter(f => matchesSearch(f) && matchesStatus(f));

    const isSelectionMode = panelMode !== 'none' && !inferenceRunning;
    const canSelectFile   = (f: SttFileView) => panelMode === 'pipeline' ? f.status === 'completed' : true;
    const selectAllForMode = () => {
        if (panelMode === 'pipeline') setSelectedIds(new Set(completedFiles.map(f => f.id)));
        else setSelectedIds(new Set(library.map(f => f.id)));
    };

    const actionLabel = panelMode === 'delete'
        ? (deleting ? t('stt.library.deleting', { count: selectedIds.size }) : t('stt.library.delete', { count: selectedIds.size }))
        : panelMode === 'pipeline'
        ? (moving ? t('stt.library.moving', { count: selectedIds.size }) : t('stt.library.move', { count: selectedIds.size }))
        : '';

    const handleAction  = panelMode === 'delete' ? handleDelete : handleMovePipeline;
    const isActing      = panelMode === 'delete' ? deleting : moving;
    const actionError   = panelMode === 'delete' ? deleteError : moveError;
    const actionBtnCls  = panelMode === 'delete' ? `${styles.btn} ${styles.btnRed}` : `${styles.btn} ${styles.btnBlue}`;
    const showInferenceCol = inferenceOpen;

    return (
        <div className={styles.libPage}>
            {/* Tour guide */}
            <TourGuide steps={tourSteps} active={tourActive} onFinish={() => setTourActive(false)} />

            {/* ── Page header ── */}
            <div className={styles.ph}>
                <div className={styles.phRow}>
                    <div style={{ display: 'flex', alignItems: 'center', gap: 10 }}>
                        <div>
                            <div className={styles.phTitle} data-tour="stt-title">{t('stt.pageTitle')}</div>
                            <div className={styles.phSub}>
                                {(initializing || filesLoading)
                                    ? t('stt.loading')
                                    : `${t('stt.file', { count: files.length })}${processing.length > 0 ? t('stt.processing', { count: processing.length }) : ''}`}
                            </div>
                        </div>
                        {/* Tour trigger button */}
                        <button
                            type="button"
                            className={styles.tourTriggerBtn}
                            onClick={() => setTourActive(true)}
                        >
                            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                <circle cx="8" cy="8" r="6.25" />
                                <path d="M6.1 6.2a1.9 1.9 0 013.6.7c0 1.3-1.7 1.5-1.7 2.7M8 11.4v.1" />
                            </svg>
                            {t('stt.tour.takeTour')}
                        </button>
                    </div>
                    <div className={styles.phActs}>
                        <div data-tour="stt-date-filter" className={`${styles.dateFilter} ${inferenceRunning ? styles.dateFilterDisabled : ''}`}>
                            <span className={styles.dateFilterIc}><IconCalendar /></span>
                            <input type="date" className={styles.dateInput} value={draftFrom}
                                onChange={e => handleFromChange(e.target.value)}
                                onKeyDown={e => { if (e.key === 'Enter') handleApply(); }}
                                disabled={inferenceRunning || initializing || filesLoading} />
                            <span className={styles.dateSep}>–</span>
                            <input type="date" className={styles.dateInput} value={draftTo}
                                onChange={e => handleToChange(e.target.value)}
                                onKeyDown={e => { if (e.key === 'Enter') handleApply(); }}
                                disabled={inferenceRunning || filesLoading} />
                            {isDirty && isDateValid && !inferenceRunning && (
                                <button className={styles.dateApply} onClick={handleApply} disabled={initializing || filesLoading}>{t('stt.apply')}</button>
                            )}
                        </div>
                        {dateError && <span className={styles.dateValidationError}>{dateError}</span>}
                        <button className={styles.btn} onClick={handleRefresh}
                            disabled={initializing || filesLoading || inferenceRunning}
                            title={inferenceRunning ? t('stt.refreshDisabled') : t('stt.refreshTitle')}>
                            <IconRefresh />
                        </button>
                    </div>
                </div>
            </div>

            {/* ── Three-column body ── */}
            <div className={`${styles.libBody} ${showInferenceCol ? styles.libBodyInference : ''}`}>
                {/* COL 1: Upload */}
                <div data-tour="stt-upload">
                    <UploadInline />
                </div>

                {/* COL 2: Library */}
                <div className={styles.libCol}>
                    <div className={styles.colHeading} data-tour="stt-library-heading">
                        <span className={styles.colHeadingIc}><IconLibrary /></span>
                        <span className={styles.colHeadingText}>{t('stt.library.title')}</span>
                        <div className={styles.colHeadingRight}>
                            {searchOpen && panelMode !== 'none' ? (
                                <button
                                    className={`${styles.filterIconBtn} ${statusFilter !== 'all' ? styles.filterIconBtnActive : ''}`}
                                    onClick={() => { setSearchOpen(false); setSearchQuery(''); }}
                                    title={t('stt.filterLabel', { filter: statusFilter })}
                                >
                                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                        <path d="M2 4h12M4 8h8M6 12h4" />
                                    </svg>
                                    {statusFilter !== 'all' && <span className={styles.filterIconDot} />}
                                </button>
                            ) : (
                                <div className={styles.statusFilterChips} data-tour="stt-status-chips">
                                    {(['all', 'completed', 'pending', 'queued', 'running'] as const).map(s => (
                                        <button key={s}
                                            className={`${styles.statusChip} ${statusFilter === s ? styles.statusChipActive : ''} ${s !== 'all' ? styles[`statusChip_${s}`] : ''}`}
                                            onClick={() => { setStatusFilter(s as StatusFilter); if (panelMode !== 'none' && searchOpen) { setSearchOpen(false); setSearchQuery(''); } }}>
                                            {s === 'all' ? t('stt.library.filterAll') : t(`stt.fileCard.status.${s}`)}
                                        </button>
                                    ))}
                                </div>
                            )}
                            <div className={styles.sectionHeaderRight}>
                                {searchOpen && (
                                    <div className={styles.searchBox}>
                                        <span className={styles.searchBoxIc}><IconSearch /></span>
                                        <input className={styles.searchBoxInput} type="text" placeholder={t('stt.library.searchPlaceholder')}
                                            value={searchQuery} onChange={e => setSearchQuery(e.target.value)} autoFocus
                                            onKeyDown={e => { if (e.key === 'Escape') { setSearchOpen(false); setSearchQuery(''); } }} />
                                        {searchQuery && <button className={styles.searchBoxClear} onClick={() => setSearchQuery('')}><IconClose /></button>}
                                    </div>
                                )}
                                <button className={`${styles.searchToggleBtn} ${searchOpen ? styles.searchToggleBtnActive : ''}`}
                                    onClick={() => { setSearchOpen(v => !v); if (searchOpen) setSearchQuery(''); }}>
                                    <IconSearch />
                                </button>
                            </div>
                        </div>
                    </div>

                    {/* ── Action toolbar ── */}
                    {panelMode === 'none' && !inferenceRunning && (
                        <div className={styles.actionToolbar}>
                            <button data-tour="stt-action-inference" className={`${styles.actionToolbarBtn} ${styles.actionToolbarBtnInference}`} onClick={() => openPanel('inference')}>
                                <IconInference />
                                <span>{t('stt.library.selectForInference')}</span>
                            </button>
                            <button data-tour="stt-action-delete" className={`${styles.actionToolbarBtn} ${styles.actionToolbarBtnDelete}`} onClick={() => openPanel('delete')}>
                                <IconDelete />
                                <span>{t('stt.library.deleteFiles')}</span>
                            </button>
                            <button data-tour="stt-action-pipeline" className={`${styles.actionToolbarBtn} ${styles.actionToolbarBtnPipeline}`} onClick={() => openPanel('pipeline')}>
                                <IconPipeline />
                                <span>{t('stt.library.movePipeline')}</span>
                            </button>
                        </div>
                    )}

                    {/* Sticky action bars */}
                    {(panelMode === 'delete' || panelMode === 'pipeline') && library.length > 0 && (
                        <div className={`${styles.selectAllRow} ${panelMode === 'delete' ? styles.selectAllRowDelete : styles.selectAllRowPipeline}`}>
                            <div className={styles.selectAllLeft}>
                                <span className={styles.selectAllHint}>
                                    {selectedIds.size > 0
                                        ? t('stt.library.selected', { count: selectedIds.size })
                                        : panelMode === 'pipeline' && completedFiles.length === 0
                                        ? t('stt.library.noPipelineFiles')
                                        : panelMode === 'pipeline'
                                        ? t('stt.library.selectCompleted')
                                        : t('stt.library.noneSelected')}
                                </span>
                                <div className={styles.selectAllActions}>
                                    <button className={styles.selectAllBtn} onClick={selectAllForMode}
                                        disabled={panelMode === 'pipeline'
                                            ? completedFiles.length === 0 || selectedIds.size === completedFiles.length
                                            : selectedIds.size === library.length}>
                                        {t('stt.library.selectAll')}
                                    </button>
                                    <button className={styles.selectAllBtn} onClick={() => setSelectedIds(new Set())} disabled={selectedIds.size === 0}>
                                        {t('stt.library.clear')}
                                    </button>
                                </div>
                            </div>
                            <div className={styles.selectAllRight}>
                                {actionError && <span className={styles.selectAllError}>{actionError}</span>}
                                <button className={actionBtnCls} disabled={selectedIds.size === 0 || isActing} onClick={handleAction}>
                                    {actionLabel}
                                </button>
                                <button className={styles.btn} onClick={closePanel} disabled={isActing}><IconClose /></button>
                            </div>
                        </div>
                    )}
                    {inferenceOpen && !inferenceRunning && library.length > 0 && (
                        <div className={styles.selectAllRow}>
                            <span className={styles.selectAllHint}>
                                {selectedIds.size > 0 ? t('stt.library.inferenceSelected', { count: selectedIds.size }) : t('stt.library.inferenceNone')}
                            </span>
                            <div className={styles.selectAllActions}>
                                <button className={styles.selectAllBtn} onClick={() => setSelectedIds(new Set(library.map(f => f.id)))}
                                    disabled={selectedIds.size === library.length}>{t('stt.library.selectAll')}</button>
                                <button className={styles.selectAllBtn} onClick={() => setSelectedIds(new Set())} disabled={selectedIds.size === 0}>{t('stt.library.clear')}</button>
                            </div>
                        </div>
                    )}

                    {/* Scrollable content */}
                    <div className={styles.colBody} data-tour="stt-library-body">
                        {filesError && (
                            <div className={styles.errorBanner}>
                                {t('stt.library.loadError', { error: filesError })}
                                <button className={styles.btn} onClick={handleRefresh}>{t('stt.library.retry')}</button>
                            </div>
                        )}
                        {(initializing || filesLoading) && (
                            <div className={styles.libLoading}>
                                <div className={styles.spinner} />
                                <span>{t('stt.library.loading')}</span>
                            </div>
                        )}
                        {library.length === 0 && !filesLoading && !initializing ? (
                            <div className={styles.libEmpty}>
                                <div className={styles.libEmptyTitle}>{t('stt.library.emptyTitle')}</div>
                                <div className={styles.libEmptyHint}>{t('stt.library.emptyHint')}</div>
                            </div>
                        ) : visibleLibrary.length === 0 && normalizedQuery ? (
                            <div className={styles.libEmpty}>
                                <div className={styles.libEmptyTitle}>{t('stt.library.noMatchTitle', { query: searchQuery })}</div>
                                <div className={styles.libEmptyHint}>{t('stt.library.noMatchHint')}</div>
                            </div>
                        ) : (
                            <div className={styles.dateGroups}>
                                {groupByDate(visibleLibrary, t('stt.unknownDate')).map((group, gi) => (
                                    <div key={group.label} className={styles.dateGroup}>
                                        <div className={styles.dateGroupLabel}
                                            style={{ color: DATE_LABEL_COLORS[gi % DATE_LABEL_COLORS.length] }}>
                                            <span className={styles.dateGroupDot} />
                                            {group.label}
                                        </div>
                                        <div className={styles.libCardGrid}>
                                            {group.files.map(f => {
                                                const selectable = isSelectionMode && canSelectFile(f);
                                                const dimmed     = isSelectionMode && !canSelectFile(f);
                                                const handleCardOpen = () => {
                                                    if (selectable) { toggleFileSelect(f.id); return; }
                                                    if (!isSelectionMode && f.status === 'completed') onOpen(f.id);
                                                };
                                                return (
                                                    <FileCard key={f.id} file={f} variant="card"
                                                        onOpen={handleCardOpen}
                                                        onGenerate={() => {}} disableGenerate={true}
                                                        selected={selectedIds.has(f.id)}
                                                        selectionMode={selectable}
                                                        dimmed={dimmed}
                                                    />
                                                );
                                            })}
                                        </div>
                                    </div>
                                ))}
                            </div>
                        )}
                    </div>
                </div>

                {/* COL 3: Inference panel */}
                {showInferenceCol && (
                    <InferencePanel
                        selectedIds={selectedIds}
                        onSubmitted={() => setSelectedIds(new Set())}
                        onClose={closePanel}
                    />
                )}
            </div>
        </div>
    );
};

export default FileLibrary;
















{
  "_comment": "Add this 'tour' block inside the 'stt' object in en.json",
  "stt": {
    "tour": {
      "takeTour": "Take a tour",
      "titleTitle": "Welcome to the STT Pipeline",
      "titleBody": "Upload audio or video files, run AI transcription, edit captions, and manage your library — all in one place. This tour will walk you through the key features.",
      "dateFilterTitle": "Date Filter",
      "dateFilterBody": "Filter your library by date range. Set the From and To dates and click Apply to load files from a specific period. The Refresh button reloads the current range.",
      "uploadTitle": "Upload Files",
      "uploadBody": "Drag and drop audio or video files here, or use Browse Files / Browse Folder. Up to 5 files upload at once. Supported formats: MP3, WAV, AAC, FLAC, MP4, MOV and more.",
      "libraryTitle": "File Library",
      "libraryBody": "All your uploaded files appear here. Use the status chips to filter by status, or the search icon to find files by name or ID.",
      "statusChipsTitle": "Status Filters",
      "statusChipsBody": "Filter files by status: All, Completed, Pending, Queued, or Running. Only Completed files can be opened to view and edit their transcript.",
      "actionInferenceTitle": "Select Files for Inference",
      "actionInferenceBody": "Click this to enter selection mode for transcription. Any file can be selected — checkboxes appear on each card. Choose a model and language, then run inference to transcribe the selected files.",
      "actionDeleteTitle": "Delete Files",
      "actionDeleteBody": "Click this to enter delete mode. Select any files you want to permanently remove, then confirm with the Delete button. This action can't be undone, so use it carefully.",
      "actionPipelineTitle": "Move to Lecture Pipeline",
      "actionPipelineBody": "Click this to enter pipeline mode. Only Completed files can be selected here — Pending, Queued, and Running files are grayed out. Selected files are sent downstream to the Lecture Pipeline for further processing.",
      "fileCardsTitle": "File Cards",
      "fileCardsBody": "Each card shows the file name, ID, type, and status badge. A teal circle indicates the file is already in the Lecture Pipeline. Click any Completed card to open the transcript viewer and edit captions."
    }
  }
}



















{
  "_comment": "Add this 'tour' block inside the 'stt' object in ko.json",
  "stt": {
    "tour": {
      "takeTour": "둘러보기",
      "titleTitle": "STT 파이프라인 소개",
      "titleBody": "오디오 또는 동영상 파일을 업로드하고 AI 전사를 실행하며 자막을 편집하고 라이브러리를 관리하세요 — 모두 한 곳에서. 이 투어에서 주요 기능을 안내해 드립니다.",
      "dateFilterTitle": "날짜 필터",
      "dateFilterBody": "날짜 범위로 라이브러리를 필터링하세요. 시작일과 종료일을 설정하고 적용을 클릭하면 해당 기간의 파일을 불러옵니다. 새로고침 버튼은 현재 범위를 다시 불러옵니다.",
      "uploadTitle": "파일 업로드",
      "uploadBody": "오디오 또는 동영상 파일을 여기에 드래그 앤 드롭하거나 파일 찾아보기 / 폴더 찾아보기를 사용하세요. 한 번에 최대 5개 파일 업로드 가능. 지원 형식: MP3, WAV, AAC, FLAC, MP4, MOV 등.",
      "libraryTitle": "파일 라이브러리",
      "libraryBody": "업로드한 모든 파일이 여기에 표시됩니다. 상태 칩으로 필터링하거나 검색 아이콘으로 파일명 또는 ID를 검색하세요.",
      "statusChipsTitle": "상태 필터",
      "statusChipsBody": "전체, 완료, 대기, 대기 중, 실행 중 상태로 파일을 필터링합니다. 완료된 파일만 클릭하여 전사본을 보고 편집할 수 있습니다.",
      "actionInferenceTitle": "추론을 위한 파일 선택",
      "actionInferenceBody": "이 버튼을 클릭하면 전사를 위한 선택 모드로 전환됩니다. 모든 파일을 선택할 수 있으며 각 카드에 체크박스가 표시됩니다. 모델과 언어를 선택한 후 추론을 실행하면 선택한 파일이 전사됩니다.",
      "actionDeleteTitle": "파일 삭제",
      "actionDeleteBody": "이 버튼을 클릭하면 삭제 모드로 전환됩니다. 영구적으로 제거하려는 파일을 선택한 후 삭제 버튼으로 확인하세요. 이 작업은 되돌릴 수 없으니 신중하게 사용하세요.",
      "actionPipelineTitle": "강의 파이프라인으로 이동",
      "actionPipelineBody": "이 버튼을 클릭하면 파이프라인 모드로 전환됩니다. 완료된 파일만 선택할 수 있으며, 대기·대기열·실행 중인 파일은 회색으로 비활성화됩니다. 선택한 파일은 추가 처리를 위해 강의 파이프라인으로 전달됩니다.",
      "fileCardsTitle": "파일 카드",
      "fileCardsBody": "각 카드에는 파일명, ID, 유형, 상태 배지가 표시됩니다. 청록색 원은 이미 강의 파이프라인에 추가된 파일을 나타냅니다. 완료된 카드를 클릭하면 전사 뷰어를 열어 자막을 편집할 수 있습니다."
    }
  }
}
