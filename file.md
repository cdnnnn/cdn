//Createmetric.tsx
import { useEffect, useMemo, useState } from 'react';
import { useNavigate } from 'react-router-dom';
import {
  AlertCircle, Check, CheckCircle2, Loader2, XCircle,
} from 'lucide-react';
import styles from './CustomMetrics.module.scss';
import { useToast } from './useToast';
import {
  fetchTemplates, fetchCodeTemplate, fetchModels, fetchModelHealth,
  fetchDatasets, fetchDatasetPreview, dryRunMetric, saveMetric,
  MetricTemplate, BuiltinCheck, ModelSummary, DatasetSummary,
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
    fetchTemplates()
      .then((res) => {
        setTemplates(res.data.templates || []);
        setBuiltinChecks(res.data.builtin_checks || []);
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
    fetchCodeTemplate(evalType)
      .then((res) => {
        setCode(res.data.code_snippet);
        setLanguage(res.data.language);
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
    fetchModels()
      .then((res) => {
        const list = res.data.models || [];
        setModels(list);
        const initial: Record<string, { status: ModelHealthStatus }> = {};
        list.forEach((m) => { initial[m.id] = { status: 'checking' }; });
        setModelHealth(initial);

        list.forEach((m) => {
          fetchModelHealth(m.id)
            .then((h) => {
              if (h.status === 'success' && 'data' in h) {
                setModelHealth((prev) => ({
                  ...prev,
                  [m.id]: { status: h.data.is_healthy ? 'healthy' : 'unhealthy', latencyMs: h.data.latency_ms },
                }));
              } else {
                setModelHealth((prev) => ({ ...prev, [m.id]: { status: 'unhealthy' } }));
              }
            })
            .catch(() => setModelHealth((prev) => ({ ...prev, [m.id]: { status: 'unhealthy' } })));
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
    fetchDatasets(evalType)
      .then((res) => setDatasets(res.data.datasets || []))
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
    fetchDatasetPreview(selectedDatasetId)
      .then((res) => {
        const qs = res.data.questions || [];
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
    dryRunMetric({
      metric_config: {
        name: name || 'Untitled Metric',
        code,
        rules: builtinChecks.filter((c) => selectedRuleIds.has(c.id)).map((c) => c.logic),
      },
      judge_config: { model_id: selectedModelId },
      test_case_ids: Array.from(selectedQuestionIds),
    })
      .then((res) => setDryRun(res.data))
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
    saveMetric({
      metric_config: {
        name,
        description,
        code,
        rules: builtinChecks.filter((c) => selectedRuleIds.has(c.id)).map((c) => c.logic),
        judge_config: { model_id: selectedModelId },
      },
    })
      .then((res) => setSavedId(res.data?.id || 'saved'))
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


















//Metrics.ts
// API layer for the Custom Metrics feature (Create Metric wizard).
// Endpoints and payload shapes follow the spec exactly.

const API_BASE = import.meta.env.VITE_API_BASE_URL || '';

async function request<T>(path: string, options?: RequestInit): Promise<T> {
  const res = await fetch(`${API_BASE}${path}`, {
    headers: { 'Content-Type': 'application/json', ...(options?.headers || {}) },
    ...options,
  });

  if (!res.ok) {
    let message = `Request failed (${res.status})`;
    try {
      const body = await res.json();
      message = body?.message || message;
    } catch {
      // response wasn't JSON — keep the generic message
    }
    throw new Error(message);
  }

  return res.json() as Promise<T>;
}

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

export function fetchTemplates() {
  return request<{ status: string; data: TemplatesData }>('/metrics/templates');
}

// ---- Step 3: starter code for an eval type -------------------------------
export interface CodeTemplateData {
  eval_type: string;
  language: string;
  code_snippet: string;
}

export function fetchCodeTemplate(evalType: string) {
  return request<{ status: string; data: CodeTemplateData }>(
    `/metrics/code-templates/${encodeURIComponent(evalType)}`,
  );
}

// ---- Step 4: judge models + health ---------------------------------------
export interface ModelSummary {
  id: string;
  name: string;
  provider: string;
}

export function fetchModels() {
  return request<{ status: string; data: { models: ModelSummary[] } }>('/models');
}

export interface ModelHealthData {
  is_healthy: boolean;
  latency_ms?: number;
}

export function fetchModelHealth(modelId: string) {
  return request<{ status: string; data: ModelHealthData } | { status: string; message: string }>(
    `/models/health/${encodeURIComponent(modelId)}`,
  );
}

// ---- Step 5/6: datasets + preview ------------------------------------------
export interface DatasetSummary {
  id: string;
  name: string;
  question_count: number;
}

export function fetchDatasets(evalType: string) {
  return request<{ status: string; data: { total_count: number; datasets: DatasetSummary[] } }>(
    `/datasets?eval_type=${encodeURIComponent(evalType)}`,
  );
}

export interface PreviewQuestion {
  id: string;
  input: string;
  expected_output: string;
}

export function fetchDatasetPreview(datasetId: string) {
  return request<{
    status: string;
    data: { dataset_id: string; total_questions: number; preview_limit: number; questions: PreviewQuestion[] };
  }>(`/datasets/${encodeURIComponent(datasetId)}/preview`);
}

// ---- Step 7: dry run -------------------------------------------------------
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

export function dryRunMetric(payload: DryRunRequest) {
  return request<{ status: string; data: DryRunData }>('/metrics/custom/preview', {
    method: 'POST',
    body: JSON.stringify(payload),
  });
}

// ---- Step 8: save ----------------------------------------------------------
export interface SaveMetricRequest {
  metric_config: MetricConfig & { judge_config: JudgeConfig };
}

export interface SaveMetricData {
  id?: string;
  name?: string;
}

export function saveMetric(payload: SaveMetricRequest) {
  return request<{ status: string; data?: SaveMetricData }>('/metrics/custom', {
    method: 'POST',
    body: JSON.stringify(payload),
  });
}














//Custommetrics.module.scss
@use '../../styles/_variables' as *;

// ===========================================================================
// Custom Metrics — Dashboard / Create Metric / Upload Dataset.
// Mirrors the History/Reports/Comparison/Sidebar design system: ink/paper
// palette, ultramarine signal accent, mono instrument labels, hover-lift.
// Shared by all three sub-pages via CSS module composition.
// ===========================================================================

$ink:      var(--ink-1);
$ink-2:    var(--ink-2);
$ink-3:    var(--ink-3);
$paper:    var(--paper);
$card:     var(--card);
$line:     var(--line);
$line-2:   var(--line-2);
$signal:   #2B2BF5;
$signal-2: #1C1CC7;
$wash:     var(--signal-wash);
$ok:       #0FA968;
$ok-wash:  var(--ok-wash);
$amber:    #E08600;
$amber-wash: var(--amber-wash);
$danger:   #DC2626;
$danger-wash: var(--danger-wash);
$violet:   #6D28D9;
$violet-wash: rgba(109, 40, 217, 0.1);
$sky:      #0369A1;
$sky-wash: var(--sky-wash);
$ink-wash: var(--ink-wash);

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

$soft: 0 1px 2px rgba(20, 22, 27, 0.05);
$lift: 0 14px 30px -14px rgba(20, 22, 27, 0.22);

// Font scaling: matches the Sidebar's pattern — `.cm` sets a single base
// font-size, and every descendant font-size below is expressed in `em`
// (relative to that base), so bumping `.cm`'s font-size on wide screens
// scales the whole feature proportionally from one place.
$cm-base-font: 0.875rem;

%micro {
  font-family: $mono;
  font-size: 0.7857em;
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

@keyframes cm-spin { to { transform: rotate(360deg); } }
@keyframes cm-toast-in {
  from { opacity: 0; transform: translate(-50%, 8px); }
  to   { opacity: 1; transform: translate(-50%, 0); }
}

// ---- shared page header -----------------------------------------------
.cm {
  // master scale control — every em-based font-size in this module
  // responds to this (mirrors Sidebar.module.scss)
  font-size: $cm-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding: 24px 32px 20px;
    border-bottom: 1px solid $line;
    background: $card;

    h1 {
      font-family: $display;
      font-size: 1.7143em;
      font-weight: 800;
      letter-spacing: -0.02em;
      color: $ink;
      line-height: 1.2;
    }
  }

  &__header-eyebrow {
    @extend %micro;
    display: flex;
    align-items: center;
    gap: 8px;
    color: $signal;
    margin-bottom: 6px;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $signal;
    }
  }

  &__header-sub {
    margin-top: 4px;
    font-size: 0.9643em;
    color: $ink-2;
  }
}

.pg-body-scroll {
  overflow-y: auto;
  padding: 20px 32px 32px;
}

// ---- buttons -------------------------------------------------------------
.btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 9px 16px;
  border-radius: 10px;
  border: 1px solid $line;
  background: $card;
  color: $ink-2;
  font-family: $sans;
  font-size: 0.9286em;
  font-weight: 650;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease, box-shadow 0.15s ease, transform 0.15s ease;

  &:hover:not(:disabled) { border-color: $ink-3; color: $ink; box-shadow: $soft; }
  &:disabled { opacity: 0.45; cursor: not-allowed; }
}

.btn-sm { padding: 6px 11px; font-size: 0.8571em; border-radius: 8px; }

.btn-primary {
  border-color: $signal;
  background: $signal;
  color: #fff;

  &:hover:not(:disabled) { background: $signal-2; border-color: $signal-2; color: #fff; transform: translateY(-1px); box-shadow: $lift; }
}

.btn-secondary {
  background: $paper;
}

.btn-ai {
  border-color: rgba($violet, 0.3);
  background: $violet-wash;
  color: $violet;

  &:hover:not(:disabled) { border-color: $violet; background: rgba($violet, 0.16); color: $violet; }
}

.btn-icon {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 26px;
  height: 26px;
  border-radius: 7px;
  border: 1px solid transparent;
  background: transparent;
  color: $ink-3;
  cursor: pointer;
  transition: background 0.15s ease, color 0.15s ease, border-color 0.15s ease;

  &:hover { background: $danger-wash; border-color: rgba($danger, 0.2); color: $danger; }
}

.spin { animation: cm-spin 0.8s linear infinite; }

// ---- toggle groups ---------------------------------------------------------
.btn-group {
  display: inline-flex;
  padding: 3px;
  background: $paper;
  border: 1px solid $line;
  border-radius: 11px;
  gap: 2px;
}

.btn-toggle {
  padding: 7px 16px;
  border-radius: 8px;
  border: none;
  background: transparent;
  color: $ink-2;
  font-family: $sans;
  font-size: 0.9286em;
  font-weight: 650;
  cursor: pointer;
  transition: background 0.15s ease, color 0.15s ease, box-shadow 0.15s ease;

  &:hover { color: $ink; }

  &.active {
    background: $card;
    color: $signal;
    box-shadow: $soft;
    font-weight: 700;
  }
}

.toggle-container {
  display: inline-flex;
  border: 1px solid $line;
  border-radius: 11px;
  overflow: hidden;
}

.toggle-btn {
  padding: 9px 18px;
  border: none;
  background: $paper;
  color: $ink-2;
  font-family: $sans;
  font-size: 0.9286em;
  font-weight: 650;
  cursor: pointer;
  transition: background 0.15s ease, color 0.15s ease;

  &:hover { color: $ink; }

  &.active {
    background: $signal;
    color: #fff;
  }

  &:not(:last-child) { border-right: 1px solid $line; }
}

// ---- cards (dashboard) ------------------------------------------------------
.cards-row {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 16px;
}

.card {
  background: $card;
  border: 1px solid $line;
  border-radius: 16px;
  box-shadow: $soft;
  overflow: hidden;
}

.card-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 18px;
  border-bottom: 1px solid $line;

  h3 {
    font-family: $display;
    font-size: 1.0714em;
    font-weight: 700;
    color: $ink;
  }
}

.card-body { padding: 4px 0 8px; }

// ---- generic table -----------------------------------------------------
.table-wrap {
  overflow-x: auto;
}

.table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.9286em;

  thead th {
    text-align: left;
    background: $paper;
    @extend %micro;
    font-size: 0.6429em;
    color: $ink-3;
    padding: 10px 18px;
    white-space: nowrap;
  }

  tbody tr {
    border-top: 1px solid $line-2;
    transition: background 0.13s ease;
    &:hover { background: $paper; }
  }

  tbody td {
    padding: 11px 18px;
    color: $ink;
    vertical-align: middle;
  }
}

.badge {
  display: inline-flex;
  align-items: center;
  font-family: $mono;
  font-size: 0.7143em;
  font-weight: 700;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  border-radius: 6px;
  padding: 3px 8px;
  white-space: nowrap;
  color: $signal;
  background: $wash;

  &--code  { color: $violet; background: $violet-wash; }
  &--model { color: $signal; background: $wash; }
  &--rag   { color: $sky; background: $sky-wash; }
  &--agent { color: $amber; background: $amber-wash; }
}

// ---- forms ---------------------------------------------------------------
.form-group {
  margin-bottom: 20px;

  label {
    display: block;
    @extend %micro;
    font-size: 0.7857em;
    color: $ink-2;
    margin-bottom: 8px;
  }
}

.input,
.select {
  width: 100%;
  border: 1.5px solid $line;
  border-radius: 9px;
  padding: 9px 12px;
  font-size: 0.9286em;
  font-family: $sans;
  color: $ink;
  background: $card;

  &::placeholder { color: $ink-3; }
  &:focus { outline: none; border-color: $signal; box-shadow: 0 0 0 3px $wash; }
}

.select { cursor: pointer; }

.hint {
  font-size: 0.8929em;
  color: $ink-3;
  margin: -4px 0 16px;
}

.panel {
  background: $card;
  border: 1px solid $line;
  border-radius: 16px;
  box-shadow: $soft;
  padding: 24px;
}

.panel + .panel { margin-top: 16px; }

// ---- metric template cards --------------------------------------------
.metric-templates {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 12px;
  margin-bottom: 24px;
}

.metric-card {
  position: relative;
  border: 1.5px solid $line;
  border-radius: 14px;
  padding: 14px;
  cursor: pointer;
  background: $paper;
  transition: border-color 0.15s ease, box-shadow 0.15s ease, background 0.15s ease;

  &:hover { border-color: $ink-3; }

  &.selected {
    border-color: $signal;
    background: $wash;
    box-shadow: 0 0 0 1px $signal inset;
  }
}

.metric-card-header {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 6px;
  font-family: $display;
  font-weight: 700;
  font-size: 0.9643em;
  color: $ink;

  input[type='radio'] { accent-color: $signal; }
}

.metric-card p {
  font-size: 0.8571em;
  color: $ink-2;
  line-height: 1.45;
  margin-bottom: 8px;
}

.metric-card code {
  display: block;
  font-family: $mono;
  font-size: 0.7857em;
  color: $signal;
  background: $card;
  border: 1px solid $line;
  border-radius: 7px;
  padding: 6px 8px;
  overflow-x: auto;
  white-space: nowrap;
}

// ---- custom rule builder ---------------------------------------------
.custom-rules-section {
  border-top: 1px solid $line;
  padding-top: 20px;
  margin-bottom: 20px;

  h4 {
    font-family: $display;
    font-size: 1em;
    font-weight: 700;
    color: $ink;
    margin-bottom: 12px;
  }
}

.optional {
  @extend %micro;
  font-size: 0.7143em;
  color: $ink-3;
  margin-left: 6px;
}

.rule-item {
  margin-bottom: 8px;
}

.rule-fields {
  display: flex;
  align-items: center;
  gap: 8px;

  select, input {
    flex-shrink: 0;
  }
}

.rule-field-select { width: 150px; }
.rule-operator { width: 140px; }
.rule-compare-type { width: 140px; }
.rule-value { flex: 1; min-width: 0; }

.add-rule-row {
  display: flex;
  align-items: center;
  gap: 8px;
  margin: 10px 0 16px;
}

.gate-select { width: 90px; font-weight: 700; color: $signal; }

.rule-preview {
  label {
    display: block;
    @extend %micro;
    font-size: 0.7143em;
    color: $ink-3;
    margin-bottom: 6px;
  }

  code {
    display: block;
    font-family: $mono;
    font-size: 0.8571em;
    color: $ink;
    background: $paper;
    border: 1px solid $line;
    border-radius: 9px;
    padding: 10px 12px;
  }
}

.threshold-row {
  display: flex;
  gap: 24px;
  margin-top: 16px;

  .rule-field {
    label {
      display: block;
      @extend %micro;
      font-size: 0.7143em;
      color: $ink-2;
      margin-bottom: 6px;
    }

    input {
      width: 100px;
    }
  }
}

// ---- code editor -----------------------------------------------------
.code-editor {
  border: 1px solid $line;
  border-radius: 14px;
  overflow: hidden;
  margin-bottom: 24px;
}

.editor-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 14px;
  background: $paper;
  border-bottom: 1px solid $line;
  @extend %micro;
  font-size: 0.7857em;
  color: $ink-2;
}

.code-area {
  width: 100%;
  min-height: 320px;
  border: none;
  resize: vertical;
  padding: 16px;
  font-family: $mono;
  font-size: 0.8929em;
  line-height: 1.6;
  color: $ink;
  background: $card;

  &:focus { outline: none; }
}

// ---- validation ---------------------------------------------------------
.validation-section {
  border-top: 1px solid $line;
  padding-top: 20px;
  margin-bottom: 24px;
}

.validation-header {
  margin-bottom: 12px;

  h3 {
    font-family: $display;
    font-size: 1.0714em;
    font-weight: 700;
    color: $ink;
  }

  p {
    font-size: 0.8929em;
    color: $ink-3;
    margin-top: 2px;
  }
}

.validation-controls {
  display: flex;
  gap: 10px;
  margin-bottom: 16px;

  select { max-width: 280px; }
}

.validation-results {
  border: 1px solid $line;
  border-radius: 14px;
  overflow: hidden;
}

.validation-summary {
  display: flex;
  gap: 24px;
  padding: 12px 18px;
  background: $paper;
  border-top: 1px solid $line;
  font-size: 0.9286em;
  color: $ink-2;

  strong { color: $ink; font-family: $mono; }
}

.cell-pass { font-family: $mono; font-weight: 700; color: $ok; }
.cell-fail { font-family: $mono; font-weight: 700; color: $danger; }
.cell-num { font-family: $mono; font-weight: 700; color: $ink; }

// ---- form actions ---------------------------------------------------------
.form-actions {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  padding-top: 8px;
}

// ---- upload wizard: steps -------------------------------------------------
.steps {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 28px;
}

.step {
  display: flex;
  align-items: center;
  gap: 8px;
  opacity: 0.55;
  transition: opacity 0.15s ease;

  &.active, &.done { opacity: 1; }
}

.step-num {
  width: 26px;
  height: 26px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-family: $mono;
  font-size: 0.8571em;
  font-weight: 700;
  color: $ink-3;
  background: $paper;
  border: 1.5px solid $line;
}

.step.active .step-num {
  color: #fff;
  background: $signal;
  border-color: $signal;
}

.step.done .step-num {
  color: $signal;
  background: $wash;
  border-color: $signal;
}

.step-label {
  font-size: 0.9286em;
  font-weight: 650;
  color: $ink-2;
}

.step.active .step-label { color: $ink; font-weight: 700; }

.step-line {
  width: 32px;
  height: 1.5px;
  background: $line;
}

// ---- dropzone --------------------------------------------------------
.dropzone {
  border: 1.5px dashed $line;
  border-radius: 16px;
  padding: 40px 20px;
  text-align: center;
  cursor: pointer;
  background: $paper;
  transition: border-color 0.15s ease, background 0.15s ease;
  margin-bottom: 16px;

  &:hover, &.drag { border-color: $signal; background: $wash; }
}

.dropzone-icon {
  width: 44px;
  height: 44px;
  margin: 0 auto 12px;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: $card;
  border: 1px solid $line;
  color: $signal;
}

.dropzone p { font-size: 1em; color: $ink; font-weight: 650; margin-bottom: 2px; }
.dropzone-hint { font-size: 0.8929em; color: $ink-3 !important; font-weight: 500 !important; }
.dropzone-formats { font-family: $mono; font-size: 0.7857em !important; color: $ink-3 !important; margin-top: 8px !important; font-weight: 500 !important; }

.file-info {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 14px;
  border: 1px solid $line;
  border-radius: 12px;
  background: $card;
  margin-bottom: 16px;

  .file-name { font-weight: 700; color: $ink; font-size: 0.9286em; }
  .file-size { font-family: $mono; font-size: 0.8214em; color: $ink-3; }
}

// ---- detected columns / chips ------------------------------------------
.detected-columns {
  margin-top: 20px;

  h4 {
    @extend %micro;
    font-size: 0.7857em;
    color: $ink-2;
    margin-bottom: 10px;
  }
}

.column-chips {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.chip {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  font-family: $mono;
  font-size: 0.7857em;
  font-weight: 700;
  color: $signal;
  background: $wash;
  border: 1px solid rgba($signal, 0.18);
  border-radius: 999px;
  padding: 4px 10px;

  span { color: $ink-3; font-weight: 500; text-transform: none; letter-spacing: 0; }
}

// ---- field mapping --------------------------------------------------------
.mapping-row {
  display: flex;
  align-items: center;
  gap: 14px;
  padding: 12px 0;
  border-bottom: 1px solid $line-2;

  &:last-child { border-bottom: none; }
}

.mapping-target {
  flex: 0 0 220px;
  font-family: $display;
  font-weight: 700;
  font-size: 0.9643em;
  color: $ink;
}

.required { color: $danger; margin-right: 3px; }

.mapping-hint {
  display: block;
  font-family: $sans;
  font-weight: 500;
  font-size: 0.8214em;
  color: $ink-3;
  margin-top: 2px;
}

.mapping-arrow { color: $ink-3; flex-shrink: 0; }
.mapping-source { flex: 1; }

.ai-assist {
  display: flex;
  align-items: center;
  gap: 10px;
  margin: 20px 0;
}

.ai-hint { font-size: 0.8571em; color: $ink-3; }

// ---- json preview / summary --------------------------------------------
.json-preview {
  background: $paper;
  border: 1px solid $line;
  border-radius: 14px;
  padding: 16px;
  margin-bottom: 20px;
  max-height: 340px;
  overflow: auto;

  pre {
    font-family: $mono;
    font-size: 0.8571em;
    line-height: 1.6;
    color: $ink;
    white-space: pre-wrap;
    word-break: break-word;
  }
}

.transform-summary {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 12px;
  margin-bottom: 24px;
}

.summary-item {
  display: flex;
  flex-direction: column;
  gap: 4px;
  padding: 14px 16px;
  background: $paper;
  border: 1px solid $line;
  border-radius: 14px;

  .summary-label { @extend %micro; font-size: 0.6429em; color: $ink-3; }
  .summary-value { font-family: $mono; font-size: 1.2143em; font-weight: 700; color: $ink; }
  .summary-value--danger { color: $danger; }
  .summary-value--ok { color: $ok; }
}

.empty {
  padding: 24px;
  text-align: center;
  color: $ink-3;
  font-size: 0.9286em;
}

// ---- toast --------------------------------------------------------------
.toast {
  position: fixed;
  left: 50%;
  bottom: 28px;
  transform: translateX(-50%);
  z-index: 200;
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 11px 18px;
  border-radius: 11px;
  background: #14161B;
  color: #fff;
  font-size: 0.9286em;
  font-weight: 650;
  box-shadow: $lift;
  animation: cm-toast-in 0.18s ease;

  &--ok::before { content: ''; width: 6px; height: 6px; border-radius: 50%; background: $ok; flex-shrink: 0; }
  &--error::before { content: ''; width: 6px; height: 6px; border-radius: 50%; background: #FF6B6B; flex-shrink: 0; }
}

@media (max-width: 900px) {
  .cards-row { grid-template-columns: 1fr; }
  .metric-templates { grid-template-columns: 1fr; }
  .transform-summary { grid-template-columns: repeat(2, 1fr); }
}

@media (max-width: 640px) {
  .cm__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .pg-body-scroll { padding: 16px 18px 22px; }
  .rule-fields { flex-wrap: wrap; }
  .mapping-row { flex-wrap: wrap; }
  .mapping-target { flex: 1 1 100%; }
}
// ---- Add to CustomMetrics.module.scss -----------------------------------
// Paste at the end of the file, before the closing @media blocks (or after
// them — order doesn't matter since these are new, non-overlapping classes).

// ---- wizard step rail (supports 8 steps — scrolls on narrow screens) ----
.steps {
  overflow-x: auto;
  padding-bottom: 2px;

  &::-webkit-scrollbar { height: 4px; }
  &::-webkit-scrollbar-thumb { background: $line; border-radius: 2px; }
}

// ---- section heading inside a wizard step --------------------------------
.step-heading {
  font-family: $display;
  font-size: 1.0714em; // 0.9375rem / 0.875rem
  font-weight: 700;
  color: $ink;
  margin-bottom: 4px;
}

.step-sub {
  font-size: 0.8929em; // 0.78125rem / 0.875rem
  color: $ink-3;
  margin-bottom: 20px;
}

.textarea {
  width: 100%;
  min-height: 96px;
  border: 1.5px solid $line;
  border-radius: 9px;
  padding: 9px 12px;
  font-size: 0.9286em; // 0.8125rem / 0.875rem
  font-family: $sans;
  color: $ink;
  background: $card;
  resize: vertical;

  &::placeholder { color: $ink-3; }
  &:focus { outline: none; border-color: $signal; box-shadow: 0 0 0 3px $wash; }
}

// ---- multi-select checkbox list (built-in checks) --------------------
.checkbox-list {
  display: flex;
  flex-direction: column;
  gap: 2px;
  border: 1px solid $line;
  border-radius: 12px;
  overflow: hidden;
}

.checkbox-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 14px;
  background: $card;
  cursor: pointer;
  transition: background 0.13s ease;

  &:not(:last-child) { border-bottom: 1px solid $line-2; }
  &:hover { background: $paper; }

  input[type='checkbox'] { accent-color: $signal; flex-shrink: 0; }
}

.checkbox-item__name {
  font-weight: 650;
  font-size: 0.9286em; // 0.8125rem / 0.875rem
  color: $ink;
}

.checkbox-item__logic {
  font-family: $mono;
  font-size: 0.7857em; // 0.6875rem / 0.875rem
  color: $ink-3;
  margin-left: auto;
}

// ---- code editor language badge ------------------------------------------
.lang-badge {
  display: inline-flex;
  align-items: center;
  font-family: $mono;
  font-size: 0.7143em; // 0.625rem / 0.875rem
  font-weight: 700;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  color: $ok;
  background: $ok-wash;
  border-radius: 6px;
  padding: 3px 8px;
}

.editor-header-left {
  display: flex;
  align-items: center;
  gap: 8px;
}

// ---- judge model list ----------------------------------------------------
.model-list {
  display: flex;
  flex-direction: column;
  gap: 2px;
  border: 1px solid $line;
  border-radius: 12px;
  overflow: hidden;
}

.model-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 11px 14px;
  background: $card;
  cursor: pointer;
  transition: background 0.13s ease, opacity 0.13s ease;

  &:not(:last-child) { border-bottom: 1px solid $line-2; }
  &:hover:not(&--disabled) { background: $paper; }

  &--selected { background: $wash; }
  &--disabled { cursor: not-allowed; opacity: 0.5; }

  input[type='radio'] { accent-color: $signal; flex-shrink: 0; }
}

.model-item__name {
  font-weight: 650;
  font-size: 0.9286em; // 0.8125rem / 0.875rem
  color: $ink;
}

.model-item__provider {
  font-family: $mono;
  font-size: 0.7143em; // 0.625rem / 0.875rem
  color: $ink-3;
}

.model-item__status {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  margin-left: auto;
  font-family: $mono;
  font-size: 0.7143em; // 0.625rem / 0.875rem
  font-weight: 700;
  text-transform: uppercase;
}

.health-dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  flex-shrink: 0;

  &--healthy { background: $ok; }
  &--unhealthy { background: $danger; }
  &--checking { background: $ink-3; animation: cm-spin 1s linear infinite; border-radius: 2px; }
}

.status-healthy { color: $ok; }
.status-unhealthy { color: $danger; }
.status-checking { color: $ink-3; }

// ---- dataset preview list ---------------------------------------------
.preview-list {
  border: 1px solid $line;
  border-radius: 14px;
  overflow: hidden;
  max-height: 360px;
  overflow-y: auto;
}

.preview-item {
  display: flex;
  align-items: flex-start;
  gap: 10px;
  padding: 12px 14px;
  border-bottom: 1px solid $line-2;
  transition: background 0.13s ease;

  &:last-child { border-bottom: none; }
  &:hover { background: $paper; }

  input[type='checkbox'] { accent-color: $signal; margin-top: 3px; flex-shrink: 0; }
}

.preview-item__body { min-width: 0; flex: 1; }
.preview-item__q { font-size: 0.9286em; color: $ink; font-weight: 650; margin-bottom: 3px; }
.preview-item__a { font-size: 0.8214em; color: $ink-2; }
.preview-item__a-label { font-family: $mono; font-size: 0.7143em; color: $ink-3; margin-right: 5px; }

// ---- success modal ---------------------------------------------------
.modal-overlay {
  position: fixed;
  inset: 0;
  z-index: 300;
  display: flex;
  align-items: center;
  justify-content: center;
  background: rgba(10, 12, 18, 0.5);
  padding: 20px;
}

.modal {
  width: 100%;
  max-width: 380px;
  background: $card;
  border-radius: 18px;
  box-shadow: $lift;
  padding: 28px 24px 22px;
  text-align: center;
}

.modal-icon {
  width: 48px;
  height: 48px;
  margin: 0 auto 14px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  background: $ok-wash;
  color: $ok;
}

.modal h3 {
  font-family: $display;
  font-size: 1.1429em; // 1rem / 0.875rem
  font-weight: 800;
  color: $ink;
  margin-bottom: 6px;
}

.modal p {
  font-size: 0.9286em; // 0.8125rem / 0.875rem
  color: $ink-2;
  margin-bottom: 4px;
}

.modal-id {
  display: inline-block;
  font-family: $mono;
  font-size: 0.7857em; // 0.6875rem / 0.875rem
  font-weight: 700;
  color: $signal;
  background: $wash;
  border-radius: 7px;
  padding: 3px 8px;
  margin-bottom: 20px;
}

.modal-actions {
  display: flex;
  gap: 10px;
  justify-content: center;
}

// ---- generic banners -------------------------------------------------
.error-banner {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 14px;
  border-radius: 10px;
  background: $danger-wash;
  border: 1px solid rgba($danger, 0.2);
  color: $danger;
  font-size: 0.8929em; // 0.78125rem / 0.875rem
  margin-bottom: 16px;
}

.success-banner {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 14px;
  border-radius: 10px;
  background: $ok-wash;
  border: 1px solid rgba($ok, 0.2);
  color: $ok;
  font-size: 0.8929em; // 0.78125rem / 0.875rem
  margin-bottom: 16px;
  font-weight: 650;
}

.loading-row {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 20px;
  justify-content: center;
  color: $ink-3;
  font-size: 0.8929em; // 0.78125rem / 0.875rem
}














//Custommetricsdashboard.tsx
import { useNavigate } from 'react-router-dom';
import { Gauge, X } from 'lucide-react';
import styles from './CustomMetrics.module.scss';
import { SAVED_METRICS } from './mockData';

export default function CustomMetricsDashboard() {
  const navigate = useNavigate();

  const typeBadgeClass = (type: string) =>
    `${styles.badge} ${styles[`badge--${type}`] || ''}`;

  return (
    <div className={`page-enter pg-shell ${styles.cm}`}>
      <div className={styles['cm__header']}>
        <div>
          <p className={styles['cm__header-eyebrow']}>Custom Metrics</p>
          <h1>Dashboard</h1>
          <p className={styles['cm__header-sub']}>Saved metrics for evaluation</p>
        </div>
      </div>

      <div className={`pg-body ${styles['pg-body-scroll']}`}>
        <div className={styles.card}>
          <div className={styles['card-header']}>
            <h3>Saved Metrics</h3>
            <button
              type="button"
              className={`${styles.btn} ${styles['btn-sm']}`}
              onClick={() => navigate('/app/custom-metrics/create')}
            >
              + New
            </button>
          </div>
          <div className={styles['card-body']}>
            {SAVED_METRICS.length === 0 ? (
              <div className={styles.empty}>
                <Gauge size={16} /> No metrics saved yet — create your first custom metric to get started.
              </div>
            ) : (
              <div className={styles['table-wrap']}>
                <table className={styles.table}>
                  <thead>
                    <tr>
                      <th>Name</th>
                      <th>Type</th>
                      <th>Created</th>
                      <th></th>
                    </tr>
                  </thead>
                  <tbody>
                    {SAVED_METRICS.map((m) => (
                      <tr key={m.id}>
                        <td>{m.name}</td>
                        <td><span className={typeBadgeClass(m.type)}>{m.type === 'code' ? 'Code' : 'Visual'}</span></td>
                        <td>{m.created}</td>
                        <td>
                          <button type="button" className={styles['btn-icon']} title="Delete">
                            <X size={13} />
                          </button>
                        </td>
                      </tr>
                    ))}
                  </tbody>
                </table>
              </div>
            )}
          </div>
        </div>
      </div>
    </div>
  );
}

















//Sidebar.tsx
import { useState } from 'react';
import { Link, NavLink, useLocation } from 'react-router-dom';
import {
  Home, Link2, Cpu, BookOpen, Play, FlaskConical, GitCompare, FileText, LogOut,
  Gauge, ChevronDown, LayoutDashboard, PenSquare,
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { logout } from '../../store/slices/authSlice';
import ThemeToggle from '../common/ThemeToggle';
import styles from './Sidebar.module.scss';

const navItems = [
  { to: '/app/dashboard', icon: <Home size={18} />, label: 'Dashboard' },
  { to: '/app/providers', icon: <Link2 size={18} />, label: 'Providers' },
  { to: '/app/models', icon: <Cpu size={18} />, label: 'Models' },
  { to: '/app/datasets', icon: <BookOpen size={18} />, label: 'Datasets' },
];

const workflowItems = [
  { to: '/app/run-evaluation', icon: <Play size={18} />, label: 'New Evaluation' },
  { to: '/app/history', icon: <FlaskConical size={18} />, label: 'History' },
  { to: '/app/comparison', icon: <GitCompare size={18} />, label: 'Comparison' },
  { to: '/app/reports', icon: <FileText size={18} />, label: 'Reports' },
];

const customMetricsSubItems = [
  { to: '/app/custom-metrics/dashboard', icon: <LayoutDashboard size={15} />, label: 'Dashboard' },
  { to: '/app/custom-metrics/create', icon: <PenSquare size={15} />, label: 'Create Metric' },
];

export default function Sidebar() {
  const dispatch = useAppDispatch();
  const user = useAppSelector((s) => s.auth.user);
  const location = useLocation();

  const isOnCustomMetrics = location.pathname.startsWith('/app/custom-metrics');
  const [customMetricsOpen, setCustomMetricsOpen] = useState(isOnCustomMetrics);

  const navLinkClass = ({ isActive }: { isActive: boolean }) =>
    `${styles['nav-item']} ${isActive ? styles.active : ''}`;

  const subNavLinkClass = ({ isActive }: { isActive: boolean }) =>
    `${styles['nav-item']} ${styles['nav-item--sub']} ${isActive ? styles.active : ''}`;

  return (
    <div className={styles.sidebar}>
      <Link to="/" className={styles['sidebar__logo']}>
        <div className={styles['sidebar__mark']}>&#9670;</div>
        SemcoEval
      </Link>
      <nav className={styles['sidebar__nav']}>
        {navItems.map((item) => (
          <NavLink key={item.to} to={item.to} className={navLinkClass}>
            {item.icon}
            {item.label}
          </NavLink>
        ))}

        <div className={styles['sidebar__section']}>Workflow</div>
        {workflowItems.map((item) => (
          <NavLink key={item.to} to={item.to} className={navLinkClass}>
            {item.icon}
            {item.label}
          </NavLink>
        ))}

        <button
          type="button"
          className={`${styles['nav-item']} ${styles['nav-item--expandable']} ${isOnCustomMetrics ? styles.active : ''}`}
          onClick={() => setCustomMetricsOpen((o) => !o)}
          aria-expanded={customMetricsOpen}
        >
          <Gauge size={18} />
          Custom Metrics
          <ChevronDown
            size={14}
            className={`${styles['nav-item__chevron']} ${customMetricsOpen ? styles['nav-item__chevron--open'] : ''}`}
          />
        </button>

        <div className={`${styles['nav-submenu']} ${customMetricsOpen ? styles['nav-submenu--open'] : ''}`}>
          <div className={styles['nav-submenu__inner']}>
            {customMetricsSubItems.map((item) => (
              <NavLink key={item.to} to={item.to} className={subNavLinkClass}>
                {item.icon}
                {item.label}
              </NavLink>
            ))}
          </div>
        </div>
      </nav>
      <div className={styles['sidebar__foot']}>
        <div className={styles['sidebar__theme-row']}>
          <span>Theme</span>
          <ThemeToggle />
        </div>
        <div className={styles['sidebar__user']}>
          <div className={styles['sidebar__avatar']}>
            {(user?.name || user?.email || '?').slice(0, 1).toUpperCase()}
          </div>
          <div className={styles['sidebar__user-info']}>
            <div className={styles['sidebar__user-name']}>{user?.name || 'Account'}</div>
            <div className={styles['sidebar__user-email']}>{user?.email}</div>
          </div>
          <button
            type="button"
            className={styles['sidebar__logout']}
            title="Log out"
            onClick={() => dispatch(logout())}
          >
            <LogOut size={15} />
          </button>
        </div>
      </div>
    </div>
  );
}

















//Approutes.tsx
import { Routes, Route, Navigate } from 'react-router-dom';
import Landing from '../components/landing/Landing';
import AppShell from '../components/layout/AppShell';
import ProtectedRoute from './ProtectedRoute';
import Dashboard from '../components/dashboard/Dashboard';
import Providers from '../components/providers/Providers';
import ModelCatalog from '../components/models/ModelCatalog';
import TestSuites from '../components/suites/TestSuites';
import NewEvaluation from '../components/evaluations/NewEvaluation';
import Evaluations from '../components/evaluations/Evaluations';
import EvaluationDetail from '../components/evaluations/EvaluationDetail';
import Comparison from '../components/comparison/Comparison';
import CustomMetricsDashboard from '../components/CustomMetrics/CustomMetricsDashboard';
import CreateMetric from '../components/CustomMetrics/CreateMetric';

const MOCKS_ENABLED = import.meta.env.VITE_ENABLE_MOCKS === 'true';

export default function AppRoutes() {
  return (
    <Routes>
      {/* In mock mode, main.tsx already seeds a session before render, so
          skip the landing/SSO page entirely and land straight in the app. */}
      <Route path="/" element={MOCKS_ENABLED ? <Navigate to="/app/dashboard" replace /> : <Landing />} />

      <Route element={<ProtectedRoute />}>
        <Route path="/app" element={<AppShell />}>
          <Route index element={<Navigate to="dashboard" replace />} />
          <Route path="dashboard" element={<Dashboard />} />
          <Route path="providers" element={<Providers />} />
          <Route path="models" element={<ModelCatalog />} />
          <Route path="suites" element={<TestSuites />} />
          <Route path="new-eval" element={<NewEvaluation />} />
          <Route path="evaluations" element={<Evaluations />} />
          <Route path="evaluations/:id" element={<EvaluationDetail />} />
          <Route path="comparison" element={<Comparison />} />

          <Route path="custom-metrics">
            <Route index element={<Navigate to="dashboard" replace />} />
            <Route path="dashboard" element={<CustomMetricsDashboard />} />
            <Route path="create" element={<CreateMetric />} />
          </Route>
        </Route>
      </Route>

      <Route path="*" element={<Navigate to="/" replace />} />
    </Routes>
  );
}
