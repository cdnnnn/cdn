import api from '../axiosInstance';

// ---- Step 2: templates, built-in checks & placeholders --------------------
export interface MetricTemplate {
  id: string;
  name: string;
  type: string; // e.g. "llm" | "rag"
}

export interface BuiltinCheck {
  id: string;
  name: string;
  logic: string;
}

export interface MetricPlaceholder {
  name: string;
  label: string;
  description: string;
  syntax: string; // e.g. "{input}"
  category: string; // e.g. "basic" | "rag"
}

export interface TemplatesData {
  templates: MetricTemplate[];
  builtin_checks: BuiltinCheck[];
  placeholders: MetricPlaceholder[];
}

// ---- Step 3: starter code for an eval type -------------------------------
export interface CodeTemplateData {
  eval_type: string;
  language: string;
  code_snippet: string;
}

// ---- Step 4: judge models + health ---------------------------------------
export interface ModelSummary {
  id: string;
  name: string;
  provider: string;
}

export interface ModelHealthData {
  is_healthy: boolean;
  latency_ms?: number;
}

// ---- Step 5/6: datasets + preview ------------------------------------------
export interface DatasetSummary {
  id: string;
  name: string;
  question_count: number;
}

export interface PreviewQuestion {
  id: string;
  input: string;
  expected_output: string;
}

export interface DatasetPreviewData {
  dataset_id: string;
  total_questions: number;
  preview_limit: number;
  questions: PreviewQuestion[];
}

// ---- Dashboard: saved custom metrics ---------------------------------
// `definition` shape varies by metric_type — "simple" metrics carry a
// `subtype`/`params` pair, rule-based ones carry a `rules` array. Both are
// optional here since a given metric will only ever populate one branch.
export interface CustomMetricRuleDef {
  field: string;
  operator: string;
  value: string;
  compared_to_field: boolean;
}

export interface CustomMetricDefinition {
  subtype?: string;
  params?: Record<string, unknown>;
  rules?: CustomMetricRuleDef[];
}

export interface CustomMetric {
  id: string;
  name: string;
  description: string;
  metric_type: string; // e.g. "simple" | "code"
  eval_types: string[];
  definition: CustomMetricDefinition;
  requires_judge: boolean;
  threshold: number;
  is_active: boolean;
  created_by_id: number;
  created_at: string;
  updated_at: string;
}

// ---- Step 7/8: dry run + save ---------------------------------------------
export interface MetricConfig {
  name: string;
  description?: string;
  code: string;
  rules: string[];
}

export interface JudgeConfig {
  model_id: string;
}

export interface DryRunRequest {
  metric_config: Pick<MetricConfig, 'name' | 'code' | 'rules'>;
  judge_config: JudgeConfig;
  test_case_ids: string[];
}

export interface DryRunResultRow {
  question_id: string;
  score: number;
  reason: string;
  passed: boolean;
}

export interface DryRunSummary {
  total_run: number;
  passed: number;
  failed: number;
  average_score: number;
}

export interface DryRunData {
  is_valid: boolean;
  execution_time_ms: number;
  results: DryRunResultRow[];
  summary: DryRunSummary;
}

export interface SaveMetricRequest {
  metric_config: MetricConfig & { judge_config: JudgeConfig };
}

export interface SaveMetricData {
  id?: string;
  name?: string;
}

// None of these endpoints wrap their body in a { status, data } envelope —
// every response below is the payload itself, so each call just unwraps
// axios's own `r.data` and normalizes array fields to [] where the backend
// might omit them, same pattern as evaluationsApi.list's normalizeListItem.
export const metricsApi = {
  // Dashboard — GET /metrics/custom -> { metrics: [...] }
  list: () =>
    api.get<{ metrics: CustomMetric[] }>('/metrics/custom').then((r) => r.data.metrics || []),

  // Step 2 — GET /metrics/templates -> { templates, builtin_checks, placeholders }.
  // `placeholders` documents the `{field}` tokens available for use in the
  // code editor (Step 3) — not consumed by the UI yet, but returned here so
  // it's ready when needed.
  getTemplates: () =>
    api.get<TemplatesData>('/metrics/templates').then((r) => ({
      templates: r.data.templates || [],
      builtin_checks: r.data.builtin_checks || [],
      placeholders: r.data.placeholders || [],
    })),

  // Step 3 — "Insert Starter Code". Fired once a template's eval type is
  // known; injects the returned snippet straight into the code editor.
  getCodeTemplate: (evalType: string) =>
    api.get<CodeTemplateData>(`/metrics/code-templates/${evalType}`).then((r) => r.data),

  // Step 4 — judge model list. Called once when the Judge Model step is
  // entered; each model's health is then pinged individually via
  // checkModelHealth so the list can render "Checking…" per row instead of
  // blocking on every ping before showing anything.
  listModels: () =>
    api.get<{ models: ModelSummary[] }>('/models').then((r) => r.data.models || []),

  // Step 4 — per-model health ping. On failure (network error, or a
  // `{status:"error", message}` error body) callers treat the model as
  // unhealthy/offline rather than surfacing the error, so this resolves to
  // a plain ModelHealthData rather than throwing for the "unreachable" case.
  checkModelHealth: (modelId: string) =>
    api
      .get<ModelHealthData | { status: string; message: string }>(`/models/health/${modelId}`)
      .then((r) => ('is_healthy' in r.data ? r.data : { is_healthy: false }))
      .catch(() => ({ is_healthy: false })),

  // Step 5 — datasets filtered by the eval type resolved from the chosen
  // template. Re-fetched whenever eval type changes.
  listDatasets: (evalType: string) =>
    api
      .get<{ total_count: number; datasets: DatasetSummary[] }>('/datasets', { params: { eval_type: evalType } })
      .then((r) => r.data.datasets || []),

  // Step 6 — sample question preview for the selected dataset (first N,
  // per preview_limit). Questions default to [] if the backend omits them.
  previewDataset: (datasetId: string) =>
    api.get<DatasetPreviewData>(`/datasets/${datasetId}/preview`).then((r) => ({
      ...r.data,
      questions: r.data.questions || [],
    })),

  // Step 7 — dry run against the selected test cases. Does not persist
  // anything; only `is_valid` gates whether Step 8's Save button unlocks.
  dryRun: (payload: DryRunRequest) =>
    api.post<DryRunData>('/metrics/custom/preview', payload).then((r) => ({
      ...r.data,
      results: r.data.results || [],
    })),

  // Step 8 — persists the metric. Response body shape isn't fully specified
  // by the spec beyond "200 status", so `id`/`name` are optional here —
  // callers should fall back to a generic success message if `id` is absent.
  create: (payload: SaveMetricRequest) =>
    api.post<SaveMetricData | void>('/metrics/custom', payload).then((r) => r.data || {}),
};
