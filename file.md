//Custommtricsdashboard.tsx
import { useEffect, useMemo, useState, useCallback } from 'react';
import {
  Search, Gauge, LayoutDashboard, PenSquare, ListFilter, AlertCircle, Loader2, Trash2, X,
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

// Maps an eval type to its badge color variant; anything outside the
// known set (model/agent/rag) falls back to a neutral badge.
function evalTypeVariant(t: string): string {
  const known = ['model', 'agent', 'rag'];
  return known.includes((t ?? '').toLowerCase()) ? (t ?? '').toLowerCase() : 'default';
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
    <div className="page-enter pg-shell">
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
                    return (
                      <tr key={m.id} title={m.description}>
                        <td style={{ fontWeight: 700 }}>{m.name ?? '—'}</td>
                        <td>
                          <div className={styles['type-badge-group']}>
                            {(m.eval_types ?? []).map((t) => (
                              <span key={t} className={`${styles['type-badge']} ${styles[`type-badge--${evalTypeVariant(t)}`]}`}>
                                {(t ?? '').toUpperCase()}
                              </span>
                            ))}
                          </div>
                        </td>
                        <td><span className={`${styles['type-badge']} ${styles['type-badge--metric']}`}>{m.metric_type}</span></td>
                        <td style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", fontSize: 13, color: 'var(--text-secondary)' }}>{m.threshold}</td>
                        <td style={{ color: 'var(--text-secondary)' }}>{m.requires_judge ? 'Yes' : 'No'}</td>
                        <td><span className={`badge ${m.is_active ? 'badge-green' : 'badge-gray'}`}>{m.is_active ? 'Active' : 'Inactive'}</span></td>
                        <td style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", fontSize: 13, color: 'var(--text-secondary)' }}>{formatDate(m.created_at)}</td>
                        <td>
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

      {pendingDeleteId && (() => {
        const target = metrics.find((m) => m.id === pendingDeleteId);
        const isDeleting = deletingId === pendingDeleteId;
        return (
          // Backdrop is intentionally non-interactive — no onClick here —
          // so clicking outside the modal does not dismiss it. Only the
          // close icon and Cancel button call cancelDelete().
          <div className={styles['confirm-overlay']}>
            <div className={styles['confirm-modal']} role="dialog" aria-modal="true" aria-label="Confirm delete metric">
              <div className={styles['confirm-modal__top']}>
                <div className={styles['confirm-modal__icon']}>
                  <AlertCircle size={20} />
                </div>
                <button
                  type="button"
                  className={styles['confirm-modal__close']}
                  aria-label="Close"
                  disabled={isDeleting}
                  onClick={cancelDelete}
                >
                  <X size={16} />
                </button>
              </div>

              <div className={styles['confirm-modal__title']}>Delete this metric?</div>
              <div className={styles['confirm-modal__text']}>
                {target ? <>This will permanently delete <strong>{target.name}</strong>. This action can&apos;t be undone.</> : "This action can't be undone."}
              </div>

              <div className={styles['confirm-modal__actions']}>
                <button
                  type="button"
                  className={`${styles['confirm-modal__btn']} ${styles['confirm-modal__btn--cancel']}`}
                  disabled={isDeleting}
                  onClick={cancelDelete}
                >
                  Cancel
                </button>
                <button
                  type="button"
                  className={`${styles['confirm-modal__btn']} ${styles['confirm-modal__btn--danger']}`}
                  disabled={isDeleting}
                  onClick={() => confirmDelete(pendingDeleteId)}
                >
                  {isDeleting ? <Loader2 size={14} className={styles.spin} /> : <Trash2 size={14} />}
                  Delete
                </button>
              </div>
            </div>
          </div>
        );
      })()}
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
@keyframes cm-modal-in { from { opacity: 0; transform: translateY(12px) scale(0.98); } to { opacity: 1; transform: translateY(0) scale(1); } }
@keyframes cm-overlay-in { from { opacity: 0; } to { opacity: 1; } }

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
// eval-type / metric-type badges — replaces the generic global "tag" pill
// with color-coded badges so eval types (model/agent/rag) are scannable
// at a glance. Colors reuse the same theme tokens as the rest of the app
// (no new hex values); each eval type gets a distinct existing wash.
// ---------------------------------------------------------------------------
.type-badge {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-family: $mono;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  padding: 3px 9px;
  border-radius: 6px;
  border: 1px solid transparent;
  white-space: nowrap;

  &::before {
    content: '';
    width: 5px;
    height: 5px;
    border-radius: 50%;
    flex-shrink: 0;
  }

  // eval-type variants
  &--model {
    color: $signal;
    background: $wash;
    border-color: rgba($signal, 0.16);
    &::before { background: $signal; }
  }
  &--agent {
    color: $sky-ink;
    background: $sky-ink-wash;
    border-color: rgba($sky-ink, 0.18);
    &::before { background: $sky-ink; }
  }
  &--rag {
    color: $rose-ink;
    background: $rose-ink-wash;
    border-color: rgba($rose-ink, 0.18);
    &::before { background: $rose-ink; }
  }
  // fallback for any eval type outside the known set
  &--default {
    color: $ink-2;
    background: $ink-wash;
    border-color: $line;
    &::before { background: $ink-3; }
  }

  // metric-type column reuses the same shape but stays neutral/monochrome
  // so it reads as a distinct dimension from the colored eval-type badges
  &--metric {
    color: $ink-2;
    background: $paper;
    border-color: $line;
    &::before { display: none; }
  }
}

.type-badge-group {
  display: flex;
  flex-wrap: wrap;
  gap: 5px;
}

// ---------------------------------------------------------------------------
// delete confirmation modal — click-outside is intentionally inert; only
// the close icon or Cancel button dismiss it.
// ---------------------------------------------------------------------------
.confirm-overlay {
  position: fixed;
  inset: 0;
  z-index: 300;
  display: flex;
  align-items: center;
  justify-content: center;
  background: rgba(10, 12, 18, 0.55);
  padding: 20px;
  animation: cm-overlay-in 0.15s ease;
}

.confirm-modal {
  width: 100%;
  max-width: 380px;
  background: $card;
  border-radius: 18px;
  box-shadow: $lift;
  padding: 24px 24px 20px;
  animation: cm-modal-in 0.2s ease;
}

.confirm-modal__top {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 12px;
  margin-bottom: 14px;
}

.confirm-modal__icon {
  flex-shrink: 0;
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background: $danger-wash;
  color: $danger;
  display: flex;
  align-items: center;
  justify-content: center;
}

.confirm-modal__close {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  border-radius: 8px;
  border: none;
  background: transparent;
  color: $ink-3;
  cursor: pointer;
  transition: background 0.14s ease, color 0.14s ease;

  &:hover { background: $paper; color: $ink; }
}

.confirm-modal__title {
  font-family: $display;
  font-size: 1.2308em; // 1rem / 0.8125rem
  font-weight: 800;
  color: $ink;
  margin-bottom: 6px;
}

.confirm-modal__text {
  font-size: 0.9615em; // 0.78125rem / 0.8125rem
  color: $ink-2;
  line-height: 1.5;
  margin-bottom: 20px;

  strong { color: $ink; font-weight: 700; }
}

.confirm-modal__actions {
  display: flex;
  gap: 10px;
}

.confirm-modal__btn {
  flex: 1;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  padding: 10px 14px;
  border-radius: 10px;
  font-size: 0.9615em; // 0.78125rem / 0.8125rem
  font-weight: 650;
  cursor: pointer;
  transition: all 0.15s ease;

  &--cancel {
    border: 1.5px solid $line;
    background: $card;
    color: $ink-2;

    &:hover:not(:disabled) { border-color: $ink-3; color: $ink; }
  }

  &--danger {
    border: 1.5px solid $danger;
    background: $danger;
    color: #fff;

    &:hover:not(:disabled) { filter: brightness(0.92); }
    &:disabled { opacity: 0.6; cursor: not-allowed; }
  }
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
