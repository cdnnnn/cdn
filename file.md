//Evaluations.ts
import api from '../axiosInstance';
import type {
  AgentBenchmarkRunMultiRequest,
  AgentBenchmarkRunRequest,
  CreateEvaluationRequest,
  CreateEvaluationResponse,
  DatasetPreviewResponse,
  EvaluationsListResponse,
  EvaluationResultsResponse,
  EvaluationListItem,
  ModelResult,
} from '../../types';

// Re-exported for convenience so existing imports of these two request
// types from this module (e.g. in evaluationsSlice.ts) keep working —
// the canonical definitions now live in ../../types.
export type { AgentBenchmarkRunRequest, AgentBenchmarkRunMultiRequest };

// Normalizes one list-item so array fields the UI iterates over
// (model_ids.length, selected_metrics.map, etc.) are never null/undefined,
// even if the backend omits them for a given row. Same normalize-at-the-
// boundary pattern as benchmarksApi.list's `tasks`.
function normalizeListItem(e: EvaluationListItem): EvaluationListItem {
  return {
    ...e,
    model_ids: e.model_ids || [],
    selected_metrics: e.selected_metrics || [],
    selected_category: e.selected_category || [],
    datasets_config: e.datasets_config || [],
  };
}

export const evaluationsApi = {
  // Populates the History sidebar list. Called on mount and every 10s
  // (silent poll) — see History.tsx.
  list: () =>
    api.get<EvaluationsListResponse>('/evaluations').then((r) => (r.data.evaluations || []).map(normalizeListItem)),

  create: (payload: CreateEvaluationRequest) =>
    api.post<CreateEvaluationResponse>('/evaluations', payload).then((r) => r.data),

  start: (evaluationId: string) =>
    api.post<void>(`/evaluations/${evaluationId}/start`).then(() => undefined),

  // Stops a running evaluation — used by the "Stop evaluation" button on a
  // running card in History.tsx (behind a confirm dialog). The backend
  // responds 200 with { status: 'cancelled', evaluation_id } on success.
  cancel: (evaluationId: string) =>
    api.post<{ status: string; evaluation_id: string }>(`/evaluations/${evaluationId}/cancel`).then((r) => r.data),

  // Only ever called when the selected evaluation's status === 'completed'.
  // The backend returns 400 with { detail: "Execution not completed." } if
  // called too early — callers should surface err.response.data.detail.
  //
  // Also normalizes at the boundary: `total_test` (singular, as sent by the
  // API) -> `total_tests`; and `results`/`metric_scores`/`details`/
  // `selected_metrics` default to []/{} when the backend omits them, so
  // downstream code can rely on them always being iterable.
  results: (evaluationId: string) =>
    api.get<EvaluationResultsResponse>(`/evaluations/${evaluationId}/results`).then((r) => {
      const data = r.data;
      return {
        ...data,
        selected_metrics: data.selected_metrics || [],
        results: (data.results || []).map((m) => {
          const raw = m as unknown as ModelResult & { total_test?: number };
          return {
            ...raw,
            total_tests: raw.total_tests ?? raw.total_test ?? 0,
            metric_scores: raw.metric_scores || {},
            details: raw.details || [],
          };
        }),
      };
    }),

  // Convenience helper used by the wizard's "Start Evaluation" (step 7):
  // create, then immediately start. Only for draft.type 'model' | 'rag'.
  createAndStart: async (payload: CreateEvaluationRequest) => {
    const created = await evaluationsApi.create(payload);
    const id = created.id || created.evaluation_id;
    if (!id) {
      throw new Error('Evaluation was created but no id was returned by the server.');
    }
    await evaluationsApi.start(id);
    return id;
  },

  // POST /agent-benchmark/run — draft.type === 'agent', no framework selected.
  // 200 OK response means successful submission; no meaningful body is relied upon.
  runAgentBenchmark: (payload: AgentBenchmarkRunRequest) =>
    api.post<void>('/agent-benchmark/run', payload).then(() => undefined),

  // POST /agent-benchmark/run-multi — draft.type === 'agent', framework selected.
  // 200 OK response means successful submission; no meaningful body is relied upon.
  runAgentBenchmarkMulti: (payload: AgentBenchmarkRunMultiRequest) =>
    api.post<void>('/agent-benchmark/run-multi', payload).then(() => undefined),

  // GET /datasets/{id}/preview?limit={limit}&offset={offset} — lives here
  // (not in the datasets API module) since it's only ever used from the
  // evaluation wizard's Test Suite step preview slider. Paginated, 20
  // questions per page by default (limit=20, offset starts at 0). Total
  // page count is derived on the caller's side from the dataset's own
  // `question_count` (from the /datasets list), not from this response.
  previewDataset: (datasetId: string, limit: number, offset: number) =>
    api
      .get<DatasetPreviewResponse>(`/datasets/${datasetId}/preview`, { params: { limit, offset } })
      .then((r) => r.data),
};



















//Evaluationsslice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import type { PayloadAction } from '@reduxjs/toolkit';
import { evaluationsApi } from '../../api/endpoints/evaluations';
import type { AgentBenchmarkRunMultiRequest, AgentBenchmarkRunRequest } from '../../api/endpoints/evaluations';
import type {
  CreateEvaluationRequest,
  EvaluationDraft,
  EvaluationListItem,
  EvaluationResultsResponse,
} from '../../types';

type AsyncStatus = 'idle' | 'loading' | 'succeeded' | 'failed';

interface EvaluationsState {
  draft: EvaluationDraft;

  // History list (GET /evaluations) — silently re-fetched every 10s from
  // History.tsx. `listStatus` only gates the *initial* loading/error UI;
  // components should check `list.length === 0` alongside it so a failed
  // background poll never shows a spinner/error over existing data (spec §2.4).
  list: EvaluationListItem[];
  listStatus: AsyncStatus;
  listError: string | null;

  // Per-evaluation results (GET /evaluations/{id}/results), fetched lazily
  // and only once status === 'completed' (spec §2.3).
  resultsByEvalId: Record<string, EvaluationResultsResponse>;
  resultsStatusByEvalId: Record<string, AsyncStatus>;
  resultsErrorByEvalId: Record<string, string | null>;

  launching: boolean;
  launchError: string | null;

  // Cancel-in-flight tracking for the "Stop evaluation" action on a running
  // card in History.tsx (confirm dialog -> POST /evaluations/{id}/cancel).
  cancelingId: string | null;
  cancelError: string | null;
}

const initialDraft: EvaluationDraft = {
  name: '',
  type: null,
  providers: [],
  models: [],
  dataset: null,
  subgroup: [],
  runSamplesMode: 'custom',
  runSamples: 10,
  metrics: [],
  judgeModelId: null,
  agentFramework: null,
};

const initialState: EvaluationsState = {
  draft: initialDraft,
  list: [],
  listStatus: 'idle',
  listError: null,
  resultsByEvalId: {},
  resultsStatusByEvalId: {},
  resultsErrorByEvalId: {},
  launching: false,
  launchError: null,
  cancelingId: null,
  cancelError: null,
};

export const fetchEvaluations = createAsyncThunk('evaluations/fetchList', () => evaluationsApi.list());

export const fetchEvaluationResults = createAsyncThunk(
  'evaluations/fetchResults',
  async (evaluationId: string, { rejectWithValue }) => {
    try {
      const data = await evaluationsApi.results(evaluationId);
      return { evaluationId, data };
    } catch (err) {
      const detail =
        (err as { response?: { data?: { detail?: string } } })?.response?.data?.detail ||
        (err as Error)?.message ||
        'Failed to load results';
      return rejectWithValue({ evaluationId, message: detail });
    }
  }
);

// Shared across all three launch thunks below, cancelEvaluation, and
// fetchEvaluationResults above — the backend's error body on 4xx responses
// is { detail: string }, so that's what should end up in state, not axios's
// generic "Request failed with status code 400".
function extractErrorDetail(err: unknown, fallback: string): string {
  return (
    (err as { response?: { data?: { detail?: string } } })?.response?.data?.detail ||
    (err as Error)?.message ||
    fallback
  );
}

// POST /evaluations then /evaluations/{id}/start — draft.type 'model' | 'rag'.
export const launchEvaluation = createAsyncThunk(
  'evaluations/launch',
  async (payload: CreateEvaluationRequest, { rejectWithValue }) => {
    try {
      return await evaluationsApi.createAndStart(payload);
    } catch (err) {
      return rejectWithValue(extractErrorDetail(err, 'Failed to launch evaluation'));
    }
  }
);

// POST /agent-benchmark/run — draft.type 'agent', no agentFramework selected.
export const runAgentBenchmark = createAsyncThunk(
  'evaluations/runAgentBenchmark',
  async (payload: AgentBenchmarkRunRequest, { rejectWithValue }) => {
    try {
      return await evaluationsApi.runAgentBenchmark(payload);
    } catch (err) {
      return rejectWithValue(extractErrorDetail(err, 'Failed to launch agent benchmark'));
    }
  }
);

// POST /agent-benchmark/run-multi — draft.type 'agent', agentFramework selected.
export const runAgentBenchmarkMulti = createAsyncThunk(
  'evaluations/runAgentBenchmarkMulti',
  async (payload: AgentBenchmarkRunMultiRequest, { rejectWithValue }) => {
    try {
      return await evaluationsApi.runAgentBenchmarkMulti(payload);
    } catch (err) {
      return rejectWithValue(extractErrorDetail(err, 'Failed to launch agent benchmark'));
    }
  }
);

// POST /evaluations/{id}/cancel — "Stop evaluation" button on a running
// card in History.tsx, behind a confirm dialog. `evaluationId` is threaded
// through action.meta.arg (createAsyncThunk's default), so the reducers
// below can key cancelingId/list updates off it without it being part of
// the rejected payload.
export const cancelEvaluation = createAsyncThunk(
  'evaluations/cancel',
  async (evaluationId: string, { rejectWithValue }) => {
    try {
      return await evaluationsApi.cancel(evaluationId);
    } catch (err) {
      return rejectWithValue(extractErrorDetail(err, 'Failed to stop evaluation'));
    }
  }
);

const evaluationsSlice = createSlice({
  name: 'evaluations',
  initialState,
  reducers: {
    setDraft(state, action: PayloadAction<Partial<EvaluationDraft>>) {
      state.draft = { ...state.draft, ...action.payload };
    },
    // Step 2: changing type clears any previously selected metrics (spec §1.2).
    setDraftType(state, action: PayloadAction<EvaluationDraft['type']>) {
      state.draft.type = action.payload;
      state.draft.metrics = [];
      if (action.payload !== 'agent') {
        state.draft.agentFramework = null;
      }
    },
    resetDraft(state) {
      state.draft = initialDraft;
    },
    // Local-only removal — no DELETE /evaluations/{id} endpoint exists yet
    // (spec §4.6). Does not persist; a background poll will bring it back
    // if the backend still has it.
    removeEvaluationLocal(state, action: PayloadAction<string>) {
      state.list = state.list.filter((e) => e.id !== action.payload);
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchEvaluations.pending, (state) => {
        if (state.list.length === 0) state.listStatus = 'loading';
      })
      .addCase(fetchEvaluations.fulfilled, (state, action) => {
        state.listStatus = 'succeeded';
        state.listError = null;
        state.list = action.payload;
      })
      .addCase(fetchEvaluations.rejected, (state, action) => {
        // Background polls fail silently (spec §2.4) — only surface the
        // error state when we have nothing on screen yet.
        if (state.list.length === 0) {
          state.listStatus = 'failed';
          state.listError = action.error.message || 'Failed to load evaluations';
        }
      })
      .addCase(fetchEvaluationResults.pending, (state, action) => {
        state.resultsStatusByEvalId[action.meta.arg] = 'loading';
        state.resultsErrorByEvalId[action.meta.arg] = null;
      })
      .addCase(fetchEvaluationResults.fulfilled, (state, action) => {
        const { evaluationId, data } = action.payload;
        state.resultsStatusByEvalId[evaluationId] = 'succeeded';
        state.resultsByEvalId[evaluationId] = data;
      })
      .addCase(fetchEvaluationResults.rejected, (state, action) => {
        const payload = action.payload as { evaluationId: string; message: string } | undefined;
        const id = payload?.evaluationId ?? action.meta.arg;
        state.resultsStatusByEvalId[id] = 'failed';
        state.resultsErrorByEvalId[id] = payload?.message || 'Failed to load results';
      })

      // ---- launch: three thunks (Model/RAG, Agent-benchmark, Agent-multi) ---
      // all share the same launching/launchError flags and all clear the
      // draft on success, exactly like the original launchEvaluation did.
      .addCase(launchEvaluation.pending, (state) => {
        state.launching = true;
        state.launchError = null;
      })
      .addCase(launchEvaluation.fulfilled, (state) => {
        state.launching = false;
        state.draft = initialDraft;
      })
      .addCase(launchEvaluation.rejected, (state, action) => {
        state.launching = false;
        state.launchError = (action.payload as string) || action.error.message || 'Failed to launch evaluation';
      })

      .addCase(runAgentBenchmark.pending, (state) => {
        state.launching = true;
        state.launchError = null;
      })
      .addCase(runAgentBenchmark.fulfilled, (state) => {
        state.launching = false;
        state.draft = initialDraft;
      })
      .addCase(runAgentBenchmark.rejected, (state, action) => {
        state.launching = false;
        state.launchError = (action.payload as string) || action.error.message || 'Failed to launch agent benchmark';
      })

      .addCase(runAgentBenchmarkMulti.pending, (state) => {
        state.launching = true;
        state.launchError = null;
      })
      .addCase(runAgentBenchmarkMulti.fulfilled, (state) => {
        state.launching = false;
        state.draft = initialDraft;
      })
      .addCase(runAgentBenchmarkMulti.rejected, (state, action) => {
        state.launching = false;
        state.launchError = (action.payload as string) || action.error.message || 'Failed to launch agent benchmark';
      })

      // ---- cancel (stop a running evaluation) --------------------------------
      .addCase(cancelEvaluation.pending, (state, action) => {
        state.cancelingId = action.meta.arg;
        state.cancelError = null;
      })
      .addCase(cancelEvaluation.fulfilled, (state, action) => {
        state.cancelingId = null;
        // Optimistically flip the row to 'canceled' right away rather than
        // waiting for the next 10s background poll to pick it up.
        const item = state.list.find((e) => e.id === action.payload.evaluation_id);
        if (item) item.status = 'canceled';
      })
      .addCase(cancelEvaluation.rejected, (state, action) => {
        state.cancelingId = null;
        state.cancelError = (action.payload as string) || action.error.message || 'Failed to stop evaluation';
      });
  },
});

export const { setDraft, setDraftType, resetDraft, removeEvaluationLocal } = evaluationsSlice.actions;
export default evaluationsSlice.reducer;
