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

// Shared across all three launch thunks below (and fetchEvaluationResults
// above) — the backend's error body on 4xx responses is { detail: string },
// so that's what should end up in state.launchError, not axios's generic
// "Request failed with status code 400".
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
      });
  },
});

export const { setDraft, setDraftType, resetDraft, removeEvaluationLocal } = evaluationsSlice.actions;
export default evaluationsSlice.reducer;
