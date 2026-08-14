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
