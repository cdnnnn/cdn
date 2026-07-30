import { useMemo, useState, type FC, type FormEvent } from 'react';
import { PlugZap, CheckCircle2, Boxes, Settings2, Unplug, Search, X, Key } from 'lucide-react';
import { PROVIDERS } from '../RunEvaluation/data';
import type { Provider } from '../RunEvaluation/types';
import './Providers.scss';

const Providers: FC = () => {
  const [providers, setProviders] = useState<Provider[]>(PROVIDERS);
  const [query, setQuery] = useState('');
  const [activePanel, setActivePanel] = useState<string | null>(null);
  const [keyInput, setKeyInput] = useState('');

  const connectedCount = providers.filter((p) => p.status === 'connected').length;

  const filtered = useMemo(
    () => providers.filter((p) => !query || p.name.toLowerCase().includes(query.toLowerCase())),
    [providers, query]
  );

  const openPanel = (id: string, existingKey?: string) => {
    setActivePanel((prev) => (prev === id ? null : id));
    setKeyInput(existingKey ?? '');
  };

  const handleSubmit = (e: FormEvent, id: string) => {
    e.preventDefault();
    if (!keyInput.trim()) return;
    setProviders((prev) =>
      prev.map((p) => (p.id === id ? { ...p, status: 'connected', apiKey: `sk-****-${keyInput.slice(-4)}` } : p))
    );
    setActivePanel(null);
  };

  const handleDisconnect = (id: string) => {
    setProviders((prev) => prev.map((p) => (p.id === id ? { ...p, status: 'not_connected', apiKey: undefined } : p)));
    setActivePanel(null);
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

      <div className="providers-page__search">
        <Search size={15} />
        <input type="text" placeholder="Search providers..." value={query} onChange={(e) => setQuery(e.target.value)} />
        {query && (
          <button type="button" className="providers-page__search-clear" onClick={() => setQuery('')} aria-label="Clear search">
            <X size={13} />
          </button>
        )}
      </div>

      <div className="providers-page__body">
        <div className="providers-page__grid">
          {filtered.map((p) => {
            const connected = p.status === 'connected';
            const panelOpen = activePanel === p.id;

            return (
              <div className="providers-page__card" key={p.id}>
                <h3 className="providers-page__name">{p.name}</h3>
                <p className="providers-page__desc">{p.desc}</p>

                <div className="providers-page__tags">
                  <span className={`providers-page__tag${connected ? ' providers-page__tag--jade' : ' providers-page__tag--gray'}`}>
                    {connected && <CheckCircle2 size={11} strokeWidth={2.5} />}
                    {connected ? 'Connected' : 'Not connected'}
                  </span>
                </div>

                <div className="providers-page__stat-row">
                  <div className="providers-page__stat">
                    <span className="providers-page__stat-value n">{p.modelCount}</span>
                    <span className="providers-page__stat-label">Models</span>
                  </div>
                  <div className="providers-page__meta-list">
                    <span className="providers-page__meta-item">
                      <Boxes size={12} /> {p.modelCount} models available
                    </span>
                    {connected && (
                      <span className="providers-page__meta-item">
                        <Key size={12} /> {p.apiKey}
                      </span>
                    )}
                  </div>
                </div>

                {panelOpen && (
                  <form className="providers-page__connect-form" onSubmit={(e) => handleSubmit(e, p.id)}>
                    <label className="providers-page__field-label" htmlFor={`key-${p.id}`}>
                      API Key
                    </label>
                    <input
                      id={`key-${p.id}`}
                      type="password"
                      className="providers-page__input"
                      placeholder="Enter API key"
                      value={keyInput}
                      onChange={(e) => setKeyInput(e.target.value)}
                      autoFocus
                    />
                    <div className="providers-page__form-actions">
                      <button type="button" className="providers-page__btn providers-page__btn--outline" onClick={() => setActivePanel(null)}>
                        Cancel
                      </button>
                      <button type="submit" className="providers-page__btn providers-page__btn--primary">
                        Save
                      </button>
                    </div>
                  </form>
                )}

                {!panelOpen && (
                  <div className="providers-page__actions">
                    {connected ? (
                      <>
                        <button
                          type="button"
                          className="providers-page__btn providers-page__btn--outline"
                          onClick={() => openPanel(p.id, p.apiKey)}
                        >
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
                      <button
                        type="button"
                        className="providers-page__btn providers-page__btn--primary"
                        onClick={() => openPanel(p.id)}
                      >
                        <PlugZap size={13} /> Connect
                      </button>
                    )}
                  </div>
                )}
              </div>
            );
          })}

          {filtered.length === 0 && (
            <div className="providers-page__empty">
              <Search size={22} />
              <p>No providers match your search.</p>
            </div>
          )}
        </div>
      </div>
    </div>
  );
};

export default Providers;



















@use '../../../styles/variables' as *;

.providers-page {
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

  /* ---------- search ---------- */
  &__search {
    flex-shrink: 0;
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
    grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
    gap: 14px;
  }

  &__card {
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

    &--jade {
      color: $success;
      background: $success-subtle;
    }

    &--gray {
      color: $text-tertiary;
      background: $bg-inset;
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
    font-family: $font-body;

    svg {
      flex-shrink: 0;
      color: $text-tertiary;
    }
  }

  /* ---------- inline connect form ---------- */
  &__connect-form {
    margin-top: 2px;
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  &__field-label {
    font-size: 0.71875rem;
    font-weight: 600;
    color: $text-secondary;
  }

  &__input {
    width: 100%;
    border: 1px solid $border-default;
    border-radius: 8px;
    padding: 8px 11px;
    font-size: 0.8125rem;
    font-family: $font-body;
    color: $text-primary;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &::placeholder {
      color: #a8b1bb;
    }

    &:focus {
      outline: none;
      border-color: $primary;
      box-shadow: 0 0 0 3px $primary-light;
    }
  }

  &__form-actions {
    display: flex;
    justify-content: flex-end;
    gap: 8px;
  }

  /* ---------- actions ---------- */
  &__actions {
    display: flex;
    justify-content: flex-end;
    gap: 6px;
    margin-top: 2px;
  }

  &__btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 5px;
    font-family: $font-body;
    font-size: 0.71875rem;
    font-weight: 600;
    padding: 6px 10px;
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
}
