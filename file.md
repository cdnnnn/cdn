import { useMemo, useState, type FC } from 'react';
import { useNavigate } from 'react-router-dom';
import { Search, X, Boxes, LayoutGrid, Wrench, Eye, BrainCircuit, Code2 } from 'lucide-react';
import { MODELS } from '../RunEvaluation/data';
import './Models.scss';

const CAPABILITY_FILTERS = [
  { value: 'All', icon: LayoutGrid },
  { value: 'Tool Calling', icon: Wrench },
  { value: 'Vision', icon: Eye },
  { value: 'Reasoning', icon: BrainCircuit },
  { value: 'Coding', icon: Code2 },
];

const Models: FC = () => {
  const navigate = useNavigate();
  const [query, setQuery] = useState('');
  const [capability, setCapability] = useState('All');

  const filtered = useMemo(() => {
    return MODELS.filter((m) => {
      if (query && !m.name.toLowerCase().includes(query.toLowerCase()) && !m.provider.toLowerCase().includes(query.toLowerCase())) {
        return false;
      }
      if (capability !== 'All') {
        const matches = m.capabilities.some((c) => c.toLowerCase().includes(capability.toLowerCase()));
        if (!matches) return false;
      }
      return true;
    });
  }, [query, capability]);

  return (
    <div className="models-page">
      <div className="models-page__header">
        <div className="models-page__header-left">
          <p className="models-page__header-eyebrow">Model catalog</p>
          <h1 className="models-page__title">Models</h1>
          <p className="models-page__subtitle">Browse available AI models across every connected provider</p>
        </div>

        <div className="models-page__header-meta">
          <Boxes size={13} />
          {MODELS.length} models available
        </div>
      </div>

      <div className="models-page__filters">
        <div className="models-page__search">
          <Search size={15} />
          <input type="text" placeholder="Search models..." value={query} onChange={(e) => setQuery(e.target.value)} />
          {query && (
            <button type="button" className="models-page__search-clear" onClick={() => setQuery('')} aria-label="Clear search">
              <X size={13} />
            </button>
          )}
        </div>

        <div className="models-page__seg">
          {CAPABILITY_FILTERS.map((c) => (
            <button
              key={c.value}
              type="button"
              className={`models-page__seg-item${capability === c.value ? ' models-page__seg-item--active' : ''}`}
              onClick={() => setCapability(c.value)}
            >
              <c.icon size={13} strokeWidth={2.25} />
              {c.value}
            </button>
          ))}
        </div>
      </div>

      <div className="models-page__grid">
        {filtered.map((m) => (
          <div className="models-page__card" key={m.id}>
            <div className="models-page__card-top">
              <span className="models-page__provider-badge">{m.provider}</span>
              <span className="models-page__version">{m.version}</span>
            </div>

            <h3 className="models-page__name">{m.name}</h3>
            <p className="models-page__desc">{m.description}</p>

            <div className="models-page__caps">
              {m.capabilities.map((c) => (
                <span key={c} className="models-page__cap-pill">
                  {c}
                </span>
              ))}
            </div>

            <div className="models-page__specs">
              <div className="models-page__spec">
                <span className="models-page__spec-label">Context</span>
                <span className="models-page__spec-value n">{m.contextWindow}</span>
              </div>
              <div className="models-page__spec">
                <span className="models-page__spec-label">Price</span>
                <span className="models-page__spec-value n">{m.pricing}</span>
              </div>
              <div className="models-page__spec">
                <span className="models-page__spec-label">Speed</span>
                <span className="models-page__spec-value n">{m.speedRating}</span>
              </div>
              <div className="models-page__spec">
                <span className="models-page__spec-label">Accuracy</span>
                <span className="models-page__spec-value models-page__spec-value--highlight n">{m.accuracyScore}%</span>
              </div>
            </div>

            <div className="models-page__card-actions">
              <button type="button" className="models-page__btn models-page__btn--outline">
                Details
              </button>
              <button type="button" className="models-page__btn models-page__btn--primary" onClick={() => navigate('/app/run-evaluation')}>
                Test
              </button>
            </div>
          </div>
        ))}

        {filtered.length === 0 && (
          <div className="models-page__empty">
            <Search size={22} />
            <p>No models match your filters.</p>
          </div>
        )}
      </div>
    </div>
  );
};

export default Models;

























@use '../../../styles/variables' as *;

.models-page {
  display: flex;
  flex-direction: column;
  gap: 18px;

  /* ---------- header ---------- */
  &__header {
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
    flex-shrink: 0;
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
    margin-bottom: 3px;
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

  /* ---------- filters ---------- */
  &__filters {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
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

  /* ---------- card grid ---------- */
  &__grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
    gap: 14px;
  }

  &__card {
    display: flex;
    flex-direction: column;
    gap: 12px;
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

  &__card-top {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__provider-badge {
    font-size: 0.6875rem;
    font-weight: 600;
    color: $primary;
    background: $primary-light;
    border-radius: 999px;
    padding: 3px 10px;
  }

  &__version {
    font-family: $font-mono;
    font-size: 0.6875rem;
    color: $text-tertiary;
  }

  &__name {
    font-size: 0.9375rem;
    font-weight: 700;
    letter-spacing: -0.01em;
    color: $text-primary;
  }

  &__desc {
    margin-top: -6px;
    font-size: 0.8125rem;
    line-height: 1.5;
    color: $text-secondary;
  }

  &__caps {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
  }

  &__cap-pill {
    font-size: 0.71875rem;
    color: $text-secondary;
    background: $bg-subtle;
    border: 1px solid $border-subtle;
    border-radius: 6px;
    padding: 3px 8px;
  }

  &__specs {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 8px;
    margin-top: 2px;
    padding-top: 13px;
    border-top: 1px solid $border-subtle;
  }

  &__spec {
    display: flex;
    flex-direction: column;
    gap: 4px;
    min-width: 0;
  }

  &__spec-label {
    font-size: 0.59375rem;
    font-weight: 600;
    letter-spacing: 0.05em;
    text-transform: uppercase;
    color: $text-tertiary;
  }

  &__spec-value {
    font-size: 0.71875rem;
    font-weight: 600;
    color: $text-secondary;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;

    &--highlight {
      color: $success;
      font-weight: 700;
    }
  }

  &__card-actions {
    display: flex;
    justify-content: flex-end;
    gap: 6px;
    margin-top: 2px;
  }

  &__btn {
    text-align: center;
    font-family: $font-body;
    font-size: 0.71875rem;
    font-weight: 600;
    padding: 5px 12px;
    border-radius: 7px;
    border: 1px solid transparent;
    cursor: pointer;
    transition: background 0.14s ease, border-color 0.14s ease, color 0.14s ease;

    &--outline {
      background: $bg-main;
      border-color: $border-default;
      color: $text-primary;

      &:hover {
        border-color: $text-primary;
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

  /* ---------- responsive ---------- */
  @media (max-width: 480px) {
    &__specs {
      grid-template-columns: repeat(2, 1fr);
    }
  }
}
