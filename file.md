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

// GET /datasets/{id}/preview?limit={limit} — limit is user-chosen, 1–20
// inclusive (spec: Test Suite Change-2). Powers the right-to-left preview
// slider on each dataset card.
export interface DatasetPreviewQuestion {
  id: string;
  input: Record<string, unknown> & { prompt?: string };
  expected: Record<string, unknown> & { answer?: string };
  category: string;
  choices: string[];
}
export interface DatasetPreviewResponse {
  dataset_id: string;
  questions: DatasetPreviewQuestion[];
}

// ---------- Metrics ----------
export interface MetricsResponse {
  all_metrics: string[];
  custom_agent_metrics: string[];
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
}
export interface CreateEvaluationResponse {
  id?: string;
  evaluation_id?: string;
  [key: string]: unknown;
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

// UI-only draft built up across the wizard's 7 steps (spec §6).
export interface EvaluationDraft {
  name: string;
  type: 'model' | 'agent' | 'rag' | null;
  providers: string[];
  models: string[];
  dataset: string | null;
  subgroup: string[];
  runSamples: number; // default 10
  metrics: string[];
  judgeModelId: string | null;
  // judgeApiKey intentionally omitted — no longer collected (spec §1.4)
  agentFramework: string | null;
}
