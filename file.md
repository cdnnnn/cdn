//Models.ts
import api from '../axiosInstance';
import type { Model, CustomModelRequest } from '../../types';

export interface ModelCategory {
  value: string;
  label: string;
  description: string;
}

export interface DiscoveredModel {
  id: string;
  name: string;
  context_window: number | null;
  max_model_len: number | null;
  owned_by: string;
  already_added: boolean;
}

export interface DiscoverModelsRequest {
  base_url: string;
  api_key?: string;
}

export interface DiscoverModelsResponse {
  base_url: string;
  models: DiscoveredModel[];
  total: number;
  errors: string[];
}

export interface DeleteCustomModelResponse {
  status: string;
  model_id: string;
}

export const modelsApi = {
  list: () => api.get<{ models: Model[] }>('/models').then((r) => r.data.models ?? []),

  createCustom: (payload: CustomModelRequest) =>
    api.post<void>('/models/custom', payload).then(() => undefined),

  // DELETE /models/custom/:modelId — remove a previously registered custom model
  deleteCustom: (modelId: string) =>
    api.delete<DeleteCustomModelResponse>(`/models/custom/${modelId}`).then((r) => r.data),

  // GET /models/by-provider/:providerId — all models registered under a single provider
  listByProvider: (providerId: string) =>
    api
      .get<{ models: Model[]; total: number }>(`/models/by-provider/${providerId}`)
      .then((r) => ({ models: r.data.models ?? [], total: r.data.total ?? 0 })),

  // GET /models/categories — used to populate the Category dropdown
  listCategories: () =>
    api.get<{ categories: ModelCategory[] }>('/models/categories').then((r) => r.data.categories ?? []),

  // POST /models/discover — optional endpoint probe to auto-fill model name/id/context window
  discover: (payload: DiscoverModelsRequest) =>
    api.post<DiscoverModelsResponse>('/models/discover', payload).then((r) => ({
      base_url: r.data.base_url,
      models: r.data.models ?? [],
      total: r.data.total ?? 0,
      errors: r.data.errors ?? [],
    })),
};














//Modelsslice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { modelsApi, type ModelCategory } from '../../api/endpoints/models';
import type { Model, CustomModelRequest } from '../../types';

type FetchStatus = 'idle' | 'loading' | 'succeeded' | 'failed';

interface ModelsState {
  items: Model[];
  status: FetchStatus;
  error: string | null;
  creating: boolean;
  byProvider: Record<string, Model[]>;
  byProviderStatus: Record<string, FetchStatus>;
  categories: ModelCategory[];
  categoriesStatus: FetchStatus;
  deletingId: string | null;
}

const initialState: ModelsState = {
  items: [],
  status: 'idle',
  error: null,
  creating: false,
  byProvider: {},
  byProviderStatus: {},
  categories: [],
  categoriesStatus: 'idle',
  deletingId: null,
};

export const fetchModels = createAsyncThunk('models/fetchAll', () => modelsApi.list());

export const fetchModelsByProvider = createAsyncThunk(
  'models/fetchByProvider',
  async (providerId: string) => {
    const { models } = await modelsApi.listByProvider(providerId);
    return { providerId, models };
  }
);

export const fetchModelCategories = createAsyncThunk(
  'models/fetchCategories',
  () => modelsApi.listCategories()
);

export const createCustomModel = createAsyncThunk(
  'models/createCustom',
  async (payload: CustomModelRequest, { dispatch }) => {
    await modelsApi.createCustom(payload);
    // spec: no meaningful body returned, so refetch afterwards
    await dispatch(fetchModels());
  }
);

export const deleteCustomModel = createAsyncThunk(
  'models/deleteCustom',
  async ({ modelId, providerId }: { modelId: string; providerId: string }) => {
    const res = await modelsApi.deleteCustom(modelId);
    return { modelId: res.model_id || modelId, providerId };
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
        state.items = action.payload ?? [];
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
        state.byProvider[action.payload.providerId] = action.payload.models ?? [];
      })
      .addCase(fetchModelsByProvider.rejected, (state, action) => {
        state.byProviderStatus[action.meta.arg] = 'failed';
      })
      .addCase(fetchModelCategories.pending, (state) => {
        state.categoriesStatus = 'loading';
      })
      .addCase(fetchModelCategories.fulfilled, (state, action) => {
        state.categoriesStatus = 'succeeded';
        state.categories = action.payload ?? [];
      })
      .addCase(fetchModelCategories.rejected, (state) => {
        state.categoriesStatus = 'failed';
      })
      .addCase(deleteCustomModel.pending, (state, action) => {
        state.deletingId = action.meta.arg.modelId;
      })
      .addCase(deleteCustomModel.fulfilled, (state, action) => {
        state.deletingId = null;
        const { modelId, providerId } = action.payload;
        state.items = state.items.filter((m) => m.id !== modelId);
        if (state.byProvider[providerId]) {
          state.byProvider[providerId] = state.byProvider[providerId].filter((m) => m.id !== modelId);
        }
      })
      .addCase(deleteCustomModel.rejected, (state, action) => {
        state.deletingId = null;
        state.error = action.error.message || 'Failed to delete model';
      });
  },
});

export default modelsSlice.reducer;





















//Providermodelssidebar.tsx
import { X, Loader2, Inbox, Trash2 } from 'lucide-react';
import type { Provider, Model } from '../../types';
import styles from './Providers.module.scss';

interface ProviderModelsSidebarProps {
  provider: Provider;
  models: Model[];
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  onClose: () => void;
  /** When true, each row gets a delete affordance (only valid for the Custom provider). */
  canDelete?: boolean;
  /** Model id currently being deleted, so its row can show a spinner and disable itself. */
  deletingId?: string | null;
  onDelete?: (modelId: string) => void;
}

export default function ProviderModelsSidebar({
  provider,
  models = [],
  status,
  onClose,
  canDelete = false,
  deletingId = null,
  onDelete,
}: ProviderModelsSidebarProps) {
  const handleDelete = (m: Model) => {
    if (!onDelete) return;
    const label = m.name ?? m.id;
    if (window.confirm(`Remove "${label}"? This cannot be undone.`)) {
      onDelete(m.id);
    }
  };

  return (
    <>
      <div className={styles['providers__sidebar-overlay']} onClick={onClose} />
      <aside className={styles['providers__sidebar']}>
        <div className={styles['providers__sidebar-header']}>
          <div>
            <div className={styles['providers__sidebar-title']}>{provider?.name ?? 'Provider'}</div>
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

          {status === 'succeeded' && models.map((m) => {
            const isDeleting = deletingId === m.id;
            return (
              <div key={m.id} className={`${styles['providers__model-row']} ${isDeleting ? styles['providers__model-row--deleting'] : ''}`}>
                <div className={styles['providers__model-row-head']}>
                  <span className={styles['providers__model-row-name']}>{m.name ?? 'Unnamed model'}</span>
                  <div className={styles['providers__model-row-head-actions']}>
                    <span className={`badge ${m.is_active ? 'badge-green' : 'badge-gray'}`}>
                      {m.is_active ? 'Active' : 'Inactive'}
                    </span>
                    {canDelete && (
                      <button
                        type="button"
                        className={styles['providers__model-row-delete']}
                        onClick={() => handleDelete(m)}
                        disabled={isDeleting}
                        title="Remove model"
                        aria-label={`Remove ${m.name ?? m.id}`}
                      >
                        {isDeleting ? (
                          <Loader2 size={13} style={{ animation: 'spin 1.5s linear infinite' }} />
                        ) : (
                          <Trash2 size={13} />
                        )}
                      </button>
                    )}
                  </div>
                </div>

                <div className={styles['providers__model-row-tags']}>
                  {m.category && <span className="tag tag-ind">{m.category}</span>}
                  {(m.capabilities ?? []).map((c) => (
                    <span key={c} className="tag tag-ind">{c}</span>
                  ))}
                </div>

                <div className={styles['providers__model-row-meta']}>
                  <div>
                    <span className={styles['providers__model-row-meta-label']}>Context</span>
                    <span>{(m.context_window ?? 0).toLocaleString()}</span>
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
            );
          })}
        </div>
      </aside>
    </>
  );
}



















//Providers.tsx
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
import { fetchModelsByProvider, createCustomModel, deleteCustomModel } from '../../store/slices/modelsSlice';
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
  const customModelDeletingId = useAppSelector((s) => s.models.deletingId);
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
                <div className={styles['providers__card']} key={p.id}>
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
                        title="View models"
                        aria-label={`View models for ${p.name ?? 'provider'}`}
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
          canDelete={viewModelsProvider.name === 'Custom'}
          deletingId={customModelDeletingId}
          onDelete={(modelId) => {
            dispatch(deleteCustomModel({ modelId, providerId: viewModelsProvider.id })).then(() => {
              dispatch(fetchProviders());
            });
          }}
        />
      )}
    </div>
  );
}






















//Providers.module.scss
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

@keyframes providers-sidebar-in {
  from { transform: translateX(100%); }
  to { transform: translateX(0); }
}

@media (max-width: 768px) {
  .providers__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .providers__toolbar { padding: 12px 18px; }
  .providers__grid { grid-template-columns: 1fr; }
}
