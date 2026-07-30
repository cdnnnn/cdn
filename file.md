import { useMemo, useState, type FC } from 'react';
import { CheckCircle2, PlusCircle, RefreshCw, Trash2 } from 'lucide-react';
import { PROVIDERS, MODELS } from '../RunEvaluation/data';
import './Providers.scss';

const Providers: FC = () => {
  const [activeId, setActiveId] = useState(PROVIDERS[0]?.id ?? null);

  const active = useMemo(() => PROVIDERS.find((p) => p.id === activeId) ?? null, [activeId]);
  const activeModels = useMemo(
    () => (active ? MODELS.filter((m) => m.providerId === active.id) : []),
    [active]
  );
  const usedInEvals = useMemo(
    () => (active ? MODELS.filter((m) => m.providerId === active.id).length : 0),
    [active]
  );

  return (
    <div className="providers">
      <div className="providers__header">
        <div>
          <p className="providers__eyebrow">Workspace</p>
          <h1 className="providers__title">Providers</h1>
          <p className="providers__subtitle">Connect API keys and manage the model catalog for each provider.</p>
        </div>
      </div>

      <div className="providers__split">
        <aside className="providers__list">
          {PROVIDERS.map((p) => {
            const connected = p.status === 'connected';
            return (
              <button
                type="button"
                key={p.id}
                className={`providers__row${active?.id === p.id ? ' providers__row--active' : ''}`}
                onClick={() => setActiveId(p.id)}
              >
                <span className="providers__row-logo">{p.logo}</span>
                <span className="providers__row-info">
                  <span className="providers__row-name">{p.name}</span>
                  <span className="providers__row-sub">
                    {connected ? `${p.modelCount} models` : `${p.modelCount}+ models`}
                  </span>
                </span>
                <span className={`providers__dot${connected ? ' providers__dot--on' : ''}`} />
              </button>
            );
          })}
        </aside>

        <section className="providers__detail">
          {active ? (
            <>
              <div className="providers__detail-head">
                <span className="providers__detail-logo">{active.logo}</span>
                <div>
                  <h2 className="providers__detail-title">{active.name}</h2>
                  <p className="providers__detail-desc">{active.desc}</p>
                </div>
              </div>

              <div className="providers__stats">
                <div className="providers__stat">
                  <span className="providers__stat-value n">
                    {active.status === 'connected' ? active.modelCount : `${active.modelCount}+`}
                  </span>
                  <span className="providers__stat-label">Models available</span>
                </div>
                <div className="providers__stat">
                  <span className="providers__stat-value">
                    {active.status === 'connected' ? 'Connected' : 'Not connected'}
                  </span>
                  <span className="providers__stat-label">Status</span>
                </div>
                <div className="providers__stat">
                  <span className="providers__stat-value n">{usedInEvals}</span>
                  <span className="providers__stat-label">Used in evaluations</span>
                </div>
              </div>

              {active.status === 'connected' ? (
                <div className="providers__field">
                  <label htmlFor="api-key">API Key</label>
                  <div className="providers__field-row">
                    <input id="api-key" type="text" value="sk-••••••••••••••••wX2q" readOnly />
                    <button type="button" className="providers__btn">
                      <RefreshCw size={13} /> Rotate
                    </button>
                    <button type="button" className="providers__btn providers__btn--danger">
                      <Trash2 size={13} /> Remove
                    </button>
                  </div>
                </div>
              ) : (
                <div className="providers__field">
                  <label htmlFor="api-key">API Key</label>
                  <div className="providers__field-row">
                    <input id="api-key" type="text" placeholder="Paste API key…" />
                    <button type="button" className="providers__btn providers__btn--primary">
                      <PlusCircle size={13} /> Connect
                    </button>
                  </div>
                </div>
              )}

              {active.status === 'connected' && activeModels.length > 0 && (
                <>
                  <div className="providers__group-head">
                    <h3>Models from this provider</h3>
                    <div className="providers__group-line" />
                  </div>
                  <div className="providers__table-wrap">
                    <table className="providers__table">
                      <thead>
                        <tr>
                          <th>Model</th>
                          <th>Context</th>
                          <th>Pricing</th>
                          <th>Speed</th>
                        </tr>
                      </thead>
                      <tbody>
                        {activeModels.map((m) => (
                          <tr key={m.id}>
                            <td className="providers__cell-strong">
                              {m.name}
                              {m.id === activeModels[0]?.id && (
                                <span className="providers__badge">
                                  <CheckCircle2 size={11} /> Top pick
                                </span>
                              )}
                            </td>
                            <td className="n">{m.contextWindow}</td>
                            <td className="n">{m.pricing}</td>
                            <td className="n">{m.speedRating}</td>
                          </tr>
                        ))}
                      </tbody>
                    </table>
                  </div>
                </>
              )}
            </>
          ) : (
            <p className="providers__empty">Select a provider to manage its connection.</p>
          )}
        </section>
      </div>
    </div>
  );
};

export default Providers;














@import '../../../styles/variables';

.providers {
  display: flex;
  flex-direction: column;
  gap: 20px;

  &__header {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
  }

  &__eyebrow {
    margin: 0 0 2px;
    font-size: 12px;
    font-weight: 600;
    color: $text-tertiary;
    text-transform: uppercase;
    letter-spacing: 0.04em;
  }

  &__title {
    margin: 0 0 4px;
    font-size: 20px;
    font-weight: 700;
    color: $text-primary;
  }

  &__subtitle {
    margin: 0;
    font-size: 13.5px;
    color: $text-secondary;
  }

  &__split {
    display: grid;
    grid-template-columns: 300px 1fr;
    border: 1px solid $border-default;
    border-radius: $radius-lg;
    overflow: hidden;
    background: $bg-main;
    min-height: 560px;
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
    align-items: center;
    gap: 12px;
    text-align: left;
    padding: 13px 16px;
    border: none;
    border-bottom: 1px solid $border-subtle;
    background: transparent;
    cursor: pointer;

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

  &__row-logo {
    width: 30px;
    height: 30px;
    flex-shrink: 0;
    border-radius: $radius-sm;
    background: $bg-inset;
    color: $text-secondary;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 11.5px;
    font-weight: 700;
  }

  &__row-info {
    flex: 1;
    display: flex;
    flex-direction: column;
    gap: 2px;
    min-width: 0;
  }

  &__row-name {
    font-size: 13px;
    font-weight: 600;
    color: $text-primary;
  }

  &__row-sub {
    font-size: 11px;
    color: $text-tertiary;
  }

  &__dot {
    width: 7px;
    height: 7px;
    border-radius: 50%;
    background: $border-strong;
    flex-shrink: 0;

    &--on {
      background: $success;
    }
  }

  /* ---------- right detail ---------- */
  &__detail {
    padding: 26px 30px;
    overflow-y: auto;
    min-height: 0;
  }

  &__detail-head {
    display: flex;
    align-items: flex-start;
    gap: 16px;
    margin-bottom: 20px;
  }

  &__detail-logo {
    width: 48px;
    height: 48px;
    flex-shrink: 0;
    border-radius: $radius-md;
    background: $bg-inset;
    color: $text-secondary;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 16px;
    font-weight: 700;
  }

  &__detail-title {
    margin: 0 0 4px;
    font-size: 17px;
    font-weight: 700;
    color: $text-primary;
  }

  &__detail-desc {
    margin: 0;
    font-size: 12.5px;
    color: $text-tertiary;
    max-width: 440px;
  }

  &__stats {
    display: flex;
    border: 1px solid $border-default;
    border-radius: $radius-md;
    overflow: hidden;
    margin-bottom: 22px;
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
    font-size: 18px;
    font-weight: 700;
    color: $text-primary;
  }

  &__stat-label {
    font-size: 11px;
    color: $text-tertiary;
    text-transform: uppercase;
    letter-spacing: 0.03em;
  }

  /* ---------- api key field ---------- */
  &__field {
    margin-bottom: 20px;

    label {
      display: block;
      font-size: 12px;
      font-weight: 600;
      color: $text-secondary;
      margin-bottom: 6px;
    }
  }

  &__field-row {
    display: flex;
    gap: 8px;

    input {
      flex: 1;
      border: 1px solid $border-default;
      border-radius: $radius-sm;
      padding: 9px 12px;
      font-size: 13px;
      font-family: monospace;
      color: $text-tertiary;
      background: $bg-subtle;

      &::placeholder {
        color: $text-tertiary;
      }

      &:focus {
        outline: none;
        border-color: $primary;
      }
    }
  }

  &__btn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    border: 1px solid $border-default;
    background: $bg-main;
    color: $text-secondary;
    padding: 8px 13px;
    border-radius: $radius-sm;
    font-size: 12.5px;
    font-weight: 600;
    cursor: pointer;
    white-space: nowrap;

    &:hover {
      background: $bg-inset;
    }

    &--primary {
      background: $primary;
      border-color: $primary;
      color: #fff;

      &:hover {
        background: $primary-hover;
      }
    }

    &--danger {
      color: $danger;

      &:hover {
        background: $danger-subtle;
      }
    }
  }

  /* ---------- model catalog ---------- */
  &__group-head {
    display: flex;
    align-items: center;
    gap: 10px;
    margin: 26px 0 12px;

    h3 {
      margin: 0;
      font-size: 12.5px;
      text-transform: uppercase;
      letter-spacing: 0.04em;
      color: $text-tertiary;
      white-space: nowrap;
    }
  }

  &__group-line {
    flex: 1;
    height: 1px;
    background: $border-default;
  }

  &__table-wrap {
    overflow-x: auto;
  }

  &__table {
    width: 100%;
    border-collapse: collapse;
    font-size: 12.5px;

    th {
      text-align: left;
      padding: 9px 12px;
      color: $text-tertiary;
      font-size: 11px;
      font-weight: 600;
      text-transform: uppercase;
      letter-spacing: 0.03em;
      border-bottom: 1px solid $border-default;
    }

    td {
      padding: 10px 12px;
      border-bottom: 1px solid $border-subtle;
      color: $text-secondary;
    }

    tr:last-child td {
      border-bottom: none;
    }
  }

  &__cell-strong {
    font-weight: 600;
    color: $text-primary;
    display: flex;
    align-items: center;
    gap: 8px;
  }

  &__badge {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    background: $primary-light;
    color: $primary;
    font-size: 10.5px;
    font-weight: 600;
    padding: 2px 8px;
    border-radius: 999px;
  }

  &__empty {
    padding: 24px;
    text-align: center;
    color: $text-tertiary;
    font-size: 13px;
  }
}
