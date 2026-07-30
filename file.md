import { useMemo, useState, type FC } from 'react';
import { useNavigate } from 'react-router-dom';
import { Upload, Database, Star, Play } from 'lucide-react';
import { TEST_SUITES } from '../RunEvaluation/data';
import './Datasets.scss';

const CATEGORIES = ['All', 'Agents', 'Coding', 'General', 'RAG', 'Finance', 'Healthcare'];

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

      <div className="datasets-page__tabs">
        {CATEGORIES.map((c) => (
          <button
            key={c}
            type="button"
            className={`datasets-page__tab${category === c ? ' datasets-page__tab--active' : ''}`}
            onClick={() => setCategory(c)}
          >
            {c}
          </button>
        ))}
      </div>

      <div className="datasets-page__body">
        <div className="datasets-page__grid">
          {filtered.map((d) => (
            <div className="datasets-page__card" key={d.id}>
              {d.featured && (
                <span className="datasets-page__featured">
                  <Star size={11} strokeWidth={2.5} /> Featured
                </span>
              )}

              <span className="datasets-page__category-tag">{d.category}</span>
              <h3 className="datasets-page__name">{d.name}</h3>
              <p className="datasets-page__desc">{d.description}</p>

              <div className="datasets-page__details">
                <div className="datasets-page__detail-row">
                  <span className="datasets-page__detail-label">Questions</span>
                  <span className="datasets-page__detail-value n">{d.questions}</span>
                </div>
                <div className="datasets-page__detail-row">
                  <span className="datasets-page__detail-label">Language</span>
                  <span className="datasets-page__detail-value">{d.language}</span>
                </div>
                <div className="datasets-page__detail-row">
                  <span className="datasets-page__detail-label">Difficulty</span>
                  <span className="datasets-page__detail-value">{d.difficulty}</span>
                </div>
                <div className="datasets-page__detail-row">
                  <span className="datasets-page__detail-label">Version</span>
                  <span className="datasets-page__detail-value n">{d.version}</span>
                </div>
                <div className="datasets-page__detail-row">
                  <span className="datasets-page__detail-label">Maintainer</span>
                  <span className="datasets-page__detail-value">{d.maintainer}</span>
                </div>
              </div>

              <button type="button" className="datasets-page__use-btn" onClick={() => navigate('/app/run-evaluation')}>
                <Play size={13} strokeWidth={2.25} /> Use
              </button>
            </div>
          ))}

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

  /* ---------- category tabs ---------- */
  &__tabs {
    flex-shrink: 0;
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
  }

  &__tab {
    font-family: $font-body;
    font-size: 0.78125rem;
    font-weight: 500;
    color: $text-secondary;
    background: $bg-main;
    border: 1px solid $border-default;
    border-radius: 999px;
    padding: 6px 14px;
    cursor: pointer;
    transition: border-color 0.14s ease, color 0.14s ease, background 0.14s ease;

    &:hover {
      border-color: $primary;
      color: $primary;
    }

    &--active {
      background: $primary;
      border-color: $primary;
      color: #fff;
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
    grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
    gap: 14px;
  }

  &__card {
    position: relative;
    display: flex;
    flex-direction: column;
    gap: 8px;
    padding: 18px 20px;
    border: 1px solid $border-subtle;
    border-radius: 14px;
    background: $bg-main;
    box-shadow: $shadow-xs;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &:hover {
      border-color: $border-strong;
      box-shadow: $shadow-sm;
    }
  }

  &__featured {
    position: absolute;
    top: -1px;
    right: 16px;
    display: inline-flex;
    align-items: center;
    gap: 4px;
    font-size: 0.625rem;
    font-weight: 700;
    letter-spacing: 0.03em;
    color: $warning;
    background: $warning-subtle;
    border-radius: 0 0 6px 6px;
    padding: 3px 8px;
  }

  &__category-tag {
    align-self: flex-start;
    font-size: 0.6875rem;
    font-weight: 600;
    color: $primary;
    background: $primary-light;
    border-radius: 999px;
    padding: 3px 10px;
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
  }

  &__details {
    display: flex;
    flex-direction: column;
    gap: 7px;
    margin-top: 4px;
    padding-top: 13px;
    border-top: 1px solid $border-subtle;
  }

  &__detail-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
  }

  &__detail-label {
    font-size: 0.71875rem;
    color: $text-tertiary;
  }

  &__detail-value {
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-secondary;
    text-align: right;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    max-width: 60%;
  }

  &__use-btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 6px;
    margin-top: 6px;
    width: 100%;
    font-family: $font-body;
    font-size: 0.8125rem;
    font-weight: 600;
    padding: 9px 12px;
    border-radius: 8px;
    border: 1px solid $primary;
    background: $primary;
    color: #fff;
    cursor: pointer;
    transition: background 0.14s ease, border-color 0.14s ease;

    &:hover {
      background: $primary-hover;
      border-color: $primary-hover;
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
