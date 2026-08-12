import api from '../axiosInstance';
import type { MetricsResponse } from '../../types';

export const metricsApi = {
  // GET /metrics?eval_type={type} — type is 'model' | 'agent' (whatever was
  // chosen in Step 2 of the New Evaluation wizard).
  // Response: { eval_type, metrics, all_metrics }
  list: (evalType: string) =>
    api.get<MetricsResponse>('/metrics', { params: { eval_type: evalType } }).then((r) => r.data),
};









import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { metricsApi } from '../../api/endpoints/metrics';

interface MetricsState {
  allMetrics: string[];
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  error: string | null;
}

const initialState: MetricsState = {
  allMetrics: [],
  status: 'idle',
  error: null,
};

// GET /metrics?eval_type={type} — dispatched only once the user picks a
// type in Step 2 of the New Evaluation wizard (never on mount), and
// re-dispatched whenever the type changes.
export const fetchMetrics = createAsyncThunk('metrics/fetchAll', (evalType: string) =>
  metricsApi.list(evalType)
);

const metricsSlice = createSlice({
  name: 'metrics',
  initialState,
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchMetrics.pending, (state) => {
        state.status = 'loading';
      })
      .addCase(fetchMetrics.fulfilled, (state, action) => {
        state.status = 'succeeded';
        // Only `all_metrics` from the response is retained — `metrics` is
        // intentionally not stored/used.
        state.allMetrics = action.payload.all_metrics;
      })
      .addCase(fetchMetrics.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.error.message || 'Failed to load metrics';
      });
  },
});

export default metricsSlice.reducer;
