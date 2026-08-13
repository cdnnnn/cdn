import { useEffect, useState } from 'react';
import { Search, Check, Plus, Settings, Unlink, Loader2, Cable, Trash2, RefreshCw, Eye, ListPlus, ListFilter } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import {
  fetchProviders,
  createProvider,
  deleteProvider,
  connectProvider,
  disconnectProvider,
  syncModels,
} from '../../store/slices/providersSlice';
import { fetchModelsByProvider, createCustomModel } from '../../store/slices/modelsSlice';
import AddProviderDrawer from './AddProviderDrawer';
import AddCustomModelDrawer from './AddCustomModelDrawer';
import ProviderModelsSidebar from './ProviderModelsSidebar';
import { SkeletonCards } from '../common/Skeleton';
import styles from './Providers.module.scss';
import type { Provider } from '../../types';

type Filter = 'all' | 'connected' | 'available';

const FILTERS: Filter[] = ['all', 'connected', 'available'];

export default function Providers() {
  const dispatch = useAppDispatch();
  const { items, status, mutatingId, creating, syncingId } = useAppSelector((s) => s.providers);
  const modelsByProvider = useAppSelector((s) => s.models.byProvider);
  const modelsByProviderStatus = useAppSelector((s) => s.models.byProviderStatus);
  const customModelCreating = useAppSelector((s) => s.models.creating);
  const [search, setSearch] = useState('');
  const [filter, setFilter] = useState<Filter>('all');
  const [keyPromptFor, setKeyPromptFor] = useState<string | null>(null);
  const [apiKeyInput, setApiKeyInput] = useState('');
  const [drawerOpen, setDrawerOpen] = useState(false);
  const [addModelOpen, setAddModelOpen] = useState(false);
  const [viewModelsProvider, setViewModelsProvider] = useState<Provider | null>(null);

  useEffect(() => {
    dispatch(fetchProviders());
  }, [dispatch]);

  const connectedCount = items.filter((p) => p.status === 'connected').length;

  const filtered = items.filter((p) => {
    if (filter === 'connected' && p.status !== 'connected') return false;
    if (filter === 'available' && p.status === 'connected') return false;
    return !search || p.name.toLowerCase().includes(search.toLowerCase());
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
                <div className={styles['providers__card']} key={p.id}>
                  <div className={styles['providers__card-hdr']}>
                    <div className={styles['providers__card-id']}>
                      <div className={styles['providers__icon']}>
                        {p.logo_url ? <img src={p.logo_url} alt={p.name} /> : p.name[0]}
                      </div>
                      <div style={{ minWidth: 0 }}>
                        <div className={styles['providers__name']}>{p.name}</div>
                        <div className={styles['providers__count']}>{p.model_count} models</div>
                      </div>
                    </div>
                    <div className={styles['providers__card-top-actions']}>
                      <button
                        className={styles['providers__icon-btn']}
                        onClick={() => openModelsSidebar(p)}
                        title="View models"
                        aria-label={`View models for ${p.name}`}
                      >
                        <Eye size={14} />
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
                            if (window.confirm(`Delete ${p.name}? This cannot be undone.`)) {
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
          submitting={customModelCreating}
          onClose={() => setAddModelOpen(false)}
          onSubmit={(payload) => {
            dispatch(createCustomModel(payload)).then(() => {
              setAddModelOpen(false);
              dispatch(fetchProviders());
              dispatch(fetchModelsByProvider(customProvider.id));
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
        />
      )}
    </div>
  );
}
