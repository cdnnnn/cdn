//Createmetric.tsx
import { useEffect, useMemo, useRef, useState } from 'react';
import {
  AlertCircle, ArrowRight, Check, CheckCircle2, ChevronRight, Code2, Cpu, Database,
  ListChecks, Loader2, MessageSquare, Plus, Repeat, ScrollText, SlidersHorizontal, Sparkles,
  Target, TextSearch, Wrench, X, XCircle, Zap,
} from 'lucide-react';
import styles from './CreateMetric.module.scss';
import { useToast } from './useToast';
import CustomSelect from './CustomSelect';
import {
  metricsApi, AgentSubcategory, BuiltinCheckDef, EvalType, MetricType, PromptTemplate,
  ModelSummary, DatasetSummary, PreviewQuestion, ValidateMetricData, RuleDef,
} from '../../api/endpoints/metrics';

interface CreateMetricProps {
  onCancel: () => void;
  onSaved: (id: string) => void;
}

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
  { key: 'simple', label: 'Simple', desc: 'A built-in pass/fail check — no prompt or code needed.', icon: <Target size={18} /> },
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

// Agent-only: which part of the agent's behavior this metric evaluates —
// scopes both the Prompt Builder templates and the Code Editor starter
// code via a `subcategory` query param.
const AGENT_SUBCATEGORY_CARDS: { key: AgentSubcategory; label: string; desc: string; icon: JSX.Element }[] = [
  { key: 'tools', label: 'Tool Evaluation', desc: 'Score which tools the agent called and how.', icon: <Wrench size={18} /> },
  { key: 'answer', label: 'Answer Evaluation', desc: 'Score the agent\u2019s final response.', icon: <MessageSquare size={18} /> },
];

// ---- Simple metric type — Built-in Check icons --------------------------
// Built-in checks themselves now come from the API (GET /metrics/templates
// -> builtin_checks), since their id/params can vary server-side. Icons
// aren't part of that response, so map known ids to one and fall back to
// a generic icon for anything unrecognized.
const BUILTIN_CHECK_ICONS: Record<string, JSX.Element> = {
  contains_keywords: <TextSearch size={18} />,
  exact_match: <Target size={18} />,
  agent_loop_detection: <Repeat size={18} />,
  tool_correctness: <Wrench size={18} />,
};
const builtinCheckIcon = (id: string) => BUILTIN_CHECK_ICONS[id] || <ListChecks size={18} />;

type CompareType = 'field' | 'literal';
interface RuleRow { id: number; field: string; operator: string; compareType: CompareType; value: string; }
let ruleSeq = 1;

type SectionKey = 'details' | 'type' | 'config' | 'dataset';
interface SectionDef { key: SectionKey; label: string; }

export default function CreateMetric({ onCancel, onSaved }: CreateMetricProps) {
  const { showToast, ToastEl } = useToast();

  // section refs for the rail's "jump to" links
  const sectionRefs = {
    details: useRef<HTMLDivElement>(null),
    type: useRef<HTMLDivElement>(null),
    config: useRef<HTMLDivElement>(null),
    dataset: useRef<HTMLDivElement>(null),
  };
  const scrollToSection = (key: SectionKey) => {
    sectionRefs[key].current?.scrollIntoView({ behavior: 'smooth', block: 'start' });
  };

  // details
  const [name, setName] = useState('');
  const [description, setDescription] = useState('');

  // type
  const [evalType, setEvalType] = useState<EvalType | null>(null);
  const [metricType, setMetricType] = useState<MetricType | null>(null);
  // Agent-only sub-scope (Tool Evaluation / Answer Evaluation) — required
  // before Prompt Builder templates or Code Editor starter code can load
  // when evalType === 'agent'.
  const [agentSubcategory, setAgentSubcategory] = useState<AgentSubcategory | null>(null);

  // config: visual
  const [rules, setRules] = useState<RuleRow[]>([{ id: ruleSeq, field: 'actual_output', operator: 'contains', compareType: 'field', value: 'input' }]);
  const [gates, setGates] = useState<('AND' | 'OR')[]>([]);

  // templates data — GET /metrics/templates. Serves both the Prompt
  // Builder (templates + placeholders) and the Simple/Built-in Check
  // config (builtin_checks), since both live behind the same endpoint
  // and both depend on evalType (and, for agent, agentSubcategory).
  const [templates, setTemplates] = useState<PromptTemplate[]>([]);
  const [builtinChecks, setBuiltinChecks] = useState<BuiltinCheckDef[]>([]);
  const [templatesLoading, setTemplatesLoading] = useState(false);
  const [templatesError, setTemplatesError] = useState('');

  // config: prompt
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

  // config: simple — selected built-in check id + its params, keyed
  // dynamically off whatever `params` the API returned for that check
  // (no more hardcoded per-check fields).
  const [builtinCheck, setBuiltinCheck] = useState<string | null>(null);
  const [builtinParams, setBuiltinParams] = useState<Record<string, unknown>>({});

  // threshold (shared across all config types)
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
    setAgentSubcategory(null);
    setDatasets([]); setSelectedDatasetId(''); setPreviewQuestions([]); setSelectedQuestionIds(new Set());
    setCode(''); setPromptText(''); setSelectedTemplateName(''); setValidateResult(null); setSavedId('');
    setTemplates([]); setBuiltinChecks([]);
    // Built-in check availability depends on eval type (e.g. Agent Loop
    // Detection / Tool Correctness are agent-only) — clear the selection
    // so a now-unavailable check can't stay silently selected.
    setBuiltinCheck(null); setBuiltinParams({});
  };
  const handleAgentSubcategory = (s: AgentSubcategory) => {
    if (s === agentSubcategory) return;
    setAgentSubcategory(s);
    setCode(''); setPromptText(''); setSelectedTemplateName(''); setValidateResult(null); setSavedId('');
    setTemplates([]); setBuiltinChecks([]);
    setBuiltinCheck(null); setBuiltinParams({});
  };
  const handleMetricType = (t: MetricType) => {
    if (t === metricType) return;
    setMetricType(t); setValidateResult(null); setSavedId('');
    if (t !== 'simple') { setBuiltinCheck(null); setBuiltinParams({}); }
  };

  const handleBuiltinCheck = (check: BuiltinCheckDef) => {
    if (check.id === builtinCheck) return;
    setBuiltinCheck(check.id); setValidateResult(null); setSavedId('');
    // Seed params from each field's default_value so the form (and a
    // preview run without touching anything) starts from a sane state.
    const init: Record<string, unknown> = {};
    check.params.forEach((p) => {
      if (p.type === 'list' || p.type === 'string_list') {
        init[p.key] = Array.isArray(p.default_value) ? (p.default_value as string[]).join(', ') : (p.default_value ?? '');
      } else if (p.type === 'bool') {
        init[p.key] = Boolean(p.default_value);
      } else if (p.type === 'number') {
        init[p.key] = typeof p.default_value === 'number' ? p.default_value : 0;
      } else {
        init[p.key] = p.default_value ?? '';
      }
    });
    setBuiltinParams(init);
  };

  const availableBuiltinChecks = useMemo(
    () => (evalType ? builtinChecks.filter((c) => c.applicable_eval_types.includes(evalType)) : []),
    [evalType, builtinChecks],
  );
  const selectedBuiltinCheckDef = useMemo(
    () => availableBuiltinChecks.find((c) => c.id === builtinCheck) || null,
    [availableBuiltinChecks, builtinCheck],
  );

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

  // ---- templates (Prompt Builder templates + Simple built-in checks) ----
  // Both Prompt Builder and Simple need this same endpoint, scoped by
  // evalType and — for agent — by agentSubcategory. Waits for the
  // subcategory pick before fetching when evalType is 'agent'.
  useEffect(() => {
    if (!evalType) { setTemplates([]); setBuiltinChecks([]); return; }
    if (evalType === 'agent' && !agentSubcategory) { setTemplates([]); setBuiltinChecks([]); return; }
    setTemplatesLoading(true); setTemplatesError('');
    const scope = evalType === 'agent' && agentSubcategory ? { evalType, subcategory: agentSubcategory } : undefined;
    metricsApi.getPromptTemplates(scope)
      .then((res) => { setTemplates(res.templates); setBuiltinChecks(res.builtin_checks); })
      .catch((e) => setTemplatesError(e.message || 'Failed to load templates'))
      .finally(() => setTemplatesLoading(false));
  }, [evalType, agentSubcategory]);

  const matchingTemplates = useMemo(
    () => templates.filter((t) => t.category === (evalType ? EVAL_TYPE_TO_CATEGORY[evalType] : '')),
    [templates, evalType],
  );
  // Custom Prompt is available for every evaluation type — Model included,
  // same as Agent and RAG.
  const allowsCustomPrompt = evalType === 'agent' || evalType === 'rag' || evalType === 'model';

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
    if (metricType !== 'code' || !evalType) return;
    if (evalType === 'agent' && !agentSubcategory) return;
    setCodeLoading(true); setCodeError(''); setCode('');
    metricsApi.getCodeTemplate(evalType, evalType === 'agent' ? agentSubcategory ?? undefined : undefined)
      .then((res) => setCode(res.code))
      .catch((e) => setCodeError(e.message || 'Failed to load starter code'))
      .finally(() => setCodeLoading(false));
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [metricType, evalType, agentSubcategory]);

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

  // ---- gating (used for status dots + validate button, not for hiding UI) ---
  const detailsComplete = !!name.trim();
  const typeComplete = !!evalType && !!metricType && (evalType !== 'agent' || !!agentSubcategory);
  const configComplete = useMemo(() => {
    if (!metricType) return false;
    if (metricType === 'visual') return rules.every((r) => r.field && r.operator && (r.compareType === 'field' ? r.value : r.value.trim()));
    if (metricType === 'prompt') return !!promptText.trim() && !!selectedModelId;
    if (metricType === 'code') return !!code.trim();
    if (metricType === 'simple') {
      if (!selectedBuiltinCheckDef) return false;
      return selectedBuiltinCheckDef.params
        .filter((p) => p.required)
        .every((p) => {
          const v = builtinParams[p.key];
          if (p.type === 'bool') return v !== undefined;
          if (p.type === 'number') return typeof v === 'number' && !Number.isNaN(v) && v > 0;
          if (p.type === 'list' || p.type === 'string_list') return typeof v === 'string' && v.trim().length > 0;
          return typeof v === 'string' && v.trim().length > 0;
        });
    }
    return true;
  }, [metricType, rules, promptText, selectedModelId, code, selectedBuiltinCheckDef, builtinParams]);
  const datasetComplete = !!selectedDatasetId && selectedQuestionIds.size > 0;
  const canValidate = detailsComplete && typeComplete && configComplete && datasetComplete && threshold >= 0 && threshold <= 1;
  const validateSucceeded = !!validateResult && validateResult.passed > 0;

  const SECTIONS: SectionDef[] = [
    { key: 'details', label: 'Metric Details' },
    { key: 'type', label: 'Type & Target' },
    { key: 'config', label: metricType === 'prompt' ? 'Judge Prompt' : metricType === 'code' ? 'Scoring Code' : metricType === 'simple' ? 'Configuration' : 'Rules' },
    { key: 'dataset', label: 'Dataset · Validate & Save' },
  ];

  const sectionDone: Record<SectionKey, boolean> = {
    details: detailsComplete,
    type: typeComplete,
    config: configComplete,
    dataset: datasetComplete && !!validateResult,
  };

  const sectionValue: Record<SectionKey, string> = {
    details: name || 'Not set',
    type: evalType && metricType
      ? `${evalType.toUpperCase()}${evalType === 'agent' && agentSubcategory ? ` · ${agentSubcategory === 'tools' ? 'Tool Eval' : 'Answer Eval'}` : ''} · ${METRIC_TYPE_CARDS.find((c) => c.key === metricType)!.label}`
      : 'Not set',
    config: metricType ? (configComplete ? 'Configured' : 'Incomplete') : '—',
    dataset: validateResult ? `${validateResult.passed}/${validateResult.total} passed` : (selectedDatasetId ? `${selectedQuestionIds.size} selected` : 'Not set'),
  };

  // What's still missing for each incomplete section, surfaced in the rail
  // so the user knows exactly what to do next instead of just seeing
  // "Incomplete" / "Not set".
  const sectionMissing: Record<SectionKey, string> = {
    details: !name.trim() ? 'Add a metric name' : '',

    type: (() => {
      if (!evalType && !metricType) return 'Choose an evaluation type and a metric type';
      if (!evalType) return 'Choose an evaluation type';
      if (evalType === 'agent' && !agentSubcategory) return 'Choose Tool Evaluation or Answer Evaluation';
      if (!metricType) return 'Choose a metric type';
      return '';
    })(),

    config: (() => {
      if (!metricType) return 'Pick a metric type in the section above first';
      if (configComplete) return '';
      if (metricType === 'visual') return 'Fill in every rule\u2019s field, operator, and value';
      if (metricType === 'prompt') {
        if (!promptText.trim() && !selectedModelId) return 'Write a judge prompt and choose a judge model';
        if (!promptText.trim()) return 'Write a judge prompt';
        return 'Choose a judge model';
      }
      if (metricType === 'code') return 'Add your scoring code';
      if (metricType === 'simple') {
        if (!selectedBuiltinCheckDef) return 'Select a built-in check';
        const missingParam = selectedBuiltinCheckDef.params.find((p) => {
          const v = builtinParams[p.key];
          if (!p.required) return false;
          if (p.type === 'bool') return v === undefined;
          if (p.type === 'number') return !(typeof v === 'number' && v > 0);
          return !(typeof v === 'string' && v.trim().length > 0);
        });
        if (missingParam) return `Set ${missingParam.label}`;
      }
      return '';
    })(),

    dataset: (() => {
      if (!evalType) return 'Choose an evaluation type to load datasets';
      if (!selectedDatasetId) return 'Select a dataset';
      if (selectedQuestionIds.size === 0) return 'Select at least one test question';
      if (!validateResult) return 'Run validation to complete this step';
      return '';
    })(),
  };

  const completedCount = SECTIONS.filter((s) => sectionDone[s.key]).length;

  // ---- validate / save ---------------------------------------------------
  const buildDefinition = () => {
    if (metricType === 'visual') return { rules: rules.map<RuleDef>((r) => ({ field: r.field, operator: r.operator, value: r.value, compare_to_field: r.compareType === 'field' })) };
    if (metricType === 'prompt') return { prompt_template: promptText };
    if (metricType === 'code') return { code, skip_validation: true };
    if (metricType === 'simple') {
      if (!selectedBuiltinCheckDef) return {};
      // Convert each param to its API-facing value: list/string_list
      // params are edited as a comma-separated string but sent as an
      // array; number params sent as numbers; everything else as-is.
      const params: Record<string, unknown> = {};
      selectedBuiltinCheckDef.params.forEach((p) => {
        const raw = builtinParams[p.key];
        if (p.type === 'list' || p.type === 'string_list') {
          params[p.key] = typeof raw === 'string' ? raw.split(',').map((v) => v.trim()).filter(Boolean) : [];
        } else if (p.type === 'number') {
          params[p.key] = Number(raw);
        } else if (p.type === 'bool') {
          params[p.key] = Boolean(raw);
        } else {
          params[p.key] = raw;
        }
      });
      return { subtype: selectedBuiltinCheckDef.id, params };
    }
    return {};
  };

  const runValidate = () => {
    if (!canValidate || !evalType || !metricType) { showToast('Complete every section first', 'error'); return; }
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
    setName(''); setDescription(''); setEvalType(null); setMetricType(null); setAgentSubcategory(null);
    setRules([{ id: ++ruleSeq, field: 'actual_output', operator: 'contains', compareType: 'field', value: 'input' }]); setGates([]);
    setTemplates([]); setBuiltinChecks([]); setSelectedTemplateName(''); setPromptText('');
    setModels([]); setModelHealth({}); setSelectedModelId(''); setCode(''); setThreshold(0.7);
    setBuiltinCheck(null); setBuiltinParams({});
    setDatasets([]); setSelectedDatasetId(''); setPreviewQuestions([]); setSelectedQuestionIds(new Set());
    setValidateResult(null); setValidateError(''); setSavedId('');
    sectionRefs.details.current?.scrollIntoView({ behavior: 'smooth', block: 'start' });
  };

  // =========================================================================
  return (
    <div className={styles.cm}>

      <div className={styles.builder}>

        {/* ============ LEFT RAIL — jump-to links, all sections visible ============ */}
        <aside className={styles.rail}>
          <div className={styles['rail__head']}>
            <div className={styles['rail__eyebrow']}>Overview</div>
            <div className={styles['rail__sub']}>Everything is on this page — jump to any section.</div>
          </div>

          <nav className={styles['rail__steps']}>
            {SECTIONS.map((s, i) => {
              const done = sectionDone[s.key];
              return (
                <button
                  key={s.key}
                  onClick={() => scrollToSection(s.key)}
                  className={`${styles['rail-step']} ${done ? styles['rail-step--done'] : ''}`}
                >
                  <span className={styles['rail-step__marker']}>
                    {done ? <Check size={15} /> : i + 1}
                  </span>
                  <span className={styles['rail-step__body']}>
                    <span className={styles['rail-step__label']}>{s.label}</span>
                    <span className={styles['rail-step__value']}>{sectionValue[s.key]}</span>
                    {!done && sectionMissing[s.key] && (
                      <span className={styles['rail-step__missing']}>
                        <AlertCircle size={11} />
                        {sectionMissing[s.key]}
                      </span>
                    )}
                  </span>
                  <ChevronRight size={14} className={styles['rail-step__arrow']} />
                </button>
              );
            })}
          </nav>
        </aside>

        {/* ============ RIGHT WORKSPACE — all sections rendered together ============ */}
        <section className={styles.work}>
          <div className={styles['work__scroll']}>
            <div className={styles['work__inner']}>

              {/* ---- SECTION: DETAILS ---- */}
              <div className={styles.section} ref={sectionRefs.details}>
                <div className={styles['work__eyebrow']}>Section 1</div>
                <h1 className={styles['work__title']}>Name your metric</h1>
                <p className={styles['work__desc']}>Give it a clear name and, optionally, a short description of what it measures.</p>

                <div className={styles['field-row']}>
                  <div className={styles.field}>
                    <label className={styles['field__label']}>Metric Name</label>
                    <input className={styles.input} placeholder="e.g., Answer Faithfulness" value={name} onChange={(e) => setName(e.target.value)} />
                  </div>
                  <div className={styles.field}>
                    <label className={styles['field__label']}>Description</label>
                    <input className={styles.input} placeholder="What does this metric measure? (optional)" value={description} onChange={(e) => setDescription(e.target.value)} />
                  </div>
                </div>
              </div>

              {/* ---- SECTION: TYPE & TARGET ---- */}
              <div className={styles.section} ref={sectionRefs.type}>
                <div className={styles['work__eyebrow']}>Section 2</div>
                <h1 className={styles['work__title']}>Evaluation type &amp; approach</h1>
                <p className={styles['work__desc']}>Choose what you’re evaluating, then how the metric should score it.</p>

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

                {evalType === 'agent' && (
                  <div className={styles.field}>
                    <label className={styles['field__label']}>Agent Focus</label>
                    <div className={`${styles['opt-grid']} ${styles['opt-grid--3']}`}>
                      {AGENT_SUBCATEGORY_CARDS.map((c) => (
                        <button key={c.key} className={`${styles.opt} ${agentSubcategory === c.key ? styles['opt--selected'] : ''}`} onClick={() => handleAgentSubcategory(c.key)}>
                          {agentSubcategory === c.key && <span className={styles['opt__check']}><Check size={12} /></span>}
                          <span className={styles['opt__icon']}>{c.icon}</span>
                          <div className={styles['opt__title']}>{c.label}</div>
                          <div className={styles['opt__desc']}>{c.desc}</div>
                        </button>
                      ))}
                    </div>
                  </div>
                )}

                <div className={styles.field}>
                  <label className={styles['field__label']}>Metric Type</label>
                  <div className={`${styles['opt-grid']} ${styles['opt-grid--4']}`}>
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
              </div>

              {/* ---- SECTION: CONFIG ---- */}
              <div className={styles.section} ref={sectionRefs.config}>
                <div className={styles['work__eyebrow']}>Section 3</div>
                <h1 className={styles['work__title']}>{SECTIONS[2].label}</h1>

                {!metricType && (
                  <div className={styles.empty}>Pick a metric type above to configure it here.</div>
                )}

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
                            <div className={styles['rule__head']}>
                              <span className={styles['rule__index']}>Rule {i + 1}</span>
                              <button className={styles['btn-icon']} title="Remove" onClick={() => removeRule(rule.id)}><X size={15} /></button>
                            </div>
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

                    {templatesError && <div className={`${styles.banner} ${styles['banner--err']}`}><AlertCircle size={15} /> {templatesError}</div>}
                    {evalType === 'agent' && !agentSubcategory ? (
                      <div className={styles.empty}>Choose Tool Evaluation or Answer Evaluation above first.</div>
                    ) : templatesLoading ? (
                      <div className={styles.loading}><Loader2 size={15} className={styles.spin} /> Loading templates…</div>
                    ) : (
                      <div className={styles['tpl-list']}>
                        {matchingTemplates.length === 0 && !allowsCustomPrompt && <div className={styles.empty}>No templates for this evaluation type.</div>}
                        {matchingTemplates.map((t) => (
                          <label key={t.name} className={`${styles.tpl} ${selectedTemplateName === t.name ? styles['tpl--selected'] : ''}`}>
                            <input type="radio" name="tpl" hidden checked={selectedTemplateName === t.name} onChange={() => { setSelectedTemplateName(t.name); setPromptText(t.template); }} />
                            <span className={styles['tpl__radio']} />
                            <span className={styles['tpl__body']}>
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
                            <span className={styles['tpl__body']}>
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
                                <span className={styles['model__body']}>
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
                    {evalType === 'agent' && !agentSubcategory ? (
                      <div className={styles.empty}>Choose Tool Evaluation or Answer Evaluation above first.</div>
                    ) : (
                      <div className={styles.code}>
                        <div className={styles['code__bar']}>
                          <span className={styles['code__lang']}>Python</span>
                          {codeLoading && <Loader2 size={13} className={styles.spin} />}
                        </div>
                        <textarea className={styles['code__area']} spellCheck={false} value={code} onChange={(e) => setCode(e.target.value)} placeholder="# scoring function" />
                      </div>
                    )}
                  </>
                )}

                {/* simple — Built-in Check (checks + params come from the API) */}
                {metricType === 'simple' && (
                  <>
                    <p className={styles['work__desc']}>Pick a built-in check. Available checks depend on the evaluation type selected above.</p>

                    {templatesError && <div className={`${styles.banner} ${styles['banner--err']}`}><AlertCircle size={15} /> {templatesError}</div>}

                    {!evalType ? (
                      <div className={styles.empty}>Choose an evaluation type above to see available checks.</div>
                    ) : evalType === 'agent' && !agentSubcategory ? (
                      <div className={styles.empty}>Choose Tool Evaluation or Answer Evaluation above first.</div>
                    ) : templatesLoading ? (
                      <div className={styles.loading}><Loader2 size={15} className={styles.spin} /> Loading checks…</div>
                    ) : availableBuiltinChecks.length === 0 ? (
                      <div className={styles.empty}>No built-in checks for this evaluation type.</div>
                    ) : (
                      <div className={`${styles['opt-grid']} ${availableBuiltinChecks.length >= 4 ? styles['opt-grid--4'] : ''}`}>
                        {availableBuiltinChecks.map((c) => (
                          <button
                            key={c.id}
                            className={`${styles.opt} ${builtinCheck === c.id ? styles['opt--selected'] : ''}`}
                            onClick={() => handleBuiltinCheck(c)}
                          >
                            {builtinCheck === c.id && <span className={styles['opt__check']}><Check size={12} /></span>}
                            <span className={styles['opt__icon']}>{builtinCheckIcon(c.id)}</span>
                            <div className={styles['opt__title']}>{c.name}</div>
                            <div className={styles['opt__desc']}>{c.description}</div>
                          </button>
                        ))}
                      </div>
                    )}

                    {selectedBuiltinCheckDef?.params.map((p) => (
                      <div key={p.key} className={`${styles.field} ${styles['field--fit']}`} style={{ marginTop: '18px' }}>
                        {p.type === 'bool' ? (
                          <div className={styles['switch-row']}>
                            <div>
                              <div className={styles['switch-row__label']}>{p.label}</div>
                            </div>
                            <button
                              type="button"
                              role="switch"
                              aria-checked={Boolean(builtinParams[p.key])}
                              className={`${styles.switch} ${builtinParams[p.key] ? styles['switch--on'] : ''}`}
                              onClick={() => setBuiltinParams((prev) => ({ ...prev, [p.key]: !prev[p.key] }))}
                            >
                              <span className={styles['switch__thumb']} />
                            </button>
                          </div>
                        ) : p.type === 'number' ? (
                          <>
                            <label className={styles['field__label']}>{p.label}</label>
                            <input
                              type="number"
                              min={1}
                              step={1}
                              className={styles.input}
                              value={typeof builtinParams[p.key] === 'number' ? (builtinParams[p.key] as number) : ''}
                              onChange={(e) => {
                                const raw = e.target.value;
                                if (raw === '') { setBuiltinParams((prev) => ({ ...prev, [p.key]: '' })); return; }
                                const n = Math.floor(Number(raw));
                                setBuiltinParams((prev) => ({ ...prev, [p.key]: Number.isFinite(n) && n > 0 ? n : 1 }));
                              }}
                            />
                          </>
                        ) : p.type === 'list' || p.type === 'string_list' ? (
                          <>
                            <label className={styles['field__label']}>{p.label} (comma-separated)</label>
                            <input
                              className={styles.input}
                              placeholder="Enter one or more values, separated by commas"
                              value={typeof builtinParams[p.key] === 'string' ? (builtinParams[p.key] as string) : ''}
                              onChange={(e) => setBuiltinParams((prev) => ({ ...prev, [p.key]: e.target.value }))}
                            />
                          </>
                        ) : (
                          <>
                            <label className={styles['field__label']}>{p.label}</label>
                            <input
                              className={styles.input}
                              value={typeof builtinParams[p.key] === 'string' ? (builtinParams[p.key] as string) : ''}
                              onChange={(e) => setBuiltinParams((prev) => ({ ...prev, [p.key]: e.target.value }))}
                            />
                          </>
                        )}
                      </div>
                    ))}
                  </>
                )}

                {/* threshold — shared across all config types */}
                {metricType && (
                  <div className={styles.field} style={{ marginTop: '26px' }}>
                    <label className={styles['field__label']}>Pass Threshold</label>
                    <div className={`${styles.thr} ${styles['field--fit']}`}>
                      <div className={styles['thr__row']}>
                        <span className={styles['thr__cap']}>Minimum score required to pass</span>
                        <span className={styles['thr__value']}>{threshold.toFixed(2)}</span>
                      </div>
                      <input type="range" className={styles['thr__slider']} min={0} max={1} step={0.01} value={threshold} onChange={(e) => setThreshold(Number(e.target.value))} />
                      <div className={styles['thr__scale']}><span>0.00</span><span>0.50</span><span>1.00</span></div>
                    </div>
                  </div>
                )}
              </div>

              {/* ---- SECTION: DATASET ---- */}
              <div className={`${styles.section} ${styles['section--last']}`} ref={sectionRefs.dataset}>
                <div className={styles['work__eyebrow']}>Section 4</div>
                <h1 className={styles['work__title']}>Choose test data &amp; validate</h1>
                <p className={styles['work__desc']}>Pick a dataset and questions, run validation, then save your metric.</p>

                {!evalType ? (
                  <div className={styles.empty}>Choose an evaluation type above to load datasets.</div>
                ) : (
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
                              {datasets.map((d) => {
                                const selected = selectedDatasetId === d.id;
                                return (
                                  <div
                                    key={d.id}
                                    className={`${styles.ds} ${selected ? styles['ds--selected'] : ''}`}
                                    onClick={() => selectDataset(d.id)}
                                    title={d.name}
                                  >
                                    <span className={styles['ds__check']}><Check size={11} /></span>
                                    <span className={styles['ds__icon']}><Database size={14} /></span>
                                    <span className={styles['ds__name']}>{d.name}</span>
                                    <span className={styles['ds__count']}>{d.question_count} {d.question_count === 1 ? 'question' : 'questions'}</span>
                                  </div>
                                );
                              })}
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
                )}

                {/* ---- validate & save ---- */}
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
              </div>

            </div>
          </div>

          {/* ---- sticky footer ---- */}
          <div className={styles['work__foot']}>
            <span className={styles['work__foot-info']}>
              {completedCount}/{SECTIONS.length} sections ready
            </span>

            <div className={styles['work__foot-actions']}>
              {!validateResult ? (
                <>
                  <button className={`${styles.btn} ${styles['btn--primary']}`} onClick={runValidate} disabled={validating || !canValidate}>
                    {validating ? <Loader2 size={15} className={styles.spin} /> : <Sparkles size={15} />}
                    {validating ? 'Validating…' : 'Run Validation'}
                    {!validating && <ArrowRight size={15} />}
                  </button>
                  <button className={`${styles.btn} ${styles['btn--ghost']}`} onClick={onCancel}>Cancel</button>
                </>
              ) : (
                <>
                  <button className={`${styles.btn} ${styles['btn--ok']}`} onClick={handleSave} disabled={saving}>
                    {saving ? <Loader2 size={15} className={styles.spin} /> : <Check size={15} />}
                    Save Metric
                  </button>
                  <button className={`${styles.btn} ${styles['btn--ghost']}`} onClick={onCancel}>Cancel</button>
                </>
              )}
            </div>
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
              <button className={`${styles.btn} ${styles['btn--primary']}`} onClick={() => onSaved(savedId)}>Go to Dashboard</button>
            </div>
          </div>
        </div>
      )}

      {ToastEl}
    </div>
  );
}




















//Metrics.ts
import api from '../axiosInstance';

// ---- Evaluation type & metric type (client-side only, no API) -----------
export type EvalType = 'model' | 'agent' | 'rag';
export type MetricType = 'visual' | 'prompt' | 'code' | 'simple';

// ---- Prompt Builder — GET /metrics/templates -----------------------------
export interface PromptTemplate {
  category: string; // "llm" | "agent" | "rag"
  description: string;
  label: string;
  name: string;
  template: string;
  uses_placeholders: string[];
}

// A reusable {placeholder} the judge prompt can reference — returned
// alongside templates so the Prompt Builder can show what each token means.
export interface PromptPlaceholder {
  name: string;
  label: string;
  description: string;
  syntax: string; // e.g. "{input}"
  category: string;
}

// One configurable parameter of a Built-in Check (Simple metric type),
// rendered as a form field whose input type is driven by `type`.
export interface BuiltinCheckParam {
  key: string;
  label: string;
  type: 'bool' | 'number' | 'string' | 'list' | 'string_list';
  default_value: unknown;
  required: boolean;
}

export interface BuiltinCheckDef {
  id: string; // e.g. "contains_keywords" — used as definition.subtype
  name: string;
  description: string;
  applicable_eval_types: EvalType[];
  params: BuiltinCheckParam[];
}

export interface TemplatesResponse {
  templates: PromptTemplate[];
  placeholders: PromptPlaceholder[];
  builtin_checks: BuiltinCheckDef[];
}

// Agent-only sub-scoping for both /metrics/templates and
// /metrics/code-templates/agent — Tool Evaluation vs Answer Evaluation.
export type AgentSubcategory = 'tools' | 'answer';

// ---- Code Editor — GET /metrics/code-templates/{eval_type} ----------------
export interface CodeTemplateData {
  eval_type: string;
  code: string;
}

// ---- Judge model (Prompt Builder) — GET /models ---------------------------
export interface ModelSummary {
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
  base_url: string;
}

export interface ModelHealthData {
  success: boolean;
  message: string;
  model_id: string;
  response: string;
}

// ---- Datasets ---------------------------------------------------------
export interface DatasetSummary {
  id: string;
  name: string;
  question_count: number;
}

export interface PreviewQuestion {
  id: string;
  input: { prompt: string };
  expected: { answer: string };
}

export interface DatasetPreviewData {
  dataset_id: string;
  questions: PreviewQuestion[];
}

// ---- Validate (dry run) — POST /metrics/custom/preview --------------------
export interface RuleDef {
  field: string;
  operator: string;
  value: string;
  compare_to_field: boolean;
}

export interface MetricDefinition {
  rules?: RuleDef[];
  // NB: the spec's own example literally spells this "prompt_tenplate" —
  // treating that as a typo and using the correct spelling here.
  prompt_template?: string;
  code?: string;
  skip_validation?: boolean;
  // Simple metric type — Built-in Check (contains_keyword / exact_match /
  // agent_loop_detection / tool_correctness).
  subtype?: string;
  params?: Record<string, unknown>;
}

export interface TestCasePayload {
  input: string;
  actual_output: string;
  expected_output: string;
  context: string[];
  retrieval_context: string[];
  tools_called: string[];
  expected_tools: string[];
}

export interface JudgeConfig {
  model_id: string;
}

export interface ValidateMetricRequest {
  actual_output: string;
  context: string[];
  definition: MetricDefinition;
  description: string;
  eval_types: EvalType[];
  expected_output: string;
  expected_tools: string[];
  gates: string[];
  input: string;
  judge_config: JudgeConfig | null;
  metric_type: string; // "condition" | "prompt" | "code" | "simple"
  name: string;
  retrieval_context: string[];
  test_cases: TestCasePayload[];
  threshold: string; // sent as a string, e.g. "0.70"
  tools_called: string[];
}

export interface ValidateResultItem {
  score: number;
  reason: string;
  success: boolean;
  test_case: TestCasePayload;
}

export interface ValidateMetricData {
  results: ValidateResultItem[];
  total: number;
  passed: number;
}

// ---- Save — POST /metrics/custom -------------------------------------
export interface SaveMetricRequest {
  definition: MetricDefinition;
  description: string;
  eval_types: EvalType[];
  metric_type: string;
  name: string;
  threshold: string;
  // Not shown in the spec's request sample, but included defensively since
  // Prompt Builder metrics can't be scored without a judge model — drop
  // this if the backend rejects the extra field.
  judge_config?: JudgeConfig | null;
}

export interface SaveMetricData {
  id?: string;
  name?: string;
}

// ---- Delete — DELETE /metrics/custom/{metric_id} --------------------------
export interface DeleteMetricData {
  status: string;
  metric_id: string;
}

// ---- Dashboard: saved custom metrics ---------------------------------
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
  metric_type: string;
  eval_types: string[];
  definition: CustomMetricDefinition;
  requires_judge: boolean;
  threshold: number;
  is_active: boolean;
  created_by_id: number;
  created_at: string;
  updated_at: string;
}

// None of these endpoints wrap their body in a { status, data } envelope —
// every response below is the payload itself, so each call just unwraps
// axios's own `r.data` and normalizes array fields to [] where the backend
// might omit them.
export const metricsApi = {
  // Dashboard — GET /metrics/custom -> { metrics: [...] }
  list: () =>
    api.get<{ metrics: CustomMetric[] }>('/metrics/custom').then((r) => r.data.metrics || []),

  // Prompt Builder + Simple/Built-in Check — GET /metrics/templates ->
  // { templates, placeholders, builtin_checks }. Agent eval type further
  // scopes the response by subcategory (Tool Evaluation vs Answer
  // Evaluation) via query params instead of the plain unscoped call.
  getPromptTemplates: (scope?: { evalType: EvalType; subcategory: AgentSubcategory }) =>
    api
      .get<TemplatesResponse>('/metrics/templates', {
        params: scope ? { eval_type: scope.evalType, subcategory: scope.subcategory } : undefined,
      })
      .then((r) => ({
        templates: r.data.templates || [],
        placeholders: r.data.placeholders || [],
        builtin_checks: r.data.builtin_checks || [],
      })),

  // Code Editor — GET /metrics/code-templates/{eval_type}, same agent
  // subcategory scoping as getPromptTemplates above.
  getCodeTemplate: (evalType: EvalType, subcategory?: AgentSubcategory) =>
    api
      .get<CodeTemplateData>(`/metrics/code-templates/${evalType}`, {
        params: evalType === 'agent' && subcategory ? { subcategory } : undefined,
      })
      .then((r) => r.data),

  // Prompt Builder — GET /models
  listModels: () =>
    api.get<{ models: ModelSummary[] }>('/models').then((r) => r.data.models || []),

  // Prompt Builder — per-model health ping. Failures (network error, or a
  // body missing `success`) resolve to an "unreachable" fallback instead
  // of throwing, since an offline model is a normal UI state, not an
  // exceptional one.
  checkModelHealth: (modelId: string) =>
    api
      .get<ModelHealthData>(`/models/health/${modelId}`)
      .then((r) => ('success' in r.data ? r.data : { success: false, message: 'Unreachable', model_id: modelId, response: '' }))
      .catch(() => ({ success: false, message: 'Unreachable', model_id: modelId, response: '' })),

  // Dataset selection — GET /datasets?eval_type={evalType}
  listDatasets: (evalType: EvalType) =>
    api
      .get<{ total_count: number; datasets: DatasetSummary[] }>('/datasets', { params: { eval_type: evalType } })
      .then((r) => r.data.datasets || []),

  // GET /datasets/{dataset_id}/preview
  previewDataset: (datasetId: string) =>
    api.get<DatasetPreviewData>(`/datasets/${datasetId}/preview`).then((r) => ({
      ...r.data,
      questions: r.data.questions || [],
    })),

  // Footer "Validate Metric" — POST /metrics/custom/preview. Doesn't
  // persist anything; a successful response with results unlocks Save.
  validate: (payload: ValidateMetricRequest) =>
    api.post<ValidateMetricData>('/metrics/custom/preview', payload).then((r) => ({
      ...r.data,
      results: r.data.results || [],
    })),

  // "Save Metric" — POST /metrics/custom. Response body beyond "200 OK"
  // isn't specified, so `id`/`name` are optional here.
  create: (payload: SaveMetricRequest) =>
    api.post<SaveMetricData | void>('/metrics/custom', payload).then((r) => r.data || {}),

  // Dashboard "Delete" — DELETE /metrics/custom/{metric_id} -> { status, metric_id }
  remove: (metricId: string) =>
    api.delete<DeleteMetricData>(`/metrics/custom/${metricId}`).then((r) => r.data),
};
