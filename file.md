//Datasets.tsx
import { useEffect, useMemo, useState, type FC } from 'react';
import { useNavigate } from 'react-router-dom';
import { Database, Search, X, LayoutGrid, Tag, RefreshCw, AlertCircle, ExternalLink, ListChecks, Sparkles } from 'lucide-react';
import { fetchBenchmarks } from './api';
import type { Benchmark } from './types';
import Spinner from '../../../components/Spinner/Spinner';
import './Datasets.scss';

const CAPABILITY_TINTS = ['blue', 'violet', 'amber', 'jade', 'rose'] as const;

function capabilityTint(capability: string) {
  let hash = 0;
  for (let i = 0; i < capability.length; i += 1) hash = (hash * 31 + capability.charCodeAt(i)) >>> 0;
  return CAPABILITY_TINTS[hash % CAPABILITY_TINTS.length];
}

const TASK_TINTS = ['blue', 'violet', 'amber', 'jade'] as const;

const Datasets: FC = () => {
  const navigate = useNavigate();
  const [benchmarks, setBenchmarks] = useState<Benchmark[]>([]);
  const [total, setTotal] = useState(0);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const [query, setQuery] = useState('');
  const [type, setType] = useState('All');
  const [selectedKey, setSelectedKey] = useState<string | null>(null);

  const load = () => {
    setLoading(true);
    setError(null);
    fetchBenchmarks()
      .then((res) => {
        setBenchmarks(res.benchmarks);
        setTotal(res.total);
        setSelectedKey((prev) => prev ?? (res.benchmarks[0] ? res.benchmarks[0].name : null));
      })
      .catch((err) => {
        setError(err instanceof Error ? err.message : 'Failed to load benchmarks.');
      })
      .finally(() => setLoading(false));
  };

  useEffect(() => {
    load();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  const types = useMemo(() => {
    const set = new Set(benchmarks.map((b) => b.type));
    return ['All', ...Array.from(set).sort()];
  }, [benchmarks]);

  const filtered = useMemo(
    () =>
      benchmarks.filter((b) => {
        if (type !== 'All' && b.type !== type) return false;
        if (query && !b.name.toLowerCase().includes(query.toLowerCase()) && !b.description.toLowerCase().includes(query.toLowerCase())) {
          return false;
        }
        return true;
      }),
    [benchmarks, query, type]
  );

  const selected = useMemo(
    () => filtered.find((b) => b.name === selectedKey) ?? filtered[0] ?? null,
    [filtered, selectedKey]
  );

  return (
    <div className="datasets-page">
      <div className="datasets-page__header">
        <div className="datasets-page__header-left">
          <p className="datasets-page__header-eyebrow">Test suite library</p>
          <h1 className="datasets-page__title">Test Suites</h1>
          <p className="datasets-page__subtitle">Benchmark datasets and custom tests</p>
        </div>

        <div className="datasets-page__header-right">
          <div className="datasets-page__header-meta">
            <Database size={13} />
            {total} suites available
          </div>
          <button type="button" className="datasets-page__btn datasets-page__btn--outline" onClick={load} disabled={loading}>
            <RefreshCw size={14} strokeWidth={2.25} className={loading ? 'datasets-page__spin' : undefined} /> Refresh
          </button>
        </div>
      </div>

      <div className="datasets-page__filters">
        <div className="datasets-page__search">
          <Search size={15} />
          <input type="text" placeholder="Search test suites..." value={query} onChange={(e) => setQuery(e.target.value)} />
          {query && (
            <button type="button" className="datasets-page__search-clear" onClick={() => setQuery('')} aria-label="Clear search">
              <X size={13} />
            </button>
          )}
        </div>

        <div className="datasets-page__seg">
          {types.map((t) => (
            <button
              key={t}
              type="button"
              className={`datasets-page__seg-item${type === t ? ' datasets-page__seg-item--active' : ''}`}
              onClick={() => setType(t)}
            >
              {t === 'All' ? <LayoutGrid size={13} strokeWidth={2.25} /> : <Tag size={13} strokeWidth={2.25} />}
              {t}
            </button>
          ))}
        </div>
      </div>

      {loading && (
        <div className="datasets-page__loading">
          <Spinner label="Loading test suites…" />
        </div>
      )}

      {!loading && error && (
        <div className="datasets-page__empty datasets-page__empty--error">
          <AlertCircle size={22} />
          <p>{error}</p>
          <button type="button" className="datasets-page__btn datasets-page__btn--outline" onClick={load}>
            <RefreshCw size={14} strokeWidth={2.25} /> Try again
          </button>
        </div>
      )}

      {!loading && !error && filtered.length === 0 && (
        <div className="datasets-page__empty">
          <Database size={22} />
          <p>No test suites match your filters.</p>
        </div>
      )}

      {!loading && !error && filtered.length > 0 && (
        <div className="datasets-page__layout">
          <div className="datasets-page__sidebar">
            <div className="datasets-page__sidebar-list">
              {filtered.map((b) => {
                const isActive = selected?.name === b.name;
                return (
                  <button
                    type="button"
                    key={b.name}
                    className={`datasets-page__item${isActive ? ' datasets-page__item--active' : ''}`}
                    onClick={() => setSelectedKey(b.name)}
                  >
                    <div className="datasets-page__item-top">
                      <span className="datasets-page__item-name">{b.name}</span>
                      <span className="datasets-page__tag datasets-page__tag--blue">{b.type}</span>
                    </div>
                    <div className="datasets-page__item-meta">
                      <span>{b.required_capabilities.length} capabilities</span>
                      <span className="datasets-page__item-questions n">{b.task_count} tasks</span>
                    </div>
                  </button>
                );
              })}
            </div>
          </div>

          {selected && (
            <div className="datasets-page__detail">
              <div className="datasets-page__detail-head">
                <div className="datasets-page__detail-head-left">
                  <div className="datasets-page__detail-tags">
                    <span className="datasets-page__tag datasets-page__tag--blue">
                      <Tag size={11} strokeWidth={2.5} />
                      {selected.type}
                    </span>
                  </div>
                  <h2 className="datasets-page__detail-name">{selected.name}</h2>
                </div>

                <button type="button" className="datasets-page__btn datasets-page__btn--primary" onClick={() => navigate('/app/run-evaluation')}>
                  Use in Evaluation
                </button>
              </div>

              <p className="datasets-page__detail-desc">{selected.description}</p>

              <div className="datasets-page__stat-row">
                <div className="datasets-page__stat-card">
                  <span className="datasets-page__stat-icon">
                    <ListChecks size={16} strokeWidth={2} />
                  </span>
                  <span className="datasets-page__stat-body">
                    <span className="datasets-page__stat-card-label">Tasks</span>
                    <span className="datasets-page__stat-card-value n">{selected.task_count}</span>
                  </span>
                </div>
                <div className="datasets-page__stat-card">
                  <span className="datasets-page__stat-icon">
                    <Sparkles size={16} strokeWidth={2} />
                  </span>
                  <span className="datasets-page__stat-body">
                    <span className="datasets-page__stat-card-label">Capabilities</span>
                    <span className="datasets-page__stat-card-value n">{selected.required_capabilities.length}</span>
                  </span>
                </div>
                <div className="datasets-page__stat-card">
                  <span className="datasets-page__stat-icon">
                    <Database size={16} strokeWidth={2} />
                  </span>
                  <span className="datasets-page__stat-body">
                    <span className="datasets-page__stat-card-label">HuggingFace Dataset</span>
                    <span className="datasets-page__stat-card-value datasets-page__stat-card-value--sm">{selected.huggingface_dataset}</span>
                  </span>
                </div>
              </div>

              {selected.required_capabilities.length > 0 && (
                <>
                  <p className="datasets-page__section-title">
                    <Sparkles size={15} strokeWidth={2.25} />
                    <span>Required capabilities</span>
                  </p>
                  <div className="datasets-page__caps">
                    {selected.required_capabilities.map((c) => (
                      <span key={c} className={`datasets-page__cap-pill datasets-page__cap-pill--${capabilityTint(c)}`}>
                        {c}
                      </span>
                    ))}
                  </div>
                </>
              )}

              {selected.tasks.length > 0 && (
                <>
                  <p className="datasets-page__section-title">
                    <ListChecks size={15} strokeWidth={2.25} />
                    <span>Sample tasks</span>
                  </p>
                  <div className="datasets-page__task-grid">
                    {selected.tasks.map((t, i) => {
                      const tint = TASK_TINTS[i % TASK_TINTS.length];
                      const num = String(i + 1).padStart(2, '0');
                      return (
                        <div className={`datasets-page__task-card datasets-page__task-card--${tint}`} key={t.name}>
                          <span className="datasets-page__task-card-ghost">{num}</span>
                          <p className="datasets-page__task-card-name">{t.name}</p>
                          <p className="datasets-page__task-card-value">{t.value}</p>
                        </div>
                      );
                    })}
                  </div>
                </>
              )}

              <p className="datasets-page__section-title">
                <ExternalLink size={15} strokeWidth={2.25} />
                <span>Source</span>
              </p>
              <div className="datasets-page__meta-list">
                <span className="datasets-page__meta-item">
                  <ExternalLink size={13} /> {selected.huggingface_dataset}
                </span>
              </div>
            </div>
          )}
        </div>
      )}
    </div>
  );
};

export default Datasets;

























//Datasets.scss
@use '../../../styles/variables' as *;

.datasets-page {
  display: flex;
  flex-direction: column;
  gap: 16px;
  // Caps the page at a comfortable working width and centers it, so on very
  // wide viewports (1800px+) the sidebar/detail columns don't stretch into an
  // unusably wide layout — the extra space becomes gutters instead.
  max-width: 1680px;
  margin-left: auto;
  margin-right: auto;
  // Same flex chain as History/Models/Providers: main-layout__body / workspace-layout /
  // workspace-layout__content already resolve to the exact viewport height,
  // so height: 100% fills it precisely. workspace-layout__content also carries
  // a 3rem bottom padding, which would leave a gap below this page — pull most
  // of that back in, keeping a small 0.75rem breathing-room strip at the bottom.
  height: calc(100% + 3rem - 0.75rem);
  margin-bottom: calc(-3rem + 0.75rem);
  min-height: 0;

  /* ---------- header ---------- */
  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding-bottom: 18px;
    margin-bottom: 2px;
    border-bottom: 1px solid $border-subtle;
  }

  &__header-left {
    display: flex;
    flex-direction: column;
  }

  &__header-right {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 10px;
  }

  &__header-eyebrow {
    display: flex;
    align-items: center;
    gap: 8px;
    font-family: $font-mono;
    font-size: 0.6875rem;
    font-weight: 700;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    color: $primary;
    margin-bottom: 6px;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $primary;
    }
  }

  &__header-meta {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-secondary;
    background: $bg-subtle;
    border: 1px solid $border-subtle;
    border-radius: 999px;
    padding: 7px 13px;
    white-space: nowrap;
  }

  &__title {
    font-size: 21px;
    font-weight: 800;
    letter-spacing: -0.02em;
    color: $text-primary;
  }

  &__subtitle {
    margin-top: 3px;
    color: $text-secondary;
    font-size: 0.84375rem;
  }

  &__btn {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    font-family: $font-body;
    font-size: 0.8125rem;
    font-weight: 600;
    padding: 9px 14px;
    border-radius: 8px;
    border: 1px solid transparent;
    cursor: pointer;
    white-space: nowrap;
    transition: background 0.14s ease, border-color 0.14s ease, color 0.14s ease, opacity 0.14s ease;

    &:disabled {
      opacity: 0.6;
      cursor: not-allowed;
    }

    &--outline {
      background: $bg-main;
      border-color: $border-default;
      color: $text-secondary;

      &:hover:not(:disabled) {
        border-color: $text-primary;
        color: $text-primary;
      }
    }

    &--primary {
      background: $primary;
      border-color: $primary;
      color: #fff;

      &:hover {
        background: $primary-hover;
        border-color: $primary-hover;
      }
    }
  }

  /* ---------- filters ---------- */
  &__filters {
    flex-shrink: 0;
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 12px;
  }

  &__search {
    display: flex;
    align-items: center;
    gap: 9px;
    width: 280px;
    max-width: 100%;
    border: 1px solid $border-default;
    border-radius: 10px;
    padding: 9px 12px;
    background: $bg-main;
    color: $text-tertiary;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &:focus-within {
      border-color: $primary;
      box-shadow: 0 0 0 3px $primary-light;
    }

    input {
      flex: 1;
      border: none;
      outline: none;
      font-size: 0.8125rem;
      color: $text-primary;
      background: transparent;
      font-family: $font-body;
      min-width: 0;

      &::placeholder {
        color: $text-tertiary;
      }
    }
  }

  &__search-clear {
    flex-shrink: 0;
    width: 18px;
    height: 18px;
    border-radius: 50%;
    border: none;
    background: $bg-inset;
    color: $text-tertiary;
    display: grid;
    place-items: center;
    cursor: pointer;
    transition: background 0.14s ease, color 0.14s ease;

    &:hover {
      background: $border-default;
      color: $text-primary;
    }
  }

  /* ---------- segmented category bar ---------- */
  &__seg {
    flex-shrink: 0;
    display: inline-flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 2px;
    padding: 3px;
    border: 1px solid $border-subtle;
    border-radius: 11px;
    background: $bg-subtle;
  }

  &__seg-item {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-family: $font-body;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-tertiary;
    background: transparent;
    border: none;
    border-radius: 8px;
    padding: 7px 12px;
    cursor: pointer;
    transition: background 0.14s ease, color 0.14s ease, box-shadow 0.14s ease;

    svg {
      opacity: 0.8;
    }

    &:hover {
      color: $text-primary;
    }

    &--active {
      background: $bg-main;
      color: $primary;
      box-shadow: $shadow-xs;

      svg {
        opacity: 1;
      }
    }
  }

  /* ---------- master-detail layout ---------- */
  &__layout {
    flex: 1;
    display: grid;
    grid-template-columns: 340px 1fr;
    gap: 16px;
    min-height: 0;
    overflow: hidden;
  }

  /* ---------- sidebar list ---------- */
  &__sidebar {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 16px;
    box-shadow: $shadow-xs;
    height: 100%;
    min-height: 0;
    display: flex;
    flex-direction: column;
    overflow: hidden;
  }

  &__sidebar-list {
    flex: 1;
    display: flex;
    flex-direction: column;
    min-height: 0;
    overflow-y: auto;
  }

  &__item {
    display: flex;
    flex-direction: column;
    gap: 6px;
    padding: 14px 16px;
    text-align: left;
    border: none;
    border-bottom: 1px solid $border-subtle;
    border-left: 3px solid transparent;
    background: $bg-main;
    width: 100%;
    cursor: pointer;
    transition: background 0.12s ease, border-color 0.12s ease;

    &:last-child {
      border-bottom: none;
    }

    &:hover {
      background: $bg-subtle;
    }

    &--active {
      background: $primary-light;
      border-left-color: $primary;

      .datasets-page__item-name {
        color: $primary;
      }
    }
  }

  &__item-top {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__item-name {
    font-size: 0.84375rem;
    font-weight: 600;
    color: $text-primary;
    line-height: 1.35;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__item-featured {
    flex-shrink: 0;
    display: inline-flex;
    color: $warning;
  }

  &__item-meta {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
    font-size: 0.75rem;
    color: $text-tertiary;
  }

  &__item-questions {
    flex-shrink: 0;
  }

  /* ---------- tags (shared by sidebar + detail) ---------- */
  &__tag {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 4px;
    font-size: 0.625rem;
    font-weight: 600;
    border-radius: 999px;
    padding: 2px 8px;

    &--blue {
      color: $primary;
      background: $primary-light;
    }

    &--violet {
      color: $violet;
      background: $violet-light;
    }

    &--amber {
      color: $warning;
      background: $warning-subtle;
    }

    &--jade {
      color: $success;
      background: $success-subtle;
    }

    &--rose {
      color: $danger;
      background: $danger-subtle;
    }

    &--featured {
      color: $warning;
      background: $warning-subtle;
    }
  }

  /* ---------- detail panel ---------- */
  &__detail {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 16px;
    box-shadow: $shadow-xs;
    padding: 26px 28px;
    height: 100%;
    min-height: 0;
    overflow-y: auto;
  }

  &__detail-head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
    padding-bottom: 18px;
    margin-bottom: 16px;
    border-bottom: 1px solid $border-subtle;
    position: sticky;
    top: -26px;
    padding-top: 26px;
    margin-top: -26px;
    background: $bg-main;
    z-index: 1;
  }

  &__detail-head-left {
    display: flex;
    flex-direction: column;
    gap: 8px;
    min-width: 0;
  }

  &__detail-tags {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;

    .datasets-page__tag {
      font-size: 0.6875rem;
      padding: 3px 10px;
    }
  }

  &__detail-name {
    font-size: 1.25rem;
    font-weight: 800;
    letter-spacing: -0.01em;
    line-height: 1.3;
    color: $text-primary;
  }

  &__detail-desc {
    font-size: 0.84375rem;
    line-height: 1.6;
    color: $text-secondary;
    margin-bottom: 24px;
  }

  /* ---------- section titles ---------- */
  &__section-title {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 0.8125rem;
    font-weight: 700;
    color: $text-primary;
    margin: 0 0 12px;

    svg {
      color: $primary;
      flex-shrink: 0;
    }

    &:not(:first-of-type) {
      margin-top: 24px;
    }
  }

  /* ---------- stat cards ---------- */
  &__stat-row {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 12px;
    margin-bottom: 24px;
  }

  &__stat-card {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-left: 3px solid $primary;
    border-radius: 10px;
    padding: 14px 16px;
    display: flex;
    align-items: center;
    gap: 12px;
    min-width: 0;
  }

  &__stat-icon {
    flex-shrink: 0;
    width: 34px;
    height: 34px;
    border-radius: 9px;
    background: $primary-light;
    color: $primary;
    display: grid;
    place-items: center;
  }

  &__stat-body {
    display: flex;
    flex-direction: column;
    gap: 2px;
    min-width: 0;
  }

  &__stat-card-label {
    font-size: 0.6875rem;
    font-weight: 600;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $text-tertiary;
  }

  &__stat-card-value {
    font-size: 1.125rem;
    font-weight: 800;
    color: $text-primary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;

    &--sm {
      font-size: 0.9375rem;
      font-weight: 700;
    }
  }

  /* ---------- meta list ---------- */
  &__meta-list {
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  &__meta-item {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 0.8125rem;
    color: $text-secondary;

    svg {
      flex-shrink: 0;
      color: $text-tertiary;
    }
  }

  /* ---------- empty state ---------- */
  /* ---------- loading — plain, no border, just centers the spinner ---------- */
  &__loading {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 64px 20px;
  }

  &__empty {
    flex-shrink: 0;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 10px;
    padding: 52px 20px;
    border: 1px dashed $border-strong;
    border-radius: 14px;
    color: $text-tertiary;
    font-size: 0.84375rem;

    svg {
      color: $text-tertiary;
    }

    &--error {
      border-style: solid;
      border-color: $danger-subtle;
      background: $danger-subtle;
      color: $danger;

      svg {
        color: $danger;
      }
    }
  }

  &__spin {
    animation: datasets-page-spin 0.9s linear infinite;
  }

  /* ---------- task list — top accent bar + ghost number ---------- */
  &__task-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(230px, 1fr));
    gap: 14px;
    margin-bottom: 24px;
  }

  &__task-card {
    position: relative;
    min-height: 138px;
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-top: 3px solid var(--task-tint, #{$primary});
    border-radius: 12px;
    padding: 18px 16px 16px;
    overflow: hidden;
    transition: box-shadow 0.15s ease, transform 0.15s ease;

    &:hover {
      box-shadow: $shadow-md;
      transform: translateY(-2px);
    }

    &--blue {
      --task-tint: #{$primary};
    }

    &--violet {
      --task-tint: #{$violet};
    }

    &--amber {
      --task-tint: #{$warning};
    }

    &--jade {
      --task-tint: #{$success};
    }
  }

  &__task-card-ghost {
    position: absolute;
    right: 8px;
    top: -6px;
    font-size: 3.5rem;
    font-weight: 800;
    color: var(--task-tint, #{$primary});
    opacity: 0.08;
    line-height: 1;
    pointer-events: none;
  }

  &__task-card-name {
    position: relative;
    font-size: 0.84375rem;
    font-weight: 700;
    color: $text-primary;
    margin-bottom: 8px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__task-card-value {
    position: relative;
    font-size: 0.78125rem;
    color: $text-secondary;
    line-height: 1.55;
    display: -webkit-box;
    -webkit-line-clamp: 3;
    -webkit-box-orient: vertical;
    overflow: hidden;
  }

  /* ---------- responsive ---------- */
  @media (max-width: 900px) {
    &__layout {
      grid-template-columns: 1fr;
      grid-template-rows: 16rem 1fr;
      overflow: visible;
    }

    &__sidebar,
    &__detail {
      height: auto;
      max-height: 22rem;
    }

    &__stat-row {
      grid-template-columns: repeat(2, 1fr);
    }
  }

  @media (max-width: 520px) {
    &__detail {
      padding: 18px 16px;
    }

    &__detail-head {
      flex-direction: column;
    }

    &__stat-row {
      grid-template-columns: 1fr;
    }
  }

  /* ---------- ultra-wide: nudge key text sizes up a touch ---------- */
  @media (min-width: 1800px) {
    &__title {
      font-size: 23px;
    }

    &__subtitle {
      font-size: 0.90625rem;
    }

    &__item-name {
      font-size: 0.90625rem;
    }

    &__detail-name {
      font-size: 1.375rem;
    }

    &__detail-desc {
      font-size: 0.90625rem;
    }

    &__stat-card-value {
      font-size: 1.375rem;
    }

    &__stat-card-value--sm {
      font-size: 1.0625rem;
    }

    &__task-card-name {
      font-size: 0.875rem;
    }

    &__task-card-value {
      font-size: 0.84375rem;
    }
  }
}

@keyframes datasets-page-spin {
  to {
    transform: rotate(360deg);
  }
}
