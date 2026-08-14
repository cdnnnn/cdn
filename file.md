//models.ts
import api from '../axiosInstance';
import type { Model, CustomModelRequest } from '../../types';

export const modelsApi = {
  list: () => api.get<{ models: Model[] }>('/models').then((r) => r.data.models ?? []),

  createCustom: (payload: CustomModelRequest) =>
    api.post<void>('/models/custom', payload).then(() => undefined),

  // GET /models/by-provider/:providerId — all models registered under a single provider
  listByProvider: (providerId: string) =>
    api
      .get<{ models: Model[]; total: number }>(`/models/by-provider/${providerId}`)
      .then((r) => ({ models: r.data.models ?? [], total: r.data.total ?? 0 })),
};








//providers.ts
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
  list: () => api.get<{ providers: Provider[] }>('/providers').then((r) => r.data.providers ?? []),

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













//modelsSlice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { modelsApi } from '../../api/endpoints/models';
import type { Model, CustomModelRequest } from '../../types';

type FetchStatus = 'idle' | 'loading' | 'succeeded' | 'failed';

interface ModelsState {
  items: Model[];
  status: FetchStatus;
  error: string | null;
  creating: boolean;
  byProvider: Record<string, Model[]>;
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
      });
  },
});

export default modelsSlice.reducer;















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
        state.items = action.payload ?? [];
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
        if (action.payload) state.items.push(action.payload);
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
        const p = state.items.find((i) => i.id === action.payload?.provider_id);
        if (p) p.model_count = action.payload?.total_available ?? p.model_count;
      })
      .addCase(syncModels.rejected, (state) => {
        state.syncingId = null;
      });
  },
});

export default providersSlice.reducer;
















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
        />
      )}
    </div>
  );
}















//ModelCatalog.tsx
import { useEffect, useMemo, useState } from 'react';
import { Search, Boxes, ChevronUp, ChevronDown, ChevronsUpDown, ChevronLeft, ChevronRight, ChevronsLeft, ChevronsRight, ListFilter } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchModels } from '../../store/slices/modelsSlice';
import { fetchProviders } from '../../store/slices/providersSlice';
import { SkeletonTableRows } from '../common/Skeleton';
import type { Model } from '../../types';
import styles from './ModelCatalog.module.scss';

type SortKey = 'name' | 'provider' | 'context_window' | 'price' | 'accuracy' | 'status';
type SortDir = 'asc' | 'desc';

const PAGE_SIZE_OPTIONS = [10, 25, 50, 100];

// Builds a compact page-number list with ellipses, e.g. [1, '…', 4, 5, 6, '…', 12]
function buildPageList(current: number, total: number): (number | '…')[] {
  if (total <= 7) return Array.from({ length: total }, (_, i) => i + 1);
  const pages = new Set<number>([1, total, current, current - 1, current + 1]);
  const sorted = [...pages].filter((p) => p >= 1 && p <= total).sort((a, b) => a - b);
  const result: (number | '…')[] = [];
  let prev = 0;
  for (const p of sorted) {
    if (prev && p - prev > 1) result.push('…');
    result.push(p);
    prev = p;
  }
  return result;
}

interface SortableThProps {
  label: string;
  sortKey: SortKey;
  activeKey: SortKey;
  dir: SortDir;
  onSort: (key: SortKey) => void;
}

function SortableTh({ label, sortKey, activeKey, dir, onSort }: SortableThProps) {
  const active = activeKey === sortKey;
  return (
    <th className={styles['model-catalog__sortable-th']}>
      <button
        type="button"
        className={`${styles['model-catalog__sort-btn']} ${active ? styles['model-catalog__sort-btn--active'] : ''}`}
        onClick={() => onSort(sortKey)}
      >
        {label}
        {active ? (
          dir === 'asc' ? <ChevronUp size={13} /> : <ChevronDown size={13} />
        ) : (
          <ChevronsUpDown size={13} className={styles['model-catalog__sort-icon-idle']} />
        )}
      </button>
    </th>
  );
}

export default function ModelCatalog() {
  const dispatch = useAppDispatch();
  const { items, status } = useAppSelector((s) => s.models);
  const providers = useAppSelector((s) => s.providers.items);
  const [search, setSearch] = useState('');
  const [capFilter, setCapFilter] = useState('All');
  const [sortKey, setSortKey] = useState<SortKey>('name');
  const [sortDir, setSortDir] = useState<SortDir>('asc');
  const [page, setPage] = useState(1);
  const [pageSize, setPageSize] = useState(10);

  useEffect(() => {
    dispatch(fetchModels());
    dispatch(fetchProviders());
  }, [dispatch]);

  const caps = useMemo(() => ['All', ...new Set(items.flatMap((m) => m.capabilities ?? []))], [items]);
  const providerName = (id: string) => providers.find((p) => p.id === id)?.name || id || 'Unknown';

  const filtered = useMemo(() => {
    return items.filter((m) => {
      if (capFilter !== 'All' && !(m.capabilities ?? []).includes(capFilter)) return false;
      const q = search.toLowerCase();
      return (
        !q ||
        (m.name ?? '').toLowerCase().includes(q) ||
        providerName(m.provider_id).toLowerCase().includes(q)
      );
    });
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [items, providers, search, capFilter]);

  const sorted = useMemo(() => {
    const dir = sortDir === 'asc' ? 1 : -1;
    const compare = (a: Model, b: Model): number => {
      switch (sortKey) {
        case 'name':
          return (a.name ?? '').localeCompare(b.name ?? '') * dir;
        case 'provider':
          return providerName(a.provider_id).localeCompare(providerName(b.provider_id)) * dir;
        case 'context_window':
          return ((a.context_window ?? 0) - (b.context_window ?? 0)) * dir;
        case 'price':
          return ((a.input_price ?? -1) - (b.input_price ?? -1)) * dir;
        case 'accuracy':
          return ((a.accuracy_score ?? -1) - (b.accuracy_score ?? -1)) * dir;
        case 'status':
          return (Number(a.is_active) - Number(b.is_active)) * dir;
        default:
          return 0;
      }
    };
    return [...filtered].sort(compare);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [filtered, sortKey, sortDir, providers]);

  const total = sorted.length;
  const totalPages = Math.max(1, Math.ceil(total / pageSize));
  const safePage = Math.min(page, totalPages);
  const startIdx = (safePage - 1) * pageSize;
  const pageItems = sorted.slice(startIdx, startIdx + pageSize);
  const pageList = useMemo(() => buildPageList(safePage, totalPages), [safePage, totalPages]);

  useEffect(() => {
    setPage(1);
  }, [search, capFilter, pageSize]);

  const toggleSort = (key: SortKey) => {
    if (sortKey === key) {
      setSortDir((d) => (d === 'asc' ? 'desc' : 'asc'));
    } else {
      setSortKey(key);
      setSortDir('asc');
    }
  };

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
          {(items?.length ?? 0)} model{(items?.length ?? 0) === 1 ? '' : 's'} listed
        </div>
      </div>
      <div className="pg-toolbar">
        <div className="toolbar">
          <div className="search-box">
            <Search size={16} color="var(--text-muted)" />
            <input placeholder="Search models or providers…" value={search} onChange={(e) => setSearch(e.target.value)} />
          </div>
          <div style={{ display: 'flex', gap: 8, alignItems: 'center' }}>
            <div className={styles['model-catalog__filter-group']}>
              <span className={styles['model-catalog__toolbar-label']}>
                <ListFilter size={11} /> Capability
              </span>
              <div className="pills">{caps.map((c) => <button key={c} className={`pill ${capFilter === c ? 'on' : ''}`} onClick={() => setCapFilter(c)}>{c}</button>)}</div>
            </div>
          </div>
        </div>
      </div>
      <div className="pg-body">
        <div className="tw">
          <table className="tbl">
            <thead>
              <tr>
                <SortableTh label="Model" sortKey="name" activeKey={sortKey} dir={sortDir} onSort={toggleSort} />
                <SortableTh label="Provider" sortKey="provider" activeKey={sortKey} dir={sortDir} onSort={toggleSort} />
                <th>Capabilities</th>
                <SortableTh label="Context" sortKey="context_window" activeKey={sortKey} dir={sortDir} onSort={toggleSort} />
                <SortableTh label="Price (in/out)" sortKey="price" activeKey={sortKey} dir={sortDir} onSort={toggleSort} />
                <SortableTh label="Accuracy" sortKey="accuracy" activeKey={sortKey} dir={sortDir} onSort={toggleSort} />
                <SortableTh label="Status" sortKey="status" activeKey={sortKey} dir={sortDir} onSort={toggleSort} />
              </tr>
            </thead>
            <tbody>
              {status === 'loading' && <SkeletonTableRows columns={7} rows={6} />}
              {status !== 'loading' && pageItems.map((m) => (
                <tr key={m.id}>
                  <td style={{ fontWeight: 700 }}>{m.name ?? '—'}</td>
                  <td style={{ color: 'var(--text-secondary)' }}>{providerName(m.provider_id)}</td>
                  <td>{(m.capabilities ?? []).map((c) => <span key={c} className="tag tag-ind">{c}</span>)}</td>
                  <td style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", fontSize: 13 }}>{(m.context_window ?? 0).toLocaleString()}</td>
                  <td style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", fontSize: 13, color: 'var(--text-secondary)' }}>
                    {m.input_price != null ? `$${m.input_price.toFixed(2)}` : '—'} / {m.output_price != null ? `$${m.output_price.toFixed(2)}` : '—'}
                  </td>
                  <td>
                    <span style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", fontWeight: 700, color: (m.accuracy_score ?? 0) >= 90 ? '#10B981' : 'var(--text-primary)' }}>
                      {m.accuracy_score != null ? `${m.accuracy_score}%` : '—'}
                    </span>
                  </td>
                  <td><span className={`badge ${m.is_active ? 'badge-green' : 'badge-gray'}`}>{m.is_active ? 'Active' : 'Inactive'}</span></td>
                </tr>
              ))}
              {status !== 'loading' && pageItems.length === 0 && (
                <tr>
                  <td colSpan={7} className={styles['model-catalog__empty']}>No models match your filters.</td>
                </tr>
              )}
            </tbody>
          </table>

          {status !== 'loading' && total > 0 && (
            <div className={styles['model-catalog__pagination']}>
              <div className={styles['model-catalog__pagination-info']}>
                <span>
                  Showing <strong>{startIdx + 1}–{Math.min(startIdx + pageSize, total)}</strong> of <strong>{total}</strong> model{total === 1 ? '' : 's'}
                </span>
                <div className={styles['model-catalog__page-size']}>
                  <label htmlFor="model-catalog-page-size">Rows per page</label>
                  <select
                    id="model-catalog-page-size"
                    value={pageSize}
                    onChange={(e) => setPageSize(Number(e.target.value))}
                  >
                    {PAGE_SIZE_OPTIONS.map((n) => (
                      <option key={n} value={n}>{n}</option>
                    ))}
                  </select>
                </div>
              </div>

              <div className={styles['model-catalog__pager']}>
                <button
                  className={styles['model-catalog__page-btn']}
                  disabled={safePage === 1}
                  onClick={() => setPage(1)}
                  aria-label="First page"
                >
                  <ChevronsLeft size={14} />
                </button>
                <button
                  className={styles['model-catalog__page-btn']}
                  disabled={safePage === 1}
                  onClick={() => setPage((p) => Math.max(1, p - 1))}
                  aria-label="Previous page"
                >
                  <ChevronLeft size={14} />
                </button>

                {pageList.map((p, i) =>
                  p === '…' ? (
                    <span key={`dots-${i}`} className={styles['model-catalog__page-dots']}>…</span>
                  ) : (
                    <button
                      key={p}
                      className={`${styles['model-catalog__page-btn']} ${styles['model-catalog__page-btn--num']} ${p === safePage ? styles['model-catalog__page-btn--active'] : ''}`}
                      onClick={() => setPage(p)}
                      aria-current={p === safePage ? 'page' : undefined}
                    >
                      {p}
                    </button>
                  )
                )}

                <button
                  className={styles['model-catalog__page-btn']}
                  disabled={safePage === totalPages}
                  onClick={() => setPage((p) => Math.min(totalPages, p + 1))}
                  aria-label="Next page"
                >
                  <ChevronRight size={14} />
                </button>
                <button
                  className={styles['model-catalog__page-btn']}
                  disabled={safePage === totalPages}
                  onClick={() => setPage(totalPages)}
                  aria-label="Last page"
                >
                  <ChevronsRight size={14} />
                </button>
              </div>
            </div>
          )}
        </div>
      </div>
    </div>
  );
}














//Providermodelssidebar.tsx
import { X, Loader2, Inbox } from 'lucide-react';
import type { Provider, Model } from '../../types';
import styles from './Providers.module.scss';

interface ProviderModelsSidebarProps {
  provider: Provider;
  models: Model[];
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  onClose: () => void;
}

export default function ProviderModelsSidebar({ provider, models = [], status, onClose }: ProviderModelsSidebarProps) {
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

          {status === 'succeeded' && models.map((m) => (
            <div key={m.id} className={styles['providers__model-row']}>
              <div className={styles['providers__model-row-head']}>
                <span className={styles['providers__model-row-name']}>{m.name ?? 'Unnamed model'}</span>
                <span className={`badge ${m.is_active ? 'badge-green' : 'badge-gray'}`}>
                  {m.is_active ? 'Active' : 'Inactive'}
                </span>
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
          ))}
        </div>
      </aside>
    </>
  );
}
