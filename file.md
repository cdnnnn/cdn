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
