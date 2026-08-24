import { useEffect, useMemo, useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { AlertCircle, CheckCircle2, Loader2, Plus, X, XCircle } from 'lucide-react';
import styles from './CustomMetrics.module.scss';
import { useToast } from './useToast';
import CustomSelect from './CustomSelect';
import {
  metricsApi, EvalType, MetricType, PromptTemplate,
  ModelSummary, DatasetSummary, PreviewQuestion, ValidateMetricData,
  RuleDef,
} from '../../api/endpoints/metrics';

// ---- static config -----------------------------------------------------
const EVAL_TYPE_CARDS: { key: EvalType; label: string; desc: string; disabled?: boolean }[] = [
  { key: 'model', label: 'Model', desc: 'Evaluate a single model\u2019s input/output against expected answers.' },
  { key: 'agent', label: 'Agent', desc: 'Evaluate tool calls and task completion for agentic workflows.' },
  { key: 'rag', label: 'RAG', desc: 'Evaluate retrieval-augmented answers against retrieved context.', disabled: true },
];

const METRIC_TYPE_CARDS: { key: MetricType; label: string; desc: string }[] = [
  { key: 'visual', label: 'Visual Builder', desc: 'Combine field comparisons with AND/OR logic \u2014 no code required.' },
  { key: 'prompt', label: 'Prompt Builder', desc: 'Use an LLM judge with a template or custom scoring prompt.' },
  { key: 'code', label: 'Code Editor', desc: 'Write a custom scoring function in Python.' },
  { key: 'simple', label: 'Simple', desc: 'A minimal, no-configuration pass/fail metric.' },
];

const FIELDS_BY_EVAL_TYPE: Record<EvalType, string[]> = {
  model: ['input', 'actual_output', 'expected_output'],
  agent: ['input', 'actual_output', 'expected_output', 'tools_called', 'expected_tools'],
  rag: ['input', 'actual_output', 'expected_output', 'tools_called', 'expected_tools'],
};

const OPERATORS: { value: string; label: string }[] = [
  { value: 'contains', label: 'contains' },
  { value: 'not_contains', label: 'not contains' },
  { value: 'equals', label: 'equals' },
  { value: 'starts_with', label: 'starts with' },
  { value: 'ends_with', label: 'ends with' },
  { value: 'greater_than', label: 'greater than' },
  { value: 'less_than', label: 'less than' },
  { value: 'regex_match', label: 'regex match' },
];

const OPERATOR_SYMBOL: Record<string, string> = {
  contains: 'in', not_contains: 'not in', equals: '==', starts_with: 'starts with',
  ends_with: 'ends with', greater_than: '>', less_than: '<', regex_match: 'matches',
};

const METRIC_TYPE_TO_API: Record<MetricType, string> = {
  visual: 'condition', prompt: 'prompt', code: 'code', simple: 'simple',
};

type CompareType = 'field' | 'literal';

interface RuleRow {
  id: number;
  field: string;
  operator: string;
  compareType: CompareType;
  value: string;
}

let ruleSeq = 1;

export default function CreateMetric() {
  const navigate = useNavigate();
  const { showToast, ToastEl } = useToast();

  // ---- Section 1: details -----------------------------------------------
  const [name, setName] = useState('');
  const [description, setDescription] = useState('');

  // ---- Section 2: eval type -----------------------------------------------
  const [evalType, setEvalType] = useState<EvalType | null>(null);

  // ---- Section 3: metric type -----------------------------------------------
  const [metricType, setMetricType] = useState<MetricType | null>(null);

  // ---- Section 4a: visual builder rules -------------------------------
  const [rules, setRules] = useState<RuleRow[]>([
    { id: ruleSeq, field: 'actual_output', operator: 'contains', compareType: 'field', value: 'input' },
  ]);
  const [gates, setGates] = useState<('AND' | 'OR')[]>([]);

  // ---- Section 4b: prompt builder -----------------------------------------
  const [promptTemplates, setPromptTemplates] = useState<PromptTemplate[]>([]);
  const [promptTemplatesLoading, setPromptTemplatesLoading] = useState(false);
  const [promptTemplatesError, setPromptTemplatesError] = useState('');
  const [selectedTemplateName, setSelectedTemplateName] = useState<string>('');
  const [promptText, setPromptText] = useState('');

  const [models, setModels] = useState<ModelSummary[]>([]);
  const [modelsLoading, setModelsLoading] = useState(false);
  const [modelsError, setModelsError] = useState('');
  const [modelHealth, setModelHealth] = useState<Record<string, 'checking' | 'healthy' | 'unhealthy'>>({});
  const [selectedModelId, setSelectedModelId] = useState('');

  // ---- Section 4c: code editor -----------------------------------------
  const [code, setCode] = useState('');
  const [codeLoading, setCodeLoading] = useState(false);
  const [codeError, setCodeError] = useState('');

  // ---- Section 5: threshold -----------------------------------------------
  const [threshold, setThreshold] = useState(0.7);

  // ---- Section 6: dataset -----------------------------------------------
  const [datasets, setDatasets] = useState<DatasetSummary[]>([]);
  const [datasetsLoading, setDatasetsLoading] = useState(false);
  const [datasetsError, setDatasetsError] = useState('');
  const [selectedDatasetId, setSelectedDatasetId] = useState('');

  const [previewQuestions, setPreviewQuestions] = useState<PreviewQuestion[]>([]);
  const [previewLoading, setPreviewLoading] = useState(false);
  const [previewError, setPreviewError] = useState('');
  const [selectedQuestionIds, setSelectedQuestionIds] = useState<Set<string>>(new Set());

  // ---- footer: validate / save -----------------------------------------
  const [validating, setValidating] = useState(false);
  const [validateError, setValidateError] = useState('');
  const [validateResult, setValidateResult] = useState<ValidateMetricData | null>(null);
  const [saving, setSaving] = useState(false);
  const [saveError, setSaveError] = useState('');
  const [savedId, setSavedId] = useState('');

  const fields = evalType ? FIELDS_BY_EVAL_TYPE[evalType] : [];

  // ---- reset downstream state whenever eval type or metric type changes --
  const handleEvalType = (t: EvalType) => {
    if (t === evalType) return;
    setEvalType(t);
    setDatasets([]); setSelectedDatasetId('');
    setPreviewQuestions([]); setSelectedQuestionIds(new Set());
    setCode(''); setPromptText(''); setSelectedTemplateName('');
    setValidateResult(null); setSavedId('');
  };

  const handleMetricType = (t: MetricType) => {
    if (t === metricType) return;
    setMetricType(t);
    setValidateResult(null); setSavedId('');
  };

  // ---- Section 4a: rules ------------------------------------------------
  const addRule = () => {
    ruleSeq += 1;
    setRules((r) => [...r, { id: ruleSeq, field: fields[0] || 'input', operator: 'contains', compareType: 'literal', value: '' }]);
    setGates((g) => [...g, 'AND']);
  };

  const removeRule = (id: number) => {
    setRules((r) => {
      if (r.length <= 1) return r;
      const idx = r.findIndex((row) => row.id === id);
      const nextRules = r.filter((row) => row.id !== id);
      setGates((g) => g.filter((_, i) => i !== Math.max(0, idx - 1)));
      return nextRules;
    });
  };

  const updateRule = (id: number, patch: Partial<RuleRow>) => {
    setRules((r) => r.map((row) => (row.id === id ? { ...row, ...patch } : row)));
  };

  const toggleGate = (idx: number) => {
    setGates((g) => g.map((v, i) => (i === idx ? (v === 'AND' ? 'OR' : 'AND') : v)));
  };

  const ruleSummary = useMemo(() => {
    return rules
      .map((r, i) => {
        const compare = r.compareType === 'field' ? r.value || '<field>' : `"${r.value || '…'}"`;
        const line = `${r.field} ${OPERATOR_SYMBOL[r.operator] || r.operator} ${compare}`;
        return i === 0 ? line : `${gates[i - 1] || 'AND'} ${line}`;
      })
      .join(' ');
  }, [rules, gates]);

  // ---- Section 4b: prompt builder ---------------------------------------
  useEffect(() => {
    if (metricType !== 'prompt' || promptTemplates.length > 0) return;
    setPromptTemplatesLoading(true);
    setPromptTemplatesError('');
    metricsApi.getPromptTemplates()
      .then(setPromptTemplates)
      .catch((err) => setPromptTemplatesError(err.message || 'Failed to load templates'))
      .finally(() => setPromptTemplatesLoading(false));
  }, [metricType, promptTemplates.length]);

  const matchingTemplates = useMemo(
    () => promptTemplates.filter((t) => t.category === evalType),
    [promptTemplates, evalType],
  );
  const allowsCustomPrompt = evalType === 'agent' || evalType === 'rag';

  const selectTemplate = (t: PromptTemplate) => {
    setSelectedTemplateName(t.name);
    setPromptText(t.template);
  };

  const selectCustomPrompt = () => {
    setSelectedTemplateName('__custom__');
    setPromptText('');
  };

  useEffect(() => {
    if (metricType !== 'prompt' || models.length > 0) return;
    setModelsLoading(true);
    setModelsError('');
    metricsApi.listModels()
      .then((list) => {
        setModels(list);
        const initial: Record<string, 'checking'> = {};
        list.forEach((m) => { initial[m.id] = 'checking'; });
        setModelHealth(initial);
        list.forEach((m) => {
          metricsApi.checkModelHealth(m.id).then((h) => {
            setModelHealth((prev) => ({ ...prev, [m.id]: h.success ? 'healthy' : 'unhealthy' }));
          });
        });
      })
      .catch((err) => setModelsError(err.message || 'Failed to load models'))
      .finally(() => setModelsLoading(false));
  }, [metricType, models.length]);

  // ---- Section 4c: code editor -----------------------------------------
  useEffect(() => {
    if (metricType !== 'code' || !evalType || code) return;
    setCodeLoading(true);
    setCodeError('');
    metricsApi.getCodeTemplate(evalType)
      .then((res) => setCode(res.code))
      .catch((err) => setCodeError(err.message || 'Failed to load starter code'))
      .finally(() => setCodeLoading(false));
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [metricType, evalType]);

  // ---- Section 6: datasets ------------------------------------------------
  useEffect(() => {
    if (!evalType) return;
    setDatasetsLoading(true);
    setDatasetsError('');
    setSelectedDatasetId('');
    setPreviewQuestions([]);
    metricsApi.listDatasets(evalType)
      .then(setDatasets)
      .catch((err) => setDatasetsError(err.message || 'Failed to load datasets'))
      .finally(() => setDatasetsLoading(false));
  }, [evalType]);

  const selectDataset = (id: string) => {
    setSelectedDatasetId(id);
    setValidateResult(null); setSavedId('');
    setPreviewLoading(true);
    setPreviewError('');
    metricsApi.previewDataset(id)
      .then((res) => {
        const qs = res.questions.slice(0, 5);
        setPreviewQuestions(qs);
        setSelectedQuestionIds(new Set(qs.map((q) => q.id)));
      })
      .catch((err) => setPreviewError(err.message || 'Failed to load dataset preview'))
      .finally(() => setPreviewLoading(false));
  };

  const toggleQuestion = (id: string) => {
    setSelectedQuestionIds((prev) => {
      const next = new Set(prev);
      next.has(id) ? next.delete(id) : next.add(id);
      return next;
    });
  };

  const selectAllQuestions = () => setSelectedQuestionIds(new Set(previewQuestions.map((q) => q.id)));
  const clearAllQuestions = () => setSelectedQuestionIds(new Set());

  // ---- validation gating --------------------------------------------------
  const configComplete = (): boolean => {
    if (!metricType) return false;
    if (metricType === 'visual') return rules.every((r) => r.field && r.operator && (r.compareType === 'field' ? r.value : r.value.trim()));
    if (metricType === 'prompt') return !!promptText.trim() && !!selectedModelId;
    if (metricType === 'code') return !!code.trim();
    return true; // simple
  };

  const canValidate = !!name.trim() && !!evalType && configComplete() && threshold >= 0 && threshold <= 1
    && !!selectedDatasetId && selectedQuestionIds.size > 0;

  // ---- Validate -----------------------------------------------------------
  const buildDefinition = () => {
    if (metricType === 'visual') {
      return { rules: rules.map<RuleDef>((r) => ({
        field: r.field, operator: r.operator, value: r.value, compare_to_field: r.compareType === 'field',
      })) };
    }
    if (metricType === 'prompt') return { prompt_template: promptText };
    if (metricType === 'code') return { code, skip_validation: true };
    return {};
  };

  const runValidate = () => {
    if (!canValidate || !evalType || !metricType) {
      showToast('Fill in all required fields first', 'error');
      return;
    }
    setValidating(true);
    setValidateError('');
    setValidateResult(null);

    const selectedQuestions = previewQuestions.filter((q) => selectedQuestionIds.has(q.id));

    metricsApi.validate({
      actual_output: '',
      context: [],
      definition: buildDefinition(),
      description,
      eval_types: [evalType],
      expected_output: '',
      expected_tools: [],
      gates: metricType === 'visual' ? gates : [],
      input: '',
      judge_config: metricType === 'prompt' ? { model_id: selectedModelId } : null,
      metric_type: METRIC_TYPE_TO_API[metricType],
      name,
      retrieval_context: [],
      test_cases: selectedQuestions.map((q) => ({
        input: q.input?.prompt || '',
        actual_output: '',
        expected_output: q.expected?.answer || '',
        context: [],
        retrieval_context: [],
        tools_called: [],
        expected_tools: [],
      })),
      threshold: threshold.toFixed(2),
      tools_called: [],
    })
      .then(setValidateResult)
      .catch((err) => setValidateError(err.message || 'Validation failed'))
      .finally(() => setValidating(false));
  };

  const validateSucceeded = !!validateResult && validateResult.passed > 0;

  // ---- Save -----------------------------------------------------------
  const handleSave = () => {
    if (!validateSucceeded || !evalType || !metricType) {
      showToast('Run a successful validation before saving', 'error');
      return;
    }
    setSaving(true);
    setSaveError('');
    metricsApi.create({
      definition: buildDefinition(),
      description,
      eval_types: [evalType],
      metric_type: METRIC_TYPE_TO_API[metricType],
      name,
      threshold: threshold.toFixed(2),
      judge_config: metricType === 'prompt' ? { model_id: selectedModelId } : null,
    })
      .then((res) => setSavedId(res.id || 'saved'))
      .catch((err) => setSaveError(err.message || 'Failed to save metric'))
      .finally(() => setSaving(false));
  };

  const resetForm = () => {
    setName(''); setDescription('');
    setEvalType(null); setMetricType(null);
    setRules([{ id: ++ruleSeq, field: 'actual_output', operator: 'contains', compareType: 'field', value: 'input' }]);
    setGates([]);
    setPromptTemplates([]); setSelectedTemplateName(''); setPromptText('');
    setModels([]); setModelHealth({}); setSelectedModelId('');
    setCode('');
    setThreshold(0.7);
    setDatasets([]); setSelectedDatasetId('');
    setPreviewQuestions([]); setSelectedQuestionIds(new Set());
    setValidateResult(null); setValidateError('');
    setSavedId('');
  };

  return (
    <div className={`page-enter pg-shell ${styles.cm}`}>
      <div className={styles['cm__header']}>
        <div>
          <p className={styles['cm__header-eyebrow']}>Custom Metrics</p>
          <h1>Create Custom Metric</h1>
          <p className={styles['cm__header-sub']}>Define, configure, and validate a custom evaluation metric</p>
        </div>
      </div>

      <div className={`pg-body ${styles['pg-body-scroll']} ${styles['pg-body-scroll--with-footer']}`}>
        <div className={styles.panel}>

          {/* ---- Section 1: Metric Details ---- */}
          <div className={styles.section}>
            <h3 className={styles['section-title']}>Metric Details</h3>
            <p className={styles['section-sub']}>Give your metric a name and describe what it measures</p>

            <div className={styles['form-group']}>
              <label>Name *</label>
              <input className={styles.input} placeholder="e.g., Answer Faithfulness" value={name} onChange={(e) => setName(e.target.value)} />
            </div>
            <div className={styles['form-group']}>
              <label>Description</label>
              <textarea className={styles.textarea} placeholder="What does this metric measure? (optional)" value={description} onChange={(e) => setDescription(e.target.value)} />
            </div>
          </div>

          {/* ---- Section 2: Evaluation Type ---- */}
          <div className={styles.section}>
            <h3 className={styles['section-title']}>Evaluation Type</h3>
            <p className={styles['section-sub']}>Which kind of evaluation is this metric for?</p>

            <div className={styles['card-grid']}>
              {EVAL_TYPE_CARDS.map((c) => (
                <div
                  key={c.key}
                  className={`${styles['select-card']} ${evalType === c.key ? styles['select-card--selected'] : ''} ${c.disabled ? styles['select-card--disabled'] : ''}`}
                  onClick={() => !c.disabled && handleEvalType(c.key)}
                >
                  {c.disabled && <span className={styles['select-card__badge']}>Coming soon</span>}
                  <div className={styles['select-card__title']}>{c.label}</div>
                  <div className={styles['select-card__desc']}>{c.desc}</div>
                </div>
              ))}
            </div>
          </div>

          {/* ---- Section 3: Metric Type ---- */}
          <div className={styles.section}>
            <h3 className={styles['section-title']}>Metric Type</h3>
            <p className={styles['section-sub']}>How should this metric score a response?</p>

            <div className={`${styles['card-grid']} ${styles['card-grid--4']}`}>
              {METRIC_TYPE_CARDS.map((c) => (
                <div
                  key={c.key}
                  className={`${styles['select-card']} ${metricType === c.key ? styles['select-card--selected'] : ''}`}
                  onClick={() => handleMetricType(c.key)}
                >
                  <div className={styles['select-card__title']}>{c.label}</div>
                  <div className={styles['select-card__desc']}>{c.desc}</div>
                </div>
              ))}
            </div>
          </div>

          {/* ---- Section 4a: Rules (Visual Builder) ---- */}
          {metricType === 'visual' && evalType && (
            <div className={styles.section}>
              <h3 className={styles['section-title']}>Rules</h3>
              <p className={styles['section-sub']}>Combine one or more field comparisons</p>

              <div className={styles['rule-row-wrap']}>
                {rules.map((rule, i) => (
                  <div key={rule.id}>
                    {i > 0 && (
                      <div className={styles['rule-gate']}>
                        <div className={styles['rule-gate-btn']}>
                          {(['AND', 'OR'] as const).map((g) => (
                            <button
                              key={g}
                              type="button"
                              className={`${styles['rule-gate-opt']} ${gates[i - 1] === g ? styles.active : ''}`}
                              onClick={() => toggleGate(i - 1)}
                            >
                              {g}
                            </button>
                          ))}
                        </div>
                      </div>
                    )}
                    <div className={styles['rule-item']}>
                      <div className={styles['rule-fields']}>
                        <CustomSelect
                          className={styles['rule-field-select']}
                          value={rule.field}
                          onChange={(v) => updateRule(rule.id, { field: v })}
                          options={fields.map((f) => ({ value: f, label: f }))}
                        />
                        <CustomSelect
                          className={styles['rule-operator']}
                          value={rule.operator}
                          onChange={(v) => updateRule(rule.id, { operator: v })}
                          options={OPERATORS}
                        />
                        <CustomSelect
                          className={styles['rule-compare-type']}
                          value={rule.compareType}
                          onChange={(v) => updateRule(rule.id, { compareType: v as CompareType, value: '' })}
                          options={[
                            { value: 'field', label: 'Compared to Field' },
                            { value: 'literal', label: 'Literal Value' },
                          ]}
                        />
                        {rule.compareType === 'literal' ? (
                          <input className={`${styles.input} ${styles['rule-value']}`} placeholder="enter value" value={rule.value} onChange={(e) => updateRule(rule.id, { value: e.target.value })} />
                        ) : (
                          <CustomSelect
                            className={styles['rule-value']}
                            value={rule.value}
                            onChange={(v) => updateRule(rule.id, { value: v })}
                            placeholder="select field…"
                            options={fields.map((f) => ({ value: f, label: f }))}
                          />
                        )}
                        <button type="button" className={styles['btn-icon']} title="Remove" onClick={() => removeRule(rule.id)}>
                          <X size={14} />
                        </button>
                      </div>
                    </div>
                  </div>
                ))}
              </div>

              <div style={{ marginTop: '10px' }}>
                <button type="button" className={`${styles.btn} ${styles['btn-sm']}`} onClick={addRule}>
                  <Plus size={13} /> Add Rule
                </button>
              </div>

              <div className={styles['rule-summary']}>
                <label>Summary</label>
                <code>{ruleSummary || 'No rules defined'}</code>
              </div>
            </div>
          )}

          {/* ---- Section 4b: Judge Prompt (Prompt Builder) ---- */}
          {metricType === 'prompt' && evalType && (
            <div className={styles.section}>
              <h3 className={styles['section-title']}>Judge Prompt</h3>
              <p className={styles['section-sub']}>Pick a template or write a custom scoring prompt</p>

              {promptTemplatesError && <div className={styles['error-banner']}><AlertCircle size={14} /> {promptTemplatesError}</div>}

              {promptTemplatesLoading ? (
                <div className={styles['loading-row']}><Loader2 size={14} className={styles.spin} /> Loading templates…</div>
              ) : (
                <>
                  <div className={styles['prompt-template-list']}>
                    {matchingTemplates.length === 0 && !allowsCustomPrompt && (
                      <div className={styles.empty}>No templates available for this evaluation type.</div>
                    )}
                    {matchingTemplates.map((t) => (
                      <label key={t.name} className={`${styles['prompt-template-item']} ${selectedTemplateName === t.name ? styles['prompt-template-item--selected'] : ''}`}>
                        <input type="radio" name="prompt-template" checked={selectedTemplateName === t.name} onChange={() => selectTemplate(t)} />
                        <div>
                          <div className={styles['prompt-template-item__label']}>{t.label}</div>
                          <div className={styles['prompt-template-item__desc']}>{t.description}</div>
                          {t.uses_placeholders?.length > 0 && (
                            <div className={styles['prompt-template-item__placeholders']}>
                              {t.uses_placeholders.map((p) => <span key={p} className={styles.chip}>{`{${p}}`}</span>)}
                            </div>
                          )}
                        </div>
                      </label>
                    ))}
                    {allowsCustomPrompt && (
                      <label className={`${styles['prompt-template-item']} ${selectedTemplateName === '__custom__' ? styles['prompt-template-item--selected'] : ''}`}>
                        <input type="radio" name="prompt-template" checked={selectedTemplateName === '__custom__'} onChange={selectCustomPrompt} />
                        <div>
                          <div className={styles['prompt-template-item__label']}>Custom Prompt</div>
                          <div className={styles['prompt-template-item__desc']}>Write your own judge prompt from scratch</div>
                        </div>
                      </label>
                    )}
                  </div>

                  {selectedTemplateName && (
                    <div className={styles['form-group']}>
                      <label>Prompt</label>
                      <textarea className={styles.textarea} style={{ minHeight: '160px' }} value={promptText} onChange={(e) => setPromptText(e.target.value)} placeholder="Enter your judge prompt…" />
                    </div>
                  )}
                </>
              )}

              <h3 className={styles['section-title']} style={{ marginTop: '20px' }}>Judge Model</h3>
              <p className={styles['section-sub']}>Choose which model will score responses</p>

              {modelsError && <div className={styles['error-banner']}><AlertCircle size={14} /> {modelsError}</div>}

              {modelsLoading ? (
                <div className={styles['loading-row']}><Loader2 size={14} className={styles.spin} /> Loading models…</div>
              ) : models.length === 0 ? (
                <div className={styles.empty}>No models available.</div>
              ) : (
                <div className={styles['model-list']}>
                  {models.map((m) => {
                    const health = modelHealth[m.id] || 'checking';
                    const disabled = health === 'unhealthy';
                    const selected = selectedModelId === m.id;
                    return (
                      <label key={m.id} className={`${styles['model-item']} ${selected ? styles['model-item--selected'] : ''} ${disabled ? styles['model-item--disabled'] : ''}`}>
                        <input type="radio" name="judge-model" checked={selected} disabled={disabled} onChange={() => setSelectedModelId(m.id)} />
                        <span className={styles['model-item__name']}>{m.name}</span>
                        <span className={styles['model-item__provider']}>{m.provider_id}</span>
                        <span className={`${styles['model-item__status']} ${styles[`status-${health}`]}`}>
                          <span className={`${styles['health-dot']} ${styles[`health-dot--${health}`]}`} />
                          {health === 'checking' && 'Checking…'}
                          {health === 'healthy' && 'Healthy'}
                          {health === 'unhealthy' && 'Offline'}
                        </span>
                      </label>
                    );
                  })}
                </div>
              )}
            </div>
          )}

          {/* ---- Section 4c: Scoring Function (Code Editor) ---- */}
          {metricType === 'code' && evalType && (
            <div className={styles.section}>
              <h3 className={styles['section-title']}>Scoring Function</h3>
              <p className={styles['section-sub']}>Starter code is based on the evaluation type — edit as needed</p>

              {codeError && <div className={styles['error-banner']}><AlertCircle size={14} /> {codeError}</div>}

              <div className={styles['code-editor']}>
                <div className={styles['editor-header']}>
                  <span>Scoring Function</span>
                  {codeLoading && <Loader2 size={12} className={styles.spin} />}
                </div>
                <textarea className={styles['code-area']} spellCheck={false} value={code} onChange={(e) => setCode(e.target.value)} placeholder="# Your scoring function will appear here" />
              </div>
            </div>
          )}

          {/* ---- Section 5: Threshold ---- */}
          <div className={styles.section}>
            <h3 className={styles['section-title']}>Threshold</h3>
            <p className={styles['section-sub']}>Minimum score (0.00–1.00) required to pass</p>
            <div className={styles['form-group']} style={{ maxWidth: '160px' }}>
              <input
                type="number" className={styles.input} min={0} max={1} step={0.01}
                value={threshold}
                onChange={(e) => setThreshold(Math.min(1, Math.max(0, Number(e.target.value))))}
              />
            </div>
          </div>

          {/* ---- Section 6: Dataset ---- */}
          <div className={styles.section}>
            <h3 className={styles['section-title']}>Test Dataset</h3>
            <p className={styles['section-sub']}>Choose a dataset to preview and validate against</p>

            {!evalType ? (
              <div className={styles.empty}>Select an evaluation type first.</div>
            ) : datasetsError ? (
              <div className={styles['error-banner']}><AlertCircle size={14} /> {datasetsError}</div>
            ) : datasetsLoading ? (
              <div className={styles['loading-row']}><Loader2 size={14} className={styles.spin} /> Loading datasets…</div>
            ) : datasets.length === 0 ? (
              <div className={styles.empty}>No datasets found for this evaluation type.</div>
            ) : (
              <div className={styles['dataset-grid']}>
                {datasets.map((d) => (
                  <div
                    key={d.id}
                    className={`${styles['dataset-card']} ${selectedDatasetId === d.id ? styles['dataset-card--selected'] : ''}`}
                    onClick={() => selectDataset(d.id)}
                  >
                    <span className={styles['dataset-card__radio']} />
                    <div className={styles['dataset-card__body']}>
                      <div className={styles['dataset-card__name']}>{d.name}</div>
                      <div className={styles['dataset-card__count']}>{d.question_count} questions</div>
                    </div>
                  </div>
                ))}
              </div>
            )}

            {previewError && <div className={styles['error-banner']}><AlertCircle size={14} /> {previewError}</div>}

            {previewLoading ? (
              <div className={styles['loading-row']}><Loader2 size={14} className={styles.spin} /> Loading preview…</div>
            ) : previewQuestions.length > 0 && (
              <>
                <div className={styles['preview-toolbar']}>
                  <span className={styles['section-sub']} style={{ marginBottom: 0 }}>{selectedQuestionIds.size} of {previewQuestions.length} selected</span>
                  <div className={styles['preview-toolbar__actions']}>
                    <button type="button" className={styles['link-btn']} onClick={selectAllQuestions}>Select all</button>
                    <span style={{ color: 'var(--line)' }}>·</span>
                    <button type="button" className={styles['link-btn']} onClick={clearAllQuestions}>Clear</button>
                  </div>
                </div>
                <div className={styles['preview-list']}>
                  {previewQuestions.map((q) => (
                    <label key={q.id} className={styles['preview-item']}>
                      <input type="checkbox" checked={selectedQuestionIds.has(q.id)} onChange={() => toggleQuestion(q.id)} />
                      <div className={styles['preview-item__body']}>
                        <div className={styles['preview-item__q']}>{q.input?.prompt}</div>
                        <div className={styles['preview-item__a']}>
                          <span className={styles['preview-item__a-label']}>Expected:</span>{q.expected?.answer}
                        </div>
                      </div>
                    </label>
                  ))}
                </div>
              </>
            )}
          </div>

          {/* ---- Validation results ---- */}
          {validateError && <div className={styles['error-banner']}><AlertCircle size={14} /> {validateError}</div>}

          {validateResult && (
            <div className={styles.section}>
              {validateSucceeded ? (
                <div className={styles['success-banner']}><CheckCircle2 size={14} /> Metric is valid! Ready to save.</div>
              ) : (
                <div className={styles['error-banner']}><XCircle size={14} /> Metric failed validation — review the results below.</div>
              )}
              <div className={styles['validation-results']}>
                <table className={styles.table}>
                  <thead>
                    <tr><th>Input</th><th>Output</th><th>Score</th><th>Reason</th><th>Status</th></tr>
                  </thead>
                  <tbody>
                    {validateResult.results.map((r, i) => (
                      <tr key={i}>
                        <td>{r.test_case.input}</td>
                        <td>{r.test_case.actual_output}</td>
                        <td className={r.success ? styles['cell-pass'] : styles['cell-fail']}>{r.score.toFixed(2)}</td>
                        <td>{r.reason}</td>
                        <td className={r.success ? styles['cell-pass'] : styles['cell-fail']}>{r.success ? 'Pass' : 'Fail'}</td>
                      </tr>
                    ))}
                  </tbody>
                </table>
                <div className={styles['validation-summary']}>
                  <span>{validateResult.passed}/{validateResult.total} passed</span>
                </div>
              </div>
            </div>
          )}
        </div>
      </div>

      <div className={styles['sticky-footer']}>
        <span className={styles['sticky-footer__info']}>
          {!canValidate && 'Complete all required fields to validate'}
        </span>
        <button type="button" className={`${styles.btn} ${styles['btn-secondary']}`} onClick={() => navigate('/app/custom-metrics/dashboard')}>
          Cancel
        </button>
        <button type="button" className={`${styles.btn} ${styles['btn-primary']}`} onClick={runValidate} disabled={!canValidate || validating}>
          {validating ? <Loader2 size={13} className={styles.spin} /> : null}
          Validate Metric
        </button>
        {validateSucceeded && (
          <button type="button" className={`${styles.btn} ${styles['btn-primary']}`} onClick={handleSave} disabled={saving}>
            {saving ? <Loader2 size={13} className={styles.spin} /> : null}
            Save Metric
          </button>
        )}
      </div>

      {saveError && (
        <div className={styles.toast}><AlertCircle size={14} /> {saveError}</div>
      )}

      {savedId && (
        <div className={styles['modal-overlay']}>
          <div className={styles.modal}>
            <div className={styles['modal-icon']}><CheckCircle2 size={22} /></div>
            <h3>Metric created successfully!</h3>
            <p>Your metric is now available for evaluations.</p>
            <div className={styles['modal-id']}>ID: {savedId}</div>
            <div className={styles['modal-actions']}>
              <button type="button" className={`${styles.btn} ${styles['btn-secondary']}`} onClick={resetForm}>Create Another</button>
              <button type="button" className={`${styles.btn} ${styles['btn-primary']}`} onClick={() => navigate('/app/custom-metrics/dashboard')}>Go to Dashboard</button>
            </div>
          </div>
        </div>
      )}

      {ToastEl}
    </div>
  );
}
