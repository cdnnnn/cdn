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
  // ⚠️ Not always present on the real API response — normalized to [] at the
  // fetch boundary (benchmarksApi.list), so consumers can trust these are
  // always arrays. See spec §5 "Known data-contract gap".
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

// ---------- Datasets (Test Suite step) ----------
// `dataset_type` distinguishes the built-in DeepEval suites from datasets a
// user uploaded themselves. Both are shown in the Test Suite grid; only
// 'custom' gets the "Custom" tag, and both are filterable via the
// All / Custom / Deepeval chip group in the step header (spec: Test Suite
// Change-1).
export type DatasetType = 'custom' | 'deepeval' | string;

export interface Dataset {
  id: string;
  name: string;
  description?: string;
  category: string;
  eval_type: string;
  dataset_type: DatasetType;
  question_count: number;
  dataset_categories: string[];
}
export interface DatasetsResponse {
  datasets: Dataset[];
}

// GET /datasets/{id}/preview?limit={limit}&offset={offset} — paginated,
// 20 per page by default. Powers the right-to-left preview slider on each
// dataset card.
//
// Two response shapes are both observed in practice for `input`/`expected`
// (and whether `choices`/`metadata`/`subgroup` are present at all depends
// on the dataset), so every field below is optional except `id` — the UI
// renders whichever subset actually shows up rather than assuming one
// fixed shape:
//   Shape A (e.g. multiple-choice): input.prompt, expected.answer, choices
//   Shape B (e.g. RAG/retrieval):   input.question/source/type,
//                                   expected.answer/doc_id/section_id,
//                                   metadata, subgroup
export interface DatasetPreviewQuestion {
  id: string;
  input?: {
    prompt?: string;
    question?: string;
    source?: string;
    type?: string;
    [key: string]: unknown;
  };
  expected?: {
    answer?: string;
    doc_id?: string;
    section_id?: number;
    [key: string]: unknown;
  };
  metadata?: Record<string, unknown>;
  category?: string;
  subgroup?: string;
  choices?: string[];
}
export interface DatasetPreviewResponse {
  dataset_id: string;
  questions: DatasetPreviewQuestion[];
}

// ---------- Metrics ----------
// GET /metrics?eval_type={type}
export interface CustomMetric {
  id: string;
  name: string;
  metrics_type: string;
  // When true, this specific custom metric needs a judge model to grade
  // it — surfaced as a small indicator on its chip. Doesn't gate whether
  // the Judge Model picker itself shows (that's now always shown/mandatory
  // regardless of metric selection — see NewEvaluation.tsx).
  required_judge: boolean;
  eval_types: string[];
  description: string;
}
export interface MetricsResponse {
  eval_type: string;
  all_metrics: string[];
  custom: CustomMetric[];
}

// ---------- Evaluations: create/start ----------
export interface JudgeConfig {
  model_id: string;
  base_url: string;
  // NOTE: populated with the judge model's own id, not a real credential —
  // the Judge API Key field was removed from the UI entirely (spec §1.4).
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
  run_samples: number;
  selected_category?: string[];
  judge_config?: JudgeConfig;
  // RAG-only — how many retrieved documents/chunks to consider per query.
  // Test Suite step shows this input only when draft.type === 'rag'
  // (default 5, 1–50); omitted entirely for Model/Agent.
  top_k?: number;
  // Free-text evaluation instruction — optional. Either typed by the user
  // directly or pre-filled via POST /evaluations/generate-instruction and
  // then edited. See GenerateInstructionRequest/Response below.
  instruction?: string;
}
export interface CreateEvaluationResponse {
  id?: string;
  evaluation_id?: string;
  [key: string]: unknown;
}

// POST /evaluations/generate-instruction — Metrics step's "Generate
// Instruction" button. `questions` is built from a 5-question dataset
// preview (see NewEvaluation.tsx `generateInstruction`), reusing the same
// `input`/`expected` shapes as DatasetPreviewQuestion.
export interface GenerateInstructionQuestion {
  input?: DatasetPreviewQuestion['input'];
  expected?: DatasetPreviewQuestion['expected'];
}
export interface GenerateInstructionRequest {
  model_id: string;
  eval_type: string;
  questions: GenerateInstructionQuestion[];
}
export interface GenerateInstructionResponse {
  instruction: string;
}

// ---------- Evaluations: agent-benchmark launch ----------
// draft.type === 'agent'. Which request shape is sent depends on whether
// an agent framework was chosen in Step 2 (see NewEvaluation.tsx `launch`):
//   no framework  -> POST /agent-benchmark/run       (AgentBenchmarkRunRequest)
//   framework set -> POST /agent-benchmark/run-multi (AgentBenchmarkRunMultiRequest)
export interface AgentBenchmarkRunRequest {
  dataset_id: string;
  model_ids: string[];
  evaluation_name: string;
  run_samples: number;
}
export interface AgentBenchmarkRunMultiRequest {
  dataset_id: string;
  model_ids: string[];
  evaluation_name: string;
  selected_metrics: string[];
  selected_categories: string[];
  run_samples: number;
}

// ---------- Evaluations: list (History) ----------
export type EvaluationStatusValue = 'pending' | 'running' | 'completed' | 'failed' | 'canceled';

// Nested summary of the report generated for this evaluation, if any.
// Only present once the backend has created a report row for the eval —
// absent/undefined while the eval is still pending/running with no report yet.
export interface EvaluationReportSummary {
  report_id: string;
  title: string;
  status: string;
  created_at: string;
}

export interface EvaluationListItem {
  id: string;
  name: string;
  description: string;
  eval_type: string;
  dataset_id: string;
  datasets_config: { dataset_id: string }[];
  benchmark: string;
  model_ids: string[];
  selected_metrics: string[];
  run_samples: number;
  selected_category: string[];
  status: EvaluationStatusValue;
  progress: number;
  total_questions: number;
  top_model: string | null;
  top_score: number | null;
  created_at: string;
  started_at: string | null;
  completed_at: string | null;
  // Present once a report has been generated for this evaluation (spec: new
  // "download from History" requirement). When `report.report_id` is set,
  // History should offer the same download options as the Reports page.
  report?: EvaluationReportSummary | null;
}
export interface EvaluationsListResponse {
  evaluations: EvaluationListItem[];
  // Total row count across all pages (not just the current page's length) —
  // added alongside GET /evaluations?status=&eval_type=&offset=&limit=
  // pagination support. Drives History's pagination bar (evaluationsSlice.ts
  // `total`/`page`/`pageSize`).
  total: number;
}

// ---------- Evaluations: results ----------
export interface TestDetail {
  task: string;
  input: string;
  expected_output: string;
  actual_output: string;
  passed: boolean;
}
export interface ModelResult {
  model_id: string;
  provider: string | null;
  rank: number;
  score: number;
  accuracy: number;
  passed_tests: number;
  failed_tests: number;
  // Normalized from the API's `total_test` (singular) at the fetch boundary
  // (evaluationsApi.results) — see benchmarksApi.list for the same pattern.
  total_tests: number;
  metric_scores: Record<string, number>;
  details: TestDetail[];
}
export interface EvaluationResultsResponse {
  evaluation_id: string;
  name: string;
  eval_type: string;
  dataset_id: string;
  benchmark: string;
  model_ids: string[];
  selected_metrics: string[];
  status: EvaluationStatusValue;
  total_questions: number;
  top_model: string;
  top_score: number;
  started_at: string | null;
  results: ModelResult[];
}

// Per-model retry override under retryConfigMode === 'individual' — see
// EvaluationDraft.modelRetryConfig below. Keyed by model id in the draft.
export interface ModelRetryConfig {
  max_retries: number;
  timeout: number;
}

// UI-only draft built up across the wizard's 7 steps (spec §6).
export interface EvaluationDraft {
  name: string;
  type: 'model' | 'agent' | 'rag' | null;
  providers: string[];
  models: string[];
  // 'individual': each selected model can have its own max_retries/timeout,
  // stored per model id in modelRetryConfig. 'all': a single retryConfigAll
  // applies to every selected model. Default 'individual'.
  retryConfigMode: 'individual' | 'all';
  // Used when retryConfigMode === 'all'.
  retryConfigAll: ModelRetryConfig;
  // Used when retryConfigMode === 'individual' — keyed by model id; a model
  // with no entry yet falls back to sensible defaults in the UI.
  modelRetryConfig: Record<string, ModelRetryConfig>;
  dataset: string | null;
  subgroup: string[];
  // 'custom': runSamples is a user-entered count, sent as-is.
  // 'full': the whole dataset is used — runSamples is sent as 0 regardless
  // of the last custom value entered (see NewEvaluation.tsx `launch`).
  runSamplesMode: 'custom' | 'full'; // default 'custom'
  runSamples: number; // default 10 — only meaningful when runSamplesMode === 'custom'
  metrics: string[];
  judgeModelId: string | null;
  // judgeApiKey intentionally omitted — no longer collected (spec §1.4)
  agentFramework: string | null;
  // RAG-only — see CreateEvaluationRequest.top_k. Only meaningful (and
  // only shown in the UI) when type === 'rag'.
  topK: number; // default 5, 1–50
  // Optional free-text evaluation instruction (Metrics step). Either
  // typed directly or pre-filled via the "Generate Instruction" button
  // and then edited — always editable either way.
  instruction: string; // default ''
  // When true, a question a model gets wrong is retried up to
  // retestMaxRounds additional times before being scored as failed.
  retestOnWrong: boolean; // default false
  retestMaxRounds: number; // default 3 — only meaningful when retestOnWrong
  // Which metric's pass/fail determines whether a retest is triggered;
  // null falls back to the run's primary/first selected metric.
  retestVerifyMetric: string | null; // default null
}
