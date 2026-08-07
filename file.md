import { useMemo, useState, type FC } from 'react';
import { Search, Check, X, Loader2, AlertTriangle } from 'lucide-react';
import type { ModelApi, ProviderApi } from '../types';

interface Props {
  models: ModelApi[];
  providerCatalog: ProviderApi[];
  selectedProviders: string[];
  selected: string[];
  onToggle: (id: string) => void;
  onClear: () => void;
  loading: boolean;
  error: string | null;
}

function formatContextWindow(tokens: number): string {
  if (tokens >= 1_000_000) return `${(tokens / 1_000_000).toLocaleString()}M tokens`;
  if (tokens >= 1_000) return `${Math.round(tokens / 1000)}k tokens`;
  return `${tokens} tokens`;
}

function formatPrice(price: number | null): string {
  return price === null ? '—' : `$${price.toFixed(2)}`;
}

function formatPricing(m: ModelApi): string {
  if (m.input_price === null && m.output_price === null) return 'Custom pricing';
  return `${formatPrice(m.input_price)} in · ${formatPrice(m.output_price)} out /1M`;
}

const ModelsStep: FC<Props> = ({
  models,
  providerCatalog,
  selectedProviders,
  selected,
  onToggle,
  onClear,
  loading,
  error,
}) => {
  const [query, setQuery] = useState('');
  const [capFilters, setCapFilters] = useState<string[]>([]);

  const providerNameById = useMemo(() => {
    const map = new Map<string, string>();
    providerCatalog.forEach((p) => map.set(p.id, p.name));
    return map;
  }, [providerCatalog]);

  const activeModels = useMemo(() => models.filter((m) => m.is_active), [models]);

  const pool = useMemo(
    () =>
      selectedProviders.length
        ? activeModels.filter((m) => selectedProviders.includes(m.provider_id))
        : activeModels,
    [activeModels, selectedProviders]
  );

  const allCapabilities = useMemo(() => {
    const set = new Set<string>();
    pool.forEach((m) => m.capabilities.forEach((c) => set.add(c)));
    return Array.from(set).sort();
  }, [pool]);

  const filtered = useMemo(() => {
    return pool.filter((m: ModelApi) => {
      const providerName = providerNameById.get(m.provider_id) ?? '';
      if (
        query &&
        !m.name.toLowerCase().includes(query.toLowerCase()) &&
        !providerName.toLowerCase().includes(query.toLowerCase())
      ) {
        return false;
      }
      if (capFilters.length && !capFilters.every((c) => m.capabilities.includes(c))) return false;
      return true;
    });
  }, [pool, query, capFilters, providerNameById]);

  const toggleCap = (cap: string) =>
    setCapFilters((prev) => (prev.includes(cap) ? prev.filter((c) => c !== cap) : [...prev, cap]));
  const resetFilters = () => {
    setCapFilters([]);
    setQuery('');
  };

  return (
    <div className="run-eval__card run-eval__card--wide">
      <h2 className="run-eval__step-title">Choose models</h2>
      <p className="run-eval__step-desc">Select the models you want to compare. Use filters to narrow the list.</p>

      {loading && (
        <div className="run-eval__loading-state">
          <Loader2 size={18} className="run-eval__spin" />
          Loading models…
        </div>
      )}

      {!loading && error && (
        <div className="run-eval__inline-error">
          <AlertTriangle size={15} />
          {error}
        </div>
      )}

      {!loading && !error && (
        <div className="run-eval__models-layout">
          <aside className="run-eval__filters">
            <div className="run-eval__filters-head">
              <span>Filters</span>
              <button type="button" className="run-eval__link" onClick={resetFilters}>
                Reset all
              </button>
            </div>

            <div className="run-eval__filters-scroll">
              <div className="run-eval__filter-section">
                <p className="run-eval__filter-title">Capabilities</p>
                <div className="run-eval__filter-options">
                  {allCapabilities.map((cap) => (
                    <label key={cap} className="run-eval__filter-chip">
                      <input type="checkbox" checked={capFilters.includes(cap)} onChange={() => toggleCap(cap)} />
                      {cap}
                    </label>
                  ))}
                  {allCapabilities.length === 0 && <p className="run-eval__filter-empty">No capabilities listed.</p>}
                </div>
              </div>
            </div>
          </aside>

          <div className="run-eval__models-main">
            <div className="run-eval__search-bar">
              <Search size={15} />
              <input
                type="text"
                placeholder="Search models..."
                value={query}
                onChange={(e) => setQuery(e.target.value)}
              />
            </div>

            {capFilters.length > 0 && (
              <div className="run-eval__active-filters">
                {capFilters.map((c) => (
                  <span key={c} className="run-eval__tag">
                    {c}
                    <button type="button" onClick={() => toggleCap(c)}>
                      <X size={11} />
                    </button>
                  </span>
                ))}
              </div>
            )}

            <div className="run-eval__models-scroll">
              <div className="run-eval__models-grid">
                {filtered.map((m) => {
                  const isSelected = selected.includes(m.id);
                  return (
                    <button
                      key={m.id}
                      type="button"
                      className={`run-eval__model-card${isSelected ? ' run-eval__model-card--selected' : ''}`}
                      onClick={() => onToggle(m.id)}
                    >
                      <div className="run-eval__model-top">
                        <span className="run-eval__model-name">{m.name}</span>
                        {isSelected && (
                          <span className="run-eval__type-check">
                            <Check size={12} strokeWidth={2.75} />
                          </span>
                        )}
                      </div>
                      <span className="run-eval__model-provider">
                        {providerNameById.get(m.provider_id) ?? m.provider_id}
                      </span>
                      <div className="run-eval__model-caps">
                        {m.capabilities.slice(0, 3).map((c) => (
                          <span key={c} className="run-eval__chip run-eval__chip--static">
                            {c}
                          </span>
                        ))}
                      </div>
                      <div className="run-eval__model-meta n">
                        <span>{formatContextWindow(m.context_window)}</span>
                        <span>{formatPricing(m)}</span>
                        {m.accuracy_score !== null && <span>Accuracy {m.accuracy_score.toFixed(1)}%</span>}
                      </div>
                    </button>
                  );
                })}
                {filtered.length === 0 && <p className="run-eval__empty">No models match these filters.</p>}
              </div>
            </div>

            {selected.length > 0 && (
              <div className="run-eval__selected-bar">
                <span>
                  <strong>{selected.length}</strong> models selected
                </span>
                <button type="button" className="run-eval__btn run-eval__btn--sm" onClick={onClear}>
                  Clear
                </button>
              </div>
            )}
          </div>
        </div>
      )}
    </div>
  );
};

export default ModelsStep;
























@use '../../../styles/variables' as *;

.run-eval {
  display: flex;
  flex-direction: column;
  gap: 18px;
  min-height: 0;
  height: calc(100% + 3rem - 0.75rem);
  margin-bottom: calc(-3rem + 0.75rem);

  /* ---------- header ---------- */
  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 1rem;
  }

  &__header-eyebrow {
    font-size: 0.75rem;
    font-weight: 700;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $primary;
    margin-bottom: 2px;
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

  &__header-meta {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-tertiary;
    background: $bg-subtle;
    border: 1px solid $border-subtle;
    border-radius: 999px;
    padding: 6px 12px;
  }

  /* ---------- generic buttons ---------- */
  &__btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 7px;
    font-family: $font-body;
    font-size: 0.8125rem;
    font-weight: 600;
    padding: 9px 14px;
    border-radius: 8px;
    border: 1px solid $border-default;
    background: $bg-main;
    color: $text-secondary;
    cursor: pointer;
    white-space: nowrap;
    transition: background 0.12s ease, border-color 0.12s ease, color 0.12s ease, box-shadow 0.12s ease;

    &:hover:not(:disabled) {
      border-color: $primary;
      box-shadow: $shadow-sm;
    }

    &:disabled {
      opacity: 0.5;
      cursor: not-allowed;
    }

    &--primary {
      background: $primary;
      border-color: $primary;
      color: #fff;

      &:hover:not(:disabled) {
        background: $primary-hover;
        border-color: $primary-hover;
        color: #fff;
      }
    }

    &--secondary {
      background: $bg-main;
      color: $text-secondary;
    }

    &--danger:hover:not(:disabled) {
      border-color: $danger;
      color: $danger;
      background: $danger-subtle;
    }

    &--sm {
      padding: 6px 10px;
      font-size: 0.75rem;

      svg {
        width: 13px;
        height: 13px;
      }
    }

    &--lg {
      padding: 11px 20px;
      font-size: 0.875rem;
    }
  }

  &__link {
    background: none;
    border: none;
    padding: 0;
    font-family: $font-body;
    font-size: 0.75rem;
    font-weight: 600;
    color: $primary;
    cursor: pointer;

    &:hover {
      text-decoration: underline;
    }
  }

  &__spin {
    animation: run-eval-spin 0.8s linear infinite;
  }

  /* ---------- wizard shell ---------- */
  &__wizard {
    flex: 1;
    display: flex;
    gap: 20px;
    min-height: 0;
  }

  /* ---------- sidebar step list ---------- */
  &__sidebar {
    flex: 0 0 240px;
    display: flex;
    flex-direction: column;
    gap: 4px;
    min-height: 0;
    overflow-y: auto;
  }

  &__step {
    display: flex;
    align-items: flex-start;
    gap: 12px;
    text-align: left;
    padding: 12px;
    border: 1px solid transparent;
    border-radius: $radius-lg;
    background: transparent;
    cursor: pointer;
    transition: background 0.12s ease, border-color 0.12s ease;

    &:disabled {
      cursor: not-allowed;
      opacity: 0.55;
    }

    &--active {
      background: $primary-light;
      border-color: $primary;
    }

    &--complete:hover {
      background: $bg-subtle;
    }
  }

  &__step-marker {
    flex-shrink: 0;
    width: 28px;
    height: 28px;
    border-radius: 50%;
    display: grid;
    place-items: center;
    background: $bg-inset;
    color: $text-tertiary;
  }

  &__step--active &__step-marker {
    background: $primary;
    color: #fff;
  }

  &__step--complete &__step-marker {
    background: $success-subtle;
    color: $success;
  }

  &__step-text {
    display: flex;
    flex-direction: column;
    gap: 2px;
    min-width: 0;
  }

  &__step-label {
    font-size: 0.8125rem;
    font-weight: 700;
    color: $text-primary;
  }

  /* ---------- main content column ---------- */
  &__content {
    flex: 1;
    display: flex;
    flex-direction: column;
    min-width: 0;
    min-height: 0;
  }

  &__step-kicker {
    flex-shrink: 0;
    font-size: 0.6875rem;
    font-weight: 700;
    letter-spacing: 0.05em;
    text-transform: uppercase;
    color: $text-tertiary;
    margin-bottom: 10px;
  }

  &__body {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
  }

  /* ---------- step card shell ---------- */
  &__card {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 16px;
    box-shadow: $shadow-xs;
    padding: 26px 28px;
    display: flex;
    flex-direction: column;
    min-height: 0;

    &--wide {
      height: 100%;
    }
  }

  &__step-title {
    font-size: 1.0625rem;
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $text-primary;
    margin-bottom: 4px;
  }

  &__step-desc {
    font-size: 0.8125rem;
    color: $text-secondary;
    margin-bottom: 18px;
  }

  &__step-header-row {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
    margin-bottom: 4px;
  }

  &__error {
    margin-top: 14px;
    font-size: 0.8125rem;
    font-weight: 600;
    color: $danger;
  }

  &__hint {
    display: flex;
    align-items: flex-start;
    gap: 8px;
    margin-top: 18px;
    padding: 12px 14px;
    border-radius: 10px;
    background: $bg-subtle;
    color: $text-tertiary;
    font-size: 0.75rem;

    svg {
      flex-shrink: 0;
      margin-top: 1px;
    }
  }

  &__empty {
    grid-column: 1 / -1;
    padding: 32px 16px;
    text-align: center;
    color: $text-tertiary;
    font-size: 0.8125rem;
  }

  &__filter-empty {
    color: $text-tertiary;
    font-size: 0.75rem;
  }

  &__loading-state,
  &__inline-error {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 0.8125rem;
    padding: 14px;
    border-radius: 10px;
  }

  &__loading-state {
    color: $text-tertiary;
    background: $bg-subtle;
  }

  &__inline-error {
    color: $danger;
    background: $danger-subtle;
  }

  /* ---------- generic chips / tags ---------- */
  &__chip {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 5px 10px;
    border-radius: 999px;
    border: 1px solid $border-default;
    background: $bg-main;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-secondary;
    cursor: pointer;
    white-space: nowrap;
    transition: background 0.12s ease, border-color 0.12s ease, color 0.12s ease;

    &:hover {
      border-color: $primary;
    }

    &--active {
      background: $primary;
      border-color: $primary;
      color: #fff;
    }

    &--static {
      cursor: default;
      background: $bg-subtle;
      border-color: $border-subtle;
      color: $text-tertiary;
      font-weight: 500;

      &:hover {
        border-color: $border-subtle;
      }
    }
  }

  &__tag {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 5px 6px 5px 10px;
    border-radius: 999px;
    background: $primary-light;
    color: $primary;
    font-size: 0.75rem;
    font-weight: 600;

    button {
      display: grid;
      place-items: center;
      width: 16px;
      height: 16px;
      border-radius: 50%;
      border: none;
      background: rgba(255, 255, 255, 0.5);
      color: $primary;
      cursor: pointer;

      &:hover {
        background: #fff;
      }
    }
  }

  &__badge {
    display: inline-flex;
    align-items: center;
    padding: 3px 9px;
    border-radius: 999px;
    background: $bg-subtle;
    color: $text-tertiary;
    font-size: 0.625rem;
    font-weight: 700;
    letter-spacing: 0.02em;

    &--soft {
      background: $success-subtle;
      color: $success;
    }
  }

  &__type-check {
    flex-shrink: 0;
    width: 20px;
    height: 20px;
    border-radius: 50%;
    display: grid;
    place-items: center;
    background: $primary;
    color: #fff;
  }

  &__radio {
    flex-shrink: 0;
    width: 16px;
    height: 16px;
    border-radius: 50%;
    border: 2px solid $border-strong;
    background: $bg-main;

    &--checked {
      border-color: $primary;
      border-width: 5px;
    }
  }

  &__checkbox {
    flex-shrink: 0;
    width: 16px;
    height: 16px;
    border-radius: 4px;
    border: 1.5px solid $border-strong;
    background: $bg-main;
    display: grid;
    place-items: center;
    color: #fff;

    &--checked {
      background: $primary;
      border-color: $primary;
    }
  }

  &__filter-title {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 0.6875rem;
    font-weight: 700;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $text-tertiary;
    margin-bottom: 10px;
  }

  /* ================================================================
     Step 1 — Name
     ================================================================ */
  &__field {
    display: flex;
    flex-direction: column;
    gap: 6px;
    max-width: 420px;

    &--judge {
      max-width: none;
      margin-top: 14px;
    }
  }

  &__label {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 0.75rem;
    font-weight: 700;
    color: $text-secondary;
  }

  &__input {
    border: 1px solid $border-default;
    border-radius: 10px;
    padding: 10px 12px;
    font-size: 0.84375rem;
    font-family: $font-body;
    color: $text-primary;
    background: $bg-main;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &:focus {
      outline: none;
      border-color: $primary;
      box-shadow: 0 0 0 3px $primary-light;
    }

    &::placeholder {
      color: $text-tertiary;
    }
  }

  &__name-suggestions {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    margin-top: 12px;
  }

  /* ================================================================
     Step 2 — Type
     ================================================================ */
  &__type-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
    gap: 14px;
  }

  &__type-card {
    position: relative;
    display: flex;
    flex-direction: column;
    align-items: flex-start;
    gap: 10px;
    text-align: left;
    padding: 18px;
    border: 1px solid $border-subtle;
    border-radius: $radius-lg;
    background: $bg-main;
    cursor: pointer;
    transition: border-color 0.12s ease, box-shadow 0.12s ease, background 0.12s ease;

    &:hover {
      border-color: $primary;
      box-shadow: $shadow-sm;
    }

    &--selected {
      border-color: $primary;
      background: $primary-light;
    }

    &--framework {
      padding: 14px;
    }
  }

  &__type-icon {
    width: 40px;
    height: 40px;
    border-radius: $radius-md;
    display: grid;
    place-items: center;
    background: $bg-subtle;
    color: $text-secondary;
  }

  &__type-card--selected &__type-icon {
    background: $bg-main;
    color: $primary;
  }

  &__type-content {
    display: flex;
    flex-direction: column;
    gap: 4px;
  }

  &__type-title {
    font-size: 0.875rem;
    font-weight: 700;
    color: $text-primary;
  }

  &__type-desc {
    font-size: 0.75rem;
    color: $text-tertiary;
    line-height: 1.45;
  }

  &__type-card &__badge {
    margin-top: 2px;
  }

  &__type-card &__type-check {
    position: absolute;
    top: 14px;
    right: 14px;
  }

  &__framework-section {
    margin-top: 24px;
    padding-top: 20px;
    border-top: 1px solid $border-subtle;
  }

  &__framework-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
    gap: 12px;
    margin-top: 12px;
  }

  /* ================================================================
     Step 3 — Providers
     ================================================================ */
  &__provider-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
    gap: 14px;
  }

  &__provider-card {
    display: flex;
    flex-direction: column;
    gap: 10px;
    text-align: left;
    padding: 16px;
    border: 1px solid $border-subtle;
    border-radius: $radius-lg;
    background: $bg-main;
    cursor: pointer;
    transition: border-color 0.12s ease, box-shadow 0.12s ease, background 0.12s ease;

    &:hover {
      border-color: $primary;
      box-shadow: $shadow-sm;
    }

    &--selected {
      border-color: $primary;
      background: $primary-light;
    }

    &--disabled {
      cursor: not-allowed;
      opacity: 0.55;
    }
  }

  /* ================================================================
     Step 4 — Models  (independent-scroll filters + card list)
     ================================================================ */
  &__models-layout {
    flex: 1;
    display: flex;
    gap: 20px;
    min-height: 0;
  }

  &__filters {
    flex: 0 0 220px;
    display: flex;
    flex-direction: column;
    min-height: 0;
    border: 1px solid $border-subtle;
    border-radius: $radius-lg;
    background: $bg-subtle;
    overflow: hidden;
  }

  &__filters-head {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 14px 14px 10px;
    font-size: 0.75rem;
    font-weight: 700;
    color: $text-primary;
  }

  // Scrolls independently from the model card list on the right.
  &__filters-scroll {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding: 0 14px 14px;
  }

  &__filter-section {
    padding-bottom: 16px;
    margin-bottom: 16px;
    border-bottom: 1px solid $border-subtle;

    &:last-child {
      border-bottom: none;
      margin-bottom: 0;
      padding-bottom: 0;
    }
  }

  &__filter-options {
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  &__filter-chip {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 0.78125rem;
    color: $text-secondary;
    cursor: pointer;

    input[type='checkbox'] {
      width: 14px;
      height: 14px;
      accent-color: $primary;
      cursor: pointer;
    }

    &:hover {
      color: $text-primary;
    }
  }

  &__models-main {
    flex: 1;
    display: flex;
    flex-direction: column;
    gap: 12px;
    min-width: 0;
    min-height: 0;
  }

  &__search-bar {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 9px;
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

  &__active-filters {
    flex-shrink: 0;
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
  }

  // Scrolls independently from the filters panel on the left.
  &__models-scroll {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
  }

  &__models-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
    gap: 12px;
    align-content: start;
  }

  &__model-card {
    display: flex;
    flex-direction: column;
    gap: 8px;
    text-align: left;
    padding: 14px;
    border: 1px solid $border-subtle;
    border-radius: $radius-lg;
    background: $bg-main;
    cursor: pointer;
    transition: border-color 0.12s ease, box-shadow 0.12s ease, background 0.12s ease;

    &:hover {
      border-color: $primary;
      box-shadow: $shadow-sm;
    }

    &--selected {
      border-color: $primary;
      background: $primary-light;
    }
  }

  &__model-top {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 8px;
  }

  &__model-name {
    font-size: 0.84375rem;
    font-weight: 700;
    color: $text-primary;
  }

  &__model-provider {
    font-size: 0.75rem;
    color: $text-tertiary;
  }

  &__model-caps {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
  }

  &__model-meta {
    display: flex;
    flex-wrap: wrap;
    gap: 10px;
    font-size: 0.71875rem;
    color: $text-tertiary;
    margin-top: 2px;
  }

  &__selected-bar {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
    padding: 10px 14px;
    border-radius: 10px;
    background: $primary-light;
    color: $primary;
    font-size: 0.8125rem;
  }

  /* ================================================================
     Step 5 — Dataset
     ================================================================ */
  &__tabs {
    display: flex;
    gap: 4px;
    padding: 4px;
    border-radius: 10px;
    background: $bg-subtle;
    width: fit-content;
    margin-bottom: 18px;
  }

  &__tab {
    padding: 8px 16px;
    border-radius: 8px;
    border: none;
    background: transparent;
    font-family: $font-body;
    font-size: 0.8125rem;
    font-weight: 600;
    color: $text-tertiary;
    cursor: pointer;
    transition: background 0.12s ease, color 0.12s ease;

    &--active {
      background: $bg-main;
      color: $text-primary;
      box-shadow: $shadow-xs;
    }
  }

  &__category-filters {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    margin-bottom: 16px;
  }

  &__dataset-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(260px, 1fr));
    gap: 14px;
  }

  &__dataset-card {
    display: flex;
    flex-direction: column;
    gap: 8px;
    text-align: left;
    padding: 16px;
    border: 1px solid $border-subtle;
    border-radius: $radius-lg;
    background: $bg-main;
    cursor: pointer;
    transition: border-color 0.12s ease, box-shadow 0.12s ease, background 0.12s ease;

    &:hover {
      border-color: $primary;
      box-shadow: $shadow-sm;
    }

    &--selected {
      border-color: $primary;
      background: $primary-light;
    }
  }

  &__dataset-top {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 8px;
  }

  &__dataset-top-actions {
    display: flex;
    align-items: center;
    gap: 8px;
    flex-shrink: 0;
  }

  &__dataset-name {
    font-size: 0.84375rem;
    font-weight: 700;
    color: $text-primary;
  }

  &__dataset-desc {
    font-size: 0.75rem;
    color: $text-secondary;
    line-height: 1.45;
  }

  &__dataset-meta {
    display: flex;
    gap: 10px;
    font-size: 0.71875rem;
    color: $text-tertiary;
  }

  &__dataset-caps {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
  }

  &__subgroup-btn {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    padding: 4px 8px;
    border-radius: 999px;
    background: $bg-subtle;
    color: $text-tertiary;
    font-size: 0.6875rem;
    font-weight: 700;
    cursor: pointer;

    &:hover {
      background: $primary-light;
      color: $primary;
    }
  }

  &__upload-zone {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 8px;
    padding: 48px 20px;
    border: 1.5px dashed $border-strong;
    border-radius: $radius-lg;
    color: $text-tertiary;
    text-align: center;

    h3 {
      font-size: 0.9375rem;
      font-weight: 700;
      color: $text-primary;
      margin-top: 4px;
    }

    p {
      font-size: 0.8125rem;
    }
  }

  &__format-chips {
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    gap: 8px;
    margin-top: 8px;
  }

  /* ---------- subgroup drawer ---------- */
  &__drawer-overlay {
    position: fixed;
    inset: 0;
    background: rgba(15, 23, 42, 0.4);
    opacity: 0;
    pointer-events: none;
    z-index: 40;
    transition: opacity 0.18s ease;

    &--open {
      opacity: 1;
      pointer-events: auto;
    }
  }

  &__drawer {
    position: fixed;
    top: 0;
    right: 0;
    bottom: 0;
    width: min(380px, 100%);
    background: $bg-main;
    box-shadow: $shadow-lg;
    transform: translateX(100%);
    transition: transform 0.22s ease;
    display: flex;
    flex-direction: column;

    &--open {
      transform: translateX(0);
    }
  }

  &__drawer-header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
    padding: 20px;
    border-bottom: 1px solid $border-subtle;
  }

  &__drawer-eyebrow {
    font-size: 0.6875rem;
    font-weight: 700;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $text-tertiary;
    margin-bottom: 4px;
  }

  &__drawer-title {
    font-size: 1rem;
    font-weight: 800;
    color: $text-primary;
  }

  &__drawer-close {
    flex-shrink: 0;
    width: 28px;
    height: 28px;
    border-radius: 50%;
    border: none;
    background: $bg-subtle;
    color: $text-tertiary;
    display: grid;
    place-items: center;
    cursor: pointer;

    &:hover {
      background: $border-default;
      color: $text-primary;
    }
  }

  &__drawer-body {
    flex: 1;
    overflow-y: auto;
    padding: 12px;
    display: flex;
    flex-direction: column;
    gap: 4px;
  }

  &__drawer-task {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 10px;
    border-radius: 8px;
    border: none;
    background: transparent;
    text-align: left;
    cursor: pointer;
    transition: background 0.12s ease;

    &:hover {
      background: $bg-subtle;
    }

    &--selected {
      background: $primary-light;
    }
  }

  &__drawer-task-name {
    font-size: 0.8125rem;
    color: $text-primary;
  }

  /* ================================================================
     Step 6 — Metrics
     ================================================================ */
  &__metrics-count {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 10px;
    font-size: 0.75rem;
    color: $text-tertiary;
  }

  &__metrics-count-num {
    font-size: 0.9375rem;
    font-weight: 800;
    color: $primary;
  }

  &__metrics-bulk-actions {
    display: flex;
    align-items: center;
    gap: 8px;
  }

  &__metrics-bulk-divider {
    width: 1px;
    height: 12px;
    background: $border-default;
  }

  &__metrics-toggle-all {
    background: none;
    border: none;
    padding: 0;
    font-family: $font-body;
    font-size: 0.75rem;
    font-weight: 600;
    color: $primary;
    cursor: pointer;

    &:hover:not(:disabled) {
      text-decoration: underline;
    }

    &:disabled {
      color: $text-tertiary;
      cursor: not-allowed;
    }
  }

  &__metrics-layout {
    flex: 1;
    display: flex;
    gap: 24px;
    min-height: 0;
    margin-top: 16px;
  }

  &__metrics-main {
    flex: 1;
    min-width: 0;
    overflow-y: auto;
  }

  &__metric-group {
    margin-bottom: 22px;

    &:last-child {
      margin-bottom: 0;
    }
  }

  &__metrics-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(180px, 1fr));
    gap: 10px;
  }

  &__metric-card {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
    padding: 12px 14px;
    border: 1px solid $border-subtle;
    border-radius: $radius-md;
    background: $bg-main;
    cursor: pointer;
    transition: border-color 0.12s ease, box-shadow 0.12s ease, background 0.12s ease;

    &:hover {
      border-color: $primary;
      box-shadow: $shadow-sm;
    }

    &--selected {
      border-color: $primary;
      background: $primary-light;
    }
  }

  &__metric-name {
    font-size: 0.8125rem;
    font-weight: 600;
    color: $text-primary;
  }

  /* ---------- judge panel ---------- */
  &__judge-panel {
    flex: 0 0 280px;
    display: flex;
    flex-direction: column;
    border: 1px solid $border-subtle;
    border-radius: $radius-lg;
    background: $bg-subtle;
    padding: 16px;
    overflow-y: auto;
  }

  &__judge-hint {
    font-size: 0.75rem;
    color: $text-tertiary;
    line-height: 1.45;
    margin-bottom: 14px;
  }

  &__judge-empty {
    padding: 16px;
    border-radius: 10px;
    background: $bg-main;
    border: 1px dashed $border-strong;
    color: $text-tertiary;
    font-size: 0.75rem;
    text-align: center;
  }

  &__judge-list {
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  &__judge-row {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 10px;
    border: 1px solid $border-subtle;
    border-radius: $radius-md;
    background: $bg-main;
    cursor: pointer;
    text-align: left;
    transition: border-color 0.12s ease, background 0.12s ease;

    &:hover {
      border-color: $primary;
    }

    &--selected {
      border-color: $primary;
      background: $primary-light;
    }
  }

  &__judge-row-text {
    display: flex;
    flex-direction: column;
    gap: 2px;
    min-width: 0;
  }

  &__judge-row-name {
    font-size: 0.8125rem;
    font-weight: 700;
    color: $text-primary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__judge-row-meta {
    font-size: 0.6875rem;
    color: $text-tertiary;
  }

  /* ================================================================
     Step 7 — Review
     ================================================================ */
  &__review {
    display: flex;
    flex-direction: column;
    gap: 2px;
  }

  &__review-row {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
    padding: 11px 0;
    border-bottom: 1px solid $border-subtle;
    font-size: 0.8125rem;

    span:first-child {
      color: $text-tertiary;
      font-weight: 600;
      flex-shrink: 0;
    }

    span:last-child {
      color: $text-primary;
      font-weight: 600;
      text-align: right;
    }

    &--highlight {
      span:last-child {
        color: $primary;
        font-size: 0.9375rem;
        font-weight: 800;
      }
    }
  }

  &__review-divider {
    height: 1px;
    background: $border-default;
    margin: 6px 0;
  }

  /* ---------- footer nav ---------- */
  &__nav {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
    margin-top: 18px;
  }

  /* ---------- responsive ---------- */
  @media (max-width: 1100px) {
    &__models-layout {
      flex-direction: column;
    }

    &__filters {
      flex: 0 0 auto;
      max-height: 220px;
    }

    &__metrics-layout {
      flex-direction: column;
    }

    &__judge-panel {
      flex: 0 0 auto;
      max-height: 320px;
    }
  }

  @media (max-width: 900px) {
    height: auto;
    margin-bottom: 0;

    &__wizard {
      flex-direction: column;
    }

    &__sidebar {
      flex: 0 0 auto;
      flex-direction: row;
      flex-wrap: wrap;
      overflow-y: visible;
    }

    &__step {
      flex: 1 1 200px;
    }
  }

  @media (max-width: 520px) {
    &__card {
      padding: 18px 16px;
    }

    &__type-grid,
    &__provider-grid,
    &__dataset-grid,
    &__models-grid,
    &__metrics-grid {
      grid-template-columns: 1fr;
    }
  }
}

@keyframes run-eval-spin {
  to {
    transform: rotate(360deg);
  }
}
