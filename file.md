// ═══════════════════════════════════════════════
// pages/SttTranscription/SttTranscription.tsx
//
// Top-level container — 3 persistent tabs (Upload & Manage / Inference /
// View Results), mirroring pages/UploadInfer/UploadInfer.tsx. All three
// tab panes stay mounted at all times (toggled via CSS display) so file
// lists, inference polling, and scroll position survive switching tabs
// instead of resetting.
// ═══════════════════════════════════════════════
import React, { useCallback, useEffect, useRef, useState } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import { selectSttFileViews, fetchSttFiles, clearSttFiles, type SttFileView } from '../../store/sttSlice';
import FileLibrary, { type FileLibraryHandle } from './components/FileLibrary';
import SttFileSidebar from './components/SttFileSidebar';
import InferencePanel from './components/InferencePanel';
import TranscriptDetail from './components/TranscriptDetail';
import TourGuide, { type TourStep } from './components/TourGuide';
import styles from './SttTranscription.module.scss';

type TabId = 'upload' | 'infer' | 'results';

const SttTranscription: React.FC = () => {
    const { t } = useTranslation();
    const dispatch = useAppDispatch();
    const files = useAppSelector(selectSttFileViews);

    const [activeTab, setActiveTab] = useState<TabId>('upload');

    // Refresh the file list (/stt/files/by-date) every time the user
    // switches tabs, so each tab always shows the latest server state
    // (e.g. a file finishing inference while the user was on another tab).
    // No args needed — the thunk falls back to the current dateFrom/dateTo
    // already held in state.
    const isFirstTabRender = React.useRef(true);
    useEffect(() => {
        if (isFirstTabRender.current) {
            // FileLibrary's own mount effect already fires the initial
            // /by-date call — skip firing a duplicate one here.
            //
            // The cleanup below reverts the flag rather than leaving it
            // false. React StrictMode (dev only) intentionally runs this
            // effect, its cleanup, then the effect again on mount to help
            // surface missing cleanup bugs — without reverting here, that
            // second invocation would see isFirstTabRender already false
            // and fire an extra, unwanted /by-date call on first load.
            isFirstTabRender.current = false;
            return () => { isFirstTabRender.current = true; };
        }
        dispatch(clearSttFiles());   // wipe stale list + flip filesLoading true so all 3 tabs show a spinner
        dispatch(fetchSttFiles());
    }, [activeTab, dispatch]);

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

    // Results tab sidebar collapse — lets the media player expand to fill
    // the space once a file is open. Resets whenever the user leaves the
    // Results tab, so navigating back in always shows the sidebar again
    // rather than remembering a collapse from a previous visit.
    const [resultsSidebarCollapsed, setResultsSidebarCollapsed] = useState(false);
    const toggleResultsSidebar = useCallback(() => setResultsSidebarCollapsed(v => !v), []);
    useEffect(() => {
        if (activeTab !== 'results') setResultsSidebarCollapsed(false);
    }, [activeTab]);

    const goToInfer = useCallback(() => setActiveTab('infer'), []);

    // ── Tour ── One continuous tour spanning all 3 tabs, driven by a
    // single TourGuide instance (see bottom of this component) so the
    // step counter ("N / total") never resets partway through — it only
    // resets when the whole tour is (re)started via startTour().
    //
    // FileLibrary no longer owns its own TourGuide; instead we hold a ref
    // to it so steps that live "under" its delete/pipeline panels can
    // close them before the next target is measured.
    const fileLibraryRef = useRef<FileLibraryHandle>(null);

    const [tourActive, setTourActive] = useState(false);
    // Whether the file-list step (mid-tour) should point at an actual row
    // or the empty state — decided once, when the tour starts, so a
    // mid-tour data change can't desync the step index from what's on screen.
    const [tourHasInferFiles, setTourHasInferFiles] = useState(false);
    const [tourHasResultsFiles, setTourHasResultsFiles] = useState(false);

    const startTour = useCallback(() => {
        setActiveTab('upload');
        setTourHasInferFiles(files.length > 0);
        setTourHasResultsFiles(files.some(f => f.status === 'completed'));
        setTourActive(true);
    }, [files]);

    const tourAllSteps: TourStep[] = [
        // ── Upload & Manage tab (formerly owned by FileLibrary itself) ──
        { target: 'stt-title',            title: t('stt.tour.titleTitle'),            content: t('stt.tour.titleBody'),            placement: 'bottom', onEnter: () => setActiveTab('upload') },
        { target: 'stt-date-filter',      title: t('stt.tour.dateFilterTitle'),       content: t('stt.tour.dateFilterBody'),       placement: 'bottom' },
        { target: 'stt-upload',           title: t('stt.tour.uploadTitle'),           content: t('stt.tour.uploadBody'),           placement: 'right' },
        { target: 'stt-library-heading',  title: t('stt.tour.libraryTitle'),          content: t('stt.tour.libraryBody'),          placement: 'bottom' },
        { target: 'stt-date-filter',      title: t('stt.tour.dateFilterTitle'),       content: t('stt.tour.dateFilterBody'),       placement: 'bottom', onEnter: () => fileLibraryRef.current?.closePanel() },
        { target: 'stt-status-chips',     title: t('stt.tour.statusChipsTitle'),      content: t('stt.tour.statusChipsBody'),      placement: 'bottom', onEnter: () => fileLibraryRef.current?.closePanel() },
        { target: 'stt-sort',             title: t('stt.tour.sortTitle'),             content: t('stt.tour.sortBody'),             placement: 'bottom', onEnter: () => fileLibraryRef.current?.closePanel() },
        { target: 'stt-action-search',    title: t('stt.tour.actionsTriggerTitle'),   content: t('stt.tour.actionsTriggerBody'),   placement: 'bottom', onEnter: () => fileLibraryRef.current?.closePanel() },
        { target: 'stt-action-delete',    title: t('stt.tour.actionDeleteTitle'),     content: t('stt.tour.actionDeleteBody'),     placement: 'left',   onEnter: () => fileLibraryRef.current?.closePanel() },
        { target: 'stt-action-pipeline',  title: t('stt.tour.actionPipelineTitle'),   content: t('stt.tour.actionPipelineBody'),   placement: 'left',   onEnter: () => fileLibraryRef.current?.closePanel() },
        { target: 'stt-library-body',     title: t('stt.tour.fileCardsTitle'),        content: t('stt.tour.fileCardsBody'),        placement: 'top' },

        // ── Inference tab ──
        { target: 'infer-status-filter', title: t('stt.tour.inferStatusTitle'), content: t('stt.tour.inferStatusBody'), placement: 'bottom', onEnter: () => setActiveTab('infer') },
        { target: 'infer-search',        title: t('stt.tour.inferSearchTitle'), content: t('stt.tour.inferSearchBody'), placement: 'bottom' },
        { target: 'infer-file-row',      title: t('stt.tour.inferFilesTitle'), content: t('stt.tour.inferFilesBody'),      placement: 'right' },
        { target: 'infer-file-empty',    title: t('stt.tour.inferFilesTitle'), content: t('stt.tour.inferFilesEmptyBody'), placement: 'right' },
        { target: 'infer-model-field',   title: t('stt.tour.inferModelTitle'),    content: t('stt.tour.inferModelBody'),    placement: 'right' },
        { target: 'infer-language-field',title: t('stt.tour.inferLanguageTitle'), content: t('stt.tour.inferLanguageBody'), placement: 'right' },
        { target: 'infer-chunk-field',   title: t('stt.tour.inferChunkTitle'),    content: t('stt.tour.inferChunkBody'),    placement: 'right' },
        { target: 'infer-run',           title: t('stt.tour.inferRunTitle'),      content: t('stt.tour.inferRunBody'),      placement: 'top' },

        // ── Results tab ──
        { target: 'results-search',      title: t('stt.tour.resultsSearchTitle'), content: t('stt.tour.resultsSearchBody'), placement: 'bottom', onEnter: () => setActiveTab('results') },
        { target: 'results-sort',        title: t('stt.tour.resultsSortTitle'),   content: t('stt.tour.resultsSortBody'),   placement: 'bottom' },
        { target: 'results-file-row',    title: t('stt.tour.resultsFilesTitle'), content: t('stt.tour.resultsFilesBody'),      placement: 'right' },
        { target: 'results-file-empty',  title: t('stt.tour.resultsFilesTitle'), content: t('stt.tour.resultsFilesEmptyBody'), placement: 'right' },
        { target: 'results-content',     title: t('stt.tour.resultsContentTitle'), content: t('stt.tour.resultsContentBody'), placement: 'left' },
    ];

    // Only one of each file-list pair makes sense to show — drop whichever
    // doesn't match what's actually on screen right now.
    const tourSteps = tourAllSteps.filter(s => {
        if (s.target === 'infer-file-row')     return tourHasInferFiles;
        if (s.target === 'infer-file-empty')   return !tourHasInferFiles;
        if (s.target === 'results-file-row')   return tourHasResultsFiles;
        if (s.target === 'results-file-empty') return !tourHasResultsFiles;
        return true;
    });

    const tabs: { id: TabId; label: string; desc: string }[] = [
        { id: 'upload',  label: t('stt.tabs.upload'),  desc: t('stt.tabs.uploadDesc') },
        { id: 'infer',   label: t('stt.tabs.infer'),   desc: t('stt.tabs.inferDesc') },
        { id: 'results', label: t('stt.tabs.results'), desc: t('stt.tabs.resultsDesc') },
    ];

    return (
        <div className={styles.sttPage}>
            {/* ── Header — title and self-explanatory tab cards share one row ── */}
            <div className={styles.sttHeaderBar}>
                <div className={styles.phTitleRow}>
                    <div className={styles.phTitle} data-tour="stt-title">{t('stt.pageTitle')}</div>
                    <button type="button" className={styles.tourTriggerBtn} onClick={startTour}>
                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                            <circle cx="8" cy="8" r="6.25" />
                            <path d="M6.1 6.2a1.9 1.9 0 013.6.7c0 1.3-1.7 1.5-1.7 2.7M8 11.4v.1" />
                        </svg>
                        {t('stt.tour.takeTour')}
                    </button>
                </div>

                <div className={styles.tabbar} role="tablist" data-tour="stt-tabbar">
                    {tabs.map(tab => (
                        <button
                            key={tab.id}
                            type="button"
                            role="tab"
                            aria-selected={activeTab === tab.id}
                            className={`${styles.tabBtn} ${activeTab === tab.id ? styles.tabBtnActive : ''}`}
                            onClick={() => setActiveTab(tab.id)}
                        >
                            <span className={styles.tabLabel}>{tab.label}</span>
                            <span className={styles.tabDesc}>{tab.desc}</span>
                        </button>
                    ))}
                </div>
            </div>

            {/* ── Tab 1: Upload & Manage ── */}
            <div className={styles.sttTabPane} style={{ display: activeTab === 'upload' ? 'flex' : 'none' }}>
                <FileLibrary
                    ref={fileLibraryRef}
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
                            active={activeTab === 'infer'}
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
                        collapsed={resultsSidebarCollapsed}
                    />
                    <div className={styles.sttResultsMain} data-tour="results-content">
                        {activeFile ? (
                            <TranscriptDetail
                                file={activeFile}
                                sidebarCollapsed={resultsSidebarCollapsed}
                                onToggleSidebar={toggleResultsSidebar}
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

            <TourGuide steps={tourSteps} active={tourActive} onFinish={() => setTourActive(false)} />
        </div>
    );
};

export default SttTranscription;



















// ═══════════════════════════════════════════════
// pages/SttTranscription/components/FileLibrary.tsx
// ═══════════════════════════════════════════════
import React, { useEffect, useMemo, useState, useCallback, forwardRef, useImperativeHandle } from 'react';
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
import styles from '../SttTranscription.module.scss';
import sttApi from '../../../services/sttApi';

interface Props {
    onGoToInfer: () => void; // switch the parent tab bar to the Inference tab
    active: boolean;         // whether this tab is currently visible (for tour targeting)
}

// Imperative handle exposed to the parent (SttTranscription.tsx) — the tour
// now lives entirely there (single continuous TourGuide instance), but a
// few of its steps still need to close whatever panel (delete/pipeline) is
// open in here before measuring the next target.
export interface FileLibraryHandle {
    closePanel: () => void;
}

// Panel modes — only one can be active at a time (inference selection now lives in the Inference tab)
type PanelMode = 'none' | 'delete' | 'pipeline';

// ── Icons ──────────────────────────────────────
const IconCalendar: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <rect x="2" y="3.5" width="12" height="10" rx="1.5" /><path d="M2 6h12M5.5 2v3M10.5 2v3" />
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

// Fixed 4-button sliding window around the current page — e.g. current=57,
// total=200 → [56, 57, 58, 59]. Keeps the row compact and predictable
// instead of a variable count with ellipsis gaps. Mirrors pages/UploadInfer/FilePanel.tsx.
const PAGE_WINDOW = 4;
const getPageNumbers = (current: number, total: number): number[] => {
    if (total <= PAGE_WINDOW) return Array.from({ length: total }, (_, i) => i + 1);
    const start = Math.max(1, Math.min(current - 1, total - PAGE_WINDOW + 1));
    return Array.from({ length: PAGE_WINDOW }, (_, i) => start + i);
};
const PAGE_SIZE_OPTIONS = [50, 75, 100];

const FileLibrary = forwardRef<FileLibraryHandle, Props>(({ onGoToInfer, active }, ref) => {
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

    // Let the parent's single TourGuide instance reset panelMode before
    // measuring steps like stt-date-filter / stt-status-chips / stt-sort,
    // which sit behind whichever panel happens to be open.
    useImperativeHandle(ref, () => ({ closePanel }), [closePanel]);

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

    // ── Sort (client-side, applied to the library array before date-grouping) ──
    type SortKey = 'id' | 'name' | 'date' | 'status';
    const [sortKey, setSortKey] = useState<SortKey>('date');
    const [sortDir, setSortDir] = useState<'asc' | 'desc'>('desc');
    const handleSort = (key: SortKey) => {
        if (sortKey === key) setSortDir(d => (d === 'asc' ? 'desc' : 'asc'));
        else { setSortKey(key); setSortDir('asc'); }
    };

    // ── Pagination (client-side, applied after search/status filtering) ──
    const [filesPage, setFilesPage] = useState(1);
    const [filesPageSize, setFilesPageSize] = useState(PAGE_SIZE_OPTIONS[0]);
    const goToPage = (p: number) => setFilesPage(Math.max(1, p));
    const handlePageSizeChange = (size: number) => { setFilesPageSize(size); setFilesPage(1); };

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

    // Guards the one-time initial load below against React StrictMode's
    // dev-only double-invocation of mount effects (run effect → cleanup →
    // run again, to help surface missing-cleanup bugs). Without this,
    // fetchInferenceProgress and fetchSttFiles below each fired twice on
    // first navigation to this page. FileLibrary is a persistent tab pane
    // that's never actually unmounted/remounted during normal use, so a
    // plain (non-reverting) ref is correct here — this really should only
    // ever run once.
    const hasFetchedInitial = React.useRef(false);
    useEffect(() => {
        if (hasFetchedInitial.current) return;
        hasFetchedInitial.current = true;

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
        const processing = files.filter(f => f.status === 'queued' || f.status === 'running');
        const dir = sortDir === 'asc' ? 1 : -1;
        const cmp = (a: SttFileView, b: SttFileView): number => {
            if (sortKey === 'id')     return (a.id - b.id) * dir;
            if (sortKey === 'name')   return a.original_name.localeCompare(b.original_name) * dir;
            if (sortKey === 'status') return a.status.localeCompare(b.status) * dir;
            return (new Date(a.inserted_at).getTime() - new Date(b.inserted_at).getTime()) * dir;
        };
        const library = [...files].sort(cmp);
        return { processing, library };
    }, [files, sortKey, sortDir]);

    const completedFiles = library.filter(f => f.status === 'completed');
    const visibleLibrary = library.filter(f => matchesSearch(f) && matchesStatus(f));

    // Reset to page 1 whenever the underlying result set changes shape,
    // so the user never lands on a now-empty trailing page.
    useEffect(() => { setFilesPage(1); }, [normalizedQuery, statusFilter, sortKey, sortDir, dateFrom, dateTo]);

    const filesTotal      = visibleLibrary.length;
    const filesTotalPages = Math.max(1, Math.ceil(filesTotal / filesPageSize));
    const currentPage      = Math.min(filesPage, filesTotalPages);
    const paginatedLibrary = visibleLibrary.slice((currentPage - 1) * filesPageSize, currentPage * filesPageSize);

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

            {/* ── Three-column body ── */}
            <div className={styles.libBody}>
                {/* COL 1: Upload */}
                <UploadInline />

                {/* COL 2: Library */}
                <div className={styles.libCol}>
                    <div className={styles.colHeading} data-tour="stt-library-heading">
                        <span className={styles.colHeadingIc}><IconLibrary /></span>
                        <span className={styles.colHeadingText}>{t('stt.library.title')}</span>
                        <span className={styles.colHeadingSub}>
                            {(initializing || filesLoading)
                                ? t('stt.loading')
                                : `${t('stt.file', { count: files.length })}${processing.length > 0 ? t('stt.processing', { count: processing.length }) : ''}`}
                        </span>

                        {/* Sort + search — inline with the title, same arrangement as FilePanel.tsx's section2TitleRow */}
                        <div className={styles.sortHeader} data-tour="stt-sort">
                            <span className={styles.sortHeaderLabel}>
                                <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                                    <path d="M2 4h10M4 7h6M6 10h2" />
                                </svg>
                                {t('stt.sortBy')}
                            </span>
                            <div className={styles.sortCols}>
                                {([
                                    { key: 'id' as SortKey,     label: t('stt.sidebar.sort.id') },
                                    { key: 'name' as SortKey,   label: t('stt.sidebar.sort.name') },
                                    { key: 'date' as SortKey,   label: t('stt.sidebar.sort.date') },
                                    { key: 'status' as SortKey, label: t('stt.sidebar.sort.status') },
                                ]).map(({ key, label }) => (
                                    <button
                                        key={key}
                                        className={`${styles.sortCol} ${sortKey === key ? styles.sortColActive : ''}`}
                                        onClick={() => handleSort(key)}
                                    >
                                        {label}
                                        <svg viewBox="0 0 10 12" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round"
                                            className={sortKey === key ? (sortDir === 'asc' ? styles.sortAsc : styles.sortDesc) : styles.sortInactive}>
                                            <path d="M5 1v10M2 8l3 3 3-3" />
                                        </svg>
                                    </button>
                                ))}
                            </div>
                        </div>

                        <span className={styles.vSep} aria-hidden="true" />

                        {/* Date filter — now lives in the title row, same as FilePanel.tsx's section2TitleRow */}
                        <div className={styles.dateFilter} data-tour="stt-date-filter">
                            <svg className={styles.dateIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                <rect x="2" y="3" width="12" height="11" rx="1.5" /><path d="M2 6.5h12M5 2v2.5M11 2v2.5" />
                            </svg>
                            <span className={styles.filterBarLabel}>{t('stt.dateRangeLabel')}</span>
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
                                <button className={styles.applyBtn} onClick={handleApply} disabled={initializing || filesLoading}>{t('stt.apply')}</button>
                            )}
                        </div>
                        {dateError && <span className={styles.dateValidationError}>{dateError}</span>}

                        <div className={styles.sectionHeaderRight}>
                            {searchOpen && (
                                <div className={styles.searchBox}>
                                    <span className={styles.searchBoxIc}><IconSearch /></span>
                                    <input className={styles.searchBoxInput} type="text" placeholder={t('stt.library.searchPlaceholder')}
                                        value={searchQuery} onChange={e => setSearchQuery(e.target.value)} autoFocus
                                        onKeyDown={e => { if (e.key === 'Escape') { setSearchOpen(false); setSearchQuery(''); } }} />
                                    <button
                                        className={styles.searchBoxClear}
                                        onClick={() => { if (searchQuery) setSearchQuery(''); else setSearchOpen(false); }}
                                        title={searchQuery ? t('stt.library.clear') : t('stt.library.closeSearch')}
                                    >
                                        <IconClose />
                                    </button>
                                </div>
                            )}
                        </div>
                    </div>

                    {/* ── Filter row — status filter + actions, right-aligned as one group (mirrors FilePanel.tsx's .section2 filterSortRow/filterBarRow exactly; sort + date range now live in the title row above) ── */}
                    <div className={styles.filterSortRow}>
                        <div className={styles.filterBarRow}>
                            {/* Status filter + Actions — grouped on the right */}
                            <div className={styles.filterBarRight}>
                                <div className={styles.statusFilterWrap} data-tour="stt-status-chips">
                                    <svg className={styles.dateIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                        <path d="M2 4h12M4.5 8h7M7 12h2" />
                                    </svg>
                                    <span className={styles.filterBarLabel}>{t('stt.statusLabel')}</span>
                                    <select
                                        className={styles.statusFilterSelect}
                                        value={statusFilter}
                                        disabled={initializing || filesLoading}
                                        onChange={e => setStatusFilter(e.target.value as StatusFilter)}
                                    >
                                        {(['all', 'completed', 'pending', 'queued', 'running'] as const).map(s => (
                                            <option key={s} value={s}>{s === 'all' ? t('stt.library.filterAll') : t(`stt.fileCard.status.${s}`)}</option>
                                        ))}
                                    </select>
                                </div>

                                {/* Actions — shown inline as a labeled button row, matching FilePanel.tsx's actionsInlineWrap */}
                                {panelMode === 'none' && !inferenceRunning && (
                                    <>
                                        <span className={styles.vSep} aria-hidden="true" />
                                        <div className={styles.actionsInlineWrap} data-tour="stt-actions">
                                            <span className={styles.filterBarLabel}>{t('stt.actionsLabel')}</span>

                                            <button
                                                data-tour="stt-action-search"
                                                className={`${styles.actionTile} ${styles.actionTileSearch} ${searchOpen ? styles.actionTileActive : ''}`}
                                                onClick={() => { setSearchOpen(v => !v); if (searchOpen) setSearchQuery(''); }}
                                                title={t('stt.library.searchBtn')}
                                            >
                                                <IconSearch />
                                                <span className={styles.actionTileLabel}>{t('stt.library.searchBtn')}</span>
                                            </button>
                                            <button
                                                data-tour="stt-action-delete"
                                                className={`${styles.actionTile} ${styles.actionTileDelete}`}
                                                onClick={() => openPanel('delete')}
                                                title={t('stt.library.deleteFiles')}
                                            >
                                                <IconDelete />
                                                <span className={styles.actionTileLabel}>{t('stt.library.deleteFiles')}</span>
                                            </button>
                                            <button
                                                data-tour="stt-action-pipeline"
                                                className={`${styles.actionTile} ${styles.actionTilePipeline}`}
                                                onClick={() => openPanel('pipeline')}
                                                title={t('stt.library.movePipeline')}
                                            >
                                                <IconPipeline />
                                                <span className={styles.actionTileLabel}>{t('stt.library.movePipeline')}</span>
                                            </button>
                                        </div>
                                    </>
                                )}
                            </div>
                        </div>
                    </div>

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
                            <div className={styles.libCardGrid}>
                                {paginatedLibrary.map(f => {
                                    const selectable = isSelectionMode && canSelectFile(f);
                                    const dimmed     = isSelectionMode && !canSelectFile(f);
                                    const handleCardOpen = () => {
                                        if (selectable) toggleFileSelect(f.id);
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
                        )}
                    </div>

                    {/* Pagination footer — same design/behavior as pages/UploadInfer/FilePanel.tsx */}
                    {!filesLoading && !initializing && !filesError && filesTotal > 0 && (
                        <div className={styles.paginationBar}>
                            <div className={styles.paginationTopRow}>
                                <div className={styles.pageSizeGroup}>
                                    <span className={styles.pageSizeLabel}>{t('stt.library.perPage')}</span>
                                    <select
                                        className={styles.pageSizeSelect}
                                        value={filesPageSize}
                                        onChange={e => handlePageSizeChange(Number(e.target.value))}
                                    >
                                        {PAGE_SIZE_OPTIONS.map(size => (
                                            <option key={size} value={size}>{size}</option>
                                        ))}
                                    </select>
                                </div>

                                <div className={styles.pageNav}>
                                    <button
                                        className={styles.pageNavBtn}
                                        onClick={() => goToPage(currentPage - 1)}
                                        disabled={currentPage <= 1}
                                        aria-label="Previous page"
                                    >
                                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                                            <path d="M10 3L6 8l4 5" />
                                        </svg>
                                    </button>

                                    {(() => {
                                        const pageWindow = getPageNumbers(currentPage, filesTotalPages);
                                        const showFirst  = pageWindow[0] > 1;
                                        const showLast   = pageWindow[pageWindow.length - 1] < filesTotalPages;
                                        return (
                                            <>
                                                {showFirst && (
                                                    <>
                                                        <button className={styles.pageNumBtn} onClick={() => goToPage(1)}>1</button>
                                                        {pageWindow[0] > 2 && <span className={styles.pageEllipsis}>…</span>}
                                                    </>
                                                )}
                                                {pageWindow.map(p => (
                                                    <button
                                                        key={p}
                                                        className={`${styles.pageNumBtn} ${p === currentPage ? styles.pageNumBtnActive : ''}`}
                                                        onClick={() => goToPage(p)}
                                                    >
                                                        {p}
                                                    </button>
                                                ))}
                                                {showLast && (
                                                    <>
                                                        {pageWindow[pageWindow.length - 1] < filesTotalPages - 1 && <span className={styles.pageEllipsis}>…</span>}
                                                        <button className={styles.pageNumBtn} onClick={() => goToPage(filesTotalPages)}>{filesTotalPages}</button>
                                                    </>
                                                )}
                                            </>
                                        );
                                    })()}

                                    <button
                                        className={styles.pageNavBtn}
                                        onClick={() => goToPage(currentPage + 1)}
                                        disabled={currentPage >= filesTotalPages}
                                        aria-label="Next page"
                                    >
                                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                                            <path d="M6 3l4 5-4 5" />
                                        </svg>
                                    </button>
                                </div>
                            </div>

                            <div className={styles.pageInfo}>
                                {t('stt.library.pageInfo', { page: currentPage, totalPages: filesTotalPages, total: filesTotal })}
                            </div>
                        </div>
                    )}
                </div>
            </div>
        </div>
    );
});

FileLibrary.displayName = 'FileLibrary';

export default FileLibrary;
