//Providers.tsx
import { useEffect, useState } from 'react';
import { Search, Check, Plus, Settings, Unlink, Loader2, Cable, Trash2, RefreshCw } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import {
  fetchProviders,
  createProvider,
  deleteProvider,
  connectProvider,
  disconnectProvider,
  syncModels,
} from '../../store/slices/providersSlice';
import AddProviderDrawer from './AddProviderDrawer';
import { SkeletonCards } from '../common/Skeleton';
import styles from './Providers.module.scss';

type Filter = 'all' | 'connected' | 'available';

export default function Providers() {
  const dispatch = useAppDispatch();
  const { items, status, mutatingId, creating, syncingId } = useAppSelector((s) => s.providers);
  const [search, setSearch] = useState('');
  const [filter, setFilter] = useState<Filter>('all');
  const [keyPromptFor, setKeyPromptFor] = useState<string | null>(null);
  const [apiKeyInput, setApiKeyInput] = useState('');
  const [drawerOpen, setDrawerOpen] = useState(false);

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
            <Search size={16} color="#9CA3AF" />
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
          {status !== 'loading' && filtered.map((p) => (
            <div className="card" key={p.id}>
              <div className="card-hdr">
                <div style={{ display: 'flex', alignItems: 'center', gap: 14 }}>
                  <div className={`card-icon ${styles['providers__icon']}`}>{p.logo_url ? <img src={p.logo_url} alt={p.name} /> : p.name[0]}</div>
                  <div>
                    <div className="card-title">{p.name}</div>
                    <div style={{ fontSize: 12, color: '#6B7280', marginTop: 2 }}>{p.model_count} models</div>
                  </div>
                </div>
                <span className={`badge ${p.status === 'connected' ? 'badge-green' : 'badge-gray'}`}>
                  {p.status === 'connected' ? <><Check size={11} /> Connected</> : 'Not connected'}
                </span>
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
                <div className="card-foot">
                  <button
                    className={`btn btn-sm ${p.status === 'connected' ? 'btn-ghost' : 'btn-ind'}`}
                    disabled={mutatingId === p.id}
                    onClick={() => setKeyPromptFor(p.id)}
                  >
                    {mutatingId === p.id ? (
                      <Loader2 size={14} style={{ animation: 'spin 1.5s linear infinite' }} />
                    ) : p.status === 'connected' ? (
                      <><Settings size={14} /> Configure</>
                    ) : (
                      <><Plus size={14} /> Connect</>
                    )}
                  </button>
                  {p.status === 'connected' && (
                    <>
                      <button
                        className="btn btn-sm btn-ghost"
                        disabled={syncingId === p.id}
                        onClick={() => dispatch(syncModels(p.id))}
                      >
                        {syncingId === p.id ? (
                          <Loader2 size={14} style={{ animation: 'spin 1.5s linear infinite' }} />
                        ) : (
                          <><RefreshCw size={14} /> Sync Models</>
                        )}
                      </button>
                      <button className="btn btn-sm btn-danger" disabled={mutatingId === p.id} onClick={() => dispatch(disconnectProvider(p.id))}>
                        <Unlink size={14} /> Disconnect
                      </button>
                    </>
                  )}
                  {p.status !== 'connected' && (
                    <button
                      className="btn btn-sm btn-danger"
                      disabled={mutatingId === p.id}
                      onClick={() => {
                        if (window.confirm(`Delete ${p.name}? This cannot be undone.`)) {
                          dispatch(deleteProvider(p.id));
                        }
                      }}
                    >
                      <Trash2 size={14} /> Delete
                    </button>
                  )}
                </div>
              )}
            </div>
          ))}
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
    </div>
  );
}













/AddProviderDrawer.tsx

import { useState } from 'react';
import { X } from 'lucide-react';
import type { CreateProviderRequest } from '../../types';
import styles from './AddProviderDrawer.module.scss';

interface Props {
  onClose: () => void;
  onSubmit: (payload: CreateProviderRequest) => void;
  submitting: boolean;
}

const initial: CreateProviderRequest = {
  id: '',
  name: '',
  description: '',
  base_url: '',
  logo: '',
};

export default function AddProviderDrawer({ onClose, onSubmit, submitting }: Props) {
  const [form, setForm] = useState<CreateProviderRequest>(initial);

  const set = <K extends keyof CreateProviderRequest>(key: K, value: CreateProviderRequest[K]) =>
    setForm((f) => ({ ...f, [key]: value }));

  const valid = form.id.trim() && form.name.trim() && form.base_url.trim();

  return (
    <div className={styles['drawer-overlay']} onClick={onClose}>
      <div className={styles.drawer} onClick={(e) => e.stopPropagation()}>
        <div className={styles['drawer__hdr']}>
          <h2>Add Provider</h2>
          <button className={styles['drawer__close']} onClick={onClose}><X size={18} /></button>
        </div>
        <div className={styles['drawer__body']}>
          <div className="fg"><label className="fl">Provider ID</label><input className="fi" value={form.id} onChange={(e) => set('id', e.target.value)} placeholder="e.g. openai" /></div>
          <div className="fg"><label className="fl">Display Name</label><input className="fi" value={form.name} onChange={(e) => set('name', e.target.value)} placeholder="e.g. OpenAI" /></div>
          <div className="fg"><label className="fl">Base URL</label><input className="fi" value={form.base_url} onChange={(e) => set('base_url', e.target.value)} placeholder="https://…" /></div>
          <div className="fg"><label className="fl">Logo URL <span className="opt">(optional)</span></label><input className="fi" value={form.logo} onChange={(e) => set('logo', e.target.value)} placeholder="https://…/logo.png" /></div>
          <div className="fg"><label className="fl">Description <span className="opt">(optional)</span></label><textarea className="fi" rows={3} value={form.description} onChange={(e) => set('description', e.target.value)} /></div>
        </div>
        <div className={styles['drawer__foot']}>
          <button className="btn btn-ghost" onClick={onClose}>Cancel</button>
          <button className="btn btn-ind" disabled={!valid || submitting} onClick={() => onSubmit(form)}>{submitting ? 'Adding…' : 'Add Provider'}</button>
        </div>
      </div>
    </div>
  );
}



















// AddProviderDrawer.module.scss

@use '../../styles/_variables' as *;

.drawer-overlay {
  position: fixed; inset: 0; background: rgba(17, 24, 39, .4); z-index: 100;
  display: flex; justify-content: flex-end;
}
.drawer {
  width: 420px; max-width: 100%; height: 100%; background: $surface; box-shadow: $shadow-4;
  display: flex; flex-direction: column; animation: drawerIn .25s ease both;
}
@keyframes drawerIn { from { transform: translateX(24px); opacity: 0; } to { transform: translateX(0); opacity: 1; } }
.drawer__hdr {
  display: flex; justify-content: space-between; align-items: center; padding: 20px 24px; border-bottom: 1px solid $border-light;
  h2 { font-size: 18px; font-weight: 700; }
}
.drawer__close { background: none; border: none; cursor: pointer; color: $text-muted; }
.drawer__body { flex: 1; overflow-y: auto; padding: 24px; }
.drawer__foot { display: flex; justify-content: flex-end; gap: 10px; padding: 16px 24px; border-top: 1px solid $border-light; }




















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
















//providers.api.ts
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













//index.ts
// ---------- Auth ----------
export interface SsoLoginRequest {
  token: string;
  data: string;
}
export interface SsoLoginResult {
  token: string;
  username: string;
  email: string;
  language: string;
  profile_name: string;
}
export interface SsoLoginResponse {
  status: string;
  message: string;
  result: SsoLoginResult;
}

// ---------- Providers ----------
export interface Provider {
  id: string;
  name: string;
  description: string;
  logo_url: string | null;
  base_url: string | null;
  url_template: string | null;
  model_count: number;
  status: 'connected' | 'not_connected' | string;
}
export interface ConnectProviderRequest {
  api_key: string;
}
export interface ConnectProviderResponse {
  status: 'connected';
  provider_id: string;
  models_synced: number;
}
export interface DisconnectProviderResponse {
  status: 'disconnected';
  provider_id: string;
}
export interface CreateProviderRequest {
  id: string;
  name: string;
  description: string;
  base_url: string;
  logo: string;
}
export type CreateProviderResponse = Provider;
export interface DeleteProviderResponse {
  status: 'deleted';
  provider_id: string;
}
export interface SyncModelsResponse {
  status: 'synced';
  provider_id: string;
  models_synced: number;
  total_synced: number;
  total_available: number;
  errors: string[];
}

// ---------- Models ----------
export interface Model {
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
export interface CustomModelRequest {
  base_url: string;
  category: string;
  api_key: string;
  model_id: string;
  name: string;
  context_window: number;
  description: string;
}

// ---------- Benchmarks ----------
export interface BenchmarkTask {
  name: string;
  value: string;
}
export interface Benchmark {
  name: string;
  description: string;
  tasks: BenchmarkTask[];
  task_count: number;
  required_capabilities: string[];
  huggingface_dataset: string;
  type: string;
}
export interface BenchmarksResponse {
  benchmarks: Benchmark[];
  total: number;
}

// ---------- Metrics ----------
export interface MetricsResponse {
  all_metrics: string[];
  custom_agent_metrics: string[];
}

// ---------- Evaluations ----------
export interface JudgeConfig {
  model_id: string;
  base_url: string;
  api_key: string;
}
export interface CreateEvaluationRequest {
  name: string;
  description?: string;
  eval_type: 'model' | 'agent' | 'rag' | string;
  dataset_id: string;
  benchmark?: string;
  model_ids: string[];
  metrics_config?: Record<string, unknown>;
  selected_metrics: string[];
  dataset_limit?: number;
  selected_category?: string[];
  judge_config?: JudgeConfig;
}
export interface CreateEvaluationResponse {
  id?: string;
  evaluation_id?: string;
  [key: string]: unknown;
}
export type EvaluationStatusValue = 'pending' | 'running' | 'completed' | 'failed' | 'canceled';
export interface EvaluationStatusResponse {
  status: EvaluationStatusValue;
  progress: number;
  total: number;
  celery_state: 'STARTED' | 'SUCCESS' | 'FAILURE' | 'REVOKED' | null;
  error_message: string | null;
}

// UI-only aggregate type used while the wizard builds up a draft
export interface EvaluationDraft {
  name: string;
  eval_type: string;
  selProviders: string[];
  selModels: string[];
  selBenchmark: string | null;
  selMetrics: string[];
  judgeModelId?: string;
}
