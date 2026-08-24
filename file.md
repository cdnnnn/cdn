//Createmetric.tsx
import { useEffect, useMemo, useState } from 'react';
import { useNavigate } from 'react-router-dom';
import {
  AlertCircle, ArrowLeft, ArrowRight, Check, CheckCircle2, Code2, Cpu, Loader2,
  Plus, ScrollText, SlidersHorizontal, Sparkles, Target, X, XCircle, Zap,
} from 'lucide-react';
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

type StepKey = 'details' | 'type' | 'config' | 'dataset' | 'validate';

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
    { key: 'dataset', label: 'Test Dataset' },
    { key: 'validate', label: 'Validate & Save' },
  ];

  const stepDone: Record<StepKey, boolean> = {
    details: detailsComplete,
    type: typeComplete,
    config: configComplete,
    dataset: datasetComplete,
    validate: validateSucceeded,
  };
  const stepEnabled: Record<StepKey, boolean> = {
    details: true,
    type: detailsComplete,
    config: typeComplete,
    dataset: typeComplete,
    validate: canValidate,
  };

  const stepValue: Record<StepKey, string> = {
    details: name || 'Not set',
    type: evalType && metricType ? `${evalType.toUpperCase()} · ${METRIC_TYPE_CARDS.find((c) => c.key === metricType)!.label}` : 'Not set',
    config: metricType ? (configComplete ? 'Configured' : 'Incomplete') : '—',
    dataset: selectedDatasetId ? `${selectedQuestionIds.size} selected` : 'Not set',
    validate: validateResult ? `${validateResult.passed}/${validateResult.total} passed` : 'Pending',
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
    if (!validateSucceeded || !evalType || !metricType) { showToast('Validate successfully before saving', 'error'); return; }
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

  // =========================================================================
  return (
    <div className={`page-enter ${styles.cm}`}>
      <div className={styles.builder}>

        {/* ============ LEFT RAIL ============ */}
        <aside className={styles.rail}>
          <div className={styles['rail__head']}>
            <div className={styles['rail__eyebrow']}>Custom Metric</div>
            <div className={styles['rail__title']}>{name || 'Untitled Metric'}</div>
            <div className={styles['rail__sub']}>
              {evalType && metricType
                ? `${evalType.toUpperCase()} · ${METRIC_TYPE_CARDS.find((c) => c.key === metricType)!.label}`
                : 'Build a metric step by step'}
            </div>
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
                              <div className={styles['rule__grid']}>
                                <CustomSelect value={rule.field} onChange={(v) => updateRule(rule.id, { field: v })} options={fields.map((f) => ({ value: f, label: f }))} />
                                <CustomSelect value={rule.operator} onChange={(v) => updateRule(rule.id, { operator: v })} options={OPERATORS} />
                                <CustomSelect value={rule.compareType} onChange={(v) => updateRule(rule.id, { compareType: v as CompareType, value: '' })} options={[{ value: 'field', label: 'Compared to Field' }, { value: 'literal', label: 'Literal Value' }]} />
                                {rule.compareType === 'literal'
                                  ? <input className={styles.input} placeholder="value" value={rule.value} onChange={(e) => updateRule(rule.id, { value: e.target.value })} />
                                  : <CustomSelect value={rule.value} onChange={(v) => updateRule(rule.id, { value: v })} placeholder="field…" options={fields.map((f) => ({ value: f, label: f }))} />}
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
                  <h1 className={styles['work__title']}>Choose test data</h1>
                  <p className={styles['work__desc']}>Pick a dataset, then select the questions to validate against.</p>

                  <div className={styles['data-row']}>
                    <div className={styles['data-col']}>
                      <div className={styles['data-col__head']}>Datasets</div>
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

                    <div className={styles['data-col']}>
                      <div className={styles['data-col__head']}>
                        <span>Questions {previewQuestions.length > 0 && `(${selectedQuestionIds.size}/${previewQuestions.length})`}</span>
                        {previewQuestions.length > 0 && (
                          <span style={{ display: 'flex', gap: '8px' }}>
                            <button className={styles['link-btn']} onClick={selectAllQuestions}>All</button>
                            <button className={styles['link-btn']} onClick={clearAllQuestions}>Clear</button>
                          </span>
                        )}
                      </div>
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
                </>
              )}

              {/* ---- STEP: VALIDATE ---- */}
              {step === 'validate' && (
                <>
                  <div className={styles['work__eyebrow']}>Step 5</div>
                  <h1 className={styles['work__title']}>Validate &amp; save</h1>
                  <p className={styles['work__desc']}>Run a dry-run against your selected questions. Save unlocks once it passes.</p>

                  {validateError && <div className={`${styles.banner} ${styles['banner--err']}`}><AlertCircle size={15} /> {validateError}</div>}

                  {!validateResult && !validating && (
                    <div className={`${styles.banner} ${styles['banner--info']}`}><Sparkles size={15} /> Ready to validate {selectedQuestionIds.size} test case{selectedQuestionIds.size === 1 ? '' : 's'}.</div>
                  )}

                  <button className={`${styles.btn} ${styles['btn--primary']}`} onClick={runValidate} disabled={validating || !canValidate}>
                    {validating ? <Loader2 size={15} className={styles.spin} /> : <Sparkles size={15} />}
                    {validating ? 'Validating…' : 'Run Validation'}
                  </button>

                  {validateResult && (
                    <div style={{ marginTop: '22px' }}>
                      {validateSucceeded
                        ? <div className={`${styles.banner} ${styles['banner--ok']}`}><CheckCircle2 size={15} /> Metric is valid — ready to save.</div>
                        : <div className={`${styles.banner} ${styles['banner--err']}`}><XCircle size={15} /> No test cases passed. Adjust your metric and re-run.</div>}

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
                </>
              )}
            </div>
          </div>

          {/* ---- footer nav ---- */}
          <div className={styles['work__foot']}>
            {step === 'details'
              ? <button className={`${styles.btn} ${styles['btn--ghost']}`} onClick={() => navigate('/app/custom-metrics/dashboard')}>Cancel</button>
              : <button className={styles.btn} onClick={prevStep}><ArrowLeft size={15} /> Back</button>}

            {step !== 'validate' ? (
              <>
                <span className={styles['work__foot-info']} />
                <button className={`${styles.btn} ${styles['btn--primary']}`} onClick={nextStep} disabled={!stepDone[step]}>
                  Continue <ArrowRight size={15} />
                </button>
              </>
            ) : (
              <>
                <span className={styles['work__foot-info']} />
                <button className={`${styles.btn} ${styles['btn--ok']}`} onClick={handleSave} disabled={!validateSucceeded || saving}>
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















//Createmetric.module.scss
@use '../../styles/_variables' as *;

// ===========================================================================
// Create Metric — guided two-pane builder.
// Left: numbered progress rail that doubles as a live summary.
// Right: focused workspace showing one step at a time.
// Ink/paper design system, ultramarine signal accent, mono instrument labels.
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
$ink-solid: var(--ink-solid);

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

$soft: 0 1px 2px rgba(20, 22, 27, 0.05);
$lift: 0 18px 40px -20px rgba(20, 22, 27, 0.30);

$base-font: 0.875rem;

%micro {
  font-family: $mono;
  font-size: 0.6875rem;
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

@keyframes cm-spin { to { transform: rotate(360deg); } }
@keyframes cm-fade-up { from { opacity: 0; transform: translateY(8px); } to { opacity: 1; transform: translateY(0); } }
@keyframes cm-pop { 0% { transform: scale(0.7); opacity: 0; } 60% { transform: scale(1.08); } 100% { transform: scale(1); opacity: 1; } }
@keyframes cm-modal-in { from { opacity: 0; transform: translateY(12px) scale(0.98); } to { opacity: 1; transform: translateY(0) scale(1); } }

.spin { animation: cm-spin 0.8s linear infinite; }

// ---------------------------------------------------------------------------
// shell
// ---------------------------------------------------------------------------
.cm {
  font-size: $base-font;
  height: 100%;
  display: flex;
  flex-direction: column;
  min-height: 0;

  @media (min-width: 1800px) { font-size: 1rem; }
}

.builder {
  flex: 1;
  min-height: 0;
  display: grid;
  grid-template-columns: 300px 1fr;
  gap: 0;
  overflow: hidden;
}

// ---------------------------------------------------------------------------
// LEFT RAIL
// ---------------------------------------------------------------------------
.rail {
  display: flex;
  flex-direction: column;
  min-height: 0;
  background: $card;
  border-right: 1px solid $line;
  overflow-y: auto;
}

.rail__head {
  padding: 24px 22px 18px;
  border-bottom: 1px solid $line;
}

.rail__eyebrow {
  @extend %micro;
  font-size: 0.625rem;
  color: $signal;
  display: flex;
  align-items: center;
  gap: 7px;
  margin-bottom: 10px;

  &::before { content: ''; width: 14px; height: 2px; border-radius: 2px; background: $signal; }
}

.rail__title {
  font-family: $display;
  font-size: 1.1875rem;
  font-weight: 800;
  letter-spacing: -0.02em;
  color: $ink;
  line-height: 1.25;
  word-break: break-word;
}

.rail__sub {
  margin-top: 4px;
  font-size: 0.78125rem;
  color: $ink-3;
}

.rail__steps {
  flex: 1;
  padding: 12px;
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.rail-step {
  position: relative;
  display: flex;
  align-items: center;
  gap: 12px;
  width: 100%;
  text-align: left;
  padding: 11px 12px;
  border-radius: 11px;
  border: none;
  background: transparent;
  cursor: pointer;
  transition: background 0.15s ease;

  &:hover:not(&--disabled):not(&--active) { background: $paper; }

  &--active { background: $wash; }

  &--disabled { opacity: 0.4; cursor: not-allowed; }

  // connector line between markers
  &:not(:last-child)::after {
    content: '';
    position: absolute;
    left: 27px;
    top: 38px;
    bottom: -2px;
    width: 1.5px;
    background: $line;
  }
  &--done:not(:last-child)::after { background: $signal; }
}

.rail-step__marker {
  flex-shrink: 0;
  width: 30px;
  height: 30px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-family: $mono;
  font-size: 0.8125rem;
  font-weight: 700;
  color: $ink-3;
  background: $paper;
  border: 1.5px solid $line;
  z-index: 1;
  transition: all 0.15s ease;

  .rail-step--active & { color: #fff; background: $signal; border-color: $signal; }
  .rail-step--done & { color: #fff; background: $signal; border-color: $signal; }
}

.rail-step__body { min-width: 0; flex: 1; }

.rail-step__label {
  font-size: 0.8125rem;
  font-weight: 650;
  color: $ink-2;
  transition: color 0.15s ease;

  .rail-step--active & { color: $ink; font-weight: 700; }
  .rail-step--done & { color: $ink; }
}

.rail-step__value {
  font-family: $mono;
  font-size: 0.6875rem;
  color: $ink-3;
  margin-top: 2px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;

  .rail-step--active & { color: $signal; }
}

// ---------------------------------------------------------------------------
// RIGHT WORKSPACE
// ---------------------------------------------------------------------------
.work {
  display: flex;
  flex-direction: column;
  min-height: 0;
  background: $paper;
}

.work__scroll {
  flex: 1;
  min-height: 0;
  overflow-y: auto;
  padding: 32px 36px;
}

.work__inner {
  max-width: 760px;
  margin: 0 auto;
  animation: cm-fade-up 0.28s ease;
}

.work__eyebrow {
  @extend %micro;
  font-size: 0.625rem;
  color: $ink-3;
  margin-bottom: 8px;
}

.work__title {
  font-family: $display;
  font-size: 1.5rem;
  font-weight: 800;
  letter-spacing: -0.02em;
  color: $ink;
  line-height: 1.2;
}

.work__desc {
  margin-top: 6px;
  margin-bottom: 26px;
  font-size: 0.9375rem;
  color: $ink-2;
  line-height: 1.5;
}

// ---------------------------------------------------------------------------
// footer nav (per-step)
// ---------------------------------------------------------------------------
.work__foot {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  padding: 16px 36px;
  background: $card;
  border-top: 1px solid $line;
}

.work__foot-info {
  font-size: 0.78125rem;
  color: $ink-3;
  margin-right: auto;
}

// ---------------------------------------------------------------------------
// buttons
// ---------------------------------------------------------------------------
.btn {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  padding: 10px 18px;
  border-radius: 10px;
  border: 1px solid $line;
  background: $card;
  color: $ink-2;
  font-family: $sans;
  font-size: 0.84375rem;
  font-weight: 650;
  cursor: pointer;
  transition: all 0.15s ease;
  white-space: nowrap;

  &:hover:not(:disabled) { border-color: $ink-3; color: $ink; }
  &:disabled { opacity: 0.45; cursor: not-allowed; }
}

.btn--sm { padding: 7px 12px; font-size: 0.78125rem; border-radius: 8px; }

.btn--primary {
  border-color: $signal;
  background: $signal;
  color: #fff;
  &:hover:not(:disabled) { background: $signal-2; border-color: $signal-2; color: #fff; transform: translateY(-1px); box-shadow: $lift; }
}

.btn--ghost { background: transparent; border-color: transparent; &:hover:not(:disabled) { background: $paper; border-color: $line; } }

.btn--ok {
  border-color: $ok; background: $ok; color: #fff;
  &:hover:not(:disabled) { filter: brightness(0.95); color: #fff; transform: translateY(-1px); box-shadow: $lift; }
}

.btn-icon {
  display: inline-flex; align-items: center; justify-content: center;
  width: 30px; height: 30px; border-radius: 8px;
  border: 1px solid transparent; background: transparent; color: $ink-3; cursor: pointer;
  transition: all 0.15s ease;
  &:hover { background: $danger-wash; border-color: rgba($danger, 0.2); color: $danger; }
}

// ---------------------------------------------------------------------------
// forms
// ---------------------------------------------------------------------------
.field { margin-bottom: 20px; }

.field__label {
  display: block;
  @extend %micro;
  font-size: 0.6875rem;
  color: $ink-2;
  margin-bottom: 8px;
}
.field__hint { font-size: 0.78125rem; color: $ink-3; margin-top: 6px; }

.input, .textarea {
  width: 100%;
  border: 1.5px solid $line;
  border-radius: 10px;
  padding: 11px 13px;
  font-size: 0.9375rem;
  font-family: $sans;
  color: $ink;
  background: $card;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;

  &::placeholder { color: $ink-3; }
  &:focus { outline: none; border-color: $signal; box-shadow: 0 0 0 3px $wash; }
}
.textarea { resize: vertical; min-height: 92px; line-height: 1.55; }

// ---------------------------------------------------------------------------
// selectable option cards (eval type / metric type)
// ---------------------------------------------------------------------------
.opt-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 12px;
}
.opt-grid--3 { grid-template-columns: repeat(3, 1fr); }

.opt {
  position: relative;
  text-align: left;
  border: 1.5px solid $line;
  border-radius: 16px;
  padding: 18px;
  cursor: pointer;
  background: $card;
  transition: border-color 0.16s ease, box-shadow 0.16s ease, transform 0.16s ease, background 0.16s ease;

  &:hover:not(&--disabled) { border-color: $ink-3; transform: translateY(-2px); box-shadow: $lift; }

  &--selected {
    border-color: $signal;
    background: $wash;
    box-shadow: 0 0 0 1px $signal inset;
  }
  &--disabled { opacity: 0.5; cursor: not-allowed; }
}

.opt__icon {
  width: 42px;
  height: 42px;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: $paper;
  border: 1px solid $line;
  color: $signal;
  margin-bottom: 12px;
  transition: all 0.16s ease;

  .opt--selected & { background: $signal; border-color: $signal; color: #fff; }
}

.opt__title {
  font-family: $display;
  font-weight: 700;
  font-size: 1rem;
  color: $ink;
  margin-bottom: 4px;
}
.opt__desc { font-size: 0.8125rem; color: $ink-2; line-height: 1.45; }

.opt__check {
  position: absolute;
  top: 14px; right: 14px;
  width: 20px; height: 20px;
  border-radius: 50%;
  background: $signal;
  color: #fff;
  display: flex; align-items: center; justify-content: center;
  animation: cm-pop 0.22s ease;
}

// ---------------------------------------------------------------------------
// rule builder
// ---------------------------------------------------------------------------
.rules { display: flex; flex-direction: column; }

.rule {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 12px;
  border: 1.5px solid $line;
  border-radius: 12px;
  background: $card;
}

.rule__grid {
  flex: 1;
  display: grid;
  grid-template-columns: 1fr 1fr 1fr 1fr;
  gap: 8px;
  min-width: 0;
}

.gate {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 8px 0;

  &::before, &::after { content: ''; flex: 1; height: 1px; background: $line; }
}

.gate__toggle {
  display: inline-flex;
  padding: 2px;
  background: $card;
  border: 1px solid $line;
  border-radius: 8px;
  gap: 1px;
}

.gate__opt {
  padding: 4px 12px;
  border-radius: 6px;
  border: none;
  background: transparent;
  color: $ink-2;
  font-family: $mono;
  font-size: 0.6875rem;
  font-weight: 700;
  cursor: pointer;
  &.on { background: $signal; color: #fff; }
}

.add-rule { align-self: flex-start; margin-top: 12px; }

.summary {
  margin-top: 18px;
  padding: 16px;
  border-radius: 12px;
  background: $ink-solid;
  border: 1px solid $ink-solid;
}
.summary__label {
  @extend %micro;
  font-size: 0.5625rem;
  color: rgba(255, 255, 255, 0.55);
  margin-bottom: 8px;
}
.summary__code {
  font-family: $mono;
  font-size: 0.84375rem;
  color: #fff;
  line-height: 1.7;
  word-break: break-word;
}
.summary__token { color: #9db2ff; }
.summary__gate { color: $amber; font-weight: 700; padding: 0 4px; }

// ---------------------------------------------------------------------------
// prompt templates
// ---------------------------------------------------------------------------
.tpl-list { display: flex; flex-direction: column; gap: 8px; margin-bottom: 18px; }

.tpl {
  display: flex;
  gap: 12px;
  padding: 14px;
  border: 1.5px solid $line;
  border-radius: 12px;
  cursor: pointer;
  background: $card;
  transition: border-color 0.15s ease, background 0.15s ease;

  &:hover { border-color: $ink-3; }
  &--selected { border-color: $signal; background: $wash; }
}

.tpl__radio {
  flex-shrink: 0;
  width: 18px; height: 18px;
  margin-top: 1px;
  border-radius: 50%;
  border: 1.5px solid $line-2;
  display: flex; align-items: center; justify-content: center;

  .tpl--selected & { border-color: $signal; }
  &::after { content: ''; width: 9px; height: 9px; border-radius: 50%; background: $signal; opacity: 0; transition: opacity 0.15s ease; }
  .tpl--selected &::after { opacity: 1; }
}

.tpl__label { font-weight: 700; font-size: 0.9375rem; color: $ink; }
.tpl__desc { font-size: 0.8125rem; color: $ink-2; margin-top: 3px; line-height: 1.4; }
.tpl__tags { display: flex; flex-wrap: wrap; gap: 5px; margin-top: 8px; }

.token {
  font-family: $mono;
  font-size: 0.6875rem;
  font-weight: 700;
  color: $signal;
  background: $wash;
  border: 1px solid rgba($signal, 0.16);
  border-radius: 999px;
  padding: 3px 9px;
}

// ---------------------------------------------------------------------------
// judge model list
// ---------------------------------------------------------------------------
.models { display: flex; flex-direction: column; gap: 8px; }

.model {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 13px 15px;
  border: 1.5px solid $line;
  border-radius: 12px;
  cursor: pointer;
  background: $card;
  transition: border-color 0.15s ease, background 0.15s ease, opacity 0.15s ease;

  &:hover:not(&--disabled) { border-color: $ink-3; }
  &--selected { border-color: $signal; background: $wash; }
  &--disabled { opacity: 0.5; cursor: not-allowed; }
}

.model__radio {
  flex-shrink: 0;
  width: 18px; height: 18px;
  border-radius: 50%;
  border: 1.5px solid $line-2;
  display: flex; align-items: center; justify-content: center;
  .model--selected & { border-color: $signal; }
  &::after { content: ''; width: 9px; height: 9px; border-radius: 50%; background: $signal; opacity: 0; transition: opacity 0.15s ease; }
  .model--selected &::after { opacity: 1; }
}

.model__name { font-family: $display; font-weight: 700; font-size: 0.9375rem; color: $ink; }
.model__meta { font-family: $mono; font-size: 0.6875rem; color: $ink-3; margin-top: 1px; }

.model__health {
  margin-left: auto;
  display: inline-flex; align-items: center; gap: 6px;
  font-family: $mono; font-size: 0.625rem; font-weight: 700; text-transform: uppercase;
}
.health-dot { width: 7px; height: 7px; border-radius: 50%; flex-shrink: 0; }
.health--healthy { color: $ok; .health-dot { background: $ok; } }
.health--unhealthy { color: $danger; .health-dot { background: $danger; } }
.health--checking { color: $ink-3; .health-dot { background: $ink-3; animation: cm-spin 1s linear infinite; border-radius: 2px; } }

// ---------------------------------------------------------------------------
// code editor
// ---------------------------------------------------------------------------
.code {
  border: 1px solid $line;
  border-radius: 14px;
  overflow: hidden;
}
.code__bar {
  display: flex; align-items: center; justify-content: space-between;
  padding: 10px 14px;
  background: $ink-solid;
  color: rgba(255, 255, 255, 0.7);
}
.code__lang {
  @extend %micro;
  font-size: 0.625rem;
  color: #9db2ff;
}
.code__area {
  width: 100%;
  min-height: 340px;
  border: none;
  resize: vertical;
  padding: 16px;
  font-family: $mono;
  font-size: 0.8125rem;
  line-height: 1.65;
  color: $ink;
  background: $card;
  &:focus { outline: none; }
}

// ---------------------------------------------------------------------------
// threshold slider
// ---------------------------------------------------------------------------
.thr {
  padding: 24px;
  border: 1px solid $line;
  border-radius: 16px;
  background: $card;
}
.thr__value {
  font-family: $mono;
  font-size: 2.5rem;
  font-weight: 700;
  color: $signal;
  line-height: 1;
  text-align: center;
  margin-bottom: 4px;
}
.thr__cap { text-align: center; font-size: 0.78125rem; color: $ink-3; margin-bottom: 20px; }
.thr__slider {
  -webkit-appearance: none;
  appearance: none;
  width: 100%;
  height: 6px;
  border-radius: 999px;
  background: $line;
  outline: none;
  cursor: pointer;

  &::-webkit-slider-thumb {
    -webkit-appearance: none;
    appearance: none;
    width: 22px; height: 22px;
    border-radius: 50%;
    background: $signal;
    border: 3px solid $card;
    box-shadow: 0 2px 6px rgba(43, 43, 245, 0.4);
    cursor: pointer;
  }
  &::-moz-range-thumb {
    width: 22px; height: 22px;
    border-radius: 50%;
    background: $signal;
    border: 3px solid $card;
    cursor: pointer;
  }
}
.thr__scale {
  display: flex;
  justify-content: space-between;
  margin-top: 8px;
  font-family: $mono;
  font-size: 0.625rem;
  color: $ink-3;
}

// ---------------------------------------------------------------------------
// dataset + preview (side by side)
// ---------------------------------------------------------------------------
.data-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 20px;
  align-items: start;
}
.data-col { min-width: 0; }
.data-col__head {
  @extend %micro;
  font-size: 0.625rem;
  color: $ink-3;
  margin-bottom: 10px;
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.ds-list { display: flex; flex-direction: column; gap: 8px; max-height: 380px; overflow-y: auto; padding-right: 2px; }

.ds {
  display: flex; align-items: center; gap: 12px;
  padding: 13px 15px;
  border: 1.5px solid $line;
  border-radius: 12px;
  cursor: pointer;
  background: $card;
  transition: border-color 0.15s ease, background 0.15s ease;
  &:hover { border-color: $ink-3; }
  &--selected { border-color: $signal; background: $wash; }
}
.ds__radio {
  flex-shrink: 0; width: 18px; height: 18px; border-radius: 50%;
  border: 1.5px solid $line-2; display: flex; align-items: center; justify-content: center;
  .ds--selected & { border-color: $signal; }
  &::after { content: ''; width: 9px; height: 9px; border-radius: 50%; background: $signal; opacity: 0; transition: opacity 0.15s ease; }
  .ds--selected &::after { opacity: 1; }
}
.ds__name { font-family: $display; font-weight: 700; font-size: 0.875rem; color: $ink; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.ds__count { font-family: $mono; font-size: 0.6875rem; color: $ink-3; margin-top: 1px; }

.q-list { display: flex; flex-direction: column; gap: 8px; max-height: 380px; overflow-y: auto; padding-right: 2px; }

.q {
  display: flex; gap: 10px;
  padding: 12px 14px;
  border: 1px solid $line;
  border-radius: 12px;
  background: $card;
  cursor: pointer;
  transition: border-color 0.13s ease, background 0.13s ease;
  &:hover { border-color: $ink-3; background: $paper; }
}
.q__check {
  flex-shrink: 0; width: 17px; height: 17px; margin-top: 2px;
  border-radius: 5px; border: 1.5px solid $line-2;
  display: flex; align-items: center; justify-content: center;
  color: #fff;
  .q--on & { background: $signal; border-color: $signal; }
}
.q__body { min-width: 0; }
.q__q { font-size: 0.84375rem; color: $ink; font-weight: 600; margin-bottom: 3px; }
.q__a { font-size: 0.78125rem; color: $ink-2; }
.q__a-label { font-family: $mono; font-size: 0.625rem; color: $ink-3; margin-right: 5px; }

.link-btn {
  border: none; background: none; padding: 0;
  color: $signal; font-size: 0.71875rem; font-weight: 650; cursor: pointer;
  &:hover { text-decoration: underline; }
}

// ---------------------------------------------------------------------------
// validation results
// ---------------------------------------------------------------------------
.banner {
  display: flex; align-items: center; gap: 8px;
  padding: 12px 16px; border-radius: 12px;
  font-size: 0.84375rem; font-weight: 600;
  margin-bottom: 18px;
}
.banner--ok { background: $ok-wash; color: $ok; border: 1px solid rgba($ok, 0.2); }
.banner--err { background: $danger-wash; color: $danger; border: 1px solid rgba($danger, 0.2); }
.banner--info { background: $wash; color: $signal; border: 1px solid rgba($signal, 0.18); }

.results {
  border: 1px solid $line;
  border-radius: 14px;
  overflow: hidden;
}
.results__row {
  display: flex; align-items: flex-start; gap: 14px;
  padding: 14px 16px;
  border-bottom: 1px solid $line-2;
  &:last-child { border-bottom: none; }
}
.results__score {
  flex-shrink: 0;
  font-family: $mono; font-weight: 700; font-size: 1rem;
  width: 48px; text-align: center;
}
.results__score--pass { color: $ok; }
.results__score--fail { color: $danger; }
.results__body { min-width: 0; flex: 1; }
.results__io { font-size: 0.8125rem; color: $ink; }
.results__reason { font-size: 0.78125rem; color: $ink-2; margin-top: 4px; font-style: italic; }
.results__pill {
  flex-shrink: 0;
  @extend %micro;
  font-size: 0.5625rem;
  padding: 3px 9px; border-radius: 999px;
}
.results__pill--pass { color: $ok; background: $ok-wash; }
.results__pill--fail { color: $danger; background: $danger-wash; }
.results__summary {
  display: flex; gap: 20px;
  padding: 12px 16px;
  background: $paper;
  border-top: 1px solid $line;
  font-size: 0.84375rem; color: $ink-2;
  strong { color: $ink; font-family: $mono; }
}

// ---------------------------------------------------------------------------
// misc states
// ---------------------------------------------------------------------------
.loading, .empty {
  display: flex; align-items: center; justify-content: center; gap: 8px;
  padding: 28px; text-align: center;
  color: $ink-3; font-size: 0.84375rem;
  border: 1px dashed $line;
  border-radius: 12px;
}

// ---------------------------------------------------------------------------
// success modal
// ---------------------------------------------------------------------------
.overlay {
  position: fixed; inset: 0; z-index: 300;
  display: flex; align-items: center; justify-content: center;
  background: rgba(10, 12, 18, 0.55);
  padding: 20px;
}
.modal {
  width: 100%; max-width: 400px;
  background: $card;
  border-radius: 20px;
  box-shadow: $lift;
  padding: 32px 28px 24px;
  text-align: center;
  animation: cm-modal-in 0.24s ease;
}
.modal__icon {
  width: 56px; height: 56px; margin: 0 auto 16px;
  border-radius: 50%;
  background: $ok-wash; color: $ok;
  display: flex; align-items: center; justify-content: center;
  animation: cm-pop 0.3s ease;
}
.modal__title { font-family: $display; font-size: 1.25rem; font-weight: 800; color: $ink; margin-bottom: 6px; }
.modal__text { font-size: 0.875rem; color: $ink-2; margin-bottom: 8px; }
.modal__id {
  display: inline-block;
  font-family: $mono; font-size: 0.75rem; font-weight: 700;
  color: $signal; background: $wash;
  border-radius: 8px; padding: 4px 10px; margin-bottom: 22px;
}
.modal__actions { display: flex; gap: 10px; }
.modal__actions .btn { flex: 1; justify-content: center; }

// ---------------------------------------------------------------------------
// toast
// ---------------------------------------------------------------------------
.toast {
  position: fixed;
  left: 50%; bottom: 26px;
  transform: translateX(-50%);
  z-index: 320;
  display: flex; align-items: center; gap: 8px;
  padding: 12px 18px;
  border-radius: 12px;
  background: $ink-solid; color: #fff;
  font-size: 0.84375rem; font-weight: 600;
  box-shadow: $lift;
  animation: cm-fade-up 0.2s ease;
}

// ---------------------------------------------------------------------------
// responsive
// ---------------------------------------------------------------------------
@media (max-width: 1080px) {
  .builder { grid-template-columns: 1fr; }
  .rail {
    border-right: none;
    border-bottom: 1px solid $line;
    max-height: none;
  }
  .rail__steps { flex-direction: row; overflow-x: auto; }
  .rail-step { flex-direction: column; align-items: flex-start; min-width: 130px; }
  .rail-step:not(:last-child)::after { display: none; }
}

@media (max-width: 760px) {
  .work__scroll { padding: 22px 18px; }
  .work__foot { padding: 14px 18px; }
  .opt-grid, .opt-grid--3 { grid-template-columns: 1fr; }
  .data-row { grid-template-columns: 1fr; }
  .rule { flex-wrap: wrap; }
  .rule__grid { grid-template-columns: 1fr 1fr; }
}



















//Customselect.tsx
import { useEffect, useRef, useState } from 'react';
import { Check, ChevronDown } from 'lucide-react';
import styles from './CustomSelect.module.scss';

export interface DropdownOption {
  value: string;
  label: string;
  sublabel?: string;
}

interface CustomSelectProps {
  value: string;
  onChange: (value: string) => void;
  options: DropdownOption[];
  placeholder?: string;
  disabled?: boolean;
  className?: string;
}

export default function CustomSelect({ value, onChange, options, placeholder = 'Select…', disabled, className }: CustomSelectProps) {
  const [open, setOpen] = useState(false);
  const rootRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!open) return;
    const onClickOutside = (e: MouseEvent) => {
      if (rootRef.current && !rootRef.current.contains(e.target as Node)) setOpen(false);
    };
    const onEscape = (e: KeyboardEvent) => { if (e.key === 'Escape') setOpen(false); };
    document.addEventListener('mousedown', onClickOutside);
    document.addEventListener('keydown', onEscape);
    return () => {
      document.removeEventListener('mousedown', onClickOutside);
      document.removeEventListener('keydown', onEscape);
    };
  }, [open]);

  const selected = options.find((o) => o.value === value);

  return (
    <div ref={rootRef} className={`${styles.dropdown} ${className || ''} ${disabled ? styles['dropdown--disabled'] : ''}`}>
      <button
        type="button"
        className={`${styles['dropdown__trigger']} ${open ? styles['dropdown__trigger--open'] : ''}`}
        onClick={() => !disabled && setOpen((o) => !o)}
        disabled={disabled}
      >
        <span className={selected ? styles['dropdown__value'] : styles['dropdown__placeholder']}>
          {selected ? selected.label : placeholder}
        </span>
        <ChevronDown size={14} className={`${styles['dropdown__chevron']} ${open ? styles['dropdown__chevron--open'] : ''}`} />
      </button>

      {open && (
        <div className={styles['dropdown__menu']}>
          {options.length === 0 ? (
            <div className={styles['dropdown__empty']}>No options</div>
          ) : (
            options.map((opt) => (
              <button
                key={opt.value}
                type="button"
                className={`${styles['dropdown__option']} ${opt.value === value ? styles['dropdown__option--selected'] : ''}`}
                onClick={() => { onChange(opt.value); setOpen(false); }}
              >
                <span>
                  {opt.label}
                  {opt.sublabel && <span className={styles['dropdown__option-sub']}>{opt.sublabel}</span>}
                </span>
                {opt.value === value && <Check size={13} />}
              </button>
            ))
          )}
        </div>
      )}
    </div>
  );
}





















//Customselect.module.scss
@use '../../styles/_variables' as *;

// Self-contained styles for the CustomSelect dropdown so the component
// works on any page regardless of which parent stylesheet is loaded.

$ink:   var(--ink-1);
$ink-2: var(--ink-2);
$ink-3: var(--ink-3);
$paper: var(--paper);
$card:  var(--card);
$line:  var(--line);
$signal: #2B2BF5;
$wash:  var(--signal-wash);
$mono:  $font-mono;
$sans:  $font-body;
$lift:  0 18px 40px -20px rgba(20, 22, 27, 0.30);

.dropdown { position: relative; width: 100%; }

.dropdown__trigger {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  border: 1.5px solid $line;
  border-radius: 10px;
  padding: 10px 12px;
  font-size: 0.84375rem;
  font-family: $sans;
  color: $ink;
  background: $card;
  cursor: pointer;
  text-align: left;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;

  &:hover { border-color: $ink-3; }
  &--open { border-color: $signal; box-shadow: 0 0 0 3px $wash; }
}

.dropdown__value { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.dropdown__placeholder { color: $ink-3; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.dropdown__chevron { flex-shrink: 0; color: $ink-3; transition: transform 0.15s ease; &--open { transform: rotate(180deg); } }

.dropdown__menu {
  position: absolute;
  top: calc(100% + 6px);
  left: 0;
  min-width: 100%;
  width: max-content;
  max-width: 340px;
  z-index: 40;
  max-height: 280px;
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
  font-size: 0.84375rem;
  font-weight: 550;
  text-align: left;
  cursor: pointer;
  transition: background 0.12s ease, color 0.12s ease;

  &:hover { background: $paper; color: $ink; }
  &--selected { background: $wash; color: $signal; font-weight: 700; }
  svg { flex-shrink: 0; color: $signal; }
}

.dropdown__option-sub { display: block; font-family: $mono; font-size: 0.85em; font-weight: 500; color: $ink-3; margin-top: 1px; }
.dropdown__empty { padding: 12px; text-align: center; color: $ink-3; font-size: 0.78125rem; }
