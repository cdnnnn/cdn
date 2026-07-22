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
        // ── 1. STT Pipeline (page title) ──
        { target: 'stt-title',            title: t('stt.tour.titleTitle'),            content: t('stt.tour.titleBody'),            placement: 'bottom', onEnter: () => setActiveTab('upload') },

        // ── 2. Upload & Manage tab itself ──
        { target: 'tab-upload',           title: t('stt.tour.tabUploadTitle'),        content: t('stt.tour.tabUploadBody'),        placement: 'bottom' },

        // ── 3. Upload Files (right sidebar) ──
        { target: 'stt-upload',           title: t('stt.tour.uploadTitle'),           content: t('stt.tour.uploadBody'),           placement: 'right' },

        // ── 4. Library column heading ──
        { target: 'stt-library-heading',  title: t('stt.tour.libraryTitle'),          content: t('stt.tour.libraryBody'),          placement: 'bottom' },

        // ── 5. Sort By ──
        { target: 'stt-sort',             title: t('stt.tour.sortTitle'),             content: t('stt.tour.sortBody'),             placement: 'bottom' },

        // ── 6. Date Range ──
        { target: 'stt-date-filter',      title: t('stt.tour.dateFilterTitle'),       content: t('stt.tour.dateFilterBody'),       placement: 'bottom', onEnter: () => fileLibraryRef.current?.closePanel() },

        // ── 7. Status dropdown ──
        { target: 'stt-status-chips',     title: t('stt.tour.statusChipsTitle'),      content: t('stt.tour.statusChipsBody'),      placement: 'bottom', onEnter: () => fileLibraryRef.current?.closePanel() },

        // ── 8. Search Files ──
        { target: 'stt-action-search',    title: t('stt.tour.actionsTriggerTitle'),   content: t('stt.tour.actionsTriggerBody'),   placement: 'bottom', onEnter: () => fileLibraryRef.current?.closePanel() },

        // ── 9. Delete Files ──
        { target: 'stt-action-delete',    title: t('stt.tour.actionDeleteTitle'),     content: t('stt.tour.actionDeleteBody'),     placement: 'left',   onEnter: () => fileLibraryRef.current?.closePanel() },

        // ── 10. Move to Lecture Pipeline ──
        { target: 'stt-action-pipeline',  title: t('stt.tour.actionPipelineTitle'),   content: t('stt.tour.actionPipelineBody'),   placement: 'left',   onEnter: () => fileLibraryRef.current?.closePanel() },

        // ── 11. File cards ──
        { target: 'stt-library-body',     title: t('stt.tour.fileCardsTitle'),        content: t('stt.tour.fileCardsBody'),        placement: 'top',    onEnter: () => fileLibraryRef.current?.closePanel() },

        // ── 12. Inference tab itself ──
        { target: 'tab-infer',            title: t('stt.tour.tabInferTitle'),         content: t('stt.tour.tabInferBody'),         placement: 'bottom' },

        // ── 13. Inference tab contents (unchanged) ──
        { target: 'infer-status-filter', title: t('stt.tour.inferStatusTitle'), content: t('stt.tour.inferStatusBody'), placement: 'bottom', onEnter: () => setActiveTab('infer') },
        { target: 'infer-search',        title: t('stt.tour.inferSearchTitle'), content: t('stt.tour.inferSearchBody'), placement: 'bottom' },
        { target: 'infer-file-row',      title: t('stt.tour.inferFilesTitle'), content: t('stt.tour.inferFilesBody'),      placement: 'right' },
        { target: 'infer-file-empty',    title: t('stt.tour.inferFilesTitle'), content: t('stt.tour.inferFilesEmptyBody'), placement: 'right' },
        { target: 'infer-model-field',   title: t('stt.tour.inferModelTitle'),    content: t('stt.tour.inferModelBody'),    placement: 'right' },
        { target: 'infer-language-field',title: t('stt.tour.inferLanguageTitle'), content: t('stt.tour.inferLanguageBody'), placement: 'right' },
        { target: 'infer-chunk-field',   title: t('stt.tour.inferChunkTitle'),    content: t('stt.tour.inferChunkBody'),    placement: 'right' },
        { target: 'infer-run',           title: t('stt.tour.inferRunTitle'),      content: t('stt.tour.inferRunBody'),      placement: 'top' },

        // ── 14. View Results tab itself ──
        { target: 'tab-results',         title: t('stt.tour.tabResultsTitle'),    content: t('stt.tour.tabResultsBody'),    placement: 'bottom' },

        // ── 15. Results tab contents (unchanged) ──
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
                            data-tour={`tab-${tab.id}`}
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
