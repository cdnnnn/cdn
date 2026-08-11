import { useEffect, useMemo, useState } from 'react';
import { RefreshCw, Search, ExternalLink, Layers, AlertTriangle, Database, ListFilter, X } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchBenchmarks } from '../../store/slices/benchmarksSlice';
import type { Benchmark } from '../../types';
import { SkeletonCards } from '../common/Skeleton';
import styles from './Datasets.module.scss';

// Deterministic color hash so the same capability always gets the same pill
// color across cards/renders, without a hardcoded lookup table. Palette
// matches the app's ink/paper/signal design tokens.
const PILL_COLORS = [
  { bg: '#ECEDFF', fg: '#2B2BF5' }, // signal
  { bg: '#FDF3E3', fg: '#E08600' }, // amber
  { bg: '#E7F7EF', fg: '#0FA968' }, // ok
  { bg: '#FDECEC', fg: '#DC2626' }, // danger
  { bg: '#E6F4FB', fg: '#0369A1' }, // sky
  { bg: '#F1EDFB', fg: '#6D28D9' }, // violet
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
  const [subgroupsFor, setSubgroupsFor] = useState<Benchmark | null>(null);

  useEffect(() => {
    dispatch(fetchBenchmarks());
  }, [dispatch]);

  const types = useMemo(() => ['All', ...new Set(items.map((b) => b.type))], [items]);
  const filtered = items.filter((b) => {
    if (typeFilter !== 'All' && b.type !== typeFilter) return false;
    const q = search.toLowerCase();
    return !q || b.name.toLowerCase().includes(q) || b.description.toLowerCase().includes(q);
  });

  return (
    <div className="page-enter pg-shell">
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

      <div className={styles['datasets__toolbar']}>
        <div className={styles['datasets__search']}>
          <Search size={16} />
          <input placeholder="Search datasets…" value={search} onChange={(e) => setSearch(e.target.value)} />
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

      <div className="pg-body">
        {status === 'failed' && (
          <div className={`${styles['datasets__state']} ${styles['datasets__state--error']}`}>
            <AlertTriangle size={28} />
            <div>{error || 'Failed to load datasets.'}</div>
            <button className={styles['datasets__refresh-btn']} onClick={() => dispatch(fetchBenchmarks())}>
              Retry
            </button>
          </div>
        )}

        {status === 'succeeded' && filtered.length === 0 && (
          <div className={styles['datasets__state']}>
            <Layers size={28} />
            <div>No datasets match your search.</div>
          </div>
        )}

        <div className={styles['datasets__grid']}>
          {status === 'loading' && <SkeletonCards count={6} />}
          {status !== 'loading' &&
            filtered.map((b) => {
              // Defensive per spec §3.2 / §5 — tasks & required_capabilities are
              // normalized to [] in benchmarksApi.list(), but reads still fall
              // back defensively here in case a consumer bypasses that layer.
              const tasks = b.tasks ?? [];
              const caps = b.required_capabilities ?? [];
              return (
                <div className={styles['datasets__card']} key={b.name}>
                  <div className={styles['datasets__card-hdr']}>
                    <div>
                      <div className={styles['datasets__card-name']}>{b.name}</div>
                      <span className={styles['datasets__type-tag']}>{b.type}</span>
                    </div>
                    <div className={styles['datasets__card-count']}>
                      <div className={styles['datasets__card-count-val']}>{b.task_count.toLocaleString()}</div>
                      <div className={styles['datasets__card-count-label']}>tasks</div>
                    </div>
                  </div>

                  <div className={styles['datasets__desc']}>{b.description}</div>

                  <div className={styles['datasets__stat-row']}>
                    <span>{b.task_count.toLocaleString()} tasks</span>
                    <span>{caps.length} capabilities</span>
                    <span>{b.huggingface_dataset.split('/')[0]}</span>
                  </div>

                  <div className={styles['datasets__pill-row']}>
                    {caps.map((c) => {
                      const color = hashColor(c);
                      return (
                        <span key={c} className={styles['datasets__pill']} style={{ background: color.bg, color: color.fg }}>
                          {c}
                        </span>
                      );
                    })}
                  </div>

                  <div className={styles['datasets__foot']}>
                    {tasks.length > 0 ? (
                      <button className={styles['datasets__subgroups-btn']} onClick={() => setSubgroupsFor(b)}>
                        View subgroups
                      </button>
                    ) : (
                      <span className={styles['datasets__empty-foot-slot']} />
                    )}
                    <a
                      className={styles['datasets__source-link']}
                      href={`https://huggingface.co/datasets/${b.huggingface_dataset}`}
                      target="_blank"
                      rel="noreferrer"
                    >
                      Source <ExternalLink size={12} />
                    </a>
                  </div>
                </div>
              );
            })}
        </div>
      </div>

      {subgroupsFor && (
        <div className={styles['datasets__modal-overlay']} onClick={() => setSubgroupsFor(null)}>
          <div className={styles['datasets__modal']} onClick={(e) => e.stopPropagation()}>
            <div className={styles['datasets__modal-hdr']}>
              <span>{subgroupsFor.name} — subgroups</span>
              <button className={styles['datasets__modal-close']} onClick={() => setSubgroupsFor(null)}>
                <X size={13} /> Close
              </button>
            </div>
            <div className={styles['datasets__modal-body']}>
              {(subgroupsFor.tasks ?? []).map((t) => (
                <div key={t.value} className={styles['datasets__modal-row']}>
                  <span className={styles['datasets__modal-row-name']}>{t.name}</span>
                  <code className={styles['datasets__modal-row-code']}>{t.value}</code>
                </div>
              ))}
            </div>
          </div>
        </div>
      )}
    </div>
  );
}


























@use '../../styles/_variables' as *;

// ===========================================================================
// Datasets (Test Suite Library) — matches the Run Console / Dashboard /
// Providers / Model Catalog design system: ink/paper palette, ultramarine
// signal accent, mono instrument labels, hover-lift cards.
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
$amber:    #E08600;
$amber-wash: #FDF3E3;
$danger:   #DC2626;
$danger-wash: #FDECEC;

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

.datasets {
  // ---- header ---------------------------------------------------------------
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

  // ---- toolbar ----------------------------------------------------------------
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

  // ---- state banners (error / empty) -------------------------------------------
  &__state {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 12px;
    padding: 48px 24px;
    margin-bottom: 16px;
    border: 1px dashed $line;
    border-radius: 16px;
    background: $paper;
    color: $ink-2;
    font-size: 0.875rem;
    text-align: center;

    svg { color: $ink-3; }
  }

  &__state--error svg { color: $danger; }

  // ---- card grid ----------------------------------------------------------------
  &__grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(310px, 1fr));
    gap: 12px;
  }

  &__card {
    position: relative;
    display: flex;
    flex-direction: column;
    height: 100%;
    padding: 18px 19px;
    border: 1.5px solid $line;
    border-radius: 16px;
    background: $card;
    transition: border-color 0.16s ease, box-shadow 0.16s ease, transform 0.16s ease;

    &:hover {
      border-color: $ink-3;
      box-shadow: $lift;
      transform: translateY(-2px);
    }
  }

  &__card-hdr {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
  }

  &__card-name {
    font-family: $display;
    font-size: 0.9375rem;
    font-weight: 700;
    color: $ink;
    line-height: 1.3;
  }

  &__type-tag {
    display: inline-flex;
    margin-top: 7px;
    font-family: $mono;
    font-size: 0.625rem;
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $amber;
    background: $amber-wash;
    border-radius: 6px;
    padding: 3px 8px;
  }

  &__card-count {
    flex-shrink: 0;
    text-align: right;
  }

  &__card-count-val {
    font-family: $mono;
    font-size: 1.5rem;
    font-weight: 700;
    color: $ink;
    letter-spacing: -0.02em;
    line-height: 1;
  }

  &__card-count-label {
    font-size: 0.65625rem;
    font-weight: 650;
    color: $ink-3;
    margin-top: 3px;
  }

  &__desc {
    margin-top: 11px;
    font-size: 0.8125rem;
    color: $ink-2;
    line-height: 1.5;
  }

  &__stat-row {
    display: flex;
    flex-wrap: wrap;
    gap: 14px;
    margin-top: 13px;
    padding-top: 12px;
    border-top: 1px solid $line-2;
    font-family: $mono;
    font-size: 0.71875rem;
    color: $ink-2;
  }

  &__pill-row {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    margin-top: 12px;
  }

  &__pill {
    font-family: $mono;
    font-size: 0.65625rem;
    font-weight: 700;
    letter-spacing: 0.02em;
    border-radius: 6px;
    padding: 3px 8px;
    white-space: nowrap;
  }

  &__foot {
    flex: 1;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 10px;
    margin-top: 14px;
    padding-top: 13px;
    border-top: 1px solid $line-2;
  }

  &__subgroups-btn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 6px 11px;
    border: 1px solid $line;
    border-radius: 8px;
    background: $card;
    color: $ink-2;
    font-family: $sans;
    font-size: 0.75rem;
    font-weight: 650;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $ink-3; color: $ink; background: $paper; }
  }

  &__source-link {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    font-family: $mono;
    font-size: 0.75rem;
    font-weight: 650;
    color: $signal;
    text-decoration: none;
    transition: color 0.15s ease;

    &:hover { color: $signal-2; text-decoration: underline; }
  }

  &__empty-foot-slot {
    display: inline-block;
  }

  // ---- subgroups modal ----------------------------------------------------------
  &__modal-overlay {
    position: fixed;
    inset: 0;
    z-index: 50;
    background: rgba(20, 22, 27, 0.5);
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 24px;
    animation: datasets-fade-in 0.16s ease-out;
  }

  &__modal {
    width: min(480px, 100%);
    max-height: 80vh;
    display: flex;
    flex-direction: column;
    background: $card;
    border-radius: 18px;
    border: 1px solid $line;
    box-shadow: 0 24px 60px -20px rgba(20, 22, 27, 0.4);
    overflow: hidden;
    animation: datasets-modal-in 0.2s cubic-bezier(0.22, 0.72, 0.16, 1);
  }

  &__modal-hdr {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
    padding: 16px 18px;
    border-bottom: 1px solid $line;
    background: $paper;

    span {
      font-family: $display;
      font-size: 0.9375rem;
      font-weight: 700;
      color: $ink;
    }
  }

  &__modal-close {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 6px 11px;
    border: 1px solid $line;
    border-radius: 8px;
    background: $card;
    color: $ink-2;
    font-family: $sans;
    font-size: 0.75rem;
    font-weight: 650;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $ink-3; color: $ink; }
  }

  &__modal-body {
    flex: 1;
    overflow-y: auto;
    padding: 12px 14px 16px;
    display: flex;
    flex-direction: column;
    gap: 6px;
  }

  &__modal-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
    padding: 10px 12px;
    border: 1px solid $line;
    border-radius: 10px;
    background: $paper;
  }

  &__modal-row-name {
    font-size: 0.8125rem;
    font-weight: 600;
    color: $ink;
  }

  &__modal-row-code {
    font-family: $mono;
    font-size: 0.71875rem;
    color: $ink-2;
    background: $card;
    border: 1px solid $line;
    border-radius: 6px;
    padding: 2px 7px;
    white-space: nowrap;
  }
}

@keyframes datasets-fade-in {
  from { opacity: 0; }
  to { opacity: 1; }
}

@keyframes datasets-modal-in {
  from { opacity: 0; transform: translateY(8px) scale(0.98); }
  to { opacity: 1; transform: none; }
}

@media (max-width: 768px) {
  .datasets__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .datasets__toolbar { padding: 12px 18px; }
  .datasets__grid { grid-template-columns: 1fr; }
}
