// ═══════════════════════════════════════════════
// pages/SttTranscription/SttPipeline.tsx
//
// Top-level container — 3 persistent tabs (Upload & Manage / Inference /
// View Results), mirroring pages/UploadInfer/UploadInfer.tsx. All three
// tab panes stay mounted at all times (toggled via CSS display) so file
// lists, inference polling, and scroll position survive switching tabs
// instead of resetting.
// ═══════════════════════════════════════════════
import React, { useCallback, useEffect, useState } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import { selectSttFileViews, type SttFileView } from '../../store/sttSlice';
import FileLibrary from './components/FileLibrary';
import SttFileSidebar from './components/SttFileSidebar';
import InferencePanel from './components/InferencePanel';
import TranscriptDetail from './components/TranscriptDetail';
import styles from './SttTranscription.module.scss';

type TabId = 'upload' | 'infer' | 'results';

const SttPipeline: React.FC = () => {
    const { t } = useTranslation();
    const dispatch = useAppDispatch();
    const files = useAppSelector(selectSttFileViews);

    const [activeTab, setActiveTab] = useState<TabId>('upload');

    // ── Inference tab — selection lifted here so it survives switching
    // away and back, and so InferencePanel and SttFileSidebar share it ──
    const [selectedIds, setSelectedIds] = useState<Set<number>>(new Set());
    const toggleSelect = useCallback((id: number) => {
        setSelectedIds(prev => {
            const next = new Set(prev);
            next.has(id) ? next.delete(id) : next.add(id);
            return next;
        });
    }, []);
    const selectAll = useCallback((ids: number[]) => setSelectedIds(new Set(ids)), []);
    const clearSelection = useCallback(() => setSelectedIds(new Set()), []);

    // ── Results tab — active file, lifted so both the sidebar and detail
    // view agree on what's open, and so clicking a file from the Upload
    // tab's library can jump straight here ──
    const [activeFileId, setActiveFileId] = useState<number | null>(null);
    const activeFile: SttFileView | undefined = activeFileId
        ? files.find(f => f.id === activeFileId)
        : undefined;

    // Bail out of the open file if it disappears (deleted, moved, etc.)
    useEffect(() => {
        if (activeFileId && !activeFile) setActiveFileId(null);
    }, [activeFileId, activeFile]);

    // Clicking a file (from Upload tab's library, or the Results sidebar)
    // opens its transcript and jumps to the Results tab.
    const handleFileClick = useCallback((fileId: number) => {
        setActiveFileId(fileId);
        setActiveTab('results');
    }, []);

    const goToInfer = useCallback(() => setActiveTab('infer'), []);

    const tabs: { id: TabId; label: string; desc: string }[] = [
        { id: 'upload',  label: t('stt.tabs.upload'),  desc: t('stt.tabs.uploadDesc') },
        { id: 'infer',   label: t('stt.tabs.infer'),   desc: t('stt.tabs.inferDesc') },
        { id: 'results', label: t('stt.tabs.results'), desc: t('stt.tabs.resultsDesc') },
    ];

    return (
        <div className={styles.sttPage}>
            {/* ── Header — title + tab bar ── */}
            <div className={styles.sttHeaderBar}>
                <div className={styles.sttPhTitleRow}>
                    <div className={styles.phTitle}>{t('stt.pageTitle')}</div>
                </div>

                <div className={styles.sttTabbar} role="tablist" data-tour="stt-tabbar">
                    {tabs.map(tab => (
                        <button
                            key={tab.id}
                            type="button"
                            role="tab"
                            aria-selected={activeTab === tab.id}
                            className={`${styles.sttTabBtn} ${activeTab === tab.id ? styles.sttTabBtnActive : ''}`}
                            onClick={() => setActiveTab(tab.id)}
                        >
                            <span className={styles.sttTabLabel}>{tab.label}</span>
                            <span className={styles.sttTabDesc}>{tab.desc}</span>
                        </button>
                    ))}
                </div>
            </div>

            {/* ── Tab 1: Upload & Manage ── */}
            <div className={styles.sttTabPane} style={{ display: activeTab === 'upload' ? 'flex' : 'none' }}>
                <FileLibrary
                    onOpen={handleFileClick}
                    onGoToInfer={goToInfer}
                    active={activeTab === 'upload'}
                />
            </div>

            {/* ── Tab 2: Inference ── */}
            <div className={styles.sttTabPane} style={{ display: activeTab === 'infer' ? 'flex' : 'none' }}>
                <div className={styles.sttInferTabBody}>
                    <SttFileSidebar
                        mode="select"
                        active={activeTab === 'infer'}
                        selectedIds={selectedIds}
                        onToggleSelect={toggleSelect}
                        onSelectAll={selectAll}
                        onClearSelection={clearSelection}
                    />
                    <div className={styles.sttInferMain}>
                        <InferencePanel
                            selectedIds={selectedIds}
                            onSubmitted={clearSelection}
                            onCompleted={clearSelection}
                        />
                    </div>
                </div>
            </div>

            {/* ── Tab 3: View Results ── */}
            <div className={styles.sttTabPane} style={{ display: activeTab === 'results' ? 'flex' : 'none' }}>
                <div className={styles.sttResultsTabBody}>
                    <SttFileSidebar
                        mode="view"
                        active={activeTab === 'results'}
                        onlyCompleted
                        activeFileId={activeFileId}
                        onFileClick={f => setActiveFileId(f.id)}
                    />
                    <div className={styles.sttResultsMain} data-tour="results-content">
                        {activeFile ? (
                            <TranscriptDetail
                                file={activeFile}
                                onBack={() => setActiveFileId(null)}
                            />
                        ) : (
                            <div className={styles.sttResultsEmpty}>
                                <div className={styles.sttResultsEmptyTitle}>{t('stt.results.noFileSelected')}</div>
                                <div className={styles.sttResultsEmptyHint}>{t('stt.results.noFileHint')}</div>
                            </div>
                        )}
                    </div>
                </div>
            </div>
        </div>
    );
};

export default SttPipeline;










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
    fetchInferenceProgress,
    selectInferenceRunning,
    type SttFileView,
} from '../../../store/sttSlice';
import { addToast } from '../../../store/toastSlice';
import FileCard from './FileCard';
import UploadInline from './UploadInline';
import TourGuide, { type TourStep } from './TourGuide';
import styles from '../SttTranscription.module.scss';
import sttApi from '../../../services/sttApi';

interface Props {
    onOpen: (id: number) => void;
    onGoToInfer: () => void; // switch the parent tab bar to the Inference tab
    active: boolean;         // whether this tab is currently visible (for tour targeting)
}

// Panel modes — only one can be active at a time (inference selection now lives in the Inference tab)
type PanelMode = 'none' | 'delete' | 'pipeline';

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

const FileLibrary: React.FC<Props> = ({ onOpen, onGoToInfer, active }) => {
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
                    if (queued.length > 0 || running.length > 0) onGoToInfer();
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
            <div className={styles.libBody}>
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
                            <button data-tour="stt-action-inference" className={`${styles.actionToolbarBtn} ${styles.actionToolbarBtnInference}`} onClick={onGoToInfer}>
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
            </div>
        </div>
    );
};

export default FileLibrary;













// ═══════════════════════════════════════════════
// pages/SttTranscription/components/SttFileSidebar.tsx
//
// Reusable file-picker column used by both the Inference tab
// (mode="select" — checkbox multi-select feeding InferencePanel) and
// the View Results tab (mode="view" — click a completed file to open
// its transcript). Has its own search / status filter, independent of
// whatever filter state the Upload & Manage tab is using.
// ═══════════════════════════════════════════════
import React, { useMemo, useState } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppSelector } from '../../../store/hooks';
import { selectSttFileViews, type SttFileView } from '../../../store/sttSlice';
import styles from '../SttTranscription.module.scss';

type Mode = 'select' | 'view';

interface Props {
    mode: Mode;
    active: boolean;
    // select mode
    selectedIds?: Set<number>;
    onToggleSelect?: (id: number) => void;
    onSelectAll?: (ids: number[]) => void;
    onClearSelection?: () => void;
    // view mode
    activeFileId?: number | null;
    onFileClick?: (file: SttFileView) => void;
    // view mode only shows Completed files (nothing else has a transcript yet)
    onlyCompleted?: boolean;
}

const IconSearch: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <circle cx="6.5" cy="6.5" r="4" /><path d="M11 11l2.5 2.5" />
    </svg>
);
const IconCheck: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
        <path d="M3 8.5l3.5 3.5 7-7.5" />
    </svg>
);

type SortKey = 'id' | 'name' | 'date' | 'status';

const SttFileSidebar: React.FC<Props> = ({
    mode, active,
    selectedIds, onToggleSelect, onSelectAll, onClearSelection,
    activeFileId, onFileClick,
    onlyCompleted = false,
}) => {
    const { t } = useTranslation();
    const files = useAppSelector(selectSttFileViews);

    const [search, setSearch]     = useState('');
    const [status, setStatus]     = useState<'all' | 'completed' | 'pending' | 'queued' | 'running'>(
        onlyCompleted ? 'completed' : 'all'
    );
    const [sortKey, setSortKey]   = useState<SortKey>('date');
    const [sortAsc, setSortAsc]   = useState(false);

    const toggleSort = (key: SortKey) => {
        if (sortKey === key) setSortAsc(v => !v);
        else { setSortKey(key); setSortAsc(true); }
    };

    const filtered = useMemo(() => {
        const q = search.trim().toLowerCase();
        let list = files.filter(f => {
            if (mode === 'view' && onlyCompleted && f.status !== 'completed') return false;
            if (status !== 'all' && f.status !== status) return false;
            if (q && !(String(f.id).includes(q) || f.original_name.toLowerCase().includes(q))) return false;
            return true;
        });
        list = [...list].sort((a, b) => {
            let cmp = 0;
            if (sortKey === 'id')     cmp = a.id - b.id;
            else if (sortKey === 'name')   cmp = a.original_name.localeCompare(b.original_name);
            else if (sortKey === 'status') cmp = a.status.localeCompare(b.status);
            else cmp = new Date(a.inserted_at).getTime() - new Date(b.inserted_at).getTime();
            return sortAsc ? cmp : -cmp;
        });
        return list;
    }, [files, search, status, sortKey, sortAsc, mode, onlyCompleted]);

    const statusOptions: Array<'all' | 'completed' | 'pending' | 'queued' | 'running'> =
        mode === 'view' && onlyCompleted ? ['completed'] : ['all', 'completed', 'pending', 'queued', 'running'];

    return (
        <div className={styles.sttSidebar} data-tour={mode === 'select' ? 'infer-sidebar' : 'results-sidebar'}>
            <div className={styles.sttSidebarHeader}>
                <span className={styles.sttSidebarTitle}>
                    {mode === 'select' ? t('stt.sidebar.pickForInference') : t('stt.sidebar.pickToView')}
                </span>
                <span className={styles.sttSidebarCount}>{filtered.length}</span>
            </div>

            {/* Search */}
            <div className={styles.sttSidebarSearch} data-tour={mode === 'select' ? 'infer-search' : 'results-search'}>
                <IconSearch />
                <input
                    type="text"
                    value={search}
                    onChange={e => setSearch(e.target.value)}
                    placeholder={t('stt.library.searchPlaceholder')}
                />
            </div>

            {/* Status filter */}
            {statusOptions.length > 1 && (
                <div className={styles.sttSidebarChips} data-tour={mode === 'select' ? 'infer-status-filter' : 'results-status-filter'}>
                    {statusOptions.map(s => (
                        <button key={s}
                            className={`${styles.statusChip} ${status === s ? styles.statusChipActive : ''} ${s !== 'all' ? styles[`statusChip_${s}`] : ''}`}
                            onClick={() => setStatus(s)}>
                            {s === 'all' ? t('stt.library.filterAll') : t(`stt.fileCard.status.${s}`)}
                        </button>
                    ))}
                </div>
            )}

            {/* Sort */}
            <div className={styles.sttSidebarSort} data-tour={mode === 'select' ? 'infer-sort' : 'results-sort'}>
                {(['id', 'name', 'date', 'status'] as SortKey[]).map(k => (
                    <button key={k}
                        className={`${styles.sttSortBtn} ${sortKey === k ? styles.sttSortBtnActive : ''}`}
                        onClick={() => toggleSort(k)}>
                        {t(`stt.sidebar.sort.${k}`)}
                        {sortKey === k && <span className={styles.sttSortArrow}>{sortAsc ? '↑' : '↓'}</span>}
                    </button>
                ))}
            </div>

            {/* Select all / clear — select mode only */}
            {mode === 'select' && (
                <div className={styles.sttSidebarSelectRow}>
                    <span className={styles.sttSidebarSelectHint}>
                        {selectedIds && selectedIds.size > 0
                            ? t('stt.library.selected', { count: selectedIds.size })
                            : t('stt.library.noneSelected')}
                    </span>
                    <div className={styles.selectAllActions}>
                        <button className={styles.selectAllBtn} onClick={() => onSelectAll?.(filtered.map(f => f.id))}
                            disabled={filtered.length === 0}>
                            {t('stt.library.selectAll')}
                        </button>
                        <button className={styles.selectAllBtn} onClick={onClearSelection} disabled={!selectedIds || selectedIds.size === 0}>
                            {t('stt.library.clear')}
                        </button>
                    </div>
                </div>
            )}

            {/* File list */}
            <div className={styles.sttSidebarList}>
                {filtered.length === 0 ? (
                    <div className={styles.sttSidebarEmpty}>
                        {mode === 'view' && onlyCompleted
                            ? t('stt.sidebar.noCompletedFiles')
                            : t('stt.library.emptyTitle')}
                    </div>
                ) : filtered.map(f => {
                    const isSelected = mode === 'select' && selectedIds?.has(f.id);
                    const isActive   = mode === 'view' && activeFileId === f.id;
                    const clickable  = mode === 'select' || (mode === 'view' && f.status === 'completed');
                    return (
                        <button
                            key={f.id}
                            type="button"
                            className={`${styles.sttSidebarRow} ${isSelected ? styles.sttSidebarRowSelected : ''} ${isActive ? styles.sttSidebarRowActive : ''} ${!clickable ? styles.sttSidebarRowDisabled : ''}`}
                            disabled={!clickable}
                            onClick={() => {
                                if (mode === 'select') onToggleSelect?.(f.id);
                                else onFileClick?.(f);
                            }}
                        >
                            {mode === 'select' && (
                                <span className={`${styles.sttSidebarCheckbox} ${isSelected ? styles.sttSidebarCheckboxChecked : ''}`}>
                                    {isSelected && <IconCheck />}
                                </span>
                            )}
                            <span className={styles.sttSidebarRowName} title={f.original_name}>{f.original_name}</span>
                            <span className={`${styles.pill} ${
                                f.status === 'completed' ? styles.pillGreen :
                                f.status === 'queued'    ? styles.pillAmber :
                                f.status === 'running'   ? styles.pillBlue :
                                f.status === 'failed'    ? styles.pillRed :
                                styles.pillGray
                            }`}>
                                {t(`stt.fileCard.status.${f.status}`)}
                            </span>
                        </button>
                    );
                })}
            </div>
        </div>
    );
};

export default SttFileSidebar;















// ═══════════════════════════════════════════════
// pages/SttTranscription/components/InferencePanel.tsx
// ═══════════════════════════════════════════════
import React, { useEffect, useRef, useState } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppDispatch, useAppSelector } from '../../../store/hooks';
import {
    submitBatchInference,
    fetchInferenceProgress,
    fetchInferenceProgressList,
    closeInference as _unused,
    fetchSttFiles,
    clearSttFiles,
    fetchSttModels,
    clearInferenceError,
} from '../../../store/sttSlice';
import styles from '../SttTranscription.module.scss';

interface Props {
    selectedIds: Set<number>;
    onSubmitted: () => void;
    onCompleted: () => void; // called when inference finishes — clears selection, no panel to close
}

const IconX: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
        <path d="M4 4l8 8M12 4l-8 8" />
    </svg>
);
const IconSpark: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <path d="M8 1.5l1.2 3 3 1.2-3 1.2L8 10l-1.2-3.1-3-1.2 3-1.2z" />
        <path d="M12.5 10l.7 1.8 1.8.7-1.8.7-.7 1.8-.7-1.8-1.8-.7 1.8-.7z" />
    </svg>
);

const CHUNK_OPTIONS = [
    { value: 0,  label: 'Auto' },
    { value: 5,  label: '5s'   },
    { value: 10, label: '10s'  },
    { value: 15, label: '15s'  },
    { value: 20, label: '20s'  },
];
const POLL_INTERVAL = 10_000;

const InferencePanel: React.FC<Props> = ({ selectedIds, onSubmitted, onCompleted }) => {
    const dispatch = useAppDispatch();
    const { t } = useTranslation();

    const models        = useAppSelector(s => s.stt.models);
    const modelsLoading = useAppSelector(s => s.stt.modelsLoading);
    const dateFrom      = useAppSelector(s => s.stt.dateFrom);
    const dateTo        = useAppSelector(s => s.stt.dateTo);
    const running       = useAppSelector(s => s.stt.inferenceRunning);
    const submitting    = useAppSelector(s => s.stt.inferenceSubmitting);
    const error         = useAppSelector(s => s.stt.inferenceError);
    const queued        = useAppSelector(s => s.stt.inferenceQueued);
    const runningFiles  = useAppSelector(s => s.stt.inferenceRunningFiles);
    const completed     = useAppSelector(s => s.stt.inferenceCompleted);
    const pending       = useAppSelector(s => s.stt.inferencePending);
    const progressMap   = useAppSelector(s => s.stt.inferenceProgressMap);

    const [model, setModel]   = useState('');
    const [language, setLang] = useState('ko');
    const [chunk, setChunk]   = useState(0);

    const pollRef      = useRef<ReturnType<typeof setInterval> | null>(null);
    const countRef     = useRef<ReturnType<typeof setInterval> | null>(null);
    const [countdown, setCountdown] = useState(0);

    useEffect(() => {
        if (models.length === 0) dispatch(fetchSttModels());
    }, [dispatch, models.length]);

    useEffect(() => {
        if (!model && models.length > 0) setModel(models[0]);
    }, [models, model]);

    // Poll /by-progress while running.
    // Stop when queued:[] AND running:[] — inference complete.
    // Then call /by-date once to refresh the library and close col-3.
    useEffect(() => {
        const tick = async () => {
            const result = await dispatch(fetchInferenceProgress({ from: dateFrom, to: dateTo }));
            if (!fetchInferenceProgress.fulfilled.match(result)) return;

            const { queued, running: runList } = result.payload;

            // Poll per-file progress for currently running files
            if (runList.length > 0) {
                dispatch(fetchInferenceProgressList(runList.map((f: any) => f.id)));
            }

            // Both empty → inference finished
            if (queued.length === 0 && runList.length === 0) {
                if (pollRef.current) { clearInterval(pollRef.current); pollRef.current = null; }
                if (countRef.current) { clearInterval(countRef.current); countRef.current = null; }
                dispatch(clearSttFiles());
                await dispatch(fetchSttFiles({ from: dateFrom, to: dateTo }));
                onCompleted();
            }
        };

        if (running) {
            setCountdown(POLL_INTERVAL / 1000);
            tick();
            pollRef.current = setInterval(() => { setCountdown(POLL_INTERVAL / 1000); tick(); }, POLL_INTERVAL);
            countRef.current = setInterval(() => setCountdown(prev => Math.max(0, prev - 1)), 1000);
        } else {
            if (pollRef.current)  { clearInterval(pollRef.current);  pollRef.current  = null; }
            if (countRef.current) { clearInterval(countRef.current); countRef.current = null; }
            setCountdown(0);
        }
        return () => {
            if (pollRef.current)  { clearInterval(pollRef.current);  pollRef.current  = null; }
            if (countRef.current) { clearInterval(countRef.current); countRef.current = null; }
        };
    }, [running, dispatch, dateFrom, dateTo]);

    const handleSubmit = async () => {
        if (selectedIds.size === 0 || !model || submitting) return;
        dispatch(clearInferenceError());
        const result = await dispatch(submitBatchInference({
            fileIds: Array.from(selectedIds),
            modelName: model,
            language,
            chunkSize: chunk,
        }));
        if (submitBatchInference.fulfilled.match(result)) {
            onSubmitted(); // clear selection in parent
        } else {
            // Non-200 from /batch_process — fire /by-progress once (no await, fire-and-forget)
            dispatch(fetchInferenceProgress({ from: dateFrom, to: dateTo }));
        }
    };

    const progressOf = (id: number) => progressMap[id] ?? 0;
    const statusLabel = (id: number) => {
        const p = progressOf(id);
        if (p === 100) return 'Done';
        if (p === -1)  return 'Failed';
        if (p > 0)     return `${p}%`;
        return 'Starting…';
    };

    return (
        <div className={styles.inferencePanel}>
            {/* Heading — countdown badge only; no close button since this is a persistent tab, not a col-3 panel */}
            {running && countdown > 0 && (
                <div className={styles.colHeading}>
                    <span className={styles.colHeadingIc}><IconSpark /></span>
                    <span className={styles.colHeadingText}>{t('stt.inference.title')}</span>
                    <span className={styles.infCountdownBadge}>⟳ next check in {countdown}s</span>
                </div>
            )}

            {/* CONFIG VIEW */}
            {!running && (
                <div className={styles.infColBody}>
                    {/* Selected files count */}
                    <div className={styles.infSelectedHint} data-tour="infer-settings">
                        {selectedIds.size === 0
                            ? 'Select files from the sidebar to run inference on.'
                            : `${selectedIds.size} file${selectedIds.size === 1 ? '' : 's'} selected`}
                    </div>

                    <div className={styles.infSection} data-tour="infer-model">
                        <div className={styles.infSectionTitle}>Settings</div>

                        <div className={styles.infField}>
                            <label className={styles.infLabel}>Model</label>
                            {modelsLoading
                                ? <div className={styles.infHint}>Loading…</div>
                                : <select className={styles.infSelect} value={model} onChange={e => setModel(e.target.value)}>
                                    {models.map(m => <option key={m} value={m}>{m}</option>)}
                                  </select>
                            }
                        </div>

                        <div className={styles.infField}>
                            <label className={styles.infLabel}>Language</label>
                            <select className={styles.infSelect} value={language} onChange={e => setLang(e.target.value)}>
                                <option value="ko">Korean</option>
                                <option value="en">English</option>
                            </select>
                        </div>

                        <div className={styles.infField}>
                            <label className={styles.infLabel}>Chunk size</label>
                            <select className={styles.infSelect} value={chunk} onChange={e => setChunk(Number(e.target.value))}>
                                {CHUNK_OPTIONS.map(c => <option key={c.value} value={c.value}>{c.label}</option>)}
                            </select>
                        </div>
                    </div>

                    {error && <div className={styles.infError}>{error}</div>}

                    <div className={styles.infActions} data-tour="infer-run">
                        <button
                            className={`${styles.btn} ${styles.btnBlue}`}
                            disabled={selectedIds.size === 0 || !model || submitting}
                            onClick={handleSubmit}
                        >
                            {submitting ? 'Submitting…' : `Run Inference${selectedIds.size > 0 ? ` (${selectedIds.size})` : ''}`}
                        </button>
                    </div>
                </div>
            )}

            {/* PROGRESS VIEW */}
            {running && (
                <div className={styles.infColBody}>
                    <div className={styles.infProgressGrid}>

                        {/* ── Queued group ── */}
                        <div className={`${styles.infProgressCol} ${styles.infColPending}`}>
                            <div className={styles.infProgressColHead}>
                                <div className={styles.infProgressColIcon}>
                                    <svg viewBox="0 0 16 16" fill="none" strokeWidth="1.5" strokeLinecap="round" stroke="var(--t2)">
                                        <circle cx="8" cy="8" r="5.5" /><path d="M8 5v3.5l2 1.5" />
                                    </svg>
                                </div>
                                <span>Queued</span>
                                <span className={styles.infProgressCount}>{queued.length} file{queued.length !== 1 ? 's' : ''}</span>
                                <span className={`${styles.infProgressBadge} ${styles.infBadgePending}`}>Waiting</span>
                            </div>
                            <div className={styles.infProgressBody}>
                                {queued.length === 0
                                    ? <div className={styles.infProgressEmpty}>No queued files</div>
                                    : queued.map(f => (
                                        <div key={f.id} className={styles.infProgressCard}>
                                            <div className={styles.infProgressIcon}>STT</div>
                                            <div className={styles.infProgressName} title={f.original_name}>{f.original_name}</div>
                                            <span className={`${styles.infProgressBadge} ${styles.infBadgePending}`} style={{ marginLeft: 'auto' }}>In queue</span>
                                        </div>
                                    ))
                                }
                            </div>
                        </div>

                        {/* ── Running group ── */}
                        <div className={`${styles.infProgressCol} ${styles.infColRunning}`}>
                            <div className={styles.infProgressColHead}>
                                <div className={styles.infProgressColIcon}>
                                    <svg viewBox="0 0 16 16" fill="none" strokeWidth="1.5" strokeLinecap="round" stroke="var(--amber)">
                                        <circle cx="8" cy="8" r="6" /><path d="M8 5v4M8 11v.1" />
                                    </svg>
                                </div>
                                <span>Running</span>
                                <span className={styles.infProgressCount}>{runningFiles.length} file{runningFiles.length !== 1 ? 's' : ''}</span>
                                <span className={`${styles.infProgressBadge} ${styles.infBadgeRunning}`}>
                                    <span className={styles.infBadgeDot} />Active
                                </span>
                            </div>
                            <div className={styles.infProgressBody}>
                                {runningFiles.length === 0
                                    ? <div className={styles.infProgressEmpty}>No active files</div>
                                    : runningFiles.map(f => {
                                        const pct = progressOf(f.id);
                                        return (
                                            <div key={f.id} className={`${styles.infProgressCard} ${styles.infProgressCardRunning}`}>
                                                <div className={styles.infProgressIcon}>STT</div>
                                                <div className={styles.infProgressName} title={f.original_name}>{f.original_name}</div>
                                                <div className={styles.infProgressBar}>
                                                    <div className={styles.infProgressBarTrack}>
                                                        <div className={styles.infProgressFill} style={{ width: `${Math.max(3, pct)}%` }} />
                                                    </div>
                                                </div>
                                                <div className={styles.infProgressPct}>{statusLabel(f.id)}</div>
                                            </div>
                                        );
                                    })
                                }
                            </div>
                        </div>

                        {/* ── Done group — mirrors .gDone ── */}
                        <div className={`${styles.infProgressCol} ${styles.infColDone}`}>
                            <div className={styles.infProgressColHead}>
                                <div className={styles.infProgressColIcon}>
                                    <svg viewBox="0 0 16 16" fill="none" strokeWidth="1.5" strokeLinecap="round" stroke="var(--green)">
                                        <circle cx="8" cy="8" r="5.5" /><path d="M5.5 8l2 2 3-3" />
                                    </svg>
                                </div>
                                <span>Done</span>
                                <span className={styles.infProgressCount}>{completed.length} file{completed.length !== 1 ? 's' : ''}</span>
                                <span className={`${styles.infProgressBadge} ${styles.infBadgeDone}`}>
                                    <span className={styles.infBadgeDot} />
                                    Done
                                </span>
                            </div>
                            <div className={styles.infProgressBody}>
                                {completed.length === 0
                                    ? <div className={styles.infProgressEmpty}>No completed files yet</div>
                                    : completed.map(f => (
                                        <div key={f.id} className={`${styles.infProgressCard} ${styles.infProgressCardDone}`}>
                                            <div className={styles.infProgressIcon}>STT</div>
                                            <div className={styles.infProgressName} title={f.original_name}>{f.original_name}</div>
                                            <span className={`${styles.infProgressBadge} ${styles.infBadgeDone}`} style={{ marginLeft: 'auto' }}>
                                                <span className={styles.infBadgeDot} />Done
                                            </span>
                                        </div>
                                    ))
                                }
                            </div>
                        </div>

                    </div>
                </div>
            )}
        </div>
    );
};

export default InferencePanel;




















// ═══════════════════════════════════════════════
// SttTranscription.module.scss
// Content Analytics · STT Transcription page
//
// Three views share this stylesheet:
//   1. Library  — file grid, date filter, upload panel (new)
//   2. Detail   — player + transcript editor (unchanged from original)
//   3. Toast    — shared between views
// ═══════════════════════════════════════════════
@use '../../styles/mixins' as m;

// ══════════════════════════════════════
// SHARED SHELL
// ══════════════════════════════════════
.page {
  display: flex;
  flex-direction: column;
  height: 100%;
  overflow: hidden;
}

// Library variant — same layout, but body scrolls instead of flex children
.libPage {
  display: flex;
  flex-direction: column;
  height: 100%;
  overflow: hidden;
}

// ── Page header (shared by library + detail) ─────
.ph {
  padding: 14px 22px 12px;
  background: var(--bg1);
  border-bottom: none;
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

.phRow {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 14px;
}

.phTitleWrap {
  display: flex;
  align-items: center;
  gap: 12px;
  min-width: 0;
}

.phTitle {
  font-size: 18px;
  font-weight: 600;
  color: var(--t0);
  letter-spacing: -0.3px;
  line-height: 1.2;
  font-family: var(--font-display);
  @include m.truncate;
}

.phSub {
  font-size: 11px;
  color: var(--t2);
  margin-top: 3px;
  @include m.mono;
}

// Tour trigger button — pill style, appears next to page title
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
  flex-shrink: 0;

  svg { width: 13px; height: 13px; }

  &:hover {
    background: var(--bg3);
    color: var(--t1);
    border-color: var(--bdr3);
  }
}

.phActs {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-shrink: 0;
  flex-wrap: wrap;
  justify-content: flex-end;
}

.backBtn {
  width: 32px;
  height: 32px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t1);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.12s;

  svg {
    width: 14px;
    height: 14px;
  }

  &:hover {
    background: var(--bg3);
    color: var(--t0);
    border-color: var(--bdr3);
  }
}

// ══════════════════════════════════════
// BUTTONS (preserved from original)
// ══════════════════════════════════════
.btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  padding: 7px 16px;
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
  user-select: none;
  text-decoration: none;

  svg {
    width: 13px;
    height: 13px;
  }

  &:hover {
    background: var(--bg3);
    color: var(--t0);
    border-color: var(--bdr3);
  }

  &:disabled {
    opacity: 0.4;
    cursor: not-allowed;
    pointer-events: none;
  }
}

.btnBlue {
  background: var(--blue);
  color: #fff;
  border-color: var(--blue);
  font-weight: 600;
  box-shadow: 0 2px 12px var(--blue-dim);

  &:hover {
    background: #a78bfa;
    border-color: #a78bfa;
    color: #fff;
  }
}

// ══════════════════════════════════════
// LIBRARY VIEW
// ══════════════════════════════════════
.libBody {
  flex: 1;
  overflow: hidden;
  display: grid;
  // Default: col1=400px, col2=remaining, col3=hidden
  grid-template-columns: 400px 1fr;
  gap: 0;
  min-height: 0;
  background: var(--bg0);

  @media (max-width: 1500px) {
    grid-template-columns: 350px 1fr;
  }
}

// When inference is open: col1=400px, col3=400px, col2 takes remaining
.libBodyInference {
  grid-template-columns: 400px 1fr 400px;

  @media (max-width: 1500px) {
    grid-template-columns: 350px 1fr 350px;
  }
}

// Card selected for inference
.libCardSelected {
  border-color: rgba(139, 92, 246, 0.5) !important;
  background: rgba(139, 92, 246, 0.07) !important;
  box-shadow: 0 0 0 1px rgba(139, 92, 246, 0.25);
}

// When inference panel is open, all cards become selectable
.libCardSelectable {
  cursor: pointer;

  &:hover {
    border-color: rgba(139, 92, 246, 0.35);
    background: var(--bg2);
  }
}

// Checkbox overlay in top-left corner of card when in selection mode
.libCardCheckbox {
  position: absolute;
  top: 8px;
  right: 8px;
  width: 16px;
  height: 16px;
  border-radius: 4px;
  border: 1.5px solid var(--bdr3);
  background: var(--bg1);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 6;
  transition: all 0.12s;

  svg { width: 9px; height: 9px; display: none; }
}

.libCardCheckboxChecked {
  background: #a78bfa;
  border-color: #a78bfa;
  color: #fff;

  svg { display: block; }
}

// Hint in inference panel when no files selected
.infSelectedHint {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  padding: 7px 12px;
  background: rgba(139, 92, 246, 0.06);
  border-bottom: 1px solid rgba(139, 92, 246, 0.2);
  font-size: 11px;
  color: #a78bfa;
  @include m.mono;
  font-weight: 500;
  flex-shrink: 0;
}
.dateFilterDisabled {
  opacity: 0.5;
  pointer-events: none;
}

// ── Select All / Clear row — sticky, outside scroll area ─────────
.selectAllRow {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  padding: 7px 12px;
  background: rgba(139, 92, 246, 0.06);
  border-bottom: 1px solid rgba(139, 92, 246, 0.2);
  flex-shrink: 0;
  animation: fadeSlide 0.15s ease;
}

.selectAllHint {
  font-size: 11px;
  color: #a78bfa;
  @include m.mono;
  font-weight: 500;
}

.selectAllActions {
  display: flex;
  align-items: center;
  gap: 6px;
}

.selectAllBtn {
  font-size: 11px;
  font-weight: 500;
  padding: 3px 9px;
  border-radius: 99px;
  border: 1px solid rgba(139,92,246,0.3);
  background: transparent;
  color: #a78bfa;
  cursor: pointer;
  font-family: var(--font-ui);
  transition: all 0.12s;

  &:hover:not(:disabled) { background: rgba(139,92,246,0.12); border-color: rgba(139,92,246,0.5); }
  &:disabled { opacity: 0.35; cursor: not-allowed; }
}

// Dimmed — not selectable in current mode
.libCardDimmed {
  opacity: 0.4;
  pointer-events: none;
  filter: grayscale(0.4);
}

// Generic action icon button (delete, pipeline)
.actionIconBtn {
  width: 30px;
  height: 30px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.14s;
  flex-shrink: 0;

  svg { width: 13px; height: 13px; }
  &:hover { background: var(--bg2); color: var(--t0); border-color: var(--bdr3); }
}

.actionIconBtnDelete {
  border-color: rgba(239,68,68,0.4);
  background: rgba(239,68,68,0.08);
  color: var(--red);
  &:hover { background: rgba(239,68,68,0.16); border-color: rgba(239,68,68,0.6); }
}

.actionIconBtnPipeline {
  border-color: rgba(56,196,186,0.4);
  background: rgba(56,196,186,0.08);
  color: #38c4ba;
  &:hover { background: rgba(56,196,186,0.16); border-color: rgba(56,196,186,0.6); }
}

.selectAllRowDelete {
  background: rgba(239,68,68,0.06);
  border-bottom-color: rgba(239,68,68,0.2);
  .selectAllHint { color: var(--red); }
  .selectAllBtn { border-color: rgba(239,68,68,0.3); color: var(--red); &:hover:not(:disabled) { background: rgba(239,68,68,0.1); } }
}

.selectAllRowPipeline {
  background: rgba(56,196,186,0.06);
  border-bottom-color: rgba(56,196,186,0.2);
  .selectAllHint { color: #38c4ba; }
  .selectAllBtn { border-color: rgba(56,196,186,0.3); color: #38c4ba; &:hover:not(:disabled) { background: rgba(56,196,186,0.1); } }
}

.selectAllLeft {
  display: flex;
  align-items: center;
  gap: 8px;
  flex: 1;
  min-width: 0;
}

.selectAllRight {
  display: flex;
  align-items: center;
  gap: 6px;
  flex-shrink: 0;
}

.selectAllError {
  font-size: 11px;
  color: var(--red);
  @include m.mono;
}

.btnRed {
  background: rgba(239,68,68,0.1);
  color: var(--red);
  border: 1px solid rgba(239,68,68,0.3);
  &:hover:not(:disabled) { background: rgba(239,68,68,0.18); }
  &:disabled { opacity: 0.4; cursor: not-allowed; }
}

.inferenceIconBtn {
  width: 30px;
  height: 30px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  position: relative;
  transition: all 0.14s;
  flex-shrink: 0;

  svg { width: 13px; height: 13px; }

  &:hover:not(:disabled) {
    background: var(--bg2);
    color: var(--t0);
    border-color: var(--bdr3);
  }

  &:disabled { cursor: not-allowed; opacity: 0.45; }
}

.inferenceIconBtnActive {
  background: rgba(139, 92, 246, 0.18);
  color: #c4b5fd;
  border-color: rgba(139, 92, 246, 0.55);
  box-shadow: 0 0 0 1px rgba(139, 92, 246, 0.2);
}

.inferenceIconBtnRunning {
  background: rgba(59, 130, 246, 0.1);
  color: var(--blue);
  border-color: rgba(59, 130, 246, 0.25);
}

// Running pulsing dot badge on the inference icon
.inferenceIconBtnDot {
  position: absolute;
  top: 3px;
  right: 3px;
  width: 5px;
  height: 5px;
  border-radius: 50%;
  background: var(--blue);
  animation: breathe 1.4s ease-in-out infinite;
}

// ══════════════════════════════════════
// INFERENCE PANEL (3rd column)
// ══════════════════════════════════════
.inferencePanel {
  display: flex;
  flex-direction: column;
  overflow: hidden;
  position: relative;

  // Left gradient border separator
  &::before {
    content: '';
    position: absolute;
    top: 0; left: 0; bottom: 0;
    width: 1px;
    background: linear-gradient(180deg,
      rgba(139,92,246,0) 0%, rgba(139,92,246,0.55) 20%,
      rgba(56,196,186,0.65) 50%, rgba(240,160,48,0.55) 80%,
      rgba(240,160,48,0) 100%);
    pointer-events: none;
    z-index: 1;
  }
}

.inferencePanelClose {
  width: 24px;
  height: 24px;
  border-radius: var(--r);
  border: 1px solid transparent;
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.12s;
  margin-left: auto;

  svg { width: 12px; height: 12px; }
  &:hover { background: var(--bg3); color: var(--t0); border-color: var(--bdr2); }
}

.infColBody {
  flex: 1;
  overflow-y: auto;
  padding: 14px 18px 24px;
  display: flex;
  flex-direction: column;
  gap: 16px;
  min-height: 0;
  @include m.scrollbar;
}

.infSection {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.infSectionTitle {
  font-size: 10px;
  font-weight: 600;
  color: var(--t2);
  text-transform: uppercase;
  letter-spacing: 0.08em;
  @include m.mono;
}

.infFileList {
  display: flex;
  flex-direction: column;
  gap: 2px;
  border: 1px solid var(--bdr);
  border-radius: var(--rl);
  overflow: hidden;
  max-height: 280px;
  overflow-y: auto;
  @include m.scrollbar;
}

.infEmpty {
  font-size: 12px;
  color: var(--t2);
  padding: 12px;
  text-align: center;
}

.infFileRow {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 9px 12px;
  background: var(--bg1);
  border: none;
  border-bottom: 1px solid var(--bdr);
  cursor: pointer;
  transition: background 0.1s;
  text-align: left;
  width: 100%;

  &:last-child { border-bottom: none; }
  &:hover { background: var(--bg2); }
}

.infFileRowSelected {
  background: rgba(139,92,246,0.07);
  &:hover { background: rgba(139,92,246,0.12); }
}

.infCheckbox {
  width: 16px;
  height: 16px;
  border-radius: 4px;
  border: 1.5px solid var(--bdr3);
  background: transparent;
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.12s;

  svg { width: 10px; height: 10px; }
}

.infCheckboxChecked {
  background: #a78bfa;
  border-color: #a78bfa;
  color: #fff;
}

.infFileName {
  flex: 1;
  font-size: 12px;
  font-weight: 500;
  color: var(--t0);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.infFilePill {
  font-size: 10px;
  padding: 2px 7px;
  border-radius: 99px;
  font-weight: 500;
  @include m.mono;
  flex-shrink: 0;
}

.infFilePillGreen {
  background: rgba(34,197,94,0.1);
  color: var(--green);
  border: 1px solid rgba(34,197,94,0.2);
}

.infFilePillGray {
  background: var(--bg3);
  color: var(--t2);
  border: 1px solid var(--bdr2);
}

.infField {
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.infLabel {
  font-size: 10px;
  font-weight: 600;
  color: var(--t2);
  text-transform: uppercase;
  letter-spacing: 0.06em;
  @include m.mono;
}

.infSelect {
  width: 100%;
  padding: 7px 10px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: var(--bg2);
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 12px;
  outline: none;
  cursor: pointer;
  transition: border-color 0.12s;

  &:focus, &:hover { border-color: var(--bdr3); }
}

.infHint {
  font-size: 11px;
  color: var(--t2);
  @include m.mono;
}

.infError {
  font-size: 11px;
  color: var(--red);
  @include m.mono;
  padding: 8px 10px;
  background: var(--red-dim);
  border: 1px solid var(--red-bdr);
  border-radius: var(--r);
}

.infActions {
  padding-top: 4px;
}

// ── Progress view (3 columns) ──
.infProgressGrid {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

// ── Column (status group) — mirrors .bgroup ──────
.infProgressCol {
  background: var(--bg1);
  border: 1px solid var(--bdr);
  border-radius: var(--rl);
  overflow: hidden;
  display: flex;
  flex-direction: column;

  // Running group — amber border like .gRun
  &.infColRunning { border-color: var(--amber-bdr); }
  // Done group — green border like .gDone
  &.infColDone    { border-color: var(--green-bdr); }
  // Pending group — subtle border
  &.infColPending { border-color: var(--bdr2); }
}

// ── Column header — mirrors .bgroupHeader ─────────
.infProgressColHead {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 9px 14px;
  font-size: 13px;
  font-weight: 500;
  color: var(--t0);
  border-bottom: 1px solid var(--bdr);
}

// Icon badge — mirrors .bgroupIcon
.infProgressColIcon {
  width: 26px;
  height: 26px;
  border-radius: 7px;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;

  svg { width: 13px; height: 13px; }

  .infColRunning & { background: var(--amber-dim); }
  .infColDone    & { background: var(--green-dim); }
  .infColPending & { background: var(--bg3); }
}

.infProgressDot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  flex-shrink: 0;
}

.infProgressCount {
  font-size: 12px;
  color: var(--t2);
  @include m.mono;
  margin-left: 4px;
}

// Status badge — mirrors .badge .bRun / .bDone / .bQ
.infProgressBadge {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-size: 11px;
  padding: 2px 8px;
  border-radius: 99px;
  font-weight: 500;
  border: 1px solid transparent;
  @include m.mono;
  margin-left: auto;

  // Breathing dot inside badge
  .infBadgeDot {
    width: 5px;
    height: 5px;
    border-radius: 50%;
    flex-shrink: 0;
  }
}

.infBadgeRunning {
  background: var(--amber-dim);
  color: var(--amber);
  border-color: var(--amber-bdr);
  .infBadgeDot { background: var(--amber); animation: breathe 1.2s infinite; }
}

.infBadgeDone {
  background: var(--green-dim);
  color: var(--green);
  border-color: var(--green-bdr);
  .infBadgeDot { background: var(--green); }
}

.infBadgePending {
  background: var(--bg3);
  color: var(--t2);
  border-color: var(--bdr);
}

// ── Column body (file rows) — mirrors .bgroupBody ─
.infProgressBody {
  padding: 8px 14px 10px;
  display: flex;
  flex-direction: column;
  gap: 5px;
}

// ── Individual file row — mirrors .brow ───────────
.infProgressCard {
  display: flex;
  align-items: center;
  gap: 9px;
  padding: 7px 10px;
  background: var(--bg2);
  border: 1px solid var(--bdr);
  border-radius: var(--r);
  position: relative;
  overflow: hidden;

  // Raise children above the sweep mask
  > * { position: relative; z-index: 5; }
}

.infProgressCardDone {
  border-color: rgba(34,197,94,0.15);
  background: rgba(34,197,94,0.04);
}

// Running card — amber sweep line around the border
.infProgressCardRunning {
  border-color: var(--amber-bdr);
  background: color-mix(in srgb, var(--bg2) 96%, var(--amber) 4%);

  // Sweep: large amber conic-gradient square centred on the card
  &::before {
    content: '';
    position: absolute;
    width: 300px;
    height: 300px;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%) rotate(0deg);
    background: conic-gradient(
      from 0deg at 50% 50%,
      transparent    0deg,
      transparent  310deg,
      rgba(245, 158, 11, 0.25) 315deg,
      #f59e0b       325deg,
      rgba(245, 158, 11, 0.25) 335deg,
      transparent  338deg,
      transparent  360deg
    );
    animation: infSweep 2s linear infinite;
    z-index: 3;
    pointer-events: none;
  }

  // Mask that leaves only the border strip visible
  &::after {
    content: '';
    position: absolute;
    inset: 2px;
    border-radius: calc(var(--r) - 2px);
    background: color-mix(in srgb, var(--bg2) 96%, var(--amber) 4%);
    z-index: 4;
    pointer-events: none;
  }
}

@keyframes infSweep {
  from { transform: translate(-50%, -50%) rotate(0deg); }
  to   { transform: translate(-50%, -50%) rotate(360deg); }
}

// File type icon badge — mirrors .browIcon
.infProgressIcon {
  width: 22px;
  height: 22px;
  border-radius: 5px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 9px;
  font-weight: 700;
  @include m.mono;
  flex-shrink: 0;

  .infColRunning & { background: var(--amber-dim); color: var(--amber); }
  .infColDone    & { background: var(--green-dim); color: var(--green); }
  .infColPending & { background: var(--bg3);       color: var(--t2); }
}

.infProgressName {
  font-size: 12px;
  font-weight: 500;
  color: var(--t0);
  flex: 1;
  min-width: 0;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.infProgressMeta {
  font-size: 11px;
  color: var(--t2);
  @include m.mono;
  flex-shrink: 0;
}

// Progress bar — mirrors .browBar / .browFill (amber for running)
.infProgressBar {
  width: 60px;
  flex-shrink: 0;
}

.infProgressBarTrack {
  height: 3px;
  background: var(--bg3);
  border-radius: 99px;
  overflow: hidden;
}

.infProgressFill {
  height: 100%;
  border-radius: 99px;
  background: var(--amber);          // amber fill like .browFill
  transition: width 0.4s ease;
}

.infProgressPct {
  font-size: 11px;
  color: var(--amber);
  @include m.mono;
  min-width: 26px;
  text-align: right;
  flex-shrink: 0;
}

.infProgressEmpty {
  font-size: 12px;
  color: var(--t2);
  padding: 6px 2px;
  opacity: 0.6;
  @include m.mono;
}

.infPollingHint {
  display: none; // moved to header
}

.infCountdownBadge {
  margin-left: auto;
  font-size: 10px;
  font-weight: 500;
  color: var(--blue);
  @include m.mono;
  letter-spacing: 0.04em;
  white-space: nowrap;
  background: var(--blue-dim);
  border: 1px solid var(--blue-bdr);
  padding: 2px 7px;
  border-radius: 99px;
}





// Left column — upload zone
.uploadCol {
  display: flex;
  flex-direction: column;
  overflow: hidden;
  position: relative;

  // Gradient column separator
  &::after {
    content: '';
    position: absolute;
    top: 0;
    right: 0;
    bottom: 0;
    width: 1px;
    background: linear-gradient(180deg,
      rgba(139, 92, 246, 0.0) 0%,
      rgba(139, 92, 246, 0.5) 20%,
      rgba(56, 196, 186, 0.6) 50%,
      rgba(240, 160, 48, 0.5) 80%,
      rgba(240, 160, 48, 0.0) 100%);
    pointer-events: none;
  }
}

// Right column — library
.libCol {
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

// ── Column heading — fixed, never scrolls ──
.colHeading {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 0 22px;
  height: 52px;
  flex-shrink: 0;
  position: relative;
  background: var(--bg1);

  // Gradient underline
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

// ── Column scroll body — scrollable content below the heading ──
.colBody {
  flex: 1;
  overflow-y: auto;
  padding: 16px 22px 28px;
  display: flex;
  flex-direction: column;
  gap: 14px;
  min-height: 0;
  @include m.scrollbar;
}

.colHeadingIc {
  width: 24px;
  height: 24px;
  border-radius: 6px;
  background: var(--bg2);
  border: 1px solid var(--bdr2);
  display: flex;
  align-items: center;
  justify-content: center;
  color: var(--t2);
  flex-shrink: 0;

  svg { width: 12px; height: 12px; }
}

.colHeadingText {
  font-size: 12px;
  font-weight: 600;
  color: var(--t1);
  font-family: var(--font-display);
  letter-spacing: 0.02em;
  flex-shrink: 0;
  white-space: nowrap;
}

.uploadLimitBadge {
  font-size: 10px;
  font-weight: 500;
  color: var(--amber);
  background: var(--amber-dim);
  border: 1px solid var(--amber-bdr);
  border-radius: 99px;
  padding: 2px 8px;
  @include m.mono;
  white-space: nowrap;
}

// ── Date filter ─────────────────────────────────
.dateFilter {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 3px 4px 3px 8px;
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  background: var(--bg1);
  height: 28px;
}

.dateFilterIc {
  color: var(--t2);
  display: flex;
  align-items: center;
  flex-shrink: 0;

  svg { width: 11px; height: 11px; }
}

.dateInput {
  background: transparent;
  border: none;
  outline: none;
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 10px;
  padding: 1px 2px;
  font-variant-numeric: tabular-nums;
  cursor: pointer;
  color-scheme: dark;
  width: 90px;

  &::-webkit-calendar-picker-indicator {
    cursor: pointer;
    opacity: 0.4;
    filter: invert(0.5);
    width: 10px;
    height: 10px;
    transition: opacity 0.12s;
  }

  &:hover::-webkit-calendar-picker-indicator { opacity: 0.8; }
}

.dateSep {
  color: var(--t2);
  font-size: 10px;
  user-select: none;
  flex-shrink: 0;
}

.dateReset {
  background: transparent;
  border: none;
  color: var(--t2);
  font-size: 10px;
  padding: 3px 6px;
  border-radius: 4px;
  cursor: pointer;
  font-family: var(--font-ui);
  transition: all 0.12s;
  white-space: nowrap;

  &:hover { color: var(--t0); background: var(--bg3); }
}

.dateValidationError {
  font-size: 11px;
  color: var(--red);
  white-space: nowrap;
  @include m.mono;
}

// Light theme date picker
:global(html.light) {
  .dateInput {
    color-scheme: light;
  }
  .dateInput::-webkit-calendar-picker-indicator { filter: none; }
}

// ── Status filter chips (Library column heading) ──
.colHeadingRight {
  display: flex;
  align-items: center;
  gap: 6px;
  min-width: 0;
  flex: 1;
  justify-content: flex-end;
  flex-wrap: nowrap;
  overflow: hidden;
}

.statusFilterChips {
  display: flex;
  align-items: center;
  gap: 3px;
  flex-shrink: 0;
}

// Collapsed filter icon — shown instead of chips when search is open in split view
.filterIconBtn {
  width: 28px;
  height: 28px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  position: relative;
  flex-shrink: 0;
  transition: all 0.12s;

  svg { width: 13px; height: 13px; }

  &:hover { background: var(--bg2); color: var(--t1); border-color: var(--bdr3); }
}

.filterIconBtnActive {
  background: rgba(139,92,246,0.1);
  color: #a78bfa;
  border-color: rgba(139,92,246,0.3);
}

// Dot badge when a non-'all' filter is active
.filterIconDot {
  position: absolute;
  top: 4px;
  right: 4px;
  width: 5px;
  height: 5px;
  border-radius: 50%;
  background: #a78bfa;
}

// ── Action toolbar — icon + text buttons, own row below colHeading ──
// Lighter background than colHeading so it reads as a distinct toolbar strip.
.actionToolbar {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 8px;
  padding: 8px 22px;
  background: var(--bg2);
  border-bottom: 1px solid var(--bdr);
  flex-shrink: 0;
}

.actionToolbarBtn {
  display: inline-flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 4px;
  min-width: 84px;
  padding: 8px 10px;
  border-radius: var(--r);
  border: 1px solid transparent;
  cursor: pointer;
  font-size: 11px;
  font-weight: 600;
  font-family: var(--font-ui);
  white-space: nowrap;
  text-align: center;
  transition: all 0.14s;
  flex-shrink: 0;

  svg { width: 16px; height: 16px; flex-shrink: 0; }
  span { display: inline; line-height: 1.2; }

  &:active { filter: brightness(0.92); }
}

.actionToolbarBtnInference {
  background: color-mix(in srgb, var(--violet) 16%, var(--bg2));
  border-color: color-mix(in srgb, var(--violet) 45%, var(--bg2));
  color: var(--violet);

  &:hover { background: color-mix(in srgb, var(--violet) 24%, var(--bg2)); }
}

.actionToolbarBtnDelete {
  background: color-mix(in srgb, var(--red) 16%, var(--bg2));
  border-color: color-mix(in srgb, var(--red) 45%, var(--bg2));
  color: var(--red);

  &:hover { background: color-mix(in srgb, var(--red) 24%, var(--bg2)); }
}

.actionToolbarBtnPipeline {
  background: color-mix(in srgb, var(--teal) 16%, var(--bg2));
  border-color: color-mix(in srgb, var(--teal) 45%, var(--bg2));
  color: var(--teal);

  &:hover { background: color-mix(in srgb, var(--teal) 24%, var(--bg2)); }
}

.statusChip {
  font-size: 10px;
  font-weight: 500;
  padding: 3px 9px;
  border-radius: 99px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  font-family: var(--font-ui);
  transition: all 0.15s;
  white-space: nowrap;

  &:hover { background: var(--bg2); color: var(--t1); border-color: var(--bdr3); }
}

// "All" active — blue so it's immediately distinct
.statusChipActive {
  background: rgba(59, 130, 246, 0.12);
  color: var(--blue);
  border-color: rgba(59, 130, 246, 0.35);
  font-weight: 600;
}

// Per-status accent colours when active
.statusChip_completed.statusChipActive {
  background: rgba(34, 197, 94, 0.12);
  color: var(--green);
  border-color: rgba(34, 197, 94, 0.35);
  font-weight: 600;
}

.statusChip_pending.statusChipActive {
  background: rgba(139, 92, 246, 0.12);
  color: #a78bfa;
  border-color: rgba(139, 92, 246, 0.35);
  font-weight: 600;
}

.statusChip_failed.statusChipActive {
  background: rgba(239, 68, 68, 0.12);
  color: var(--red);
  border-color: rgba(239, 68, 68, 0.35);
  font-weight: 600;
}

.statusChip_queued.statusChipActive {
  background: rgba(245, 158, 11, 0.12);
  color: var(--amber);
  border-color: rgba(245, 158, 11, 0.35);
  font-weight: 600;
}

.statusChip_running.statusChipActive {
  background: rgba(59, 130, 246, 0.12);
  color: var(--blue);
  border-color: rgba(59, 130, 246, 0.35);
  font-weight: 600;
}



// ── Sections (Processing / Library) ─────────────
.section {
  margin-bottom: 28px;

  &:last-child {
    margin-bottom: 0;
  }
}

.sectionHeader {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 12px;
  padding: 0 2px;
}

.sectionTitle {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 12px;
  font-weight: 600;
  color: var(--t1);
  font-family: var(--font-display);
  letter-spacing: 0.04em;
  text-transform: uppercase;
}

.sectionDot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: var(--blue);
  box-shadow: 0 0 0 3px var(--blue-dim);
  animation: breathe 1.4s ease-in-out infinite;
}

.sectionCount {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 20px;
  height: 18px;
  padding: 0 6px;
  border-radius: 9px;
  background: var(--bg3);
  color: var(--t1);
  font-size: 10px;
  font-weight: 600;
  letter-spacing: 0;
  text-transform: none;
}

.sectionHint {
  font-size: 10px;
  color: var(--t2);
  @include m.mono;
  letter-spacing: 0.04em;
}

// ── Countdown pill ───────────────────────────────
// Lives in the Processing section header's right slot.
// Shows "Refreshes in Xs" with a thin arc that drains clockwise.
.countdownWrap {
  display: inline-flex;
  align-items: center;
  gap: 7px;
}

.countdownArc {
  position: relative;
  width: 16px;
  height: 16px;
  flex-shrink: 0;

  svg {
    width: 16px;
    height: 16px;
    transform: rotate(-90deg); // start arc at 12 o'clock
  }

  // Background ring
  .countdownTrack {
    fill: none;
    stroke: var(--bdr2);
    stroke-width: 2;
  }

  // Draining foreground arc
  .countdownFill {
    fill: none;
    stroke: var(--blue);
    stroke-width: 2;
    stroke-linecap: round;
    transition: stroke-dashoffset 0.25s linear;
  }
}

.countdownLabel {
  font-size: 10px;
  color: var(--t2);
  @include m.mono;
  letter-spacing: 0.04em;
  white-space: nowrap;
}

// ── Section header right slot (search) ──────────
.sectionHeaderRight {
  display: flex;
  align-items: center;
  gap: 8px;
}

.searchToggleBtn {
  width: 28px;
  height: 28px;
  border-radius: var(--r);
  border: 1px solid transparent;
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.12s;
  flex-shrink: 0;

  svg {
    width: 14px;
    height: 14px;
  }

  &:hover {
    background: var(--bg3);
    color: var(--t0);
    border-color: var(--bdr2);
  }
}

.searchToggleBtnActive {
  background: var(--blue-dim);
  color: var(--blue);
  border-color: var(--blue-bdr);

  &:hover {
    background: var(--blue-dim);
    color: var(--blue);
  }
}

.searchBox {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 4px 8px;
  border: 1px solid var(--blue-bdr);
  border-radius: var(--r);
  background: var(--bg1);
  height: 28px;
  animation: fadeSlide 0.14s ease;
  width: 140px;
  flex-shrink: 1;
  min-width: 0;
}

.searchBoxIc {
  color: var(--t2);
  flex-shrink: 0;
  display: flex;
  align-items: center;

  svg {
    width: 12px;
    height: 12px;
  }
}

.searchBoxInput {
  flex: 1;
  background: transparent;
  border: none;
  outline: none;
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 12px;
  min-width: 0;

  &::placeholder {
    color: var(--t2);
  }
}

.searchBoxClear {
  width: 18px;
  height: 18px;
  border-radius: 3px;
  border: none;
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  flex-shrink: 0;
  transition: all 0.12s;

  svg {
    width: 10px;
    height: 10px;
  }

  &:hover {
    color: var(--t0);
    background: var(--bg3);
  }
}


.libLoading {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 12px;
  padding: 60px 20px;
  color: var(--t2);
  font-size: 12px;
  font-family: var(--font-ui);
}

.libEmpty {
  padding: 40px 20px;
  text-align: center;
  border: 1px dashed var(--bdr2);
  border-radius: var(--rl);
  background: var(--bg1);
}

.libEmptyTitle {
  font-size: 13px;
  color: var(--t1);
  font-weight: 500;
  font-family: var(--font-display);
  margin-bottom: 4px;
}

.libEmptyHint {
  font-size: 11px;
  color: var(--t2);
}

// ══════════════════════════════════════
// FILE LIST — Option D: grouped by date
// ══════════════════════════════════════

// ── Date groups ──────────────────────────────────
.dateGroups {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.dateGroup {
  display: flex;
  flex-direction: column;
  gap: 0;
  animation: groupEntrance 0.32s ease both;
}

// Stagger entrance per group (up to 12 groups) via nth-child
.dateGroup:nth-child(1)  { animation-delay: 0ms; }
.dateGroup:nth-child(2)  { animation-delay: 40ms; }
.dateGroup:nth-child(3)  { animation-delay: 80ms; }
.dateGroup:nth-child(4)  { animation-delay: 120ms; }
.dateGroup:nth-child(5)  { animation-delay: 160ms; }
.dateGroup:nth-child(6)  { animation-delay: 200ms; }
.dateGroup:nth-child(7)  { animation-delay: 240ms; }
.dateGroup:nth-child(8)  { animation-delay: 280ms; }
.dateGroup:nth-child(9)  { animation-delay: 320ms; }
.dateGroup:nth-child(10) { animation-delay: 360ms; }
.dateGroup:nth-child(11) { animation-delay: 400ms; }
.dateGroup:nth-child(12) { animation-delay: 440ms; }

@keyframes groupEntrance {
  from { opacity: 0; transform: translateY(8px); }
  to   { opacity: 1; transform: translateY(0); }
}

.dateGroupLabel {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 11px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  padding: 0 2px 10px;
  @include m.mono;
  // colour is set inline via style prop from FileLibrary
}

// The animated icon that sits before the date text
.dateGroupDot {
  position: relative;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 14px;
  height: 14px;
  flex-shrink: 0;

  // Inner solid dot
  &::before {
    content: '';
    position: absolute;
    width: 5px;
    height: 5px;
    border-radius: 50%;
    background: currentColor;
    animation: dotPulse 2.4s ease-in-out infinite;
  }

  // Expanding ring
  &::after {
    content: '';
    position: absolute;
    width: 12px;
    height: 12px;
    border-radius: 50%;
    border: 1.5px solid currentColor;
    opacity: 0;
    animation: ringExpand 2.4s ease-out infinite;
  }
}

@keyframes dotPulse {
  0%, 100% { transform: scale(1);    opacity: 1;    }
  50%       { transform: scale(1.25); opacity: 0.75; }
}

@keyframes ringExpand {
  0%   { transform: scale(0.3); opacity: 0.7; }
  60%  { transform: scale(1);   opacity: 0;   }
  100% { transform: scale(1);   opacity: 0;   }
}

// ── File list (flat rows, borders merge) ──
.fileList {
  display: flex;
  flex-direction: column;
  border: 1px solid var(--bdr);
  border-radius: var(--rl);
  overflow: hidden;
}

// ── File row ─────────────────────────────────────
.fileRow {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 11px 14px;
  background: var(--bg1);
  border-bottom: 1px solid var(--bdr);
  position: relative;
  transition: background 0.12s;

  &:last-child { border-bottom: none; }

  &::before {
    content: '';
    position: absolute;
    left: 0;
    top: 0;
    bottom: 0;
    width: 3px;
    background: var(--row-bar, transparent);
  }
}

.fileRowBarCompleted { --row-bar: var(--green); }
.fileRowBarFailed    { --row-bar: var(--red); }
.fileRowBarPending   { --row-bar: var(--bdr3); }

.fileRowBarQueued {
  --row-bar: var(--amber);
  overflow: hidden;
  .cardSweep {
    background: conic-gradient(
      from 0deg at 50% 50%,
      transparent 0deg, transparent 315deg,
      rgba(245,158,11,0.3) 320deg, #f59e0b 330deg,
      rgba(245,158,11,0.3) 340deg, transparent 344deg, transparent 360deg
    );
    --sweep-speed: 3s;
  }
}

.fileRowBarRunning {
  --row-bar: var(--blue);
  overflow: hidden;
  .cardSweep {
    background: conic-gradient(
      from 0deg at 50% 50%,
      transparent 0deg, transparent 310deg,
      rgba(59,130,246,0.25) 315deg, #3b82f6 325deg,
      rgba(59,130,246,0.25) 335deg, transparent 338deg, transparent 360deg
    );
    --sweep-speed: 1.8s;
  }
}

.fileRowClickable {
  cursor: pointer;
  &:hover { background: var(--bg2); }
  &:focus-visible {
    outline: none;
    background: var(--bg2);
    box-shadow: inset 0 0 0 2px var(--blue-bdr);
  }
}

.fileRowIcon {
  flex-shrink: 0;
  width: 28px;
  height: 28px;
  border-radius: 7px;
  display: flex;
  align-items: center;
  justify-content: center;

  svg { width: 13px; height: 13px; }
}

.fileRowIconVideo {
  background: rgba(99, 102, 241, 0.1);
  color: rgba(165, 180, 252, 0.9);
  border: 1px solid rgba(99, 102, 241, 0.18);
}

.fileRowIconAudio {
  background: rgba(20, 184, 166, 0.1);
  color: rgba(94, 234, 212, 0.85);
  border: 1px solid rgba(20, 184, 166, 0.16);
}

.fileRowMain {
  flex: 1;
  min-width: 0;
}

.fileRowName {
  font-size: 13px;
  font-weight: 500;
  color: var(--t0);
  @include m.truncate;
}

.fileRowSub {
  font-size: 11px;
  color: var(--t2);
  margin-top: 1px;
  @include m.mono;
}

.pill {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  font-size: 11px;
  font-weight: 500;
  padding: 3px 9px;
  border-radius: 99px;
  white-space: nowrap;
  flex-shrink: 0;
  @include m.mono;
}

.pillDot {
  width: 5px;
  height: 5px;
  border-radius: 50%;
  background: currentColor;
  flex-shrink: 0;
}

.pillGreen { background: rgba(34, 197, 94, 0.1);  color: var(--green); border: 1px solid rgba(34, 197, 94, 0.2); }
.pillBlue  { background: rgba(59, 130, 246, 0.1);  color: var(--blue);  border: 1px solid rgba(59, 130, 246, 0.2); }
.pillAmber { background: rgba(245, 158, 11, 0.1);  color: var(--amber); border: 1px solid rgba(245, 158, 11, 0.2); }
.pillRed   { background: rgba(239, 68, 68, 0.1);   color: var(--red);   border: 1px solid rgba(239, 68, 68, 0.2); }
.pillGray  { background: var(--bg2); color: var(--t2); border: 1px solid var(--bdr2); }

.fileRowProgress {
  width: 80px;
  height: 3px;
  background: var(--bg3);
  border-radius: 99px;
  overflow: hidden;
  flex-shrink: 0;
}

.fileRowProgressFill {
  height: 100%;
  background: var(--blue);
  border-radius: 99px;
  transition: width 0.4s ease;
}

.fileRowAction {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 4px 10px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t1);
  font-size: 11px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.12s;
  font-family: var(--font-ui);
  white-space: nowrap;
  flex-shrink: 0;

  svg { width: 11px; height: 11px; }

  &:hover {
    background: var(--bg3);
    color: var(--t0);
    border-color: var(--bdr3);
  }
}

.fileRowChevron {
  color: var(--bdr3);
  display: flex;
  align-items: center;
  flex-shrink: 0;
  svg { width: 14px; height: 14px; }
}

// ── Processing sweep line ──────────────────────
// For flat rows the conic-gradient must be a square centred on the row so
// the arc travels the full perimeter evenly. We use a large fixed square
// (300px covers any row width up to ~300px). The row's overflow:hidden
// clips it to the row boundary. The mask leaves a 2px strip on all edges.
// All row children are raised to z-index:5 so they sit above the mask.

.fileRow > * {
  position: relative;
  z-index: 5;
}

.cardSweep {
  position: absolute;
  // Centre a 300px square on the row regardless of row dimensions
  width: 300px;
  height: 300px;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%) rotate(0deg);
  z-index: 3;
  animation: sweepAround var(--sweep-speed, 2s) linear infinite;
  pointer-events: none;
  border-radius: 0;
}

.cardSweepMask {
  position: absolute;
  inset: 2px;
  border-radius: calc(var(--rl) - 2px);
  background: var(--bg1);
  z-index: 4;
  pointer-events: none;
}

@keyframes sweepAround {
  from { transform: translate(-50%, -50%) rotate(0deg); }
  to   { transform: translate(-50%, -50%) rotate(360deg); }
}

// ══════════════════════════════════════
// LIBRARY GRID CARDS
// Windows-explorer-style tile grid for the library column.
// Each card: large icon area on top, filename + meta below.
// ══════════════════════════════════════

.libCardGrid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
  gap: 8px;
}

.libCard {
  display: flex;
  flex-direction: row;
  align-items: center;
  gap: 10px;
  width: 100%;
  background: var(--bg1);
  border: 1px solid var(--bdr);
  border-radius: var(--rl);
  overflow: hidden;
  position: relative;
  cursor: default;
  padding: 10px 12px 10px 17px;
  min-width: 0;
  // Animate transform + shadow + border + background
  transition:
    transform 0.22s cubic-bezier(0.34, 1.56, 0.64, 1),
    box-shadow 0.22s ease,
    border-color 0.18s ease,
    background 0.18s ease;

  // 3px left status bar
  &::before {
    content: '';
    position: absolute;
    left: 0;
    top: 0;
    bottom: 0;
    width: 3px;
    background: var(--card-bar, transparent);
    z-index: 1;
    transition: width 0.22s cubic-bezier(0.34, 1.56, 0.64, 1);
  }

  // Shimmer sweep overlay — slides across on hover
  &::after {
    content: '';
    position: absolute;
    top: 0;
    left: -100%;
    width: 60%;
    height: 100%;
    background: linear-gradient(
      105deg,
      transparent 20%,
      rgba(255, 255, 255, 0.04) 50%,
      transparent 80%
    );
    transition: left 0.45s ease;
    pointer-events: none;
    z-index: 2;
  }
}

.libCardClickable {
  cursor: pointer;

  &:hover {
    transform: translateY(-2px);
    background: var(--bg2);
    border-color: var(--bdr2);
    box-shadow:
      0 6px 20px rgba(0, 0, 0, 0.2),
      0 1px 4px rgba(0, 0, 0, 0.08),
      0 0 0 1px var(--bdr2);

    // Widen the accent bar on hover
    &::before {
      width: 4px;
    }

    // Trigger the shimmer sweep
    &::after {
      left: 140%;
    }

    // Scale the icon badge slightly
    .libCardIcon {
      transform: scale(1.12);
    }

    // Brighten the filename
    .libCardName {
      color: var(--t0);
      opacity: 0.95;
    }
  }

  &:active {
    transform: translateY(0px);
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.14);
    transition-duration: 0.08s;
  }

  &:focus-visible {
    outline: none;
    box-shadow: 0 0 0 2px var(--blue-bdr), 0 4px 16px rgba(0, 0, 0, 0.18);
  }
}

// Status bar colours
.libCardBarCompleted { --card-bar: var(--green); }
.libCardBarPending   { --card-bar: var(--bdr3); }
.libCardBarFailed    { --card-bar: var(--red); }
.libCardBarQueued    { --card-bar: var(--amber); }
.libCardBarRunning   { --card-bar: var(--blue); }

// ── Icon badge (inline, left of text) ─────────────
.libCardIcon {
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  align-self: center;
  width: 30px;
  height: 30px;
  border-radius: 7px;
  position: relative;
  transition: transform 0.22s cubic-bezier(0.34, 1.56, 0.64, 1);
}

.libCardIconVideo {
  background: rgba(99, 102, 241, 0.1);
  border: 1px solid rgba(99, 102, 241, 0.18);
  .libCardIconGlyph { color: rgba(165, 180, 252, 0.9); }
}

.libCardIconAudio {
  background: rgba(20, 184, 166, 0.1);
  border: 1px solid rgba(20, 184, 166, 0.16);
  .libCardIconGlyph { color: rgba(94, 234, 212, 0.85); }
}

.libCardIconGlyph {
  svg { width: 14px; height: 14px; }
}

// Hidden — no longer used
.libCardStatusBadge { display: none; }
.libCardProgressBar { display: none; }
.libCardProgressFill { display: none; }

// ── Info area: three lines ────────────────────────
.libCardInfo {
  display: flex;
  flex-direction: column;
  gap: 3px;
  min-width: 0;
  flex: 1;
}

.libCardName {
  font-size: 13px;
  font-weight: 500;
  color: var(--t0);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.libCardMeta {
  display: flex;
  align-items: center;
  gap: 5px;
  font-size: 11px;
  color: var(--t2);
  flex-wrap: nowrap;
  @include m.mono;
}

.libCardMetaSep { opacity: 0.5; }

.libCardDate { display: none; }

// Card highlight when file has been sent to Lecture Pipeline
.libCardInPipeline {
  border-color: rgba(56, 196, 186, 0.45) !important;
  box-shadow:
    0 0 0 1px rgba(56, 196, 186, 0.18),
    inset 0 0 12px rgba(56, 196, 186, 0.05);

  // Override the left accent bar with teal
  --card-bar: #38c4ba;

  &:hover {
    border-color: rgba(56, 196, 186, 0.65) !important;
    box-shadow:
      0 0 0 1px rgba(56, 196, 186, 0.28),
      0 4px 16px rgba(56, 196, 186, 0.12),
      inset 0 0 12px rgba(56, 196, 186, 0.07);
  }
}

// Badge shown when file has been added to lecture pipeline (icon only)
.lecturePipelineBadge {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: rgba(56, 196, 186, 0.12);
  color: #38c4ba;
  border: 1px solid rgba(56, 196, 186, 0.35);
  flex-shrink: 0;

  svg { width: 11px; height: 11px; }
}

// libCardBottom no longer used — pill is inline in libCardMeta
.libCardBottom { display: none; }

// Action button (Generate / Retry / Re-generate)
.libCardAction {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 4px 9px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t1);
  font-size: 11px;
  font-weight: 500;
  cursor: pointer;
  font-family: var(--font-ui);
  transition: all 0.12s;

  svg { width: 11px; height: 11px; }

  &:hover {
    background: var(--bg3);
    color: var(--t0);
    border-color: var(--bdr3);
  }
}


.uploadPanel {
  background: var(--bg1);
  border: 1px solid var(--bdr2);
  border-radius: var(--rl);
  margin-bottom: 22px;
  overflow: hidden;
  animation: fadeSlide 0.18s ease;
  box-shadow: var(--shadow-sm);
}

.uploadPanelHeader {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 16px;
  border-bottom: none;
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
        rgba(139, 92, 246, 0.35) 20%,
        rgba(56, 196, 186, 0.4) 50%,
        rgba(240, 160, 48, 0.35) 80%,
        rgba(240, 160, 48, 0.0) 100%);
    pointer-events: none;
  }
}

.uploadPanelTitle {
  font-size: 13px;
  font-weight: 600;
  color: var(--t0);
  font-family: var(--font-display);
  letter-spacing: -0.1px;
}

.uploadPanelClose {
  width: 26px;
  height: 26px;
  border-radius: var(--r);
  border: 1px solid transparent;
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.12s;

  svg {
    width: 13px;
    height: 13px;
  }

  &:hover {
    background: var(--bg3);
    color: var(--t0);
    border-color: var(--bdr2);
  }
}

.uploadPanelBody {
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.uploadField {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.uploadLabel {
  @include m.label-caps;
}

// Drop area
.uploadDrop {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 6px;
  padding: 22px 16px;
  border: 1px dashed var(--bdr2);
  border-radius: var(--r);
  background: var(--bg2);
  cursor: pointer;
  transition: all 0.14s;
  text-align: center;

  &:hover {
    border-color: var(--blue-bdr);
    background: var(--blue-dim);
  }
}

.uploadDropActive {
  border-color: var(--blue);
  background: var(--blue-dim);
}

.uploadDropIc {
  color: var(--blue);
  margin-bottom: 2px;

  svg {
    width: 22px;
    height: 22px;
  }
}

.uploadDropText {
  font-size: 13px;
  font-weight: 500;
  color: var(--t0);
  font-family: var(--font-display);
}

.uploadDropHint {
  font-size: 10px;
  color: var(--t2);
  @include m.mono;
  letter-spacing: 0.04em;
}

// Selected file pill
.uploadFilePill {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 12px;
  background: var(--bg2);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
}

.uploadFileName {
  flex: 1;
  font-size: 12px;
  color: var(--t0);
  font-weight: 500;
  @include m.truncate;
}

.uploadFileSize {
  font-size: 10px;
  color: var(--t2);
  @include m.mono;
  white-space: nowrap;
}

.uploadFileClear {
  width: 22px;
  height: 22px;
  border-radius: 4px;
  border: 1px solid transparent;
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.12s;
  flex-shrink: 0;

  svg {
    width: 11px;
    height: 11px;
  }

  &:hover {
    color: var(--red);
    background: var(--red-dim);
  }
}

.uploadError {
  font-size: 11px;
  color: var(--red);
  @include m.mono;
}

// ── Model selector ──────────────────────────────
.modelGroup {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 8px;
}

.modelOpt {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 4px;
  padding: 12px 12px 10px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: var(--bg2);
  cursor: pointer;
  text-align: left;
  transition: all 0.12s;
  font-family: var(--font-ui);

  &:hover {
    border-color: var(--bdr3);
    background: var(--bg3);
  }
}

.modelOptActive {
  border-color: var(--blue);
  background: var(--blue-dim);
  box-shadow: 0 0 0 1px var(--blue-bdr);

  .modelOptName {
    color: var(--t0);
  }
}

.modelOptName {
  font-size: 12px;
  font-weight: 600;
  color: var(--t1);
}

.modelOptHint {
  font-size: 10px;
  color: var(--t2);
  @include m.mono;
  letter-spacing: 0.02em;
}

.modelOptSpeed {
  display: flex;
  gap: 3px;
  margin-top: 4px;
}

.modelOptSpeedBar {
  width: 16px;
  height: 3px;
  border-radius: 99px;
  background: var(--bdr2);
}

.modelOptSpeedBarOn {
  background: var(--blue);
}

.uploadActions {
  display: flex;
  justify-content: flex-end;
  gap: 8px;
  padding-top: 4px;
  border-top: 1px solid var(--bdr);
  margin-top: 4px;
  padding-top: 14px;
}

// ══════════════════════════════════════
// PLAYER + TRANSCRIPT (preserved from original)
// ══════════════════════════════════════
.playerBody {
  flex: 1;
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 0;
  overflow: hidden;
  min-height: 0;
  background: var(--bg0);

  @media (max-width: 1100px) {
    grid-template-columns: 1fr;
    overflow: auto;
  }
}

// Left column — video fills the full height
.mediaPane {
  display: flex;
  flex-direction: column;
  min-height: 0;
  min-width: 0;
  padding: 18px 14px 18px 22px;
  background: var(--bg0);
}

.mediaFrame {
  flex: 1;
  background: #000;
  border: 1px solid var(--bdr2);
  border-radius: 0;
  overflow: hidden;
  position: relative;
  box-shadow: none;
  min-height: 0;

  video, audio { width: 100%; height: 100%; display: block; }
}

.mediaEl {
  width: 100%;
  height: 100%;
  display: block;
  background: #000;
  object-fit: contain;
}

.mediaLoading {
  position: absolute;
  inset: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 10px;
  background: rgba(0, 0, 0, 0.45);
  backdrop-filter: blur(2px);
  color: #fff;
  font-size: 12px;
  font-family: var(--font-ui);
  pointer-events: none;
}

.mediaError {
  position: absolute;
  inset: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 6px;
  background: rgba(0, 0, 0, 0.6);
  color: #fff;
  font-size: 13px;
  font-weight: 500;
  font-family: var(--font-ui);
  padding: 20px;
  text-align: center;
}

.mediaErrorMsg {
  font-size: 11px;
  color: rgba(255, 255, 255, 0.6);
  @include m.mono;
  max-width: 80%;
  word-break: break-word;
}

.audioStage {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 18px;
  padding: 32px;
  background: linear-gradient(140deg, #0c0c12 0%, #141420 100%);
}

.audioStageIc {
  width: 72px;
  height: 72px;
  min-width: 72px;
  min-height: 72px;
  border-radius: 50%;
  background: var(--blue-dim);
  border: 1px solid var(--blue-bdr);
  color: var(--blue);
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  align-self: center;

  svg { width: 32px; height: 32px; }
}

.audioStageName {
  color: rgba(255,255,255,0.65);
  font-size: 14px;
  font-weight: 500;
  text-align: center;
  word-break: break-all;
  max-width: 75%;
  line-height: 1.4;
}

.audioEl {
  width: 100%;
  max-width: 480px;
}

// Right column — 2 equal sub-columns: liveBox | transcript list
.transcriptPane {
  display: grid;
  grid-template-rows: 300px 1fr;
  grid-template-columns: 1fr;
  background: var(--bg0);
  overflow: hidden;
  min-height: 0;
  min-width: 0;
  position: relative;

  // Left gradient border (separator from mediaPane)
  &::before {
    content: '';
    position: absolute;
    top: 0; left: 0; bottom: 0;
    width: 1px;
    background: linear-gradient(180deg,
      rgba(139, 92, 246, 0.0)  0%,
      rgba(139, 92, 246, 0.55) 20%,
      rgba(56, 196, 186, 0.65) 50%,
      rgba(240, 160, 48, 0.55) 80%,
      rgba(240, 160, 48, 0.0)  100%);
    pointer-events: none;
    z-index: 1;
  }

  @media (max-width: 1500px) {
    grid-template-rows: 1fr 1fr;
  }

  @media (max-width: 1100px) {
    grid-template-rows: auto auto;
    grid-template-columns: 1fr;
    max-height: 60vh;
    overflow: auto;
  }
}

// Top row — Live cue spotlight
.liveBox {
  display: flex;
  flex-direction: column;
  background: var(--bg1);
  overflow: hidden;
  min-height: 0;
  min-width: 0;
  position: relative;

  // Horizontal gradient separator between liveBox and transcript list
  &::after {
    content: '';
    position: absolute;
    bottom: 0; left: 0; right: 0;
    height: 1px;
    background: linear-gradient(90deg,
      rgba(139, 92, 246, 0.0)  0%,
      rgba(139, 92, 246, 0.5)  20%,
      rgba(56, 196, 186, 0.6)  50%,
      rgba(240, 160, 48, 0.5)  80%,
      rgba(240, 160, 48, 0.0)  100%);
    pointer-events: none;
    z-index: 2;
  }

  // Accent top line
  &::before {
    content: '';
    position: absolute;
    top: 0; left: 0; right: 0;
    height: 2px;
    background: linear-gradient(90deg,
      transparent 0%,
      var(--blue-bdr) 25%,
      rgba(56,196,186,0.45) 55%,
      transparent 100%);
    transition: background 0.2s;
    z-index: 1;
  }
}

.liveBoxEditing {
  &::before {
    background: linear-gradient(90deg, transparent 0%, var(--amber-bdr) 40%, transparent 100%);
  }
}

// LiveBox header — fixed at top
.liveBoxHeader {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 14px 9px;
  flex-shrink: 0;
  position: relative;
  background: var(--bg1);
  z-index: 2;

  &::after {
    content: '';
    position: absolute;
    bottom: 0; left: 0; right: 0;
    height: 1px;
    background: linear-gradient(90deg,
      rgba(139,92,246,0) 0%, rgba(139,92,246,0.3) 30%,
      rgba(56,196,186,0.35) 60%, rgba(240,160,48,0) 100%);
  }
}

.liveBoxHeaderTitle {
  font-size: 11px;
  font-weight: 600;
  color: var(--t2);
  text-transform: uppercase;
  letter-spacing: 0.07em;
  @include m.mono;
}

// LiveBox body — scrollable, content starts from top
.liveBoxBody {
  flex: 1;
  overflow-y: auto;
  padding: 14px 16px 16px;
  display: flex;
  flex-direction: column;
  gap: 6px;
  align-items: flex-start;    // content aligns to top
  justify-content: flex-start;
  @include m.scrollbar;
}

.liveBoxMeta {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 8px;
  font-size: 10px;
  color: var(--t2);
  letter-spacing: 0.06em;
  text-transform: uppercase;
  @include m.mono;
  flex-shrink: 0;
  width: 100%;
}

.liveBoxInfo { color: var(--t2); }
.liveBoxStatus { font-weight: 600; }
.liveBoxStatusEditing { color: var(--amber); }
.liveBoxStatusSaved   { color: var(--green); }
.liveBoxStatusFailed  { color: var(--red); }

.liveBoxText {
  font-family: var(--font-display);
  font-size: 16px;
  line-height: 1.6;
  color: var(--t0);
  letter-spacing: -0.1px;
  flex-shrink: 0;
  width: 100%;
}

// Right sub-column — transcript list (70% → now 50%)
.transcriptScrollPane {
  display: flex;
  flex-direction: column;
  background: var(--bg0);
  overflow: hidden;
  min-height: 0;
}

.transcriptHeader {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 11px 16px 10px;
  flex-shrink: 0;
  position: relative;
  background: var(--bg1);

  &::after {
    content: '';
    position: absolute;
    bottom: 0; left: 0; right: 0;
    height: 1px;
    background: linear-gradient(90deg,
      rgba(139,92,246,0) 0%, rgba(139,92,246,0.35) 20%,
      rgba(56,196,186,0.4) 50%, rgba(240,160,48,0.35) 80%,
      rgba(240,160,48,0) 100%);
  }
}

.transcriptTitle {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 12px;
  font-weight: 600;
  color: var(--t1);
  font-family: var(--font-display);
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.jobCount {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 20px;
  height: 17px;
  padding: 0 6px;
  border-radius: 9px;
  background: var(--bg2);
  color: var(--t1);
  font-size: 10px;
  font-weight: 600;
  text-transform: none;
  letter-spacing: 0;
}

.transcriptHint {
  font-size: 10px;
  color: var(--t2);
  letter-spacing: 0.03em;
  @include m.mono;
}

// ── Cue list ──────────────────────────────────
.cueList {
  flex: 1;
  overflow-y: auto;
  padding: 6px 8px 16px;
  display: flex;
  flex-direction: column;
  gap: 2px;

  &::-webkit-scrollbar { width: 5px; }
  &::-webkit-scrollbar-thumb {
    background: var(--bdr2);
    border-radius: 3px;
    &:hover { background: var(--bdr3); }
  }
}

.cueEmpty {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 10px;
  padding: 40px 20px;
  color: var(--t2);
  font-size: 12px;
}

// ══════════════════════════════════════
// SCRUB BAR
// ══════════════════════════════════════
.timelineWrap {
  padding: 12px 14px 8px;
  border-bottom: 1px solid var(--bdr2);
  flex-shrink: 0;
  background: var(--bg1);
}

.timeline {
  position: relative;
  width: 100%;
}

.timelineTrack {
  position: relative;
  height: 22px;
  cursor: pointer;
  display: flex;
  align-items: center;
}

.timelineLine {
  position: absolute;
  left: 0; right: 0;
  top: 50%;
  transform: translateY(-50%);
  height: 2px;
  background: var(--bdr2);
  border-radius: 99px;
}

.timelineLinePlayed {
  position: absolute;
  left: 0;
  top: 50%;
  transform: translateY(-50%);
  height: 2px;
  background: linear-gradient(90deg, var(--blue), #38c4ba);
  border-radius: 99px;
  transition: width 0.18s linear;
}

.timelineMarker {
  position: absolute;
  top: 50%;
  width: 18px; height: 18px;
  margin-left: -9px;
  border-radius: 50%;
  border: 2px solid;
  background: var(--bg1);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  z-index: 2;
  transform: translateY(-50%);
  transition: transform 0.12s;

  &:hover { transform: translateY(-50%) scale(1.15); }
  svg { width: 10px; height: 10px; }
}

.timelineMarkerPast   { border-color: var(--blue); background: var(--blue); color: #fff; }
.timelineMarkerActive { border-color: var(--amber); background: var(--amber); color: #fff; box-shadow: 0 0 0 4px rgba(240,160,48,0.18); }
.timelineMarkerFuture { border-color: var(--bdr3); background: var(--bg1); }

.timelinePlayhead {
  position: absolute;
  top: 50%;
  width: 10px; height: 10px;
  margin-left: -5px;
  border-radius: 50%;
  background: var(--amber);
  pointer-events: none;
  z-index: 1;
  transform: translateY(-50%);
  transition: left 0.18s linear;
}

.timelineAxis {
  position: relative;
  height: 18px;
  margin-top: 2px;
}

.timelineAxisTick {
  position: absolute;
  top: 0;
  font-size: 10px;
  font-variant-numeric: tabular-nums;
  color: var(--t2);
  transform: translateX(-50%);
  white-space: nowrap;
  @include m.mono;
}

// ══════════════════════════════════════
// CUE CARDS
// ══════════════════════════════════════
.cueCard {
  background: transparent;
  border: 1px solid transparent;
  border-radius: 7px;
  padding: 9px 11px;
  transition: background 0.12s, border-color 0.12s;

  &:hover {
    background: var(--bg2);
    border-color: var(--bdr);
  }
}

.cueCardActive {
  background: color-mix(in srgb, var(--blue-dim) 65%, transparent 35%);
  border-color: var(--blue-bdr);
  &:hover { background: var(--blue-dim); }
}

.cueCardEditing {
  background: var(--bg2);
  border-color: var(--amber-bdr);
  box-shadow: 0 0 0 1px var(--amber-dim);
}

.cueCardTimeRow {
  display: flex;
  align-items: center;
  gap: 7px;
  margin-bottom: 4px;
}

.cueCardTime {
  font-size: 10px;
  color: var(--blue);
  background: rgba(59,130,246,0.08);
  border: 1px solid rgba(59,130,246,0.14);
  border-radius: 4px;
  padding: 2px 6px;
  cursor: pointer;
  font-weight: 600;
  @include m.mono;
  transition: all 0.12s;

  &:hover:not(:disabled) { background: rgba(59,130,246,0.15); }
  &:disabled { cursor: default; opacity: 0.5; }
}

.cueCardCurrentTag {
  font-size: 10px;
  color: var(--amber);
  font-weight: 600;
  @include m.mono;
}

.cueCardSpacer { flex: 1; }

.cueCardAction {
  width: 24px; height: 24px;
  border-radius: var(--r);
  border: 1px solid transparent;
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  opacity: 0;
  transition: all 0.12s;

  svg { width: 11px; height: 11px; }

  .cueCard:hover &, .cueCardActive & { opacity: 1; }
  &:hover { background: var(--bg3); color: var(--t0); border-color: var(--bdr2); }
}

.cueCardEditActions { display: flex; gap: 5px; flex-shrink: 0; }

.cueCardActionSave,
.cueCardActionCancel {
  width: 24px; height: 24px;
  border-radius: var(--r);
  border: 1px solid;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.12s;

  svg { width: 12px; height: 12px; }
  &:disabled { opacity: 0.5; cursor: not-allowed; }
}

.cueCardActionSave {
  background: var(--green); color: #fff; border-color: var(--green);
  &:hover:not(:disabled) { opacity: 0.85; }
}

.cueCardActionCancel {
  background: transparent; color: var(--t2); border-color: var(--bdr2);
  &:hover:not(:disabled) { background: var(--bg3); color: var(--red); border-color: var(--red-bdr); }
}

.cueCardText {
  font-size: 12.5px;
  line-height: 1.5;
  color: var(--t0);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.cueCardTextareaWrap {
  position: relative;
  display: flex;
  flex-direction: column;
  gap: 4px;
  width: 100%;
}

.cueCardTextarea {
  width: 100%;
  background: var(--bg0);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  padding: 7px 10px;
  font-size: 13px;
  line-height: 1.5;
  color: var(--t0);
  outline: none;
  resize: vertical;
  min-height: 56px;
  font-family: var(--font-ui);
  transition: border-color 0.12s;

  &:focus { border-color: var(--amber-bdr); box-shadow: 0 0 0 2px var(--amber-dim); }
}

.cueCharCount {
  font-size: 10px;
  color: var(--t2);
  text-align: right;
  @include m.mono;
  letter-spacing: 0.03em;
}

.cueCharCountWarn {
  color: var(--amber);
  font-weight: 600;
}

// ── Empty-text save confirmation ────────────────
.cueCardEmptyConfirm {
  display: flex;
  flex-direction: column;
  gap: 8px;
  padding: 10px 12px;
  background: rgba(245, 158, 11, 0.07);
  border: 1px solid var(--amber-bdr);
  border-radius: var(--r);
  animation: fadeSlide 0.14s ease;
}

.cueCardEmptyWarn {
  font-size: 12px;
  color: var(--amber);
  font-weight: 500;
  line-height: 1.4;
  @include m.mono;
}

.cueCardEmptyActions {
  display: flex;
  gap: 7px;
  align-items: center;
}

.cueCardConfirmCancel {
  font-size: 11px;
  font-weight: 500;
  padding: 4px 10px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t1);
  cursor: pointer;
  font-family: var(--font-ui);
  transition: all 0.12s;

  &:hover { background: var(--bg3); color: var(--t0); border-color: var(--bdr3); }
}

.cueCardConfirmSave {
  font-size: 11px;
  font-weight: 600;
  padding: 4px 12px;
  border-radius: var(--r);
  border: 1px solid var(--amber-bdr);
  background: var(--amber-dim);
  color: var(--amber);
  cursor: pointer;
  font-family: var(--font-ui);
  display: inline-flex;
  align-items: center;
  gap: 5px;
  transition: all 0.12s;

  &:hover:not(:disabled) { background: rgba(245,158,11,0.2); }
  &:disabled { opacity: 0.5; cursor: not-allowed; }
}



.cueRow {
  display: grid;
  grid-template-columns: 52px 1fr 14px;
  gap: 8px;
  align-items: start;
  padding: 8px 8px 8px 6px;
  border-radius: var(--r);
  border: 1px solid transparent;
  transition: background 0.12s, border-color 0.12s;
  position: relative;

  &:hover {
    background: var(--bg2);
  }
}

.cueRowActive {
  background: var(--blue-dim);
  border-color: var(--blue-bdr);

  &:hover {
    background: var(--blue-dim);
  }

  .cueTime {
    background: var(--blue);
    color: #fff;
    border-color: var(--blue);
  }
}

.cueRowEdited {
  .cueTime {
    border-left: 2px solid var(--amber);
  }
}

.cueTime {
  font-family: var(--font-ui);
  font-size: 10px;
  font-variant-numeric: tabular-nums;
  color: var(--t2);
  background: var(--bg2);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  padding: 4px;
  cursor: pointer;
  height: 24px;
  letter-spacing: 0.02em;
  font-weight: 500;
  margin-top: 1px;
  transition: all 0.12s;

  &:hover {
    background: var(--bg3);
    color: var(--t0);
  }
}

.cueText {
  width: 100%;
  background: transparent;
  border: none;
  outline: none;
  resize: none;
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 13px;
  line-height: 1.5;
  padding: 3px 4px;
  border-radius: 3px;
  overflow: hidden;
  min-height: 24px;
  transition: background 0.12s;

  &:focus {
    background: var(--bg0);
    box-shadow: inset 0 0 0 1px var(--amber-bdr);
  }
}

.cueEditedDot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: var(--amber);
  margin-top: 10px;
  flex-shrink: 0;
}

// ══════════════════════════════════════
// MODAL (upload + future overlays)
// ══════════════════════════════════════
.modalBackdrop {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.55);
  backdrop-filter: blur(2px);
  z-index: 200;
  display: flex;
  align-items: flex-start;
  justify-content: center;
  padding: 80px 20px 20px;
  overflow-y: auto;
  animation: fadeIn 0.15s ease;
}

.modalCard {
  width: 100%;
  max-width: 560px;
  background: var(--bg1);
  border: 1px solid var(--bdr2);
  border-radius: var(--rl);
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.35);
  animation: modalSlide 0.18s ease;
  overflow: hidden;

  // The UploadPanel inside the modal has its own header/border. Strip its
  // outer border + margin so it doesn't double up with the modal card.
  >.uploadPanel {
    border: none;
    margin: 0;
    box-shadow: none;
    border-radius: 0;
  }
}

@keyframes fadeIn {
  from {
    opacity: 0;
  }

  to {
    opacity: 1;
  }
}

@keyframes modalSlide {
  from {
    opacity: 0;
    transform: translateY(-12px);
  }

  to {
    opacity: 1;
    transform: translateY(0);
  }
}

// ══════════════════════════════════════
// DATE FILTER · APPLY BUTTON
// ══════════════════════════════════════
.dateApply {
  background: var(--blue);
  border: none;
  color: #fff;
  font-size: 10px;
  font-weight: 600;
  padding: 3px 8px;
  border-radius: 4px;
  cursor: pointer;
  font-family: var(--font-ui);
  transition: all 0.12s;
  letter-spacing: 0.02em;
  white-space: nowrap;

  &:hover:not(:disabled) { background: #a78bfa; }
  &:disabled { opacity: 0.4; cursor: not-allowed; background: var(--bg3); color: var(--t2); }
}

// ══════════════════════════════════════
// ERROR BANNER (network / API errors)
// ══════════════════════════════════════
.errorBanner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  padding: 10px 14px;
  margin-bottom: 18px;
  background: var(--red-dim);
  border: 1px solid var(--red-bdr);
  border-radius: var(--r);
  color: var(--red);
  font-size: 12px;
  font-family: var(--font-ui);
}

// ══════════════════════════════════════
// INFO BANNER (neutral notices, e.g. "a job is running")
// ══════════════════════════════════════
.infoBanner {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 14px;
  margin-bottom: 18px;
  background: var(--blue-dim);
  border: 1px solid var(--blue-bdr);
  border-radius: var(--r);
  color: var(--blue);
  font-size: 12px;
  font-family: var(--font-ui);
}

// Disabled drop-zone treatment (during in-flight upload)
.uploadDropDisabled {
  opacity: 0.55;
  cursor: not-allowed;
  pointer-events: none;
}

// ══════════════════════════════════════
// INLINE UPLOAD WIDGET
// (replacement for the modal upload flow — background uploads with
//  per-task progress pills, non-blocking)
// ══════════════════════════════════════
// uploadInline wrapper removed — component now renders uploadCol directly

// ── Compact horizontal dropzone ──
// Taller than before so it reads as a proper landing zone rather than just
// a hint strip. A vivid blue wash on hover/drag makes the intent obvious.
// ── Drop zone ─────────────────────────────────
.uploadInlineDrop {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 10px;
  min-height: 160px;
  padding: 28px 20px;
  border: 1.5px dashed var(--bdr2);
  border-radius: var(--rl);
  background: var(--bg1);
  cursor: pointer;
  transition: all 0.15s ease;
  text-align: center;

  &:hover {
    border-color: var(--blue-bdr);
    background: var(--blue-dim);
  }
}

.uploadInlineDropActive {
  border-color: var(--blue);
  border-style: solid;
  background: var(--blue-dim);
}

.uploadInlineDropIc {
  color: var(--t2);
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 36px;
  height: 36px;
  border-radius: 8px;
  background: var(--bg2);
  border: 1px solid var(--bdr2);

  svg { width: 16px; height: 16px; }
}

.uploadInlineDropMain {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 5px;
}

.uploadInlineDropText {
  font-size: 13px;
  font-weight: 500;
  color: var(--t1);
  font-family: var(--font-display);
}

.uploadInlineDropHint {
  font-size: 11px;
  color: var(--t2);
  line-height: 1.5;
  @include m.mono;
  letter-spacing: 0.02em;
}

.uploadInlineDropFormats {
  display: none;
}

// Browse files / folder button row
.uploadBrowseRow {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-top: 6px;
}

.uploadBrowseBtn {
  padding: 5px 14px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: var(--bg2);
  color: var(--t1);
  font-size: 11px;
  font-weight: 500;
  cursor: pointer;
  font-family: var(--font-ui);
  transition: all 0.14s;

  &:hover {
    background: var(--bg3);
    border-color: var(--bdr3);
    color: var(--t0);
  }
}

.uploadBrowseSep {
  font-size: 11px;
  color: var(--t2);
}


.uploadTaskList {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.uploadTask {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 10px 12px;
  background: var(--bg1);
  border: 1px solid var(--bdr);
  border-radius: var(--r);
  animation: fadeSlide 0.18s ease;
  transition: background 0.18s, border-color 0.18s;
}

.uploadTaskUploading {
  border-color: var(--blue-bdr);
}

.uploadTaskDone {
  border-color: var(--green-bdr);
  background: var(--green-dim);
  animation: fadeSlide 0.18s ease, fadeOutLate 2.5s ease forwards;
}

.uploadTaskFailed {
  border-color: var(--red-bdr);
  background: var(--red-dim);
}

@keyframes fadeOutLate {

  // Stay full opacity for most of the lifetime, then fade out near the end
  // so the user has time to read "Uploaded" before the row vanishes.
  0%,
  80% {
    opacity: 1;
  }

  100% {
    opacity: 0;
  }
}

.uploadTaskIc {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 24px;
  height: 24px;

  .uploadTaskDone & {
    color: var(--green);
  }

  .uploadTaskFailed & {
    color: var(--red);
  }

  svg {
    width: 15px;
    height: 15px;
  }
}

// Small dark spinner (used inside light/blue task rows, not the green save btn)
.spinnerSmallDark {
  width: 14px;
  height: 14px;
  border: 1.5px solid var(--bdr2);
  border-top-color: var(--blue);
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

.uploadTaskInfo {
  flex: 1;
  min-width: 0;
}

.uploadTaskName {
  font-size: 12px;
  font-weight: 600;
  color: var(--t0);
  @include m.truncate;
}

.uploadTaskMeta {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 10px;
  color: var(--t2);
  margin-top: 2px;
  @include m.mono;
  letter-spacing: 0.02em;
}

.uploadTaskDoneText {
  color: var(--green);
  font-weight: 600;
}

.uploadTaskFailText {
  color: var(--red);
  font-weight: 500;
  // Failed messages can be long; let them wrap rather than truncate.
  white-space: normal;
}

.uploadTaskProgress {
  margin-top: 6px;
  height: 3px;
  background: var(--bg3);
  border-radius: 99px;
  overflow: hidden;
}

.uploadTaskProgressFill {
  height: 100%;
  background: linear-gradient(90deg, var(--blue), #a78bfa);
  border-radius: 99px;
  transition: width 0.18s ease;
}

// Indeterminate slider — shown when the server didn't expose Content-Length.
// A 30%-wide bar slides back and forth across the track.
.uploadTaskProgressIndeterminate {
  width: 30%;
  height: 100%;
  background: linear-gradient(90deg, var(--blue), #a78bfa);
  border-radius: 99px;
  animation: indeterminate 1.4s ease-in-out infinite;
}

@keyframes indeterminate {
  0% {
    transform: translateX(-100%);
  }

  100% {
    transform: translateX(333%);
  }

  // moves a 30% bar across full width
}

.uploadTaskActions {
  display: flex;
  align-items: center;
  gap: 6px;
  flex-shrink: 0;
}

.uploadTaskRetry {
  background: transparent;
  border: 1px solid var(--red-bdr);
  color: var(--red);
  font-size: 11px;
  font-weight: 600;
  padding: 3px 9px;
  border-radius: var(--r);
  cursor: pointer;
  font-family: var(--font-ui);
  transition: all 0.12s;

  &:hover {
    background: var(--red);
    color: #fff;
  }
}

.uploadTaskDismiss {
  width: 22px;
  height: 22px;
  border-radius: var(--r);
  border: 1px solid transparent;
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.12s;

  svg {
    width: 11px;
    height: 11px;
  }

  &:hover {
    background: var(--bg3);
    color: var(--t0);
    border-color: var(--bdr2);
  }
}

// ══════════════════════════════════════
// UPLOAD-PANEL EXTRAS
// ══════════════════════════════════════
.uploadHint {
  font-size: 11px;
  color: var(--t2);
  padding: 8px 2px;
  @include m.mono;
}

.uploadLink {
  background: transparent;
  border: none;
  color: var(--blue);
  font-size: 11px;
  font-weight: 600;
  cursor: pointer;
  margin-left: 8px;
  padding: 0;
  font-family: var(--font-ui);

  &:hover {
    text-decoration: underline;
  }
}

.uploadSelect {
  width: 100%;
  padding: 8px 10px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: var(--bg2);
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 12px;
  outline: none;
  cursor: pointer;
  transition: border-color 0.12s;

  &:hover,
  &:focus {
    border-color: var(--bdr3);
  }
}

// ══════════════════════════════════════
// SPINNER (caption loading)
// ══════════════════════════════════════
.spinner {
  width: 22px;
  height: 22px;
  border: 2px solid var(--bdr2);
  border-top-color: var(--blue);
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

.spinnerSmall {
  width: 12px;
  height: 12px;
  border: 1.5px solid rgba(255, 255, 255, 0.35);
  border-top-color: #fff;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

// ══════════════════════════════════════
// TOAST
// ══════════════════════════════════════
.toast {
  position: fixed;
  bottom: 24px;
  left: 50%;
  transform: translateX(-50%);
  background: var(--bg1);
  color: var(--green);
  border: 1px solid var(--green-bdr);
  padding: 9px 18px;
  border-radius: var(--r);
  font-size: 12px;
  font-family: var(--font-ui);
  font-weight: 500;
  z-index: 100;
  animation: fadeSlide 0.18s ease;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.18);
}

// ══════════════════════════════════════
// ANIMATIONS
// ══════════════════════════════════════
@keyframes fadeSlide {
  from {
    opacity: 0;
    transform: translateY(5px);
  }

  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes breathe {

  0%,
  100% {
    opacity: 1;
  }

  50% {
    opacity: 0.35;
  }
}
// ═══════════════════════════════════════════════
// SttFileSidebar — reusable file-picker column
// used by the Inference and View Results tabs
// ═══════════════════════════════════════════════
.sttSidebar {
  width: 320px;
  flex-shrink: 0;
  display: flex;
  flex-direction: column;
  border-right: 1px solid var(--bdr);
  background: var(--bg1);
  min-height: 0;
  overflow: hidden;

  @media (max-width: 1500px) { width: 280px; }
}

.sttSidebarHeader {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 16px;
  border-bottom: 1px solid var(--bdr);
  flex-shrink: 0;
}

.sttSidebarTitle {
  font-size: 12px;
  font-weight: 600;
  color: var(--t0);
  font-family: var(--font-display);
}

.sttSidebarCount {
  font-size: 11px;
  color: var(--t2);
  @include m.mono;
  background: var(--bg2);
  padding: 1px 8px;
  border-radius: 99px;
}

.sttSidebarSearch {
  display: flex;
  align-items: center;
  gap: 8px;
  margin: 10px 14px 0;
  padding: 6px 10px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: var(--bg2);
  flex-shrink: 0;

  svg { width: 13px; height: 13px; color: var(--t2); flex-shrink: 0; }

  input {
    flex: 1;
    border: none;
    background: transparent;
    outline: none;
    font-size: 12px;
    color: var(--t0);
    font-family: var(--font-ui);
    min-width: 0;

    &::placeholder { color: var(--t2); }
  }
}

.sttSidebarChips {
  display: flex;
  flex-wrap: wrap;
  gap: 4px;
  padding: 10px 14px 0;
  flex-shrink: 0;
}

.sttSidebarSort {
  display: flex;
  gap: 4px;
  padding: 8px 14px 0;
  flex-shrink: 0;
}

.sttSortBtn {
  font-size: 10.5px;
  font-weight: 500;
  padding: 3px 8px;
  border-radius: var(--r);
  border: 1px solid transparent;
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  @include m.mono;
  display: inline-flex;
  align-items: center;
  gap: 2px;
  transition: all 0.12s;

  &:hover { background: var(--bg2); color: var(--t1); }
}

.sttSortBtnActive {
  background: var(--blue-dim);
  color: var(--blue);
  border-color: var(--blue-bdr);
}

.sttSortArrow { font-size: 10px; }

.sttSidebarSelectRow {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  padding: 10px 14px;
  margin: 10px 14px 0;
  background: rgba(139, 92, 246, 0.06);
  border: 1px solid rgba(139, 92, 246, 0.2);
  border-radius: var(--r);
  flex-shrink: 0;
}

.sttSidebarSelectHint {
  font-size: 11px;
  color: #a78bfa;
  @include m.mono;
  font-weight: 500;
}

.sttSidebarList {
  flex: 1;
  overflow-y: auto;
  padding: 10px 10px 16px;
  display: flex;
  flex-direction: column;
  gap: 4px;
  min-height: 0;
  @include m.scrollbar;
}

.sttSidebarEmpty {
  font-size: 12px;
  color: var(--t2);
  text-align: center;
  padding: 32px 12px;
}

.sttSidebarRow {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 10px;
  border-radius: var(--r);
  border: 1px solid transparent;
  background: transparent;
  cursor: pointer;
  text-align: left;
  transition: all 0.12s;
  width: 100%;

  &:hover:not(:disabled) { background: var(--bg2); border-color: var(--bdr2); }
}

.sttSidebarRowSelected {
  background: rgba(139, 92, 246, 0.08);
  border-color: rgba(139, 92, 246, 0.35);
}

.sttSidebarRowActive {
  background: var(--blue-dim);
  border-color: var(--blue-bdr);
}

.sttSidebarRowDisabled {
  opacity: 0.4;
  cursor: not-allowed;
  pointer-events: none;
}

.sttSidebarCheckbox {
  width: 16px;
  height: 16px;
  border-radius: 4px;
  border: 1.5px solid var(--bdr3);
  background: transparent;
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.12s;

  svg { width: 10px; height: 10px; }
}

.sttSidebarCheckboxChecked {
  background: #a78bfa;
  border-color: #a78bfa;
  color: #fff;
}

.sttSidebarRowName {
  flex: 1;
  min-width: 0;
  font-size: 12px;
  font-weight: 500;
  color: var(--t0);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

// ═══════════════════════════════════════════════
// Tab shell — page header + tab bar + tab panes
// ═══════════════════════════════════════════════
.sttPage {
  display: flex;
  flex-direction: column;
  height: 100%;
  min-height: 0;
  background: var(--bg0);
}

.sttHeaderBar {
  flex-shrink: 0;
  padding: 16px 24px 0;
  background: var(--bg1);
  border-bottom: 1px solid var(--bdr);
}

.sttPhTitleRow {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 14px;
}

.sttTabbar {
  display: flex;
  gap: 4px;
}

.sttTabBtn {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 2px;
  padding: 10px 18px;
  border: none;
  border-bottom: 2px solid transparent;
  background: transparent;
  cursor: pointer;
  transition: all 0.14s;

  &:hover { background: var(--bg2); }
}

.sttTabBtnActive {
  border-bottom-color: var(--blue);
  background: var(--bg2);
}

.sttTabLabel {
  font-size: 13px;
  font-weight: 600;
  color: var(--t0);
  font-family: var(--font-display);
}

.sttTabDesc {
  font-size: 10.5px;
  color: var(--t2);
  @include m.mono;
}

.sttTabPane {
  flex: 1;
  min-height: 0;
  display: flex;
  overflow: hidden;
}

// ── Inference tab body — sidebar + full-width InferencePanel ──
.sttInferTabBody {
  flex: 1;
  min-height: 0;
  display: flex;
  overflow: hidden;
}

.sttInferMain {
  flex: 1;
  min-height: 0;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

// ── Results tab body — sidebar + TranscriptDetail / empty state ──
.sttResultsTabBody {
  flex: 1;
  min-height: 0;
  display: flex;
  overflow: hidden;
}

.sttResultsMain {
  flex: 1;
  min-height: 0;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.sttResultsEmpty {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 8px;
  color: var(--t2);
  padding: 40px;
}

.sttResultsEmptyTitle {
  font-size: 14px;
  font-weight: 600;
  color: var(--t1);
}

.sttResultsEmptyHint {
  font-size: 12px;
  color: var(--t2);
  text-align: center;
  max-width: 320px;
}















{
  "_comment": "Add these keys inside the existing 'stt' object in en.json",
  "stt": {
    "tabs": {
      "upload": "Upload & Manage",
      "uploadDesc": "Add & browse your files",
      "infer": "Inference",
      "inferDesc": "Select files & run transcription",
      "results": "View Results",
      "resultsDesc": "Browse and edit transcripts"
    },
    "sidebar": {
      "pickForInference": "Pick files to transcribe",
      "pickToView": "Pick a file to view",
      "noCompletedFiles": "No completed files yet",
      "sort": {
        "id": "ID",
        "name": "Name",
        "date": "Date",
        "status": "Status"
      }
    },
    "results": {
      "noFileSelected": "No file selected",
      "noFileHint": "Click a completed file in the list to open its transcript here."
    }
  }
}

















{
  "_comment": "Add these keys inside the existing 'stt' object in ko.json",
  "stt": {
    "tabs": {
      "upload": "업로드 및 관리",
      "uploadDesc": "파일 추가 및 탐색",
      "infer": "추론",
      "inferDesc": "파일 선택 및 전사 실행",
      "results": "결과 보기",
      "resultsDesc": "전사본 탐색 및 편집"
    },
    "sidebar": {
      "pickForInference": "전사할 파일 선택",
      "pickToView": "볼 파일 선택",
      "noCompletedFiles": "아직 완료된 파일이 없습니다",
      "sort": {
        "id": "ID",
        "name": "이름",
        "date": "날짜",
        "status": "상태"
      }
    },
    "results": {
      "noFileSelected": "파일이 선택되지 않음",
      "noFileHint": "목록에서 완료된 파일을 클릭하면 여기에 전사본이 열립니다."
    }
  }
}
