//Providers.tsx
import { useMemo, useState, type FC, type FormEvent } from 'react';
import { PlugZap, CheckCircle2, Boxes, Settings2, Unplug, Search, X, Key, Gauge } from 'lucide-react';
import { PROVIDERS, MODELS } from '../RunEvaluation/data';
import type { Provider } from '../RunEvaluation/types';
import './Providers.scss';

const TINTS = ['blue', 'violet', 'amber', 'jade', 'rose'] as const;

function tintFor(id: string) {
  let hash = 0;
  for (let i = 0; i < id.length; i += 1) hash = (hash * 31 + id.charCodeAt(i)) >>> 0;
  return TINTS[hash % TINTS.length];
}

const MODEL_PREVIEW_LIMIT = 5;

const Providers: FC = () => {
  const [providers, setProviders] = useState<Provider[]>(PROVIDERS);
  const [query, setQuery] = useState('');
  const [modelsModalFor, setModelsModalFor] = useState<Provider | null>(null);
  const [connectModalFor, setConnectModalFor] = useState<Provider | null>(null);
  const [keyInput, setKeyInput] = useState('');

  const connectedCount = providers.filter((p) => p.status === 'connected').length;

  const filtered = useMemo(
    () => providers.filter((p) => !query || p.name.toLowerCase().includes(query.toLowerCase())),
    [providers, query]
  );

  const modelsFor = (providerId: string) => MODELS.filter((m) => m.providerId === providerId);

  const avgAccuracyFor = (providerId: string) => {
    const models = modelsFor(providerId);
    if (models.length === 0) return null;
    const sum = models.reduce((acc, m) => acc + m.accuracyScore, 0);
    return (sum / models.length).toFixed(1);
  };

  const openConnect = (p: Provider) => {
    setKeyInput(p.apiKey ?? '');
    setConnectModalFor(p);
  };

  const handleSubmit = (e: FormEvent) => {
    e.preventDefault();
    if (!connectModalFor || !keyInput.trim()) return;
    setProviders((prev) =>
      prev.map((p) => (p.id === connectModalFor.id ? { ...p, status: 'connected', apiKey: `sk-****-${keyInput.slice(-4)}` } : p))
    );
    setConnectModalFor(null);
  };

  const handleDisconnect = (id: string) => {
    setProviders((prev) => prev.map((p) => (p.id === id ? { ...p, status: 'not_connected', apiKey: undefined } : p)));
  };

  return (
    <div className="providers-page">
      <div className="providers-page__header">
        <div className="providers-page__header-left">
          <p className="providers-page__header-eyebrow">Provider connections</p>
          <h1 className="providers-page__title">Providers</h1>
          <p className="providers-page__subtitle">Manage AI service connections and API keys</p>
        </div>

        <div className="providers-page__header-meta">
          <PlugZap size={13} />
          {connectedCount} of {providers.length} connected
        </div>
      </div>

      <div className="providers-page__filters">
        <div className="providers-page__search">
          <Search size={15} />
          <input type="text" placeholder="Search providers..." value={query} onChange={(e) => setQuery(e.target.value)} />
          {query && (
            <button type="button" className="providers-page__search-clear" onClick={() => setQuery('')} aria-label="Clear search">
              <X size={13} />
            </button>
          )}
        </div>
      </div>

      {filtered.length === 0 ? (
        <div className="providers-page__empty">
          <Search size={22} />
          <p>No providers match your search.</p>
        </div>
      ) : (
        <div className="providers-page__grid">
          {filtered.map((p) => {
            const connected = p.status === 'connected';
            const tint = tintFor(p.id);
            const models = modelsFor(p.id);
            const avgAccuracy = avgAccuracyFor(p.id);

            return (
              <div className="providers-page__card" key={p.id}>
                <div className="providers-page__card-top">
                  <div className="providers-page__card-top-left">
                    <span className={`providers-page__avatar providers-page__avatar--${tint}`}>{p.logo}</span>
                    <span className="providers-page__card-name">{p.name}</span>
                  </div>
                  <span className={`providers-page__tag${connected ? ' providers-page__tag--jade' : ' providers-page__tag--gray'}`}>
                    {connected && <CheckCircle2 size={11} strokeWidth={2.5} />}
                    {connected ? 'Connected' : 'Not connected'}
                  </span>
                </div>

                <p className="providers-page__card-desc">{p.desc}</p>

                <div className="providers-page__card-stats">
                  <div className="providers-page__card-stat">
                    <span className="providers-page__card-stat-label">
                      <Boxes size={10} /> Models
                    </span>
                    <span className="providers-page__card-stat-value n">{p.modelCount}</span>
                  </div>
                  <div className="providers-page__card-stat">
                    <span className="providers-page__card-stat-label">
                      <Gauge size={10} /> Avg. Accuracy
                    </span>
                    <span className="providers-page__card-stat-value providers-page__card-stat-value--accent n">
                      {avgAccuracy ? `${avgAccuracy}%` : '—'}
                    </span>
                  </div>
                  <div className="providers-page__card-stat">
                    <span className="providers-page__card-stat-label">
                      <Key size={10} /> API Key
                    </span>
                    <span className="providers-page__card-stat-value providers-page__card-stat-value--sm">
                      {connected ? p.apiKey : '—'}
                    </span>
                  </div>
                </div>

                {models.length > 0 && (
                  <>
                    <span className="providers-page__card-section-label">Models</span>
                    <div className="providers-page__card-models">
                      {models.slice(0, MODEL_PREVIEW_LIMIT).map((m) => (
                        <div className="providers-page__card-model-row" key={m.id}>
                          <span className="providers-page__card-model-name">{m.name}</span>
                          <span className="providers-page__card-model-accuracy n">{m.accuracyScore}%</span>
                        </div>
                      ))}
                    </div>
                    {models.length > MODEL_PREVIEW_LIMIT && (
                      <button type="button" className="providers-page__card-view-all" onClick={() => setModelsModalFor(p)}>
                        View all {models.length} models
                      </button>
                    )}
                  </>
                )}

                <div className="providers-page__card-foot">
                  {connected ? (
                    <>
                      <button type="button" className="providers-page__btn providers-page__btn--outline" onClick={() => openConnect(p)}>
                        <Settings2 size={13} /> Configure
                      </button>
                      <button
                        type="button"
                        className="providers-page__btn providers-page__btn--danger-outline"
                        onClick={() => handleDisconnect(p.id)}
                      >
                        <Unplug size={13} /> Disconnect
                      </button>
                    </>
                  ) : (
                    <button type="button" className="providers-page__btn providers-page__btn--primary" onClick={() => openConnect(p)}>
                      <PlugZap size={13} /> Connect
                    </button>
                  )}
                </div>
              </div>
            );
          })}
        </div>
      )}

      {modelsModalFor && (
        <div className="providers-page__overlay" onClick={() => setModelsModalFor(null)}>
          <div className="providers-page__modal" onClick={(e) => e.stopPropagation()}>
            <div className="providers-page__modal-head">
              <div>
                <span className={`providers-page__avatar providers-page__avatar--${tintFor(modelsModalFor.id)}`}>
                  {modelsModalFor.logo}
                </span>
                <h2 className="providers-page__modal-title">{modelsModalFor.name}</h2>
                <p className="providers-page__modal-sub">{modelsFor(modelsModalFor.id).length} models</p>
              </div>
              <button type="button" className="providers-page__modal-close" onClick={() => setModelsModalFor(null)} aria-label="Close">
                <X size={16} />
              </button>
            </div>

            <div className="providers-page__modal-body">
              {modelsFor(modelsModalFor.id).map((m) => (
                <div className="providers-page__card-model-row" key={m.id}>
                  <span className="providers-page__card-model-name">
                    {m.name} <span className="providers-page__card-model-version">{m.version}</span>
                  </span>
                  <span className="providers-page__card-model-accuracy n">{m.accuracyScore}%</span>
                </div>
              ))}
            </div>
          </div>
        </div>
      )}

      {connectModalFor && (
        <div className="providers-page__overlay" onClick={() => setConnectModalFor(null)}>
          <div className="providers-page__modal providers-page__modal--sm" onClick={(e) => e.stopPropagation()}>
            <div className="providers-page__modal-head">
              <div>
                <span className={`providers-page__avatar providers-page__avatar--${tintFor(connectModalFor.id)}`}>
                  {connectModalFor.logo}
                </span>
                <h2 className="providers-page__modal-title">{connectModalFor.name}</h2>
                <p className="providers-page__modal-sub">{connectModalFor.status === 'connected' ? 'Update API key' : 'Connect provider'}</p>
              </div>
              <button type="button" className="providers-page__modal-close" onClick={() => setConnectModalFor(null)} aria-label="Close">
                <X size={16} />
              </button>
            </div>

            <form className="providers-page__connect-form" onSubmit={handleSubmit}>
              <label className="providers-page__field-label" htmlFor="provider-key">
                API Key
              </label>
              <div className="providers-page__input-wrap">
                <Key size={14} />
                <input
                  id="provider-key"
                  type="password"
                  className="providers-page__input"
                  placeholder="Enter API key"
                  value={keyInput}
                  onChange={(e) => setKeyInput(e.target.value)}
                  autoFocus
                />
              </div>
              <div className="providers-page__form-actions">
                <button type="button" className="providers-page__btn providers-page__btn--outline" onClick={() => setConnectModalFor(null)}>
                  Cancel
                </button>
                <button type="submit" className="providers-page__btn providers-page__btn--primary">
                  Save
                </button>
              </div>
            </form>
          </div>
        </div>
      )}
    </div>
  );
};

export default Providers;

















//Providers.scss
@use '../../../styles/variables' as *;

.providers-page {
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

  /* ---------- filters ---------- */
  &__filters {
    flex-shrink: 0;
    display: flex;
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

  /* ---------- avatar ---------- */
  &__avatar {
    flex-shrink: 0;
    width: 34px;
    height: 34px;
    border-radius: 10px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    font-size: 0.75rem;
    font-weight: 700;
    letter-spacing: -0.01em;

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

  /* ---------- status tag ---------- */
  &__tag {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 4px;
    font-size: 0.6875rem;
    font-weight: 600;
    border-radius: 999px;
    padding: 3px 10px;

    &--jade {
      color: $success;
      background: $success-subtle;
    }

    &--gray {
      color: $text-tertiary;
      background: $bg-inset;
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
    align-items: center;
    justify-content: space-between;
    gap: 10px;
    margin-bottom: 10px;
  }

  &__card-top-left {
    display: flex;
    align-items: center;
    gap: 10px;
    min-width: 0;
  }

  &__card-name {
    font-size: 0.9375rem;
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $text-primary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
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
    display: flex;
    align-items: center;
    gap: 4px;
    font-size: 0.625rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.04em;
    color: $text-tertiary;

    svg {
      opacity: 0.8;
    }
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

  /* ---------- models list (card preview + modal) ---------- */
  &__card-models {
    display: flex;
    flex-direction: column;
    gap: 4px;
    margin-bottom: 6px;
  }

  &__card-model-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
    padding: 5px 0;
    font-size: 0.75rem;
  }

  &__card-model-name {
    color: $text-primary;
    font-weight: 600;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__card-model-version {
    font-family: $font-mono;
    font-size: 0.6875rem;
    font-weight: 400;
    color: $text-tertiary;
  }

  &__card-model-accuracy {
    flex-shrink: 0;
    font-weight: 700;
    color: $success;
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
    margin-bottom: 10px;
    cursor: pointer;

    &:hover {
      text-decoration: underline;
    }
  }

  /* ---------- card footer / actions ---------- */
  &__card-foot {
    margin-top: auto;
    padding-top: 10px;
    border-top: 1px solid $border-subtle;
    display: flex;
    gap: 8px;
  }

  &__btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 6px;
    font-family: $font-body;
    font-size: 0.78125rem;
    font-weight: 600;
    padding: 7px 12px;
    border-radius: 8px;
    border: 1px solid transparent;
    cursor: pointer;
    transition: background 0.14s ease, border-color 0.14s ease, color 0.14s ease;
    flex: 1;

    &--outline {
      background: $bg-main;
      border-color: $border-default;
      color: $text-primary;

      &:hover {
        border-color: $text-primary;
      }
    }

    &--danger-outline {
      background: $bg-main;
      border-color: $border-default;
      color: $text-tertiary;

      &:hover {
        border-color: $danger;
        color: $danger;
        background: $danger-subtle;
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

  /* ---------- empty state ---------- */
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
  }

  /* ---------- modals (view-all-models + connect form) ---------- */
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

    &--sm {
      max-width: 24rem;
      max-height: none;
    }
  }

  &__modal-head {
    flex-shrink: 0;
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
    padding: 20px 22px 16px;
    border-bottom: 1px solid $border-subtle;

    .providers-page__avatar {
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
    padding: 8px 22px 22px;
    display: flex;
    flex-direction: column;
  }

  /* ---------- connect form (inside its own modal) ---------- */
  &__connect-form {
    padding: 20px 22px 22px;
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  &__field-label {
    font-size: 0.71875rem;
    font-weight: 600;
    color: $text-secondary;
  }

  &__input-wrap {
    display: flex;
    align-items: center;
    gap: 8px;
    border: 1px solid $border-default;
    border-radius: 8px;
    padding: 0 11px;
    background: $bg-main;
    color: $text-tertiary;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &:focus-within {
      border-color: $primary;
      box-shadow: 0 0 0 3px $primary-light;
    }
  }

  &__input {
    flex: 1;
    width: 100%;
    border: none;
    outline: none;
    padding: 8px 0;
    font-size: 0.8125rem;
    font-family: $font-body;
    color: $text-primary;
    background: transparent;

    &::placeholder {
      color: $text-tertiary;
    }
  }

  &__form-actions {
    display: flex;
    justify-content: flex-end;
    gap: 8px;
    margin-top: 6px;
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

    &__card-stat-label {
      font-size: 0.6875rem;
    }

    &__card-section-label {
      font-size: 0.71875rem;
    }

    &__card-model-name {
      font-size: 0.8125rem;
    }
  }
}
