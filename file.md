//Models.tsx
import { useEffect, useMemo, useRef, useState, type FC } from 'react';
import { Search, X, RefreshCw, AlertCircle, Boxes, CheckCircle2, ExternalLink, Check, ChevronDown } from 'lucide-react';
import { fetchModels } from './api';
import type { ModelApi } from './types';
import Spinner from '../../../components/Spinner/Spinner';
import './Models.scss';

const PILL_TINTS = ['blue', 'violet', 'amber', 'jade', 'rose'] as const;

function pillTint(value: string) {
  let hash = 0;
  for (let i = 0; i < value.length; i += 1) hash = (hash * 31 + value.charCodeAt(i)) >>> 0;
  return PILL_TINTS[hash % PILL_TINTS.length];
}

function formatPrice(value: number | null): string {
  return value === null ? '—' : `$${value}`;
}

// ---------- dropdown, structurally the same as History's <Select /> ----------
interface SelectOption {
  value: string;
  label: string;
}

const CapabilitySelect: FC<{ value: string; options: SelectOption[]; onChange: (value: string) => void }> = ({
  value,
  options,
  onChange,
}) => {
  const [open, setOpen] = useState(false);
  const ref = useRef<HTMLDivElement>(null);
  const current = options.find((o) => o.value === value) ?? options[0];

  useEffect(() => {
    const handler = (e: MouseEvent) => {
      if (ref.current && !ref.current.contains(e.target as Node)) setOpen(false);
    };
    document.addEventListener('mousedown', handler);
    return () => document.removeEventListener('mousedown', handler);
  }, []);

  return (
    <div className="models-page__select" ref={ref}>
      <button
        type="button"
        className={`models-page__select-trigger${open ? ' models-page__select-trigger--open' : ''}`}
        onClick={() => setOpen((v) => !v)}
      >
        <span>{current?.label}</span>
        <ChevronDown size={14} className="models-page__select-chevron" />
      </button>

      {open && (
        <div className="models-page__select-menu">
          {options.map((o) => (
            <button
              key={o.value}
              type="button"
              className={`models-page__select-option${o.value === value ? ' models-page__select-option--active' : ''}`}
              onClick={() => {
                onChange(o.value);
                setOpen(false);
              }}
            >
              {o.label}
              {o.value === value && <Check size={13} strokeWidth={2.5} />}
            </button>
          ))}
        </div>
      )}
    </div>
  );
};

const Models: FC = () => {
  const [models, setModels] = useState<ModelApi[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const [query, setQuery] = useState('');
  const [capability, setCapability] = useState('All');

  const load = () => {
    setLoading(true);
    setError(null);
    fetchModels()
      .then((res) => setModels(res.models))
      .catch((err) => setError(err instanceof Error ? err.message : 'Failed to load models.'))
      .finally(() => setLoading(false));
  };

  useEffect(() => {
    load();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  const capabilities = useMemo(() => {
    const set = new Set<string>();
    models.forEach((m) => m.capabilities.forEach((c) => set.add(c)));
    return ['All', ...Array.from(set).sort()];
  }, [models]);

  const capabilityOptions: SelectOption[] = useMemo(
    () => capabilities.map((c) => ({ value: c, label: c === 'All' ? 'All capabilities' : c })),
    [capabilities]
  );

  const filtered = useMemo(() => {
    return models.filter((m) => {
      if (query && !m.name.toLowerCase().includes(query.toLowerCase()) && !m.provider_id.toLowerCase().includes(query.toLowerCase())) {
        return false;
      }
      if (capability !== 'All' && !m.capabilities.includes(capability)) return false;
      return true;
    });
  }, [models, query, capability]);

  return (
    <div className="models-page">
      <div className="models-page__header">
        <div className="models-page__header-left">
          <p className="models-page__header-eyebrow">Model catalog</p>
          <h1 className="models-page__title">Models</h1>
          <p className="models-page__subtitle">Browse available AI models across every connected provider</p>
        </div>

        <div className="models-page__header-right">
          <div className="models-page__header-meta">
            <Boxes size={13} />
            {models.length} models available
          </div>
          <button type="button" className="models-page__btn models-page__btn--outline" onClick={load} disabled={loading}>
            <RefreshCw size={14} strokeWidth={2.25} className={loading ? 'models-page__spin' : undefined} /> Refresh
          </button>
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

        <CapabilitySelect value={capability} options={capabilityOptions} onChange={setCapability} />
      </div>

      {loading && (
        <div className="models-page__loading">
          <Spinner label="Loading models…" />
        </div>
      )}

      {!loading && error && (
        <div className="models-page__empty models-page__empty--error">
          <AlertCircle size={22} />
          <p>{error}</p>
          <button type="button" className="models-page__btn models-page__btn--outline" onClick={load}>
            <RefreshCw size={14} strokeWidth={2.25} /> Try again
          </button>
        </div>
      )}

      {!loading && !error && filtered.length === 0 && (
        <div className="models-page__empty">
          <Search size={22} />
          <p>No models match your filters.</p>
        </div>
      )}

      {!loading && !error && filtered.length > 0 && (
        <div className="models-page__grid">
          {filtered.map((m) => (
            <div className="models-page__card" key={m.id}>
              <div className="models-page__card-top">
                <span className="models-page__card-name">{m.name}</span>
                <span className={`models-page__tag${m.is_active ? ' models-page__tag--jade' : ' models-page__tag--gray'}`}>
                  {m.is_active && <CheckCircle2 size={11} strokeWidth={2.5} />}
                  {m.is_active ? 'Active' : 'Inactive'}
                </span>
              </div>

              <div className="models-page__card-meta">
                <span className="models-page__tag models-page__tag--blue">{m.provider_id}</span>
                <span className="models-page__tag models-page__tag--outline">{m.category}</span>
              </div>

              <div className="models-page__card-stats">
                <div className="models-page__card-stat">
                  <span className="models-page__card-stat-label">Accuracy</span>
                  <span className="models-page__card-stat-value models-page__card-stat-value--accent n">
                    {m.accuracy_score === null ? '—' : `${m.accuracy_score}%`}
                  </span>
                </div>
                <div className="models-page__card-stat">
                  <span className="models-page__card-stat-label">Agent Score</span>
                  <span className="models-page__card-stat-value n">{m.agent_score === null ? '—' : `${m.agent_score}%`}</span>
                </div>
                <div className="models-page__card-stat">
                  <span className="models-page__card-stat-label">Context</span>
                  <span className="models-page__card-stat-value n">{m.context_window.toLocaleString()}</span>
                </div>
                <div className="models-page__card-stat">
                  <span className="models-page__card-stat-label">Price (in / out)</span>
                  <span className="models-page__card-stat-value models-page__card-stat-value--sm n">
                    {formatPrice(m.input_price)} / {formatPrice(m.output_price)}
                  </span>
                </div>
              </div>

              {m.capabilities.length > 0 && (
                <>
                  <span className="models-page__card-section-label">Capabilities</span>
                  <div className="models-page__caps">
                    {m.capabilities.map((c) => (
                      <span key={c} className={`models-page__cap-pill models-page__cap-pill--${pillTint(c)}`}>
                        {c}
                      </span>
                    ))}
                  </div>
                </>
              )}

              <div className="models-page__card-foot">
                {m.base_url ? (
                  <span className="models-page__card-foot-source">
                    <ExternalLink size={12} /> {m.base_url}
                  </span>
                ) : (
                  <span className="models-page__card-foot-source">Default endpoint</span>
                )}
              </div>
            </div>
          ))}
        </div>
      )}
    </div>
  );
};

export default Models;

















//Models.scss
@use '../../../styles/variables' as *;

.models-page {
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
  }

  &__spin {
    animation: models-page-spin 0.9s linear infinite;
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

  /* ---------- capability dropdown (structurally mirrors History's <Select />) ---------- */
  &__select {
    position: relative;
    flex-shrink: 0;
    width: 200px;
  }

  &__select-trigger {
    width: 100%;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
    border: 1px solid $border-default;
    border-radius: 10px;
    padding: 9px 12px;
    background: $bg-main;
    font-size: 0.8125rem;
    font-weight: 500;
    font-family: $font-body;
    color: $text-primary;
    cursor: pointer;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &:hover {
      border-color: $border-strong;
    }

    &--open {
      border-color: $primary;
      box-shadow: 0 0 0 3px $primary-light;
    }
  }

  &__select-chevron {
    flex-shrink: 0;
    color: $text-tertiary;
    transition: transform 0.16s ease;
  }

  &__select-trigger--open &__select-chevron {
    transform: rotate(180deg);
  }

  &__select-menu {
    position: absolute;
    top: calc(100% + 6px);
    left: 0;
    right: 0;
    z-index: 20;
    max-height: 16rem;
    overflow-y: auto;
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 10px;
    box-shadow: $shadow-lg;
    padding: 5px;
    display: flex;
    flex-direction: column;
    gap: 1px;
  }

  &__select-option {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
    width: 100%;
    text-align: left;
    padding: 8px 10px;
    border: none;
    border-radius: 7px;
    background: transparent;
    font-size: 0.8125rem;
    font-family: $font-body;
    color: $text-secondary;
    cursor: pointer;
    transition: background 0.12s ease, color 0.12s ease;

    &:hover {
      background: $bg-subtle;
      color: $text-primary;
    }

    &--active {
      color: $primary;
      font-weight: 600;

      svg {
        color: $primary;
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

    &--jade {
      color: $success;
      background: $success-subtle;
    }

    &--gray {
      color: $text-tertiary;
      background: $bg-inset;
    }

    &--outline {
      color: $text-secondary;
      background: transparent;
      border: 1px solid $border-default;
    }
  }

  /* ---------- capability pills ---------- */
  &__caps {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    margin-bottom: 10px;
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
    grid-template-columns: repeat(3, 1fr);
    gap: 16px;
  }

  &__card {
    display: flex;
    flex-direction: column;
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-left: 3px solid $card-accent;
    border-radius: 0.75rem;
    padding: 15px 18px;
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
    margin-bottom: 6px;
  }

  &__card-meta {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    margin-bottom: 10px;
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
    line-height: 1.5;
    margin-bottom: 10px;
  }

  &__card-stats {
    display: flex;
    flex-wrap: wrap;
    gap: 16px 20px;
    margin-bottom: 10px;
    padding-bottom: 10px;
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

    &--accent {
      color: $success;
    }
  }

  &__card-section-label {
    font-size: 0.65625rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    color: $text-tertiary;
    margin-bottom: 6px;
    display: block;
  }

  &__card-foot {
    margin-top: auto;
    padding-top: 10px;
    border-top: 1px solid $border-subtle;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
  }

  &__card-foot-source {
    font-family: $font-mono;
    font-size: 0.6875rem;
    color: $text-tertiary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
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

  /* ---------- responsive ---------- */
  @media (max-width: 1500px) {
    &__grid {
      grid-template-columns: repeat(2, 1fr);
    }
  }

  @media (max-width: 1000px) {
    &__grid {
      grid-template-columns: 1fr;
    }
  }

  @media (max-width: 640px) {
    &__card-stats {
      flex-wrap: wrap;
      gap: 14px;
    }
  }

  /* ---------- ultra-wide: nudge key text sizes up a touch ---------- */
  @media (min-width: 1800px) {
    &__title {
      font-size: 22px;
    }

    &__subtitle {
      font-size: 0.875rem;
    }

    &__card-name {
      font-size: 0.96875rem;
    }

    &__card-desc {
      font-size: 0.8125rem;
    }

    &__card-stat-value {
      font-size: 0.96875rem;
    }

    &__card-stat-value--sm {
      font-size: 0.8125rem;
    }

    &__cap-pill {
      font-size: 0.75rem;
    }

    &__card-stat-label {
      font-size: 0.65625rem;
    }

    &__card-section-label {
      font-size: 0.6875rem;
    }
  }
}

@keyframes models-page-spin {
  to {
    transform: rotate(360deg);
  }
}
