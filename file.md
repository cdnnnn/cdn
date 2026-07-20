// ═══════════════════════════════════════════════
// pages/SttTranscription/SttTranscription.tsx
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
import { selectSttFileViews, fetchSttFiles, clearSttFiles, type SttFileView } from '../../store/sttSlice';
import FileLibrary from './components/FileLibrary';
import SttFileSidebar from './components/SttFileSidebar';
import InferencePanel from './components/InferencePanel';
import TranscriptDetail from './components/TranscriptDetail';
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
            isFirstTabRender.current = false;
            return;
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

    // ── Tour ── The tour's steps all target elements inside the Upload &
    // Manage tab, so starting it also switches to that tab.
    const [tourActive, setTourActive] = useState(false);
    const startTour = useCallback(() => { setActiveTab('upload'); setTourActive(true); }, []);

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
                    onGoToInfer={goToInfer}
                    active={activeTab === 'upload'}
                    tourActive={tourActive}
                    onTourFinish={() => setTourActive(false)}
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
        </div>
    );
};

export default SttTranscription;
