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
  HeartPulse,
  Globe,
  User,
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
  { value: 'Healthcare', icon: HeartPulse },
] as const;

const CATEGORY_TINTS: Record<string, 'blue' | 'violet' | 'amber' | 'jade' | 'rose'> = {
  Agents: 'violet',
  Coding: 'blue',
  General: 'jade',
  RAG: 'blue',
  Finance: 'amber',
  Healthcare: 'rose',
};

const DIFFICULTY_TINTS: Record<string, 'blue' | 'violet' | 'amber' | 'jade' | 'rose'> = {
  Medium: 'jade',
  High: 'blue',
  Advanced: 'violet',
  Expert: 'rose',
};

const Datasets: FC = () => {
  const navigate = useNavigate();
  const [category, setCategory] = useState('All');

  const filtered = useMemo(
    () => TEST_SUITES.filter((d) => category === 'All' || d.category === category),
    [category]
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

      <div className="datasets-page__body">
        <div className="datasets-page__grid">
          {filtered.map((d) => {
            const catIcon = CATEGORIES.find((c) => c.value === d.category)?.icon ?? Database;
            const catTint = CATEGORY_TINTS[d.category] ?? 'blue';
            const diffTint = DIFFICULTY_TINTS[d.difficulty] ?? 'blue';
            const CatIcon = catIcon;

            return (
              <div className="datasets-page__card" key={d.id}>
                <h3 className="datasets-page__name">{d.name}</h3>
                <p className="datasets-page__desc">{d.description}</p>

                <div className="datasets-page__tags">
                  <span className={`datasets-page__tag datasets-page__tag--${catTint}`}>
                    <CatIcon size={11} strokeWidth={2.5} />
                    {d.category}
                  </span>
                  <span className={`datasets-page__tag datasets-page__tag--${diffTint}`}>{d.difficulty}</span>
                  {d.featured && (
                    <span className="datasets-page__tag datasets-page__tag--featured">
                      <Star size={11} strokeWidth={2.5} />
                      Featured
                    </span>
                  )}
                </div>

                <div className="datasets-page__stat-row">
                  <div className="datasets-page__stat">
                    <span className="datasets-page__stat-value n">{d.questions.toLocaleString()}</span>
                    <span className="datasets-page__stat-label">Questions</span>
                  </div>
                  <div className="datasets-page__meta-list">
                    <span className="datasets-page__meta-item">
                      <Globe size={12} /> {d.language}
                    </span>
                    <span className="datasets-page__meta-item">
                      <User size={12} /> {d.maintainer}
                    </span>
                    <span className="datasets-page__meta-item n">{d.version}</span>
                  </div>
                </div>
              </div>
            );
          })}

          {filtered.length === 0 && (
            <div className="datasets-page__empty">
              <Database size={22} />
              <p>No test suites in this category.</p>
            </div>
          )}
        </div>
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

  /* ---------- segmented category bar ---------- */
  &__seg {
    flex-shrink: 0;
    display: inline-flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 2px;
    align-self: flex-start;
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

  /* ---------- scrollable body ---------- */
  &__body {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding-right: 4px;
    margin-right: -4px;
  }

  /* ---------- card grid ---------- */
  &__grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(310px, 1fr));
    gap: 14px;
  }

  &__card {
    position: relative;
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 18px 20px;
    border: 1px solid $border-subtle;
    border-radius: 14px;
    background: $bg-main;
    box-shadow: $shadow-xs;
    transition: border-color 0.14s ease, box-shadow 0.14s ease, transform 0.14s ease;

    &:hover {
      border-color: $border-strong;
      box-shadow: $shadow-sm;
      transform: translateY(-1px);
    }
  }

  &__name {
    font-size: 0.9375rem;
    font-weight: 700;
    letter-spacing: -0.01em;
    line-height: 1.35;
    color: $text-primary;
  }

  &__desc {
    margin-top: -4px;
    font-size: 0.8125rem;
    line-height: 1.5;
    color: $text-secondary;
    display: -webkit-box;
    -webkit-line-clamp: 2;
    -webkit-box-orient: vertical;
    overflow: hidden;
  }

  &__tags {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
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

  /* ---------- stat row ---------- */
  &__stat-row {
    display: flex;
    align-items: center;
    gap: 14px;
    margin-top: 4px;
    padding-top: 13px;
    border-top: 1px solid $border-subtle;
  }

  &__stat {
    flex-shrink: 0;
    display: flex;
    flex-direction: column;
    gap: 2px;
    padding-right: 14px;
    border-right: 1px solid $border-subtle;
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
  }

  &__meta-list {
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 4px;
  }

  &__meta-item {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 0.71875rem;
    color: $text-secondary;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;

    svg {
      flex-shrink: 0;
      color: $text-tertiary;
    }
  }

  /* ---------- empty state ---------- */
  &__empty {
    grid-column: 1 / -1;
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
  }
}
