//modelsSlice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { modelsApi } from '../../api/endpoints/models';
import type { Model, CustomModelRequest } from '../../types';

type FetchStatus = 'idle' | 'loading' | 'succeeded' | 'failed';
type HealthStatus = 'idle' | 'loading' | 'success' | 'failed';

interface ModelsState {
  items: Model[];
  status: FetchStatus;
  error: string | null;
  creating: boolean;
  byProvider: Record<string, Model[]>;
  byProviderStatus: Record<string, FetchStatus>;
  // Per-model health-check results, keyed by model id. A model id absent
  // from this map is treated as 'idle' (never checked) by consumers.
  healthById: Record<string, HealthStatus>;
}

const initialState: ModelsState = {
  items: [],
  status: 'idle',
  error: null,
  creating: false,
  byProvider: {},
  byProviderStatus: {},
  healthById: {},
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

// GET /models/:id/health — checks whether a single model is currently
// reachable. Called per-model (e.g. once its provider is selected in the
// New Evaluation wizard) rather than in bulk.
export const checkModelHealth = createAsyncThunk(
  'models/checkHealth',
  async (modelId: string) => {
    const status = await modelsApi.checkHealth(modelId);
    return { modelId, status };
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
      })
      .addCase(checkModelHealth.pending, (state, action) => {
        state.healthById[action.meta.arg] = 'loading';
      })
      .addCase(checkModelHealth.fulfilled, (state, action) => {
        state.healthById[action.payload.modelId] = action.payload.status;
      })
      .addCase(checkModelHealth.rejected, (state, action) => {
        // A failed health-check *request* (network/500) is treated the same
        // as an unhealthy model — either way it must not be selectable.
        state.healthById[action.meta.arg] = 'failed';
      });
  },
});

export default modelsSlice.reducer;






















//models.ts
import api from '../axiosInstance';
import type { Model, CustomModelRequest } from '../../types';

// Shape returned by the individual model health-check endpoint.
// Adjust field names here if your actual API response differs
// (e.g. if it returns `healthy: boolean` instead of `status`).
export interface ModelHealthResponse {
  status: 'success' | 'failed';
}

export const modelsApi = {
  list: () => api.get<{ models: Model[] }>('/models').then((r) => r.data.models),

  createCustom: (payload: CustomModelRequest) =>
    api.post<void>('/models/custom', payload).then(() => undefined),

  // GET /models/by-provider/:providerId — all models registered under a single provider
  listByProvider: (providerId: string) =>
    api
      .get<{ models: Model[]; total: number }>(`/models/by-provider/${providerId}`)
      .then((r) => r.data),

  // GET /models/:id/health — checks whether an individual model is currently
  // reachable/healthy. Update the URL/response mapping here if your actual
  // health-check contract differs.
  checkHealth: (modelId: string) =>
    api.get<ModelHealthResponse>(`/models/${modelId}/health`).then((r) => r.data.status),
};
