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
  EvaluationProgressResponse,
  ModelResult,
  GenerateInstructionRequest,
  GenerateInstructionResponse,
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

export interface ListEvaluationsParams {
  offset: number;
  limit: number;
  // Omit (leave undefined) rather than passing 'All' — axios drops
  // undefined params from the query string automatically, which is how we
  // satisfy "don't send status/eval_type at all when no filter is applied".
  status?: string;
  eval_type?: string;
}

export const evaluationsApi = {
  // Populates the History sidebar list. Called on mount, on page/page-size/
  // filter changes, and every 10s (silent poll) — see History.tsx. Backed by
  // GET /evaluations?status=&eval_type=&offset=&limit= with a { evaluations,
  // total } response; `total` is the full filtered count across all pages,
  // not just this page's length, and drives the pagination bar.
  list: (params: ListEvaluationsParams) =>
    api.get<EvaluationsListResponse>('/evaluations', { params }).then((r) => ({
      evaluations: (r.data.evaluations || []).map(normalizeListItem),
      total: r.data.total ?? 0,
    })),

  create: (payload: CreateEvaluationRequest) =>
    api.post<CreateEvaluationResponse>('/evaluations', payload).then((r) => r.data),

  start: (evaluationId: string) =>
    api.post<void>(`/evaluations/${evaluationId}/start`).then(() => undefined),

  // Stops a running evaluation — used by the "Stop evaluation" button on a
  // running card in History.tsx (behind a confirm dialog). The backend
  // responds 200 with { status: 'cancelled', evaluation_id } on success.
  cancel: (evaluationId: string) =>
    api.post<{ status: string; evaluation_id: string }>(`/evaluations/${evaluationId}/cancel`).then((r) => r.data),

  // Deletes an evaluation outright — used by the "Delete" action on a
  // non-running card in History.tsx (behind a confirm dialog). The backend
  // responds 200 with { status: 'deleted', evaluation_id } on success.
  remove: (evaluationId: string) =>
    api.delete<{ status: string; evaluation_id: string }>(`/evaluations/${evaluationId}`).then((r) => r.data),

  // Polled for 'running' rows only, right after each list() fetch — see
  // History.tsx's fetchEvaluations effect. Note: the path segment is named
  // `dataset_id` in the API spec, but the id passed is the evaluation's own
  // id (same one used everywhere else — cancel, remove, results, etc.).
  //
  // Normalizes celery_state to a concrete { current, total } (defaulting
  // to 0/0) if the backend omits it, so the percentage math in History.tsx
  // never has to deal with an entirely missing object.
  getProgress: (evaluationId: string) =>
    api.get<EvaluationProgressResponse>(`/evaluations/${evaluationId}/status`).then((r) => {
      const data = r.data;
      return {
        ...data,
        progress: data.progress ?? 0,
        total: data.total ?? 0,
        celery_state: {
          current: data.celery_state?.current ?? 0,
          total: data.celery_state?.total ?? 0,
        },
      };
    }),

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
            metric_breakdown: raw.metric_breakdown || {},
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

  // POST /evaluations/generate-instruction — Metrics step's "Generate
  // Instruction" button. See NewEvaluation.tsx `generateInstruction` for
  // how model_id/eval_type/questions are assembled.
  generateInstruction: (payload: GenerateInstructionRequest) =>
    api.post<GenerateInstructionResponse>('/evaluations/generate-instruction', payload).then((r) => r.data),
};
