import { useEffect, useMemo, useState, type FC } from 'react';
import { useNavigate } from 'react-router-dom';
import { Database, Search, X, LayoutGrid, Tag, RefreshCw, AlertCircle, ExternalLink, Play } from 'lucide-react';
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

const Datasets: FC = () => {
  const navigate = useNavigate();
  const [benchmarks, setBenchmarks] = useState<Benchmark[]>([]);
  const [total, setTotal] = useState(0);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const [query, setQuery] = useState('');
  const [type, setType] = useState('All');
  const [tasksModalFor, setTasksModalFor] = useState<Benchmark | null>(null);

  const load = () => {
    setLoading(true);
    setError(null);
    fetchBenchmarks()
      .then((res) => {
        setBenchmarks(res.benchmarks);
        setTotal(res.total);
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
        <div className="datasets-page__grid">
          {filtered.map((b) => (
            <div className="datasets-page__card" key={b.name}>
              <div className="datasets-page__card-top">
                <span className="datasets-page__card-name">{b.name}</span>
                <span className="datasets-page__tag datasets-page__tag--blue">{b.type}</span>
              </div>

              <p className="datasets-page__card-desc">{b.description}</p>

              <div className="datasets-page__card-stats">
                <div className="datasets-page__card-stat">
                  <span className="datasets-page__card-stat-label">Tasks</span>
                  <span className="datasets-page__card-stat-value n">{b.task_count}</span>
                </div>
                <div className="datasets-page__card-stat">
                  <span className="datasets-page__card-stat-label">Capabilities</span>
                  <span className="datasets-page__card-stat-value n">{b.required_capabilities.length}</span>
                </div>
                <div className="datasets-page__card-stat">
                  <span className="datasets-page__card-stat-label">Dataset</span>
                  <span className="datasets-page__card-stat-value datasets-page__card-stat-value--sm">{b.huggingface_dataset}</span>
                </div>
              </div>

              {b.required_capabilities.length > 0 && (
                <>
                  <span className="datasets-page__card-section-label">Required capabilities</span>
                  <div className="datasets-page__caps">
                    {b.required_capabilities.map((c) => (
                      <span key={c} className={`datasets-page__cap-pill datasets-page__cap-pill--${capabilityTint(c)}`}>
                        {c}
                      </span>
                    ))}
                  </div>
                </>
              )}

              {b.tasks.length > 0 && (
                <>
                  <span className="datasets-page__card-section-label">Sample tasks</span>
                  <div className="datasets-page__card-tasks">
                    {b.tasks.slice(0, 5).map((t) => (
                      <p className="datasets-page__card-task" key={t.name}>
                        <b>{t.name}:</b> <span>{t.value}</span>
                      </p>
                    ))}
                  </div>
                  {b.tasks.length > 5 && (
                    <button type="button" className="datasets-page__card-view-all" onClick={() => setTasksModalFor(b)}>
                      View all {b.tasks.length} tasks
                    </button>
                  )}
                </>
              )}

              <div className="datasets-page__card-foot">
                <span className="datasets-page__card-foot-source">
                  <ExternalLink size={12} /> {b.huggingface_dataset}
                </span>
                <button type="button" className="datasets-page__card-use" onClick={() => navigate('/app/run-evaluation')}>
                  <Play size={12} strokeWidth={2.25} /> Use in Evaluation
                </button>
              </div>
            </div>
          ))}
        </div>
      )}

      {tasksModalFor && (
        <div className="datasets-page__overlay" onClick={() => setTasksModalFor(null)}>
          <div className="datasets-page__modal" onClick={(e) => e.stopPropagation()}>
            <div className="datasets-page__modal-head">
              <div>
                <span className="datasets-page__tag datasets-page__tag--blue">{tasksModalFor.type}</span>
                <h2 className="datasets-page__modal-title">{tasksModalFor.name}</h2>
                <p className="datasets-page__modal-sub">All {tasksModalFor.tasks.length} tasks</p>
              </div>
              <button type="button" className="datasets-page__modal-close" onClick={() => setTasksModalFor(null)} aria-label="Close">
                <X size={16} />
              </button>
            </div>

            <div className="datasets-page__modal-body">
              {tasksModalFor.tasks.map((t) => (
                <p className="datasets-page__card-task" key={t.name}>
                  <b>{t.name}:</b> <span>{t.value}</span>
                </p>
              ))}
            </div>
          </div>
        </div>
      )}
    </div>
  );
};

export default Datasets;




























@use '../../../styles/variables' as *;

.datasets-page {
  display: flex;
  flex-direction: column;
  gap: 16px;

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
      color: $on-primary;

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

  /* ---------- tags ---------- */
  &__tag {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 4px;
    font-size: 0.6875rem;
    font-weight: 700;
    border-radius: 999px;
    padding: 3px 10px;

    &--blue {
      color: $primary;
      background: $primary-light;
    }
  }

  /* ---------- capability pills (shared) ---------- */
  &__caps {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    margin-bottom: 16px;
  }

  &__cap-pill {
    font-size: 0.71875rem;
    font-weight: 600;
    border-radius: 999px;
    padding: 3px 10px;

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
  }

  /* ---------- full-info card grid ---------- */
  &__grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(360px, 1fr));
    gap: 16px;
  }

  &__card {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-left: 3px solid $primary;
    border-radius: 12px;
    padding: 18px 20px;
    box-shadow: $shadow-xs;
    transition: box-shadow 0.15s ease, transform 0.15s ease;

    &:hover {
      box-shadow: $shadow-md;
      transform: translateY(-2px);
    }
  }

  &__card-top {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 10px;
    margin-bottom: 8px;
  }

  &__card-name {
    font-size: 0.9375rem;
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $text-primary;
  }

  &__card-desc {
    font-size: 0.78125rem;
    color: $text-secondary;
    line-height: 1.55;
    margin-bottom: 14px;
  }

  &__card-stats {
    display: flex;
    gap: 20px;
    margin-bottom: 14px;
    padding-bottom: 14px;
    border-bottom: 1px solid $border-subtle;
  }

  &__card-stat-label {
    font-size: 0.625rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.04em;
    color: $text-tertiary;
  }

  &__card-stat-value {
    font-size: 0.9375rem;
    font-weight: 800;
    color: $text-primary;
    display: block;
    margin-top: 2px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;

    &--sm {
      font-size: 0.78125rem;
      font-weight: 700;
    }
  }

  &__card-section-label {
    font-size: 0.65625rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    color: $text-tertiary;
    margin-bottom: 8px;
    display: block;
  }

  &__card-tasks {
    display: flex;
    flex-direction: column;
    gap: 6px;
    margin-bottom: 10px;
  }

  &__card-task {
    font-size: 0.75rem;
    line-height: 1.55;

    b {
      color: $text-primary;
      font-weight: 700;
    }

    span {
      color: $text-secondary;
    }
  }

  &__card-view-all {
    display: inline-flex;
    align-items: center;
    font-family: $font-body;
    font-size: 0.71875rem;
    font-weight: 700;
    color: $primary;
    background: transparent;
    border: none;
    padding: 0;
    margin-bottom: 14px;
    cursor: pointer;

    &:hover {
      text-decoration: underline;
    }
  }

  &__card-foot {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
    padding-top: 12px;
    border-top: 1px solid $border-subtle;
  }

  &__card-foot-source {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    font-size: 0.6875rem;
    color: $text-tertiary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;

    svg {
      flex-shrink: 0;
    }
  }

  &__card-use {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 5px;
    font-family: $font-body;
    font-size: 0.71875rem;
    font-weight: 700;
    color: $primary;
    background: $primary-light;
    border: 1px solid transparent;
    border-radius: 999px;
    padding: 5px 11px;
    cursor: pointer;
    transition: background 0.14s ease, color 0.14s ease;

    &:hover {
      background: $primary;
      color: $on-primary;
    }
  }

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

  /* ---------- tasks modal ---------- */
  &__overlay {
    position: fixed;
    inset: 0;
    z-index: 200;
    background: rgba(0, 0, 0, 0.45);
    backdrop-filter: blur(2px);
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 24px;
  }

  &__modal {
    width: 100%;
    max-width: 32rem;
    max-height: min(80vh, 40rem);
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 14px;
    box-shadow: $shadow-lg;
    display: flex;
    flex-direction: column;
    overflow: hidden;
  }

  &__modal-head {
    flex-shrink: 0;
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
    padding: 20px 22px 16px;
    border-bottom: 1px solid $border-subtle;

    .datasets-page__tag {
      margin-bottom: 8px;
    }
  }

  &__modal-title {
    font-size: 1.0625rem;
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $text-primary;
  }

  &__modal-sub {
    margin-top: 3px;
    font-size: 0.75rem;
    color: $text-tertiary;
  }

  &__modal-close {
    flex-shrink: 0;
    width: 30px;
    height: 30px;
    border-radius: 8px;
    border: 1px solid $border-default;
    background: $bg-main;
    color: $text-tertiary;
    display: grid;
    place-items: center;
    cursor: pointer;
    transition: border-color 0.14s ease, color 0.14s ease;

    &:hover {
      border-color: $text-primary;
      color: $text-primary;
    }
  }

  &__modal-body {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding: 18px 22px 22px;
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  /* ---------- responsive ---------- */
  @media (max-width: 640px) {
    &__grid {
      grid-template-columns: 1fr;
    }

    &__card-stats {
      flex-wrap: wrap;
      gap: 14px;
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

    &__card-name {
      font-size: 1.03125rem;
    }

    &__card-desc {
      font-size: 0.84375rem;
    }

    &__card-stat-value {
      font-size: 1.03125rem;
    }

    &__card-stat-value--sm {
      font-size: 0.84375rem;
    }

    &__card-task {
      font-size: 0.8125rem;
    }
  }
}

@keyframes datasets-page-spin {
  to {
    transform: rotate(360deg);
  }
}
