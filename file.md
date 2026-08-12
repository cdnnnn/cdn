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

// ---------- Model health check ----------
// GET /models/health/{model_id}
export interface ModelHealthResponse {
  success: boolean;
  message: string;
  model_id: string;
  response: string;
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

// ---------- Metrics ----------
// GET /metrics?eval_type={type}
// `type` is one of the New Evaluation wizard's eval types ('model' | 'agent'
// | 'rag'). Only `all_metrics` is consumed by the UI — the `metrics` field
// (a type-specific subset) is intentionally not surfaced anywhere.
export interface MetricsResponse {
  eval_type: string;
  metrics: string[];
  all_metrics: string[];
}

// ---------- Evaluations: create/start ----------
// All fields optional: a run either carries a real judge (model_id,
// base_url, api_key all populated) or is submitted as `{}` when the
// LLM_Judge metric isn't selected — see CreateEvaluationRequest below.
export interface JudgeConfig {
  model_id?: string;
  base_url?: string;
  // NOTE: populated with the judge model's own id, not a real credential —
  // the Judge API Key field was removed from the UI entirely (spec §1.4).
  api_key?: string;
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
  // {} when the LLM_Judge metric isn't selected; a fully-populated
  // JudgeConfig when it is and a judge model has been chosen.
  judge_config?: JudgeConfig;
}
export interface CreateEvaluationResponse {
  id?: string;
  evaluation_id?: string;
  [key: string]: unknown;
}

// ---------- Evaluations: Agent-type launch (bypasses /evaluations) ----------
// POST /agent-benchmark/run — draft.type === 'agent', no agentFramework selected.
export interface AgentBenchmarkRunRequest {
  dataset_id: string;
  model_ids: string[];
  evaluation_name: string;
  run_samples: number;
}
// POST /agent-benchmark/run-multi — draft.type === 'agent', agentFramework selected.
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
}
export interface EvaluationsListResponse {
  evaluations: EvaluationListItem[];
}

// ---------- Evaluations: results ----------
export interface TestDetail {
  test_id: string;
  input: string;
  output: string;
  expected: string;
  latency_seconds: number;
  passed: boolean;
  score: number;
  metric_scores: Record<string, number>;
}
export interface ModelResult {
  model_id: string;
  provider: string;
  rank: number;
  score: number;
  accuracy: number;
  passed_tests: number;
  failed_tests: number;
  total_tests: number;
  metric_scores: Record<string, number>;
  details: TestDetail[];
}
export interface EvaluationResultsResponse {
  evaluation_id: string;
  status: EvaluationStatusValue;
  top_model: string;
  top_score: number;
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
