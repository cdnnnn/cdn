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

const FileLibrary: React.FC<Props> = ({ onOpen, onGoToInfer, active, tourActive, onTourFinish }) => {
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
        { target: 'stt-date-filter',      title: t('stt.tour.dateFilterTitle'),       content: t('stt.tour.dateFilterBody'),       placement: 'bottom', onEnter: () => setActionsOpen(false) },
        { target: 'stt-status-chips',     title: t('stt.tour.statusChipsTitle'),      content: t('stt.tour.statusChipsBody'),      placement: 'bottom', onEnter: () => setActionsOpen(false) },
        { target: 'stt-sort',             title: t('stt.tour.sortTitle'),             content: t('stt.tour.sortBody'),             placement: 'bottom', onEnter: () => setActionsOpen(false) },
        {
            target: 'stt-actions-trigger',
            title: t('stt.tour.actionsTriggerTitle'),
            content: t('stt.tour.actionsTriggerBody'),
            placement: 'bottom',
            onEnter: () => { setPanelMode('none'); setActionsOpen(false); },
        },
        {
            target: 'stt-action-delete',
            title: t('stt.tour.actionDeleteTitle'),
            content: t('stt.tour.actionDeleteBody'),
            placement: 'left',
            onEnter: () => { setPanelMode('none'); setActionsOpen(true); },
        },
        {
            target: 'stt-action-pipeline',
            title: t('stt.tour.actionPipelineTitle'),
            content: t('stt.tour.actionPipelineBody'),
            placement: 'left',
            onEnter: () => { setPanelMode('none'); setActionsOpen(true); },
        },
        { target: 'stt-library-body',     title: t('stt.tour.fileCardsTitle'),        content: t('stt.tour.fileCardsBody'),        placement: 'top', onEnter: () => setActionsOpen(false) },
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

    // ── Actions popover (Inference / Delete / Pipeline collapsed into one trigger) ──
    const [actionsOpen, setActionsOpen] = useState(false);
    const actionsPopoverRef = React.useRef<HTMLDivElement | null>(null);
    useEffect(() => {
        if (!actionsOpen) return;
        const onDocClick = (e: MouseEvent) => {
            if (actionsPopoverRef.current && !actionsPopoverRef.current.contains(e.target as Node)) {
                setActionsOpen(false);
            }
        };
        document.addEventListener('mousedown', onDocClick);
        return () => document.removeEventListener('mousedown', onDocClick);
    }, [actionsOpen]);

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
                        <button className={styles.colHeadingRefresh} onClick={handleRefresh}
                            disabled={initializing || filesLoading || inferenceRunning}
                            title={inferenceRunning ? t('stt.refreshDisabled') : t('stt.refreshTitle')}>
                            <IconRefresh />
                        </button>
                    </div>

                    {/* ── Filter / Sort row — date filter + status filter + actions on the left/right, sort + search below ── */}
                    <div className={styles.filterSortRow}>
                        <div className={styles.filterBarRow}>
                            {/* Date filter */}
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

                                {/* Actions — collapsed into a single trigger + popover */}
                                {panelMode === 'none' && !inferenceRunning && (
                                    <div className={styles.actionsPopoverWrap} ref={actionsPopoverRef}>
                                        <button
                                            type="button"
                                            data-tour="stt-actions-trigger"
                                            className={`${styles.actionsTrigger} ${actionsOpen ? styles.actionsTriggerOpen : ''}`}
                                            onClick={() => setActionsOpen(o => !o)}
                                            aria-expanded={actionsOpen}
                                            aria-haspopup="true"
                                        >
                                            <span className={styles.filterBarLabel}>{t('stt.actionsLabel')}</span>
                                            <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                                <path d="M7 1.5l1.4 3.4L12 6l-3.6 1.4L7 12.5l-1.4-4.1L2 7l3.6-1.1z" />
                                            </svg>
                                            <svg className={styles.actionsTriggerChevron} viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                                <path d="M3 4.5l3 3 3-3" />
                                            </svg>
                                        </button>

                                        {actionsOpen && (
                                            <div className={styles.actionsPopover} role="menu" data-tour="stt-actions">
                                                <button
                                                    data-tour="stt-action-delete"
                                                    className={`${styles.actionTile} ${styles.actionTileDelete}`}
                                                    onClick={() => { setActionsOpen(false); openPanel('delete'); }}
                                                >
                                                    <IconDelete />
                                                    <span className={styles.actionTileLabel}>{t('stt.library.deleteFiles')}</span>
                                                </button>
                                                <button
                                                    data-tour="stt-action-pipeline"
                                                    className={`${styles.actionTile} ${styles.actionTilePipeline}`}
                                                    onClick={() => { setActionsOpen(false); openPanel('pipeline'); }}
                                                >
                                                    <IconPipeline />
                                                    <span className={styles.actionTileLabel}>{t('stt.library.movePipeline')}</span>
                                                </button>
                                            </div>
                                        )}
                                    </div>
                                )}
                            </div>
                        </div>

                        {/* Sort row + search */}
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
