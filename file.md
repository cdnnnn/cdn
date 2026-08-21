import api from '../axiosInstance';

// ---- Step 2: templates & built-in checks --------------------------------
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

export interface TemplatesData {
  templates: MetricTemplate[];
  builtin_checks: BuiltinCheck[];
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

// Every endpoint here responds with the same { status, data } envelope, so
// each call unwraps to `data` at the boundary — callers never see `status`.
type Envelope<T> = { status: string; data: T };

export const metricsApi = {
  // Populates Step 2 (template dropdown + built-in check list). Normalizes
  // both arrays to [] so the UI can map/filter without null checks, same
  // as evaluationsApi.list's normalizeListItem.
  getTemplates: () =>
    api.get<Envelope<TemplatesData>>('/metrics/templates').then((r) => ({
      templates: r.data.data.templates || [],
      builtin_checks: r.data.data.builtin_checks || [],
    })),

  // Step 3 — "Insert Starter Code". Fired once a template's eval type is
  // known; injects the returned snippet straight into the code editor.
  getCodeTemplate: (evalType: string) =>
    api.get<Envelope<CodeTemplateData>>(`/metrics/code-templates/${evalType}`).then((r) => r.data.data),

  // Step 4 — judge model list. Called once when the Judge Model step is
  // entered; each model's health is then pinged individually via
  // checkModelHealth so the list can render "Checking…" per row instead of
  // blocking on every ping before showing anything.
  listModels: () =>
    api.get<Envelope<{ models: ModelSummary[] }>>('/models').then((r) => r.data.data.models || []),

  // Step 4 — per-model health ping. On failure (network error or a
  // `{status:"error", message}` body) callers treat the model as
  // unhealthy/offline rather than surfacing the error, so this resolves to
  // a plain ModelHealthData rather than throwing for the "unreachable" case.
  checkModelHealth: (modelId: string) =>
    api
      .get<Envelope<ModelHealthData> | { status: string; message: string }>(`/models/health/${modelId}`)
      .then((r) => {
        if (r.data.status === 'success' && 'data' in r.data) return r.data.data;
        return { is_healthy: false };
      })
      .catch(() => ({ is_healthy: false })),

  // Step 5 — datasets filtered by the eval type resolved from the chosen
  // template. Re-fetched whenever eval type changes.
  listDatasets: (evalType: string) =>
    api
      .get<Envelope<{ total_count: number; datasets: DatasetSummary[] }>>('/datasets', { params: { eval_type: evalType } })
      .then((r) => r.data.data.datasets || []),

  // Step 6 — sample question preview for the selected dataset (first N,
  // per preview_limit). Questions default to [] if the backend omits them.
  previewDataset: (datasetId: string) =>
    api.get<Envelope<DatasetPreviewData>>(`/datasets/${datasetId}/preview`).then((r) => ({
      ...r.data.data,
      questions: r.data.data.questions || [],
    })),

  // Step 7 — dry run against the selected test cases. Does not persist
  // anything; only `is_valid` gates whether Step 8's Save button unlocks.
  dryRun: (payload: DryRunRequest) =>
    api.post<Envelope<DryRunData>>('/metrics/custom/preview', payload).then((r) => ({
      ...r.data.data,
      results: r.data.data.results || [],
    })),

  // Step 8 — persists the metric. Response body shape isn't fully specified
  // by the spec beyond "200 status", so `id`/`name` are optional here —
  // callers should fall back to a generic success message if `id` is absent.
  create: (payload: SaveMetricRequest) =>
    api.post<Envelope<SaveMetricData> | void>('/metrics/custom', payload).then((r) => (r.data as Envelope<SaveMetricData>)?.data || {}),
};














import { useEffect, useMemo, useState } from 'react';
import { useNavigate } from 'react-router-dom';
import {
  AlertCircle, Check, CheckCircle2, Loader2, XCircle,
} from 'lucide-react';
import styles from './CustomMetrics.module.scss';
import { useToast } from './useToast';
import {
  metricsApi, MetricTemplate, BuiltinCheck, ModelSummary, DatasetSummary,
  PreviewQuestion, DryRunData,
} from '../../api/endpoints/metrics';

const STEP_LABELS = [
  'Basic Info', 'Template & Rules', 'Code', 'Judge Model',
  'Dataset', 'Preview Cases', 'Validate', 'Save',
];

type ModelHealthStatus = 'checking' | 'healthy' | 'unhealthy';

export default function CreateMetric() {
  const navigate = useNavigate();
  const { showToast, ToastEl } = useToast();

  const [step, setStep] = useState(1);

  // Step 1
  const [name, setName] = useState('');
  const [description, setDescription] = useState('');

  // Step 2
  const [templates, setTemplates] = useState<MetricTemplate[]>([]);
  const [builtinChecks, setBuiltinChecks] = useState<BuiltinCheck[]>([]);
  const [templatesLoading, setTemplatesLoading] = useState(false);
  const [templatesError, setTemplatesError] = useState('');
  const [selectedTemplateId, setSelectedTemplateId] = useState('');
  const [selectedRuleIds, setSelectedRuleIds] = useState<Set<string>>(new Set());

  const selectedTemplate = useMemo(
    () => templates.find((t) => t.id === selectedTemplateId) || null,
    [templates, selectedTemplateId],
  );
  const evalType = selectedTemplate?.type || '';

  // Step 3
  const [code, setCode] = useState('');
  const [language, setLanguage] = useState('');
  const [codeLoading, setCodeLoading] = useState(false);
  const [codeError, setCodeError] = useState('');

  // Step 4
  const [models, setModels] = useState<ModelSummary[]>([]);
  const [modelsLoading, setModelsLoading] = useState(false);
  const [modelsError, setModelsError] = useState('');
  const [modelHealth, setModelHealth] = useState<Record<string, { status: ModelHealthStatus; latencyMs?: number }>>({});
  const [selectedModelId, setSelectedModelId] = useState('');

  // Step 5
  const [datasets, setDatasets] = useState<DatasetSummary[]>([]);
  const [datasetsLoading, setDatasetsLoading] = useState(false);
  const [datasetsError, setDatasetsError] = useState('');
  const [selectedDatasetId, setSelectedDatasetId] = useState('');

  // Step 6
  const [previewQuestions, setPreviewQuestions] = useState<PreviewQuestion[]>([]);
  const [previewLoading, setPreviewLoading] = useState(false);
  const [previewError, setPreviewError] = useState('');
  const [selectedQuestionIds, setSelectedQuestionIds] = useState<Set<string>>(new Set());

  // Step 7
  const [dryRun, setDryRun] = useState<DryRunData | null>(null);
  const [dryRunLoading, setDryRunLoading] = useState(false);
  const [dryRunError, setDryRunError] = useState('');

  // Step 8
  const [saving, setSaving] = useState(false);
  const [saveError, setSaveError] = useState('');
  const [savedId, setSavedId] = useState('');

  // ---- Step 2: load templates once -------------------------------------
  useEffect(() => {
    setTemplatesLoading(true);
    setTemplatesError('');
    metricsApi.getTemplates()
      .then((res) => {
        setTemplates(res.templates);
        setBuiltinChecks(res.builtin_checks);
      })
      .catch((err) => setTemplatesError(err.message || 'Failed to load templates'))
      .finally(() => setTemplatesLoading(false));
  }, []);

  const toggleRule = (id: string) => {
    setSelectedRuleIds((prev) => {
      const next = new Set(prev);
      next.has(id) ? next.delete(id) : next.add(id);
      return next;
    });
  };

  // ---- Step 3: insert starter code for the selected template's type ------
  const insertStarterCode = () => {
    if (!evalType) {
      showToast('Select a template first', 'error');
      return;
    }
    setCodeLoading(true);
    setCodeError('');
    metricsApi.getCodeTemplate(evalType)
      .then((res) => {
        setCode(res.code_snippet);
        setLanguage(res.language);
      })
      .catch((err) => setCodeError(err.message || 'Failed to load starter code'))
      .finally(() => setCodeLoading(false));
  };

  // auto-insert starter code the first time step 3 is reached, if empty
  useEffect(() => {
    if (step === 3 && !code && evalType) {
      insertStarterCode();
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [step]);

  // ---- Step 4: load models, then ping health for each --------------------
  useEffect(() => {
    if (step !== 4 || models.length > 0) return;
    setModelsLoading(true);
    setModelsError('');
    metricsApi.listModels()
      .then((list) => {
        setModels(list);
        const initial: Record<string, { status: ModelHealthStatus }> = {};
        list.forEach((m) => { initial[m.id] = { status: 'checking' }; });
        setModelHealth(initial);

        list.forEach((m) => {
          metricsApi.checkModelHealth(m.id)
            .then((h) => {
              setModelHealth((prev) => ({
                ...prev,
                [m.id]: { status: h.is_healthy ? 'healthy' : 'unhealthy', latencyMs: h.latency_ms },
              }));
            });
        });
      })
      .catch((err) => setModelsError(err.message || 'Failed to load models'))
      .finally(() => setModelsLoading(false));
  }, [step, models.length]);

  // ---- Step 5: load datasets filtered by eval type ------------------------
  useEffect(() => {
    if (step !== 5 || !evalType) return;
    setDatasetsLoading(true);
    setDatasetsError('');
    setSelectedDatasetId('');
    metricsApi.listDatasets(evalType)
      .then((list) => setDatasets(list))
      .catch((err) => setDatasetsError(err.message || 'Failed to load datasets'))
      .finally(() => setDatasetsLoading(false));
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [step, evalType]);

  // ---- Step 6: preview questions once a dataset is chosen -----------------
  const loadPreview = () => {
    if (!selectedDatasetId) {
      showToast('Select a dataset first', 'error');
      return;
    }
    setPreviewLoading(true);
    setPreviewError('');
    metricsApi.previewDataset(selectedDatasetId)
      .then((res) => {
        const qs = res.questions;
        setPreviewQuestions(qs);
        setSelectedQuestionIds(new Set(qs.map((q) => q.id)));
      })
      .catch((err) => setPreviewError(err.message || 'Failed to load preview'))
      .finally(() => setPreviewLoading(false));
  };

  useEffect(() => {
    if (step === 6 && previewQuestions.length === 0 && selectedDatasetId) loadPreview();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [step]);

  const toggleQuestion = (id: string) => {
    setSelectedQuestionIds((prev) => {
      const next = new Set(prev);
      next.has(id) ? next.delete(id) : next.add(id);
      return next;
    });
  };

  // ---- Step 7: dry run --------------------------------------------------
  const runDryRun = () => {
    if (!code.trim()) { showToast('Add metric code first', 'error'); return; }
    if (!selectedModelId) { showToast('Select a judge model first', 'error'); return; }
    if (selectedQuestionIds.size === 0) { showToast('Select at least one test case', 'error'); return; }

    setDryRunLoading(true);
    setDryRunError('');
    setDryRun(null);
    metricsApi.dryRun({
      metric_config: {
        name: name || 'Untitled Metric',
        code,
        rules: builtinChecks.filter((c) => selectedRuleIds.has(c.id)).map((c) => c.logic),
      },
      judge_config: { model_id: selectedModelId },
      test_case_ids: Array.from(selectedQuestionIds),
    })
      .then((res) => setDryRun(res))
      .catch((err) => setDryRunError(err.message || 'Validation failed'))
      .finally(() => setDryRunLoading(false));
  };

  // ---- Step 8: save --------------------------------------------------------
  const handleSave = () => {
    if (!dryRun?.is_valid) {
      showToast('Run a successful validation before saving', 'error');
      return;
    }
    setSaving(true);
    setSaveError('');
    metricsApi.create({
      metric_config: {
        name,
        description,
        code,
        rules: builtinChecks.filter((c) => selectedRuleIds.has(c.id)).map((c) => c.logic),
        judge_config: { model_id: selectedModelId },
      },
    })
      .then((res) => setSavedId(res.id || 'saved'))
      .catch((err) => setSaveError(err.message || 'Failed to save metric'))
      .finally(() => setSaving(false));
  };

  const resetWizard = () => {
    setStep(1);
    setName(''); setDescription('');
    setSelectedTemplateId(''); setSelectedRuleIds(new Set());
    setCode(''); setLanguage(''); setCodeError('');
    setModels([]); setModelHealth({}); setSelectedModelId('');
    setDatasets([]); setSelectedDatasetId('');
    setPreviewQuestions([]); setSelectedQuestionIds(new Set());
    setDryRun(null); setDryRunError('');
    setSaving(false); setSaveError(''); setSavedId('');
  };

  // ---- step gating -----------------------------------------------------
  const canProceed = (): boolean => {
    switch (step) {
      case 1: return name.trim().length > 0;
      case 2: return !!selectedTemplateId;
      case 3: return code.trim().length > 0;
      case 4: return !!selectedModelId;
      case 5: return !!selectedDatasetId;
      case 6: return selectedQuestionIds.size > 0;
      case 7: return !!dryRun?.is_valid;
      default: return true;
    }
  };

  const goNext = () => {
    if (step === 5 && previewQuestions.length === 0) loadPreview();
    setStep((s) => Math.min(8, s + 1));
  };
  const goBack = () => setStep((s) => Math.max(1, s - 1));

  const stepClass = (n: number) => `${styles.step} ${step === n ? styles.active : ''} ${step > n ? styles.done : ''}`;

  return (
    <div className={`page-enter pg-shell ${styles.cm}`}>
      <div className={styles['cm__header']}>
        <div>
          <p className={styles['cm__header-eyebrow']}>Custom Metrics</p>
          <h1>Create Custom Metric</h1>
          <p className={styles['cm__header-sub']}>Define, configure, and validate a custom LLM-as-judge metric</p>
        </div>
      </div>

      <div className={`pg-body ${styles['pg-body-scroll']}`}>
        <div className={styles.steps}>
          {STEP_LABELS.map((label, i) => (
            <div key={label} style={{ display: 'inline-flex', alignItems: 'center', gap: '10px' }}>
              <div className={stepClass(i + 1)}>
                <span className={styles['step-num']}>{step > i + 1 ? <Check size={12} /> : i + 1}</span>
                <span className={styles['step-label']}>{label}</span>
              </div>
              {i < STEP_LABELS.length - 1 && <div className={styles['step-line']} />}
            </div>
          ))}
        </div>

        <div className={styles.panel}>
          {/* ---- Step 1: Basic Info ---- */}
          {step === 1 && (
            <>
              <div className={styles['form-group']}>
                <label>Metric Name *</label>
                <input
                  className={styles.input}
                  placeholder="e.g., Answer Faithfulness"
                  value={name}
                  onChange={(e) => setName(e.target.value)}
                />
              </div>
              <div className={styles['form-group']}>
                <label>Description</label>
                <textarea
                  className={styles.textarea}
                  placeholder="What does this metric measure? (optional)"
                  value={description}
                  onChange={(e) => setDescription(e.target.value)}
                />
              </div>
            </>
          )}

          {/* ---- Step 2: Template & Rules ---- */}
          {step === 2 && (
            <>
              <h3 className={styles['step-heading']}>Select Template &amp; Rules</h3>
              <p className={styles['step-sub']}>Choose a starting point and any built-in checks to combine with it</p>

              {templatesError && <div className={styles['error-banner']}><AlertCircle size={14} /> {templatesError}</div>}

              {templatesLoading ? (
                <div className={styles['loading-row']}><Loader2 size={14} className={styles.spin} /> Loading templates…</div>
              ) : (
                <>
                  <div className={styles['form-group']}>
                    <label>Template</label>
                    <select
                      className={styles.select}
                      value={selectedTemplateId}
                      onChange={(e) => { setSelectedTemplateId(e.target.value); setCode(''); }}
                    >
                      <option value="">Select a template…</option>
                      {templates.map((t) => (
                        <option key={t.id} value={t.id}>{t.name} ({t.type.toUpperCase()})</option>
                      ))}
                    </select>
                  </div>

                  <div className={styles['form-group']}>
                    <label>Built-in Checks <span className={styles.optional}>(optional)</span></label>
                    {builtinChecks.length === 0 ? (
                      <div className={styles.empty}>No built-in checks available.</div>
                    ) : (
                      <div className={styles['checkbox-list']}>
                        {builtinChecks.map((c) => (
                          <label key={c.id} className={styles['checkbox-item']}>
                            <input
                              type="checkbox"
                              checked={selectedRuleIds.has(c.id)}
                              onChange={() => toggleRule(c.id)}
                            />
                            <span className={styles['checkbox-item__name']}>{c.name}</span>
                            <span className={styles['checkbox-item__logic']}>{c.logic}</span>
                          </label>
                        ))}
                      </div>
                    )}
                  </div>
                </>
              )}
            </>
          )}

          {/* ---- Step 3: Code Configuration ---- */}
          {step === 3 && (
            <>
              <h3 className={styles['step-heading']}>Code Configuration</h3>
              <p className={styles['step-sub']}>Starter code is based on the template type — edit as needed</p>

              {codeError && <div className={styles['error-banner']}><AlertCircle size={14} /> {codeError}</div>}

              <div className={styles['code-editor']}>
                <div className={styles['editor-header']}>
                  <div className={styles['editor-header-left']}>
                    <span>Metric Code</span>
                    {language && <span className={styles['lang-badge']}>{language}</span>}
                  </div>
                  <button
                    type="button"
                    className={`${styles.btn} ${styles['btn-sm']}`}
                    onClick={insertStarterCode}
                    disabled={codeLoading || !evalType}
                  >
                    {codeLoading ? <Loader2 size={12} className={styles.spin} /> : null}
                    Insert Starter Code
                  </button>
                </div>
                <textarea
                  className={styles['code-area']}
                  spellCheck={false}
                  value={code}
                  onChange={(e) => setCode(e.target.value)}
                  placeholder="# Your metric code will appear here"
                />
              </div>
            </>
          )}

          {/* ---- Step 4: Judge Model ---- */}
          {step === 4 && (
            <>
              <h3 className={styles['step-heading']}>Select Judge Model</h3>
              <p className={styles['step-sub']}>Choose which model will score responses for this metric</p>

              {modelsError && <div className={styles['error-banner']}><AlertCircle size={14} /> {modelsError}</div>}

              {modelsLoading ? (
                <div className={styles['loading-row']}><Loader2 size={14} className={styles.spin} /> Loading models…</div>
              ) : models.length === 0 ? (
                <div className={styles.empty}>No models available.</div>
              ) : (
                <div className={styles['model-list']}>
                  {models.map((m) => {
                    const health = modelHealth[m.id];
                    const disabled = health?.status === 'unhealthy';
                    const selected = selectedModelId === m.id;
                    return (
                      <label
                        key={m.id}
                        className={`${styles['model-item']} ${selected ? styles['model-item--selected'] : ''} ${disabled ? styles['model-item--disabled'] : ''}`}
                      >
                        <input
                          type="radio"
                          name="judge-model"
                          checked={selected}
                          disabled={disabled}
                          onChange={() => setSelectedModelId(m.id)}
                        />
                        <span className={styles['model-item__name']}>{m.name}</span>
                        <span className={styles['model-item__provider']}>{m.provider}</span>
                        <span className={`${styles['model-item__status']} ${styles[`status-${health?.status || 'checking'}`]}`}>
                          <span className={`${styles['health-dot']} ${styles[`health-dot--${health?.status || 'checking'}`]}`} />
                          {health?.status === 'checking' && 'Checking…'}
                          {health?.status === 'healthy' && 'Healthy'}
                          {health?.status === 'unhealthy' && 'Offline'}
                        </span>
                      </label>
                    );
                  })}
                </div>
              )}
            </>
          )}

          {/* ---- Step 5: Dataset Selection ---- */}
          {step === 5 && (
            <>
              <h3 className={styles['step-heading']}>Dataset Selection</h3>
              <p className={styles['step-sub']}>Choose the question bank to test this metric against ({evalType.toUpperCase() || 'eval type'})</p>

              {datasetsError && <div className={styles['error-banner']}><AlertCircle size={14} /> {datasetsError}</div>}

              {datasetsLoading ? (
                <div className={styles['loading-row']}><Loader2 size={14} className={styles.spin} /> Loading datasets…</div>
              ) : datasets.length === 0 ? (
                <div className={styles.empty}>No datasets found for this evaluation type.</div>
              ) : (
                <div className={styles['form-group']}>
                  <label>Dataset</label>
                  <select
                    className={styles.select}
                    value={selectedDatasetId}
                    onChange={(e) => { setSelectedDatasetId(e.target.value); setPreviewQuestions([]); }}
                  >
                    <option value="">Select a dataset…</option>
                    {datasets.map((d) => (
                      <option key={d.id} value={d.id}>{d.name} ({d.question_count} questions)</option>
                    ))}
                  </select>
                </div>
              )}

              <button
                type="button"
                className={`${styles.btn} ${styles['btn-sm']}`}
                onClick={loadPreview}
                disabled={!selectedDatasetId || previewLoading}
              >
                {previewLoading ? <Loader2 size={12} className={styles.spin} /> : null}
                Preview Questions
              </button>
            </>
          )}

          {/* ---- Step 6: Preview Test Cases ---- */}
          {step === 6 && (
            <>
              <h3 className={styles['step-heading']}>Preview Test Cases</h3>
              <p className={styles['step-sub']}>Pick which questions to include in the dry run ({selectedQuestionIds.size} selected)</p>

              {previewError && <div className={styles['error-banner']}><AlertCircle size={14} /> {previewError}</div>}

              {previewLoading ? (
                <div className={styles['loading-row']}><Loader2 size={14} className={styles.spin} /> Loading test cases…</div>
              ) : previewQuestions.length === 0 ? (
                <div className={styles.empty}>No preview loaded yet — go back and click "Preview Questions".</div>
              ) : (
                <div className={styles['preview-list']}>
                  {previewQuestions.map((q) => (
                    <label key={q.id} className={styles['preview-item']}>
                      <input
                        type="checkbox"
                        checked={selectedQuestionIds.has(q.id)}
                        onChange={() => toggleQuestion(q.id)}
                      />
                      <div className={styles['preview-item__body']}>
                        <div className={styles['preview-item__q']}>{q.input}</div>
                        <div className={styles['preview-item__a']}>
                          <span className={styles['preview-item__a-label']}>Expected:</span>{q.expected_output}
                        </div>
                      </div>
                    </label>
                  ))}
                </div>
              )}
            </>
          )}

          {/* ---- Step 7: Validate (Dry Run) ---- */}
          {step === 7 && (
            <>
              <h3 className={styles['step-heading']}>Validate Metric</h3>
              <p className={styles['step-sub']}>Test your logic against the selected questions without saving</p>

              {dryRunError && <div className={styles['error-banner']}><AlertCircle size={14} /> {dryRunError}</div>}

              <button type="button" className={`${styles.btn} ${styles['btn-primary']}`} onClick={runDryRun} disabled={dryRunLoading}>
                {dryRunLoading ? <Loader2 size={13} className={styles.spin} /> : null}
                Validate / Dry Run
              </button>

              {dryRun && (
                <div style={{ marginTop: '18px' }}>
                  {dryRun.is_valid ? (
                    <div className={styles['success-banner']}><CheckCircle2 size={14} /> Metric is valid! Ready to save.</div>
                  ) : (
                    <div className={styles['error-banner']}><XCircle size={14} /> Metric failed validation — review the results below.</div>
                  )}

                  <div className={styles['validation-results']}>
                    <table className={styles.table}>
                      <thead>
                        <tr>
                          <th>Question ID</th>
                          <th>Score</th>
                          <th>Reason</th>
                          <th>Status</th>
                        </tr>
                      </thead>
                      <tbody>
                        {dryRun.results.map((r) => (
                          <tr key={r.question_id}>
                            <td className={styles['cell-num']}>{r.question_id}</td>
                            <td className={r.passed ? styles['cell-pass'] : styles['cell-fail']}>{r.score.toFixed(2)}</td>
                            <td>{r.reason}</td>
                            <td className={r.passed ? styles['cell-pass'] : styles['cell-fail']}>{r.passed ? 'Pass' : 'Fail'}</td>
                          </tr>
                        ))}
                      </tbody>
                    </table>
                    <div className={styles['validation-summary']}>
                      <span>{dryRun.summary.passed}/{dryRun.summary.total_run} passed</span>
                      <span>Avg. Score: <strong>{dryRun.summary.average_score.toFixed(2)}</strong></span>
                    </div>
                  </div>
                </div>
              )}
            </>
          )}

          {/* ---- Step 8: Save ---- */}
          {step === 8 && (
            <>
              <h3 className={styles['step-heading']}>Create Metric</h3>
              <p className={styles['step-sub']}>Review and save — this metric will be available for future evaluations</p>

              {saveError && <div className={styles['error-banner']}><AlertCircle size={14} /> {saveError}</div>}

              <div className={styles['transform-summary']}>
                <div className={styles['summary-item']}>
                  <span className={styles['summary-label']}>Name</span>
                  <span className={styles['summary-value']}>{name || '—'}</span>
                </div>
                <div className={styles['summary-item']}>
                  <span className={styles['summary-label']}>Eval Type</span>
                  <span className={styles['summary-value']}>{evalType.toUpperCase() || '—'}</span>
                </div>
                <div className={styles['summary-item']}>
                  <span className={styles['summary-label']}>Rules</span>
                  <span className={styles['summary-value']}>{selectedRuleIds.size}</span>
                </div>
                <div className={styles['summary-item']}>
                  <span className={styles['summary-label']}>Last Validation</span>
                  <span className={`${styles['summary-value']} ${dryRun?.is_valid ? styles['summary-value--ok'] : styles['summary-value--danger']}`}>
                    {dryRun?.is_valid ? 'Passed' : 'Not validated'}
                  </span>
                </div>
              </div>

              <button
                type="button"
                className={`${styles.btn} ${styles['btn-primary']}`}
                onClick={handleSave}
                disabled={saving || !dryRun?.is_valid}
              >
                {saving ? <Loader2 size={13} className={styles.spin} /> : null}
                Create Metric
              </button>
            </>
          )}

          {/* ---- nav ---- */}
          <div className={styles['form-actions']}>
            {step > 1 && (
              <button type="button" className={`${styles.btn} ${styles['btn-secondary']}`} onClick={goBack}>
                Back
              </button>
            )}
            {step === 1 && (
              <button
                type="button"
                className={`${styles.btn} ${styles['btn-secondary']}`}
                onClick={() => navigate('/app/custom-metrics/dashboard')}
              >
                Cancel
              </button>
            )}
            {step < 8 && (
              <button
                type="button"
                className={`${styles.btn} ${styles['btn-primary']}`}
                onClick={goNext}
                disabled={!canProceed()}
              >
                Next
              </button>
            )}
          </div>
        </div>
      </div>

      {savedId && (
        <div className={styles['modal-overlay']}>
          <div className={styles.modal}>
            <div className={styles['modal-icon']}><CheckCircle2 size={22} /></div>
            <h3>Metric created successfully!</h3>
            <p>Your metric is now available for evaluations.</p>
            <div className={styles['modal-id']}>ID: {savedId}</div>
            <div className={styles['modal-actions']}>
              <button type="button" className={`${styles.btn} ${styles['btn-secondary']}`} onClick={resetWizard}>
                Create Another
              </button>
              <button
                type="button"
                className={`${styles.btn} ${styles['btn-primary']}`}
                onClick={() => navigate('/app/custom-metrics/dashboard')}
              >
                Go to Dashboard
              </button>
            </div>
          </div>
        </div>
      )}

      {ToastEl}
    </div>
  );
}


