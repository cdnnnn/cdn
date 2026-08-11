//Providers.tsx
import { useEffect, useState } from 'react';
import { Search, Check, Plus, Settings, Unlink, Loader2, Cable, Trash2, RefreshCw, Eye, ListPlus } from 'lucide-react';
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
      <div className="pg-toolbar">
        <div className="toolbar">
          <div className="search-box">
            <Search size={16} color="var(--text-muted)" />
            <input placeholder="Search providers…" value={search} onChange={(e) => setSearch(e.target.value)} />
          </div>
          <div style={{ display: 'flex', gap: 8, alignItems: 'center' }}>
            <div className="pills">
              {(['all', 'connected', 'available'] as Filter[]).map((f) => (
                <button key={f} className={`pill ${filter === f ? 'on' : ''}`} onClick={() => setFilter(f)}>
                  {f[0].toUpperCase() + f.slice(1)}
                </button>
              ))}
            </div>
            <button className="btn btn-ind btn-sm" onClick={() => setDrawerOpen(true)}><Plus size={14} /> Add Provider</button>
          </div>
        </div>
      </div>
      <div className="pg-body">
        <div className="cards-grid">
          {status === 'loading' && <SkeletonCards count={6} />}
          {status !== 'loading' && filtered.map((p) => {
            const isCustom = p.name === 'Custom';
            return (
              <div className="card" key={p.id} style={{ display: 'flex', flexDirection: 'column', height: '100%' }}>
                <div className="card-hdr">
                  <div style={{ display: 'flex', alignItems: 'center', gap: 14 }}>
                    <div className={`card-icon ${styles['providers__icon']}`}>{p.logo_url ? <img src={p.logo_url} alt={p.name} /> : p.name[0]}</div>
                    <div>
                      <div className="card-title">{p.name}</div>
                      <div style={{ fontSize: 12, color: 'var(--text-secondary)', marginTop: 2 }}>{p.model_count} models</div>
                    </div>
                  </div>
                  <div style={{ display: 'flex', alignItems: 'center', gap: 6 }}>
                    <button
                      className={`btn btn-sm btn-ghost ${styles['providers__icon-btn']}`}
                      onClick={() => openModelsSidebar(p)}
                      title="View models"
                      aria-label={`View models for ${p.name}`}
                    >
                      <Eye size={14} />
                    </button>
                    {p.status === 'connected' ? (
                      <span className="badge badge-green"><Check size={11} /> Connected</span>
                    ) : (
                      <span className={styles['providers__badge-idle']}>Not connected</span>
                    )}
                  </div>
                </div>
                <div className="card-desc" style={{ flex: 1 }}>{p.description}</div>

                {keyPromptFor === p.id ? (
                  <div className={styles['providers__key-form']}>
                    <input
                      className="fi"
                      type="password"
                      placeholder="Paste API key…"
                      value={apiKeyInput}
                      onChange={(e) => setApiKeyInput(e.target.value)}
                      autoFocus
                    />
                    <div className={styles['providers__key-actions']}>
                      <button className="btn btn-sm btn-ind" onClick={() => submitConnect(p.id)}>Save</button>
                      <button className="btn btn-sm btn-ghost" onClick={() => setKeyPromptFor(null)}>Cancel</button>
                    </div>
                  </div>
                ) : (
                  <div className={styles['providers__foot-actions']}>
                    {isCustom && (
                      <button
                        className={`btn btn-sm btn-ind ${styles['providers__foot-btn']}`}
                        onClick={() => setAddModelOpen(true)}
                      >
                        <ListPlus size={13} /> Add Model
                      </button>
                    )}
                    <button
                      className={`btn btn-sm ${p.status === 'connected' ? 'btn-ghost' : 'btn-ind'} ${styles['providers__foot-btn']}`}
                      disabled={mutatingId === p.id}
                      onClick={() => setKeyPromptFor(p.id)}
                    >
                      {mutatingId === p.id ? (
                        <Loader2 size={13} style={{ animation: 'spin 1.5s linear infinite' }} />
                      ) : p.status === 'connected' ? (
                        <><Settings size={13} /> Configure</>
                      ) : (
                        <><Plus size={13} /> Connect</>
                      )}
                    </button>
                    {p.status === 'connected' && (
                      <>
                        <button
                          className={`btn btn-sm btn-ghost ${styles['providers__foot-btn']}`}
                          disabled={syncingId === p.id}
                          onClick={() => dispatch(syncModels(p.id))}
                        >
                          {syncingId === p.id ? (
                            <Loader2 size={13} style={{ animation: 'spin 1.5s linear infinite' }} />
                          ) : (
                            <><RefreshCw size={13} /> Sync</>
                          )}
                        </button>
                        <button
                          className={`btn btn-sm btn-danger ${styles['providers__foot-btn']}`}
                          disabled={mutatingId === p.id}
                          onClick={() => dispatch(disconnectProvider(p.id))}
                        >
                          <Unlink size={13} /> Disconnect
                        </button>
                      </>
                    )}
                    {p.status !== 'connected' && (
                      <button
                        className={`btn btn-sm btn-danger ${styles['providers__foot-btn']}`}
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

















//Providers.module.scss
@use '../../styles/_variables' as *;

.providers {
  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding: 24px 32px 18px;
    margin-bottom: 24px;
    border-bottom: 1px solid $border-light;

    h1 {
      font-family: $font-display;
      font-size: 1.5rem;
      font-weight: 800;
      letter-spacing: -0.02em;
      color: $text-primary;
      line-height: 1.2;
    }
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
    color: $indigo;
    margin-bottom: 6px;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $indigo;
    }
  }

  &__header-sub {
    margin-top: 4px;
    font-size: 0.875rem;
    color: $text-secondary;
  }

  &__header-meta {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-secondary;
    background: $surface-alt;
    border: 1px solid $border-light;
    border-radius: 999px;
    padding: 7px 13px;
    white-space: nowrap;
    margin-bottom: 3px;
  }

  &__loading { display: flex; align-items: center; gap: 8px; color: $text-secondary; font-size: 13px; margin-bottom: 16px; }
  &__icon {
    background: $indigo-pale; color: $indigo; font-size: 18px; font-weight: 700;
    img { width: 24px; height: 24px; object-fit: contain; }
  }
  &__key-form { display: flex; gap: 8px; margin-top: 4px; }
  &__key-actions { display: flex; gap: 6px; }

  &__badge-idle {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 4px 11px;
    border-radius: 999px;
    font-size: 0.6875rem;
    font-weight: 600;
    color: $text-muted;
    background: transparent;
    border: 1px dashed $border-light;
    white-space: nowrap;

    &::before {
      content: '';
      flex-shrink: 0;
      width: 6px;
      height: 6px;
      border-radius: 50%;
      background: $text-muted;
      opacity: 0.6;
    }
  }

  &__icon-btn {
    padding: 6px !important;
    min-width: auto;
    border-radius: 8px;
  }

  // Action row at the bottom of each provider card. Wraps so buttons never
  // overflow the card, with tight, even spacing between them.
  &__foot-actions {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 6px;
    margin-top: 12px;
  }

  &__foot-btn {
    flex: 0 0 auto;
    padding: 6px 10px !important;
    font-size: 12.5px !important;
    line-height: 1.2;
    gap: 4px !important;
    white-space: nowrap;
  }

  // --- Provider models sidebar ---
  &__sidebar-overlay {
    position: fixed;
    inset: 0;
    background: rgba(15, 18, 26, 0.45);
    z-index: 40;
  }

  &__sidebar {
    position: fixed;
    top: 0;
    right: 0;
    bottom: 0;
    width: min(420px, 100vw);
    background: var(--surface, #fff);
    background: var(--surface, #fff);
    border-left: 1px solid $border-light;
    box-shadow: -12px 0 32px rgba(15, 18, 26, 0.12);
    z-index: 41;
    display: flex;
    flex-direction: column;
    animation: providers-sidebar-in 0.2s ease-out;
  }

  &__sidebar-header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
    padding: 20px 20px 16px;
    border-bottom: 1px solid $border-light;
  }

  &__sidebar-title {
    font-family: $font-display;
    font-size: 1.05rem;
    font-weight: 800;
    color: $text-primary;
  }

  &__sidebar-subtitle {
    margin-top: 3px;
    font-size: 0.75rem;
    color: $text-secondary;
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
    color: $text-secondary;
    font-size: 13px;
    text-align: center;
  }

  &__model-row {
    border: 1px solid $border-light;
    border-radius: 12px;
    padding: 14px;
    background: $surface-alt;
  }

  &__model-row-head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__model-row-name {
    font-weight: 700;
    font-size: 0.875rem;
    color: $text-primary;
  }

  &__model-row-tags {
    margin-top: 8px;
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
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
      font-size: 0.8125rem;
      color: $text-primary;
    }
  }

  &__model-row-meta-label {
    font-size: 0.6875rem;
    text-transform: uppercase;
    letter-spacing: 0.06em;
    color: $text-secondary;
  }

  &__model-row-url {
    margin-top: 10px;
    padding-top: 10px;
    border-top: 1px dashed $border-light;
    font-family: $font-mono;
    font-size: 0.75rem;
    color: $text-secondary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
}

@keyframes providers-sidebar-in {
  from { transform: translateX(100%); }
  to { transform: translateX(0); }
}
