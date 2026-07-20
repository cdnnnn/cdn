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

    // ── Tour, stage 1 ── Covers the Upload & Manage tab; owned by
    // FileLibrary itself since several steps need to open/close its own
    // internal popovers. Starting it also switches to that tab.
    const [tourActive, setTourActive] = useState(false);
    const startTour = useCallback(() => { setActiveTab('upload'); setTourActive(true); }, []);

    // ── Tour, stage 2 ── Covers the Inference and Results tabs. Chained to
    // start automatically the moment stage 1 finishes, so clicking "Next"
    // past the last Upload & Manage step carries the user straight into
    // the Inference tab rather than ending the tour there. Whether the
    // file-list step points at an actual row or the empty state is decided
    // once, at hand-off, so a mid-tour data change can't desync the step
    // index from what's on screen.
    const [postTourActive, setPostTourActive] = useState(false);
    const [tourHasInferFiles, setTourHasInferFiles] = useState(false);
    const [tourHasResultsFiles, setTourHasResultsFiles] = useState(false);
    const handleUploadTourFinish = useCallback(() => {
        setTourActive(false);
        setTourHasInferFiles(files.length > 0);
        setTourHasResultsFiles(files.some(f => f.status === 'completed'));
        setActiveTab('infer');
        setPostTourActive(true);
    }, [files]);

    const postTourAllSteps: TourStep[] = [
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
    const postTourSteps = postTourAllSteps.filter(s => {
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
                    onGoToInfer={goToInfer}
                    active={activeTab === 'upload'}
                    tourActive={tourActive}
                    onTourFinish={handleUploadTourFinish}
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

            <TourGuide steps={postTourSteps} active={postTourActive} onFinish={() => setPostTourActive(false)} />
        </div>
    );
};

export default SttTranscription;













// ═══════════════════════════════════════════════
// pages/SttTranscription/components/SttFileSidebar.tsx
//
// Reusable file-picker column used by both the Inference tab
// (mode="select" — checkbox multi-select feeding InferencePanel) and
// the View Results tab (mode="view" — click a completed file to open
// its transcript). Has its own search / status filter, independent of
// whatever filter state the Upload & Manage tab is using.
// ═══════════════════════════════════════════════
import React, { useEffect, useMemo, useState } from 'react';
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
    // When true, collapses this column to 0 width (state/filters stay intact,
    // just visually hidden) — used by the Results tab so the transcript's
    // media player can expand to fill the space. Defaults to false.
    collapsed?: boolean;
}

const IconSearch: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <circle cx="6.5" cy="6.5" r="4" /><path d="M11 11l2.5 2.5" />
    </svg>
);
const IconFilter: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <path d="M2 4h12M4.5 8h7M7 12h2" />
    </svg>
);
const IconCheck: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
        <path d="M3 8.5l3.5 3.5 7-7.5" />
    </svg>
);
const IconSort: React.FC = () => (
    <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
        <path d="M2 4h10M4 7h6M6 10h2" />
    </svg>
);
const IconSortArrow: React.FC = () => (
    <svg viewBox="0 0 10 12" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <path d="M5 1v10M2 8l3 3 3-3" />
    </svg>
);

type SortKey = 'id' | 'name' | 'date' | 'status';

// Fixed 4-button sliding window around the current page — same pagination
// treatment as pages/SttTranscription/components/FileLibrary.tsx.
const PAGE_WINDOW = 4;
const getPageNumbers = (current: number, total: number): number[] => {
    if (total <= PAGE_WINDOW) return Array.from({ length: total }, (_, i) => i + 1);
    const start = Math.max(1, Math.min(current - 1, total - PAGE_WINDOW + 1));
    return Array.from({ length: PAGE_WINDOW }, (_, i) => start + i);
};
const PAGE_SIZE_OPTIONS = [50, 75, 100];

const SttFileSidebar: React.FC<Props> = ({
    mode, active,
    selectedIds, onToggleSelect, onSelectAll, onClearSelection,
    activeFileId, onFileClick,
    onlyCompleted = false,
    collapsed = false,
}) => {
    const { t } = useTranslation();
    const files = useAppSelector(selectSttFileViews);
    const filesLoading = useAppSelector(s => s.stt.filesLoading);

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

    // ── Pagination (client-side, applied after search/status filter/sort) ──
    const [page, setPage]         = useState(1);
    const [pageSize, setPageSize] = useState(PAGE_SIZE_OPTIONS[0]);
    const goToPage = (p: number) => setPage(Math.max(1, p));
    const handlePageSizeChange = (size: number) => { setPageSize(size); setPage(1); };

    // Reset to page 1 whenever the filtered result set changes shape, so
    // the user never lands on a now-empty trailing page.
    useEffect(() => { setPage(1); }, [search, status, sortKey, sortAsc, mode, onlyCompleted]);

    const totalPages   = Math.max(1, Math.ceil(filtered.length / pageSize));
    const currentPage  = Math.min(page, totalPages);
    const paginated    = filtered.slice((currentPage - 1) * pageSize, currentPage * pageSize);

    return (
        <div className={`${styles.sttSidebar} ${collapsed ? styles.sttSidebarCollapsed : ''}`} data-tour={mode === 'select' ? 'infer-sidebar' : 'results-sidebar'}>
            <div className={styles.sttSidebarHeader}>
                <span className={styles.sttSidebarTitle}>
                    {mode === 'select' ? t('stt.sidebar.pickForInference') : t('stt.sidebar.pickToView')}
                </span>
                <span className={styles.sttSidebarCount}>{filtered.length}</span>
            </div>

            {/* Status filter + Search — same row, same design as the Upload tab / reference sidebar */}
            <div className={styles.sttSidebarFilterRow}>
                {statusOptions.length > 1 && (
                    <div className={styles.sttSidebarStatusWrap} data-tour={mode === 'select' ? 'infer-status-filter' : 'results-status-filter'}>
                        <IconFilter />
                        <select
                            className={styles.sttSidebarStatusSelect}
                            value={status}
                            disabled={filesLoading}
                            onChange={e => setStatus(e.target.value as typeof status)}
                        >
                            {statusOptions.map(s => (
                                <option key={s} value={s}>{s === 'all' ? t('stt.library.filterAll') : t(`stt.fileCard.status.${s}`)}</option>
                            ))}
                        </select>
                    </div>
                )}

                <div className={styles.sttSidebarSearchWrap} data-tour={mode === 'select' ? 'infer-search' : 'results-search'}>
                    <IconSearch />
                    <input
                        type="text"
                        className={styles.sttSidebarSearchInput}
                        value={search}
                        onChange={e => setSearch(e.target.value)}
                        placeholder={t('stt.library.searchPlaceholder')}
                    />
                </div>
            </div>

            {/* Sort — boxed pill row with arrow-direction icons, same design as the Upload tab / reference sidebar */}
            <div className={styles.sttSidebarSortHeader} data-tour={mode === 'select' ? 'infer-sort' : 'results-sort'}>
                <span className={styles.sttSidebarSortHeaderLabel}>
                    <IconSort />
                    {t('stt.sortBy')}
                </span>
                <div className={styles.sttSidebarSortCols}>
                    {(['id', 'name', 'date', 'status'] as SortKey[]).map(k => (
                        <button
                            key={k}
                            className={`${styles.sttSidebarSortCol} ${sortKey === k ? styles.sttSidebarSortColActive : ''}`}
                            onClick={() => toggleSort(k)}
                            disabled={filesLoading}
                        >
                            {t(`stt.sidebar.sort.${k}`)}
                            <span className={sortKey === k ? (sortAsc ? styles.sttSidebarSortAsc : styles.sttSidebarSortDesc) : styles.sttSidebarSortInactive}>
                                <IconSortArrow />
                            </span>
                        </button>
                    ))}
                </div>
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
                {filesLoading ? (
                    <div className={styles.libLoading}>
                        <div className={styles.spinner} />
                        <span>{t('stt.library.loading')}</span>
                    </div>
                ) : filtered.length === 0 ? (
                    <div className={styles.sttSidebarEmpty} data-tour={mode === 'select' ? 'infer-file-empty' : 'results-file-empty'}>
                        {mode === 'view' && onlyCompleted
                            ? t('stt.sidebar.noCompletedFiles')
                            : t('stt.library.emptyTitle')}
                    </div>
                ) : paginated.map((f, idx) => {
                    const isSelected = mode === 'select' && selectedIds?.has(f.id);
                    const isActive   = mode === 'view' && activeFileId === f.id;
                    const clickable  = mode === 'select' || (mode === 'view' && f.status === 'completed');
                    return (
                        <button
                            key={f.id}
                            type="button"
                            data-tour={idx === 0 ? (mode === 'select' ? 'infer-file-row' : 'results-file-row') : undefined}
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
                            <div className={styles.sttSidebarRowInfo}>
                                <div className={styles.sttSidebarRowName} title={f.original_name}>{f.original_name}</div>
                                <div className={styles.sttSidebarRowMeta}>{f.inserted_at} · #{f.id}</div>
                            </div>
                            <span className={`${styles.pill} ${
                                isSelected               ? styles.pillBlue :
                                f.status === 'completed' ? styles.pillGreen :
                                f.status === 'queued'    ? styles.pillAmber :
                                f.status === 'running'   ? styles.pillBlue :
                                f.status === 'failed'    ? styles.pillRed :
                                styles.pillGray
                            }`}>
                                {isSelected ? (
                                    <svg viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" width="10" height="10">
                                        <path d="M2 6l3 3 5-5" />
                                    </svg>
                                ) : (
                                    <>
                                        {f.status === 'completed' && (
                                            <svg viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" width="9" height="9">
                                                <path d="M2 6l3 3 5-5" />
                                            </svg>
                                        )}
                                        {(f.status === 'running' || f.status === 'queued') && (
                                            <span className={styles.pillDot} />
                                        )}
                                        {t(`stt.fileCard.status.${f.status}`)}
                                    </>
                                )}
                            </span>
                        </button>
                    );
                })}
            </div>

            {/* Pagination footer — same client-side behavior/controls as FileLibrary.tsx, compact for the sidebar column */}
            {!filesLoading && filtered.length > 0 && (
                <div className={styles.sttSidebarPagination}>
                    <div className={styles.sttSidebarPaginationTopRow}>
                        <select
                            className={styles.sttSidebarPageSizeSelect}
                            value={pageSize}
                            onChange={e => handlePageSizeChange(Number(e.target.value))}
                        >
                            {PAGE_SIZE_OPTIONS.map(size => (
                                <option key={size} value={size}>{t('stt.library.perPageShort', { count: size })}</option>
                            ))}
                        </select>

                        <div className={styles.sttSidebarPageNav}>
                            <button
                                className={styles.sttSidebarPageNavBtn}
                                onClick={() => goToPage(currentPage - 1)}
                                disabled={currentPage <= 1}
                                aria-label="Previous page"
                            >
                                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                                    <path d="M10 3L6 8l4 5" />
                                </svg>
                            </button>

                            {(() => {
                                const pageWindow = getPageNumbers(currentPage, totalPages);
                                const showFirst  = pageWindow[0] > 1;
                                const showLast   = pageWindow[pageWindow.length - 1] < totalPages;
                                return (
                                    <>
                                        {showFirst && (
                                            <>
                                                <button className={styles.sttSidebarPageNumBtn} onClick={() => goToPage(1)}>1</button>
                                                {pageWindow[0] > 2 && <span className={styles.sttSidebarPageEllipsis}>…</span>}
                                            </>
                                        )}
                                        {pageWindow.map(p => (
                                            <button
                                                key={p}
                                                className={`${styles.sttSidebarPageNumBtn} ${p === currentPage ? styles.sttSidebarPageNumBtnActive : ''}`}
                                                onClick={() => goToPage(p)}
                                            >
                                                {p}
                                            </button>
                                        ))}
                                        {showLast && (
                                            <>
                                                {pageWindow[pageWindow.length - 1] < totalPages - 1 && <span className={styles.sttSidebarPageEllipsis}>…</span>}
                                                <button className={styles.sttSidebarPageNumBtn} onClick={() => goToPage(totalPages)}>{totalPages}</button>
                                            </>
                                        )}
                                    </>
                                );
                            })()}

                            <button
                                className={styles.sttSidebarPageNavBtn}
                                onClick={() => goToPage(currentPage + 1)}
                                disabled={currentPage >= totalPages}
                                aria-label="Next page"
                            >
                                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                                    <path d="M6 3l4 5-4 5" />
                                </svg>
                            </button>
                        </div>
                    </div>
                    <div className={styles.sttSidebarPageInfo}>
                        {t('stt.library.pageInfo', { page: currentPage, totalPages, total: filtered.length })}
                    </div>
                </div>
            )}
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
    // Whether the Inference tab is the one currently showing — triggers a
    // fresh /get_audio_models call every time the user navigates here,
    // not just on first mount.
    active: boolean;
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
const IconRefresh: React.FC<{ className?: string }> = ({ className }) => (
    <svg className={className} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round" strokeLinejoin="round">
        <path d="M13.5 8A5.5 5.5 0 1 1 10 3.07" />
        <path d="M10 2v3h3" />
    </svg>
);
const IconWarn: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <path d="M8 2L14 13H2L8 2z" /><path d="M8 7v3M8 11.5v.1" />
    </svg>
);
const IconOk: React.FC = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <circle cx="8" cy="8" r="5.5" /><path d="M5.5 8l2 2 3-3" />
    </svg>
);
const IconRun: React.FC = () => (
    <svg width="15" height="15" viewBox="0 0 16 16" aria-hidden="true">
        <path d="M5 3l8 5-8 5V3z" fill="currentColor" />
    </svg>
);

const CHUNK_OPTIONS = [
    { value: 'auto', labelKey: 'stt.inference.chunkAuto' },
    { value: '5',    labelKey: '5s'  },
    { value: '10',   labelKey: '10s' },
    { value: '15',   labelKey: '15s' },
    { value: '20',   labelKey: '20s' },
];
const POLL_INTERVAL = 10_000;

const InferencePanel: React.FC<Props> = ({ selectedIds, onSubmitted, onCompleted, active }) => {
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
    const [chunk, setChunk]   = useState('auto');

    const pollRef      = useRef<ReturnType<typeof setInterval> | null>(null);
    const countRef     = useRef<ReturnType<typeof setInterval> | null>(null);
    const [countdown, setCountdown] = useState(0);

    // Fresh model list every time the user navigates to this tab — not
    // just on first mount, so a model added/removed server-side while the
    // user was elsewhere shows up without needing a manual refresh.
    useEffect(() => {
        if (active) dispatch(fetchSttModels());
        // eslint-disable-next-line react-hooks/exhaustive-deps
    }, [active, dispatch]);

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
            chunkSize: chunk === 'auto' ? 0 : Number(chunk),
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

    const canRunInference = selectedIds.size > 0 && !!model && !submitting;
    const runBtnSubLabel = (() => {
        if (selectedIds.size === 0) return 'Select files first';
        if (!model) return 'No model selected';
        return `${selectedIds.size} file${selectedIds.size !== 1 ? 's' : ''} ready`;
    })();

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

                    <div className={styles.infSection}>
                        <div className={styles.infSectionTitle}>Settings</div>

                        <div className={`${styles.infField} ${styles.infModelField}`} data-tour="infer-model-field">
                            <div className={styles.infModelLabelRow}>
                                <label className={styles.infLabel}>
                                    Model {modelsLoading && <span className={styles.loadingDot}>…</span>}
                                </label>
                                <button
                                    type="button"
                                    className={styles.infRefreshBtn}
                                    onClick={() => dispatch(fetchSttModels())}
                                    disabled={modelsLoading}
                                    title="Refresh models"
                                >
                                    <IconRefresh className={modelsLoading ? styles.spinning : undefined} />
                                    Refresh
                                </button>
                            </div>
                            <select
                                className={styles.infSelect}
                                value={model}
                                onChange={e => setModel(e.target.value)}
                                disabled={modelsLoading || models.length === 0}
                            >
                                {models.length === 0
                                    ? <option value="">No models available</option>
                                    : models.map(m => <option key={m} value={m}>{m}</option>)
                                }
                            </select>
                            {!modelsLoading && models.length === 0 && (
                                <div className={styles.modelWarn}>
                                    <IconWarn />
                                    No audio models found. Try refreshing.
                                </div>
                            )}
                            {!modelsLoading && models.length > 0 && (
                                <div className={styles.modelOk}>
                                    <IconOk />
                                    {models.length} model{models.length !== 1 ? 's' : ''} available
                                </div>
                            )}
                        </div>

                        <div className={styles.infField} data-tour="infer-language-field">
                            <label className={styles.infLabel}>Language</label>
                            <select className={styles.infSelect} value={language} onChange={e => setLang(e.target.value)}>
                                <option value="ko">Korean</option>
                                <option value="en">English</option>
                            </select>
                        </div>

                        <div className={styles.infField} data-tour="infer-chunk-field">
                            <label className={styles.infLabel}>Chunk size</label>
                            <select className={styles.infSelect} value={chunk} onChange={e => setChunk(e.target.value)}>
                                {CHUNK_OPTIONS.map(c => <option key={c.value} value={c.value}>{t(c.labelKey)}</option>)}
                            </select>
                        </div>
                    </div>

                    {error && <div className={styles.infError}>{error}</div>}

                    <div className={styles.infActions} data-tour="infer-run">
                        <button
                            className={`${styles.runBtn} ${canRunInference ? styles.runBtnReady : styles.runBtnDisabled}`}
                            onClick={handleSubmit}
                            disabled={!canRunInference}
                            aria-label={`Run inference on ${selectedIds.size} file${selectedIds.size !== 1 ? 's' : ''}`}
                        >
                            <span className={styles.runBtnIconWrap}>
                                {submitting ? <div className={styles.spinnerSmall} /> : <IconRun />}
                            </span>
                            <span className={styles.runBtnText}>
                                <span className={styles.runBtnTitle}>{submitting ? 'Submitting…' : 'Run inference'}</span>
                                <span className={styles.runBtnSub}>{runBtnSubLabel}</span>
                            </span>
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





















{
  "_comment": "Add these keys inside the existing 'stt' object in en.json",
  "stt": {
    "inference": {
      "chunkAuto": "Auto"
    },
    "library": {
      "perPage": "Per page",
      "perPageShort": "{{count}} / page",
      "pageInfo": "Page {{page}} of {{totalPages}} · {{total}} files",
      "searchBtn": "Search"
    },
    "tour": {
      "inferStatusTitle": "Filter by status",
      "inferStatusBody": "Narrow the list down to files that are pending, queued, running, or already completed.",
      "inferSearchTitle": "Search files",
      "inferSearchBody": "Find a file by name or ID without scrolling through the whole list.",
      "inferFilesTitle": "Pick files to run",
      "inferFilesBody": "Check off the files you want to run inference on. You can select as many as you like.",
      "inferFilesEmptyBody": "Once you've uploaded files, they'll show up here for you to select.",
      "inferModelTitle": "Choose a model",
      "inferModelBody": "Pick which speech-to-text model to run. Use Refresh if a model you expect isn't listed.",
      "inferLanguageTitle": "Set the language",
      "inferLanguageBody": "Choose the spoken language in your audio/video files for more accurate transcription.",
      "inferChunkTitle": "Set chunk size",
      "inferChunkBody": "Controls how the audio is split for processing — Auto works well for most files.",
      "inferRunTitle": "Run inference",
      "inferRunBody": "Once you've selected files and a model, kick off the batch here. You can track progress on this same tab.",
      "resultsSearchTitle": "Search completed files",
      "resultsSearchBody": "Quickly find a completed file by name or ID.",
      "resultsSortTitle": "Sort the list",
      "resultsSortBody": "Reorder completed files by ID, name, date, or status.",
      "resultsFilesTitle": "Open a transcript",
      "resultsFilesBody": "Click a completed file to load its transcript and media player on the right.",
      "resultsFilesEmptyBody": "Once a file finishes processing, it'll show up here to open.",
      "resultsContentTitle": "Transcript & player",
      "resultsContentBody": "The video/audio player and synced transcript editor appear here once a file is open."
    }
  }
}




















{
  "_comment": "Add these keys inside the existing 'stt' object in ko.json",
  "stt": {
    "inference": {
      "chunkAuto": "자동"
    },
    "library": {
      "perPage": "페이지당",
      "perPageShort": "{{count}}개씩",
      "pageInfo": "{{page}} / {{totalPages}} 페이지 · 총 {{total}}개 파일",
      "searchBtn": "검색"
    },
    "tour": {
      "inferStatusTitle": "상태로 필터링",
      "inferStatusBody": "대기 중, 대기열, 실행 중, 완료 상태의 파일로 목록을 좁혀보세요.",
      "inferSearchTitle": "파일 검색",
      "inferSearchBody": "전체 목록을 스크롤하지 않고 이름이나 ID로 파일을 찾을 수 있습니다.",
      "inferFilesTitle": "실행할 파일 선택",
      "inferFilesBody": "추론을 실행할 파일을 체크하세요. 원하는 만큼 선택할 수 있습니다.",
      "inferFilesEmptyBody": "파일을 업로드하면 이곳에 표시되어 선택할 수 있습니다.",
      "inferModelTitle": "모델 선택",
      "inferModelBody": "실행할 음성 인식 모델을 선택하세요. 원하는 모델이 없으면 새로고침을 눌러보세요.",
      "inferLanguageTitle": "언어 설정",
      "inferLanguageBody": "더 정확한 변환을 위해 오디오/비디오 파일의 사용 언어를 선택하세요.",
      "inferChunkTitle": "청크 크기 설정",
      "inferChunkBody": "오디오를 처리를 위해 나누는 방식을 설정합니다 — 대부분의 파일에는 자동이 적합합니다.",
      "inferRunTitle": "추론 실행",
      "inferRunBody": "파일과 모델을 선택했다면 여기서 배치 작업을 시작하세요. 같은 탭에서 진행 상황을 확인할 수 있습니다.",
      "resultsSearchTitle": "완료된 파일 검색",
      "resultsSearchBody": "이름이나 ID로 완료된 파일을 빠르게 찾아보세요.",
      "resultsSortTitle": "목록 정렬",
      "resultsSortBody": "ID, 이름, 날짜, 상태별로 완료된 파일을 정렬할 수 있습니다.",
      "resultsFilesTitle": "전사 결과 열기",
      "resultsFilesBody": "완료된 파일을 클릭하면 오른쪽에 전사 결과와 미디어 플레이어가 표시됩니다.",
      "resultsFilesEmptyBody": "파일 처리가 완료되면 이곳에 표시되어 열어볼 수 있습니다.",
      "resultsContentTitle": "전사 결과 및 플레이어",
      "resultsContentBody": "파일을 열면 이곳에 비디오/오디오 플레이어와 동기화된 전사 편집기가 표시됩니다."
    }
  }
}
