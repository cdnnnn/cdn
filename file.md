import { useMemo, useState, type FC } from 'react';
import { Search, X, Boxes, LayoutGrid, Wrench, Eye, BrainCircuit, Code2, Info } from 'lucide-react';
import { MODELS } from '../RunEvaluation/data';
import type { ModelInfo } from '../RunEvaluation/types';
import './Models.scss';

const CAPABILITY_FILTERS = [
  { value: 'All', icon: LayoutGrid },
  { value: 'Tool Calling', icon: Wrench },
  { value: 'Vision', icon: Eye },
  { value: 'Reasoning', icon: BrainCircuit },
  { value: 'Coding', icon: Code2 },
];

const PILL_TINTS = ['blue', 'violet', 'amber', 'jade', 'rose'] as const;

function pillTint(capability: string) {
  let hash = 0;
  for (let i = 0; i < capability.length; i += 1) hash = (hash * 31 + capability.charCodeAt(i)) >>> 0;
  return PILL_TINTS[hash % PILL_TINTS.length];
}

const Models: FC = () => {
  const [query, setQuery] = useState('');
  const [capability, setCapability] = useState('All');
  const [detailModel, setDetailModel] = useState<ModelInfo | null>(null);

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

      <div className="models-page__body">
        <div className="models-page__grid">
          {filtered.map((m) => (
            <div className="models-page__card" key={m.id}>
              <div className="models-page__card-top">
                <span className="models-page__provider-badge">{m.provider}</span>
                <span className="models-page__card-top-right">
                  <span className="models-page__version">{m.version}</span>
                  <button
                    type="button"
                    className="models-page__info-btn"
                    title="View details"
                    aria-label="View details"
                    onClick={() => setDetailModel(m)}
                  >
                    <Info size={14} strokeWidth={2} />
                  </button>
                </span>
              </div>

              <h3 className="models-page__name">{m.name}</h3>
              <p className="models-page__desc">{m.description}</p>

              <div className="models-page__caps">
                {m.capabilities.map((c) => (
                  <span key={c} className={`models-page__cap-pill models-page__cap-pill--${pillTint(c)}`}>
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

      {detailModel && (
        <div className="models-page__overlay" onClick={() => setDetailModel(null)}>
          <div className="models-page__modal" onClick={(e) => e.stopPropagation()}>
            <div className="models-page__modal-head">
              <div>
                <span className="models-page__provider-badge">{detailModel.provider}</span>
                <h2 className="models-page__modal-title">{detailModel.name}</h2>
              </div>
              <button type="button" className="models-page__modal-close" onClick={() => setDetailModel(null)} aria-label="Close">
                <X size={16} />
              </button>
            </div>

            <p className="models-page__modal-desc">{detailModel.description}</p>

            <div className="models-page__modal-specs">
              <div className="models-page__modal-spec">
                <span className="models-page__spec-label">Version</span>
                <span className="models-page__spec-value n">{detailModel.version}</span>
              </div>
              <div className="models-page__modal-spec">
                <span className="models-page__spec-label">Context Window</span>
                <span className="models-page__spec-value n">{detailModel.contextWindow}</span>
              </div>
              <div className="models-page__modal-spec">
                <span className="models-page__spec-label">Pricing</span>
                <span className="models-page__spec-value n">{detailModel.pricing}</span>
              </div>
              <div className="models-page__modal-spec">
                <span className="models-page__spec-label">Speed</span>
                <span className="models-page__spec-value n">{detailModel.speedRating}</span>
              </div>
              <div className="models-page__modal-spec">
                <span className="models-page__spec-label">Accuracy Score</span>
                <span className="models-page__spec-value models-page__spec-value--highlight n">{detailModel.accuracyScore}%</span>
              </div>
              <div className="models-page__modal-spec">
                <span className="models-page__spec-label">Agent Score</span>
                <span className="models-page__spec-value models-page__spec-value--highlight n">{detailModel.agentScore}%</span>
              </div>
            </div>

            <p className="models-page__modal-label">Capabilities</p>
            <div className="models-page__caps">
              {detailModel.capabilities.map((c) => (
                <span key={c} className={`models-page__cap-pill models-page__cap-pill--${pillTint(c)}`}>
                  {c}
                </span>
              ))}
            </div>
          </div>
        </div>
      )}
    </div>
  );
};

export default Models;





























@use '../../../styles/variables' as *;

.models-page {
  display: flex;
  flex-direction: column;
  height: calc(100vh - 166px);
  min-height: 0;
  gap: 18px;

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
    flex-shrink: 0;
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 12px;
  }

  &__body {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding-right: 4px;
    margin-right: -4px;
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

  &__card-top-right {
    display: flex;
    align-items: center;
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

  &__info-btn {
    display: grid;
    place-items: center;
    width: 22px;
    height: 22px;
    flex-shrink: 0;
    border-radius: 6px;
    border: 1px solid $border-default;
    background: $bg-main;
    color: $text-tertiary;
    cursor: pointer;
    transition: border-color 0.14s ease, color 0.14s ease;

    &:hover {
      border-color: $primary;
      color: $primary;
    }
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

  /* ---------- detail modal ---------- */
  &__overlay {
    position: fixed;
    inset: 0;
    z-index: 100;
    background: rgba(14, 21, 38, 0.45);
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 24px;
  }

  &__modal {
    width: 100%;
    max-width: 480px;
    max-height: 85vh;
    overflow-y: auto;
    background: $bg-main;
    border-radius: 16px;
    box-shadow: $shadow-xl;
    padding: 24px 26px;
  }

  &__modal-head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
  }

  &__modal-title {
    margin-top: 8px;
    font-size: 1.1875rem;
    font-weight: 800;
    letter-spacing: -0.015em;
    color: $text-primary;
  }

  &__modal-close {
    flex-shrink: 0;
    display: grid;
    place-items: center;
    width: 28px;
    height: 28px;
    border-radius: 8px;
    border: 1px solid $border-default;
    background: $bg-main;
    color: $text-tertiary;
    cursor: pointer;
    transition: border-color 0.14s ease, color 0.14s ease;

    &:hover {
      border-color: $text-primary;
      color: $text-primary;
    }
  }

  &__modal-desc {
    margin-top: 14px;
    font-size: 0.84375rem;
    line-height: 1.6;
    color: $text-secondary;
  }

  &__modal-specs {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 14px;
    margin-top: 18px;
    padding: 16px;
    background: $bg-subtle;
    border-radius: 12px;
  }

  &__modal-spec {
    display: flex;
    flex-direction: column;
    gap: 4px;
  }

  &__modal-label {
    margin-top: 18px;
    margin-bottom: 8px;
    font-family: $font-mono;
    font-size: 0.65625rem;
    font-weight: 600;
    letter-spacing: 0.08em;
    text-transform: uppercase;
    color: $text-tertiary;
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
