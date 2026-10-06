import { useEffect, useMemo, useState } from 'react';
import { Search, Boxes, ChevronUp, ChevronDown, ChevronsUpDown, ChevronLeft, ChevronRight, ChevronsLeft, ChevronsRight, ListFilter } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchModels } from '../../store/slices/modelsSlice';
import { fetchProviders } from '../../store/slices/providersSlice';
import { SkeletonTableRows } from '../common/Skeleton';
import type { Model } from '../../types';
import styles from './ModelCatalog.module.scss';

type SortKey = 'name' | 'provider' | 'context_window' | 'price' | 'accuracy' | 'status';
type SortDir = 'asc' | 'desc';

const PAGE_SIZE_OPTIONS = [10, 25, 50, 100];
const ACCURACY_HIGH_THRESHOLD = 90;

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
    <th className={styles['model-catalog__sortable-th']}>
      <button
        type="button"
        className={`${styles['model-catalog__sort-btn']} ${active ? styles['model-catalog__sort-btn--active'] : ''}`}
        onClick={() => onSort(sortKey)}
      >
        {label}
        {active ? (
          dir === 'asc' ? <ChevronUp size={13} /> : <ChevronDown size={13} />
        ) : (
          <ChevronsUpDown size={13} className={styles['model-catalog__sort-icon-idle']} />
        )}
      </button>
    </th>
  );
}

export default function ModelCatalog() {
  const dispatch = useAppDispatch();
  const { items, status } = useAppSelector((s) => s.models);
  const providers = useAppSelector((s) => s.providers.items);
  const [search, setSearch] = useState('');
  const [capFilter, setCapFilter] = useState('All');
  const [sortKey, setSortKey] = useState<SortKey>('name');
  const [sortDir, setSortDir] = useState<SortDir>('asc');
  const [page, setPage] = useState(1);
  const [pageSize, setPageSize] = useState(10);

  useEffect(() => {
    dispatch(fetchModels());
    dispatch(fetchProviders());
  }, [dispatch]);

  const caps = useMemo(() => ['All', ...new Set(items.flatMap((m) => m.capabilities))], [items]);
  const providerName = (id: string) => providers.find((p) => p.id === id)?.name || id;

  const filtered = useMemo(() => {
    return items.filter((m) => {
      if (capFilter !== 'All' && !m.capabilities.includes(capFilter)) return false;
      const q = search.toLowerCase();
      return !q || m.name.toLowerCase().includes(q) || providerName(m.provider_id).toLowerCase().includes(q);
    });
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [items, providers, search, capFilter]);

  const sorted = useMemo(() => {
    const dir = sortDir === 'asc' ? 1 : -1;
    const compare = (a: Model, b: Model): number => {
      switch (sortKey) {
        case 'name':
          return a.name.localeCompare(b.name) * dir;
        case 'provider':
          return providerName(a.provider_id).localeCompare(providerName(b.provider_id)) * dir;
        case 'context_window':
          return (a.context_window - b.context_window) * dir;
        case 'price':
          return ((a.input_price ?? -1) - (b.input_price ?? -1)) * dir;
        case 'accuracy':
          return ((a.accuracy_score ?? -1) - (b.accuracy_score ?? -1)) * dir;
        case 'status':
          return (Number(a.is_active) - Number(b.is_active)) * dir;
        default:
          return 0;
      }
    };
    return [...filtered].sort(compare);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [filtered, sortKey, sortDir, providers]);

  const total = sorted.length;
  const totalPages = Math.max(1, Math.ceil(total / pageSize));
  const safePage = Math.min(page, totalPages);
  const startIdx = (safePage - 1) * pageSize;
  const pageItems = sorted.slice(startIdx, startIdx + pageSize);
  const pageList = useMemo(() => buildPageList(safePage, totalPages), [safePage, totalPages]);

  useEffect(() => {
    setPage(1);
  }, [search, capFilter, pageSize]);

  const toggleSort = (key: SortKey) => {
    if (sortKey === key) {
      setSortDir((d) => (d === 'asc' ? 'desc' : 'asc'));
    } else {
      setSortKey(key);
      setSortDir('asc');
    }
  };

  return (
    <div className="page-enter pg-shell">
      <div className={styles['model-catalog__header']}>
        <div>
          <p className={styles['model-catalog__header-eyebrow']}>Catalog</p>
          <h1>Model Catalog</h1>
          <p className={styles['model-catalog__header-sub']}>All models across connected providers</p>
        </div>
        <div className={styles['model-catalog__header-meta']}>
          <Boxes size={13} />
          {items.length} model{items.length === 1 ? '' : 's'} listed
        </div>
      </div>

      <div className={styles['model-catalog__toolbar']}>
        <div className={styles['model-catalog__search']}>
          <Search size={16} />
          <input placeholder="Search models or providers…" value={search} onChange={(e) => setSearch(e.target.value)} />
        </div>

        <div className={styles['model-catalog__filter-group']}>
          <span className={styles['model-catalog__toolbar-label']}>
            <ListFilter size={11} /> Capability
          </span>
          {caps.map((c) => (
            <button
              key={c}
              className={`${styles['model-catalog__filter-pill']} ${capFilter === c ? styles['model-catalog__filter-pill--on'] : ''}`}
              onClick={() => setCapFilter(c)}
            >
              {c}
            </button>
          ))}
        </div>
      </div>

      <div className="pg-body">
        <div className={styles['model-catalog__table-wrap']}>
          <table className={styles['model-catalog__table']}>
            <thead>
              <tr>
                <SortableTh label="Model" sortKey="name" activeKey={sortKey} dir={sortDir} onSort={toggleSort} />
                <SortableTh label="Provider" sortKey="provider" activeKey={sortKey} dir={sortDir} onSort={toggleSort} />
                <th>Capabilities</th>
                <SortableTh label="Context" sortKey="context_window" activeKey={sortKey} dir={sortDir} onSort={toggleSort} />
                <SortableTh label="Price (in/out)" sortKey="price" activeKey={sortKey} dir={sortDir} onSort={toggleSort} />
                <SortableTh label="Accuracy" sortKey="accuracy" activeKey={sortKey} dir={sortDir} onSort={toggleSort} />
                <SortableTh label="Status" sortKey="status" activeKey={sortKey} dir={sortDir} onSort={toggleSort} />
              </tr>
            </thead>
            <tbody>
              {status === 'loading' && <SkeletonTableRows columns={7} rows={6} />}
              {status !== 'loading' &&
                pageItems.map((m) => (
                  <tr key={m.id}>
                    <td className={styles['model-catalog__name-cell']}>{m.name}</td>
                    <td className={styles['model-catalog__provider-cell']}>{providerName(m.provider_id)}</td>
                    <td>
                      <div className={styles['model-catalog__caps-cell']}>
                        {m.capabilities.map((c) => (
                          <span key={c} className={styles['model-catalog__tag']}>
                            {c}
                          </span>
                        ))}
                      </div>
                    </td>
                    <td className={styles['model-catalog__mono-cell']}>{m.context_window.toLocaleString()}</td>
                    <td className={`${styles['model-catalog__mono-cell']} ${styles['model-catalog__mono-cell--muted']}`}>
                      {m.input_price != null ? `$${m.input_price.toFixed(2)}` : '—'} / {m.output_price != null ? `$${m.output_price.toFixed(2)}` : '—'}
                    </td>
                    <td>
                      <span
                        className={`${styles['model-catalog__accuracy']} ${
                          (m.accuracy_score || 0) >= ACCURACY_HIGH_THRESHOLD ? styles['model-catalog__accuracy--high'] : ''
                        }`}
                      >
                        {m.accuracy_score != null ? `${m.accuracy_score}%` : '—'}
                      </span>
                    </td>
                    <td>
                      <span className={`${styles['model-catalog__status']} ${styles[`model-catalog__status--${m.is_active ? 'active' : 'inactive'}`]}`}>
                        {m.is_active ? 'Active' : 'Inactive'}
                      </span>
                    </td>
                  </tr>
                ))}
              {status !== 'loading' && pageItems.length === 0 && (
                <tr>
                  <td colSpan={7} className={styles['model-catalog__empty']}>
                    No models match your filters.
                  </td>
                </tr>
              )}
            </tbody>
          </table>

          {status !== 'loading' && total > 0 && (
            <div className={styles['model-catalog__pagination']}>
              <div className={styles['model-catalog__pagination-info']}>
                <span>
                  Showing <strong>{startIdx + 1}–{Math.min(startIdx + pageSize, total)}</strong> of <strong>{total}</strong> model
                  {total === 1 ? '' : 's'}
                </span>
                <div className={styles['model-catalog__page-size']}>
                  <label htmlFor="model-catalog-page-size">Rows per page</label>
                  <select id="model-catalog-page-size" value={pageSize} onChange={(e) => setPageSize(Number(e.target.value))}>
                    {PAGE_SIZE_OPTIONS.map((n) => (
                      <option key={n} value={n}>
                        {n}
                      </option>
                    ))}
                  </select>
                </div>
              </div>

              <div className={styles['model-catalog__pager']}>
                <button
                  className={styles['model-catalog__page-btn']}
                  disabled={safePage === 1}
                  onClick={() => setPage(1)}
                  aria-label="First page"
                >
                  <ChevronsLeft size={14} />
                </button>
                <button
                  className={styles['model-catalog__page-btn']}
                  disabled={safePage === 1}
                  onClick={() => setPage((p) => Math.max(1, p - 1))}
                  aria-label="Previous page"
                >
                  <ChevronLeft size={14} />
                </button>

                {pageList.map((p, i) =>
                  p === '…' ? (
                    <span key={`dots-${i}`} className={styles['model-catalog__page-dots']}>
                      …
                    </span>
                  ) : (
                    <button
                      key={p}
                      className={`${styles['model-catalog__page-btn']} ${styles['model-catalog__page-btn--num']} ${
                        p === safePage ? styles['model-catalog__page-btn--active'] : ''
                      }`}
                      onClick={() => setPage(p)}
                      aria-current={p === safePage ? 'page' : undefined}
                    >
                      {p}
                    </button>
                  )
                )}

                <button
                  className={styles['model-catalog__page-btn']}
                  disabled={safePage === totalPages}
                  onClick={() => setPage((p) => Math.min(totalPages, p + 1))}
                  aria-label="Next page"
                >
                  <ChevronRight size={14} />
                </button>
                <button
                  className={styles['model-catalog__page-btn']}
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
    </div>
  );
}
