import { useMemo, useState, type FC } from 'react';
import { useNavigate } from 'react-router-dom';
import {
  Upload,
  Database,
  Star,
  LayoutGrid,
  Bot,
  Code2,
  BookOpen,
  Search,
  DollarSign,
  Globe,
  User,
  X,
  ListChecks,
} from 'lucide-react';
import { TEST_SUITES } from '../RunEvaluation/data';
import './Datasets.scss';

const CATEGORIES = [
  { value: 'All', icon: LayoutGrid },
  { value: 'Agents', icon: Bot },
  { value: 'Coding', icon: Code2 },
  { value: 'General', icon: BookOpen },
  { value: 'RAG', icon: Search },
  { value: 'Finance', icon: DollarSign },
] as const;

const CATEGORY_TINTS: Record<string, 'blue' | 'violet' | 'amber' | 'jade' | 'rose'> = {
  Agents: 'violet',
  Coding: 'blue',
  General: 'jade',
  RAG: 'blue',
  Finance: 'amber',
};

const DIFFICULTY_TINTS: Record<string, 'blue' | 'violet' | 'amber' | 'jade' | 'rose'> = {
  Medium: 'jade',
  High: 'blue',
  Advanced: 'violet',
  Expert: 'rose',
};

const Datasets: FC = () => {
  const navigate = useNavigate();
  const [query, setQuery] = useState('');
  const [category, setCategory] = useState('All');
  const [activeId, setActiveId] = useState(TEST_SUITES[0]?.id ?? null);

  const filtered = useMemo(
    () =>
      TEST_SUITES.filter((d) => {
        if (category !== 'All' && d.category !== category) return false;
        if (query && !d.name.toLowerCase().includes(query.toLowerCase()) && !d.description.toLowerCase().includes(query.toLowerCase())) {
          return false;
        }
        return true;
      }),
    [query, category]
  );

  const active = useMemo(
    () => filtered.find((d) => d.id === activeId) ?? filtered[0] ?? null,
    [activeId, filtered]
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
            {TEST_SUITES.length} suites available
          </div>
          <button type="button" className="datasets-page__btn datasets-page__btn--primary" onClick={() => navigate('/app/run-evaluation')}>
            <Upload size={14} strokeWidth={2.25} /> Upload
          </button>
        </div>
      </div>

      <div className="datasets-page__toolbar">
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
          {CATEGORIES.map((c) => (
            <button
              key={c.value}
              type="button"
              className={`datasets-page__seg-item${category === c.value ? ' datasets-page__seg-item--active' : ''}`}
              onClick={() => setCategory(c.value)}
            >
              <c.icon size={13} strokeWidth={2.25} />
              {c.value}
            </button>
          ))}
        </div>
      </div>

      <div className="datasets-page__split">
        <aside className="datasets-page__list">
          {filtered.map((d) => {
            const isActive = active?.id === d.id;
            const catTint = CATEGORY_TINTS[d.category] ?? 'blue';
            return (
              <button
                key={d.id}
                type="button"
                className={`datasets-page__row${isActive ? ' datasets-page__row--active' : ''}`}
                onClick={() => setActiveId(d.id)}
              >
                <span className="datasets-page__row-name">{d.name}</span>
                <span className="datasets-page__row-meta">
                  <span className={`datasets-page__tag datasets-page__tag--${catTint}`}>{d.category}</span>
                  <span className="datasets-page__row-count n">{d.questions.toLocaleString()} q</span>
                </span>
              </button>
            );
          })}

          {filtered.length === 0 && (
            <div className="datasets-page__empty datasets-page__empty--list">
              <Database size={20} />
              <p>No test suites in this category.</p>
            </div>
          )}
        </aside>

        <section className="datasets-page__detail">
          {active ? (
            <>
              {(() => {
                const catIcon = CATEGORIES.find((c) => c.value === active.category)?.icon ?? Database;
                const catTint = CATEGORY_TINTS[active.category] ?? 'blue';
                const diffTint = DIFFICULTY_TINTS[active.difficulty] ?? 'blue';
                const CatIcon = catIcon;

                return (
                  <>
                    <div className="datasets-page__detail-head">
                      <div>
                        <h2 className="datasets-page__name">{active.name}</h2>
                        <p className="datasets-page__desc">{active.description}</p>
                      </div>
                    </div>

                    <div className="datasets-page__tags">
                      <span className={`datasets-page__tag datasets-page__tag--${catTint}`}>
                        <CatIcon size={11} strokeWidth={2.5} />
                        {active.category}
                      </span>
                      <span className={`datasets-page__tag datasets-page__tag--${diffTint}`}>{active.difficulty}</span>
                      {active.featured && (
                        <span className="datasets-page__tag datasets-page__tag--featured">
                          <Star size={11} strokeWidth={2.5} />
                          Featured
                        </span>
                      )}
                    </div>

                    <div className="datasets-page__stats">
                      <div className="datasets-page__stat">
                        <span className="datasets-page__stat-value n">{active.questions.toLocaleString()}</span>
                        <span className="datasets-page__stat-label">Questions</span>
                      </div>
                      <div className="datasets-page__stat">
                        <span className="datasets-page__stat-value n">{active.subgroups.length}</span>
                        <span className="datasets-page__stat-label">Subcategories</span>
                      </div>
                      <div className="datasets-page__stat">
                        <span className="datasets-page__stat-value">{active.version}</span>
                        <span className="datasets-page__stat-label">Version</span>
                      </div>
                    </div>

                    <div className="datasets-page__meta-row">
                      <span className="datasets-page__meta-item">
                        <Globe size={12} /> {active.language}
                      </span>
                      <span className="datasets-page__meta-item">
                        <User size={12} /> {active.maintainer}
                      </span>
                      <span className="datasets-page__meta-item">
                        <ListChecks size={12} /> {active.task}
                      </span>
                    </div>

                    {/* ---------- question preview ---------- */}
                    <div className="datasets-page__group-head">
                      <h3>Question breakdown</h3>
                      <div className="datasets-page__group-line" />
                    </div>

                    <div className="datasets-page__preview-list">
                      {active.subgroups.map((s) => {
                        const pct = Math.round((s.count / active.questions) * 100);
                        return (
                          <div className="datasets-page__preview-item" key={s.id}>
                            <div className="datasets-page__preview-item-top">
                              <span className="datasets-page__preview-item-name">{s.name}</span>
                              <span className="datasets-page__preview-item-count n">{s.count.toLocaleString()} questions</span>
                            </div>
                            <div className="datasets-page__preview-bar-wrap">
                              <div className="datasets-page__preview-bar" style={{ width: `${pct}%` }} />
                            </div>
                          </div>
                        );
                      })}
                    </div>
                  </>
                );
              })()}
            </>
          ) : (
            <p className="datasets-page__empty">Select a test suite to preview its questions.</p>
          )}
        </section>
      </div>
    </div>
  );
};

export default Datasets;






















@use '../../../styles/variables' as *;

.datasets-page {
  display: flex;
  flex-direction: column;
  height: calc(100vh - 166px);
  min-height: 0;
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
    transition: background 0.14s ease, border-color 0.14s ease;

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

  /* ---------- toolbar: search + category segment ---------- */
  &__toolbar {
    flex-shrink: 0;
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
  }

  &__search {
    display: flex;
    align-items: center;
    gap: 9px;
    width: 300px;
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

  /* ---------- master-detail split ---------- */
  &__split {
    flex: 1;
    min-height: 0;
    display: grid;
    grid-template-columns: 320px 1fr;
    border: 1px solid $border-default;
    border-radius: 14px;
    overflow: hidden;
    background: $bg-main;
  }

  /* ---------- left list ---------- */
  &__list {
    border-right: 1px solid $border-default;
    background: $bg-subtle;
    overflow-y: auto;
  }

  &__row {
    width: 100%;
    display: flex;
    flex-direction: column;
    gap: 6px;
    text-align: left;
    padding: 14px 16px;
    border: none;
    border-bottom: 1px solid $border-subtle;
    background: transparent;
    cursor: pointer;
    transition: background 0.14s ease;

    &:hover {
      background: $bg-inset;
    }

    &--active {
      background: $bg-main;
      border-left: 3px solid $primary;
      padding-left: 13px;

      &:hover {
        background: $bg-main;
      }
    }
  }

  &__row-name {
    font-size: 0.8125rem;
    font-weight: 700;
    letter-spacing: -0.01em;
    line-height: 1.4;
    color: $text-primary;
  }

  &__row-meta {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__row-count {
    font-size: 0.6875rem;
    font-weight: 600;
    color: $text-tertiary;
    flex-shrink: 0;
  }

  /* ---------- right detail ---------- */
  &__detail {
    overflow-y: auto;
    padding: 28px 32px;
  }

  &__detail-head {
    margin-bottom: 14px;
  }

  &__name {
    margin: 0 0 6px;
    font-size: 1.1875rem;
    font-weight: 800;
    letter-spacing: -0.015em;
    line-height: 1.35;
    color: $text-primary;
  }

  &__desc {
    margin: 0;
    font-size: 0.875rem;
    line-height: 1.6;
    color: $text-secondary;
    max-width: 720px;
  }

  &__tags {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    margin-bottom: 22px;
  }

  &__tag {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    font-size: 0.6875rem;
    font-weight: 600;
    border-radius: 999px;
    padding: 3px 10px;

    &--blue {
      color: $primary;
      background: $primary-light;
    }

    &--violet {
      color: #7c3aed;
      background: #f3e8ff;
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

  /* ---------- stats ---------- */
  &__stats {
    display: flex;
    border: 1px solid $border-default;
    border-radius: 10px;
    overflow: hidden;
    margin-bottom: 18px;
  }

  &__stat {
    flex: 1;
    padding: 12px 16px;
    border-right: 1px solid $border-subtle;
    display: flex;
    flex-direction: column;
    gap: 2px;

    &:last-child {
      border-right: none;
    }
  }

  &__stat-value {
    font-size: 1.25rem;
    font-weight: 800;
    letter-spacing: -0.02em;
    color: $text-primary;
    line-height: 1;
  }

  &__stat-label {
    font-size: 0.625rem;
    font-weight: 600;
    letter-spacing: 0.05em;
    text-transform: uppercase;
    color: $text-tertiary;
    margin-top: 2px;
  }

  /* ---------- meta row ---------- */
  &__meta-row {
    display: flex;
    flex-wrap: wrap;
    gap: 16px;
    margin-bottom: 26px;
  }

  &__meta-item {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 0.75rem;
    color: $text-secondary;

    svg {
      flex-shrink: 0;
      color: $text-tertiary;
    }
  }

  /* ---------- question breakdown / preview ---------- */
  &__group-head {
    display: flex;
    align-items: center;
    gap: 10px;
    margin: 0 0 14px;

    h3 {
      margin: 0;
      font-family: $font-mono;
      font-size: 0.6875rem;
      font-weight: 600;
      text-transform: uppercase;
      letter-spacing: 0.08em;
      color: $text-tertiary;
      white-space: nowrap;
    }
  }

  &__group-line {
    flex: 1;
    height: 1px;
    background: $border-default;
  }

  &__preview-list {
    display: flex;
    flex-direction: column;
    gap: 14px;
    max-width: 560px;
  }

  &__preview-item {
    display: flex;
    flex-direction: column;
    gap: 6px;
  }

  &__preview-item-top {
    display: flex;
    align-items: baseline;
    justify-content: space-between;
    gap: 10px;
  }

  &__preview-item-name {
    font-size: 0.8125rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__preview-item-count {
    font-size: 0.71875rem;
    font-weight: 600;
    color: $text-tertiary;
    flex-shrink: 0;
  }

  &__preview-bar-wrap {
    height: 6px;
    border-radius: 4px;
    background: $bg-inset;
    overflow: hidden;
  }

  &__preview-bar {
    height: 100%;
    border-radius: 4px;
    background: $primary;
  }

  /* ---------- empty state ---------- */
  &__empty {
    padding: 52px 20px;
    text-align: center;
    color: $text-tertiary;
    font-size: 0.84375rem;

    &--list {
      display: flex;
      flex-direction: column;
      align-items: center;
      gap: 10px;

      svg {
        color: $text-tertiary;
      }
    }
  }

  /* ---------- responsive ---------- */
  @media (max-width: 800px) {
    &__split {
      grid-template-columns: 1fr;
    }

    &__list {
      display: none;
    }
  }
}
