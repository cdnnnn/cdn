import { useCallback, useEffect, useRef, useState } from 'react';
import { Search, Check, Plus, Settings, Unlink, Loader2, Cable, Trash2, RefreshCw, Eye, ListPlus, ListFilter, ListChecks } from 'lucide-react';
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

  // Latest-request-wins: if the page-load fetch and a connect/disconnect
  // refresh overlap, only the most recent response is applied.
  const creditsRequestRef = useRef(0);
  const loadOpenRouterCredits = useCallback(() => {
    const requestId = ++creditsRequestRef.current;
    setOpenrouterCreditsStatus('loading');
    usageApi
      .getOpenRouterCredits()
      .then((res) => {
        if (requestId !== creditsRequestRef.current) return;
        setOpenrouterCredits(res);
        setOpenrouterCreditsStatus('succeeded');
      })
      .catch(() => {
        if (requestId !== creditsRequestRef.current) return;
        setOpenrouterCreditsStatus('failed');
      });
  }, []);

  // Normal page load: fetch once the openrouter provider shows up in the list.
  useEffect(() => {
    if (!hasOpenRouterProvider || openrouterCreditsStatus !== 'idle') return;
    loadOpenRouterCredits();
  }, [hasOpenRouterProvider, openrouterCreditsStatus, loadOpenRouterCredits]);

  const connectedCount = items.filter((p) => p.status === 'connected').length;

  const filtered = items.filter((p) => {
    if (filter === 'connected' && p.status !== 'connected') return false;
    if (filter === 'available' && p.status === 'connected') return false;
    return !search || (p.name ?? '').toLowerCase().includes(search.toLowerCase());
  });

  const submitConnect = (providerId: string) => {
    if (!apiKeyInput.trim()) return;
    dispatch(connectProvider({ providerId, payload: { api_key: apiKeyInput } }))
      .unwrap()
      // Refresh the balance only after the connect call has succeeded.
      .then(() => {
        if (providerId === 'openrouter') loadOpenRouterCredits();
      })
      .catch(() => {});
    setKeyPromptFor(null);
    setApiKeyInput('');
  };

  const handleDisconnect = (providerId: string) => {
    dispatch(disconnectProvider(providerId))
      .unwrap()
      // Refresh the balance only after the disconnect call has succeeded.
      .then(() => {
        if (providerId === 'openrouter') loadOpenRouterCredits();
      })
      .catch(() => {});
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

                  {p.id === 'openrouter' && (() => {
                    const credits = openrouterCredits;
                    const isLoading = openrouterCreditsStatus === 'loading';
                    const isReady = openrouterCreditsStatus === 'succeeded' && credits;
                    const pct = isReady && credits.total_credits > 0
                      ? Math.min(100, (credits.total_usage / credits.total_credits) * 100)
                      : 0;
                    const level = pct >= 90 ? 'danger' : pct >= 70 ? 'warn' : 'ok';
                    const stats = [
                      { label: 'Total Credits', value: credits?.total_credits.toLocaleString() },
                      { label: 'Total Usage', value: credits?.total_usage.toLocaleString() },
                      { label: 'Remaining', value: credits?.remaining.toLocaleString(), highlight: true },
                    ];
                    return (
                      <div className={styles['providers__credits']}>
                        {(isLoading || isReady) && (
                          <>
                            <div className={styles['providers__credits-line']}>
                              {stats.map((st) => (
                                <span key={st.label} className={styles['providers__credits-item']}>
                                  <span className={styles['providers__credits-label']}>{st.label}</span>
                                  {isLoading ? (
                                    <span className={styles['providers__credits-skeleton-bar']} />
                                  ) : (
                                    <span
                                      className={`${styles['providers__credits-value']} ${st.highlight ? styles['providers__credits-value--highlight'] : ''}`}
                                    >
                                      {st.value}
                                      <span className={styles['providers__credits-currency']}>{credits?.currency}</span>
                                    </span>
                                  )}
                                </span>
                              ))}
                            </div>
                            <div className={styles['providers__credits-bar-track']}>
                              {isReady && (
                                <div
                                  className={`${styles['providers__credits-bar-fill']} ${styles[`providers__credits-bar-fill--${level}`]}`}
                                  style={{ width: `${pct}%` }}
                                />
                              )}
                            </div>
                          </>
                        )}

                        {openrouterCreditsStatus === 'failed' && (
                          <span className={styles['providers__credits-error']}>Couldn't load credit balance</span>
                        )}
                      </div>
                    );
                  })()}

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
                            onClick={() => handleDisconnect(p.id)}
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
