//Datasets.tsx
import { useEffect, useMemo, useState, useRef, useCallback } from 'react';
import {
  RefreshCw, Search, ExternalLink, Layers, AlertTriangle, Database, ListFilter, X,
  Check, Boxes, Hash, ArrowRight, Filter, ChevronsUpDown,
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchBenchmarks } from '../../store/slices/benchmarksSlice';
import type { Benchmark } from '../../types';
import styles from './Datasets.module.scss';

// Deterministic color hash so the same capability/type always gets the same
// pill color across renders, without a hardcoded lookup table. Palette matches
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
  const { items, status, error } = useAppSelector((s) => s.benchmarks);

  const [search, setSearch] = useState('');
  const [typeFilter, setTypeFilter] = useState('All');
  const [capFilter, setCapFilter] = useState<string[]>([]);      // active capability facets
  const [selectedName, setSelectedName] = useState<string | null>(null);
  const [subQuery, setSubQuery] = useState('');                  // subgroup filter inside detail
  const searchRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    dispatch(fetchBenchmarks());
  }, [dispatch]);

  const types = useMemo(() => ['All', ...new Set(items.map((b) => b.type))], [items]);
  const maxCount = useMemo(() => items.reduce((m, b) => Math.max(m, b.task_count), 1), [items]);

  const filtered = useMemo(() => {
    const q = search.trim().toLowerCase();
    return items.filter((b) => {
      if (typeFilter !== 'All' && b.type !== typeFilter) return false;
      const caps = b.required_capabilities ?? [];
      if (capFilter.length && !capFilter.some((c) => caps.includes(c))) return false;
      if (!q) return true;
      return b.name.toLowerCase().includes(q) || b.description.toLowerCase().includes(q);
    });
  }, [items, search, typeFilter, capFilter]);

  // Keep a valid selection as the filtered set changes.
  useEffect(() => {
    if (!filtered.length) return;
    if (!filtered.some((b) => b.name === selectedName)) setSelectedName(filtered[0].name);
  }, [filtered, selectedName]);

  const selected = items.find((b) => b.name === selectedName) ?? null;

  const toggleCap = useCallback((c: string) => {
    setCapFilter((prev) => (prev.includes(c) ? prev.filter((x) => x !== c) : [...prev, c]));
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
      const idx = filtered.findIndex((b) => b.name === selectedName);
      if (idx === -1) {
        if (filtered[0]) setSelectedName(filtered[0].name);
        return;
      }
      const next = e.key === 'ArrowDown'
        ? Math.min(idx + 1, filtered.length - 1)
        : Math.max(idx - 1, 0);
      setSelectedName(filtered[next].name);
    };
    window.addEventListener('keydown', onKey);
    return () => window.removeEventListener('keydown', onKey);
  }, [filtered, selectedName]);

  // Defensive per spec §3.2 / §5 — normalized to [] in benchmarksApi.list(),
  // but reads still fall back defensively here.
  const subgroups = selected?.tasks ?? [];
  const selCaps = selected?.required_capabilities ?? [];
  const shownSubgroups = subgroups.filter(
    (t) =>
      !subQuery ||
      t.name.toLowerCase().includes(subQuery.toLowerCase()) ||
      t.value.toLowerCase().includes(subQuery.toLowerCase())
  );

  return (
    <div className="page-enter pg-shell">
      {/* ---- header (unchanged) ------------------------------------------- */}
      <div className={styles['datasets__header']}>
        <div>
          <p className={styles['datasets__header-eyebrow']}>Datasets</p>
          <h1>Test Suite Library</h1>
          <p className={styles['datasets__header-sub']}>
            Browse every benchmark available for evaluations, independent of any single wizard run.
          </p>
        </div>
        <div className={styles['datasets__header-meta']}>
          <span className={styles['datasets__header-count']}>
            <Database size={13} /> {items.length} suites available
          </span>
          <button className={styles['datasets__refresh-btn']} onClick={() => dispatch(fetchBenchmarks())}>
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
          {types.map((t) => (
            <button
              key={t}
              className={`${styles['datasets__filter-pill']} ${typeFilter === t ? styles['datasets__filter-pill--on'] : ''}`}
              onClick={() => setTypeFilter(t)}
            >
              {t}
            </button>
          ))}
        </div>
      </div>

      {/* ---- active capability facets ------------------------------------- */}
      {capFilter.length > 0 && (
        <div className={styles['datasets__facets']}>
          <Filter size={12} />
          <span className={styles['datasets__facets-lead']}>Showing suites tagged</span>
          {capFilter.map((c) => {
            const col = hashColor(c);
            return (
              <button
                key={c}
                className={styles['datasets__facet']}
                style={{ background: col.bg, color: col.fg }}
                onClick={() => toggleCap(c)}
              >
                {c} <X size={11} />
              </button>
            );
          })}
          <button className={styles['datasets__facets-clear']} onClick={() => setCapFilter([])}>
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
            <button className={styles['datasets__refresh-btn']} onClick={() => dispatch(fetchBenchmarks())}>
              Retry
            </button>
          </div>
        )}

        {status !== 'failed' && (
          <>
            {/* LIST RAIL — the scale-spine lives here */}
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
                      <span className={styles['datasets__skel']} style={{ width: '100%', height: 3 }} />
                      <span className={styles['datasets__skel']} style={{ width: '35%' }} />
                    </div>
                  ))}

                {status !== 'loading' && filtered.length === 0 && (
                  <div className={styles['datasets__empty-rail']}>
                    <Layers size={22} />
                    <p>No suites match.<br />Loosen a filter to see more.</p>
                  </div>
                )}

                {status !== 'loading' &&
                  filtered.map((b) => {
                    const on = b.name === selectedName;
                    const accent = hashColor(b.type);
                    const caps = b.required_capabilities ?? [];
                    const w = Math.max(4, (b.task_count / maxCount) * 100);
                    return (
                      <button
                        key={b.name}
                        className={`${styles['datasets__row']} ${on ? styles['datasets__row--on'] : ''}`}
                        onClick={() => setSelectedName(b.name)}
                      >
                        <span className={styles['datasets__row-accent']} style={{ background: accent.fg }} />
                        <div className={styles['datasets__row-top']}>
                          <span className={styles['datasets__row-name']}>{b.name}</span>
                          <span className={styles['datasets__row-count']}>{b.task_count.toLocaleString()}</span>
                        </div>
                        {/* scale-spine: instant sense of dataset size across the whole library */}
                        <div className={styles['datasets__spine']}>
                          <span style={{ width: `${w}%`, background: accent.fg }} />
                        </div>
                        <div className={styles['datasets__row-foot']}>
                          <span className={styles['datasets__row-type']} style={{ color: accent.fg }}>
                            {b.type}
                          </span>
                          <span className={styles['datasets__row-dots']}>
                            {caps.slice(0, 4).map((c) => (
                              <i key={c} style={{ background: hashColor(c).fg }} title={c} />
                            ))}
                          </span>
                        </div>
                      </button>
                    );
                  })}
              </div>
            </aside>

            {/* DETAIL — everything the modal used to hide, now first-class */}
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
                  <p>Select a suite to inspect its subgroups, capabilities, and source.</p>
                </div>
              ) : (
                <DetailView
                  benchmark={selected}
                  subgroups={subgroups}
                  shownSubgroups={shownSubgroups}
                  caps={selCaps}
                  capFilter={capFilter}
                  toggleCap={toggleCap}
                  subQuery={subQuery}
                  setSubQuery={setSubQuery}
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
  benchmark: Benchmark;
  subgroups: NonNullable<Benchmark['tasks']>;
  shownSubgroups: NonNullable<Benchmark['tasks']>;
  caps: string[];
  capFilter: string[];
  toggleCap: (c: string) => void;
  subQuery: string;
  setSubQuery: (v: string) => void;
}

function DetailView({
  benchmark: b, subgroups, shownSubgroups, caps, capFilter, toggleCap, subQuery, setSubQuery,
}: DetailViewProps) {
  const accent = hashColor(b.type);
  return (
    <div className={styles['datasets__detail-scroll']} key={b.name}>
      <div className={styles['datasets__hero']}>
        <span className={styles['datasets__hero-bar']} style={{ background: accent.fg }} />
        <div className={styles['datasets__hero-top']}>
          <div>
            <span className={styles['datasets__hero-type']} style={{ color: accent.fg }}>{b.type}</span>
            <h2>{b.name}</h2>
          </div>
          <a
            className={styles['datasets__source']}
            href={`https://huggingface.co/datasets/${b.huggingface_dataset}`}
            target="_blank"
            rel="noreferrer"
          >
            {b.huggingface_dataset} <ExternalLink size={13} />
          </a>
        </div>
        <p className={styles['datasets__hero-desc']}>{b.description}</p>
      </div>

      <div className={styles['datasets__stats']}>
        <Stat label="Tasks" value={b.task_count.toLocaleString()} />
        <Stat label="Subgroups" value={subgroups.length || '—'} />
        <Stat label="Capabilities" value={caps.length} />
        <Stat label="Publisher" value={b.huggingface_dataset.split('/')[0]} mono />
      </div>

      <div className={styles['datasets__section']}>
        <div className={styles['datasets__section-head']}>
          <h3>Capabilities</h3>
          <span className={styles['datasets__section-hint']}>
            click to find similar suites <ArrowRight size={11} />
          </span>
        </div>
        <div className={styles['datasets__caps']}>
          {caps.map((c) => {
            const col = hashColor(c);
            const active = capFilter.includes(c);
            return (
              <button
                key={c}
                className={`${styles['datasets__cap']} ${active ? styles['datasets__cap--active'] : ''}`}
                style={active
                  ? { background: col.fg, color: '#fff', borderColor: col.fg }
                  : { background: col.bg, color: col.fg, borderColor: 'transparent' }}
                onClick={() => toggleCap(c)}
              >
                {c}{active && <Check size={12} />}
              </button>
            );
          })}
        </div>
      </div>

      <div className={styles['datasets__section']}>
        <div className={styles['datasets__section-head']}>
          <h3>Subgroups {subgroups.length > 0 && <em>{subgroups.length}</em>}</h3>
          {subgroups.length > 6 && (
            <div className={styles['datasets__subsearch']}>
              <Search size={13} />
              <input
                placeholder="Filter subgroups…"
                value={subQuery}
                onChange={(e) => setSubQuery(e.target.value)}
              />
              {subQuery && (
                <button
                  className={styles['datasets__subsearch-clear']}
                  onClick={() => setSubQuery('')}
                  aria-label="Clear filter"
                >
                  <X size={12} />
                </button>
              )}
            </div>
          )}
        </div>

        {subgroups.length === 0 ? (
          <div className={styles['datasets__single']}>
            <Hash size={15} />
            <div>
              <strong>Runs as a single group.</strong>
              <span>All {b.task_count.toLocaleString()} tasks execute together — no subset selection needed.</span>
            </div>
          </div>
        ) : (
          <div className={styles['datasets__subgrid']}>
            {shownSubgroups.map((t) => (
              <span key={t.value} className={styles['datasets__sub']}>
                <i className={styles['datasets__sub-dot']} style={{ background: accent.fg }} />
                {t.name}
              </span>
            ))}
            {shownSubgroups.length === 0 && (
              <p className={styles['datasets__sub-none']}>No subgroup matches “{subQuery}”.</p>
            )}
          </div>
        )}
      </div>

      <div className={styles['datasets__cta-row']}>
        <a
          className={styles['datasets__secondary']}
          href={`https://huggingface.co/datasets/${b.huggingface_dataset}`}
          target="_blank"
          rel="noreferrer"
        >
          View on Hugging Face <ExternalLink size={13} />
        </a>
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





















//Datasets.module.scss
@use '../../styles/_variables' as *;

// ===========================================================================
// Datasets (Test Suite Library) — master/detail library browser.
// Header + toolbar are unchanged from the original. The body below replaces
// the card grid + modal with a scrollable list rail and a rich detail pane.
// Uses the app's existing font variables ($font-mono / $font-body /
// $font-display) — no font-family is introduced or overridden here.
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
$amber:    #E08600;
$amber-wash: #FDF3E3;
$danger:   #DC2626;

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

$soft: 0 1px 2px rgba(20, 22, 27, 0.05);

%micro {
  font-family: $mono;
  font-size: 0.6875rem;
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

.datasets {
  // ---- header (unchanged) ---------------------------------------------------
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
    max-width: 52ch;
  }

  &__header-meta {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 10px;
    margin-bottom: 3px;
  }

  &__header-count {
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
  }

  &__refresh-btn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 8px 13px;
    border: 1px solid $line;
    border-radius: 999px;
    background: $card;
    color: $ink-2;
    font-family: $sans;
    font-size: 0.78125rem;
    font-weight: 650;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $ink-3; color: $ink; background: $paper; }
  }

  // ---- toolbar (unchanged) --------------------------------------------------
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
    max-width: 360px;
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
      &:focus { outline: none; border-color: $signal; background: $card; }
    }
  }

  &__toolbar-label {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 5px 8px 5px 9px;
    color: $ink-3;
  }

  &__filters {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    padding: 4px;
    background: $paper;
    border: 1px solid $line;
    border-radius: 999px;
    flex-wrap: wrap;
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

  // ---- capability facet bar -------------------------------------------------
  &__facets {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 8px;
    flex-wrap: wrap;
    padding: 10px 32px;
    background: $wash;
    border-bottom: 1px solid $line;
    color: $signal-2;
  }

  &__facets-lead {
    font-size: 0.75rem;
    font-weight: 650;
    color: $ink-2;
  }

  &__facet {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 4px 6px 4px 10px;
    border: 0;
    border-radius: 999px;
    font-family: $mono;
    font-size: 0.6875rem;
    font-weight: 700;
    cursor: pointer;
    transition: filter 0.12s ease;

    &:hover { filter: brightness(0.96); }
  }

  &__facets-clear {
    margin-left: 2px;
    border: 0;
    background: none;
    color: $signal;
    font-family: $sans;
    font-size: 0.75rem;
    font-weight: 700;
    cursor: pointer;

    &:hover { text-decoration: underline; }
  }

  // ---- body split -----------------------------------------------------------
  // NOTE: relies on the page shell (.pg-shell) being a flex column that gives
  // this element the remaining height. If your shell differs, set a height /
  // min-height on &__body instead of flex:1.
  &__body {
    flex: 1;
    min-height: 0;
    display: grid;
    grid-template-columns: minmax(300px, 360px) 1fr;
  }

  // ---- list rail ------------------------------------------------------------
  &__rail {
    display: flex;
    flex-direction: column;
    min-height: 0;
    border-right: 1px solid $line;
    background: $card;
  }

  &__rail-head {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 12px 18px;
    border-bottom: 1px solid $line-2;
    font-family: $mono;
    font-size: 0.6875rem;
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ink-3;
  }

  &__rail-hint {
    display: inline-flex;
    align-items: center;
    gap: 5px;
  }

  &__rail-scroll {
    flex: 1;
    overflow-y: auto;
    padding: 8px;
    display: flex;
    flex-direction: column;
    gap: 2px;
  }

  &__row {
    position: relative;
    text-align: left;
    width: 100%;
    border: 0;
    cursor: pointer;
    background: transparent;
    border-radius: 12px;
    padding: 11px 13px 12px;
    display: flex;
    flex-direction: column;
    gap: 8px;
    font-family: $sans;
    transition: background 0.13s ease;

    &:hover { background: $paper; }

    &--on { background: $wash; }
  }

  &__row-accent {
    position: absolute;
    left: 3px;
    top: 12px;
    bottom: 12px;
    width: 3px;
    border-radius: 2px;
    opacity: 0;
    transition: opacity 0.13s ease;

    .datasets__row--on & { opacity: 1; }
  }

  &__row-top {
    display: flex;
    align-items: baseline;
    justify-content: space-between;
    gap: 10px;
  }

  &__row-name {
    font-family: $display;
    font-size: 0.9375rem;
    font-weight: 600;
    color: $ink;
  }

  &__row-count {
    font-family: $mono;
    font-size: 0.75rem;
    font-weight: 700;
    color: $ink-3;
  }

  &__spine {
    height: 3px;
    border-radius: 2px;
    background: $line-2;
    overflow: hidden;

    span { display: block; height: 100%; border-radius: 2px; opacity: 0.85; }
  }

  &__row-foot {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__row-type {
    font-family: $mono;
    font-size: 0.625rem;
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
  }

  &__row-dots {
    display: inline-flex;
    gap: 4px;

    i { width: 6px; height: 6px; border-radius: 99px; display: block; }
  }

  &__empty-rail {
    margin: auto;
    text-align: center;
    color: $ink-3;
    padding: 40px 16px;

    svg { margin-bottom: 8px; }
    p { font-size: 0.8125rem; line-height: 1.5; margin: 0; }
  }

  // ---- loading skeleton -----------------------------------------------------
  &__skel-row {
    padding: 11px 13px;
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  &__skel {
    display: block;
    height: 10px;
    border-radius: 6px;
    background: linear-gradient(90deg, $line-2 25%, $paper 50%, $line-2 75%);
    background-size: 200% 100%;
    animation: datasets-shimmer 1.2s ease-in-out infinite;
  }

  // ---- detail ---------------------------------------------------------------
  &__detail {
    min-height: 0;
    display: flex;
    flex-direction: column;
  }

  &__detail-scroll {
    flex: 1;
    overflow-y: auto;
    padding: 26px 30px 40px;
    animation: datasets-detail-in 0.22s ease;
  }

  &__detail-empty {
    margin: auto;
    text-align: center;
    color: $ink-3;
    max-width: 280px;

    svg { margin-bottom: 10px; }
    p { font-size: 0.875rem; line-height: 1.5; }
  }

  &__hero {
    position: relative;
    padding-left: 18px;
    margin-bottom: 22px;
  }

  &__hero-bar {
    position: absolute;
    left: 0;
    top: 4px;
    bottom: 4px;
    width: 4px;
    border-radius: 3px;
  }

  &__hero-top {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
  }

  &__hero-type {
    font-family: $mono;
    font-size: 0.6875rem;
    font-weight: 700;
    letter-spacing: 0.1em;
    text-transform: uppercase;
  }

  &__hero {
    h2 {
      font-family: $display;
      font-size: 1.875rem;
      font-weight: 700;
      letter-spacing: -0.02em;
      margin: 5px 0 0;
      line-height: 1.1;
      color: $ink;
    }
  }

  &__hero-desc {
    margin: 14px 0 0;
    font-size: 0.9375rem;
    line-height: 1.6;
    color: $ink-2;
  }

  &__source {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-family: $mono;
    font-size: 0.75rem;
    font-weight: 500;
    color: $signal;
    text-decoration: none;
    padding: 5px 10px;
    border: 1px solid $line;
    border-radius: 8px;
    background: $card;
    white-space: nowrap;
    transition: border-color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $signal; background: $wash; }
  }

  &__stats {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 1px;
    background: $line;
    border: 1px solid $line;
    border-radius: 14px;
    overflow: hidden;
    margin-bottom: 26px;
  }

  &__stat {
    background: $card;
    padding: 15px 16px;
    display: flex;
    flex-direction: column;
    gap: 3px;
  }

  &__stat-val {
    font-family: $display;
    font-size: 1.375rem;
    font-weight: 700;
    color: $ink;
    letter-spacing: -0.02em;
    line-height: 1;

    &--mono { font-family: $mono; font-size: 0.9375rem; font-weight: 700; }
  }

  &__stat-label {
    font-family: $mono;
    font-size: 0.625rem;
    font-weight: 700;
    letter-spacing: 0.1em;
    text-transform: uppercase;
    color: $ink-3;
  }

  &__section {
    margin-bottom: 26px;
  }

  &__section-head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
    margin-bottom: 12px;

    h3 {
      font-family: $display;
      font-size: 0.9375rem;
      font-weight: 700;
      color: $ink;
      margin: 0;
      display: flex;
      align-items: center;
      gap: 8px;

      em {
        font-family: $mono;
        font-style: normal;
        font-size: 0.6875rem;
        font-weight: 700;
        color: $ink-3;
        background: $paper;
        border: 1px solid $line;
        border-radius: 99px;
        padding: 2px 8px;
      }
    }
  }

  &__section-hint {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    font-size: 0.71875rem;
    color: $ink-3;
  }

  &__caps {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
  }

  &__cap {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 6px 12px;
    border: 1px solid;
    border-radius: 8px;
    font-family: $mono;
    font-size: 0.71875rem;
    font-weight: 700;
    cursor: pointer;
    transition: transform 0.13s ease;

    &:hover { transform: translateY(-1px); }

    &--active { box-shadow: $soft; }
  }

  &__subsearch {
    position: relative;
    display: inline-flex;
    align-items: center;

    > svg {
      position: absolute;
      left: 13px;
      color: $ink-3;
      pointer-events: none;
    }

    input {
      width: 220px;
      max-width: 46vw;
      border: 1.5px solid $line;
      border-radius: 999px;
      padding: 8px 32px 8px 34px;
      font-size: 0.78125rem;
      font-family: $sans;
      background: $paper;
      color: $ink;
      transition: border-color 0.15s ease, background 0.15s ease, box-shadow 0.15s ease;

      &::placeholder { color: $ink-3; }
      &:focus {
        outline: none;
        border-color: $signal;
        background: $card;
        box-shadow: 0 0 0 3px rgba(43, 43, 245, 0.13);
      }
    }
  }

  &__subsearch-clear {
    position: absolute;
    right: 7px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 19px;
    height: 19px;
    padding: 0;
    border: 0;
    border-radius: 99px;
    background: $line-2;
    color: $ink-2;
    cursor: pointer;
    transition: background 0.13s ease, color 0.13s ease;

    &:hover { background: $ink-3; color: #fff; }
  }

  // Subgroups are chips that size to their own label — no fixed columns,
  // so a two-word subset and a long one both sit comfortably on the row.
  &__subgrid {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
  }

  &__sub {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    padding: 7px 15px 7px 12px;
    border: 1px solid $line;
    border-radius: 999px;
    background: $card;
    font-family: $sans;
    font-size: 0.8125rem;
    font-weight: 600;
    color: $ink-2;
    text-transform: capitalize;
    transition: border-color 0.14s ease, color 0.14s ease, background 0.14s ease,
      transform 0.14s ease, box-shadow 0.14s ease;

    &:hover {
      border-color: $ink-3;
      color: $ink;
      background: $paper;
      transform: translateY(-1px);
      box-shadow: $soft;
    }
  }

  &__sub-dot {
    width: 6px;
    height: 6px;
    border-radius: 99px;
    flex-shrink: 0;
    opacity: 0.9;
  }

  &__sub-none {
    width: 100%;
    font-size: 0.8125rem;
    color: $ink-3;
  }

  &__single {
    display: flex;
    gap: 11px;
    align-items: flex-start;
    padding: 14px 16px;
    border: 1px dashed $line;
    border-radius: 12px;
    background: $paper;
    color: $ink-3;

    strong { display: block; font-size: 0.8125rem; color: $ink; font-weight: 650; }
    span { display: block; font-size: 0.78125rem; color: $ink-2; margin-top: 2px; }
  }

  &__cta-row {
    display: flex;
    gap: 10px;
    flex-wrap: wrap;
    padding-top: 20px;
    border-top: 1px solid $line-2;
  }

  &__secondary {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    padding: 10px 16px;
    border: 1px solid $line;
    border-radius: 10px;
    background: $card;
    color: $ink-2;
    font-family: $sans;
    font-size: 0.84375rem;
    font-weight: 650;
    text-decoration: none;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $ink-3; color: $ink; background: $paper; }
  }

  // ---- state banner (error) -------------------------------------------------
  &__state {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 12px;
    padding: 48px 24px;
    margin: 24px 32px;
    border: 1px dashed $line;
    border-radius: 16px;
    background: $paper;
    color: $ink-2;
    font-size: 0.875rem;
    text-align: center;

    svg { color: $ink-3; }
  }

  &__state--error svg { color: $danger; }
}

@keyframes datasets-shimmer {
  from { background-position: 200% 0; }
  to { background-position: -200% 0; }
}

@keyframes datasets-detail-in {
  from { opacity: 0; transform: translateY(6px); }
  to { opacity: 1; transform: none; }
}

@media (max-width: 820px) {
  .datasets__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .datasets__toolbar { padding: 12px 18px; }
  .datasets__facets { padding: 10px 18px; }
  .datasets__body { grid-template-columns: 1fr; grid-template-rows: minmax(180px, 34vh) 1fr; }
  .datasets__rail { border-right: 0; border-bottom: 1px solid $line; }
  .datasets__detail-scroll { padding: 20px 18px 32px; }
  .datasets__stats { grid-template-columns: repeat(2, 1fr); }
  .datasets__hero h2 { font-size: 1.5rem; }
}

@media (prefers-reduced-motion: reduce) {
  .datasets__skel,
  .datasets__detail-scroll { animation: none; }
  .datasets__row,
  .datasets__cap,
  .datasets__sub { transition: none; }
}
