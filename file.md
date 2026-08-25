//models.ts
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

export interface UpdateCustomModelRequest {
  model_id: string;
  name: string;
  description: string;
}

export interface UpdateCustomModelResponse {
  model_id: string;
  name: string;
  description: string;
}

export const modelsApi = {
  list: () => api.get<{ models: Model[] }>('/models').then((r) => r.data.models ?? []),

  createCustom: (payload: CustomModelRequest) =>
    api.post<void>('/models/custom', payload).then(() => undefined),

  // POST /models/custom — same endpoint as create, keyed by model_id: used here
  // to update just the name/description of an existing custom model.
  updateCustom: (payload: UpdateCustomModelRequest) =>
    api.post<UpdateCustomModelResponse>('/models/custom', payload).then((r) => r.data),

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

















//modelsslice.ts
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
  updatingId: string | null;
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
  updatingId: null,
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

export const updateCustomModel = createAsyncThunk(
  'models/updateCustom',
  async (payload: { model_id: string; name: string; description: string }) => {
    const res = await modelsApi.updateCustom(payload);
    return {
      modelId: res.model_id || payload.model_id,
      name: res.name ?? payload.name,
      description: res.description ?? payload.description,
    };
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
      })
      .addCase(updateCustomModel.pending, (state, action) => {
        state.updatingId = action.meta.arg.model_id;
      })
      .addCase(updateCustomModel.fulfilled, (state, action) => {
        state.updatingId = null;
        const { modelId, name, description } = action.payload;
        const patch = (m: Model) => {
          if (m.id !== modelId) return m;
          return { ...m, name, description } as Model & { description: string };
        };
        state.items = state.items.map(patch);
        for (const providerId of Object.keys(state.byProvider)) {
          state.byProvider[providerId] = state.byProvider[providerId].map(patch);
        }
      })
      .addCase(updateCustomModel.rejected, (state, action) => {
        state.updatingId = null;
        state.error = action.error.message || 'Failed to update model';
      });
  },
});

export default modelsSlice.reducer;


















//Confirmdialog.tsx
import { AlertTriangle, Loader2, X } from 'lucide-react';
import styles from './ConfirmDialog.module.scss';

interface ConfirmDialogProps {
  title: string;
  message: string;
  confirmLabel?: string;
  cancelLabel?: string;
  tone?: 'danger' | 'default';
  loading?: boolean;
  onConfirm: () => void;
  onCancel: () => void;
}

export default function ConfirmDialog({
  title,
  message,
  confirmLabel = 'Confirm',
  cancelLabel = 'Cancel',
  tone = 'default',
  loading = false,
  onConfirm,
  onCancel,
}: ConfirmDialogProps) {
  return (
    <div className={styles['confirm-overlay']} onClick={loading ? undefined : onCancel}>
      <div
        className={styles['confirm-card']}
        role="alertdialog"
        aria-modal="true"
        aria-labelledby="confirm-dialog-title"
        onClick={(e) => e.stopPropagation()}
      >
        <button className={styles['confirm-close']} onClick={onCancel} disabled={loading} aria-label="Close">
          <X size={15} />
        </button>

        <div className={`${styles['confirm-icon']} ${tone === 'danger' ? styles['confirm-icon--danger'] : ''}`}>
          <AlertTriangle size={18} />
        </div>

        <div id="confirm-dialog-title" className={styles['confirm-title']}>{title}</div>
        <p className={styles['confirm-message']}>{message}</p>

        <div className={styles['confirm-actions']}>
          <button className={styles['confirm-btn-ghost']} onClick={onCancel} disabled={loading}>
            {cancelLabel}
          </button>
          <button
            className={`${styles['confirm-btn']} ${tone === 'danger' ? styles['confirm-btn--danger'] : ''}`}
            onClick={onConfirm}
            disabled={loading}
          >
            {loading ? <Loader2 size={14} className={styles['confirm-spin']} /> : confirmLabel}
          </button>
        </div>
      </div>
    </div>
  );
}



















//Confirmdialog.module.scss
@use '../../styles/_variables' as *;

$drawer-base-font: 13px;

.confirm-overlay {
  position: fixed;
  inset: 0;
  bottom: $footer-height;
  background: rgba(17, 24, 39, 0.5);
  z-index: 110;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
  animation: confirm-overlay-in 0.15s ease both;

  font-size: $drawer-base-font;
  @media (min-width: 1800px) { font-size: 16px; }
}
@keyframes confirm-overlay-in { from { opacity: 0; } to { opacity: 1; } }

.confirm-card {
  position: relative;
  width: 100%;
  max-width: 340px;
  padding: 22px 22px 20px;
  background: $surface;
  border-radius: 16px;
  box-shadow: $shadow-4;
  animation: confirm-card-in 0.18s cubic-bezier(0.22, 0.72, 0.16, 1) both;
}
@keyframes confirm-card-in {
  from { opacity: 0; transform: translateY(6px) scale(0.98); }
  to { opacity: 1; transform: translateY(0) scale(1); }
}

.confirm-close {
  position: absolute;
  top: 12px;
  right: 12px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 26px;
  height: 26px;
  border: none;
  border-radius: 7px;
  background: none;
  color: $text-muted;
  cursor: pointer;
  transition: background 0.14s ease, color 0.14s ease;

  &:hover:not(:disabled) { background: $surface-alt; color: $text-primary; }
  &:disabled { opacity: 0.5; cursor: not-allowed; }
}

.confirm-icon {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 40px;
  height: 40px;
  border-radius: 11px;
  margin-bottom: 14px;
  background: $amber-pale;
  color: $amber-dark;

  &--danger {
    background: $red-pale;
    color: #DC2626;
  }
}

.confirm-title {
  font-size: 1.1538em; // 15px / 13px
  font-weight: 750;
  color: $text-primary;
  margin-bottom: 6px;
}

.confirm-message {
  font-size: 1em; // 13px / 13px (base)
  line-height: 1.5;
  color: $text-secondary;
  margin: 0 0 20px;
}

.confirm-actions {
  display: flex;
  justify-content: flex-end;
  gap: 8px;
}

.confirm-btn-ghost {
  padding: 8px 14px;
  border-radius: 9px;
  border: 1px solid $border;
  background: $surface;
  color: $text-secondary;
  font-size: 0.9231em; // 12px / 13px
  font-weight: 650;
  cursor: pointer;
  transition: border-color 0.14s ease, color 0.14s ease, background 0.14s ease;

  &:hover:not(:disabled) { border-color: $text-muted; color: $text-primary; background: $surface-alt; }
  &:disabled { opacity: 0.6; cursor: not-allowed; }
}

.confirm-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 88px;
  padding: 8px 16px;
  border-radius: 9px;
  border: 1px solid $indigo;
  background: $indigo;
  color: #fff;
  font-size: 0.9231em; // 12px / 13px
  font-weight: 700;
  cursor: pointer;
  transition: background 0.14s ease, border-color 0.14s ease;

  &:hover:not(:disabled) { background: $indigo-dark; border-color: $indigo-dark; }
  &:disabled { opacity: 0.65; cursor: not-allowed; }

  &--danger {
    border-color: #DC2626;
    background: #DC2626;

    &:hover:not(:disabled) { background: #B91C1C; border-color: #B91C1C; }
  }
}

.confirm-spin { animation: confirm-spin 0.8s linear infinite; }
@keyframes confirm-spin { to { transform: rotate(360deg); } }


















//Providermodelssidebar.tsx
import { useEffect, useRef, useState } from 'react';
import { X, Loader2, Inbox, Trash2, Pencil, Check, XCircle } from 'lucide-react';
import type { Provider, Model } from '../../types';
import ConfirmDialog from './ConfirmDialog';
import styles from './Providers.module.scss';

// The update-name/description API returns `description`, but the shared
// `Model` type (from GET /models, /models/by-provider) doesn't declare that
// field — extend it locally rather than widen the shared type on a guess.
type ModelWithDescription = Model & { description?: string };

interface ProviderModelsSidebarProps {
  provider: Provider;
  models: Model[];
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  onClose: () => void;
  /** Edit + delete affordances only make sense for the Custom provider. */
  canManage?: boolean;
  deletingId?: string | null;
  updatingId?: string | null;
  onDelete?: (modelId: string) => void;
  onEdit?: (payload: { model_id: string; name: string; description: string }) => void;
}

export default function ProviderModelsSidebar({
  provider,
  models = [],
  status,
  onClose,
  canManage = false,
  deletingId = null,
  updatingId = null,
  onDelete,
  onEdit,
}: ProviderModelsSidebarProps) {
  const [editingId, setEditingId] = useState<string | null>(null);
  const [editName, setEditName] = useState('');
  const [editDescription, setEditDescription] = useState('');
  const [pendingDelete, setPendingDelete] = useState<ModelWithDescription | null>(null);

  const startEdit = (m: ModelWithDescription) => {
    setEditingId(m.id);
    setEditName(m.name ?? '');
    setEditDescription(m.description ?? '');
  };

  const cancelEdit = () => {
    setEditingId(null);
    setEditName('');
    setEditDescription('');
  };

  const saveEdit = (modelId: string) => {
    if (!onEdit || !editName.trim()) return;
    onEdit({ model_id: modelId, name: editName.trim(), description: editDescription.trim() });
  };

  // Once an in-flight update for the row currently being edited finishes
  // (updatingId flips away from this row's id), exit edit mode.
  const prevUpdatingId = useRef<string | null>(null);
  useEffect(() => {
    if (editingId && prevUpdatingId.current === editingId && updatingId !== editingId) {
      setEditingId(null);
      setEditName('');
      setEditDescription('');
    }
    prevUpdatingId.current = updatingId;
  }, [updatingId, editingId]);

  // Same idea for the delete confirmation dialog — close it once the
  // in-flight delete for the pending model finishes.
  const prevDeletingId = useRef<string | null>(null);
  useEffect(() => {
    if (pendingDelete && prevDeletingId.current === pendingDelete.id && deletingId !== pendingDelete.id) {
      setPendingDelete(null);
    }
    prevDeletingId.current = deletingId;
  }, [deletingId, pendingDelete]);

  const confirmDelete = () => {
    if (pendingDelete && onDelete) {
      onDelete(pendingDelete.id);
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

          {status === 'succeeded' && models.map((raw) => {
            const m = raw as ModelWithDescription;
            const isDeleting = deletingId === m.id;
            const isUpdating = updatingId === m.id;
            const isEditing = editingId === m.id;

            return (
              <div
                key={m.id}
                className={`${styles['providers__model-row']} ${isDeleting ? styles['providers__model-row--deleting'] : ''}`}
              >
                {isEditing ? (
                  <div className={styles['providers__model-edit']}>
                    <div className={styles['providers__model-edit-field']}>
                      <label>Model Name</label>
                      <input
                        className={styles['providers__model-edit-input']}
                        value={editName}
                        onChange={(e) => setEditName(e.target.value)}
                        placeholder="Model name"
                        disabled={isUpdating}
                        autoFocus
                      />
                    </div>
                    <div className={styles['providers__model-edit-field']}>
                      <label>Description</label>
                      <textarea
                        className={styles['providers__model-edit-textarea']}
                        rows={2}
                        value={editDescription}
                        onChange={(e) => setEditDescription(e.target.value)}
                        placeholder="Optional description"
                        disabled={isUpdating}
                      />
                    </div>
                    <div className={styles['providers__model-edit-actions']}>
                      <button
                        type="button"
                        className={styles['providers__model-edit-cancel']}
                        onClick={cancelEdit}
                        disabled={isUpdating}
                      >
                        <XCircle size={13} /> Cancel
                      </button>
                      <button
                        type="button"
                        className={styles['providers__model-edit-save']}
                        onClick={() => saveEdit(m.id)}
                        disabled={isUpdating || !editName.trim()}
                      >
                        {isUpdating ? <Loader2 size={13} style={{ animation: 'spin 1.5s linear infinite' }} /> : <Check size={13} />}
                        {isUpdating ? 'Saving…' : 'Save'}
                      </button>
                    </div>
                  </div>
                ) : (
                  <>
                    <div className={styles['providers__model-row-head']}>
                      <span className={styles['providers__model-row-name']}>{m.name ?? 'Unnamed model'}</span>
                      <div className={styles['providers__model-row-head-actions']}>
                        <span className={`badge ${m.is_active ? 'badge-green' : 'badge-gray'}`}>
                          {m.is_active ? 'Active' : 'Inactive'}
                        </span>
                        {canManage && (
                          <>
                            <button
                              type="button"
                              className={styles['providers__model-row-edit']}
                              onClick={() => startEdit(m)}
                              title="Edit model"
                              aria-label={`Edit ${m.name ?? m.id}`}
                            >
                              <Pencil size={13} />
                            </button>
                            <button
                              type="button"
                              className={styles['providers__model-row-delete']}
                              onClick={() => setPendingDelete(m)}
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
                          </>
                        )}
                      </div>
                    </div>

                    {m.description && (
                      <p className={styles['providers__model-row-desc']}>{m.description}</p>
                    )}

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
                  </>
                )}
              </div>
            );
          })}
        </div>
      </aside>

      {pendingDelete && (
        <ConfirmDialog
          title="Remove this model?"
          message={`"${pendingDelete.name ?? pendingDelete.id}" will be permanently removed from ${provider?.name ?? 'this provider'}. This can't be undone.`}
          confirmLabel="Remove Model"
          tone="danger"
          loading={deletingId === pendingDelete.id}
          onCancel={() => setPendingDelete(null)}
          onConfirm={confirmDelete}
        />
      )}
    </>
  );
}




















//Providers.tsx
import { useEffect, useState } from 'react';
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
  const customModelUpdatingId = useAppSelector((s) => s.models.updatingId);
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
          canManage={viewModelsProvider.name === 'Custom'}
          deletingId={customModelDeletingId}
          updatingId={customModelUpdatingId}
          onDelete={(modelId) => {
            dispatch(deleteCustomModel({ modelId, providerId: viewModelsProvider.id })).then(() => {
              dispatch(fetchProviders());
            });
          }}
          onEdit={(payload) => {
            dispatch(updateCustomModel(payload));
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
    margin-top: 8px;
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
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

@keyframes providers-sidebar-in {
  from { transform: translateX(100%); }
  to { transform: translateX(0); }
}

@media (max-width: 768px) {
  .providers__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .providers__toolbar { padding: 12px 18px; }
  .providers__grid { grid-template-columns: 1fr; }
}
