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























@use '../../styles/_variables' as *;

// ===========================================================================
// Model Catalog — matches the Run Console / Dashboard / Providers design
// system: ink/paper palette, ultramarine signal accent, mono instrument
// labels, hover-lift cards, mono numerals for data-dense cells.
// ===========================================================================

$ink:      #14161B;
$ink-2:    #565B66;
$ink-3:    #8A909B;
$paper:    #F5F6F8;
$card:     #FFFFFF;
$line:     #E6E8EC;
$line-2:   #EEF0F3;
$signal:   #2B2BF5;
$signal-2: #1C1CC7;
$wash:     #ECEDFF;
$ok:       #0FA968;
$ok-wash:  #E7F7EF;
$ink-wash: #EEF0F2;

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

$soft: 0 1px 2px rgba(20, 22, 27, 0.05);
$lift: 0 14px 30px -14px rgba(20, 22, 27, 0.22);

%micro {
  font-family: $mono;
  font-size: 0.6875rem;
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

.model-catalog {
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

    h1 {
      font-family: $display;
      font-size: 1.5rem;
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
    font-size: 0.84375rem;
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
    font-size: 0.71875rem;
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ink-2;
    white-space: nowrap;
    margin-bottom: 3px;
  }

  // ---- toolbar --------------------------------------------------------------
  &__toolbar {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 14px;
    padding: 14px 32px;
    background: $card;
    border-bottom: 1px solid $line;
    flex-wrap: wrap;
  }

  &__search {
    position: relative;
    flex: 1;
    max-width: 340px;
    min-width: 200px;

    svg {
      position: absolute;
      top: 50%;
      left: 13px;
      transform: translateY(-50%);
      color: $ink-3;
      pointer-events: none;
    }

    input {
      width: 100%;
      border: 1.5px solid $line;
      border-radius: 10px;
      padding: 9px 12px 9px 38px;
      font-size: 0.84375rem;
      font-family: $sans;
      color: $ink;
      background: $paper;
      transition: border-color 0.15s ease, background 0.15s ease;

      &::placeholder { color: $ink-3; }
      &:focus {
        outline: none;
        border-color: $signal;
        background: $card;
      }
    }
  }

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
    font-size: 0.625rem;
    color: $ink-3;
    white-space: nowrap;
  }

  &__filter-pill {
    padding: 6px 13px;
    border: 0;
    border-radius: 999px;
    background: transparent;
    color: $ink-2;
    font-size: 0.78125rem;
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

  &__loading {
    display: flex;
    align-items: center;
    gap: 8px;
    color: $ink-2;
    font-size: 0.8125rem;
    margin-bottom: 16px;
  }

  // ---- table shell ----------------------------------------------------------
  &__table-wrap {
    border: 1px solid $line;
    border-radius: 16px;
    background: $card;
    overflow: hidden;
  }

  &__table {
    width: 100%;
    border-collapse: collapse;
    font-size: 0.84375rem;

    thead th {
      text-align: left;
      background: $paper;
      border-bottom: 1px solid $line;
      @extend %micro;
      font-size: 0.625rem;
      color: $ink-3;
      padding: 12px 16px;
      white-space: nowrap;
    }

    tbody tr {
      border-bottom: 1px solid $line-2;
      transition: background 0.13s ease;

      &:last-child { border-bottom: 0; }
      &:hover { background: $paper; }
    }

    tbody td {
      padding: 13px 16px;
      color: $ink;
      vertical-align: middle;
    }
  }

  &__name-cell {
    font-family: $display;
    font-weight: 700;
    color: $ink;
  }

  &__provider-cell {
    color: $ink-2;
  }

  &__caps-cell {
    display: flex;
    flex-wrap: wrap;
    gap: 5px;
  }

  &__tag {
    font-family: $mono;
    font-size: 0.65625rem;
    font-weight: 600;
    letter-spacing: 0.02em;
    color: $ink-2;
    background: $paper;
    border: 1px solid $line;
    border-radius: 6px;
    padding: 2px 7px;
    white-space: nowrap;
  }

  &__mono-cell {
    font-family: $mono;
    font-size: 0.8125rem;
    color: $ink;
  }

  &__mono-cell--muted {
    color: $ink-2;
  }

  &__accuracy {
    font-family: $mono;
    font-weight: 700;
    font-size: 0.8125rem;
    color: $ink;

    &--high { color: $ok; }
  }

  &__status {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 3px 10px 3px 8px;
    border-radius: 999px;
    font-family: $mono;
    font-size: 0.625rem;
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;

    &::before { content: ''; width: 5px; height: 5px; border-radius: 50%; }

    &--active {
      color: $ok;
      background: $ok-wash;
      &::before { background: $ok; }
    }

    &--inactive {
      color: $ink-3;
      background: $ink-wash;
      &::before { background: $ink-3; }
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
    font-size: 0.625rem;
    letter-spacing: 0.14em;
    text-transform: uppercase;
    color: $ink-3;
    text-align: left;
    transition: color 0.15s ease;

    &:hover {
      color: $ink;

      .model-catalog__sort-icon-idle { opacity: 0.7; }
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
    font-size: 0.84375rem;
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
    font-size: 0.78125rem;
    color: $ink-2;

    strong { color: $ink; font-weight: 700; }
  }

  &__page-size {
    display: flex;
    align-items: center;
    gap: 8px;

    label {
      font-size: 0.71875rem;
      color: $ink-3;
      white-space: nowrap;
    }

    select {
      appearance: none;
      -webkit-appearance: none;
      font: inherit;
      font-size: 0.78125rem;
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
    font-size: 0.78125rem;
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
    font-size: 0.78125rem;
  }
}

@media (max-width: 768px) {
  .model-catalog__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .model-catalog__toolbar { padding: 12px 18px; }
}
