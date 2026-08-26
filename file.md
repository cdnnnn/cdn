//Custommetrics.module.scss
@use '../../styles/_variables' as *;

// ===========================================================================
// Custom Metrics Dashboard — mirrors the Model Catalog design system
// exactly: ink/paper palette, ultramarine signal accent, mono instrument
// labels, sortable table headers, filter pills, and the same pagination
// bar. Reuses the app's shared global classes (pg-shell, pg-toolbar,
// toolbar, search-box, pills, pill, pg-body, tw, tbl, tag, tag-ind, badge,
// badge-green, badge-gray) for the toolbar/table body, exactly like
// Model Catalog does — this module only supplies the header, the tab
// switcher (Dashboard/Create Metric — unique to this page), sortable
// column headers, pagination, and a couple of shared bits (toast, spin)
// used elsewhere in this feature.
//
// Font scaling: `.custom-metrics` sets a single base font-size — the same
// 0.8125rem base Model Catalog uses. All descendant font-sizes are
// expressed in `em` (relative to that base), so bumping the base on wide
// screens scales the whole page proportionally from one place — same
// convention, same numbers, as Model Catalog.
// ===========================================================================

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

$soft: 0 1px 2px rgba(20, 22, 27, 0.05);
$lift: 0 14px 30px -14px rgba(20, 22, 27, 0.22);

// base font-size the dashboard's internal `em` scale is built on — matches
// Model Catalog's base exactly, so both pages scale identically together.
$custom-metrics-base-font: 0.8125rem;

%micro {
  font-family: $mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

@keyframes cm-spin { to { transform: rotate(360deg); } }
@keyframes cm-toast-in { from { opacity: 0; transform: translate(-50%, 8px); } to { opacity: 1; transform: translate(-50%, 0); } }

.spin { animation: cm-spin 0.8s linear infinite; }

.custom-metrics {
  // master scale control — every em-based font-size below responds to this
  font-size: $custom-metrics-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  // ---- header -------------------------------------------------------------
  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding: 24px 32px 20px;
    margin-bottom: 20px;
    border-bottom: 1px solid $line;
    background: $card;
    flex-wrap: wrap;

    h1 {
      font-family: $display;
      font-size: 1.8462em; // 1.5rem / 0.8125rem
      font-weight: 800;
      letter-spacing: -0.02em;
      color: $ink;
      line-height: 1.2;
    }
  }

  &__header-eyebrow {
    @extend %micro;
    display: flex;
    align-items: center;
    gap: 8px;
    color: $signal;
    margin-bottom: 6px;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $signal;
    }
  }

  &__header-sub {
    margin-top: 4px;
    font-size: 1.0385em; // 0.84375rem / 0.8125rem
    color: $ink-2;
  }

  &__header-meta {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 7px;
    padding: 7px 13px;
    border-radius: 999px;
    border: 1px solid $line;
    background: $paper;
    font-family: $mono;
    font-size: 0.8846em; // 0.71875rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ink-2;
    white-space: nowrap;
    margin-bottom: 3px;
  }

  // ---- toolbar filter group (Eval Type pills) ------------------------------
  &__filter-group {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    padding: 4px;
    background: $paper;
    border: 1px solid $line;
    border-radius: 999px;
    flex-wrap: wrap;
  }

  &__toolbar-label {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 5px 10px 5px 11px;
    @extend %micro;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    color: $ink-3;
    white-space: nowrap;
  }

  &__filter-pill {
    padding: 6px 13px;
    border: 0;
    border-radius: 999px;
    background: transparent;
    color: $ink-2;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: all 0.15s ease;

    &:hover { color: $ink; }

    &--on {
      background: $card;
      color: $signal;
      box-shadow: $soft;
    }
  }

  // --- sortable column headers ----------------------------------------------
  &__sortable-th {
    padding: 0 !important;
  }

  &__sort-btn {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    width: 100%;
    padding: 12px 16px;
    background: none;
    border: none;
    cursor: pointer;
    font: inherit;
    font-family: $mono;
    font-weight: 700;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    letter-spacing: 0.14em;
    text-transform: uppercase;
    color: $ink-3;
    text-align: left;
    transition: color 0.15s ease;

    &:hover {
      color: $ink;

      .custom-metrics__sort-icon-idle { opacity: 0.7; }
    }

    &--active {
      color: $signal;
    }
  }

  &__sort-icon-idle {
    opacity: 0.28;
    transition: opacity 0.15s ease;
  }

  &__empty {
    text-align: center;
    padding: 44px 16px !important;
    color: $ink-3;
    font-size: 1.0385em; // 0.84375rem / 0.8125rem
  }

  // --- pagination bar ---------------------------------------------------------
  &__pagination {
    display: flex;
    align-items: center;
    justify-content: space-between;
    flex-wrap: wrap;
    gap: 12px;
    padding: 14px 20px;
    border-top: 1px solid $line;
    background: $paper;
  }

  &__pagination-info {
    display: flex;
    align-items: center;
    flex-wrap: wrap;
    gap: 18px;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    color: $ink-2;

    strong { color: $ink; font-weight: 700; }
  }

  &__page-size {
    display: flex;
    align-items: center;
    gap: 8px;

    label {
      font-size: 0.8846em; // 0.71875rem / 0.8125rem
      color: $ink-3;
      white-space: nowrap;
    }

    select {
      appearance: none;
      -webkit-appearance: none;
      font: inherit;
      font-size: 0.9615em; // 0.78125rem / 0.8125rem
      font-weight: 650;
      color: $ink;
      background: $card;
      border: 1px solid $line;
      border-radius: 8px;
      padding: 5px 26px 5px 10px;
      cursor: pointer;
      background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='10' height='6' viewBox='0 0 10 6' fill='none'%3E%3Cpath d='M1 1L5 5L9 1' stroke='%23565B66' stroke-width='1.5' stroke-linecap='round' stroke-linejoin='round'/%3E%3C/svg%3E");
      background-repeat: no-repeat;
      background-position: right 10px center;

      &:hover { border-color: $ink-3; }
      &:focus { outline: none; border-color: $signal; box-shadow: 0 0 0 3px $wash; }
    }
  }

  &__pager {
    display: flex;
    align-items: center;
    gap: 4px;
  }

  &__page-btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    min-width: 30px;
    height: 30px;
    padding: 0 6px;
    border-radius: 8px;
    border: 1px solid transparent;
    background: transparent;
    color: $ink-2;
    font-family: $mono;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: background 0.14s ease, color 0.14s ease;

    &:hover:not(:disabled) {
      background: $wash;
      color: $signal;
    }

    &:disabled {
      opacity: 0.35;
      cursor: not-allowed;
    }

    &--num {
      min-width: 30px;
    }

    &--active {
      background: $signal;
      color: #fff;

      &:hover:not(:disabled) {
        background: $signal;
        color: #fff;
      }
    }
  }

  &__page-dots {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    min-width: 20px;
    height: 30px;
    color: $ink-3;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
  }
}

// ---------------------------------------------------------------------------
// tab switcher (Dashboard / Create Metric) — unique to this page, not part
// of the Model Catalog reference, sized on the same em scale.
// ---------------------------------------------------------------------------
.tabbar {
  flex-shrink: 0;
  display: inline-flex;
  padding: 3px;
  gap: 2px;
  border-radius: 11px;
  border: 1px solid $line;
  background: $paper;
  margin-bottom: 3px;
}

.tab {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  padding: 8px 14px;
  border-radius: 8px;
  border: none;
  background: transparent;
  color: $ink-2;
  font-family: $sans;
  font-size: 0.9615em; // 0.78125rem / 0.8125rem
  font-weight: 650;
  cursor: pointer;
  white-space: nowrap;
  transition: all 0.15s ease;

  &:hover:not(&--active) { color: $ink; }
}

.tab--active {
  background: $card;
  color: $signal;
  box-shadow: $soft;
}

// ---------------------------------------------------------------------------
// error banner (fetch/delete failures)
// ---------------------------------------------------------------------------
.error-banner {
  display: flex; align-items: center; gap: 8px;
  padding: 10px 14px; border-radius: 10px;
  background: $danger-wash; border: 1px solid rgba($danger, 0.2);
  color: $danger; font-size: 1em; margin-bottom: 16px;
}

// ---------------------------------------------------------------------------
// toast (used by useToast.tsx)
// ---------------------------------------------------------------------------
.toast {
  position: fixed;
  left: 50%;
  bottom: 28px;
  transform: translateX(-50%);
  z-index: 400;
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 12px 18px;
  border-radius: 11px;
  background: #14161B;
  color: #fff;
  font-size: 0.8125rem;
  font-weight: 650;
  box-shadow: $lift;
  animation: cm-toast-in 0.18s ease;
}
.toast--ok::before { content: ''; width: 6px; height: 6px; border-radius: 50%; background: $ok; }
.toast--error::before { content: ''; width: 6px; height: 6px; border-radius: 50%; background: $danger; }
.toast--info::before { content: ''; width: 6px; height: 6px; border-radius: 50%; background: $signal; }

@media (max-width: 768px) {
  .custom-metrics__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
}

















//Custommetricsdashboard.tsx
import { useEffect, useMemo, useState, useCallback } from 'react';
import {
  Search, Gauge, LayoutDashboard, PenSquare, ListFilter, AlertCircle, Loader2, Trash2,
  ChevronUp, ChevronDown, ChevronsUpDown, ChevronLeft, ChevronRight, ChevronsLeft, ChevronsRight,
} from 'lucide-react';
import { SkeletonTableRows } from '../common/Skeleton';
import styles from './CustomMetrics.module.scss';
import { metricsApi, CustomMetric } from '../../api/endpoints/metrics';
import CreateMetric from './CreateMetric';

type View = 'dashboard' | 'create';
type SortKey = 'name' | 'type' | 'threshold' | 'judge' | 'status' | 'created';
type SortDir = 'asc' | 'desc';

const PAGE_SIZE_OPTIONS = [10, 25, 50, 100];

function formatDate(iso: string) {
  const d = new Date(iso);
  return Number.isNaN(d.getTime()) ? iso : d.toLocaleDateString(undefined, { year: 'numeric', month: 'short', day: 'numeric' });
}

// Builds a compact page-number list with ellipses, e.g. [1, '…', 4, 5, 6, '…', 12]
function buildPageList(current: number, total: number): (number | '…')[] {
  if (total <= 7) return Array.from({ length: total }, (_, i) => i + 1);
  const pages = new Set<number>([1, total, current, current - 1, current + 1]);
  const sorted = [...pages].filter((p) => p >= 1 && p <= total).sort((a, b) => a - b);
  const result: (number | '…')[] = [];
  let prev = 0;
  for (const p of sorted) {
    if (prev && p - prev > 1) result.push('…');
    result.push(p);
    prev = p;
  }
  return result;
}

interface SortableThProps {
  label: string;
  sortKey: SortKey;
  activeKey: SortKey;
  dir: SortDir;
  onSort: (key: SortKey) => void;
}

function SortableTh({ label, sortKey, activeKey, dir, onSort }: SortableThProps) {
  const active = activeKey === sortKey;
  return (
    <th className={styles['custom-metrics__sortable-th']}>
      <button
        type="button"
        className={`${styles['custom-metrics__sort-btn']} ${active ? styles['custom-metrics__sort-btn--active'] : ''}`}
        onClick={() => onSort(sortKey)}
      >
        {label}
        {active ? (
          dir === 'asc' ? <ChevronUp size={13} /> : <ChevronDown size={13} />
        ) : (
          <ChevronsUpDown size={13} className={styles['custom-metrics__sort-icon-idle']} />
        )}
      </button>
    </th>
  );
}

export default function CustomMetricsDashboard() {
  const [view, setView] = useState<View>('dashboard');

  const [metrics, setMetrics] = useState<CustomMetric[]>([]);
  const [status, setStatus] = useState<'idle' | 'loading' | 'succeeded' | 'failed'>('loading');
  const [error, setError] = useState('');

  const [search, setSearch] = useState('');
  const [evalFilter, setEvalFilter] = useState('All');
  const [sortKey, setSortKey] = useState<SortKey>('name');
  const [sortDir, setSortDir] = useState<SortDir>('asc');
  const [page, setPage] = useState(1);
  const [pageSize, setPageSize] = useState(10);

  // delete state
  const [pendingDeleteId, setPendingDeleteId] = useState('');
  const [deletingId, setDeletingId] = useState('');
  const [deleteError, setDeleteError] = useState('');

  const fetchMetrics = useCallback(() => {
    setStatus('loading');
    setError('');
    metricsApi.list()
      .then((list) => { setMetrics(list); setStatus('succeeded'); })
      .catch((err) => { setError(err.message || 'Failed to load metrics'); setStatus('failed'); })
      .finally(() => {});
  }, []);

  useEffect(() => { fetchMetrics(); }, [fetchMetrics]);

  const evalTypeOptions = useMemo(() => ['All', ...new Set(metrics.flatMap((m) => m.eval_types ?? []))], [metrics]);

  const filtered = useMemo(() => {
    return metrics.filter((m) => {
      if (evalFilter !== 'All' && !(m.eval_types ?? []).includes(evalFilter)) return false;
      const q = search.toLowerCase();
      return !q || (m.name ?? '').toLowerCase().includes(q) || (m.description ?? '').toLowerCase().includes(q);
    });
  }, [metrics, search, evalFilter]);

  const sorted = useMemo(() => {
    const dir = sortDir === 'asc' ? 1 : -1;
    const compare = (a: CustomMetric, b: CustomMetric): number => {
      switch (sortKey) {
        case 'name':
          return (a.name ?? '').localeCompare(b.name ?? '') * dir;
        case 'type':
          return (a.metric_type ?? '').localeCompare(b.metric_type ?? '') * dir;
        case 'threshold':
          return ((a.threshold ?? 0) - (b.threshold ?? 0)) * dir;
        case 'judge':
          return (Number(a.requires_judge) - Number(b.requires_judge)) * dir;
        case 'status':
          return (Number(a.is_active) - Number(b.is_active)) * dir;
        case 'created':
          return (new Date(a.created_at).getTime() - new Date(b.created_at).getTime()) * dir;
        default:
          return 0;
      }
    };
    return [...filtered].sort(compare);
  }, [filtered, sortKey, sortDir]);

  const total = sorted.length;
  const totalPages = Math.max(1, Math.ceil(total / pageSize));
  const safePage = Math.min(page, totalPages);
  const startIdx = (safePage - 1) * pageSize;
  const pageItems = sorted.slice(startIdx, startIdx + pageSize);
  const pageList = useMemo(() => buildPageList(safePage, totalPages), [safePage, totalPages]);

  useEffect(() => {
    setPage(1);
  }, [search, evalFilter, pageSize]);

  const toggleSort = (key: SortKey) => {
    if (sortKey === key) {
      setSortDir((d) => (d === 'asc' ? 'desc' : 'asc'));
    } else {
      setSortKey(key);
      setSortDir('asc');
    }
  };

  const requestDelete = (id: string) => {
    setDeleteError('');
    setPendingDeleteId(id);
  };
  const cancelDelete = () => setPendingDeleteId('');
  const confirmDelete = (id: string) => {
    setDeletingId(id);
    setDeleteError('');
    metricsApi.remove(id)
      .then(() => {
        setMetrics((prev) => prev.filter((m) => m.id !== id));
        setPendingDeleteId('');
      })
      .catch((err) => setDeleteError(err.message || 'Failed to delete metric'))
      .finally(() => setDeletingId(''));
  };

  // After a successful save, hop back to the dashboard and refresh the list.
  const handleSaved = () => {
    setView('dashboard');
    fetchMetrics();
  };

  return (
    <div className={`page-enter pg-shell ${styles['custom-metrics']}`}>
      <div className={styles['custom-metrics__header']}>
        <div>
          <p className={styles['custom-metrics__header-eyebrow']}>Custom Metrics</p>
          <h1>{view === 'dashboard' ? 'Dashboard' : 'Create Metric'}</h1>
          <p className={styles['custom-metrics__header-sub']}>
            {view === 'dashboard' ? 'Saved metrics for evaluation' : 'Fill in every section below, then validate and save.'}
          </p>
        </div>

        <div style={{ display: 'flex', alignItems: 'center', gap: 12 }}>
          <div className={styles.tabbar}>
            <button
              type="button"
              className={`${styles.tab} ${view === 'dashboard' ? styles['tab--active'] : ''}`}
              onClick={() => setView('dashboard')}
            >
              <LayoutDashboard size={14} /> Dashboard
            </button>
            <button
              type="button"
              className={`${styles.tab} ${view === 'create' ? styles['tab--active'] : ''}`}
              onClick={() => setView('create')}
            >
              <PenSquare size={14} /> Create Metric
            </button>
          </div>

          {view === 'dashboard' && (
            <div className={styles['custom-metrics__header-meta']}>
              <Gauge size={13} />
              {metrics.length} metric{metrics.length === 1 ? '' : 's'} listed
            </div>
          )}
        </div>
      </div>

      {view === 'create' ? (
        <CreateMetric onCancel={() => setView('dashboard')} onSaved={handleSaved} />
      ) : (
        <>
          <div className="pg-toolbar">
            <div className="toolbar">
              <div className="search-box">
                <Search size={16} color="var(--text-muted)" />
                <input placeholder="Search metrics…" value={search} onChange={(e) => setSearch(e.target.value)} />
              </div>
              <div style={{ display: 'flex', gap: 8, alignItems: 'center' }}>
                <div className={styles['custom-metrics__filter-group']}>
                  <span className={styles['custom-metrics__toolbar-label']}>
                    <ListFilter size={11} /> Eval Type
                  </span>
                  {evalTypeOptions.map((t) => (
                    <button
                      key={t}
                      className={`${styles['custom-metrics__filter-pill']} ${evalFilter === t ? styles['custom-metrics__filter-pill--on'] : ''}`}
                      onClick={() => setEvalFilter(t)}
                    >
                      {t === 'All' ? 'All' : t.toUpperCase()}
                    </button>
                  ))}
                </div>
              </div>
            </div>
          </div>

          <div className="pg-body">
            {error && <div className={styles['error-banner']}><AlertCircle size={14} /> {error}</div>}
            {deleteError && <div className={styles['error-banner']}><AlertCircle size={14} /> {deleteError}</div>}

            <div className="tw">
              <table className="tbl">
                <thead>
                  <tr>
                    <SortableTh label="Name" sortKey="name" activeKey={sortKey} dir={sortDir} onSort={toggleSort} />
                    <th>Eval Types</th>
                    <SortableTh label="Type" sortKey="type" activeKey={sortKey} dir={sortDir} onSort={toggleSort} />
                    <SortableTh label="Threshold" sortKey="threshold" activeKey={sortKey} dir={sortDir} onSort={toggleSort} />
                    <SortableTh label="Judge" sortKey="judge" activeKey={sortKey} dir={sortDir} onSort={toggleSort} />
                    <SortableTh label="Status" sortKey="status" activeKey={sortKey} dir={sortDir} onSort={toggleSort} />
                    <SortableTh label="Created" sortKey="created" activeKey={sortKey} dir={sortDir} onSort={toggleSort} />
                    <th style={{ width: '1%' }} />
                  </tr>
                </thead>
                <tbody>
                  {status === 'loading' && <SkeletonTableRows columns={8} rows={6} />}
                  {status !== 'loading' && pageItems.map((m) => {
                    const isPending = pendingDeleteId === m.id;
                    const isDeleting = deletingId === m.id;
                    return (
                      <tr key={m.id} title={m.description}>
                        <td style={{ fontWeight: 700 }}>{m.name ?? '—'}</td>
                        <td>
                          <div style={{ display: 'flex', gap: 6, flexWrap: 'wrap' }}>
                            {(m.eval_types ?? []).map((t) => <span key={t} className="tag tag-ind">{(t ?? '').toUpperCase()}</span>)}
                          </div>
                        </td>
                        <td><span className="tag tag-ind">{m.metric_type}</span></td>
                        <td style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", fontSize: 13, color: 'var(--text-secondary)' }}>{m.threshold}</td>
                        <td style={{ color: 'var(--text-secondary)' }}>{m.requires_judge ? 'Yes' : 'No'}</td>
                        <td><span className={`badge ${m.is_active ? 'badge-green' : 'badge-gray'}`}>{m.is_active ? 'Active' : 'Inactive'}</span></td>
                        <td style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", fontSize: 13, color: 'var(--text-secondary)' }}>{formatDate(m.created_at)}</td>
                        <td>
                          {isPending ? (
                            <div style={{ display: 'flex', gap: 6, alignItems: 'center', whiteSpace: 'nowrap' }}>
                              <span style={{ fontSize: '0.78125rem', color: 'var(--text-secondary)' }}>Delete?</span>
                              <button
                                type="button"
                                className="pill"
                                style={{ borderColor: '#DC2626', background: '#DC2626', color: '#fff' }}
                                disabled={isDeleting}
                                onClick={() => confirmDelete(m.id)}
                              >
                                {isDeleting ? <Loader2 size={13} className={styles.spin} /> : 'Confirm'}
                              </button>
                              <button type="button" className="pill" disabled={isDeleting} onClick={cancelDelete}>Cancel</button>
                            </div>
                          ) : (
                            <button
                              type="button"
                              title="Delete metric"
                              aria-label={`Delete ${m.name}`}
                              onClick={() => requestDelete(m.id)}
                              style={{
                                display: 'inline-flex', alignItems: 'center', justifyContent: 'center',
                                width: 28, height: 28, borderRadius: 8, border: '1px solid transparent',
                                background: 'transparent', color: 'var(--text-muted)', cursor: 'pointer',
                              }}
                              onMouseEnter={(e) => {
                                e.currentTarget.style.background = 'rgba(220,38,38,0.08)';
                                e.currentTarget.style.borderColor = 'rgba(220,38,38,0.2)';
                                e.currentTarget.style.color = '#DC2626';
                              }}
                              onMouseLeave={(e) => {
                                e.currentTarget.style.background = 'transparent';
                                e.currentTarget.style.borderColor = 'transparent';
                                e.currentTarget.style.color = 'var(--text-muted)';
                              }}
                            >
                              <Trash2 size={14} />
                            </button>
                          )}
                        </td>
                      </tr>
                    );
                  })}
                  {status !== 'loading' && pageItems.length === 0 && (
                    <tr>
                      <td colSpan={8} className={styles['custom-metrics__empty']}>No metrics match your filters.</td>
                    </tr>
                  )}
                </tbody>
              </table>

              {status !== 'loading' && total > 0 && (
                <div className={styles['custom-metrics__pagination']}>
                  <div className={styles['custom-metrics__pagination-info']}>
                    <span>
                      Showing <strong>{startIdx + 1}–{Math.min(startIdx + pageSize, total)}</strong> of <strong>{total}</strong> metric{total === 1 ? '' : 's'}
                    </span>
                    <div className={styles['custom-metrics__page-size']}>
                      <label htmlFor="custom-metrics-page-size">Rows per page</label>
                      <select
                        id="custom-metrics-page-size"
                        value={pageSize}
                        onChange={(e) => setPageSize(Number(e.target.value))}
                      >
                        {PAGE_SIZE_OPTIONS.map((n) => (
                          <option key={n} value={n}>{n}</option>
                        ))}
                      </select>
                    </div>
                  </div>

                  <div className={styles['custom-metrics__pager']}>
                    <button
                      className={styles['custom-metrics__page-btn']}
                      disabled={safePage === 1}
                      onClick={() => setPage(1)}
                      aria-label="First page"
                    >
                      <ChevronsLeft size={14} />
                    </button>
                    <button
                      className={styles['custom-metrics__page-btn']}
                      disabled={safePage === 1}
                      onClick={() => setPage((p) => Math.max(1, p - 1))}
                      aria-label="Previous page"
                    >
                      <ChevronLeft size={14} />
                    </button>

                    {pageList.map((p, i) =>
                      p === '…' ? (
                        <span key={`dots-${i}`} className={styles['custom-metrics__page-dots']}>…</span>
                      ) : (
                        <button
                          key={p}
                          className={`${styles['custom-metrics__page-btn']} ${styles['custom-metrics__page-btn--num']} ${p === safePage ? styles['custom-metrics__page-btn--active'] : ''}`}
                          onClick={() => setPage(p)}
                          aria-current={p === safePage ? 'page' : undefined}
                        >
                          {p}
                        </button>
                      )
                    )}

                    <button
                      className={styles['custom-metrics__page-btn']}
                      disabled={safePage === totalPages}
                      onClick={() => setPage((p) => Math.min(totalPages, p + 1))}
                      aria-label="Next page"
                    >
                      <ChevronRight size={14} />
                    </button>
                    <button
                      className={styles['custom-metrics__page-btn']}
                      disabled={safePage === totalPages}
                      onClick={() => setPage(totalPages)}
                      aria-label="Last page"
                    >
                      <ChevronsRight size={14} />
                    </button>
                  </div>
                </div>
              )}
            </div>
          </div>
        </>
      )}
    </div>
  );
}




















//Createmetric.module.scss
@use '../../styles/_variables' as *;

// ===========================================================================
// Create Metric — single-page builder (all sections visible at once).
// Left: overview rail with a redesigned "living timeline" stepper.
// Right: every section stacked, separated by dashed dividers, capped at
// a wider 1000px reading column.
//
// Font scaling follows the same convention as Model Catalog: `.cm` sets a
// single base font-size, every descendant font-size is expressed in `em`
// relative to that base, so bumping `.cm`'s font-size on wide screens
// scales the whole builder proportionally from one place.
// ===========================================================================

// Neutrals, accents, and washes all come from the shared "ink" block in
// _variables.scss (theme-aware via _theme.scss custom properties) — same
// tokens Model Catalog uses, no locally-declared colors. $amber is kept
// as a local alias only because this file's selectors were written
// against that name; it points at the same $amber-ink token as everyone
// else.
$amber: $amber-ink;
// Toast needs to stay legible against its own dark chip in both themes
// (unlike page surfaces, which flip), so it uses the ink-1 dark value
// directly rather than the theme-flipping $ink token.
$ink-solid: #14161B;

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

$soft: 0 1px 2px rgba(20, 22, 27, 0.05);
$lift: 0 18px 40px -20px rgba(20, 22, 27, 0.30);

// base font-size the whole builder's internal `em` scale is built on
$base-font: 0.8125rem; // matches Model Catalog / Custom Metrics Dashboard base

%micro {
  font-family: $mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

@keyframes cm-spin { to { transform: rotate(360deg); } }
@keyframes cm-fade-up { from { opacity: 0; transform: translateY(8px); } to { opacity: 1; transform: translateY(0); } }
@keyframes cm-pop { 0% { transform: scale(0.7); opacity: 0; } 60% { transform: scale(1.08); } 100% { transform: scale(1); opacity: 1; } }
@keyframes cm-modal-in { from { opacity: 0; transform: translateY(12px) scale(0.98); } to { opacity: 1; transform: translateY(0) scale(1); } }
@keyframes cm-check-pop { 0% { transform: scale(0.4); opacity: 0; } 70% { transform: scale(1.15); } 100% { transform: scale(1); opacity: 1; } }

.spin { animation: cm-spin 0.8s linear infinite; }

// ---------------------------------------------------------------------------
// shell — master scale control. Every em-based font-size below responds
// to this. On very wide screens, bumping it to 1rem scales everything.
// ---------------------------------------------------------------------------
.cm {
  font-size: $base-font;
  flex: 1;
  min-height: 0;
  display: flex;
  flex-direction: column;

  @media (min-width: 1800px) { font-size: 1rem; }
}

.builder {
  flex: 1;
  min-height: 0;
  display: grid;
  grid-template-columns: 300px 1fr;
  gap: 0;
  overflow: hidden;
}

// ---------------------------------------------------------------------------
// LEFT RAIL — vertical stepper with a connecting line that tracks the
// 12px gap between rows, flat solid colors (no gradients), and a clear
// done/active state — jump to any section, any time.
// ---------------------------------------------------------------------------
.rail {
  display: flex;
  flex-direction: column;
  min-height: 0;
  background: $card;
  border-right: 1px solid $line;
  overflow-y: auto;
}

.rail__head {
  padding: 26px 22px 20px;
  border-bottom: 1px solid $line;
}

.rail__eyebrow {
  @extend %micro;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  color: $signal;
  display: flex;
  align-items: center;
  gap: 7px;
  margin-bottom: 10px;

  &::before { content: ''; width: 14px; height: 2px; border-radius: 2px; background: $signal; }
}

.rail__sub {
  margin-top: 4px;
  font-size: 0.9615em; // 0.78125rem / 0.8125rem
  color: $ink-3;
  line-height: 1.5;
}

.rail__steps {
  flex: 1;
  padding: 18px 14px 20px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.rail-step {
  position: relative;
  display: flex;
  align-items: flex-start;
  gap: 14px;
  width: 100%;
  text-align: left;
  padding: 13px 14px 13px 12px;
  border-radius: 16px;
  border: 1.5px solid transparent;
  background: transparent;
  cursor: pointer;
  transition: background 0.2s ease, border-color 0.2s ease, transform 0.2s ease, box-shadow 0.2s ease;

  &:hover {
    background: $paper;
    border-color: $line;
    transform: translateX(3px);

    .rail-step__arrow { opacity: 1; transform: translateX(0); }
  }

  &--done {
    background: $wash;
    border-color: rgba($signal, 0.16);

    &:hover { border-color: rgba($signal, 0.35); }
  }

  // vertical connector: starts right below this marker, and reaches all
  // the way through the 12px row gap into the top of the next marker.
  &:not(:last-child)::after {
    content: '';
    position: absolute;
    left: 30px;
    top: 49px;
    bottom: -25px; // 12px row gap + 13px next step's top padding
    width: 2px;
    border-radius: 2px;
    background: $line;
    z-index: 0;
    transition: background 0.25s ease;
  }
  &--done:not(:last-child)::after {
    background: $signal;
  }
}

.rail-step__marker {
  position: relative;
  z-index: 1;
  flex-shrink: 0;
  width: 36px;
  height: 36px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-family: $mono;
  font-size: 1.2308em; // 1.0rem / 0.8125rem
  font-weight: 800;
  color: $ink-3;
  background: $paper;
  border: 2px solid $line;
  transition: all 0.22s cubic-bezier(0.34, 1.56, 0.64, 1);

  .rail-step--done & {
    color: #fff;
    background: $signal;
    border-color: $signal;
    box-shadow: 0 4px 12px -3px rgba(43, 43, 245, 0.45);
    animation: cm-check-pop 0.3s ease;
  }

  .rail-step:hover:not(.rail-step--done) & {
    border-color: $ink-3;
    color: $ink-2;
    transform: scale(1.08);
  }
}

.rail-step__body {
  min-width: 0;
  flex: 1;
  padding-top: 5px;
}

.rail-step__label {
  display: block;
  font-size: 1.2308em; // 1.0rem / 0.8125rem
  font-weight: 700;
  color: $ink-2;
  transition: color 0.2s ease;

  .rail-step--done & { color: $ink; }
}

.rail-step__value {
  display: block;
  font-family: $mono;
  font-size: 1.0000em; // 0.8125rem / 0.8125rem
  color: $ink-3;
  margin-top: 5px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;

  .rail-step--done & { color: $signal; font-weight: 700; }
}

.rail-step__arrow {
  flex-shrink: 0;
  align-self: center;
  color: $ink-3;
  opacity: 0;
  transform: translateX(-4px);
  transition: all 0.2s ease;

  .rail-step--done & { color: $signal; }
}

// ---------------------------------------------------------------------------
// RIGHT WORKSPACE — all sections stacked
// ---------------------------------------------------------------------------
.work {
  display: flex;
  flex-direction: column;
  min-height: 0;
  background: $paper;
}

.work__scroll {
  flex: 1;
  min-height: 0;
  overflow-y: auto;
  padding: 32px 36px;
}

.work__inner {
  max-width: 1000px;
  margin: 0 auto;
}

.section {
  padding-bottom: 40px;
  margin-bottom: 40px;
  border-bottom: 1px dashed $line;
  animation: cm-fade-up 0.28s ease;
}
.section--last { margin-bottom: 0; }

.work__eyebrow {
  @extend %micro;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  color: $ink-3;
  margin-bottom: 8px;
}

.work__title {
  font-family: $display;
  font-size: 1.6923em; // 1.375rem / 0.8125rem
  font-weight: 800;
  letter-spacing: -0.02em;
  color: $ink;
  line-height: 1.2;
}

.work__desc {
  margin-top: 6px;
  margin-bottom: 26px;
  font-size: 1.1538em; // 0.9375rem / 0.8125rem
  color: $ink-2;
  line-height: 1.5;
}

// ---------------------------------------------------------------------------
// sticky footer (Cancel / Run Validation / Save)
// ---------------------------------------------------------------------------
.work__foot {
  flex-shrink: 0;
  position: sticky;
  bottom: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  padding: 16px 36px;
  background: $card;
  border-top: 1px solid $line;
  z-index: 5;
}

.work__foot-info {
  font-family: $mono;
  font-size: 0.9231em; // 0.75rem / 0.8125rem
  font-weight: 700;
  color: $ink-3;
}

.work__foot-actions {
  display: flex;
  align-items: center;
  gap: 10px;
}

// ---------------------------------------------------------------------------
// buttons
// ---------------------------------------------------------------------------
.btn {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  padding: 10px 18px;
  border-radius: 10px;
  border: 1px solid $line;
  background: $card;
  color: $ink-2;
  font-family: $sans;
  font-size: 1.0385em; // 0.84375rem / 0.8125rem
  font-weight: 650;
  cursor: pointer;
  transition: all 0.15s ease;
  white-space: nowrap;

  &:hover:not(:disabled) { border-color: $ink-3; color: $ink; }
  &:disabled { opacity: 0.45; cursor: not-allowed; }
}

.btn--sm { padding: 7px 12px; font-size: 0.9615em; border-radius: 8px; } // 0.78125rem / 0.8125rem

.btn--primary {
  border-color: $signal;
  background: $signal;
  color: #fff;
  &:hover:not(:disabled) { background: $signal-2; border-color: $signal-2; color: #fff; transform: translateY(-1px); box-shadow: $lift; }
}

.btn--ghost { background: transparent; border-color: transparent; &:hover:not(:disabled) { background: $paper; border-color: $line; } }

.btn--ok {
  border-color: $ok; background: $ok; color: #fff;
  &:hover:not(:disabled) { filter: brightness(0.95); color: #fff; transform: translateY(-1px); box-shadow: $lift; }
}

.btn-icon {
  display: inline-flex; align-items: center; justify-content: center;
  width: 30px; height: 30px; border-radius: 8px;
  border: 1px solid transparent; background: transparent; color: $ink-3; cursor: pointer;
  transition: all 0.15s ease;
  &:hover { background: $danger-wash; border-color: rgba($danger, 0.2); color: $danger; }
}

// ---------------------------------------------------------------------------
// forms
// ---------------------------------------------------------------------------
.field { margin-bottom: 20px; }

.field__label {
  display: block;
  @extend %micro;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  color: $ink-2;
  margin-bottom: 8px;
}
.field__hint { font-size: 0.9615em; color: $ink-3; margin-top: 6px; } // 0.78125rem / 0.8125rem

.input, .textarea {
  width: 100%;
  border: 1.5px solid $line;
  border-radius: 10px;
  padding: 11px 13px;
  font-size: 1.1538em; // 0.9375rem / 0.8125rem
  font-family: $sans;
  color: $ink;
  background: $card;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;

  &::placeholder { color: $ink-3; }
  &:focus { outline: none; border-color: $signal; box-shadow: 0 0 0 3px $wash; }
}
.textarea { resize: vertical; min-height: 92px; line-height: 1.55; }

// ---------------------------------------------------------------------------
// selectable option cards (eval type / metric type)
// ---------------------------------------------------------------------------
.opt-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 12px;
}
.opt-grid--3 { grid-template-columns: repeat(3, 1fr); }

.opt {
  position: relative;
  text-align: left;
  border: 1.5px solid $line;
  border-radius: 16px;
  padding: 18px;
  cursor: pointer;
  background: $card;
  transition: border-color 0.16s ease, box-shadow 0.16s ease, transform 0.16s ease, background 0.16s ease;

  &:hover:not(&--disabled) { border-color: $ink-3; transform: translateY(-2px); box-shadow: $lift; }

  &--selected {
    border-color: $signal;
    background: $wash;
    box-shadow: 0 0 0 1px $signal inset;
  }
  &--disabled { opacity: 0.5; cursor: not-allowed; }
}

.opt__icon {
  width: 42px;
  height: 42px;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: $paper;
  border: 1px solid $line;
  color: $signal;
  margin-bottom: 12px;
  transition: all 0.16s ease;

  .opt--selected & { background: $signal; border-color: $signal; color: #fff; }
}

.opt__title {
  font-family: $display;
  font-weight: 700;
  font-size: 1.2308em; // 1.0rem / 0.8125rem
  color: $ink;
  margin-bottom: 4px;
}
.opt__desc { font-size: 1em; color: $ink-2; line-height: 1.45; } // 0.8125rem / 0.8125rem

.opt__check {
  position: absolute;
  top: 14px; right: 14px;
  width: 20px; height: 20px;
  border-radius: 50%;
  background: $signal;
  color: #fff;
  display: flex; align-items: center; justify-content: center;
  animation: cm-pop 0.22s ease;
}

// ---------------------------------------------------------------------------
// rule builder — each rule is its own card with a solid accent bar, a
// header row (index pill + remove), and a clean fields grid underneath.
// ---------------------------------------------------------------------------
.rules { display: flex; flex-direction: column; gap: 14px; }

.rule {
  position: relative;
  padding: 16px 18px 18px;
  border: 1.5px solid $line;
  border-radius: 16px;
  background: $card;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;

  &:hover { border-color: $ink-3; box-shadow: $soft; }
}

.rule__head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  margin-bottom: 14px;
  padding-bottom: 12px;
  border-bottom: 1px dashed $line;
}

.rule__index {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  font-family: $mono;
  font-size: 0.9231em; // 0.75rem / 0.8125rem
  font-weight: 800;
  letter-spacing: 0.06em;
  text-transform: uppercase;
  color: $signal;

  &::before {
    content: '';
    width: 7px;
    height: 7px;
    border-radius: 50%;
    background: $signal;
  }
}

.rule__grid {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr 1fr;
  gap: 12px;
  min-width: 0;
}

.rule__field {
  display: flex;
  flex-direction: column;
  gap: 7px;
  min-width: 0;
}

.rule__field-label {
  @extend %micro;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  color: $ink-3;
}

.gate {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 4px 0;

  &::before, &::after { content: ''; flex: 1; height: 1px; background: $line; }
}

.gate__toggle {
  display: inline-flex;
  padding: 2px;
  background: $card;
  border: 1px solid $line;
  border-radius: 8px;
  gap: 1px;
}

.gate__opt {
  padding: 4px 12px;
  border-radius: 6px;
  border: none;
  background: transparent;
  color: $ink-2;
  font-family: $mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  cursor: pointer;
  &.on { background: $signal; color: #fff; }
}

.add-rule { align-self: flex-start; margin-top: 14px; }

.summary {
  margin-top: 18px;
  padding: 16px;
  border-radius: 12px;
  background: $paper;
  border: 1px solid $line;
}
.summary__label {
  @extend %micro;
  font-size: 0.6923em; // 0.5625rem / 0.8125rem
  color: $ink-3;
  margin-bottom: 8px;
}
.summary__code {
  font-family: $mono;
  font-size: 1.0385em; // 0.84375rem / 0.8125rem
  color: $ink;
  line-height: 1.7;
  word-break: break-word;
}
.summary__token { color: $signal; font-weight: 700; }
.summary__gate { color: $amber; font-weight: 700; padding: 0 4px; }


// ---------------------------------------------------------------------------
// prompt templates
// ---------------------------------------------------------------------------
.tpl-list { display: flex; flex-direction: column; gap: 8px; margin-bottom: 18px; }

.tpl {
  display: flex;
  gap: 12px;
  padding: 14px;
  border: 1.5px solid $line;
  border-radius: 12px;
  cursor: pointer;
  background: $card;
  transition: border-color 0.15s ease, background 0.15s ease;

  &:hover { border-color: $ink-3; }
  &--selected { border-color: $signal; background: $wash; }
}

.tpl__radio {
  flex-shrink: 0;
  width: 18px; height: 18px;
  margin-top: 1px;
  border-radius: 50%;
  border: 1.5px solid $line-2;
  display: flex; align-items: center; justify-content: center;

  .tpl--selected & { border-color: $signal; }
  &::after { content: ''; width: 9px; height: 9px; border-radius: 50%; background: $signal; opacity: 0; transition: opacity 0.15s ease; }
  .tpl--selected &::after { opacity: 1; }
}

.tpl__label { font-weight: 700; font-size: 1.1538em; color: $ink; } // 0.9375rem / 0.8125rem
.tpl__desc { font-size: 1em; color: $ink-2; margin-top: 3px; line-height: 1.4; } // 0.8125rem / 0.8125rem
.tpl__tags { display: flex; flex-wrap: wrap; gap: 5px; margin-top: 8px; }

.token {
  font-family: $mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  color: $signal;
  background: $wash;
  border: 1px solid rgba($signal, 0.16);
  border-radius: 999px;
  padding: 3px 9px;
}

// ---------------------------------------------------------------------------
// judge model list
// ---------------------------------------------------------------------------
.models { display: flex; flex-direction: column; gap: 8px; }

.model {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 13px 15px;
  border: 1.5px solid $line;
  border-radius: 12px;
  cursor: pointer;
  background: $card;
  transition: border-color 0.15s ease, background 0.15s ease, opacity 0.15s ease;

  &:hover:not(&--disabled) { border-color: $ink-3; }
  &--selected { border-color: $signal; background: $wash; }
  &--disabled { opacity: 0.5; cursor: not-allowed; }
}

.model__radio {
  flex-shrink: 0;
  width: 18px; height: 18px;
  border-radius: 50%;
  border: 1.5px solid $line-2;
  display: flex; align-items: center; justify-content: center;
  .model--selected & { border-color: $signal; }
  &::after { content: ''; width: 9px; height: 9px; border-radius: 50%; background: $signal; opacity: 0; transition: opacity 0.15s ease; }
  .model--selected &::after { opacity: 1; }
}

.model__name { font-family: $display; font-weight: 700; font-size: 1.1538em; color: $ink; } // 0.9375rem / 0.8125rem
.model__meta { font-family: $mono; font-size: 0.8462em; color: $ink-3; margin-top: 1px; } // 0.6875rem / 0.8125rem

.model__health {
  margin-left: auto;
  display: inline-flex; align-items: center; gap: 6px;
  font-family: $mono; font-size: 0.7692em; font-weight: 700; text-transform: uppercase; // 0.625rem / 0.8125rem
}
.health-dot { width: 7px; height: 7px; border-radius: 50%; flex-shrink: 0; }
.health--healthy { color: $ok; .health-dot { background: $ok; } }
.health--unhealthy { color: $danger; .health-dot { background: $danger; } }
.health--checking { color: $ink-3; .health-dot { background: $ink-3; animation: cm-spin 1s linear infinite; border-radius: 2px; } }

// ---------------------------------------------------------------------------
// code editor
// ---------------------------------------------------------------------------
.code {
  border: 1px solid $line;
  border-radius: 14px;
  overflow: hidden;
}
.code__bar {
  display: flex; align-items: center; justify-content: space-between;
  padding: 10px 14px;
  background: $ink-solid;
  color: rgba(255, 255, 255, 0.7);
}
.code__lang {
  @extend %micro;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  color: rgba(255, 255, 255, 0.55);
}
.code__area {
  width: 100%;
  min-height: 340px;
  border: none;
  resize: vertical;
  padding: 16px;
  font-family: $mono;
  font-size: 1.0000em; // 0.8125rem / 0.8125rem
  line-height: 1.65;
  color: $ink;
  background: $card;
  &:focus { outline: none; }
}

// ---------------------------------------------------------------------------
// threshold slider
// ---------------------------------------------------------------------------
.thr {
  padding: 24px;
  border: 1px solid $line;
  border-radius: 16px;
  background: $card;
}
.thr__value {
  font-family: $mono;
  font-size: 3.0769em; // 2.5rem / 0.8125rem
  font-weight: 700;
  color: $signal;
  line-height: 1;
  text-align: center;
  margin-bottom: 4px;
}
.thr__cap { text-align: center; font-size: 0.9615em; color: $ink-3; margin-bottom: 20px; } // 0.78125rem / 0.8125rem
.thr__slider {
  -webkit-appearance: none;
  appearance: none;
  width: 100%;
  height: 6px;
  border-radius: 999px;
  background: $line;
  outline: none;
  cursor: pointer;

  &::-webkit-slider-thumb {
    -webkit-appearance: none;
    appearance: none;
    width: 22px; height: 22px;
    border-radius: 50%;
    background: $signal;
    border: 3px solid $card;
    box-shadow: 0 2px 6px rgba(43, 43, 245, 0.4);
    cursor: pointer;
  }
  &::-moz-range-thumb {
    width: 22px; height: 22px;
    border-radius: 50%;
    background: $signal;
    border: 3px solid $card;
    cursor: pointer;
  }
}
.thr__scale {
  display: flex;
  justify-content: space-between;
  margin-top: 8px;
  font-family: $mono;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  color: $ink-3;
}

// ---------------------------------------------------------------------------
// dataset + preview (side by side, card-style columns)
// ---------------------------------------------------------------------------
.data-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 20px;
  align-items: start;
  margin-bottom: 8px;
}

.data-col {
  min-width: 0;
  border: 1px solid $line;
  border-radius: 16px;
  background: $card;
  overflow: hidden;
  box-shadow: $soft;
}

.data-col__head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  padding: 12px 16px;
  background: $paper;
  border-bottom: 1px solid $line;
}

.data-col__head-title {
  @extend %micro;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  color: $ink-3;
  display: flex;
  align-items: center;
  gap: 6px;
}

.data-col__count {
  font-family: $mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  color: $signal;
  background: $wash;
  border: 1px solid rgba($signal, 0.18);
  border-radius: 999px;
  padding: 2px 9px;
}

.data-col__body {
  padding: 12px;
  max-height: 400px;
  overflow-y: auto;
}

// ---- dataset cards — grid of self-sizing tiles; as many fit per row as
// space allows, wrapping to the next row otherwise. Name and count are
// stacked (not squeezed onto one line), with a top icon chip and a
// corner check badge that pops in when selected. ----------------------
.ds-list {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(160px, 1fr));
  gap: 10px;
}

.ds {
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 10px;
  min-width: 0;
  padding: 14px 14px 13px;
  border: 1.5px solid $line;
  border-radius: 14px;
  background: $card;
  cursor: pointer;
  transition: border-color 0.16s ease, background 0.16s ease, transform 0.16s ease, box-shadow 0.16s ease;

  &:hover { border-color: $ink-3; transform: translateY(-2px); box-shadow: $soft; }

  &--selected {
    border-color: $signal;
    background: $wash;
    box-shadow: 0 0 0 1px $signal inset;
  }
}

.ds__icon {
  flex-shrink: 0;
  width: 30px;
  height: 30px;
  border-radius: 9px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: $paper;
  border: 1px solid $line;
  color: $signal;
  transition: all 0.16s ease;

  .ds--selected & { background: $signal; border-color: $signal; color: #fff; }
}

.ds__name {
  width: 100%;
  font-family: $display;
  font-weight: 700;
  font-size: 1.0769em; // 0.875rem / 0.8125rem
  color: $ink;
  line-height: 1.3;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.ds__count {
  align-self: flex-start;
  display: inline-flex;
  align-items: center;
  font-family: $mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  color: $ink-3;
  background: $paper;
  border: 1px solid $line;
  border-radius: 999px;
  padding: 3px 9px;
  white-space: nowrap;
  transition: all 0.16s ease;

  .ds--selected & { color: $signal; background: rgba(255, 255, 255, 0.6); border-color: rgba($signal, 0.3); }
}

.ds__check {
  position: absolute;
  top: 10px;
  right: 10px;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: $signal;
  color: #fff;
  display: flex;
  align-items: center;
  justify-content: center;
  opacity: 0;
  transform: scale(0.5);
  transition: all 0.18s cubic-bezier(0.34, 1.56, 0.64, 1);

  .ds--selected & { opacity: 1; transform: scale(1); }
}

.q-list { display: flex; flex-direction: column; gap: 8px; }

.q {
  display: flex; gap: 10px;
  padding: 12px 14px;
  border: 1px solid $line;
  border-radius: 12px;
  background: $card;
  cursor: pointer;
  transition: border-color 0.13s ease, background 0.13s ease;
  &:hover { border-color: $ink-3; background: $paper; }
  &--on { border-color: $signal; background: $wash; }
}
.q__check {
  flex-shrink: 0; width: 17px; height: 17px; margin-top: 2px;
  border-radius: 5px; border: 1.5px solid $line-2;
  display: flex; align-items: center; justify-content: center;
  color: #fff;
  transition: all 0.13s ease;
  .q--on & { background: $signal; border-color: $signal; }
}
.q__body { min-width: 0; }
.q__q { display: block; font-size: 1.0385em; color: $ink; font-weight: 600; margin-bottom: 3px; } // 0.84375rem / 0.8125rem
.q__a { display: block; font-size: 0.9615em; color: $ink-2; } // 0.78125rem / 0.8125rem
.q__a-label { font-family: $mono; font-size: 0.7692em; color: $ink-3; margin-right: 5px; } // 0.625rem / 0.8125rem

.link-btn {
  border: none; background: none; padding: 0;
  color: $signal; font-size: 0.8846em; font-weight: 650; cursor: pointer; // 0.71875rem / 0.8125rem
  &:hover { text-decoration: underline; }
}

// ---------------------------------------------------------------------------
// validate & save
// ---------------------------------------------------------------------------
.validate-section {
  margin-top: 32px;
  padding-top: 24px;
  border-top: 1px dashed $line;
}

.validate-section__label {
  @extend %micro;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  color: $ink-2;
  margin-bottom: 6px;
}

.validate-section__desc {
  font-size: 1.0385em; // 0.84375rem / 0.8125rem
  color: $ink-2;
  margin-bottom: 16px;
}

// ---------------------------------------------------------------------------
// validation results
// ---------------------------------------------------------------------------
.banner {
  display: flex; align-items: center; gap: 8px;
  padding: 12px 16px; border-radius: 12px;
  font-size: 1.0385em; font-weight: 600; // 0.84375rem / 0.8125rem
  margin-bottom: 18px;
}
.banner--ok { background: $ok-wash; color: $ok; border: 1px solid rgba($ok, 0.2); }
.banner--err { background: $danger-wash; color: $danger; border: 1px solid rgba($danger, 0.2); }
.banner--info { background: $wash; color: $signal; border: 1px solid rgba($signal, 0.18); }

.results {
  border: 1px solid $line;
  border-radius: 14px;
  overflow: hidden;
}
.results__row {
  display: flex; align-items: flex-start; gap: 14px;
  padding: 14px 16px;
  border-bottom: 1px solid $line-2;
  &:last-child { border-bottom: none; }
}
.results__score {
  flex-shrink: 0;
  font-family: $mono; font-weight: 700; font-size: 1.2308em; // 1.0rem / 0.8125rem
  width: 48px; text-align: center;
}
.results__score--pass { color: $ok; }
.results__score--fail { color: $danger; }
.results__body { min-width: 0; flex: 1; }
.results__io { font-size: 1em; color: $ink; } // 0.8125rem / 0.8125rem
.results__reason { font-size: 0.9615em; color: $ink-2; margin-top: 4px; font-style: italic; } // 0.78125rem / 0.8125rem
.results__pill {
  flex-shrink: 0;
  @extend %micro;
  font-size: 0.6923em; // 0.5625rem / 0.8125rem
  padding: 3px 9px; border-radius: 999px;
}
.results__pill--pass { color: $ok; background: $ok-wash; }
.results__pill--fail { color: $danger; background: $danger-wash; }
.results__summary {
  display: flex; gap: 20px;
  padding: 12px 16px;
  background: $paper;
  border-top: 1px solid $line;
  font-size: 1.0385em; color: $ink-2; // 0.84375rem / 0.8125rem
  strong { color: $ink; font-family: $mono; }
}

// ---------------------------------------------------------------------------
// misc states
// ---------------------------------------------------------------------------
.loading, .empty {
  display: flex; align-items: center; justify-content: center; gap: 8px;
  padding: 28px; text-align: center;
  color: $ink-3; font-size: 1.0385em; // 0.84375rem / 0.8125rem
  border: 1px dashed $line;
  border-radius: 12px;
}

// ---------------------------------------------------------------------------
// success modal
// ---------------------------------------------------------------------------
.overlay {
  position: fixed; inset: 0; z-index: 300;
  display: flex; align-items: center; justify-content: center;
  background: rgba(10, 12, 18, 0.55);
  padding: 20px;
}
.modal {
  width: 100%; max-width: 400px;
  background: $card;
  border-radius: 20px;
  box-shadow: $lift;
  padding: 32px 28px 24px;
  text-align: center;
  animation: cm-modal-in 0.24s ease;
}
.modal__icon {
  width: 56px; height: 56px; margin: 0 auto 16px;
  border-radius: 50%;
  background: $ok-wash; color: $ok;
  display: flex; align-items: center; justify-content: center;
  animation: cm-pop 0.3s ease;
}
.modal__title { font-family: $display; font-size: 1.5385em; font-weight: 800; color: $ink; margin-bottom: 6px; } // 1.25rem / 0.8125rem
.modal__text { font-size: 1.0769em; color: $ink-2; margin-bottom: 8px; } // 0.8125rem / 0.8125rem
.modal__id {
  display: inline-block;
  font-family: $mono; font-size: 0.9231em; font-weight: 700; // 0.75rem / 0.8125rem
  color: $signal; background: $wash;
  border-radius: 8px; padding: 4px 10px; margin-bottom: 22px;
}
.modal__actions { display: flex; gap: 10px; }
.modal__actions .btn { flex: 1; justify-content: center; }

// ---------------------------------------------------------------------------
// toast (save error)
// ---------------------------------------------------------------------------
.toast {
  position: fixed;
  left: 50%; bottom: 26px;
  transform: translateX(-50%);
  z-index: 320;
  display: flex; align-items: center; gap: 8px;
  padding: 12px 18px;
  border-radius: 12px;
  background: $ink-solid; color: #fff;
  font-size: 1.0385em; font-weight: 600; // 0.84375rem / 0.8125rem
  box-shadow: $lift;
  animation: cm-fade-up 0.2s ease;
}

// ---------------------------------------------------------------------------
// responsive
// ---------------------------------------------------------------------------
@media (max-width: 1080px) {
  .builder { grid-template-columns: 1fr; }
  .rail {
    border-right: none;
    border-bottom: 1px solid $line;
    max-height: none;
  }
  .rail__steps { flex-direction: row; overflow-x: auto; }
  .rail-step { flex-direction: column; align-items: flex-start; min-width: 130px; }
  .rail-step:not(:last-child)::after { display: none; }
}

@media (max-width: 760px) {
  .page-header { padding: 16px 18px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .work__scroll { padding: 22px 18px; }
  .work__foot { padding: 14px 18px; }
  .opt-grid, .opt-grid--3 { grid-template-columns: 1fr; }
  .data-row { grid-template-columns: 1fr; }
  .ds-list { grid-template-columns: repeat(auto-fill, minmax(130px, 1fr)); }
  .rule__grid { grid-template-columns: 1fr 1fr; }
}
