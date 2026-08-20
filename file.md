//Addcustommodeldrawer.tsx
import { useEffect, useState } from 'react';
import { X, Search, Loader2, ChevronDown, CheckCircle2, RotateCcw, AlertCircle } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchModelCategories } from '../../store/slices/modelsSlice';
import { modelsApi, type DiscoveredModel } from '../../api/endpoints/models';
import type { CustomModelRequest } from '../../types';
import styles from './AddCustomModelDrawer.module.scss';

interface Props {
  onClose: () => void;
  onSubmit: (payload: CustomModelRequest) => void;
  submitting: boolean;
}

// Fields that describe the actual model (name/id/context window) start out
// hidden — the user either runs discovery against the base URL to auto-fill
// them, or opts into filling them in by hand.
type FieldsMode = 'hidden' | 'manual' | 'discovered';

export default function AddCustomModelDrawer({ onClose, onSubmit, submitting }: Props) {
  const dispatch = useAppDispatch();
  const categories = useAppSelector((s) => s.models.categories);
  const categoriesStatus = useAppSelector((s) => s.models.categoriesStatus);

  const [baseUrl, setBaseUrl] = useState('');
  const [apiKey, setApiKey] = useState('');
  const [category, setCategory] = useState('');
  const [description, setDescription] = useState('');

  const [fieldsMode, setFieldsMode] = useState<FieldsMode>('hidden');
  const [name, setName] = useState('');
  const [modelId, setModelId] = useState('');
  const [contextWindowInput, setContextWindowInput] = useState('');
  const [contextWindowLocked, setContextWindowLocked] = useState(false);

  const [discoverStatus, setDiscoverStatus] = useState<'idle' | 'loading' | 'error'>('idle');
  const [discoverError, setDiscoverError] = useState<string | null>(null);
  const [discoveredModels, setDiscoveredModels] = useState<DiscoveredModel[]>([]);
  const [selectedModelId, setSelectedModelId] = useState<string | null>(null);

  useEffect(() => {
    if (categoriesStatus === 'idle') dispatch(fetchModelCategories());
  }, [dispatch, categoriesStatus]);

  const resetModelFields = () => {
    setFieldsMode('hidden');
    setName('');
    setModelId('');
    setContextWindowInput('');
    setContextWindowLocked(false);
    setDiscoveredModels([]);
    setSelectedModelId(null);
    setDiscoverStatus('idle');
    setDiscoverError(null);
  };

  const selectDiscoveredModel = (m: DiscoveredModel) => {
    setSelectedModelId(m.id);
    setName(m.name ?? '');
    setModelId(m.id);
    const cw = m.context_window ?? m.max_model_len;
    if (cw != null) {
      setContextWindowInput(String(cw));
      setContextWindowLocked(true);
    } else {
      // Endpoint didn't report a context length — fall back to manual entry
      // for just this field since there's nothing authoritative to lock to.
      setContextWindowInput('');
      setContextWindowLocked(false);
    }
  };

  const handleDiscover = async () => {
    if (!baseUrl.trim()) return;
    setDiscoverStatus('loading');
    setDiscoverError(null);
    try {
      const res = await modelsApi.discover({
        base_url: baseUrl.trim(),
        api_key: apiKey.trim() || undefined,
      });
      if (res.models.length === 0) {
        setDiscoverStatus('error');
        setDiscoverError(res.errors[0] || 'No models were found at this endpoint.');
        return;
      }
      setDiscoverStatus('idle');
      setDiscoveredModels(res.models);
      setFieldsMode('discovered');
      if (res.models.length === 1) {
        selectDiscoveredModel(res.models[0]);
      } else {
        setSelectedModelId(null);
      }
    } catch (err) {
      setDiscoverStatus('error');
      setDiscoverError(err instanceof Error ? err.message : 'Could not reach this endpoint.');
    }
  };

  const contextWindowValid = contextWindowInput.trim() !== '' && !Number.isNaN(Number(contextWindowInput)) && Number(contextWindowInput) > 0;

  const valid =
    name.trim() &&
    modelId.trim() &&
    category.trim() &&
    baseUrl.trim() &&
    contextWindowValid;

  const handleSubmit = () => {
    if (!valid) return;
    onSubmit({
      name: name.trim(),
      model_id: modelId.trim(),
      category: category.trim(),
      base_url: baseUrl.trim(),
      api_key: apiKey,
      context_window: Number(contextWindowInput),
      description: description.trim(),
    });
  };

  return (
    <div className={styles['drawer-overlay']} onClick={onClose}>
      <div className={styles.drawer} onClick={(e) => e.stopPropagation()}>
        <div className={styles['drawer__hdr']}>
          <h2>Register Custom Model</h2>
          <button className={styles['drawer__close']} onClick={onClose}><X size={18} /></button>
        </div>

        <div className={styles['drawer__body']}>
          <div className="fg">
            <label className="fl">Base URL</label>
            <input
              className="fi"
              value={baseUrl}
              onChange={(e) => { setBaseUrl(e.target.value); }}
              placeholder="https://…"
            />
          </div>

          <div className="fg">
            <label className="fl">API Key <span className="opt">(optional)</span></label>
            <input
              className="fi"
              type="password"
              value={apiKey}
              onChange={(e) => setApiKey(e.target.value)}
              placeholder="Paste API key…"
            />
          </div>

          <div className="fg">
            <label className="fl">Category</label>
            <div className={styles['select-wrap']}>
              <select
                className={styles['select-fi']}
                value={category}
                onChange={(e) => setCategory(e.target.value)}
                disabled={categoriesStatus === 'loading'}
              >
                <option value="" disabled>
                  {categoriesStatus === 'loading' ? 'Loading categories…' : 'Select a category'}
                </option>
                {categories.map((c) => (
                  <option key={c.value} value={c.value}>{c.label}</option>
                ))}
              </select>
              <ChevronDown size={14} className={styles['select-caret']} />
            </div>
            {categoriesStatus === 'failed' && (
              <div className={styles['field-hint--error']}>Couldn't load categories. You can still type once available, or retry later.</div>
            )}
            {category && categories.find((c) => c.value === category)?.description && (
              <div className={styles['field-hint']}>{categories.find((c) => c.value === category)?.description}</div>
            )}
          </div>

          {/* --- Model identity: hidden until discovered, or the user opts into manual entry --- */}
          {fieldsMode === 'hidden' && (
            <div className={styles['discover-panel']}>
              <div className={styles['discover-panel-text']}>
                <Search size={14} />
                <span>Discover the models this endpoint serves, or enter the details yourself.</span>
              </div>
              <div className={styles['discover-panel-actions']}>
                <button
                  type="button"
                  className={styles['discover-btn']}
                  disabled={!baseUrl.trim() || discoverStatus === 'loading'}
                  onClick={handleDiscover}
                >
                  {discoverStatus === 'loading' ? (
                    <Loader2 size={14} className={styles['spin']} />
                  ) : (
                    <Search size={14} />
                  )}
                  Discover Models
                </button>
                <button type="button" className={styles['discover-btn-ghost']} onClick={() => setFieldsMode('manual')}>
                  Enter Manually
                </button>
              </div>
              {discoverStatus === 'error' && (
                <div className={styles['field-hint--error']}>
                  <AlertCircle size={13} /> {discoverError}
                </div>
              )}
            </div>
          )}

          {fieldsMode === 'discovered' && discoveredModels.length > 1 && (
            <div className="fg">
              <label className="fl">
                Discovered Models <span className="opt">({discoveredModels.length} found — select one)</span>
              </label>
              <div className={styles['discovered-list']}>
                {discoveredModels.map((m) => (
                  <button
                    type="button"
                    key={m.id}
                    className={`${styles['discovered-row']} ${selectedModelId === m.id ? styles['discovered-row--selected'] : ''}`}
                    onClick={() => selectDiscoveredModel(m)}
                  >
                    <div className={styles['discovered-row-main']}>
                      <span className={styles['discovered-row-name']}>{m.name || m.id}</span>
                      <span className={styles['discovered-row-id']}>{m.id}</span>
                    </div>
                    <div className={styles['discovered-row-meta']}>
                      {(m.context_window ?? m.max_model_len) != null && (
                        <span>{(m.context_window ?? m.max_model_len)!.toLocaleString()} ctx</span>
                      )}
                      {m.owned_by && <span>{m.owned_by}</span>}
                      {m.already_added && <span className={styles['already-added-badge']}>Already added</span>}
                    </div>
                    {selectedModelId === m.id && <CheckCircle2 size={16} className={styles['discovered-row-check']} />}
                  </button>
                ))}
              </div>
            </div>
          )}

          {(fieldsMode === 'manual' || (fieldsMode === 'discovered' && selectedModelId)) && (
            <>
              <div className="fg">
                <label className="fl">Model Name</label>
                <input className="fi" value={name} onChange={(e) => setName(e.target.value)} placeholder="e.g. My Fine-tuned Llama" />
              </div>

              <div className="fg">
                <label className="fl">
                  Model ID {fieldsMode === 'discovered' && <span className="opt">(locked from discovery)</span>}
                </label>
                <input
                  className="fi"
                  value={modelId}
                  onChange={(e) => setModelId(e.target.value)}
                  placeholder="e.g. llama-3-70b-custom"
                  disabled={fieldsMode === 'discovered'}
                />
              </div>

              <div className="fg">
                <label className="fl">
                  Context Window {contextWindowLocked && <span className="opt">(locked from discovery)</span>}
                </label>
                <input
                  className="fi"
                  type="number"
                  min={1}
                  value={contextWindowInput}
                  onChange={(e) => setContextWindowInput(e.target.value)}
                  placeholder="e.g. 128000"
                  disabled={contextWindowLocked}
                />
                {fieldsMode === 'discovered' && !contextWindowLocked && (
                  <div className={styles['field-hint']}>This endpoint didn't report a context length — enter it manually.</div>
                )}
              </div>

              <button type="button" className={styles['reset-link']} onClick={resetModelFields}>
                <RotateCcw size={12} /> {fieldsMode === 'discovered' ? 'Use a different endpoint' : 'Try auto-discovery instead'}
              </button>
            </>
          )}

          <div className="fg">
            <label className="fl">Description <span className="opt">(optional)</span></label>
            <textarea className="fi" rows={3} value={description} onChange={(e) => setDescription(e.target.value)} />
          </div>
        </div>

        <div className={styles['drawer__foot']}>
          <button className="btn btn-ghost" onClick={onClose}>Cancel</button>
          <button className="btn btn-ind" disabled={!valid || submitting} onClick={handleSubmit}>{submitting ? 'Registering…' : 'Register Model'}</button>
        </div>
      </div>
    </div>
  );
}























//Addcustommodel.module.scss
@use '../../styles/_variables' as *;

.drawer-overlay {
  position: fixed; top: 0; left: 0; right: 0; bottom: $footer-height;
  background: rgba(17, 24, 39, .4); z-index: 100;
  display: flex; justify-content: flex-end;
}
.drawer {
  width: 420px; max-width: 100%; height: calc(100% - 30px); background: $surface; box-shadow: $shadow-4;
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

// ---- category select -------------------------------------------------------
.select-wrap {
  position: relative;
}
.select-fi {
  width: 100%;
  appearance: none;
  -webkit-appearance: none;
  font: inherit;
  font-size: 13px;
  color: $text-primary;
  background: $surface;
  border: 1px solid $border;
  border-radius: 8px;
  padding: 9px 32px 9px 11px;
  cursor: pointer;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;

  &:hover:not(:disabled) { border-color: $indigo; }
  &:focus { outline: none; border-color: $indigo; box-shadow: 0 0 0 3px $indigo-pale; }
  &:disabled { cursor: not-allowed; opacity: 0.6; }
}
.select-caret {
  position: absolute;
  top: 50%;
  right: 11px;
  transform: translateY(-50%);
  color: $text-muted;
  pointer-events: none;
}

// ---- hint / error text -------------------------------------------------------
.field-hint {
  margin-top: 6px;
  font-size: 12px;
  line-height: 1.45;
  color: $text-secondary;
}
.field-hint--error {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-top: 8px;
  font-size: 12px;
  color: #DC2626;
}

// ---- optional-discovery panel -------------------------------------------------
.discover-panel {
  margin-bottom: 16px;
  padding: 14px;
  border: 1px dashed $border;
  border-radius: 10px;
  background: $surface-alt;
}
.discover-panel-text {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  font-size: 12.5px;
  line-height: 1.5;
  color: $text-secondary;
  margin-bottom: 12px;

  svg { flex-shrink: 0; margin-top: 1px; color: $text-muted; }
}
.discover-panel-actions {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}
.discover-btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 7px 12px;
  border-radius: 8px;
  border: 1px solid $indigo;
  background: $indigo;
  color: #fff;
  font-size: 12.5px;
  font-weight: 700;
  cursor: pointer;
  transition: background 0.15s ease, border-color 0.15s ease;

  &:hover:not(:disabled) { background: $indigo-dark; border-color: $indigo-dark; }
  &:disabled { opacity: 0.55; cursor: not-allowed; }
}
.discover-btn-ghost {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 7px 12px;
  border-radius: 8px;
  border: 1px solid $border;
  background: $surface;
  color: $text-secondary;
  font-size: 12.5px;
  font-weight: 650;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease;

  &:hover { border-color: $text-muted; color: $text-primary; }
}
.spin { animation: add-custom-model-spin 0.8s linear infinite; }
@keyframes add-custom-model-spin { to { transform: rotate(360deg); } }

// ---- discovered model picker -------------------------------------------------
.discovered-list {
  display: flex;
  flex-direction: column;
  gap: 6px;
  max-height: 220px;
  overflow-y: auto;
  padding: 2px;
}
.discovered-row {
  position: relative;
  display: flex;
  flex-direction: column;
  gap: 4px;
  width: 100%;
  padding: 10px 34px 10px 12px;
  border: 1.5px solid $border;
  border-radius: 9px;
  background: $surface;
  cursor: pointer;
  text-align: left;
  transition: border-color 0.14s ease, background 0.14s ease;

  &:hover { border-color: $indigo; background: $indigo-pale; }

  &--selected {
    border-color: $indigo;
    background: $indigo-pale;
  }
}
.discovered-row-main {
  display: flex;
  align-items: baseline;
  gap: 8px;
  min-width: 0;
}
.discovered-row-name {
  font-weight: 700;
  font-size: 13px;
  color: $text-primary;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.discovered-row-id {
  flex-shrink: 0;
  font-family: monospace;
  font-size: 11px;
  color: $text-muted;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.discovered-row-meta {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  font-size: 11px;
  color: $text-secondary;
}
.already-added-badge {
  padding: 1px 7px;
  border-radius: 999px;
  background: rgba(245, 158, 11, 0.16);
  color: #B45309;
  font-weight: 700;
}
.discovered-row-check {
  position: absolute;
  top: 50%;
  right: 10px;
  transform: translateY(-50%);
  color: $indigo;
}

// ---- reset / mode-switch link -------------------------------------------------
.reset-link {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  margin: -6px 0 16px;
  padding: 0;
  border: none;
  background: none;
  font-size: 12px;
  font-weight: 650;
  color: $indigo;
  cursor: pointer;

  &:hover { text-decoration: underline; }
}


























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

export const modelsApi = {
  list: () => api.get<{ models: Model[] }>('/models').then((r) => r.data.models ?? []),

  createCustom: (payload: CustomModelRequest) =>
    api.post<void>('/models/custom', payload).then(() => undefined),

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
      });
  },
});

export default modelsSlice.reducer;
