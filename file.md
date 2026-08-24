import { useEffect, useMemo, useState } from 'react';
import { useNavigate } from 'react-router-dom';
import {
  AlertCircle, ArrowLeft, ArrowRight, Boxes, Check, CheckCircle2, Code2, Cpu, Database,
  ListChecks, Loader2, Plus, ScrollText, SlidersHorizontal, Sparkles, Target, X, XCircle, Zap,
} from 'lucide-react';
// NB: the wizard was collapsed from 5 steps to 4 — Validate & Save now lives
// inside the Dataset step's footer instead of being its own step.
import styles from './CreateMetric.module.scss';
import { useToast } from './useToast';
import CustomSelect from './CustomSelect';
import {
  metricsApi, EvalType, MetricType, PromptTemplate,
  ModelSummary, DatasetSummary, PreviewQuestion, ValidateMetricData, RuleDef,
} from '../../api/endpoints/metrics';

// ---- static config -----------------------------------------------------
const EVAL_TYPE_CARDS: { key: EvalType; label: string; desc: string; icon: JSX.Element }[] = [
  { key: 'model', label: 'Model', desc: 'Score a model\u2019s output against an expected answer.', icon: <Cpu size={20} /> },
  { key: 'agent', label: 'Agent', desc: 'Evaluate tool calls and task completion for agents.', icon: <Zap size={20} /> },
  { key: 'rag', label: 'RAG', desc: 'Check answers grounded in retrieved context.', icon: <ScrollText size={20} /> },
];

const METRIC_TYPE_CARDS: { key: MetricType; label: string; desc: string; icon: JSX.Element }[] = [
  { key: 'visual', label: 'Visual Builder', desc: 'Field comparisons joined with AND/OR logic. No code.', icon: <SlidersHorizontal size={18} /> },
  { key: 'prompt', label: 'Prompt Builder', desc: 'An LLM judge scored with a prompt template.', icon: <Sparkles size={18} /> },
  { key: 'code', label: 'Code Editor', desc: 'A custom Python scoring function.', icon: <Code2 size={18} /> },
  { key: 'simple', label: 'Simple', desc: 'A minimal, zero-config pass/fail check.', icon: <Target size={18} /> },
];

const FIELDS_BY_EVAL_TYPE: Record<EvalType, string[]> = {
  model: ['input', 'actual_output', 'expected_output'],
  agent: ['input', 'actual_output', 'expected_output', 'tools_called', 'expected_tools'],
  rag: ['input', 'actual_output', 'expected_output', 'tools_called', 'expected_tools'],
};

const OPERATORS = [
  { value: 'contains', label: 'contains' },
  { value: 'not_contains', label: 'not contains' },
  { value: 'equals', label: 'equals' },
  { value: 'starts_with', label: 'starts with' },
  { value: 'ends_with', label: 'ends with' },
  { value: 'greater_than', label: 'greater than' },
  { value: 'less_than', label: 'less than' },
  { value: 'regex_match', label: 'regex match' },
];

const OP_SYMBOL: Record<string, string> = {
  contains: 'contains', not_contains: 'does not contain', equals: '==', starts_with: 'starts with',
  ends_with: 'ends with', greater_than: '>', less_than: '<', regex_match: 'matches',
};

const METRIC_TYPE_TO_API: Record<MetricType, string> = {
  visual: 'condition', prompt: 'prompt', code: 'code', simple: 'simple',
};

const EVAL_TYPE_TO_CATEGORY: Record<EvalType, string> = { model: 'llm', agent: 'agent', rag: 'rag' };

type CompareType = 'field' | 'literal';
interface RuleRow { id: number; field: string; operator: string; compareType: CompareType; value: string; }
let ruleSeq = 1;

type StepKey = 'details' | 'type' | 'config' | 'dataset';

interface StepDef { key: StepKey; label: string; }

export default function CreateMetric() {
  const navigate = useNavigate();
  const { showToast, ToastEl } = useToast();

  const [step, setStep] = useState<StepKey>('details');

  // details
  const [name, setName] = useState('');
  const [description, setDescription] = useState('');

  // type
  const [evalType, setEvalType] = useState<EvalType | null>(null);
  const [metricType, setMetricType] = useState<MetricType | null>(null);

  // config: visual
  const [rules, setRules] = useState<RuleRow[]>([{ id: ruleSeq, field: 'actual_output', operator: 'contains', compareType: 'field', value: 'input' }]);
  const [gates, setGates] = useState<('AND' | 'OR')[]>([]);

  // config: prompt
  const [promptTemplates, setPromptTemplates] = useState<PromptTemplate[]>([]);
  const [promptTemplatesLoading, setPromptTemplatesLoading] = useState(false);
  const [promptTemplatesError, setPromptTemplatesError] = useState('');
  const [selectedTemplateName, setSelectedTemplateName] = useState('');
  const [promptText, setPromptText] = useState('');
  const [models, setModels] = useState<ModelSummary[]>([]);
  const [modelsLoading, setModelsLoading] = useState(false);
  const [modelsError, setModelsError] = useState('');
  const [modelHealth, setModelHealth] = useState<Record<string, 'checking' | 'healthy' | 'unhealthy'>>({});
  const [selectedModelId, setSelectedModelId] = useState('');

  // config: code
  const [code, setCode] = useState('');
  const [codeLoading, setCodeLoading] = useState(false);
  const [codeError, setCodeError] = useState('');

  // threshold (folded into config step)
  const [threshold, setThreshold] = useState(0.7);

  // dataset
  const [datasets, setDatasets] = useState<DatasetSummary[]>([]);
  const [datasetsLoading, setDatasetsLoading] = useState(false);
  const [datasetsError, setDatasetsError] = useState('');
  const [selectedDatasetId, setSelectedDatasetId] = useState('');
  const [previewQuestions, setPreviewQuestions] = useState<PreviewQuestion[]>([]);
  const [previewLoading, setPreviewLoading] = useState(false);
  const [previewError, setPreviewError] = useState('');
  const [selectedQuestionIds, setSelectedQuestionIds] = useState<Set<string>>(new Set());

  // validate / save
  const [validating, setValidating] = useState(false);
  const [validateError, setValidateError] = useState('');
  const [validateResult, setValidateResult] = useState<ValidateMetricData | null>(null);
  const [saving, setSaving] = useState(false);
  const [saveError, setSaveError] = useState('');
  const [savedId, setSavedId] = useState('');

  const fields = evalType ? FIELDS_BY_EVAL_TYPE[evalType] : [];

  // ---- reset chains ------------------------------------------------------
  const handleEvalType = (t: EvalType) => {
    if (t === evalType) return;
    setEvalType(t);
    setDatasets([]); setSelectedDatasetId(''); setPreviewQuestions([]); setSelectedQuestionIds(new Set());
    setCode(''); setPromptText(''); setSelectedTemplateName(''); setValidateResult(null); setSavedId('');
  };
  const handleMetricType = (t: MetricType) => {
    if (t === metricType) return;
    setMetricType(t); setValidateResult(null); setSavedId('');
  };

  // ---- visual rules ------------------------------------------------------
  const addRule = () => {
    ruleSeq += 1;
    setRules((r) => [...r, { id: ruleSeq, field: fields[0] || 'input', operator: 'contains', compareType: 'literal', value: '' }]);
    setGates((g) => [...g, 'AND']);
  };
  const removeRule = (id: number) => {
    setRules((r) => {
      if (r.length <= 1) return r;
      const idx = r.findIndex((row) => row.id === id);
      setGates((g) => g.filter((_, i) => i !== Math.max(0, idx - 1)));
      return r.filter((row) => row.id !== id);
    });
  };
  const updateRule = (id: number, patch: Partial<RuleRow>) => setRules((r) => r.map((row) => (row.id === id ? { ...row, ...patch } : row)));
  const toggleGate = (idx: number) => setGates((g) => g.map((v, i) => (i === idx ? (v === 'AND' ? 'OR' : 'AND') : v)));

  // ---- prompt templates + models ----------------------------------------
  useEffect(() => {
    if (metricType !== 'prompt' || promptTemplates.length) return;
    setPromptTemplatesLoading(true); setPromptTemplatesError('');
    metricsApi.getPromptTemplates()
      .then(setPromptTemplates)
      .catch((e) => setPromptTemplatesError(e.message || 'Failed to load templates'))
      .finally(() => setPromptTemplatesLoading(false));
  }, [metricType, promptTemplates.length]);

  const matchingTemplates = useMemo(
    () => promptTemplates.filter((t) => t.category === (evalType ? EVAL_TYPE_TO_CATEGORY[evalType] : '')),
    [promptTemplates, evalType],
  );
  const allowsCustomPrompt = evalType === 'agent' || evalType === 'rag';

  useEffect(() => {
    if (metricType !== 'prompt' || models.length) return;
    setModelsLoading(true); setModelsError('');
    metricsApi.listModels()
      .then((list) => {
        setModels(list);
        const init: Record<string, 'checking'> = {};
        list.forEach((m) => { init[m.id] = 'checking'; });
        setModelHealth(init);
        list.forEach((m) => metricsApi.checkModelHealth(m.id).then((h) =>
          setModelHealth((prev) => ({ ...prev, [m.id]: h.success ? 'healthy' : 'unhealthy' }))));
      })
      .catch((e) => setModelsError(e.message || 'Failed to load models'))
      .finally(() => setModelsLoading(false));
  }, [metricType, models.length]);

  // ---- code template -----------------------------------------------------
  useEffect(() => {
    if (metricType !== 'code' || !evalType || code) return;
    setCodeLoading(true); setCodeError('');
    metricsApi.getCodeTemplate(evalType)
      .then((res) => setCode(res.code))
      .catch((e) => setCodeError(e.message || 'Failed to load starter code'))
      .finally(() => setCodeLoading(false));
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [metricType, evalType]);

  // ---- datasets ----------------------------------------------------------
  useEffect(() => {
    if (!evalType) return;
    setDatasetsLoading(true); setDatasetsError(''); setSelectedDatasetId(''); setPreviewQuestions([]);
    metricsApi.listDatasets(evalType)
      .then(setDatasets)
      .catch((e) => setDatasetsError(e.message || 'Failed to load datasets'))
      .finally(() => setDatasetsLoading(false));
  }, [evalType]);

  const selectDataset = (id: string) => {
    setSelectedDatasetId(id); setValidateResult(null); setSavedId('');
    setPreviewLoading(true); setPreviewError('');
    metricsApi.previewDataset(id)
      .then((res) => {
        const qs = res.questions.slice(0, 5);
        setPreviewQuestions(qs);
        setSelectedQuestionIds(new Set(qs.map((q) => q.id)));
      })
      .catch((e) => setPreviewError(e.message || 'Failed to load preview'))
      .finally(() => setPreviewLoading(false));
  };
  const toggleQuestion = (id: string) => setSelectedQuestionIds((prev) => {
    const next = new Set(prev); next.has(id) ? next.delete(id) : next.add(id); return next;
  });
  const selectAllQuestions = () => setSelectedQuestionIds(new Set(previewQuestions.map((q) => q.id)));
  const clearAllQuestions = () => setSelectedQuestionIds(new Set());

  // ---- rule summary ------------------------------------------------------
  const ruleSummary = useMemo(() => {
    if (!rules.length) return null;
    return rules.map((r, i) => {
      const compare = r.compareType === 'field' ? (r.value || '<field>') : `"${r.value || '…'}"`;
      return (
        <span key={r.id}>
          {i > 0 && <span className={styles['summary__gate']}>{gates[i - 1] || 'AND'}</span>}
          <span className={styles['summary__token']}>{r.field}</span>
          {' '}{OP_SYMBOL[r.operator] || r.operator}{' '}
          <span className={styles['summary__token']}>{compare}</span>
        </span>
      );
    });
  }, [rules, gates]);

  // ---- gating ------------------------------------------------------------
  const detailsComplete = !!name.trim();
  const typeComplete = !!evalType && !!metricType;
  const configComplete = useMemo(() => {
    if (!metricType) return false;
    if (metricType === 'visual') return rules.every((r) => r.field && r.operator && (r.compareType === 'field' ? r.value : r.value.trim()));
    if (metricType === 'prompt') return !!promptText.trim() && !!selectedModelId;
    if (metricType === 'code') return !!code.trim();
    return true;
  }, [metricType, rules, promptText, selectedModelId, code]);
  const datasetComplete = !!selectedDatasetId && selectedQuestionIds.size > 0;
  const canValidate = detailsComplete && typeComplete && configComplete && datasetComplete && threshold >= 0 && threshold <= 1;
  const validateSucceeded = !!validateResult && validateResult.passed > 0;

  const STEPS: StepDef[] = [
    { key: 'details', label: 'Metric Details' },
    { key: 'type', label: 'Type & Target' },
    { key: 'config', label: metricType === 'prompt' ? 'Judge Prompt' : metricType === 'code' ? 'Scoring Code' : metricType === 'simple' ? 'Configuration' : 'Rules' },
    { key: 'dataset', label: 'Dataset · Validate & Save' },
  ];

  const stepDone: Record<StepKey, boolean> = {
    details: detailsComplete,
    type: typeComplete,
    config: configComplete,
    dataset: datasetComplete && !!validateResult,
  };
  const stepEnabled: Record<StepKey, boolean> = {
    details: true,
    type: detailsComplete,
    config: typeComplete,
    dataset: typeComplete,
  };

  const stepValue: Record<StepKey, string> = {
    details: name || 'Not set',
    type: evalType && metricType ? `${evalType.toUpperCase()} · ${METRIC_TYPE_CARDS.find((c) => c.key === metricType)!.label}` : 'Not set',
    config: metricType ? (configComplete ? 'Configured' : 'Incomplete') : '—',
    dataset: validateResult ? `${validateResult.passed}/${validateResult.total} passed` : (selectedDatasetId ? `${selectedQuestionIds.size} selected` : 'Not set'),
  };

  const stepIndex = STEPS.findIndex((s) => s.key === step);
  const goStep = (k: StepKey) => { if (stepEnabled[k]) setStep(k); };
  const nextStep = () => { const n = STEPS[stepIndex + 1]; if (n && stepEnabled[n.key]) setStep(n.key); };
  const prevStep = () => { const p = STEPS[stepIndex - 1]; if (p) setStep(p.key); };

  // ---- validate / save ---------------------------------------------------
  const buildDefinition = () => {
    if (metricType === 'visual') return { rules: rules.map<RuleDef>((r) => ({ field: r.field, operator: r.operator, value: r.value, compare_to_field: r.compareType === 'field' })) };
    if (metricType === 'prompt') return { prompt_template: promptText };
    if (metricType === 'code') return { code, skip_validation: true };
    return {};
  };

  const runValidate = () => {
    if (!canValidate || !evalType || !metricType) { showToast('Complete all steps first', 'error'); return; }
    setValidating(true); setValidateError(''); setValidateResult(null);
    const selectedQs = previewQuestions.filter((q) => selectedQuestionIds.has(q.id));
    metricsApi.validate({
      actual_output: '', context: [], definition: buildDefinition(), description,
      eval_types: [evalType], expected_output: '', expected_tools: [],
      gates: metricType === 'visual' ? gates : [], input: '',
      judge_config: metricType === 'prompt' ? { model_id: selectedModelId } : null,
      metric_type: METRIC_TYPE_TO_API[metricType], name, retrieval_context: [],
      test_cases: selectedQs.map((q) => ({
        input: q.input?.prompt || '', actual_output: '', expected_output: q.expected?.answer || '',
        context: [], retrieval_context: [], tools_called: [], expected_tools: [],
      })),
      threshold: threshold.toFixed(2), tools_called: [],
    })
      .then(setValidateResult)
      .catch((e) => setValidateError(e.message || 'Validation failed'))
      .finally(() => setValidating(false));
  };

  const handleSave = () => {
    if (!validateResult || !evalType || !metricType) { showToast('Run validation before saving', 'error'); return; }
    setSaving(true); setSaveError('');
    metricsApi.create({
      definition: buildDefinition(), description, eval_types: [evalType],
      metric_type: METRIC_TYPE_TO_API[metricType], name, threshold: threshold.toFixed(2),
      judge_config: metricType === 'prompt' ? { model_id: selectedModelId } : null,
    })
      .then((res) => setSavedId(res.id || 'saved'))
      .catch((e) => setSaveError(e.message || 'Failed to save metric'))
      .finally(() => setSaving(false));
  };

  const resetForm = () => {
    setStep('details');
    setName(''); setDescription(''); setEvalType(null); setMetricType(null);
    setRules([{ id: ++ruleSeq, field: 'actual_output', operator: 'contains', compareType: 'field', value: 'input' }]); setGates([]);
    setPromptTemplates([]); setSelectedTemplateName(''); setPromptText('');
    setModels([]); setModelHealth({}); setSelectedModelId(''); setCode(''); setThreshold(0.7);
    setDatasets([]); setSelectedDatasetId(''); setPreviewQuestions([]); setSelectedQuestionIds(new Set());
    setValidateResult(null); setValidateError(''); setSavedId('');
  };

  const currentStepLabel = STEPS[stepIndex]?.label ?? '';

  // =========================================================================
  return (
    <div className={`page-enter ${styles.cm}`}>

      {/* ============ PAGE HEADER (matches Model Catalog header) ============ */}
      <div className={styles['page-header']}>
        <div>
          <p className={styles['page-header__eyebrow']}>Custom Metric</p>
          <h1 className={styles['page-header__title']}>{name || 'Create Metric'}</h1>
          <p className={styles['page-header__sub']}>
            {evalType && metricType
              ? `${evalType.toUpperCase()} · ${METRIC_TYPE_CARDS.find((c) => c.key === metricType)!.label}`
              : 'Build a metric step by step'}
          </p>
        </div>
        <div className={styles['page-header__meta']}>
          <Boxes size={13} />
          Step {stepIndex + 1} of {STEPS.length} · {currentStepLabel}
        </div>
      </div>

      <div className={styles.builder}>

        {/* ============ LEFT RAIL ============ */}
        <aside className={styles.rail}>
          <div className={styles['rail__head']}>
            <div className={styles['rail__eyebrow']}>Progress</div>
            <div className={styles['rail__sub']}>Complete each step to unlock the next.</div>
          </div>

          <nav className={styles['rail__steps']}>
            {STEPS.map((s, i) => {
              const active = step === s.key;
              const done = stepDone[s.key] && !active;
              return (
                <button
                  key={s.key}
                  disabled={!stepEnabled[s.key]}
                  onClick={() => goStep(s.key)}
                  className={`${styles['rail-step']} ${active ? styles['rail-step--active'] : ''} ${done ? styles['rail-step--done'] : ''} ${!stepEnabled[s.key] ? styles['rail-step--disabled'] : ''}`}
                >
                  <span className={styles['rail-step__marker']}>
                    {done ? <Check size={15} /> : i + 1}
                  </span>
                  <span className={styles['rail-step__body']}>
                    <span className={styles['rail-step__label']}>{s.label}</span>
                    <span className={styles['rail-step__value']}>{stepValue[s.key]}</span>
                  </span>
                </button>
              );
            })}
          </nav>
        </aside>

        {/* ============ RIGHT WORKSPACE ============ */}
        <section className={styles.work}>
          <div className={styles['work__scroll']}>
            <div className={styles['work__inner']} key={step}>

              {/* ---- STEP: DETAILS ---- */}
              {step === 'details' && (
                <>
                  <div className={styles['work__eyebrow']}>Step 1</div>
                  <h1 className={styles['work__title']}>Name your metric</h1>
                  <p className={styles['work__desc']}>Give it a clear name and, optionally, a short description of what it measures.</p>

                  <div className={styles.field}>
                    <label className={styles['field__label']}>Metric Name</label>
                    <input className={styles.input} placeholder="e.g., Answer Faithfulness" value={name} onChange={(e) => setName(e.target.value)} autoFocus />
                  </div>
                  <div className={styles.field}>
                    <label className={styles['field__label']}>Description</label>
                    <textarea className={styles.textarea} placeholder="What does this metric measure? (optional)" value={description} onChange={(e) => setDescription(e.target.value)} />
                  </div>
                </>
              )}

              {/* ---- STEP: TYPE & TARGET ---- */}
              {step === 'type' && (
                <>
                  <div className={styles['work__eyebrow']}>Step 2</div>
                  <h1 className={styles['work__title']}>Evaluation type &amp; approach</h1>
                  <p className={styles['work__desc']}>Choose what you\u2019re evaluating, then how the metric should score it.</p>

                  <div className={styles.field}>
                    <label className={styles['field__label']}>Evaluation Type</label>
                    <div className={`${styles['opt-grid']} ${styles['opt-grid--3']}`}>
                      {EVAL_TYPE_CARDS.map((c) => (
                        <button key={c.key} className={`${styles.opt} ${evalType === c.key ? styles['opt--selected'] : ''}`} onClick={() => handleEvalType(c.key)}>
                          {evalType === c.key && <span className={styles['opt__check']}><Check size={12} /></span>}
                          <span className={styles['opt__icon']}>{c.icon}</span>
                          <div className={styles['opt__title']}>{c.label}</div>
                          <div className={styles['opt__desc']}>{c.desc}</div>
                        </button>
                      ))}
                    </div>
                  </div>

                  <div className={styles.field}>
                    <label className={styles['field__label']}>Metric Type</label>
                    <div className={styles['opt-grid']}>
                      {METRIC_TYPE_CARDS.map((c) => (
                        <button key={c.key} className={`${styles.opt} ${metricType === c.key ? styles['opt--selected'] : ''}`} onClick={() => handleMetricType(c.key)}>
                          {metricType === c.key && <span className={styles['opt__check']}><Check size={12} /></span>}
                          <span className={styles['opt__icon']}>{c.icon}</span>
                          <div className={styles['opt__title']}>{c.label}</div>
                          <div className={styles['opt__desc']}>{c.desc}</div>
                        </button>
                      ))}
                    </div>
                  </div>
                </>
              )}

              {/* ---- STEP: CONFIG ---- */}
              {step === 'config' && (
                <>
                  <div className={styles['work__eyebrow']}>Step 3</div>
                  <h1 className={styles['work__title']}>{STEPS[2].label}</h1>

                  {/* visual */}
                  {metricType === 'visual' && (
                    <>
                      <p className={styles['work__desc']}>Build one or more field comparisons. Combine them with AND / OR.</p>
                      <div className={styles.rules}>
                        {rules.map((rule, i) => (
                          <div key={rule.id}>
                            {i > 0 && (
                              <div className={styles.gate}>
                                <div className={styles['gate__toggle']}>
                                  {(['AND', 'OR'] as const).map((g) => (
                                    <button key={g} className={`${styles['gate__opt']} ${gates[i - 1] === g ? styles.on : ''}`} onClick={() => toggleGate(i - 1)}>{g}</button>
                                  ))}
                                </div>
                              </div>
                            )}
                            <div className={styles.rule}>
                              <div className={styles['rule__index']}>{i + 1}</div>
                              <div className={styles['rule__grid']}>
                                <div className={styles['rule__field']}>
                                  <span className={styles['rule__field-label']}>Field</span>
                                  <CustomSelect value={rule.field} onChange={(v) => updateRule(rule.id, { field: v })} options={fields.map((f) => ({ value: f, label: f }))} />
                                </div>
                                <div className={styles['rule__field']}>
                                  <span className={styles['rule__field-label']}>Operator</span>
                                  <CustomSelect value={rule.operator} onChange={(v) => updateRule(rule.id, { operator: v })} options={OPERATORS} />
                                </div>
                                <div className={styles['rule__field']}>
                                  <span className={styles['rule__field-label']}>Compare To</span>
                                  <CustomSelect value={rule.compareType} onChange={(v) => updateRule(rule.id, { compareType: v as CompareType, value: '' })} options={[{ value: 'field', label: 'Field' }, { value: 'literal', label: 'Literal Value' }]} />
                                </div>
                                <div className={styles['rule__field']}>
                                  <span className={styles['rule__field-label']}>Value</span>
                                  {rule.compareType === 'literal'
                                    ? <input className={styles.input} placeholder="value" value={rule.value} onChange={(e) => updateRule(rule.id, { value: e.target.value })} />
                                    : <CustomSelect value={rule.value} onChange={(v) => updateRule(rule.id, { value: v })} placeholder="field…" options={fields.map((f) => ({ value: f, label: f }))} />}
                                </div>
                              </div>
                              <button className={styles['btn-icon']} title="Remove" onClick={() => removeRule(rule.id)}><X size={15} /></button>
                            </div>
                          </div>
                        ))}
                      </div>
                      <button className={`${styles.btn} ${styles['btn--sm']} ${styles['add-rule']}`} onClick={addRule}><Plus size={14} /> Add Rule</button>

                      <div className={styles.summary}>
                        <div className={styles['summary__label']}>Summary</div>
                        <div className={styles['summary__code']}>{ruleSummary || 'No rules defined'}</div>
                      </div>
                    </>
                  )}

                  {/* prompt */}
                  {metricType === 'prompt' && (
                    <>
                      <p className={styles['work__desc']}>Pick a judge prompt template (or write your own), then choose a judge model.</p>

                      {promptTemplatesError && <div className={`${styles.banner} ${styles['banner--err']}`}><AlertCircle size={15} /> {promptTemplatesError}</div>}
                      {promptTemplatesLoading ? (
                        <div className={styles.loading}><Loader2 size={15} className={styles.spin} /> Loading templates…</div>
                      ) : (
                        <div className={styles['tpl-list']}>
                          {matchingTemplates.length === 0 && !allowsCustomPrompt && <div className={styles.empty}>No templates for this evaluation type.</div>}
                          {matchingTemplates.map((t) => (
                            <label key={t.name} className={`${styles.tpl} ${selectedTemplateName === t.name ? styles['tpl--selected'] : ''}`}>
                              <input type="radio" name="tpl" hidden checked={selectedTemplateName === t.name} onChange={() => { setSelectedTemplateName(t.name); setPromptText(t.template); }} />
                              <span className={styles['tpl__radio']} />
                              <span>
                                <span className={styles['tpl__label']}>{t.label}</span>
                                <span className={styles['tpl__desc']}>{t.description}</span>
                                {t.uses_placeholders?.length > 0 && (
                                  <span className={styles['tpl__tags']}>
                                    {t.uses_placeholders.map((p) => <span key={p} className={styles.token}>{`{${p}}`}</span>)}
                                  </span>
                                )}
                              </span>
                            </label>
                          ))}
                          {allowsCustomPrompt && (
                            <label className={`${styles.tpl} ${selectedTemplateName === '__custom__' ? styles['tpl--selected'] : ''}`}>
                              <input type="radio" name="tpl" hidden checked={selectedTemplateName === '__custom__'} onChange={() => { setSelectedTemplateName('__custom__'); setPromptText(''); }} />
                              <span className={styles['tpl__radio']} />
                              <span>
                                <span className={styles['tpl__label']}>Custom Prompt</span>
                                <span className={styles['tpl__desc']}>Write your own judge prompt from scratch.</span>
                              </span>
                            </label>
                          )}
                        </div>
                      )}

                      {selectedTemplateName && (
                        <div className={styles.field}>
                          <label className={styles['field__label']}>Prompt</label>
                          <textarea className={styles.textarea} style={{ minHeight: '150px' }} value={promptText} onChange={(e) => setPromptText(e.target.value)} placeholder="Enter your judge prompt…" />
                        </div>
                      )}

                      <div className={styles.field}>
                        <label className={styles['field__label']}>Judge Model</label>
                        {modelsError && <div className={`${styles.banner} ${styles['banner--err']}`}><AlertCircle size={15} /> {modelsError}</div>}
                        {modelsLoading ? (
                          <div className={styles.loading}><Loader2 size={15} className={styles.spin} /> Loading models…</div>
                        ) : models.length === 0 ? (
                          <div className={styles.empty}>No models available.</div>
                        ) : (
                          <div className={styles.models}>
                            {models.map((m) => {
                              const health = modelHealth[m.id] || 'checking';
                              const disabled = health === 'unhealthy';
                              return (
                                <label key={m.id} className={`${styles.model} ${selectedModelId === m.id ? styles['model--selected'] : ''} ${disabled ? styles['model--disabled'] : ''}`}>
                                  <input type="radio" name="judge" hidden checked={selectedModelId === m.id} disabled={disabled} onChange={() => setSelectedModelId(m.id)} />
                                  <span className={styles['model__radio']} />
                                  <span>
                                    <span className={styles['model__name']}>{m.name}</span>
                                    <span className={styles['model__meta']}>{m.provider_id}</span>
                                  </span>
                                  <span className={`${styles['model__health']} ${styles[`health--${health}`]}`}>
                                    <span className={styles['health-dot']} />
                                    {health === 'checking' ? 'Checking' : health === 'healthy' ? 'Healthy' : 'Offline'}
                                  </span>
                                </label>
                              );
                            })}
                          </div>
                        )}
                      </div>
                    </>
                  )}

                  {/* code */}
                  {metricType === 'code' && (
                    <>
                      <p className={styles['work__desc']}>Starter code is tailored to the evaluation type. Edit it to suit your metric.</p>
                      {codeError && <div className={`${styles.banner} ${styles['banner--err']}`}><AlertCircle size={15} /> {codeError}</div>}
                      <div className={styles.code}>
                        <div className={styles['code__bar']}>
                          <span className={styles['code__lang']}>Python</span>
                          {codeLoading && <Loader2 size={13} className={styles.spin} />}
                        </div>
                        <textarea className={styles['code__area']} spellCheck={false} value={code} onChange={(e) => setCode(e.target.value)} placeholder="# scoring function" />
                      </div>
                    </>
                  )}

                  {/* simple */}
                  {metricType === 'simple' && (
                    <p className={styles['work__desc']}>The Simple metric needs no extra configuration. Set a threshold below and continue.</p>
                  )}

                  {/* threshold — shared across all config types */}
                  <div className={styles.field} style={{ marginTop: '26px' }}>
                    <label className={styles['field__label']}>Pass Threshold</label>
                    <div className={styles.thr}>
                      <div className={styles['thr__value']}>{threshold.toFixed(2)}</div>
                      <div className={styles['thr__cap']}>Minimum score required to pass</div>
                      <input type="range" className={styles['thr__slider']} min={0} max={1} step={0.01} value={threshold} onChange={(e) => setThreshold(Number(e.target.value))} />
                      <div className={styles['thr__scale']}><span>0.00</span><span>0.50</span><span>1.00</span></div>
                    </div>
                  </div>
                </>
              )}

              {/* ---- STEP: DATASET ---- */}
              {step === 'dataset' && (
                <>
                  <div className={styles['work__eyebrow']}>Step 4</div>
                  <h1 className={styles['work__title']}>Choose test data &amp; validate</h1>
                  <p className={styles['work__desc']}>Pick a dataset and questions, run validation, then save your metric.</p>

                  <div className={styles['data-row']}>
                    <div className={styles['data-col']}>
                      <div className={styles['data-col__head']}>
                        <span className={styles['data-col__head-title']}><Database size={12} /> Datasets</span>
                        {datasets.length > 0 && <span className={styles['data-col__count']}>{datasets.length}</span>}
                      </div>
                      <div className={styles['data-col__body']}>
                        {datasetsError ? <div className={`${styles.banner} ${styles['banner--err']}`}><AlertCircle size={15} /> {datasetsError}</div>
                          : datasetsLoading ? <div className={styles.loading}><Loader2 size={15} className={styles.spin} /> Loading…</div>
                          : datasets.length === 0 ? <div className={styles.empty}>No datasets for this type.</div>
                          : (
                            <div className={styles['ds-list']}>
                              {datasets.map((d) => (
                                <div key={d.id} className={`${styles.ds} ${selectedDatasetId === d.id ? styles['ds--selected'] : ''}`} onClick={() => selectDataset(d.id)}>
                                  <span className={styles['ds__radio']} />
                                  <span style={{ minWidth: 0 }}>
                                    <span className={styles['ds__name']}>{d.name}</span>
                                    <span className={styles['ds__count']}>{d.question_count} questions</span>
                                  </span>
                                </div>
                              ))}
                            </div>
                          )}
                      </div>
                    </div>

                    <div className={styles['data-col']}>
                      <div className={styles['data-col__head']}>
                        <span className={styles['data-col__head-title']}>
                          <ListChecks size={12} /> Questions
                        </span>
                        {previewQuestions.length > 0 && (
                          <span style={{ display: 'flex', alignItems: 'center', gap: '10px' }}>
                            <span className={styles['data-col__count']}>{selectedQuestionIds.size}/{previewQuestions.length}</span>
                            <span style={{ display: 'flex', gap: '8px' }}>
                              <button className={styles['link-btn']} onClick={selectAllQuestions}>All</button>
                              <button className={styles['link-btn']} onClick={clearAllQuestions}>Clear</button>
                            </span>
                          </span>
                        )}
                      </div>
                      <div className={styles['data-col__body']}>
                        {previewError ? <div className={`${styles.banner} ${styles['banner--err']}`}><AlertCircle size={15} /> {previewError}</div>
                          : previewLoading ? <div className={styles.loading}><Loader2 size={15} className={styles.spin} /> Loading…</div>
                          : previewQuestions.length === 0 ? <div className={styles.empty}>Select a dataset to preview.</div>
                          : (
                            <div className={styles['q-list']}>
                              {previewQuestions.map((q) => {
                                const on = selectedQuestionIds.has(q.id);
                                return (
                                  <div key={q.id} className={`${styles.q} ${on ? styles['q--on'] : ''}`} onClick={() => toggleQuestion(q.id)}>
                                    <span className={styles['q__check']}>{on && <Check size={12} />}</span>
                                    <span className={styles['q__body']}>
                                      <span className={styles['q__q']}>{q.input?.prompt}</span>
                                      <span className={styles['q__a']}><span className={styles['q__a-label']}>Expected:</span>{q.expected?.answer}</span>
                                    </span>
                                  </div>
                                );
                              })}
                            </div>
                          )}
                      </div>
                    </div>
                  </div>

                  {/* ---- validate & save (folded into the last step) ---- */}
                  <div className={styles['validate-section']}>
                    <div className={styles['validate-section__label']}>Validate &amp; Save</div>
                    <p className={styles['validate-section__desc']}>Run a dry-run against your selected questions. Saving unlocks once it passes.</p>

                    {validateError && <div className={`${styles.banner} ${styles['banner--err']}`}><AlertCircle size={15} /> {validateError}</div>}

                    {!validateResult && !validating && (
                      <div className={`${styles.banner} ${styles['banner--info']}`}><Sparkles size={15} /> Ready to validate {selectedQuestionIds.size} test case{selectedQuestionIds.size === 1 ? '' : 's'}.</div>
                    )}

                    {validateResult && (
                      <div style={{ marginBottom: '18px' }}>
                        {validateSucceeded
                          ? <div className={`${styles.banner} ${styles['banner--ok']}`}><CheckCircle2 size={15} /> Metric is valid — ready to save.</div>
                          : <div className={`${styles.banner} ${styles['banner--err']}`}><XCircle size={15} /> No test cases passed. You can still save, or adjust your metric and re-run.</div>}

                        <div className={styles.results}>
                          {validateResult.results.map((r, i) => (
                            <div key={i} className={styles['results__row']}>
                              <span className={`${styles['results__score']} ${r.success ? styles['results__score--pass'] : styles['results__score--fail']}`}>{r.score.toFixed(2)}</span>
                              <span className={styles['results__body']}>
                                <span className={styles['results__io']}>{r.test_case.input}</span>
                                {r.reason && <span className={styles['results__reason']}>{r.reason}</span>}
                              </span>
                              <span className={`${styles['results__pill']} ${r.success ? styles['results__pill--pass'] : styles['results__pill--fail']}`}>{r.success ? 'Pass' : 'Fail'}</span>
                            </div>
                          ))}
                          <div className={styles['results__summary']}>
                            <span>Passed: <strong>{validateResult.passed}/{validateResult.total}</strong></span>
                          </div>
                        </div>
                      </div>
                    )}
                  </div>
                </>
              )}
            </div>
          </div>

          {/* ---- footer nav ---- */}
          <div className={styles['work__foot']}>
            {step === 'details'
              ? <button className={`${styles.btn} ${styles['btn--ghost']}`} onClick={() => navigate('/app/custom-metrics/dashboard')}>Cancel</button>
              : <button className={styles.btn} onClick={prevStep}><ArrowLeft size={15} /> Back</button>}

            {step !== 'dataset' ? (
              <>
                <span className={styles['work__foot-info']} />
                <button className={`${styles.btn} ${styles['btn--primary']}`} onClick={nextStep} disabled={!stepDone[step]}>
                  Continue <ArrowRight size={15} />
                </button>
              </>
            ) : !validateResult ? (
              <>
                <span className={styles['work__foot-info']} />
                <button className={`${styles.btn} ${styles['btn--primary']}`} onClick={runValidate} disabled={validating || !canValidate}>
                  {validating ? <Loader2 size={15} className={styles.spin} /> : <Sparkles size={15} />}
                  {validating ? 'Validating…' : 'Run Validation'}
                </button>
              </>
            ) : (
              <>
                <span className={styles['work__foot-info']} />
                <button className={`${styles.btn} ${styles['btn--ok']}`} onClick={handleSave} disabled={saving}>
                  {saving ? <Loader2 size={15} className={styles.spin} /> : <Check size={15} />}
                  Save Metric
                </button>
              </>
            )}
          </div>
        </section>
      </div>

      {saveError && <div className={styles.toast}><AlertCircle size={15} /> {saveError}</div>}

      {savedId && (
        <div className={styles.overlay}>
          <div className={styles.modal}>
            <div className={styles['modal__icon']}><CheckCircle2 size={26} /></div>
            <div className={styles['modal__title']}>Metric created!</div>
            <div className={styles['modal__text']}>Your metric is now available for evaluations.</div>
            <div className={styles['modal__id']}>ID: {savedId}</div>
            <div className={styles['modal__actions']}>
              <button className={styles.btn} onClick={resetForm}>Create Another</button>
              <button className={`${styles.btn} ${styles['btn--primary']}`} onClick={() => navigate('/app/custom-metrics/dashboard')}>Go to Dashboard</button>
            </div>
          </div>
        </div>
      )}

      {ToastEl}
    </div>
  );
}
