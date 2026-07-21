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
    onGoToInfer: () => void; // switch the parent tab bar to the Inference tab
    active: boolean;         // whether this tab is currently visible (for tour targeting)
    tourActive: boolean;     // tour state now lives in the parent (SttTranscription.tsx) since
    onTourFinish: () => void; // the trigger button lives in the shared title row
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

const FileLibrary: React.FC<Props> = ({ onGoToInfer, active, tourActive, onTourFinish }) => {
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
    const tourSteps: TourStep[] = [
        { target: 'stt-title',            title: t('stt.tour.titleTitle'),            content: t('stt.tour.titleBody'),            placement: 'bottom' },
        { target: 'stt-date-filter',      title: t('stt.tour.dateFilterTitle'),       content: t('stt.tour.dateFilterBody'),       placement: 'bottom' },
        { target: 'stt-upload',           title: t('stt.tour.uploadTitle'),           content: t('stt.tour.uploadBody'),           placement: 'right' },
        { target: 'stt-library-heading',  title: t('stt.tour.libraryTitle'),          content: t('stt.tour.libraryBody'),          placement: 'bottom' },
        { target: 'stt-date-filter',      title: t('stt.tour.dateFilterTitle'),       content: t('stt.tour.dateFilterBody'),       placement: 'bottom', onEnter: () => setPanelMode('none') },
        { target: 'stt-status-chips',     title: t('stt.tour.statusChipsTitle'),      content: t('stt.tour.statusChipsBody'),      placement: 'bottom', onEnter: () => setPanelMode('none') },
        { target: 'stt-sort',             title: t('stt.tour.sortTitle'),             content: t('stt.tour.sortBody'),             placement: 'bottom', onEnter: () => setPanelMode('none') },
        {
            target: 'stt-action-search',
            title: t('stt.tour.actionsTriggerTitle'),
            content: t('stt.tour.actionsTriggerBody'),
            placement: 'bottom',
            onEnter: () => setPanelMode('none'),
        },
        {
            target: 'stt-action-delete',
            title: t('stt.tour.actionDeleteTitle'),
            content: t('stt.tour.actionDeleteBody'),
            placement: 'left',
            onEnter: () => setPanelMode('none'),
        },
        {
            target: 'stt-action-pipeline',
            title: t('stt.tour.actionPipelineTitle'),
            content: t('stt.tour.actionPipelineBody'),
            placement: 'left',
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
            {/* Tour guide */}
            <TourGuide steps={tourSteps} active={tourActive} onFinish={onTourFinish} />

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
};

export default FileLibrary;










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
      "searchBtn": "Search",
      "closeSearch": "Close search"
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
      "searchBtn": "검색",
      "closeSearch": "검색 닫기"
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
