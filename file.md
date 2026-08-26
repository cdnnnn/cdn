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
                  <div className="pills">
                    {evalTypeOptions.map((t) => (
                      <button key={t} className={`pill ${evalFilter === t ? 'on' : ''}`} onClick={() => setEvalFilter(t)}>
                        {t === 'All' ? 'All' : t.toUpperCase()}
                      </button>
                    ))}
                  </div>
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
  color: $danger; font-size: 0.8125rem; margin-bottom: 16px;
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
.toast--error::before { content: ''; width: 6px; height: 6px; border-radius: 50%; background: #FF6B6B; }
.toast--info::before { content: ''; width: 6px; height: 6px; border-radius: 50%; background: $signal; }

@media (max-width: 768px) {
  .custom-metrics__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
}


















//Createmetric.tsx
import { useEffect, useMemo, useRef, useState } from 'react';
import {
  AlertCircle, ArrowRight, Check, CheckCircle2, ChevronRight, Code2, Cpu, Database,
  ListChecks, Loader2, Plus, ScrollText, SlidersHorizontal, Sparkles, Target, X, XCircle, Zap,
} from 'lucide-react';
import styles from './CreateMetric.module.scss';
import { useToast } from './useToast';
import CustomSelect from './CustomSelect';
import {
  metricsApi, EvalType, MetricType, PromptTemplate,
  ModelSummary, DatasetSummary, PreviewQuestion, ValidateMetricData, RuleDef,
} from '../../api/endpoints/metrics';

interface CreateMetricProps {
  onCancel: () => void;
  onSaved: (id: string) => void;
}

// ---- static config -----------------------------------------------------
const EVAL_TYPE_CARDS: { key: EvalType; label: string; desc: string; icon: JSX.Element }[] = [
  { key: 'model', label: 'Model', desc: 'Score a model\u2019s output against an expected answer.', icon: <Cpu size={20} /> },
  { key: 'agent', label: 'Agent', desc: 'Evaluate tool calls and task completion for agents.', icon: <Zap size={20} /> },
  { key: 'rag', label: 'RAG', desc: 'Check answers grounded in retrieved context.', icon: <ScrollText size={20} /> },
];

const METRIC_TYPE_CARDS: { key: MetricType; label: string; desc: string; icon: JSX.Element }[] = [
  { key: 'visual', label: 'Visual Builder', desc: 'Field comparisons joined with AND/OR logic. No code.', icon: <SlidersHorizontal size={18} /> },
  { key: 'prompt', label: 'Prompt Builder', desc: 'An LLM judge scored with a prompt template.', icon: <Sparkles size={18} /> },
  { key: 'code', label: 'Code Editor', desc: 'A custom Python scoring function.', icon: <Code2 size={18} /> },
  { key: 'simple', label: 'Simple', desc: 'A minimal, zero-config pass/fail check.', icon: <Target size={18} /> },
];

const FIELDS_BY_EVAL_TYPE: Record<EvalType, string[]> = {
  model: ['input', 'actual_output', 'expected_output'],
  agent: ['input', 'actual_output', 'expected_output', 'tools_called', 'expected_tools'],
  rag: ['input', 'actual_output', 'expected_output', 'tools_called', 'expected_tools'],
};

const OPERATORS = [
  { value: 'contains', label: 'contains' },
  { value: 'not_contains', label: 'not contains' },
  { value: 'equals', label: 'equals' },
  { value: 'starts_with', label: 'starts with' },
  { value: 'ends_with', label: 'ends with' },
  { value: 'greater_than', label: 'greater than' },
  { value: 'less_than', label: 'less than' },
  { value: 'regex_match', label: 'regex match' },
];

const OP_SYMBOL: Record<string, string> = {
  contains: 'contains', not_contains: 'does not contain', equals: '==', starts_with: 'starts with',
  ends_with: 'ends with', greater_than: '>', less_than: '<', regex_match: 'matches',
};

const METRIC_TYPE_TO_API: Record<MetricType, string> = {
  visual: 'condition', prompt: 'prompt', code: 'code', simple: 'simple',
};

const EVAL_TYPE_TO_CATEGORY: Record<EvalType, string> = { model: 'llm', agent: 'agent', rag: 'rag' };

type CompareType = 'field' | 'literal';
interface RuleRow { id: number; field: string; operator: string; compareType: CompareType; value: string; }
let ruleSeq = 1;

type SectionKey = 'details' | 'type' | 'config' | 'dataset';
interface SectionDef { key: SectionKey; label: string; }

export default function CreateMetric({ onCancel, onSaved }: CreateMetricProps) {
  const { showToast, ToastEl } = useToast();

  // section refs for the rail's "jump to" links
  const sectionRefs = {
    details: useRef<HTMLDivElement>(null),
    type: useRef<HTMLDivElement>(null),
    config: useRef<HTMLDivElement>(null),
    dataset: useRef<HTMLDivElement>(null),
  };
  const scrollToSection = (key: SectionKey) => {
    sectionRefs[key].current?.scrollIntoView({ behavior: 'smooth', block: 'start' });
  };

  // details
  const [name, setName] = useState('');
  const [description, setDescription] = useState('');

  // type
  const [evalType, setEvalType] = useState<EvalType | null>(null);
  const [metricType, setMetricType] = useState<MetricType | null>(null);

  // config: visual
  const [rules, setRules] = useState<RuleRow[]>([{ id: ruleSeq, field: 'actual_output', operator: 'contains', compareType: 'field', value: 'input' }]);
  const [gates, setGates] = useState<('AND' | 'OR')[]>([]);

  // config: prompt
  const [promptTemplates, setPromptTemplates] = useState<PromptTemplate[]>([]);
  const [promptTemplatesLoading, setPromptTemplatesLoading] = useState(false);
  const [promptTemplatesError, setPromptTemplatesError] = useState('');
  const [selectedTemplateName, setSelectedTemplateName] = useState('');
  const [promptText, setPromptText] = useState('');
  const [models, setModels] = useState<ModelSummary[]>([]);
  const [modelsLoading, setModelsLoading] = useState(false);
  const [modelsError, setModelsError] = useState('');
  const [modelHealth, setModelHealth] = useState<Record<string, 'checking' | 'healthy' | 'unhealthy'>>({});
  const [selectedModelId, setSelectedModelId] = useState('');

  // config: code
  const [code, setCode] = useState('');
  const [codeLoading, setCodeLoading] = useState(false);
  const [codeError, setCodeError] = useState('');

  // threshold (shared across all config types)
  const [threshold, setThreshold] = useState(0.7);

  // dataset
  const [datasets, setDatasets] = useState<DatasetSummary[]>([]);
  const [datasetsLoading, setDatasetsLoading] = useState(false);
  const [datasetsError, setDatasetsError] = useState('');
  const [selectedDatasetId, setSelectedDatasetId] = useState('');
  const [previewQuestions, setPreviewQuestions] = useState<PreviewQuestion[]>([]);
  const [previewLoading, setPreviewLoading] = useState(false);
  const [previewError, setPreviewError] = useState('');
  const [selectedQuestionIds, setSelectedQuestionIds] = useState<Set<string>>(new Set());

  // validate / save
  const [validating, setValidating] = useState(false);
  const [validateError, setValidateError] = useState('');
  const [validateResult, setValidateResult] = useState<ValidateMetricData | null>(null);
  const [saving, setSaving] = useState(false);
  const [saveError, setSaveError] = useState('');
  const [savedId, setSavedId] = useState('');

  const fields = evalType ? FIELDS_BY_EVAL_TYPE[evalType] : [];

  // ---- reset chains ------------------------------------------------------
  const handleEvalType = (t: EvalType) => {
    if (t === evalType) return;
    setEvalType(t);
    setDatasets([]); setSelectedDatasetId(''); setPreviewQuestions([]); setSelectedQuestionIds(new Set());
    setCode(''); setPromptText(''); setSelectedTemplateName(''); setValidateResult(null); setSavedId('');
  };
  const handleMetricType = (t: MetricType) => {
    if (t === metricType) return;
    setMetricType(t); setValidateResult(null); setSavedId('');
  };

  // ---- visual rules ------------------------------------------------------
  const addRule = () => {
    ruleSeq += 1;
    setRules((r) => [...r, { id: ruleSeq, field: fields[0] || 'input', operator: 'contains', compareType: 'literal', value: '' }]);
    setGates((g) => [...g, 'AND']);
  };
  const removeRule = (id: number) => {
    setRules((r) => {
      if (r.length <= 1) return r;
      const idx = r.findIndex((row) => row.id === id);
      setGates((g) => g.filter((_, i) => i !== Math.max(0, idx - 1)));
      return r.filter((row) => row.id !== id);
    });
  };
  const updateRule = (id: number, patch: Partial<RuleRow>) => setRules((r) => r.map((row) => (row.id === id ? { ...row, ...patch } : row)));
  const toggleGate = (idx: number) => setGates((g) => g.map((v, i) => (i === idx ? (v === 'AND' ? 'OR' : 'AND') : v)));

  // ---- prompt templates + models ----------------------------------------
  useEffect(() => {
    if (metricType !== 'prompt' || promptTemplates.length) return;
    setPromptTemplatesLoading(true); setPromptTemplatesError('');
    metricsApi.getPromptTemplates()
      .then(setPromptTemplates)
      .catch((e) => setPromptTemplatesError(e.message || 'Failed to load templates'))
      .finally(() => setPromptTemplatesLoading(false));
  }, [metricType, promptTemplates.length]);

  const matchingTemplates = useMemo(
    () => promptTemplates.filter((t) => t.category === (evalType ? EVAL_TYPE_TO_CATEGORY[evalType] : '')),
    [promptTemplates, evalType],
  );
  const allowsCustomPrompt = evalType === 'agent' || evalType === 'rag';

  useEffect(() => {
    if (metricType !== 'prompt' || models.length) return;
    setModelsLoading(true); setModelsError('');
    metricsApi.listModels()
      .then((list) => {
        setModels(list);
        const init: Record<string, 'checking'> = {};
        list.forEach((m) => { init[m.id] = 'checking'; });
        setModelHealth(init);
        list.forEach((m) => metricsApi.checkModelHealth(m.id).then((h) =>
          setModelHealth((prev) => ({ ...prev, [m.id]: h.success ? 'healthy' : 'unhealthy' }))));
      })
      .catch((e) => setModelsError(e.message || 'Failed to load models'))
      .finally(() => setModelsLoading(false));
  }, [metricType, models.length]);

  // ---- code template -----------------------------------------------------
  useEffect(() => {
    if (metricType !== 'code' || !evalType || code) return;
    setCodeLoading(true); setCodeError('');
    metricsApi.getCodeTemplate(evalType)
      .then((res) => setCode(res.code))
      .catch((e) => setCodeError(e.message || 'Failed to load starter code'))
      .finally(() => setCodeLoading(false));
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [metricType, evalType]);

  // ---- datasets ----------------------------------------------------------
  useEffect(() => {
    if (!evalType) return;
    setDatasetsLoading(true); setDatasetsError(''); setSelectedDatasetId(''); setPreviewQuestions([]);
    metricsApi.listDatasets(evalType)
      .then(setDatasets)
      .catch((e) => setDatasetsError(e.message || 'Failed to load datasets'))
      .finally(() => setDatasetsLoading(false));
  }, [evalType]);

  const selectDataset = (id: string) => {
    setSelectedDatasetId(id); setValidateResult(null); setSavedId('');
    setPreviewLoading(true); setPreviewError('');
    metricsApi.previewDataset(id)
      .then((res) => {
        const qs = res.questions.slice(0, 5);
        setPreviewQuestions(qs);
        setSelectedQuestionIds(new Set(qs.map((q) => q.id)));
      })
      .catch((e) => setPreviewError(e.message || 'Failed to load preview'))
      .finally(() => setPreviewLoading(false));
  };
  const toggleQuestion = (id: string) => setSelectedQuestionIds((prev) => {
    const next = new Set(prev); next.has(id) ? next.delete(id) : next.add(id); return next;
  });
  const selectAllQuestions = () => setSelectedQuestionIds(new Set(previewQuestions.map((q) => q.id)));
  const clearAllQuestions = () => setSelectedQuestionIds(new Set());

  // ---- rule summary ------------------------------------------------------
  const ruleSummary = useMemo(() => {
    if (!rules.length) return null;
    return rules.map((r, i) => {
      const compare = r.compareType === 'field' ? (r.value || '<field>') : `"${r.value || '…'}"`;
      return (
        <span key={r.id}>
          {i > 0 && <span className={styles['summary__gate']}>{gates[i - 1] || 'AND'}</span>}
          <span className={styles['summary__token']}>{r.field}</span>
          {' '}{OP_SYMBOL[r.operator] || r.operator}{' '}
          <span className={styles['summary__token']}>{compare}</span>
        </span>
      );
    });
  }, [rules, gates]);

  // ---- gating (used for status dots + validate button, not for hiding UI) ---
  const detailsComplete = !!name.trim();
  const typeComplete = !!evalType && !!metricType;
  const configComplete = useMemo(() => {
    if (!metricType) return false;
    if (metricType === 'visual') return rules.every((r) => r.field && r.operator && (r.compareType === 'field' ? r.value : r.value.trim()));
    if (metricType === 'prompt') return !!promptText.trim() && !!selectedModelId;
    if (metricType === 'code') return !!code.trim();
    return true;
  }, [metricType, rules, promptText, selectedModelId, code]);
  const datasetComplete = !!selectedDatasetId && selectedQuestionIds.size > 0;
  const canValidate = detailsComplete && typeComplete && configComplete && datasetComplete && threshold >= 0 && threshold <= 1;
  const validateSucceeded = !!validateResult && validateResult.passed > 0;

  const SECTIONS: SectionDef[] = [
    { key: 'details', label: 'Metric Details' },
    { key: 'type', label: 'Type & Target' },
    { key: 'config', label: metricType === 'prompt' ? 'Judge Prompt' : metricType === 'code' ? 'Scoring Code' : metricType === 'simple' ? 'Configuration' : 'Rules' },
    { key: 'dataset', label: 'Dataset · Validate & Save' },
  ];

  const sectionDone: Record<SectionKey, boolean> = {
    details: detailsComplete,
    type: typeComplete,
    config: configComplete,
    dataset: datasetComplete && !!validateResult,
  };

  const sectionValue: Record<SectionKey, string> = {
    details: name || 'Not set',
    type: evalType && metricType ? `${evalType.toUpperCase()} · ${METRIC_TYPE_CARDS.find((c) => c.key === metricType)!.label}` : 'Not set',
    config: metricType ? (configComplete ? 'Configured' : 'Incomplete') : '—',
    dataset: validateResult ? `${validateResult.passed}/${validateResult.total} passed` : (selectedDatasetId ? `${selectedQuestionIds.size} selected` : 'Not set'),
  };

  const completedCount = SECTIONS.filter((s) => sectionDone[s.key]).length;

  // ---- validate / save ---------------------------------------------------
  const buildDefinition = () => {
    if (metricType === 'visual') return { rules: rules.map<RuleDef>((r) => ({ field: r.field, operator: r.operator, value: r.value, compare_to_field: r.compareType === 'field' })) };
    if (metricType === 'prompt') return { prompt_template: promptText };
    if (metricType === 'code') return { code, skip_validation: true };
    return {};
  };

  const runValidate = () => {
    if (!canValidate || !evalType || !metricType) { showToast('Complete every section first', 'error'); return; }
    setValidating(true); setValidateError(''); setValidateResult(null);
    const selectedQs = previewQuestions.filter((q) => selectedQuestionIds.has(q.id));
    metricsApi.validate({
      actual_output: '', context: [], definition: buildDefinition(), description,
      eval_types: [evalType], expected_output: '', expected_tools: [],
      gates: metricType === 'visual' ? gates : [], input: '',
      judge_config: metricType === 'prompt' ? { model_id: selectedModelId } : null,
      metric_type: METRIC_TYPE_TO_API[metricType], name, retrieval_context: [],
      test_cases: selectedQs.map((q) => ({
        input: q.input?.prompt || '', actual_output: '', expected_output: q.expected?.answer || '',
        context: [], retrieval_context: [], tools_called: [], expected_tools: [],
      })),
      threshold: threshold.toFixed(2), tools_called: [],
    })
      .then(setValidateResult)
      .catch((e) => setValidateError(e.message || 'Validation failed'))
      .finally(() => setValidating(false));
  };

  const handleSave = () => {
    if (!validateResult || !evalType || !metricType) { showToast('Run validation before saving', 'error'); return; }
    setSaving(true); setSaveError('');
    metricsApi.create({
      definition: buildDefinition(), description, eval_types: [evalType],
      metric_type: METRIC_TYPE_TO_API[metricType], name, threshold: threshold.toFixed(2),
      judge_config: metricType === 'prompt' ? { model_id: selectedModelId } : null,
    })
      .then((res) => setSavedId(res.id || 'saved'))
      .catch((e) => setSaveError(e.message || 'Failed to save metric'))
      .finally(() => setSaving(false));
  };

  const resetForm = () => {
    setName(''); setDescription(''); setEvalType(null); setMetricType(null);
    setRules([{ id: ++ruleSeq, field: 'actual_output', operator: 'contains', compareType: 'field', value: 'input' }]); setGates([]);
    setPromptTemplates([]); setSelectedTemplateName(''); setPromptText('');
    setModels([]); setModelHealth({}); setSelectedModelId(''); setCode(''); setThreshold(0.7);
    setDatasets([]); setSelectedDatasetId(''); setPreviewQuestions([]); setSelectedQuestionIds(new Set());
    setValidateResult(null); setValidateError(''); setSavedId('');
    sectionRefs.details.current?.scrollIntoView({ behavior: 'smooth', block: 'start' });
  };

  // =========================================================================
  return (
    <div className={styles.cm}>

      <div className={styles.builder}>

        {/* ============ LEFT RAIL — jump-to links, all sections visible ============ */}
        <aside className={styles.rail}>
          <div className={styles['rail__head']}>
            <div className={styles['rail__eyebrow']}>Overview</div>
            <div className={styles['rail__sub']}>Everything is on this page — jump to any section.</div>
          </div>

          <nav className={styles['rail__steps']}>
            {SECTIONS.map((s, i) => {
              const done = sectionDone[s.key];
              return (
                <button
                  key={s.key}
                  onClick={() => scrollToSection(s.key)}
                  className={`${styles['rail-step']} ${done ? styles['rail-step--done'] : ''}`}
                >
                  <span className={styles['rail-step__marker']}>
                    {done ? <Check size={15} /> : i + 1}
                  </span>
                  <span className={styles['rail-step__body']}>
                    <span className={styles['rail-step__label']}>{s.label}</span>
                    <span className={styles['rail-step__value']}>{sectionValue[s.key]}</span>
                  </span>
                  <ChevronRight size={14} className={styles['rail-step__arrow']} />
                </button>
              );
            })}
          </nav>
        </aside>

        {/* ============ RIGHT WORKSPACE — all sections rendered together ============ */}
        <section className={styles.work}>
          <div className={styles['work__scroll']}>
            <div className={styles['work__inner']}>

              {/* ---- SECTION: DETAILS ---- */}
              <div className={styles.section} ref={sectionRefs.details}>
                <div className={styles['work__eyebrow']}>Section 1</div>
                <h1 className={styles['work__title']}>Name your metric</h1>
                <p className={styles['work__desc']}>Give it a clear name and, optionally, a short description of what it measures.</p>

                <div className={styles.field}>
                  <label className={styles['field__label']}>Metric Name</label>
                  <input className={styles.input} placeholder="e.g., Answer Faithfulness" value={name} onChange={(e) => setName(e.target.value)} />
                </div>
                <div className={styles.field}>
                  <label className={styles['field__label']}>Description</label>
                  <textarea className={styles.textarea} placeholder="What does this metric measure? (optional)" value={description} onChange={(e) => setDescription(e.target.value)} />
                </div>
              </div>

              {/* ---- SECTION: TYPE & TARGET ---- */}
              <div className={styles.section} ref={sectionRefs.type}>
                <div className={styles['work__eyebrow']}>Section 2</div>
                <h1 className={styles['work__title']}>Evaluation type &amp; approach</h1>
                <p className={styles['work__desc']}>Choose what you’re evaluating, then how the metric should score it.</p>

                <div className={styles.field}>
                  <label className={styles['field__label']}>Evaluation Type</label>
                  <div className={`${styles['opt-grid']} ${styles['opt-grid--3']}`}>
                    {EVAL_TYPE_CARDS.map((c) => (
                      <button key={c.key} className={`${styles.opt} ${evalType === c.key ? styles['opt--selected'] : ''}`} onClick={() => handleEvalType(c.key)}>
                        {evalType === c.key && <span className={styles['opt__check']}><Check size={12} /></span>}
                        <span className={styles['opt__icon']}>{c.icon}</span>
                        <div className={styles['opt__title']}>{c.label}</div>
                        <div className={styles['opt__desc']}>{c.desc}</div>
                      </button>
                    ))}
                  </div>
                </div>

                <div className={styles.field}>
                  <label className={styles['field__label']}>Metric Type</label>
                  <div className={styles['opt-grid']}>
                    {METRIC_TYPE_CARDS.map((c) => (
                      <button key={c.key} className={`${styles.opt} ${metricType === c.key ? styles['opt--selected'] : ''}`} onClick={() => handleMetricType(c.key)}>
                        {metricType === c.key && <span className={styles['opt__check']}><Check size={12} /></span>}
                        <span className={styles['opt__icon']}>{c.icon}</span>
                        <div className={styles['opt__title']}>{c.label}</div>
                        <div className={styles['opt__desc']}>{c.desc}</div>
                      </button>
                    ))}
                  </div>
                </div>
              </div>

              {/* ---- SECTION: CONFIG ---- */}
              <div className={styles.section} ref={sectionRefs.config}>
                <div className={styles['work__eyebrow']}>Section 3</div>
                <h1 className={styles['work__title']}>{SECTIONS[2].label}</h1>

                {!metricType && (
                  <div className={styles.empty}>Pick a metric type above to configure it here.</div>
                )}

                {/* visual */}
                {metricType === 'visual' && (
                  <>
                    <p className={styles['work__desc']}>Build one or more field comparisons. Combine them with AND / OR.</p>
                    <div className={styles.rules}>
                      {rules.map((rule, i) => (
                        <div key={rule.id}>
                          {i > 0 && (
                            <div className={styles.gate}>
                              <div className={styles['gate__toggle']}>
                                {(['AND', 'OR'] as const).map((g) => (
                                  <button key={g} className={`${styles['gate__opt']} ${gates[i - 1] === g ? styles.on : ''}`} onClick={() => toggleGate(i - 1)}>{g}</button>
                                ))}
                              </div>
                            </div>
                          )}
                          <div className={styles.rule}>
                            <div className={styles['rule__head']}>
                              <span className={styles['rule__index']}>Rule {i + 1}</span>
                              <button className={styles['btn-icon']} title="Remove" onClick={() => removeRule(rule.id)}><X size={15} /></button>
                            </div>
                            <div className={styles['rule__grid']}>
                              <div className={styles['rule__field']}>
                                <span className={styles['rule__field-label']}>Field</span>
                                <CustomSelect value={rule.field} onChange={(v) => updateRule(rule.id, { field: v })} options={fields.map((f) => ({ value: f, label: f }))} />
                              </div>
                              <div className={styles['rule__field']}>
                                <span className={styles['rule__field-label']}>Operator</span>
                                <CustomSelect value={rule.operator} onChange={(v) => updateRule(rule.id, { operator: v })} options={OPERATORS} />
                              </div>
                              <div className={styles['rule__field']}>
                                <span className={styles['rule__field-label']}>Compare To</span>
                                <CustomSelect value={rule.compareType} onChange={(v) => updateRule(rule.id, { compareType: v as CompareType, value: '' })} options={[{ value: 'field', label: 'Field' }, { value: 'literal', label: 'Literal Value' }]} />
                              </div>
                              <div className={styles['rule__field']}>
                                <span className={styles['rule__field-label']}>Value</span>
                                {rule.compareType === 'literal'
                                  ? <input className={styles.input} placeholder="value" value={rule.value} onChange={(e) => updateRule(rule.id, { value: e.target.value })} />
                                  : <CustomSelect value={rule.value} onChange={(v) => updateRule(rule.id, { value: v })} placeholder="field…" options={fields.map((f) => ({ value: f, label: f }))} />}
                              </div>
                            </div>
                          </div>
                        </div>
                      ))}
                    </div>
                    <button className={`${styles.btn} ${styles['btn--sm']} ${styles['add-rule']}`} onClick={addRule}><Plus size={14} /> Add Rule</button>

                    <div className={styles.summary}>
                      <div className={styles['summary__label']}>Summary</div>
                      <div className={styles['summary__code']}>{ruleSummary || 'No rules defined'}</div>
                    </div>
                  </>
                )}

                {/* prompt */}
                {metricType === 'prompt' && (
                  <>
                    <p className={styles['work__desc']}>Pick a judge prompt template (or write your own), then choose a judge model.</p>

                    {promptTemplatesError && <div className={`${styles.banner} ${styles['banner--err']}`}><AlertCircle size={15} /> {promptTemplatesError}</div>}
                    {promptTemplatesLoading ? (
                      <div className={styles.loading}><Loader2 size={15} className={styles.spin} /> Loading templates…</div>
                    ) : (
                      <div className={styles['tpl-list']}>
                        {matchingTemplates.length === 0 && !allowsCustomPrompt && <div className={styles.empty}>No templates for this evaluation type.</div>}
                        {matchingTemplates.map((t) => (
                          <label key={t.name} className={`${styles.tpl} ${selectedTemplateName === t.name ? styles['tpl--selected'] : ''}`}>
                            <input type="radio" name="tpl" hidden checked={selectedTemplateName === t.name} onChange={() => { setSelectedTemplateName(t.name); setPromptText(t.template); }} />
                            <span className={styles['tpl__radio']} />
                            <span>
                              <span className={styles['tpl__label']}>{t.label}</span>
                              <span className={styles['tpl__desc']}>{t.description}</span>
                              {t.uses_placeholders?.length > 0 && (
                                <span className={styles['tpl__tags']}>
                                  {t.uses_placeholders.map((p) => <span key={p} className={styles.token}>{`{${p}}`}</span>)}
                                </span>
                              )}
                            </span>
                          </label>
                        ))}
                        {allowsCustomPrompt && (
                          <label className={`${styles.tpl} ${selectedTemplateName === '__custom__' ? styles['tpl--selected'] : ''}`}>
                            <input type="radio" name="tpl" hidden checked={selectedTemplateName === '__custom__'} onChange={() => { setSelectedTemplateName('__custom__'); setPromptText(''); }} />
                            <span className={styles['tpl__radio']} />
                            <span>
                              <span className={styles['tpl__label']}>Custom Prompt</span>
                              <span className={styles['tpl__desc']}>Write your own judge prompt from scratch.</span>
                            </span>
                          </label>
                        )}
                      </div>
                    )}

                    {selectedTemplateName && (
                      <div className={styles.field}>
                        <label className={styles['field__label']}>Prompt</label>
                        <textarea className={styles.textarea} style={{ minHeight: '150px' }} value={promptText} onChange={(e) => setPromptText(e.target.value)} placeholder="Enter your judge prompt…" />
                      </div>
                    )}

                    <div className={styles.field}>
                      <label className={styles['field__label']}>Judge Model</label>
                      {modelsError && <div className={`${styles.banner} ${styles['banner--err']}`}><AlertCircle size={15} /> {modelsError}</div>}
                      {modelsLoading ? (
                        <div className={styles.loading}><Loader2 size={15} className={styles.spin} /> Loading models…</div>
                      ) : models.length === 0 ? (
                        <div className={styles.empty}>No models available.</div>
                      ) : (
                        <div className={styles.models}>
                          {models.map((m) => {
                            const health = modelHealth[m.id] || 'checking';
                            const disabled = health === 'unhealthy';
                            return (
                              <label key={m.id} className={`${styles.model} ${selectedModelId === m.id ? styles['model--selected'] : ''} ${disabled ? styles['model--disabled'] : ''}`}>
                                <input type="radio" name="judge" hidden checked={selectedModelId === m.id} disabled={disabled} onChange={() => setSelectedModelId(m.id)} />
                                <span className={styles['model__radio']} />
                                <span>
                                  <span className={styles['model__name']}>{m.name}</span>
                                  <span className={styles['model__meta']}>{m.provider_id}</span>
                                </span>
                                <span className={`${styles['model__health']} ${styles[`health--${health}`]}`}>
                                  <span className={styles['health-dot']} />
                                  {health === 'checking' ? 'Checking' : health === 'healthy' ? 'Healthy' : 'Offline'}
                                </span>
                              </label>
                            );
                          })}
                        </div>
                      )}
                    </div>
                  </>
                )}

                {/* code */}
                {metricType === 'code' && (
                  <>
                    <p className={styles['work__desc']}>Starter code is tailored to the evaluation type. Edit it to suit your metric.</p>
                    {codeError && <div className={`${styles.banner} ${styles['banner--err']}`}><AlertCircle size={15} /> {codeError}</div>}
                    <div className={styles.code}>
                      <div className={styles['code__bar']}>
                        <span className={styles['code__lang']}>Python</span>
                        {codeLoading && <Loader2 size={13} className={styles.spin} />}
                      </div>
                      <textarea className={styles['code__area']} spellCheck={false} value={code} onChange={(e) => setCode(e.target.value)} placeholder="# scoring function" />
                    </div>
                  </>
                )}

                {/* simple */}
                {metricType === 'simple' && (
                  <p className={styles['work__desc']}>The Simple metric needs no extra configuration. Set a threshold below.</p>
                )}

                {/* threshold — shared across all config types */}
                {metricType && (
                  <div className={styles.field} style={{ marginTop: '26px' }}>
                    <label className={styles['field__label']}>Pass Threshold</label>
                    <div className={styles.thr}>
                      <div className={styles['thr__value']}>{threshold.toFixed(2)}</div>
                      <div className={styles['thr__cap']}>Minimum score required to pass</div>
                      <input type="range" className={styles['thr__slider']} min={0} max={1} step={0.01} value={threshold} onChange={(e) => setThreshold(Number(e.target.value))} />
                      <div className={styles['thr__scale']}><span>0.00</span><span>0.50</span><span>1.00</span></div>
                    </div>
                  </div>
                )}
              </div>

              {/* ---- SECTION: DATASET ---- */}
              <div className={`${styles.section} ${styles['section--last']}`} ref={sectionRefs.dataset}>
                <div className={styles['work__eyebrow']}>Section 4</div>
                <h1 className={styles['work__title']}>Choose test data &amp; validate</h1>
                <p className={styles['work__desc']}>Pick a dataset and questions, run validation, then save your metric.</p>

                {!evalType ? (
                  <div className={styles.empty}>Choose an evaluation type above to load datasets.</div>
                ) : (
                  <div className={styles['data-row']}>
                    <div className={styles['data-col']}>
                      <div className={styles['data-col__head']}>
                        <span className={styles['data-col__head-title']}><Database size={12} /> Datasets</span>
                        {datasets.length > 0 && <span className={styles['data-col__count']}>{datasets.length}</span>}
                      </div>
                      <div className={styles['data-col__body']}>
                        {datasetsError ? <div className={`${styles.banner} ${styles['banner--err']}`}><AlertCircle size={15} /> {datasetsError}</div>
                          : datasetsLoading ? <div className={styles.loading}><Loader2 size={15} className={styles.spin} /> Loading…</div>
                          : datasets.length === 0 ? <div className={styles.empty}>No datasets for this type.</div>
                          : (
                            <div className={styles['ds-list']}>
                              {datasets.map((d) => {
                                const selected = selectedDatasetId === d.id;
                                return (
                                  <div
                                    key={d.id}
                                    className={`${styles.ds} ${selected ? styles['ds--selected'] : ''}`}
                                    onClick={() => selectDataset(d.id)}
                                    title={d.name}
                                  >
                                    <span className={styles['ds__check']}><Check size={11} /></span>
                                    <span className={styles['ds__icon']}><Database size={14} /></span>
                                    <span className={styles['ds__name']}>{d.name}</span>
                                    <span className={styles['ds__count']}>{d.question_count} {d.question_count === 1 ? 'question' : 'questions'}</span>
                                  </div>
                                );
                              })}
                            </div>
                          )}
                      </div>
                    </div>

                    <div className={styles['data-col']}>
                      <div className={styles['data-col__head']}>
                        <span className={styles['data-col__head-title']}>
                          <ListChecks size={12} /> Questions
                        </span>
                        {previewQuestions.length > 0 && (
                          <span style={{ display: 'flex', alignItems: 'center', gap: '10px' }}>
                            <span className={styles['data-col__count']}>{selectedQuestionIds.size}/{previewQuestions.length}</span>
                            <span style={{ display: 'flex', gap: '8px' }}>
                              <button className={styles['link-btn']} onClick={selectAllQuestions}>All</button>
                              <button className={styles['link-btn']} onClick={clearAllQuestions}>Clear</button>
                            </span>
                          </span>
                        )}
                      </div>
                      <div className={styles['data-col__body']}>
                        {previewError ? <div className={`${styles.banner} ${styles['banner--err']}`}><AlertCircle size={15} /> {previewError}</div>
                          : previewLoading ? <div className={styles.loading}><Loader2 size={15} className={styles.spin} /> Loading…</div>
                          : previewQuestions.length === 0 ? <div className={styles.empty}>Select a dataset to preview.</div>
                          : (
                            <div className={styles['q-list']}>
                              {previewQuestions.map((q) => {
                                const on = selectedQuestionIds.has(q.id);
                                return (
                                  <div key={q.id} className={`${styles.q} ${on ? styles['q--on'] : ''}`} onClick={() => toggleQuestion(q.id)}>
                                    <span className={styles['q__check']}>{on && <Check size={12} />}</span>
                                    <span className={styles['q__body']}>
                                      <span className={styles['q__q']}>{q.input?.prompt}</span>
                                      <span className={styles['q__a']}><span className={styles['q__a-label']}>Expected:</span>{q.expected?.answer}</span>
                                    </span>
                                  </div>
                                );
                              })}
                            </div>
                          )}
                      </div>
                    </div>
                  </div>
                )}

                {/* ---- validate & save ---- */}
                <div className={styles['validate-section']}>
                  <div className={styles['validate-section__label']}>Validate &amp; Save</div>
                  <p className={styles['validate-section__desc']}>Run a dry-run against your selected questions. Saving unlocks once it passes.</p>

                  {validateError && <div className={`${styles.banner} ${styles['banner--err']}`}><AlertCircle size={15} /> {validateError}</div>}

                  {!validateResult && !validating && (
                    <div className={`${styles.banner} ${styles['banner--info']}`}><Sparkles size={15} /> Ready to validate {selectedQuestionIds.size} test case{selectedQuestionIds.size === 1 ? '' : 's'}.</div>
                  )}

                  {validateResult && (
                    <div style={{ marginBottom: '18px' }}>
                      {validateSucceeded
                        ? <div className={`${styles.banner} ${styles['banner--ok']}`}><CheckCircle2 size={15} /> Metric is valid — ready to save.</div>
                        : <div className={`${styles.banner} ${styles['banner--err']}`}><XCircle size={15} /> No test cases passed. You can still save, or adjust your metric and re-run.</div>}

                      <div className={styles.results}>
                        {validateResult.results.map((r, i) => (
                          <div key={i} className={styles['results__row']}>
                            <span className={`${styles['results__score']} ${r.success ? styles['results__score--pass'] : styles['results__score--fail']}`}>{r.score.toFixed(2)}</span>
                            <span className={styles['results__body']}>
                              <span className={styles['results__io']}>{r.test_case.input}</span>
                              {r.reason && <span className={styles['results__reason']}>{r.reason}</span>}
                            </span>
                            <span className={`${styles['results__pill']} ${r.success ? styles['results__pill--pass'] : styles['results__pill--fail']}`}>{r.success ? 'Pass' : 'Fail'}</span>
                          </div>
                        ))}
                        <div className={styles['results__summary']}>
                          <span>Passed: <strong>{validateResult.passed}/{validateResult.total}</strong></span>
                        </div>
                      </div>
                    </div>
                  )}
                </div>
              </div>

            </div>
          </div>

          {/* ---- sticky footer ---- */}
          <div className={styles['work__foot']}>
            <span className={styles['work__foot-info']}>
              {completedCount}/{SECTIONS.length} sections ready
            </span>

            <div className={styles['work__foot-actions']}>
              {!validateResult ? (
                <>
                  <button className={`${styles.btn} ${styles['btn--primary']}`} onClick={runValidate} disabled={validating || !canValidate}>
                    {validating ? <Loader2 size={15} className={styles.spin} /> : <Sparkles size={15} />}
                    {validating ? 'Validating…' : 'Run Validation'}
                    {!validating && <ArrowRight size={15} />}
                  </button>
                  <button className={`${styles.btn} ${styles['btn--ghost']}`} onClick={onCancel}>Cancel</button>
                </>
              ) : (
                <>
                  <button className={`${styles.btn} ${styles['btn--ok']}`} onClick={handleSave} disabled={saving}>
                    {saving ? <Loader2 size={15} className={styles.spin} /> : <Check size={15} />}
                    Save Metric
                  </button>
                  <button className={`${styles.btn} ${styles['btn--ghost']}`} onClick={onCancel}>Cancel</button>
                </>
              )}
            </div>
          </div>
        </section>
      </div>

      {saveError && <div className={styles.toast}><AlertCircle size={15} /> {saveError}</div>}

      {savedId && (
        <div className={styles.overlay}>
          <div className={styles.modal}>
            <div className={styles['modal__icon']}><CheckCircle2 size={26} /></div>
            <div className={styles['modal__title']}>Metric created!</div>
            <div className={styles['modal__text']}>Your metric is now available for evaluations.</div>
            <div className={styles['modal__id']}>ID: {savedId}</div>
            <div className={styles['modal__actions']}>
              <button className={styles.btn} onClick={resetForm}>Create Another</button>
              <button className={`${styles.btn} ${styles['btn--primary']}`} onClick={() => onSaved(savedId)}>Go to Dashboard</button>
            </div>
          </div>
        </div>
      )}

      {ToastEl}
    </div>
  );
}
