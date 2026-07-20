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

    // Clicking a file (from Upload tab's library, or the Results sidebar)
    // opens its transcript and jumps to the Results tab.
    const handleFileClick = useCallback((fileId: number) => {
        setActiveFileId(fileId);
        setActiveTab('results');
    }, []);

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
            {/* ── Header — title + tab bar ── */}
            <div className={styles.sttHeaderBar}>
                <div style={{ display: 'flex', alignItems: 'center', gap: 10, marginBottom: 14 }}>
                    <div className={styles.phTitle} data-tour="stt-title">{t('stt.pageTitle')}</div>
                    <button
                        type="button"
                        className={styles.tourTriggerBtn}
                        onClick={startTour}
                    >
                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                            <circle cx="8" cy="8" r="6.25" />
                            <path d="M6.1 6.2a1.9 1.9 0 013.6.7c0 1.3-1.7 1.5-1.7 2.7M8 11.4v.1" />
                        </svg>
                        {t('stt.tour.takeTour')}
                    </button>
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
                {filesLoading ? (
                    <div className={styles.libLoading}>
                        <div className={styles.spinner} />
                        <span>{t('stt.library.loading')}</span>
                    </div>
                ) : filtered.length === 0 ? (
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
