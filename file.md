//Metricsstep.tsx
import { useMemo, type FC } from 'react';
import { Check, Loader2, AlertTriangle, Gavel, Sparkles } from 'lucide-react';
import type { EvalTypeId, ModelApi } from '../types';

interface Props {
  evalType: EvalTypeId | null;
  allMetrics: string[];
  customAgentMetrics: string[];
  selected: string[];
  onToggle: (id: string) => void;
  onSelectAll: (ids: string[]) => void;
  onClearAll: () => void;
  loading: boolean;
  error: string | null;
  models: ModelApi[];
  selectedModelIds: string[];
  judgeModelId: string | null;
  onJudgeModelChange: (id: string | null) => void;
}

const MetricsStep: FC<Props> = ({
  evalType,
  allMetrics,
  customAgentMetrics,
  selected,
  onToggle,
  onSelectAll,
  onClearAll,
  loading,
  error,
  models,
  selectedModelIds,
  judgeModelId,
  onJudgeModelChange,
}) => {
  const groups = useMemo(() => {
    const list = [{ label: 'All Metrics', items: allMetrics }];
    if (evalType === 'agent' && customAgentMetrics.length > 0) {
      list.push({ label: 'Custom Agent Metrics', items: customAgentMetrics });
    }
    return list;
  }, [allMetrics, customAgentMetrics, evalType]);

  const allApplicableMetrics = useMemo(() => groups.flatMap((g) => g.items), [groups]);
  const allSelected = allApplicableMetrics.length > 0 && allApplicableMetrics.every((m) => selected.includes(m));

  const eligibleJudgeModels = useMemo(
    () => models.filter((m) => selectedModelIds.includes(m.id)),
    [models, selectedModelIds]
  );

  return (
    <div className="run-eval__card run-eval__card--wide">
      <div className="run-eval__step-header-row">
        <div>
          <h2 className="run-eval__step-title">What to measure?</h2>
          <p className="run-eval__step-desc">
            Select the metrics that matter for your use case. Nothing is selected by default.
          </p>
        </div>
        <div className="run-eval__metrics-count">
          <span className="run-eval__metrics-count-num">{selected.length}</span> selected
          {allApplicableMetrics.length > 0 && (
            <span className="run-eval__metrics-bulk-actions">
              <button
                type="button"
                className="run-eval__metrics-toggle-all"
                onClick={() => onSelectAll(allApplicableMetrics)}
                disabled={allSelected}
              >
                Select all
              </button>
              <span className="run-eval__metrics-bulk-divider" />
              <button
                type="button"
                className="run-eval__metrics-toggle-all"
                onClick={onClearAll}
                disabled={selected.length === 0}
              >
                Unselect all
              </button>
            </span>
          )}
        </div>
      </div>

      {loading && (
        <div className="run-eval__loading-state">
          <Loader2 size={18} className="run-eval__spin" />
          Loading metrics…
        </div>
      )}

      {!loading && error && (
        <div className="run-eval__inline-error">
          <AlertTriangle size={15} />
          {error}
        </div>
      )}

      {!loading && !error && (
        <div className="run-eval__metrics-layout">
          <div className="run-eval__metrics-main">
            <div className="run-eval__metrics-main-scroll">
              {groups.map((group) => (
                <div className="run-eval__metric-group" key={group.label}>
                  <p className="run-eval__filter-title">{group.label}</p>
                  <div className="run-eval__metrics-grid">
                    {group.items.map((m) => {
                      const isSelected = selected.includes(m);
                      return (
                        <button
                          key={m}
                          type="button"
                          className={`run-eval__metric-card${isSelected ? ' run-eval__metric-card--selected' : ''}`}
                          onClick={() => onToggle(m)}
                        >
                          <span className="run-eval__metric-card-icon">
                            <Sparkles size={15} strokeWidth={2} />
                          </span>
                          <span className="run-eval__metric-name">{m}</span>
                          <span
                            className={`run-eval__metric-card-check${
                              isSelected ? ' run-eval__metric-card-check--on' : ''
                            }`}
                          >
                            <Check size={12} strokeWidth={3} />
                          </span>
                        </button>
                      );
                    })}
                    {group.items.length === 0 && <p className="run-eval__empty">No metrics available.</p>}
                  </div>
                </div>
              ))}
            </div>
          </div>

          <aside className="run-eval__judge-panel">
            <p className="run-eval__filter-title">
              <Gavel size={13} strokeWidth={2.25} /> Judge Model
            </p>
            <p className="run-eval__judge-hint">
              Pick one model from your selection to grade the other models' responses.
            </p>

            <div className="run-eval__judge-panel-scroll">
              {eligibleJudgeModels.length === 0 ? (
                <div className="run-eval__judge-empty">
                  <p>Select models in the previous step to choose a judge.</p>
                </div>
              ) : (
                <div className="run-eval__judge-list">
                  {eligibleJudgeModels.map((m) => {
                    const isJudge = judgeModelId === m.id;
                    return (
                      <button
                        key={m.id}
                        type="button"
                        className={`run-eval__judge-row${isJudge ? ' run-eval__judge-row--selected' : ''}`}
                        onClick={() => onJudgeModelChange(m.id)}
                        role="radio"
                        aria-checked={isJudge}
                      >
                        <span className={`run-eval__radio${isJudge ? ' run-eval__radio--checked' : ''}`} />
                        <span className="run-eval__judge-row-text">
                          <span className="run-eval__judge-row-name">{m.name}</span>
                          <span className="run-eval__judge-row-meta">{m.provider_id}</span>
                        </span>
                      </button>
                    );
                  })}
                </div>
              )}
            </div>
          </aside>
        </div>
      )}
    </div>
  );
};

export default MetricsStep;















//Runevaluation.tsx
import { useEffect, useState, type FC, type ComponentType } from 'react';
import { useNavigate } from 'react-router-dom';
import {
  ArrowLeft,
  ArrowRight,
  Play,
  Check,
  Clock3,
  Tag,
  LayoutGrid,
  Plug,
  Cpu,
  Database,
  Gauge,
  ClipboardCheck,
  Loader2,
} from 'lucide-react';
import NameStep from './steps/NameStep';
import TypeStep from './steps/TypeStep';
import ProvidersStep from './steps/ProvidersStep';
import ModelsStep from './steps/ModelsStep';
import DatasetStep from './steps/DatasetStep';
import MetricsStep from './steps/MetricsStep';
import ReviewStep from './steps/ReviewStep';
import { createEvaluation, startEvaluation as startEvaluationRequest } from './evaluationsApi';
import { fetchModels } from './modelsApi';
import { fetchProviders } from './providersApi';
import { fetchBenchmarks } from './benchmarksApi';
import { fetchMetrics } from './metricsApi';
import {
  WIZARD_STEPS,
  type Benchmark,
  type CreateEvaluationRequest,
  type EvaluationDraft,
  type ModelApi,
  type ProviderApi,
} from './types';
import './RunEvaluation.scss';

const EMPTY_DRAFT: EvaluationDraft = {
  name: '',
  type: null,
  providers: [],
  models: [],
  dataset: null,
  subgroup: [],
  metrics: [],
  judgeModelId: null,
  agentFramework: null,
};

const STEP_ICONS: ComponentType<{ size?: number }>[] = [
  Tag,
  LayoutGrid,
  Plug,
  Cpu,
  Database,
  Gauge,
  ClipboardCheck,
];

const RunEvaluation: FC = () => {
  const navigate = useNavigate();
  const [step, setStep] = useState(1);
  const [draft, setDraft] = useState<EvaluationDraft>(EMPTY_DRAFT);
  const [error, setError] = useState<string | null>(null);
  const [submitting, setSubmitting] = useState(false);
  const totalSteps = WIZARD_STEPS.length;

  const [providers, setProviders] = useState<ProviderApi[]>([]);
  const [models, setModels] = useState<ModelApi[]>([]);
  const [benchmarks, setBenchmarks] = useState<Benchmark[]>([]);
  const [allMetrics, setAllMetrics] = useState<string[]>([]);
  const [customAgentMetrics, setCustomAgentMetrics] = useState<string[]>([]);
  const [catalogLoading, setCatalogLoading] = useState(true);
  const [catalogError, setCatalogError] = useState<string | null>(null);

  useEffect(() => {
    let cancelled = false;

    const loadCatalog = async () => {
      setCatalogLoading(true);
      setCatalogError(null);
      try {
        const [providersRes, modelsRes, benchmarksRes, metricsRes] = await Promise.all([
          fetchProviders(),
          fetchModels(),
          fetchBenchmarks(),
          fetchMetrics(),
        ]);
        if (cancelled) return;
        setProviders(providersRes.providers);
        setModels(modelsRes.models);
        setBenchmarks(benchmarksRes.benchmarks);
        setAllMetrics(metricsRes.all_metrics);
        setCustomAgentMetrics(metricsRes.custom_agent_metrics);
      } catch (err) {
        if (cancelled) return;
        setCatalogError(
          err instanceof Error ? err.message : 'Failed to load providers, models, test suites, and metrics.'
        );
      } finally {
        if (!cancelled) setCatalogLoading(false);
      }
    };

    loadCatalog();
    return () => {
      cancelled = true;
    };
  }, []);

  const toggleInArray = (key: 'providers' | 'models' | 'metrics' | 'subgroup', id: string) => {
    setDraft((d) => {
      const arr = d[key];
      const next = arr.includes(id) ? arr.filter((x) => x !== id) : [...arr, id];
      return { ...d, [key]: next };
    });
  };

  const setType = (id: EvaluationDraft['type']) => {
    setDraft((d) => (d.type === id ? d : { ...d, type: id, metrics: [], agentFramework: id === 'agent' ? d.agentFramework : null }));
  };

  const setAgentFramework = (id: string | null) => {
    setDraft((d) => (d.agentFramework === id ? { ...d, agentFramework: null } : { ...d, agentFramework: id }));
  };

  const setJudgeModel = (id: string | null) => {
    setDraft((d) => (d.judgeModelId === id ? { ...d, judgeModelId: null } : { ...d, judgeModelId: id }));
  };

  const selectAllMetrics = (ids: string[]) => {
    setDraft((d) => ({ ...d, metrics: Array.from(new Set([...d.metrics, ...ids])) }));
  };

  const clearAllMetrics = () => {
    setDraft((d) => ({ ...d, metrics: [] }));
  };

  const validate = (): boolean => {
    setError(null);
    if (step === 1 && !draft.name.trim()) {
      setError('Enter an evaluation name to continue.');
      return false;
    }
    if (step === 2 && !draft.type) {
      setError('Select an evaluation type to continue.');
      return false;
    }
    if (step === 3 && draft.providers.length === 0) {
      setError('Select at least one provider to continue.');
      return false;
    }
    if (step === 4 && draft.models.length === 0) {
      setError('Select at least one model to continue.');
      return false;
    }
    if (step === 5 && !draft.dataset) {
      setError('Select a test suite to continue.');
      return false;
    }
    if (step === 6 && draft.metrics.length === 0) {
      setError('Select at least one metric to continue.');
      return false;
    }
    return true;
  };

  const goNext = () => {
    if (!validate()) return;
    setStep((s) => Math.min(totalSteps, s + 1));
  };
  const goBack = () => setStep((s) => Math.max(1, s - 1));
  const goToStep = (target: number) => {
    if (target < step) setStep(target);
  };

  const buildPayload = (): CreateEvaluationRequest => ({
    name: draft.name.trim(),
    eval_type: draft.type ?? '',
    dataset_id: '',
    benchmark: draft.dataset ?? '',
    selected_category: draft.subgroup.length > 0 ? draft.subgroup : undefined,
    model_ids: draft.models,
    selected_metrics: draft.metrics,
    dataset_limit: 1,
    judge_config: draft.judgeModelId
      ? {
          model_id: draft.judgeModelId,
          base_url: models.find((m) => m.id === draft.judgeModelId)?.base_url ?? '',
          // No API key is collected from the user anymore — the judge
          // model's own id is sent in this field instead.
          api_key: draft.judgeModelId,
        }
      : undefined,
  });

  const startEvaluation = async () => {
    if (!validate()) return;
    setError(null);
    setSubmitting(true);
    try {
      const created = await createEvaluation(buildPayload());
      const evaluationId = created.id ?? created.evaluation_id;
      if (!evaluationId) {
        throw new Error('The server did not return an evaluation id.');
      }
      await startEvaluationRequest(evaluationId);
      navigate('/app/history', { state: { evaluationId } });
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to start the evaluation. Please try again.');
    } finally {
      setSubmitting(false);
    }
  };

  const progressPct = Math.round(((step - 1) / (totalSteps - 1)) * 100);

  return (
    <div className="run-eval">
      <div className="run-eval__header">
        <div className="run-eval__header-left">
          <p className="run-eval__header-eyebrow">Create evaluation</p>
          <h1 className="run-eval__title">New Evaluation</h1>
          <p className="run-eval__subtitle">Compare AI models with standardized tests</p>
        </div>

        <div className="run-eval__header-meta">
          <Clock3 size={13} />
          ~5 min guided setup
        </div>
      </div>

      <div className="run-eval__wizard">
        <aside className="run-eval__sidebar">
          <div className="run-eval__sidebar-progress">
            <div className="run-eval__sidebar-progress-head">
              <span>
                Step {step} of {totalSteps}
              </span>
              <span>{progressPct}%</span>
            </div>
            <div className="run-eval__sidebar-progress-track">
              <div className="run-eval__sidebar-progress-fill" style={{ width: `${progressPct}%` }} />
            </div>
          </div>

          {WIZARD_STEPS.map((s, i) => {
            const num = i + 1;
            const state = num === step ? 'active' : num < step ? 'complete' : 'upcoming';
            const Icon = STEP_ICONS[i];
            return (
              <button
                key={s.key}
                type="button"
                className={`run-eval__step run-eval__step--${state}`}
                onClick={() => goToStep(num)}
                disabled={num > step}
              >
                <span className="run-eval__step-marker">
                  {state === 'complete' ? <Check size={14} strokeWidth={3} /> : <Icon size={15} />}
                </span>
                <span className="run-eval__step-text">
                  <span className="run-eval__step-label">{s.label}</span>
                  <span className="run-eval__step-desc">{s.description}</span>
                </span>
              </button>
            );
          })}
        </aside>

        <div className="run-eval__content">
          <p className="run-eval__step-kicker">
            Step {step} of {totalSteps}
          </p>

          <div className="run-eval__body">
            {step === 1 && <NameStep name={draft.name} onChange={(name) => setDraft((d) => ({ ...d, name }))} />}
            {step === 2 && (
              <TypeStep
                value={draft.type}
                onChange={setType}
                agentFramework={draft.agentFramework}
                onAgentFrameworkChange={setAgentFramework}
              />
            )}
            {step === 3 && (
              <ProvidersStep
                providers={providers}
                loading={catalogLoading}
                error={catalogError}
                selected={draft.providers}
                onToggle={(id) => toggleInArray('providers', id)}
                onGoToProviders={() => navigate('/app/providers')}
              />
            )}
            {step === 4 && (
              <ModelsStep
                models={models}
                providerCatalog={providers}
                selectedProviders={draft.providers}
                selected={draft.models}
                onToggle={(id) => toggleInArray('models', id)}
                onClear={() => setDraft((d) => ({ ...d, models: [] }))}
                loading={catalogLoading}
                error={catalogError}
              />
            )}
            {step === 5 && (
              <DatasetStep
                evalType={draft.type}
                benchmarks={benchmarks}
                loading={catalogLoading}
                error={catalogError}
                selected={draft.dataset}
                onSelect={(id) =>
                  setDraft((d) => (d.dataset === id ? d : { ...d, dataset: id, subgroup: [] }))
                }
                subgroup={draft.subgroup}
                onToggleSubgroup={(value) => toggleInArray('subgroup', value)}
              />
            )}
            {step === 6 && (
              <MetricsStep
                evalType={draft.type}
                allMetrics={allMetrics}
                customAgentMetrics={customAgentMetrics}
                selected={draft.metrics}
                onToggle={(id) => toggleInArray('metrics', id)}
                onSelectAll={selectAllMetrics}
                onClearAll={clearAllMetrics}
                loading={catalogLoading}
                error={catalogError}
                models={models}
                selectedModelIds={draft.models}
                judgeModelId={draft.judgeModelId}
                onJudgeModelChange={setJudgeModel}
              />
            )}
            {step === 7 && <ReviewStep draft={draft} models={models} benchmarks={benchmarks} providers={providers} />}

            {error && <p className="run-eval__error">{error}</p>}
          </div>

          <div className="run-eval__nav">
            {step > 1 ? (
              <button
                type="button"
                className="run-eval__btn run-eval__btn--secondary run-eval__btn--lg"
                onClick={goBack}
                disabled={submitting}
              >
                <ArrowLeft size={16} /> Back
              </button>
            ) : (
              <span />
            )}

            {step < totalSteps ? (
              <button type="button" className="run-eval__btn run-eval__btn--primary run-eval__btn--lg" onClick={goNext}>
                Continue <ArrowRight size={16} />
              </button>
            ) : (
              <button
                type="button"
                className="run-eval__btn run-eval__btn--primary run-eval__btn--lg"
                onClick={startEvaluation}
                disabled={submitting}
              >
                {submitting ? (
                  <>
                    <Loader2 size={16} className="run-eval__spin" /> Starting…
                  </>
                ) : (
                  <>
                    <Play size={16} /> Start Evaluation
                  </>
                )}
              </button>
            )}
          </div>
        </div>
      </div>
    </div>
  );
};

export default RunEvaluation;















//Reviewstep.tsx
import { useMemo, type FC } from 'react';
import {
  Info,
  Tag,
  Cpu,
  Database,
  Gauge,
  Gavel,
  Wallet,
  Clock3,
  Layers,
  CheckCircle2,
  Workflow,
} from 'lucide-react';
import { EVAL_TYPES } from '../data';
import type { Benchmark, EvaluationDraft, ModelApi, ProviderApi } from '../types';

interface Props {
  draft: EvaluationDraft;
  models: ModelApi[];
  benchmarks: Benchmark[];
  providers?: ProviderApi[];
}

const ReviewStep: FC<Props> = ({ draft, models, benchmarks, providers = [] }) => {
  const typeInfo = EVAL_TYPES.find((t) => t.id === draft.type);

  const providerNameById = useMemo(() => {
    const map = new Map<string, string>();
    providers.forEach((p) => map.set(p.id, p.name));
    return map;
  }, [providers]);

  const selectedModels = useMemo(
    () => draft.models.map((id) => models.find((m) => m.id === id)).filter((m): m is ModelApi => Boolean(m)),
    [draft.models, models]
  );

  const dataset = benchmarks.find((b) => b.name === draft.dataset);

  // NOTE: keeps compatibility whether `subgroup` is a single string or an array.
  const subgroupValues: string[] = Array.isArray(draft.subgroup)
    ? draft.subgroup
    : draft.subgroup
    ? [draft.subgroup as unknown as string]
    : [];
  const subgroupTasks = dataset?.tasks.filter((t) => subgroupValues.includes(t.value)) ?? [];

  const judgeModel = draft.judgeModelId ? models.find((m) => m.id === draft.judgeModelId) : null;

  const { cost, minutes } = useMemo(() => {
    const questions = dataset?.task_count ?? 0;
    const modelCount = draft.models.length || 1;
    const estCost = questions * modelCount * 0.0009;
    const estMinutes = Math.max(1, Math.round((questions * modelCount) / 180));
    return { cost: estCost, minutes: estMinutes };
  }, [dataset, draft.models.length]);

  return (
    <div className="run-eval__card">
      <h2 className="run-eval__step-title">Review &amp; Run</h2>
      <p className="run-eval__step-desc">Confirm your settings before starting.</p>

      {/* ---------- headline stats ---------- */}
      <div className="run-eval__review-stats">
        <div className="run-eval__review-stat">
          <span className="run-eval__review-stat-icon">
            <Wallet size={16} strokeWidth={2} />
          </span>
          <div>
            <span className="run-eval__review-stat-label">Est. Cost</span>
            <span className="run-eval__review-stat-value">~${cost.toFixed(2)}</span>
          </div>
        </div>
        <div className="run-eval__review-stat">
          <span className="run-eval__review-stat-icon">
            <Clock3 size={16} strokeWidth={2} />
          </span>
          <div>
            <span className="run-eval__review-stat-label">Est. Time</span>
            <span className="run-eval__review-stat-value">~{minutes} min</span>
          </div>
        </div>
        <div className="run-eval__review-stat">
          <span className="run-eval__review-stat-icon">
            <Layers size={16} strokeWidth={2} />
          </span>
          <div>
            <span className="run-eval__review-stat-label">Questions</span>
            <span className="run-eval__review-stat-value">{dataset ? dataset.task_count : '—'}</span>
          </div>
        </div>
        <div className="run-eval__review-stat">
          <span className="run-eval__review-stat-icon">
            <Cpu size={16} strokeWidth={2} />
          </span>
          <div>
            <span className="run-eval__review-stat-label">Models</span>
            <span className="run-eval__review-stat-value">{selectedModels.length}</span>
          </div>
        </div>
      </div>

      {/* ---------- overview ---------- */}
      <div className="run-eval__review-section">
        <p className="run-eval__filter-title">
          <Tag size={12} strokeWidth={2.25} /> Overview
        </p>
        <div className="run-eval__review">
          <div className="run-eval__review-row">
            <span>Name</span>
            <span>{draft.name || '—'}</span>
          </div>
          <div className="run-eval__review-row">
            <span>Type</span>
            <span>{typeInfo?.title ?? '—'}</span>
          </div>
          {draft.agentFramework && (
            <div className="run-eval__review-row">
              <span>
                <Workflow size={12} strokeWidth={2.25} style={{ marginRight: 5, verticalAlign: -2 }} />
                Agent Framework
              </span>
              <span>{draft.agentFramework}</span>
            </div>
          )}
        </div>
      </div>

      {/* ---------- models ---------- */}
      <div className="run-eval__review-section">
        <p className="run-eval__filter-title">
          <Cpu size={12} strokeWidth={2.25} /> Models ({selectedModels.length})
        </p>
        {selectedModels.length > 0 ? (
          <div className="run-eval__review-model-list">
            {selectedModels.map((m) => (
              <div className="run-eval__review-model-chip" key={m.id}>
                <span className="run-eval__review-model-chip-name">{m.name}</span>
                <span className="run-eval__review-model-chip-provider">
                  {providerNameById.get(m.provider_id) ?? m.provider_id}
                </span>
              </div>
            ))}
          </div>
        ) : (
          <p className="run-eval__filter-empty">No models selected.</p>
        )}
      </div>

      {/* ---------- test suite ---------- */}
      <div className="run-eval__review-section">
        <p className="run-eval__filter-title">
          <Database size={12} strokeWidth={2.25} /> Test Suite
        </p>
        <div className="run-eval__review">
          <div className="run-eval__review-row">
            <span>Suite</span>
            <span>{dataset?.name ?? '—'}</span>
          </div>
          {dataset?.description && (
            <div className="run-eval__review-row">
              <span>Description</span>
              <span>{dataset.description}</span>
            </div>
          )}
          {dataset?.huggingface_dataset && (
            <div className="run-eval__review-row">
              <span>Source</span>
              <span>{dataset.huggingface_dataset}</span>
            </div>
          )}
          {subgroupTasks.length > 0 && (
            <div className="run-eval__review-row">
              <span>Subgroup{subgroupTasks.length === 1 ? '' : 's'}</span>
              <span>{subgroupTasks.map((t) => t.name).join(', ')}</span>
            </div>
          )}
        </div>
        {dataset && dataset.required_capabilities.length > 0 && (
          <div className="run-eval__review-caps">
            {dataset.required_capabilities.map((c) => (
              <span key={c} className="run-eval__chip run-eval__chip--static">
                {c}
              </span>
            ))}
          </div>
        )}
      </div>

      {/* ---------- metrics ---------- */}
      <div className="run-eval__review-section">
        <p className="run-eval__filter-title">
          <Gauge size={12} strokeWidth={2.25} /> Metrics ({draft.metrics.length})
        </p>
        {draft.metrics.length > 0 ? (
          <div className="run-eval__review-caps">
            {draft.metrics.map((m) => (
              <span key={m} className="run-eval__review-metric-pill">
                <CheckCircle2 size={11} strokeWidth={2.5} />
                {m}
              </span>
            ))}
          </div>
        ) : (
          <p className="run-eval__filter-empty">No metrics selected.</p>
        )}
      </div>

      {/* ---------- judge model ---------- */}
      {judgeModel && (
        <div className="run-eval__review-section">
          <p className="run-eval__filter-title">
            <Gavel size={12} strokeWidth={2.25} /> Judge Model
          </p>
          <div className="run-eval__review">
            <div className="run-eval__review-row">
              <span>Model</span>
              <span>{judgeModel.name}</span>
            </div>
          </div>
        </div>
      )}

      <div className="run-eval__hint">
        <Info size={14} />
        <span>Costs are estimates. Actual costs depend on provider pricing.</span>
      </div>
    </div>
  );
};

export default ReviewStep;
















//types.ts
export type EvalTypeId = 'model' | 'agent' | 'rag';

export interface EvalType {
  id: EvalTypeId;
  title: string;
  desc: string;
  badge: string;
}

export interface AgentFrameworkOption {
  id: string;
  title: string;
  desc: string;
}

/* ---------- API: models ---------- */

export interface ModelApi {
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

export interface ModelApiResponse {
  models: ModelApi[];
}

/* ---------- API: providers ---------- */

export type ProviderStatus = 'connected' | 'not_connected' | string;

export interface ProviderApi {
  id: string;
  name: string;
  description: string;
  logo_url: string | null;
  base_url: string | null;
  url_template: string | null;
  model_count: number;
  status: ProviderStatus;
}

export interface ProvidersResponse {
  providers: ProviderApi[];
}

/* ---------- API: benchmarks ---------- */

export interface BenchmarkTask {
  name: string;
  value: string;
}

export interface Benchmark {
  name: string;
  description: string;
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

export interface EvaluationDraft {
  name: string;
  type: EvalTypeId | null;
  providers: string[];
  models: string[];
  dataset: string | null;
  subgroup: string[];
  metrics: string[];
  judgeModelId: string | null;
  agentFramework: string | null;
}

export interface WizardStepMeta {
  key: string;
  label: string;
  description: string;
}

export const WIZARD_STEPS: WizardStepMeta[] = [
  { key: 'name', label: 'Name', description: 'Give your evaluation a name' },
  { key: 'type', label: 'Type', description: 'What kind of AI are you testing' },
  { key: 'providers', label: 'Providers', description: 'Choose connected providers' },
  { key: 'models', label: 'Models', description: 'Pick models to compare' },
  { key: 'dataset', label: 'Test Suite', description: 'Select a benchmark or dataset' },
  { key: 'metrics', label: 'Metrics', description: 'Choose what to measure' },
  { key: 'review', label: 'Review', description: 'Confirm and start the run' },
];

/* ---------- API: create / start evaluation ---------- */

export interface JudgeConfig {
  model_id: string;
  base_url: string;
  api_key: string;
}

export interface CreateEvaluationRequest {
  name: string;
  description?: string;
  eval_type: string;
  dataset_id: string;
  benchmark?: string;
  model_ids: string[];
  metrics_config?: Record<string, unknown>;
  selected_metrics: string[];
  dataset_limit?: number;
  selected_category?: string[];
  judge_config?: JudgeConfig;
}

export interface CreateEvaluationResponse {
  id?: string;
  evaluation_id?: string;
  [key: string]: unknown;
}

/* ---------- API: metrics ---------- */

export interface MetricsResponse {
  all_metrics: string[];
  custom_agent_metrics: string[];
}

export type EvaluationStatusValue = 'pending' | 'running' | 'completed' | 'failed' | 'canceled';
export type CeleryState = 'STARTED' | 'SUCCESS' | 'FAILURE' | 'REVOKED' | null;

export interface EvaluationStatusResponse {
  status: EvaluationStatusValue;
  progress: number;
  total: number;
  celery_state: CeleryState;
  error_message: string | null;
}

/* ---------- API: GET /evaluations (history list) ---------- */

export interface EvaluationDatasetConfig {
  dataset_id: string;
}

export interface EvaluationListItem {
  id: string;
  name: string;
  description: string;
  eval_type: string;
  dataset_id: string;
  datasets_config: EvaluationDatasetConfig[];
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

/* ---------- API: GET /evaluations/{id}/results ---------- */

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

// Returned with HTTP 400 when the evaluation hasn't finished running yet
export interface EvaluationResultsErrorResponse {
  detail: string;
}
