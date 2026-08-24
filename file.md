import { useEffect, useMemo, useState } from 'react';
import { createPortal } from 'react-dom';
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

      {createPortal(
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
        </div>,
        document.body,
      )}

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
.rule-value { flex: 1; min-width: 160px; }

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

.badge--simple { color: $signal; background: $wash; }
.badge--active { color: $ok; background: $ok-wash; }
.badge--inactive { color: $ink-3; background: $ink-wash; }

// ---- dataset preview list — 2-column grid of individual cards ----------
.preview-list {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 10px;
  max-height: 420px;
  overflow-y: auto;
  padding-right: 2px;
}

.preview-item {
  display: flex;
  align-items: flex-start;
  gap: 10px;
  padding: 12px 14px;
  border: 1px solid $line;
  border-radius: 12px;
  background: $card;
  transition: border-color 0.13s ease, background 0.13s ease;

  &:hover { border-color: $ink-3; background: $paper; }

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
// ---- Add to CustomMetrics.module.scss ------------------------------------

// ---- section wrapper -------------------------------------------------
.section {
  margin-bottom: 28px;

  &:last-child { margin-bottom: 0; }
}

.section-title {
  font-family: $display;
  font-size: 1.0714em; // 0.9375rem / 0.875rem
  font-weight: 700;
  color: $ink;
  margin-bottom: 2px;
}

.section-sub {
  font-size: 0.8929em; // 0.78125rem / 0.875rem
  color: $ink-3;
  margin-bottom: 16px;
}

// ---- selectable card grids (eval type / metric type) ---------------------
.card-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 12px;

  &--4 { grid-template-columns: repeat(4, 1fr); }
}

.select-card {
  position: relative;
  border: 1.5px solid $line;
  border-radius: 14px;
  padding: 16px 14px;
  cursor: pointer;
  background: $paper;
  transition: border-color 0.15s ease, box-shadow 0.15s ease, background 0.15s ease, opacity 0.15s ease;

  &:hover:not(&--disabled) { border-color: $ink-3; }

  &--selected {
    border-color: $signal;
    background: $wash;
    box-shadow: 0 0 0 1px $signal inset;
  }

  &--disabled {
    cursor: not-allowed;
    opacity: 0.5;
  }
}

.select-card__title {
  font-family: $display;
  font-weight: 700;
  font-size: 0.9286em; // 0.8125rem / 0.875rem
  color: $ink;
  margin-bottom: 5px;
}

.select-card__desc {
  font-size: 0.8214em; // 0.71875rem / 0.875rem
  color: $ink-2;
  line-height: 1.45;
}

.select-card__badge {
  position: absolute;
  top: 12px;
  right: 12px;
  font-family: $mono;
  font-size: 0.6429em; // 0.5625rem / 0.875rem
  font-weight: 700;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  color: $ink-3;
  background: $card;
  border: 1px solid $line;
  border-radius: 6px;
  padding: 2px 6px;
}

// ---- rule builder with AND/OR gates between rows -------------------------
.rule-row-wrap {
  display: flex;
  flex-direction: column;
  align-items: stretch;
}

.rule-gate {
  display: flex;
  align-items: center;
  gap: 10px;
  margin: 6px 0;

  &::before, &::after {
    content: '';
    flex: 1;
    height: 1px;
    background: $line;
  }
}

.rule-gate-btn {
  display: inline-flex;
  padding: 2px;
  background: $paper;
  border: 1px solid $line;
  border-radius: 8px;
  gap: 1px;
  flex-shrink: 0;
}

.rule-gate-opt {
  padding: 4px 10px;
  border-radius: 6px;
  border: none;
  background: transparent;
  color: $ink-2;
  font-family: $mono;
  font-size: 0.7143em; // 0.625rem / 0.875rem
  font-weight: 700;
  cursor: pointer;

  &.active { background: $signal; color: #fff; }
}

.rule-summary {
  margin-top: 14px;

  label {
    display: block;
    @extend %micro;
    font-size: 0.625rem;
    color: $ink-3;
    margin-bottom: 6px;
  }

  code {
    display: block;
    font-family: $mono;
    font-size: 0.8214em; // 0.71875rem / 0.875rem
    color: $ink;
    background: $paper;
    border: 1px solid $line;
    border-radius: 9px;
    padding: 10px 12px;
    line-height: 1.6;
  }
}

// ---- prompt builder ----------------------------------------------------
.prompt-template-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin-bottom: 16px;
}

.prompt-template-item {
  display: flex;
  align-items: flex-start;
  gap: 10px;
  padding: 12px 14px;
  border: 1.5px solid $line;
  border-radius: 12px;
  cursor: pointer;
  background: $paper;
  transition: border-color 0.15s ease, background 0.15s ease;

  &:hover { border-color: $ink-3; }
  &--selected { border-color: $signal; background: $wash; }

  input[type='radio'] { accent-color: $signal; margin-top: 3px; flex-shrink: 0; }
}

.prompt-template-item__label { font-weight: 700; font-size: 0.9286em; color: $ink; }
.prompt-template-item__desc { font-size: 0.8214em; color: $ink-2; margin-top: 2px; }
.prompt-template-item__placeholders {
  display: flex;
  flex-wrap: wrap;
  gap: 4px;
  margin-top: 6px;
}

// ---- dataset preview toolbar --------------------------------------------
.preview-toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 8px;
}

.preview-toolbar__actions {
  display: flex;
  gap: 6px;
}

.link-btn {
  border: none;
  background: none;
  padding: 0;
  color: $signal;
  font-size: 0.8214em; // 0.71875rem / 0.875rem
  font-weight: 650;
  cursor: pointer;

  &:hover { text-decoration: underline; }
}

// ---- sticky footer -------------------------------------------------------
// True viewport-fixed (not scroll-sticky) so it stays put regardless of how
// far the form is scrolled — offset by the sidebar width on desktop.
.sticky-footer {
  position: fixed;
  bottom: 0;
  left: $sidebar-width;
  right: 0;
  z-index: 20;
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 10px;
  padding: 14px 32px;
  background: $card;
  border-top: 1px solid $line;
  box-shadow: 0 -8px 20px -12px rgba(20, 22, 27, 0.15);

  @media (max-width: 768px) {
    left: 0;
  }
}

// applied to .pg-body-scroll on pages that render a .sticky-footer, so the
// last section isn't hidden behind it
.pg-body-scroll--with-footer {
  padding-bottom: 90px;
}

.sticky-footer__info {
  margin-right: auto;
  font-size: 0.8214em; // 0.71875rem / 0.875rem
  color: $ink-3;
}

@media (max-width: 900px) {
  .card-grid, .card-grid--4 { grid-template-columns: 1fr; }
  .dataset-grid, .preview-list { grid-template-columns: 1fr; }
}
// ---- Add to CustomMetrics.module.scss ------------------------------------

// ---- custom dropdown (replaces native <select> in Create Metric) -------
.dropdown {
  position: relative;
  width: 100%;

  &--disabled { opacity: 0.5; pointer-events: none; }
}

.dropdown__trigger {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  border: 1.5px solid $line;
  border-radius: 9px;
  padding: 9px 11px;
  font-size: 0.9286em; // 0.8125rem / 0.875rem
  font-family: $sans;
  color: $ink;
  background: $card;
  cursor: pointer;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
  text-align: left;

  &:hover { border-color: $ink-3; }

  &--open {
    border-color: $signal;
    box-shadow: 0 0 0 3px $wash;
  }
}

.dropdown__value { color: $ink; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.dropdown__placeholder { color: $ink-3; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }

.dropdown__chevron {
  flex-shrink: 0;
  color: $ink-3;
  transition: transform 0.15s ease;

  &--open { transform: rotate(180deg); }
}

.dropdown__menu {
  position: absolute;
  top: calc(100% + 6px);
  left: 0;
  // No `right: 0` here on purpose — that forced the menu to exactly match
  // the trigger's width, so a squeezed flex-item trigger (e.g. the 4th
  // rule column, which sits in a `flex: 1` slot) produced an unreadably
  // narrow menu. `min-width: 100%` keeps it at least as wide as the
  // trigger; `width: max-content` lets it grow to fit the longest option
  // label instead of being clipped to the trigger's shrunk width.
  min-width: 100%;
  width: max-content;
  max-width: 320px;
  z-index: 40;
  max-height: 260px;
  overflow-y: auto;
  background: $card;
  border: 1px solid $line;
  border-radius: 12px;
  box-shadow: $lift;
  padding: 4px;
}

.dropdown__option {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  padding: 8px 10px;
  border: none;
  border-radius: 8px;
  background: transparent;
  color: $ink-2;
  font-size: 0.9286em; // 0.8125rem / 0.875rem
  font-weight: 550;
  text-align: left;
  cursor: pointer;
  transition: background 0.12s ease, color 0.12s ease;

  &:hover { background: $paper; color: $ink; }

  &--selected {
    background: $wash;
    color: $signal;
    font-weight: 700;
  }

  svg { flex-shrink: 0; color: $signal; }
}

.dropdown__option-sub {
  display: block;
  font-family: $mono;
  font-size: 0.8571em; // relative to option's own font-size
  font-weight: 500;
  color: $ink-3;
  margin-top: 1px;
}

.dropdown__empty {
  padding: 12px;
  text-align: center;
  color: $ink-3;
  font-size: 0.8214em; // 0.71875rem / 0.875rem
}

// ---- dataset selection cards --------------------------------------------
.dataset-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 12px;
  margin-bottom: 16px;
}

.dataset-card {
  position: relative;
  border: 1.5px solid $line;
  border-radius: 14px;
  padding: 14px 16px;
  cursor: pointer;
  background: $paper;
  transition: border-color 0.15s ease, box-shadow 0.15s ease, background 0.15s ease;
  display: flex;
  align-items: center;
  gap: 12px;

  &:hover { border-color: $ink-3; }

  &--selected {
    border-color: $signal;
    background: $wash;
    box-shadow: 0 0 0 1px $signal inset;
  }
}

.dataset-card__radio {
  flex-shrink: 0;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  border: 1.5px solid $line-2;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: border-color 0.15s ease;

  .dataset-card--selected & { border-color: $signal; }

  &::after {
    content: '';
    width: 9px;
    height: 9px;
    border-radius: 50%;
    background: $signal;
    opacity: 0;
    transition: opacity 0.15s ease;
  }

  .dataset-card--selected &::after { opacity: 1; }
}

.dataset-card__body { min-width: 0; flex: 1; }
.dataset-card__name {
  font-family: $display;
  font-weight: 700;
  font-size: 0.9286em; // 0.8125rem / 0.875rem
  color: $ink;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.dataset-card__count {
  font-family: $mono;
  font-size: 0.7857em; // 0.6875rem / 0.875rem
  color: $ink-3;
  margin-top: 2px;
}
