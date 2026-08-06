//Models.tsx
import { useEffect, useMemo, useRef, useState, type FC, type FormEvent } from 'react';
import { Search, X, RefreshCw, AlertCircle, Boxes, CheckCircle2, ExternalLink, Check, ChevronDown, Plus, Key, Sparkles, Type, Hash, Tag, Globe, Layers, FileText } from 'lucide-react';
import { fetchModels, addCustomModel, type AddCustomModelPayload } from './api';
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

const EMPTY_FORM: AddCustomModelPayload = {
  base_url: '',
  category: '',
  api_key: '',
  model_id: '',
  name: '',
  context_window: 0,
  description: '',
};

const Models: FC = () => {
  const [models, setModels] = useState<ModelApi[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const [query, setQuery] = useState('');
  const [capability, setCapability] = useState('All');

  // ---------- add custom model modal ----------
  const [addModalOpen, setAddModalOpen] = useState(false);
  const [form, setForm] = useState<AddCustomModelPayload>(EMPTY_FORM);
  const [submitting, setSubmitting] = useState(false);
  const [submitError, setSubmitError] = useState<string | null>(null);

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

  const openAddModal = () => {
    setForm(EMPTY_FORM);
    setSubmitError(null);
    setAddModalOpen(true);
  };

  const updateForm = <K extends keyof AddCustomModelPayload>(key: K, value: AddCustomModelPayload[K]) => {
    setForm((prev) => ({ ...prev, [key]: value }));
  };

  const isFormValid = form.model_id.trim() && form.name.trim() && form.category.trim() && form.base_url.trim() && form.api_key.trim();

  const handleSubmit = (e: FormEvent) => {
    e.preventDefault();
    if (!isFormValid || submitting) return;

    setSubmitting(true);
    setSubmitError(null);

    addCustomModel(form)
      .then(() => {
        setAddModalOpen(false);
        load();
      })
      .catch((err) => {
        setSubmitError(err instanceof Error ? err.message : 'Failed to add model.');
      })
      .finally(() => setSubmitting(false));
  };

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
          <button type="button" className="models-page__btn models-page__btn--primary" onClick={openAddModal}>
            <Plus size={14} strokeWidth={2.5} /> Add Model
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
                <div className="models-page__card-stats-row">
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
                  <div className="models-page__caps">
                    {m.capabilities.map((c) => (
                      <span key={c} className={`models-page__cap-pill models-page__cap-pill--${pillTint(c)}`}>
                        {c}
                      </span>
                    ))}
                  </div>
                )}
              </div>

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

      {addModalOpen && (
        <div className="models-page__overlay" onClick={() => !submitting && setAddModalOpen(false)}>
          <div className="models-page__modal" onClick={(e) => e.stopPropagation()}>
            <div className="models-page__modal-head">
              <span className="models-page__modal-icon">
                <Sparkles size={18} strokeWidth={2.25} />
              </span>
              <div className="models-page__modal-head-text">
                <h2 className="models-page__modal-title">Add Custom Model</h2>
                <p className="models-page__modal-sub">Register a model hosted on your own endpoint</p>
              </div>
              <button
                type="button"
                className="models-page__modal-close"
                onClick={() => setAddModalOpen(false)}
                disabled={submitting}
                aria-label="Close"
              >
                <X size={16} />
              </button>
            </div>

            <form className="models-page__form" onSubmit={handleSubmit}>
              <p className="models-page__form-section-label">
                <Type size={12} strokeWidth={2.5} /> Basic info
              </p>

              <div className="models-page__field">
                <label className="models-page__field-label" htmlFor="model-name">
                  Name
                </label>
                <div className="models-page__input-wrap">
                  <Type size={14} />
                  <input
                    id="model-name"
                    type="text"
                    className="models-page__input models-page__input--inset"
                    placeholder="e.g. My Custom Model"
                    value={form.name}
                    onChange={(e) => updateForm('name', e.target.value)}
                    disabled={submitting}
                  />
                </div>
              </div>

              <div className="models-page__form-row">
                <div className="models-page__field">
                  <label className="models-page__field-label" htmlFor="model-id">
                    Model ID
                  </label>
                  <div className="models-page__input-wrap">
                    <Hash size={14} />
                    <input
                      id="model-id"
                      type="text"
                      className="models-page__input models-page__input--inset"
                      placeholder="my-custom-model"
                      value={form.model_id}
                      onChange={(e) => updateForm('model_id', e.target.value)}
                      disabled={submitting}
                    />
                  </div>
                </div>
                <div className="models-page__field">
                  <label className="models-page__field-label" htmlFor="model-category">
                    Category
                  </label>
                  <div className="models-page__input-wrap">
                    <Tag size={14} />
                    <input
                      id="model-category"
                      type="text"
                      className="models-page__input models-page__input--inset"
                      placeholder="chat, embedding…"
                      value={form.category}
                      onChange={(e) => updateForm('category', e.target.value)}
                      disabled={submitting}
                    />
                  </div>
                </div>
              </div>

              <p className="models-page__form-section-label">
                <Globe size={12} strokeWidth={2.5} /> Connection
              </p>

              <div className="models-page__field">
                <label className="models-page__field-label" htmlFor="model-base-url">
                  Base URL
                </label>
                <div className="models-page__input-wrap">
                  <Globe size={14} />
                  <input
                    id="model-base-url"
                    type="text"
                    className="models-page__input models-page__input--inset"
                    placeholder="https://your-endpoint.example.com"
                    value={form.base_url}
                    onChange={(e) => updateForm('base_url', e.target.value)}
                    disabled={submitting}
                  />
                </div>
              </div>

              <div className="models-page__field">
                <label className="models-page__field-label" htmlFor="model-api-key">
                  API Key
                </label>
                <div className="models-page__input-wrap">
                  <Key size={14} />
                  <input
                    id="model-api-key"
                    type="password"
                    className="models-page__input models-page__input--inset"
                    placeholder="Enter API key"
                    value={form.api_key}
                    onChange={(e) => updateForm('api_key', e.target.value)}
                    disabled={submitting}
                  />
                </div>
              </div>

              <p className="models-page__form-section-label">
                <Layers size={12} strokeWidth={2.5} /> Details
              </p>

              <div className="models-page__field">
                <label className="models-page__field-label" htmlFor="model-context">
                  Context Window
                </label>
                <div className="models-page__input-wrap">
                  <Layers size={14} />
                  <input
                    id="model-context"
                    type="number"
                    min={0}
                    className="models-page__input models-page__input--inset"
                    placeholder="e.g. 128000"
                    value={form.context_window || ''}
                    onChange={(e) => updateForm('context_window', Number(e.target.value) || 0)}
                    disabled={submitting}
                  />
                </div>
              </div>

              <div className="models-page__field">
                <label className="models-page__field-label" htmlFor="model-description">
                  <FileText size={12} strokeWidth={2.25} /> Description
                </label>
                <textarea
                  id="model-description"
                  className="models-page__textarea"
                  placeholder="What is this model good at?"
                  rows={3}
                  value={form.description}
                  onChange={(e) => updateForm('description', e.target.value)}
                  disabled={submitting}
                />
              </div>

              {submitError && (
                <p className="models-page__form-error">
                  <AlertCircle size={13} strokeWidth={2.25} /> {submitError}
                </p>
              )}

              <div className="models-page__form-actions">
                <button
                  type="button"
                  className="models-page__btn models-page__btn--outline"
                  onClick={() => setAddModalOpen(false)}
                  disabled={submitting}
                >
                  Cancel
                </button>
                <button type="submit" className="models-page__btn models-page__btn--primary" disabled={!isFormValid || submitting}>
                  {submitting ? (
                    <>
                      <RefreshCw size={13} strokeWidth={2.25} className="models-page__spin" /> Adding…
                    </>
                  ) : (
                    <>
                      <Plus size={13} strokeWidth={2.5} /> Add Model
                    </>
                  )}
                </button>
              </div>
            </form>
          </div>
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

    &--primary {
      background: $primary;
      border-color: $primary;
      color: $on-primary;

      &:hover:not(:disabled) {
        background: $primary-hover;
        border-color: $primary-hover;
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
    font-weight: 700;
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
    flex-direction: column;
    gap: 8px;
    margin-bottom: 10px;
  }

  &__card-stats-row {
    display: flex;
    flex-wrap: wrap;
    gap: 16px 20px;
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

  &__card-foot {
    margin-top: auto;
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

  /* ---------- add-model modal ---------- */
  &__overlay {
    position: fixed;
    inset: 0;
    z-index: 200;
    background: rgba(0, 0, 0, 0.5);
  }

  &__modal {
    position: fixed;
    top: 0;
    right: 0;
    bottom: 0;
    width: 500px;
    max-width: 90vw;
    background: $bg-main;
    border-left: 1px solid $border-subtle;
    box-shadow: $shadow-lg;
    display: flex;
    flex-direction: column;
    overflow: hidden;
    animation: models-page-slide-in 0.24s cubic-bezier(0.16, 1, 0.3, 1);
  }

  &__modal-head {
    flex-shrink: 0;
    position: relative;
    display: flex;
    align-items: center;
    gap: 14px;
    padding: 24px 24px 22px;
    background: linear-gradient(155deg, $primary 0%, $primary-hover 100%);
    overflow: hidden;

    &::before {
      content: '';
      position: absolute;
      top: -40%;
      right: -15%;
      width: 220px;
      height: 220px;
      border-radius: 50%;
      background: rgba(255, 255, 255, 0.08);
      pointer-events: none;
    }

    &::after {
      content: '';
      position: absolute;
      bottom: -60%;
      right: 20%;
      width: 140px;
      height: 140px;
      border-radius: 50%;
      background: rgba(255, 255, 255, 0.06);
      pointer-events: none;
    }
  }

  &__modal-icon {
    position: relative;
    flex-shrink: 0;
    width: 42px;
    height: 42px;
    border-radius: 12px;
    background: rgba(255, 255, 255, 0.16);
    color: $on-primary;
    display: grid;
    place-items: center;
    box-shadow: inset 0 0 0 1px rgba(255, 255, 255, 0.2);
  }

  &__modal-head-text {
    position: relative;
    min-width: 0;
  }

  &__modal-title {
    font-size: 1.0625rem;
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $on-primary;
  }

  &__modal-sub {
    margin-top: 3px;
    font-size: 0.75rem;
    color: rgba(255, 255, 255, 0.8);
  }

  &__modal-close {
    position: relative;
    flex-shrink: 0;
    margin-left: auto;
    width: 30px;
    height: 30px;
    border-radius: 8px;
    border: 1px solid rgba(255, 255, 255, 0.25);
    background: rgba(255, 255, 255, 0.1);
    color: $on-primary;
    display: grid;
    place-items: center;
    cursor: pointer;
    transition: background 0.14s ease, opacity 0.14s ease;

    &:disabled {
      opacity: 0.5;
      cursor: not-allowed;
    }

    &:hover:not(:disabled) {
      background: rgba(255, 255, 255, 0.2);
    }
  }

  /* ---------- form ---------- */
  &__form {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding: 22px 24px 24px;
    display: flex;
    flex-direction: column;
    gap: 12px;
  }

  &__form-section-label {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 0.6875rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    color: $primary;
    margin-top: 6px;

    &:first-child {
      margin-top: 0;
    }

    svg {
      opacity: 0.85;
    }
  }

  &__form-row {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 12px;
  }

  &__field {
    display: flex;
    flex-direction: column;
    gap: 6px;
    min-width: 0;
  }

  &__field-label {
    display: flex;
    align-items: center;
    gap: 5px;
    font-size: 0.71875rem;
    font-weight: 600;
    color: $text-secondary;
  }

  &__input,
  &__textarea {
    width: 100%;
    border: 1px solid $border-default;
    border-radius: 8px;
    padding: 8px 11px;
    font-size: 0.8125rem;
    font-family: $font-body;
    color: $text-primary;
    background: $bg-main;
    transition: border-color 0.14s ease;

    &:focus {
      outline: none;
      border-color: $primary;
    }

    &:disabled {
      opacity: 0.6;
      cursor: not-allowed;
    }

    &::placeholder {
      color: $text-tertiary;
    }
  }

  &__textarea {
    resize: vertical;
    min-height: 4.5rem;
    line-height: 1.5;
    background: $bg-subtle;
    border-color: transparent;

    &:focus {
      background: $bg-main;
    }
  }

  &__input-wrap {
    display: flex;
    align-items: center;
    gap: 9px;
    border: 1px solid transparent;
    border-radius: 8px;
    padding: 0 11px;
    background: $bg-subtle;
    color: $text-tertiary;
    transition: background 0.14s ease, border-color 0.14s ease, color 0.14s ease;

    &:focus-within {
      background: $bg-main;
      border-color: $primary;
      color: $primary;
    }
  }

  &__input--inset {
    border: none;
    padding: 9px 0;
    background: transparent;
  }

  &__form-error {
    display: flex;
    align-items: flex-start;
    gap: 6px;
    font-size: 0.75rem;
    color: $danger;
    background: $danger-subtle;
    border-radius: 8px;
    padding: 9px 11px;
    line-height: 1.45;

    svg {
      flex-shrink: 0;
      margin-top: 1px;
    }
  }

  &__form-actions {
    position: sticky;
    bottom: 0;
    display: flex;
    justify-content: flex-end;
    gap: 8px;
    margin: 8px -24px -24px;
    padding: 16px 24px;
    background: $bg-main;
    border-top: 1px solid $border-subtle;

    .models-page__btn {
      min-width: 6.5rem;
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
    &__card-stats-row {
      flex-wrap: wrap;
      gap: 14px;
    }

    &__form-row {
      grid-template-columns: 1fr;
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
  }
}

@keyframes models-page-spin {
  to {
    transform: rotate(360deg);
  }
}

@keyframes models-page-slide-in {
  from {
    transform: translateX(100%);
  }
  to {
    transform: translateX(0);
  }
}
