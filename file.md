import { useEffect, useState } from 'react';
import { Search, Check, Plus, Settings, Unlink, Loader2, Cable, Trash2, RefreshCw, Eye, ListPlus, ListFilter, ListChecks, Wallet } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import {
  fetchProviders,
  createProvider,
  deleteProvider,
  connectProvider,
  disconnectProvider,
  syncModels,
} from '../../store/slices/providersSlice';
import { fetchModelsByProvider, createCustomModel, deleteCustomModel, updateCustomModel } from '../../store/slices/modelsSlice';
import { usageApi, type OpenRouterCreditsResponse } from '../../api/endpoints/usage';
import AddProviderDrawer from './AddProviderDrawer';
import AddCustomModelDrawer from './AddCustomModelDrawer';
import ProviderModelsSidebar from './ProviderModelsSidebar';
import { SkeletonCards } from '../common/Skeleton';
import { useToast } from '../common/Toast';
import styles from './Providers.module.scss';
import type { Provider } from '../../types';

type Filter = 'all' | 'connected' | 'available';

const FILTERS: Filter[] = ['all', 'connected', 'available'];

// Thunks rejected via axios surface as either an Error (network/message) or
// an object with a response payload — normalize both into a display string.
function getErrorMessage(err: unknown, fallback: string): string {
  if (err && typeof err === 'object') {
    const anyErr = err as { response?: { data?: { message?: string; detail?: string } }; message?: string };
    const serverMessage = anyErr.response?.data?.message || anyErr.response?.data?.detail;
    if (serverMessage) return serverMessage;
    if (anyErr.message) return anyErr.message;
  }
  return fallback;
}

export default function Providers() {
  const dispatch = useAppDispatch();
  const { items, status, mutatingId, creating, syncingId } = useAppSelector((s) => s.providers);
  const modelsByProvider = useAppSelector((s) => s.models.byProvider);
  const modelsByProviderStatus = useAppSelector((s) => s.models.byProviderStatus);
  const customModelCreating = useAppSelector((s) => s.models.creating);
  const customModelDeletingId = useAppSelector((s) => s.models.deletingId);
  const customModelUpdatingId = useAppSelector((s) => s.models.updatingId);
  const [search, setSearch] = useState('');
  const [filter, setFilter] = useState<Filter>('all');
  const [keyPromptFor, setKeyPromptFor] = useState<string | null>(null);
  const [apiKeyInput, setApiKeyInput] = useState('');
  const [drawerOpen, setDrawerOpen] = useState(false);
  const [addModelOpen, setAddModelOpen] = useState(false);
  const [viewModelsProvider, setViewModelsProvider] = useState<Provider | null>(null);
  const toast = useToast();

  const [openrouterCredits, setOpenrouterCredits] = useState<OpenRouterCreditsResponse | null>(null);
  const [openrouterCreditsStatus, setOpenrouterCreditsStatus] = useState<'idle' | 'loading' | 'succeeded' | 'failed'>('idle');

  useEffect(() => {
    dispatch(fetchProviders());
  }, [dispatch]);

  // Only the "openrouter" provider gets a credits widget — fetch its balance
  // once that provider shows up in the list, not before.
  const hasOpenRouterProvider = items.some((p) => p.id === 'openrouter');
  useEffect(() => {
    if (!hasOpenRouterProvider || openrouterCreditsStatus !== 'idle') return;
    setOpenrouterCreditsStatus('loading');
    usageApi
      .getOpenRouterCredits()
      .then((res) => {
        setOpenrouterCredits(res);
        setOpenrouterCreditsStatus('succeeded');
      })
      .catch(() => {
        setOpenrouterCreditsStatus('failed');
      });
  }, [hasOpenRouterProvider, openrouterCreditsStatus]);

  const connectedCount = items.filter((p) => p.status === 'connected').length;

  const filtered = items.filter((p) => {
    if (filter === 'connected' && p.status !== 'connected') return false;
    if (filter === 'available' && p.status === 'connected') return false;
    return !search || (p.name ?? '').toLowerCase().includes(search.toLowerCase());
  });

  const submitConnect = (providerId: string) => {
    if (!apiKeyInput.trim()) return;
    dispatch(connectProvider({ providerId, payload: { api_key: apiKeyInput } }));
    setKeyPromptFor(null);
    setApiKeyInput('');
  };

  const openModelsSidebar = (p: Provider) => {
    setViewModelsProvider(p);
    dispatch(fetchModelsByProvider(p.id));
  };

  const customProvider = items.find((p) => p.name === 'Custom') || null;

  return (
    <div className="page-enter pg-shell">
      <div className={styles.providers__header}>
        <div>
          <p className={styles['providers__header-eyebrow']}>Integrations</p>
          <h1>Providers</h1>
          <p className={styles['providers__header-sub']}>Manage your AI provider connections</p>
        </div>
        <div className={styles['providers__header-meta']}>
          <Cable size={13} />
          {connectedCount} of {items.length} connected
        </div>
      </div>

      <div className={styles['providers__toolbar']}>
        <div className={styles['providers__search']}>
          <Search size={16} />
          <input placeholder="Search providers…" value={search} onChange={(e) => setSearch(e.target.value)} />
        </div>

        <div className={styles['providers__toolbar-right']}>
          <div className={styles['providers__filter-group']}>
            <span className={styles['providers__toolbar-label']}>
              <ListFilter size={11} /> Status
            </span>
            {FILTERS.map((f) => (
              <button
                key={f}
                className={`${styles['providers__filter-pill']} ${filter === f ? styles['providers__filter-pill--on'] : ''}`}
                onClick={() => setFilter(f)}
              >
                {f[0].toUpperCase() + f.slice(1)}
              </button>
            ))}
          </div>
          <span className={styles['providers__toolbar-divider']} />
          <button className={styles['providers__add-btn']} onClick={() => setDrawerOpen(true)}>
            <Plus size={14} /> Add Provider
          </button>
        </div>
      </div>

      <div className="pg-body">
        <div className={styles['providers__grid']}>
          {status === 'loading' && <SkeletonCards count={6} />}
          {status !== 'loading' &&
            filtered.map((p) => {
              const isCustom = p.name === 'Custom';
              return (
                <div
                  className={`${styles['providers__card']} ${p.id === 'openrouter' ? styles['providers__card--fit-content'] : ''}`}
                  key={p.id}
                >
                  <div className={styles['providers__card-hdr']}>
                    <div className={styles['providers__card-id']}>
                      <div className={styles['providers__icon']}>
                        {p.logo_url ? <img src={p.logo_url} alt={p.name ?? 'Provider'} /> : (p.name?.[0] ?? '?')}
                      </div>
                      <div style={{ minWidth: 0 }}>
                        <div className={styles['providers__name']}>{p.name ?? 'Unnamed provider'}</div>
                        <div className={styles['providers__count']}>{p.model_count ?? 0} models</div>
                      </div>
                    </div>
                    <div className={styles['providers__card-top-actions']}>
                      <button
                        className={styles['providers__icon-btn']}
                        onClick={() => openModelsSidebar(p)}
                        title={isCustom ? 'Manage models' : 'View models'}
                        aria-label={`${isCustom ? 'Manage' : 'View'} models for ${p.name ?? 'provider'}`}
                      >
                        {isCustom ? <ListChecks size={14} /> : <Eye size={14} />}
                      </button>
                      {p.status === 'connected' ? (
                        <span className={styles['providers__badge-connected']}>
                          <Check size={10} strokeWidth={3} /> Connected
                        </span>
                      ) : (
                        <span className={styles['providers__badge-idle']}>Not connected</span>
                      )}
                    </div>
                  </div>

                  <div className={styles['providers__desc']}>{p.description}</div>

                  {p.id === 'openrouter' && (
                    <div className={styles['providers__credits']}>
                      {openrouterCreditsStatus === 'loading' && (
                        <div className={styles['providers__credits-loading']}>
                          <Wallet size={13} className={styles['providers__credits-loading-icon']} />
                          <span>Fetching balance…</span>
                          <div className={styles['providers__credits-skeleton']}>
                            <div className={styles['providers__credits-skeleton-bar']} />
                            <div className={styles['providers__credits-skeleton-bar']} style={{ width: '58%' }} />
                          </div>
                        </div>
                      )}

                      {openrouterCreditsStatus === 'succeeded' && openrouterCredits && (() => {
                        const pct = openrouterCredits.total_credits > 0
                          ? Math.min(100, (openrouterCredits.total_usage / openrouterCredits.total_credits) * 100)
                          : 0;
                        const level = pct >= 90 ? 'danger' : pct >= 70 ? 'warn' : 'ok';
                        return (
                          <>
                            <div className={styles['providers__credits-hdr']}>
                              <span className={styles['providers__credits-label']}>
                                <Wallet size={12} /> Credits Remaining
                              </span>
                              <span className={styles['providers__credits-remaining']}>
                                {openrouterCredits.remaining.toLocaleString()} {openrouterCredits.currency}
                              </span>
                            </div>
                            <div className={styles['providers__credits-bar-track']}>
                              <div
                                className={`${styles['providers__credits-bar-fill']} ${styles[`providers__credits-bar-fill--${level}`]}`}
                                style={{ width: `${pct}%` }}
                              />
                            </div>
                            <div className={styles['providers__credits-meta']}>
                              <span>{openrouterCredits.total_usage.toLocaleString()} used</span>
                              <span>{openrouterCredits.total_credits.toLocaleString()} total</span>
                            </div>
                          </>
                        );
                      })()}

                      {openrouterCreditsStatus === 'failed' && (
                        <div className={styles['providers__credits-error']}>Couldn't load credit balance.</div>
                      )}
                    </div>
                  )}

                  {keyPromptFor === p.id ? (
                    <div className={styles['providers__key-form']}>
                      <input
                        className={styles['providers__key-input']}
                        type="password"
                        placeholder="Paste API key…"
                        value={apiKeyInput}
                        onChange={(e) => setApiKeyInput(e.target.value)}
                        autoFocus
                      />
                      <div className={styles['providers__key-actions']}>
                        <button
                          className={`${styles['providers__foot-btn']} ${styles['providers__foot-btn--primary']}`}
                          onClick={() => submitConnect(p.id)}
                        >
                          Save
                        </button>
                        <button
                          className={`${styles['providers__foot-btn']} ${styles['providers__foot-btn--ghost']}`}
                          onClick={() => setKeyPromptFor(null)}
                        >
                          Cancel
                        </button>
                      </div>
                    </div>
                  ) : (
                    <div className={styles['providers__foot-actions']}>
                      {isCustom && (
                        <button
                          className={`${styles['providers__foot-btn']} ${styles['providers__foot-btn--accent']}`}
                          onClick={() => setAddModelOpen(true)}
                        >
                          <ListPlus size={13} /> Add Model
                        </button>
                      )}
                      {p.status !== 'connected' && (
                        <button
                          className={`${styles['providers__foot-btn']} ${styles['providers__foot-btn--primary']}`}
                          disabled={mutatingId === p.id}
                          onClick={() => setKeyPromptFor(p.id)}
                        >
                          {mutatingId === p.id ? (
                            <Loader2 size={13} className={styles['providers__spin']} />
                          ) : (
                            <>
                              <Plus size={13} /> Connect
                            </>
                          )}
                        </button>
                      )}
                      {/* Configure button temporarily disabled per request
                      {p.status === 'connected' && (
                        <button
                          className={`${styles['providers__foot-btn']} ${styles['providers__foot-btn--ghost']}`}
                          disabled={mutatingId === p.id}
                          onClick={() => setKeyPromptFor(p.id)}
                        >
                          {mutatingId === p.id ? (
                            <Loader2 size={13} className={styles['providers__spin']} />
                          ) : (
                            <>
                              <Settings size={13} /> Configure
                            </>
                          )}
                        </button>
                      )}
                      */}
                      {p.status === 'connected' && (
                        <>
                          <button
                            className={`${styles['providers__foot-btn']} ${styles['providers__foot-btn--ghost']}`}
                            disabled={syncingId === p.id}
                            onClick={() => dispatch(syncModels(p.id))}
                          >
                            {syncingId === p.id ? (
                              <Loader2 size={13} className={styles['providers__spin']} />
                            ) : (
                              <>
                                <RefreshCw size={13} /> Sync
                              </>
                            )}
                          </button>
                          <button
                            className={`${styles['providers__foot-btn']} ${styles['providers__foot-btn--danger']}`}
                            disabled={mutatingId === p.id}
                            onClick={() => dispatch(disconnectProvider(p.id))}
                          >
                            <Unlink size={13} /> Disconnect
                          </button>
                        </>
                      )}
                      {p.status !== 'connected' && (
                        <button
                          className={`${styles['providers__foot-btn']} ${styles['providers__foot-btn--danger']}`}
                          disabled={mutatingId === p.id}
                          onClick={() => {
                            if (window.confirm(`Delete ${p.name ?? 'this provider'}? This cannot be undone.`)) {
                              dispatch(deleteProvider(p.id));
                            }
                          }}
                        >
                          <Trash2 size={13} /> Delete
                        </button>
                      )}
                    </div>
                  )}
                </div>
              );
            })}
          {status !== 'loading' && filtered.length === 0 && (
            <p className={styles['providers__empty']}>No providers match your search or filter.</p>
          )}
        </div>
      </div>

      {drawerOpen && (
        <AddProviderDrawer
          submitting={creating}
          onClose={() => setDrawerOpen(false)}
          onSubmit={(payload) => {
            dispatch(createProvider(payload)).then(() => setDrawerOpen(false));
          }}
        />
      )}

      {addModelOpen && customProvider && (
        <AddCustomModelDrawer
          mode="create"
          submitting={customModelCreating}
          onClose={() => setAddModelOpen(false)}
          onSubmit={(result) => {
            if (result.kind !== 'create') return;
            dispatch(createCustomModel(result.payload))
              .unwrap()
              .then(() => {
                setAddModelOpen(false);
                toast.success(`"${result.payload.name}" was registered successfully.`, { title: 'Model registered' });
                dispatch(fetchProviders());
                dispatch(fetchModelsByProvider(customProvider.id));
              })
              .catch((err) => {
                toast.error(getErrorMessage(err, 'Failed to register model.'), { title: 'Registration failed' });
              });
          }}
        />
      )}

      {viewModelsProvider && (
        <ProviderModelsSidebar
          provider={viewModelsProvider}
          models={modelsByProvider[viewModelsProvider.id] || []}
          status={modelsByProviderStatus[viewModelsProvider.id] || 'idle'}
          onClose={() => setViewModelsProvider(null)}
          canManage={viewModelsProvider.name === 'Custom'}
          deletingId={customModelDeletingId}
          updatingId={customModelUpdatingId}
          creatingNew={customModelCreating}
          onDelete={(modelId) => {
            dispatch(deleteCustomModel({ modelId, providerId: viewModelsProvider.id }))
              .unwrap()
              .then(() => {
                toast.success('Model removed successfully.', { title: 'Model removed' });
                dispatch(fetchProviders());
              })
              .catch((err) => {
                toast.error(getErrorMessage(err, 'Failed to remove model.'), { title: 'Removal failed' });
              });
          }}
          onEditSubmit={(result) => {
            if (result.kind === 'update') {
              return dispatch(updateCustomModel(result.payload))
                .unwrap()
                .then((res) => {
                  toast.success(`"${res.name}" was updated successfully.`, { title: 'Model updated' });
                  dispatch(fetchProviders());
                  dispatch(fetchModelsByProvider(viewModelsProvider.id));
                })
                .catch((err) => {
                  toast.error(getErrorMessage(err, 'Failed to update model.'), { title: 'Update failed' });
                  throw err;
                });
            }
            // Discovery found a different model id — register it as a new
            // model instead of mutating the one being edited.
            return dispatch(createCustomModel(result.payload))
              .unwrap()
              .then(() => {
                toast.success(`"${result.payload.name}" was registered as a new model.`, { title: 'Model registered' });
                dispatch(fetchProviders());
                dispatch(fetchModelsByProvider(viewModelsProvider.id));
              })
              .catch((err) => {
                toast.error(getErrorMessage(err, 'Failed to register model.'), { title: 'Registration failed' });
                throw err;
              });
          }}
        />
      )}
    </div>
  );
}
















@use '../../styles/_variables' as *;

// ===========================================================================
// Providers — matches the Run Console / Dashboard design system:
// ink/paper palette, ultramarine signal accent, mono instrument labels,
// hover-lift cards. Sidebar block keys are kept stable (shared with
// ProviderModelsSidebar) but recolored to the same tokens.
//
// Neutral/status tokens ($ink, $paper, $card, $line, $signal, $wash, $ok,
// $danger, etc.) now come from the shared "ink" block in _variables.scss
// (theme-aware via _theme.scss custom properties) — only font aliases and
// shadow tokens specific to this file are declared locally below.
//
// Font scaling: `.providers` sets a single base font-size. All descendant
// font-sizes are expressed in `em` (relative to that base), so bumping
// `.providers`'s font-size (e.g. on wide screens) scales the whole
// component proportionally from one place — same convention as Sidebar.
// ===========================================================================

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

$soft: 0 1px 2px rgba(20, 22, 27, 0.05);
$lift: 0 14px 30px -14px rgba(20, 22, 27, 0.22);

// base font-size the providers page's internal `em` scale is built on
$providers-base-font: 0.8125rem;

%micro {
  font-family: $mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

.providers {
  // master scale control — every em-based font-size below responds to this
  font-size: $providers-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  // ---- header -----------------------------------------------------------
  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding: 24px 32px 20px;
    margin-bottom: 20px;
    border-bottom: 1px solid $line;
    background: $card;

    h1 {
      font-family: $display;
      font-size: 1.8462em; // 1.5rem / 0.8125rem
      font-weight: 800;
      letter-spacing: -0.02em;
      color: $ink;
      line-height: 1.2;
    }
  }

  &__header-eyebrow {
    @extend %micro;
    display: flex;
    align-items: center;
    gap: 8px;
    color: $signal;
    margin-bottom: 6px;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $signal;
    }
  }

  &__header-sub {
    margin-top: 4px;
    font-size: 1.0385em; // 0.84375rem / 0.8125rem
    color: $ink-2;
  }

  &__header-meta {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 7px;
    padding: 7px 13px;
    border-radius: 999px;
    border: 1px solid $line;
    background: $paper;
    font-family: $mono;
    font-size: 0.8846em; // 0.71875rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ink-2;
    white-space: nowrap;
    margin-bottom: 3px;
  }

  // ---- toolbar ------------------------------------------------------------
  &__toolbar {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 14px;
    padding: 14px 32px;
    background: $card;
    border-bottom: 1px solid $line;
    flex-wrap: wrap;
  }

  &__search {
    position: relative;
    flex: 1;
    max-width: 340px;
    min-width: 200px;

    svg {
      position: absolute;
      top: 50%;
      left: 13px;
      transform: translateY(-50%);
      color: $ink-3;
      pointer-events: none;
    }

    input {
      width: 100%;
      border: 1.5px solid $line;
      border-radius: 10px;
      padding: 9px 12px 9px 38px;
      font-size: 1.0385em; // 0.84375rem / 0.8125rem
      font-family: $sans;
      color: $ink;
      background: $paper;
      transition: border-color 0.15s ease, background 0.15s ease;

      &::placeholder { color: $ink-3; }
      &:focus {
        outline: none;
        border-color: $signal;
        background: $card;
      }
    }
  }

  &__toolbar-right {
    display: flex;
    align-items: center;
    gap: 14px;
    flex-wrap: wrap;
  }

  &__filter-group {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    padding: 4px;
    background: $paper;
    border: 1px solid $line;
    border-radius: 999px;
  }

  &__toolbar-label {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 5px 10px 5px 11px;
    @extend %micro;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    color: $ink-3;
    white-space: nowrap;
  }

  &__filter-pill {
    padding: 6px 13px;
    border: 0;
    border-radius: 999px;
    background: transparent;
    color: $ink-2;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: all 0.15s ease;

    &:hover { color: $ink; }

    &--on {
      background: $card;
      color: $signal;
      box-shadow: $soft;
    }
  }

  &__toolbar-divider {
    flex-shrink: 0;
    width: 1px;
    height: 26px;
    background: $line;
  }

  &__add-btn {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    padding: 9px 15px;
    border: 1px solid $signal;
    border-radius: 10px;
    background: $signal;
    color: #fff;
    font-family: $sans;
    font-size: 1em; // 0.8125rem / 0.8125rem (base)
    font-weight: 650;
    cursor: pointer;
    box-shadow: $soft;
    transition: background 0.16s ease, border-color 0.16s ease, transform 0.16s ease, box-shadow 0.16s ease;

    &:hover { background: $signal-2; border-color: $signal-2; transform: translateY(-1px); box-shadow: $lift; }
  }

  // ---- provider card grid --------------------------------------------------
  &__grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(380px, 1fr));
    gap: 12px;
  }

  &__card {
    position: relative;
    display: flex;
    flex-direction: column;
    height: 100%;
    padding: 17px 18px;
    border: 1.5px solid $line;
    border-radius: 16px;
    background: $card;
    transition: border-color 0.16s ease, box-shadow 0.16s ease, transform 0.16s ease;

    // Cards still stretch to match the tallest card in their row by default
    // (so a grid of plain provider cards stays uniform, as before). This
    // modifier opts a specific card (the OpenRouter card, which grows for
    // its credits widget) out of that stretch so it can be taller on its
    // own without forcing its row siblings to grow too.
    &--fit-content {
      align-self: start;
      height: auto;
    }

    &:hover {
      border-color: $ink-3;
      box-shadow: $lift;
      transform: translateY(-2px);
    }
  }

  &__card-hdr {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 10px;
  }

  &__card-id {
    display: flex;
    align-items: center;
    gap: 13px;
    min-width: 0;
  }

  &__icon {
    flex-shrink: 0;
    width: 42px;
    height: 42px;
    border-radius: 12px;
    display: grid;
    place-items: center;
    background: $paper;
    border: 1px solid $line;
    color: $ink;
    font-family: $display;
    font-weight: 800;
    font-size: 1.3077em; // 1.0625rem / 0.8125rem

    img { width: 24px; height: 24px; object-fit: contain; }
  }

  &__name {
    font-family: $display;
    font-size: 1.1538em; // 0.9375rem / 0.8125rem
    font-weight: 700;
    color: $ink;
    line-height: 1.25;
  }

  &__count {
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    color: $ink-3;
    margin-top: 2px;
  }

  &__card-top-actions {
    display: flex;
    align-items: center;
    gap: 6px;
    flex-shrink: 0;
  }

  &__icon-btn {
    display: grid;
    place-items: center;
    width: 28px;
    height: 28px;
    border: 1px solid $line;
    border-radius: 8px;
    background: $card;
    color: $ink-2;
    cursor: pointer;
    transition: border-color 0.14s ease, color 0.14s ease, background 0.14s ease;

    &:hover { border-color: $ink-3; color: $ink; background: $paper; }
  }

  &__badge-connected {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 4px 10px 4px 8px;
    border-radius: 999px;
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ok;
    background: $ok-wash;
    white-space: nowrap;

    &::before { content: ''; width: 5px; height: 5px; border-radius: 50%; background: $ok; }
  }

  &__badge-idle {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 4px 11px;
    border-radius: 999px;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    font-weight: 600;
    color: $ink-3;
    background: transparent;
    border: 1px dashed $line;
    white-space: nowrap;

    &::before {
      content: '';
      flex-shrink: 0;
      width: 6px;
      height: 6px;
      border-radius: 50%;
      background: $ink-3;
      opacity: 0.7;
    }
  }

  &__desc {
    flex: 1;
    margin-top: 11px;
    font-size: 1em; // 0.8125rem / 0.8125rem (base)
    color: $ink-2;
    line-height: 1.5;
  }

  // ---- OpenRouter credits widget --------------------------------------------
  &__credits {
    margin-top: 12px;
    padding: 11px 12px;
    border-radius: 11px;
    background: $paper;
    border: 1px solid $line;
  }

  &__credits-loading {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 600;
    color: $ink-3;
    flex-wrap: wrap;
  }

  &__credits-loading-icon {
    animation: providers-credits-pulse 1.3s ease-in-out infinite;
  }

  &__credits-skeleton {
    flex-basis: 100%;
    display: flex;
    flex-direction: column;
    gap: 6px;
    margin-top: 4px;
  }

  &__credits-skeleton-bar {
    height: 8px;
    width: 100%;
    border-radius: 5px;
    background: linear-gradient(90deg, $card 25%, $line 37%, $card 63%);
    background-size: 400px 100%;
    animation: providers-credits-shimmer 1.4s ease-in-out infinite;
  }

  &__credits-hdr {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__credits-label {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    @extend %micro;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    color: $ink-3;
  }

  &__credits-remaining {
    font-family: $mono;
    font-weight: 750;
    font-size: 1.0769em; // 0.875rem / 0.8125rem
    color: $ink;
  }

  &__credits-bar-track {
    margin-top: 8px;
    height: 6px;
    border-radius: 999px;
    background: $card;
    border: 1px solid $line;
    overflow: hidden;
  }

  &__credits-bar-fill {
    height: 100%;
    border-radius: 999px;
    transition: width 0.6s cubic-bezier(0.22, 0.72, 0.16, 1);
    animation: providers-credits-fill-in 0.6s ease both;

    &--ok { background: $ok; }
    &--warn { background: $amber-dark; }
    &--danger { background: $danger; }
  }

  &__credits-meta {
    margin-top: 6px;
    display: flex;
    justify-content: space-between;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    color: $ink-3;
  }

  &__credits-error {
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    color: $danger;
  }

  // ---- inline API key form -------------------------------------------------
  &__key-form {
    display: flex;
    gap: 8px;
    margin-top: 12px;
  }

  &__key-input {
    flex: 1;
    border: 1.5px solid $line;
    border-radius: 9px;
    padding: 8px 11px;
    font-size: 1em; // 0.8125rem / 0.8125rem (base)
    font-family: $sans;
    color: $ink;
    background: $paper;
    transition: border-color 0.15s ease, background 0.15s ease;

    &::placeholder { color: $ink-3; }
    &:focus { outline: none; border-color: $signal; background: $card; }
  }

  &__key-actions {
    display: flex;
    gap: 6px;
    flex-shrink: 0;
  }

  // ---- footer action row ---------------------------------------------------
  &__foot-actions {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 6px;
    margin-top: 13px;
  }

  &__foot-btn {
    flex: 0 0 auto;
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 6px 11px;
    border-radius: 8px;
    border: 1px solid transparent;
    font-family: $sans;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 650;
    white-space: nowrap;
    cursor: pointer;
    transition: background 0.15s ease, border-color 0.15s ease, color 0.15s ease, transform 0.12s ease;

    &:disabled { cursor: not-allowed; opacity: 0.55; }

    &--primary {
      border-color: $signal;
      background: $signal;
      color: #fff;
      &:not(:disabled):hover { background: $signal-2; border-color: $signal-2; transform: translateY(-1px); }
    }

    &--accent {
      background: $signal;
      color: #fff;
      &:not(:disabled):hover { background: $signal-2; transform: translateY(-1px); }
    }

    &--ghost {
      background: $card;
      border-color: $line;
      color: $ink-2;
      &:not(:disabled):hover { border-color: $ink-3; color: $ink; background: $paper; }
    }

    &--danger {
      background: $danger-wash;
      color: $danger;
      &:not(:disabled):hover { background: rgba($danger, 0.16); }
    }
  }

  &__spin { animation: providers-spin 0.8s linear infinite; }

  &__empty {
    grid-column: 1 / -1;
    padding: 40px 20px;
    text-align: center;
    color: $ink-3;
    font-size: 1.0385em; // 0.84375rem / 0.8125rem
    border: 1px dashed $line;
    border-radius: 14px;
  }

  // ===========================================================================
  // Provider models sidebar — keys kept stable for ProviderModelsSidebar,
  // recolored to the ink/paper/signal system.
  // ===========================================================================
  &__sidebar-overlay {
    position: fixed;
    inset: 0;
    background: rgba(20, 22, 27, 0.45);
    z-index: 40;
  }

  &__sidebar {
    position: fixed;
    top: 0;
    right: 0;
    bottom: 0;
    width: min(420px, 100vw);
    background: $card;
    border-left: 1px solid $line;
    box-shadow: -20px 0 40px -16px rgba(20, 22, 27, 0.28);
    z-index: 41;
    display: flex;
    flex-direction: column;
    animation: providers-sidebar-in 0.22s cubic-bezier(0.22, 0.72, 0.16, 1);
  }

  &__sidebar-header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
    padding: 20px 20px 16px;
    border-bottom: 1px solid $line;
  }

  &__sidebar-title {
    font-family: $display;
    font-size: 1.3077em; // 1.0625rem / 0.8125rem
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $ink;
  }

  &__sidebar-subtitle {
    margin-top: 3px;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    color: $ink-3;
  }

  &__sidebar-body {
    flex: 1;
    overflow-y: auto;
    padding: 16px 20px 24px;
    display: flex;
    flex-direction: column;
    gap: 12px;
  }

  &__sidebar-empty {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 8px;
    padding: 48px 12px;
    color: $ink-3;
    font-size: 1em; // 0.8125rem / 0.8125rem (base)
    text-align: center;
  }

  &__model-row {
    border: 1px solid $line;
    border-radius: 12px;
    padding: 14px;
    background: $paper;
    transition: opacity 0.15s ease;

    &--deleting {
      opacity: 0.5;
      pointer-events: none;
    }
  }

  &__model-row-head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__model-row-head-actions {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 6px;
  }

  &__model-row-delete {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 24px;
    height: 24px;
    border-radius: 7px;
    border: 1px solid $line;
    background: $card;
    color: $ink-3;
    cursor: pointer;
    transition: border-color 0.14s ease, color 0.14s ease, background 0.14s ease;

    &:hover:not(:disabled) {
      border-color: rgba($danger, 0.35);
      color: $danger;
      background: $danger-wash;
    }

    &:disabled { cursor: not-allowed; opacity: 0.6; }
  }

  &__model-row-edit {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 24px;
    height: 24px;
    border-radius: 7px;
    border: 1px solid $line;
    background: $card;
    color: $ink-3;
    cursor: pointer;
    transition: border-color 0.14s ease, color 0.14s ease, background 0.14s ease;

    &:hover:not(:disabled) {
      border-color: $signal;
      color: $signal;
      background: $wash;
    }
  }

  &__model-row-desc {
    margin: 6px 0 0;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    line-height: 1.5;
    color: $ink-2;
  }

  &__model-row-name {
    font-family: $display;
    font-weight: 700;
    font-size: 1.0769em; // 0.875rem / 0.8125rem
    color: $ink;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    min-width: 0;
  }

  &__model-row-tags {
    margin-top: 10px;
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  &__tag-group {
    display: flex;
    align-items: baseline;
    flex-wrap: wrap;
    gap: 8px;
  }

  &__tag-group-label {
    @extend %micro;
    flex-shrink: 0;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    color: $ink-3;
  }

  &__tag-group-pills {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
  }

  // Category is a single classifying value — a filled, accent-colored pill
  // makes it read as "this model's category", distinct from the list below.
  &__category-badge {
    display: inline-flex;
    align-items: center;
    padding: 2px 9px;
    border-radius: 999px;
    background: $signal;
    color: #fff;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 700;
  }

  // Capabilities are a set of tags — neutral outline pills keep them
  // visually separate from the single filled category badge above.
  &__capability-badge {
    display: inline-flex;
    align-items: center;
    padding: 2px 9px;
    border-radius: 999px;
    background: $card;
    border: 1px solid $line;
    color: $ink-2;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 600;
  }

  // ---- inline model edit form ------------------------------------------------
  &__model-edit {
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  &__model-edit-field {
    display: flex;
    flex-direction: column;
    gap: 4px;

    label {
      font-size: 0.7692em; // 0.625rem / 0.8125rem
      font-weight: 700;
      letter-spacing: 0.04em;
      text-transform: uppercase;
      color: $ink-3;
    }
  }

  &__model-edit-input,
  &__model-edit-textarea {
    width: 100%;
    border: 1.5px solid $line;
    border-radius: 8px;
    padding: 7px 10px;
    font-size: 1em; // 0.8125rem / 0.8125rem (base)
    font-family: $sans;
    color: $ink;
    background: $card;
    resize: vertical;
    transition: border-color 0.15s ease, box-shadow 0.15s ease;

    &::placeholder { color: $ink-3; }
    &:focus { outline: none; border-color: $signal; box-shadow: 0 0 0 3px $wash; }
    &:disabled { opacity: 0.6; cursor: not-allowed; }
  }

  &__model-edit-actions {
    display: flex;
    justify-content: flex-end;
    gap: 6px;
  }

  &__model-edit-cancel,
  &__model-edit-save {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 6px 11px;
    border-radius: 8px;
    font-family: $sans;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: background 0.15s ease, border-color 0.15s ease, color 0.15s ease;

    &:disabled { cursor: not-allowed; opacity: 0.6; }
  }

  &__model-edit-cancel {
    border: 1px solid $line;
    background: $card;
    color: $ink-2;

    &:hover:not(:disabled) { border-color: $ink-3; color: $ink; }
  }

  &__model-edit-save {
    border: 1px solid $signal;
    background: $signal;
    color: #fff;

    &:hover:not(:disabled) { background: $signal-2; border-color: $signal-2; }
  }

  &__model-row-meta {
    margin-top: 10px;
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 8px 12px;

    > div {
      display: flex;
      flex-direction: column;
      gap: 2px;
      font-size: 1em; // 0.8125rem / 0.8125rem (base)
      color: $ink;
    }
  }

  &__model-row-meta-label {
    @extend %micro;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    color: $ink-3;
  }

  &__model-row-url {
    margin-top: 10px;
    padding-top: 10px;
    border-top: 1px dashed $line;
    font-family: $mono;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    color: $ink-2;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
}

@keyframes providers-spin {
  to { transform: rotate(360deg); }
}

@keyframes providers-credits-shimmer {
  0% { background-position: -200px 0; }
  100% { background-position: 200px 0; }
}

@keyframes providers-credits-pulse {
  0%, 100% { opacity: 0.45; transform: scale(0.92); }
  50% { opacity: 1; transform: scale(1); }
}

@keyframes providers-credits-fill-in {
  from { width: 0; }
}

@keyframes providers-sidebar-in {
  from { transform: translateX(100%); }
  to { transform: translateX(0); }
}

@media (max-width: 768px) {
  .providers__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .providers__toolbar { padding: 12px 18px; }
  .providers__grid { grid-template-columns: 1fr; }
}
