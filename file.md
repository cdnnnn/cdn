//Datasets.tsx
import { useEffect, useMemo, useState, useRef, useCallback } from 'react';
import {
  RefreshCw, Search, Layers, AlertTriangle, Database, ListFilter, X,
  Check, Boxes, ArrowRight, Filter, ChevronsUpDown,
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchDatasets } from '../../store/slices/datasetsSlice';
import type { Dataset } from '../../store/slices/datasetsSlice';
import styles from './Datasets.module.scss';

// Deterministic color hash so the same category always gets the same pill
// color across renders, without a hardcoded lookup table. Palette matches
// the app's ink/paper/signal design tokens.
const PILL_COLORS = [
  { bg: '#ECEDFF', fg: '#2B2BF5' }, // signal
  { bg: '#FDF3E3', fg: '#C56A00' }, // amber
  { bg: '#E7F7EF', fg: '#0B8F58' }, // ok
  { bg: '#FDECEC', fg: '#C81E1E' }, // danger
  { bg: '#E6F4FB', fg: '#0369A1' }, // sky
  { bg: '#F1EDFB', fg: '#6D28D9' }, // violet
  { bg: '#EAF6EC', fg: '#3F7D20' }, // moss
];
function hashColor(label: string) {
  const sum = [...label].reduce((acc, ch) => acc + ch.charCodeAt(0), 0);
  return PILL_COLORS[sum % PILL_COLORS.length];
}

export default function Datasets() {
  const dispatch = useAppDispatch();
  const { items, status, error } = useAppSelector((s) => s.datasets);

  const [search, setSearch] = useState('');
  const [datasetTypeFilter, setDatasetTypeFilter] = useState('All');
  const [tagFilter, setTagFilter] = useState<string[]>([]);       // active dataset_categories facets
  const [selectedId, setSelectedId] = useState<string | null>(null);
  const searchRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    dispatch(fetchDatasets());
  }, [dispatch]);

  const datasetTypes = useMemo(() => ['All', ...new Set(items.map((d) => d.dataset_type))], [items]);

  const filtered = useMemo(() => {
    const q = search.trim().toLowerCase();
    return items.filter((d) => {
      if (datasetTypeFilter !== 'All' && d.dataset_type !== datasetTypeFilter) return false;
      const tags = d.dataset_categories ?? [];
      if (tagFilter.length && !tagFilter.some((t) => tags.includes(t))) return false;
      if (!q) return true;
      return d.name.toLowerCase().includes(q) || d.description.toLowerCase().includes(q);
    });
  }, [items, search, datasetTypeFilter, tagFilter]);

  // Keep a valid selection as the filtered set changes.
  useEffect(() => {
    if (!filtered.length) return;
    if (!filtered.some((d) => d.id === selectedId)) setSelectedId(filtered[0].id);
  }, [filtered, selectedId]);

  const selected = items.find((d) => d.id === selectedId) ?? null;

  const toggleTag = useCallback((t: string) => {
    setTagFilter((prev) => (prev.includes(t) ? prev.filter((x) => x !== t) : [...prev, t]));
  }, []);

  // Keyboard: ↑/↓ walk the list, "/" focuses search.
  useEffect(() => {
    const onKey = (e: KeyboardEvent) => {
      const el = document.activeElement as HTMLElement | null;
      const typing = el?.tagName === 'INPUT' || el?.tagName === 'TEXTAREA';
      if (e.key === '/' && !typing) {
        e.preventDefault();
        searchRef.current?.focus();
        return;
      }
      if (typing || (e.key !== 'ArrowDown' && e.key !== 'ArrowUp')) return;
      e.preventDefault();
      const idx = filtered.findIndex((d) => d.id === selectedId);
      if (idx === -1) {
        if (filtered[0]) setSelectedId(filtered[0].id);
        return;
      }
      const next = e.key === 'ArrowDown'
        ? Math.min(idx + 1, filtered.length - 1)
        : Math.max(idx - 1, 0);
      setSelectedId(filtered[next].id);
    };
    window.addEventListener('keydown', onKey);
    return () => window.removeEventListener('keydown', onKey);
  }, [filtered, selectedId]);

  return (
    <div className="page-enter pg-shell">
      {/* ---- header (unchanged) ------------------------------------------- */}
      <div className={styles['datasets__header']}>
        <div>
          <p className={styles['datasets__header-eyebrow']}>Datasets</p>
          <h1>Test Suite Library</h1>
          <p className={styles['datasets__header-sub']}>
            Browse every dataset available for evaluations, independent of any single wizard run.
          </p>
        </div>
        <div className={styles['datasets__header-meta']}>
          <span className={styles['datasets__header-count']}>
            <Database size={13} /> {items.length} datasets available
          </span>
          <button className={styles['datasets__refresh-btn']} onClick={() => dispatch(fetchDatasets())}>
            <RefreshCw size={14} /> Refresh
          </button>
        </div>
      </div>

      {/* ---- toolbar (unchanged) ------------------------------------------ */}
      <div className={styles['datasets__toolbar']}>
        <div className={styles['datasets__search']}>
          <Search size={16} />
          <input
            ref={searchRef}
            placeholder="Search datasets…  (press /)"
            value={search}
            onChange={(e) => setSearch(e.target.value)}
          />
        </div>
        <div className={styles['datasets__filters']}>
          <span className={styles['datasets__toolbar-label']}>
            <ListFilter size={11} />
          </span>
          {datasetTypes.map((t) => (
            <button
              key={t}
              className={`${styles['datasets__filter-pill']} ${datasetTypeFilter === t ? styles['datasets__filter-pill--on'] : ''}`}
              onClick={() => setDatasetTypeFilter(t)}
            >
              {t}
            </button>
          ))}
        </div>
      </div>

      {/* ---- active tag facets --------------------------------------------- */}
      {tagFilter.length > 0 && (
        <div className={styles['datasets__facets']}>
          <Filter size={12} />
          <span className={styles['datasets__facets-lead']}>Showing datasets tagged</span>
          {tagFilter.map((t) => {
            const col = hashColor(t);
            return (
              <button
                key={t}
                className={styles['datasets__facet']}
                style={{ background: col.bg, color: col.fg }}
                onClick={() => toggleTag(t)}
              >
                {t} <X size={11} />
              </button>
            );
          })}
          <button className={styles['datasets__facets-clear']} onClick={() => setTagFilter([])}>
            Clear all
          </button>
        </div>
      )}

      {/* ---- body: master / detail ---------------------------------------- */}
      <div className={styles['datasets__body']}>
        {status === 'failed' && (
          <div
            className={`${styles['datasets__state']} ${styles['datasets__state--error']}`}
            style={{ gridColumn: '1 / -1' }}
          >
            <AlertTriangle size={28} />
            <div>{error || 'Failed to load datasets.'}</div>
            <button className={styles['datasets__refresh-btn']} onClick={() => dispatch(fetchDatasets())}>
              Retry
            </button>
          </div>
        )}

        {status !== 'failed' && (
          <>
            {/* LIST RAIL — bordered cards, one per dataset */}
            <aside className={styles['datasets__rail']}>
              <div className={styles['datasets__rail-head']}>
                <span>{filtered.length} of {items.length}</span>
                <span className={styles['datasets__rail-hint']}>
                  <ChevronsUpDown size={11} /> ↑ ↓ to move
                </span>
              </div>
              <div className={styles['datasets__rail-scroll']}>
                {status === 'loading' &&
                  Array.from({ length: 7 }).map((_, i) => (
                    <div className={styles['datasets__skel-row']} key={i}>
                      <span className={styles['datasets__skel']} style={{ width: '55%' }} />
                      <span className={styles['datasets__skel']} style={{ width: '35%' }} />
                    </div>
                  ))}

                {status !== 'loading' && filtered.length === 0 && (
                  <div className={styles['datasets__empty-rail']}>
                    <Layers size={22} />
                    <p>No datasets match.<br />Loosen a filter to see more.</p>
                  </div>
                )}

                {status !== 'loading' &&
                  filtered.map((d) => {
                    const on = d.id === selectedId;
                    const accent = hashColor(d.category);
                    const tags = d.dataset_categories ?? [];
                    return (
                      <button
                        key={d.id}
                        className={`${styles['datasets__row']} ${on ? styles['datasets__row--on'] : ''}`}
                        onClick={() => setSelectedId(d.id)}
                        style={{ borderLeftColor: accent.fg }}
                      >
                        <div className={styles['datasets__row-top']}>
                          <span className={styles['datasets__row-name']}>{d.name}</span>
                          <span className={styles['datasets__row-count']}>{d.question_count.toLocaleString()}</span>
                        </div>
                        <div className={styles['datasets__row-foot']}>
                          <span className={styles['datasets__row-type']} style={{ color: accent.fg }}>
                            {d.category}
                          </span>
                          <span className={styles['datasets__row-dots']}>
                            {tags.slice(0, 4).map((t) => (
                              <i key={t} style={{ background: hashColor(t).fg }} title={t} />
                            ))}
                          </span>
                        </div>
                      </button>
                    );
                  })}
              </div>
            </aside>

            {/* DETAIL */}
            <section className={styles['datasets__detail']}>
              {status === 'loading' ? (
                <div className={styles['datasets__detail-scroll']}>
                  <span className={styles['datasets__skel']} style={{ width: 120, height: 34, marginBottom: 22 }} />
                  <span className={styles['datasets__skel']} style={{ width: '90%', marginBottom: 8 }} />
                  <span className={styles['datasets__skel']} style={{ width: '70%' }} />
                </div>
              ) : !selected ? (
                <div className={styles['datasets__detail-empty']}>
                  <Boxes size={30} />
                  <p>Select a dataset to inspect its categories, questions, and eval type.</p>
                </div>
              ) : (
                <DetailView
                  dataset={selected}
                  tagFilter={tagFilter}
                  toggleTag={toggleTag}
                />
              )}
            </section>
          </>
        )}
      </div>
    </div>
  );
}

// ---------------------------------------------------------------------------

interface DetailViewProps {
  dataset: Dataset;
  tagFilter: string[];
  toggleTag: (t: string) => void;
}

function DetailView({ dataset: d, tagFilter, toggleTag }: DetailViewProps) {
  const accent = hashColor(d.category);
  const tags = d.dataset_categories ?? [];

  return (
    <div className={styles['datasets__detail-scroll']} key={d.id}>
      <div className={styles['datasets__hero']}>
        <span className={styles['datasets__hero-bar']} style={{ background: accent.fg }} />
        <div className={styles['datasets__hero-top']}>
          <div>
            <span className={styles['datasets__hero-type']} style={{ color: accent.fg }}>{d.category}</span>
            <h2>{d.name}</h2>
          </div>
          <span className={styles['datasets__source-badge']}>{d.dataset_type}</span>
        </div>
        <p className={styles['datasets__hero-desc']}>{d.description}</p>
      </div>

      <div className={styles['datasets__stats']}>
        <Stat label="Questions" value={d.question_count.toLocaleString()} />
        <Stat label="Categories" value={tags.length || '—'} />
        <Stat label="Eval Type" value={d.eval_type} mono />
        <Stat label="Source" value={d.dataset_type} mono />
      </div>

      <div className={styles['datasets__section']}>
        <div className={styles['datasets__section-head']}>
          <h3>Categories {tags.length > 0 && <em>{tags.length}</em>}</h3>
          {tags.length > 0 && (
            <span className={styles['datasets__section-hint']}>
              click to find similar datasets <ArrowRight size={11} />
            </span>
          )}
        </div>

        {tags.length === 0 ? (
          <div className={styles['datasets__single']}>
            <Boxes size={15} />
            <div>
              <strong>No categories tagged.</strong>
              <span>This dataset isn't broken down into subject areas.</span>
            </div>
          </div>
        ) : (
          <div className={styles['datasets__caps']}>
            {tags.map((t) => {
              const col = hashColor(t);
              const active = tagFilter.includes(t);
              return (
                <button
                  key={t}
                  className={`${styles['datasets__cap']} ${active ? styles['datasets__cap--active'] : ''}`}
                  style={active
                    ? { background: col.fg, color: '#fff', borderColor: col.fg }
                    : { background: col.bg, color: col.fg, borderColor: 'transparent' }}
                  onClick={() => toggleTag(t)}
                >
                  {t}{active && <Check size={12} />}
                </button>
              );
            })}
          </div>
        )}
      </div>
    </div>
  );
}

function Stat({ label, value, mono }: { label: string; value: string | number; mono?: boolean }) {
  return (
    <div className={styles['datasets__stat']}>
      <span className={`${styles['datasets__stat-val']} ${mono ? styles['datasets__stat-val--mono'] : ''}`}>
        {value}
      </span>
      <span className={styles['datasets__stat-label']}>{label}</span>
    </div>
  );
}
