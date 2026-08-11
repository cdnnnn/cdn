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
              <div className="card" key={p.id}>
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
                    <span className={`badge ${p.status === 'connected' ? 'badge-green' : 'badge-gray'}`}>
                      {p.status === 'connected' ? <><Check size={11} /> Connected</> : 'Not connected'}
                    </span>
                  </div>
                </div>
                <div className="card-desc">{p.description}</div>

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

      {addModelOpen && (
        <AddCustomModelDrawer
          submitting={customModelCreating}
          onClose={() => setAddModelOpen(false)}
          onSubmit={(payload) => {
            dispatch(createCustomModel(payload)).then(() => {
              setAddModelOpen(false);
              dispatch(fetchProviders());
              const custom = items.find((p) => p.name === 'Custom');
              if (custom) dispatch(fetchModelsByProvider(custom.id));
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




















//ProviderModelsSidebar.tsx
import { X, Loader2, Inbox } from 'lucide-react';
import type { Provider } from '../../types';
import type { ProviderModel } from '../../api/endpoints/models';
import styles from './Providers.module.scss';

interface ProviderModelsSidebarProps {
  provider: Provider;
  models: ProviderModel[];
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  onClose: () => void;
}

export default function ProviderModelsSidebar({ provider, models, status, onClose }: ProviderModelsSidebarProps) {
  return (
    <>
      <div className={styles['providers__sidebar-overlay']} onClick={onClose} />
      <aside className={styles['providers__sidebar']}>
        <div className={styles['providers__sidebar-header']}>
          <div>
            <div className={styles['providers__sidebar-title']}>{provider.name}</div>
            <div className={styles['providers__sidebar-subtitle']}>
              {models.length} model{models.length === 1 ? '' : 's'} available
            </div>
          </div>
          <button className="btn btn-sm btn-ghost" onClick={onClose} aria-label="Close">
            <X size={16} />
          </button>
        </div>

        <div className={styles['providers__sidebar-body']}>
          {status === 'loading' && (
            <div className={styles['providers__sidebar-empty']}>
              <Loader2 size={18} style={{ animation: 'spin 1.5s linear infinite' }} />
              <span>Loading models…</span>
            </div>
          )}

          {status === 'failed' && (
            <div className={styles['providers__sidebar-empty']}>
              <span>Couldn't load models for this provider.</span>
            </div>
          )}

          {status === 'succeeded' && models.length === 0 && (
            <div className={styles['providers__sidebar-empty']}>
              <Inbox size={18} />
              <span>No models found for this provider yet.</span>
            </div>
          )}

          {status === 'succeeded' && models.map((m) => (
            <div key={m.id} className={styles['providers__model-row']}>
              <div className={styles['providers__model-row-head']}>
                <span className={styles['providers__model-row-name']}>{m.name}</span>
                <span className={`badge ${m.is_active ? 'badge-green' : 'badge-gray'}`}>
                  {m.is_active ? 'Active' : 'Inactive'}
                </span>
              </div>

              <div className={styles['providers__model-row-tags']}>
                {m.category && <span className="tag tag-ind">{m.category}</span>}
                {m.capabilities.map((c) => (
                  <span key={c} className="tag tag-ind">{c}</span>
                ))}
              </div>

              <div className={styles['providers__model-row-meta']}>
                <div>
                  <span className={styles['providers__model-row-meta-label']}>Context</span>
                  <span>{m.context_window.toLocaleString()}</span>
                </div>
                <div>
                  <span className={styles['providers__model-row-meta-label']}>Price (in/out)</span>
                  <span>
                    {m.input_price != null ? `$${m.input_price.toFixed(2)}` : '—'} / {m.output_price != null ? `$${m.output_price.toFixed(2)}` : '—'}
                  </span>
                </div>
                <div>
                  <span className={styles['providers__model-row-meta-label']}>Accuracy</span>
                  <span>{m.accuracy_score != null ? `${m.accuracy_score}%` : '—'}</span>
                </div>
                <div>
                  <span className={styles['providers__model-row-meta-label']}>Agent Score</span>
                  <span>{m.agent_score != null ? `${m.agent_score}%` : '—'}</span>
                </div>
              </div>

              {m.base_url && (
                <div className={styles['providers__model-row-url']} title={m.base_url}>
                  {m.base_url}
                </div>
              )}
            </div>
          ))}
        </div>
      </aside>
    </>
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
















//Modelcatalog.tsx
import { useEffect, useMemo, useState } from 'react';
import { Search, Boxes } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchModels } from '../../store/slices/modelsSlice';
import { fetchProviders } from '../../store/slices/providersSlice';
import { SkeletonTableRows } from '../common/Skeleton';
import styles from './ModelCatalog.module.scss';

export default function ModelCatalog() {
  const dispatch = useAppDispatch();
  const { items, status } = useAppSelector((s) => s.models);
  const providers = useAppSelector((s) => s.providers.items);
  const [search, setSearch] = useState('');
  const [capFilter, setCapFilter] = useState('All');

  useEffect(() => {
    dispatch(fetchModels());
    dispatch(fetchProviders());
  }, [dispatch]);

  const caps = useMemo(() => ['All', ...new Set(items.flatMap((m) => m.capabilities))], [items]);
  const providerName = (id: string) => providers.find((p) => p.id === id)?.name || id;

  const filtered = items.filter((m) => {
    if (capFilter !== 'All' && !m.capabilities.includes(capFilter)) return false;
    const q = search.toLowerCase();
    return !q || m.name.toLowerCase().includes(q) || providerName(m.provider_id).toLowerCase().includes(q);
  });

  return (
    <div className="page-enter pg-shell">
      <div className={styles['model-catalog__header']}>
        <div>
          <p className={styles['model-catalog__header-eyebrow']}>Catalog</p>
          <h1>Model Catalog</h1>
          <p className={styles['model-catalog__header-sub']}>All models across connected providers</p>
        </div>
        <div className={styles['model-catalog__header-meta']}>
          <Boxes size={13} />
          {items.length} model{items.length === 1 ? '' : 's'} listed
        </div>
      </div>
      <div className="pg-toolbar">
        <div className="toolbar">
          <div className="search-box">
            <Search size={16} color="var(--text-muted)" />
            <input placeholder="Search models or providers…" value={search} onChange={(e) => setSearch(e.target.value)} />
          </div>
          <div style={{ display: 'flex', gap: 8, alignItems: 'center' }}>
            <div className="pills">{caps.map((c) => <button key={c} className={`pill ${capFilter === c ? 'on' : ''}`} onClick={() => setCapFilter(c)}>{c}</button>)}</div>
          </div>
        </div>
      </div>
      <div className="pg-body">
        <div className="tw">
          <table className="tbl">
            <thead>
              <tr><th>Model</th><th>Provider</th><th>Capabilities</th><th>Context</th><th>Price (in/out)</th><th>Accuracy</th><th>Status</th></tr>
            </thead>
            <tbody>
              {status === 'loading' && <SkeletonTableRows columns={7} rows={6} />}
              {status !== 'loading' && filtered.map((m) => (
                <tr key={m.id}>
                  <td style={{ fontWeight: 700 }}>{m.name}</td>
                  <td style={{ color: 'var(--text-secondary)' }}>{providerName(m.provider_id)}</td>
                  <td>{m.capabilities.map((c) => <span key={c} className="tag tag-ind">{c}</span>)}</td>
                  <td style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", fontSize: 13 }}>{m.context_window.toLocaleString()}</td>
                  <td style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", fontSize: 13, color: 'var(--text-secondary)' }}>
                    {m.input_price != null ? `$${m.input_price.toFixed(2)}` : '—'} / {m.output_price != null ? `$${m.output_price.toFixed(2)}` : '—'}
                  </td>
                  <td>
                    <span style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", fontWeight: 700, color: (m.accuracy_score || 0) >= 90 ? '#10B981' : 'var(--text-primary)' }}>
                      {m.accuracy_score != null ? `${m.accuracy_score}%` : '—'}
                    </span>
                  </td>
                  <td><span className={`badge ${m.is_active ? 'badge-green' : 'badge-gray'}`}>{m.is_active ? 'Active' : 'Inactive'}</span></td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      </div>
    </div>
  );
}
























//modelsSlice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { modelsApi, type ProviderModel } from '../../api/endpoints/models';
import type { Model, CustomModelRequest } from '../../types';

type FetchStatus = 'idle' | 'loading' | 'succeeded' | 'failed';

interface ModelsState {
  items: Model[];
  status: FetchStatus;
  error: string | null;
  creating: boolean;
  byProvider: Record<string, ProviderModel[]>;
  byProviderStatus: Record<string, FetchStatus>;
}

const initialState: ModelsState = {
  items: [],
  status: 'idle',
  error: null,
  creating: false,
  byProvider: {},
  byProviderStatus: {},
};

export const fetchModels = createAsyncThunk('models/fetchAll', () => modelsApi.list());

export const fetchModelsByProvider = createAsyncThunk(
  'models/fetchByProvider',
  async (providerId: string) => {
    const { models } = await modelsApi.listByProvider(providerId);
    return { providerId, models };
  }
);

export const createCustomModel = createAsyncThunk(
  'models/createCustom',
  async (payload: CustomModelRequest, { dispatch }) => {
    await modelsApi.createCustom(payload);
    // spec: no meaningful body returned, so refetch afterwards
    await dispatch(fetchModels());
  }
);

const modelsSlice = createSlice({
  name: 'models',
  initialState,
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchModels.pending, (state) => {
        state.status = 'loading';
      })
      .addCase(fetchModels.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.items = action.payload;
      })
      .addCase(fetchModels.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.error.message || 'Failed to load models';
      })
      .addCase(createCustomModel.pending, (state) => {
        state.creating = true;
      })
      .addCase(createCustomModel.fulfilled, (state) => {
        state.creating = false;
      })
      .addCase(createCustomModel.rejected, (state, action) => {
        state.creating = false;
        state.error = action.error.message || 'Failed to register custom model';
      })
      .addCase(fetchModelsByProvider.pending, (state, action) => {
        state.byProviderStatus[action.meta.arg] = 'loading';
      })
      .addCase(fetchModelsByProvider.fulfilled, (state, action) => {
        state.byProviderStatus[action.payload.providerId] = 'succeeded';
        state.byProvider[action.payload.providerId] = action.payload.models;
      })
      .addCase(fetchModelsByProvider.rejected, (state, action) => {
        state.byProviderStatus[action.meta.arg] = 'failed';
      });
  },
});

export default modelsSlice.reducer;















//Models.ts
import api from '../axiosInstance';
import type { Model, CustomModelRequest } from '../../types';

// Shape returned by GET /models/by-provider/:providerId
export interface ProviderModel {
  id: string;
  name: string;
  provider_id: string;
  category: string;
  capabilities: string[];
  context_window: number;
  input_price: number | null;
  output_price: number | null;
  accuracy_score: number | null;
  agent_score: number | null;
  is_active: boolean;
  base_url: string | null;
}

export const modelsApi = {
  list: () => api.get<{ models: Model[] }>('/models').then((r) => r.data.models),

  createCustom: (payload: CustomModelRequest) =>
    api.post<void>('/models/custom', payload).then(() => undefined),

  listByProvider: (providerId: string) =>
    api
      .get<{ models: ProviderModel[]; total: number }>(`/models/by-provider/${providerId}`)
      .then((r) => r.data),
};



















//providersSlice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { providersApi } from '../../api/endpoints/providers';
import type { Provider, ConnectProviderRequest, CreateProviderRequest } from '../../types';

interface ProvidersState {
  items: Provider[];
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  error: string | null;
  mutatingId: string | null;
  creating: boolean;
  syncingId: string | null;
}

const initialState: ProvidersState = {
  items: [],
  status: 'idle',
  error: null,
  mutatingId: null,
  creating: false,
  syncingId: null,
};

export const fetchProviders = createAsyncThunk('providers/fetchAll', () => providersApi.list());

export const createProvider = createAsyncThunk(
  'providers/create',
  (payload: CreateProviderRequest) => providersApi.create(payload)
);

export const deleteProvider = createAsyncThunk(
  'providers/delete',
  async (providerId: string) => {
    await providersApi.remove(providerId);
    return providerId;
  }
);

export const connectProvider = createAsyncThunk(
  'providers/connect',
  async ({ providerId, payload }: { providerId: string; payload: ConnectProviderRequest }) => {
    await providersApi.connect(providerId, payload);
    return providerId;
  }
);

export const disconnectProvider = createAsyncThunk(
  'providers/disconnect',
  async (providerId: string) => {
    await providersApi.disconnect(providerId);
    return providerId;
  }
);

export const syncModels = createAsyncThunk(
  'providers/syncModels',
  async (providerId: string) => providersApi.syncModels(providerId)
);

const providersSlice = createSlice({
  name: 'providers',
  initialState,
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchProviders.pending, (state) => {
        state.status = 'loading';
      })
      .addCase(fetchProviders.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.items = action.payload;
      })
      .addCase(fetchProviders.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.error.message || 'Failed to load providers';
      })
      .addCase(createProvider.pending, (state) => {
        state.creating = true;
      })
      .addCase(createProvider.fulfilled, (state, action) => {
        state.creating = false;
        state.items.push(action.payload);
      })
      .addCase(createProvider.rejected, (state, action) => {
        state.creating = false;
        state.error = action.error.message || 'Failed to create provider';
      })
      .addCase(deleteProvider.pending, (state, action) => {
        state.mutatingId = action.meta.arg;
      })
      .addCase(deleteProvider.fulfilled, (state, action) => {
        state.mutatingId = null;
        state.items = state.items.filter((i) => i.id !== action.payload);
      })
      .addCase(deleteProvider.rejected, (state) => {
        state.mutatingId = null;
      })
      .addCase(connectProvider.pending, (state, action) => {
        state.mutatingId = action.meta.arg.providerId;
      })
      .addCase(connectProvider.fulfilled, (state, action) => {
        state.mutatingId = null;
        const p = state.items.find((i) => i.id === action.payload);
        if (p) p.status = 'connected';
      })
      .addCase(connectProvider.rejected, (state) => {
        state.mutatingId = null;
      })
      .addCase(disconnectProvider.pending, (state, action) => {
        state.mutatingId = action.meta.arg;
      })
      .addCase(disconnectProvider.fulfilled, (state, action) => {
        state.mutatingId = null;
        const p = state.items.find((i) => i.id === action.payload);
        if (p) p.status = 'not_connected';
      })
      .addCase(disconnectProvider.rejected, (state) => {
        state.mutatingId = null;
      })
      .addCase(syncModels.pending, (state, action) => {
        state.syncingId = action.meta.arg;
      })
      .addCase(syncModels.fulfilled, (state, action) => {
        state.syncingId = null;
        const p = state.items.find((i) => i.id === action.payload.provider_id);
        if (p) p.model_count = action.payload.total_available;
      })
      .addCase(syncModels.rejected, (state) => {
        state.syncingId = null;
      });
  },
});

export default providersSlice.reducer;





















//Providers.ts
import api from '../axiosInstance';
import type {
  Provider,
  ConnectProviderRequest,
  ConnectProviderResponse,
  DisconnectProviderResponse,
  CreateProviderRequest,
  CreateProviderResponse,
  DeleteProviderResponse,
  SyncModelsResponse,
} from '../../types';

export const providersApi = {
  list: () => api.get<{ providers: Provider[] }>('/providers').then((r) => r.data.providers),

  create: (payload: CreateProviderRequest) =>
    api.post<CreateProviderResponse>('/providers', payload).then((r) => r.data),

  remove: (providerId: string) =>
    api.delete<DeleteProviderResponse>(`/providers/${providerId}`).then((r) => r.data),

  connect: (providerId: string, payload: ConnectProviderRequest) =>
    api
      .post<ConnectProviderResponse>(`/providers/${providerId}/connect`, payload)
      .then((r) => r.data),

  disconnect: (providerId: string) =>
    api
      .delete<DisconnectProviderResponse>(`/providers/${providerId}/disconnect`)
      .then((r) => r.data),

  syncModels: (providerId: string) =>
    api
      .post<SyncModelsResponse>(`/providers/${providerId}/sync-models`)
      .then((r) => r.data),
};


















//Modelcatalog.module.scss
@use '../../styles/_variables' as *;

.model-catalog {
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
}
