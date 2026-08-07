//ReviewStep.tsx
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
            <div className="run-eval__review-row">
              <span>API Key</span>
              <span>{draft.judgeApiKey ? '•'.repeat(10) : 'Not provided'}</span>
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
















//RunEvaluation.tsx
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
  judgeApiKey: '',
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

  const setJudgeApiKey = (value: string) => {
    setDraft((d) => ({ ...d, judgeApiKey: value }));
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
          api_key: draft.judgeApiKey,
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
                judgeApiKey={draft.judgeApiKey}
                onJudgeApiKeyChange={setJudgeApiKey}
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




















//RunEvaluation.scss
@use '../../../styles/variables' as *;

.run-eval {
  width: 100%;
  height: calc(100vh - 166px);
  display: flex;
  flex-direction: column;
  min-height: 0;

  /* ---------- page header ---------- */
  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding-bottom: 18px;
    margin-bottom: 20px;
    border-bottom: 1px solid $border-subtle;
  }

  &__header-left {
    display: flex;
    flex-direction: column;
  }

  &__header-eyebrow {
    display: flex;
    align-items: center;
    gap: 8px;
    font-family: $font-mono;
    font-size: 0.6875rem;
    font-weight: 700;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    color: $primary;
    margin-bottom: 6px;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $primary;
    }
  }

  &__header-meta {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-secondary;
    background: $bg-subtle;
    border: 1px solid $border-subtle;
    border-radius: 999px;
    padding: 7px 13px;
    white-space: nowrap;
    margin-bottom: 3px;
  }

  &__title {
    font-size: 21px;
    font-weight: 800;
    letter-spacing: -0.03em;
    color: $text-primary;
    line-height: 1.15;
  }

  &__subtitle {
    margin-top: 3px;
    color: $text-secondary;
    font-size: 0.84375rem;
  }

  /* ---------- buttons ---------- */
  &__btn {
    display: inline-flex;
    align-items: center;
    gap: 0.5rem;
    font-size: 0.90625rem;
    font-weight: 600;
    padding: 0.5625rem 0.9375rem;
    border-radius: 0.5rem;
    border: 1px solid transparent;
    cursor: pointer;
    transition: background 0.14s ease, border-color 0.14s ease, color 0.14s ease;
    font-family: $font-body;

    &--primary {
      background: $primary;
      color: #fff;
      border-color: $primary;

      &:hover {
        background: $primary-hover;
        border-color: $primary-hover;
      }
    }

    &--secondary {
      background: $bg-main;
      color: $text-primary;
      border-color: $border-default;

      &:hover {
        border-color: $text-primary;
      }
    }

    &--lg {
      padding: 0.625rem 1.125rem;
      font-size: 0.90625rem;
    }

    &--sm {
      padding: 0.375rem 0.6875rem;
      font-size: 0.84375rem;
      background: $bg-main;
      color: $text-secondary;
      border-color: $border-default;

      &:hover {
        border-color: $text-primary;
        color: $text-primary;
      }
    }

    &:disabled {
      opacity: 0.5;
      cursor: not-allowed;
    }
  }

  /* ---------- wizard shell ---------- */
  &__wizard {
    position: relative;
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 20px;
    box-shadow: $shadow-md;
    overflow: hidden;
    flex: 1;
    min-height: 0;
    display: flex;
  }

  /* ---------- sidebar / vertical stepper ---------- */
  &__sidebar {
    flex-shrink: 0;
    width: 320px;
    background: $bg-subtle;
    border-right: 1px solid $border-subtle;
    padding: 24px 14px 28px;
    display: flex;
    flex-direction: column;
    gap: 2px;
    overflow-y: auto;
  }

  /* overall progress indicator pinned above the step list */
  &__sidebar-progress {
    flex-shrink: 0;
    padding: 4px 12px 20px;
    margin-bottom: 6px;
    border-bottom: 1px solid $border-subtle;
  }

  &__sidebar-progress-head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    font-family: $font-mono;
    font-size: 0.6875rem;
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $text-tertiary;
    margin-bottom: 8px;

    span:last-child {
      color: $primary;
      font-weight: 800;
    }
  }

  &__sidebar-progress-track {
    height: 6px;
    border-radius: 999px;
    background: $border-subtle;
    overflow: hidden;
  }

  &__sidebar-progress-fill {
    height: 100%;
    border-radius: 999px;
    background: linear-gradient(90deg, $primary 0%, $primary-hover 100%);
    transition: width 0.28s cubic-bezier(0.32, 0.72, 0, 1);
  }

  &__step {
    position: relative;
    display: flex;
    align-items: flex-start;
    gap: 12px;
    text-align: left;
    width: 100%;
    border: none;
    background: transparent;
    border-radius: 0.75rem;
    padding: 11px 12px 22px 12px;
    cursor: pointer;
    transition: background 0.16s ease, transform 0.16s ease;

    &::before {
      content: '';
      position: absolute;
      top: 42px;
      left: 29px;
      width: 2px;
      height: calc(100% - 34px);
      background: $border-default;
      transition: background 0.2s ease;
    }

    &:last-child {
      padding-bottom: 10px;

      &::before {
        display: none;
      }
    }

    &:disabled {
      cursor: default;
    }

    &:not(:disabled):hover {
      background: $bg-inset;
    }

    &:not(:disabled):active {
      transform: scale(0.99);
    }
  }

  &__step-marker {
    position: relative;
    z-index: 1;
    flex-shrink: 0;
    width: 34px;
    height: 34px;
    border-radius: 50%;
    display: grid;
    place-items: center;
    background: $bg-main;
    border: 1.5px solid $border-default;
    color: $text-tertiary;
    transition: background 0.18s ease, border-color 0.18s ease, color 0.18s ease, box-shadow 0.18s ease;
  }

  &__step-text {
    display: flex;
    flex-direction: column;
    gap: 2px;
    padding-top: 5px;
    min-width: 0;
  }

  &__step-label {
    font-size: 0.875rem;
    font-weight: 700;
    color: $text-primary;
    transition: color 0.18s ease;
  }

  &__step-desc {
    font-size: 0.75rem;
    color: $text-tertiary;
    line-height: 1.35;
  }

  &__step--active {
    background: $bg-main;
    box-shadow: $shadow-sm;

    .run-eval__step-marker {
      background: $primary;
      border-color: $primary;
      color: #fff;
      box-shadow: 0 0 0 5px $primary-light;
    }

    .run-eval__step-label {
      color: $primary;
    }
  }

  &__step--complete {
    &::before {
      background: linear-gradient(180deg, $primary 0%, $primary-hover 100%);
    }

    .run-eval__step-marker {
      background: $primary-light;
      border-color: $primary;
      color: $primary;
    }

    &:not(:disabled):hover {
      background: rgba(0, 0, 0, 0.02);
    }
  }

  &__step--upcoming {
    .run-eval__step-label {
      color: $text-secondary;
    }

    .run-eval__step-desc {
      color: #a8b1bb;
    }
  }

  /* ---------- content pane ---------- */
  &__content {
    flex: 1;
    min-width: 0;
    display: flex;
    flex-direction: column;
    padding: 28px 36px 24px;
  }

  &__body {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding-right: 4px;
    margin-right: -4px;
  }

  &__spin {
    animation: run-eval-spin 0.8s linear infinite;
  }

  @keyframes run-eval-spin {
    to {
      transform: rotate(360deg);
    }
  }

  /* ---------- step kicker + heading ---------- */
  &__step-kicker {
    display: flex;
    align-items: center;
    gap: 8px;
    font-family: $font-mono;
    font-size: 0.75rem;
    font-weight: 700;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    color: $primary;
    margin-bottom: 8px;
    flex-shrink: 0;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $primary;
    }
  }

  /* ---------- step cards ---------- */
  &__card {
    &--wide {
      max-width: none;
    }
  }

  &__step-title {
    font-size: 19px;
    font-weight: 800;
    letter-spacing: -0.02em;
    line-height: 1.2;
    color: $text-primary;
  }

  &__step-desc {
    margin-top: 6px;
    font-size: 0.9375rem;
    color: $text-secondary;
    max-width: 608px;
  }

  &__step-header-row {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 1rem;
  }

  &__field {
    max-width: 480px;
    margin-top: 1.75rem;
  }

  &__label {
    display: block;
    font-size: 0.84375rem;
    font-weight: 600;
    color: $text-secondary;
    margin-bottom: 0.4375rem;
  }

  &__input {
    width: 100%;
    border: 1px solid $border-default;
    border-radius: 0.5rem;
    padding: 0.625rem 0.75rem;
    font-size: 0.9375rem;
    font-family: $font-body;
    color: $text-primary;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &::placeholder {
      color: #a8b1bb;
    }

    &:focus {
      outline: none;
      border-color: $primary;
      box-shadow: 0 0 0 0.1875rem $primary-light;
    }

    &--lg {
      padding: 0.75rem 0.875rem;
      font-size: 1rem;
    }
  }

  /* ---------- suggestion / static chips ---------- */
  &__suggestions {
    margin-top: 1.125rem;
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 0.5rem;
  }

  &__suggestions-label {
    font-size: 0.84375rem;
    color: $text-tertiary;
    margin-right: 0.125rem;
  }

  &__chip {
    font-size: 0.8125rem;
    font-weight: 500;
    color: $text-secondary;
    background: $bg-subtle;
    border: 1px solid $border-default;
    border-radius: 999px;
    padding: 0.3125rem 0.75rem;
    cursor: pointer;
    transition: border-color 0.14s ease, color 0.14s ease, background 0.14s ease;

    &:hover {
      border-color: $primary;
      color: $primary;
    }

    &--active {
      background: $primary;
      border-color: $primary;
      color: #fff;
    }

    &--static {
      cursor: default;
      font-size: 0.75rem;
      padding: 0.1875rem 0.5rem;

      &:hover {
        border-color: $border-default;
        color: $text-secondary;
      }
    }
  }

  /* ---------- eval type cards ---------- */
  &__type-grid {
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
    margin-top: 1.5rem;
  }

  &__type-card {
    position: relative;
    display: flex;
    align-items: flex-start;
    gap: 0.875rem;
    text-align: left;
    width: 100%;
    padding: 1.125rem 3rem 1.125rem 1.125rem;
    border: 1px solid $border-default;
    border-radius: 0.75rem;
    background: $bg-main;
    cursor: pointer;
    transition: border-color 0.14s ease, background 0.14s ease;

    &:hover {
      border-color: $primary;
    }

    &--selected {
      border-color: $primary;
      background: $primary-light;
    }
  }

  &__type-icon {
    width: 38px;
    height: 38px;
    flex-shrink: 0;
    border-radius: 0.5rem;
    background: $bg-subtle;
    color: $primary;
    display: grid;
    place-items: center;
  }

  &__type-content {
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
    flex: 1;
  }

  &__type-title {
    font-size: 1rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__type-desc {
    font-size: 0.875rem;
    color: $text-secondary;
    line-height: 1.5;
  }

  &__type-check {
    position: absolute;
    top: 50%;
    right: 1.125rem;
    transform: translateY(-50%);
    width: 20px;
    height: 20px;
    border-radius: 50%;
    background: $primary;
    color: #fff;
    display: grid;
    place-items: center;
  }

  /* ---------- optional agent framework sub-section ---------- */
  &__framework-section {
    margin-top: 1.75rem;
    padding-top: 1.5rem;
    border-top: 1px solid $border-subtle;
  }

  &__framework-grid {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 0.75rem;
    margin-top: 1rem;
  }

  &__type-card--framework {
    padding: 0.875rem 2.75rem 0.875rem 0.875rem;
    gap: 0.75rem;

    .run-eval__type-icon {
      width: 32px;
      height: 32px;
    }

    .run-eval__type-title {
      font-size: 0.9375rem;
    }

    .run-eval__type-desc {
      font-size: 0.8125rem;
    }
  }

  &__badge {
    align-self: flex-start;
    flex-shrink: 0;
    font-size: 0.71875rem;
    font-weight: 600;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $primary;
    background: $primary-light;
    border-radius: 0.375rem;
    padding: 0.25rem 0.5rem;

    &--soft {
      margin-top: 0.625rem;
    }
  }

  /* ---------- async states (providers / models) ---------- */
  &__loading-state {
    margin-top: 1.5rem;
    display: flex;
    align-items: center;
    gap: 0.5rem;
    font-size: 0.90625rem;
    color: $text-secondary;
    padding: 1.5rem 0;
  }

  &__inline-error {
    margin-top: 1.5rem;
    display: flex;
    align-items: center;
    gap: 0.5rem;
    font-size: 0.875rem;
    color: $danger;
    background: $danger-subtle;
    border-radius: 0.5rem;
    padding: 0.75rem 0.875rem;
  }

  &__filter-empty {
    font-size: 0.8125rem;
    color: $text-tertiary;
  }

  /* ---------- providers ---------- */
  &__provider-grid {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 0.75rem;
    margin-top: 1.5rem;
  }

  &__provider-card {
    position: relative;
    display: flex;
    align-items: flex-start;
    gap: 0.75rem;
    text-align: left;
    padding: 0.875rem 2.5rem 0.875rem 0.875rem;
    border: 1px solid $border-default;
    border-radius: 0.75rem;
    background: $bg-main;
    cursor: pointer;
    transition: border-color 0.14s ease, background 0.14s ease;

    &:hover:not(&--disabled) {
      border-color: $primary;
    }

    &--selected {
      border-color: $primary;
      background: $primary-light;
    }

    &--disabled {
      cursor: not-allowed;
      opacity: 0.7;
    }
  }

  &__provider-logo {
    width: 34px;
    height: 34px;
    flex-shrink: 0;
    border-radius: 0.5rem;
    background: $text-primary;
    color: #fff;
    font-weight: 700;
    font-size: 0.875rem;
    display: grid;
    place-items: center;
    margin-top: 0.0625rem;
    overflow: hidden;

    img {
      width: 100%;
      height: 100%;
      object-fit: cover;
    }
  }

  &__provider-info {
    display: flex;
    flex-direction: column;
    gap: 0.3125rem;
    min-width: 0;
    flex: 1;
  }

  &__provider-name-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 0.5rem;
  }

  &__provider-name {
    font-size: 0.9375rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__provider-desc {
    font-size: 0.8125rem;
    color: $text-tertiary;
  }

  &__status-badge {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 0.3125rem;
    font-size: 0.71875rem;
    font-weight: 600;
    letter-spacing: 0.01em;
    color: $text-tertiary;
    background: $bg-inset;
    border-radius: 999px;
    padding: 0.1875rem 0.5rem;

    &--on {
      color: $success;
      background: $success-subtle;
    }
  }

  &__hint {
    margin-top: 1.25rem;
    display: flex;
    align-items: center;
    gap: 0.5rem;
    font-size: 0.875rem;
    color: $text-tertiary;

    svg {
      flex-shrink: 0;
    }
  }

  &__link {
    background: none;
    border: none;
    padding: 0;
    color: $primary;
    font-weight: 600;
    font-size: inherit;
    cursor: pointer;

    &:hover {
      text-decoration: underline;
    }
  }

  /* ---------- models step ---------- */
  /* EDITED: fixed-height layout so the filters column and the model card
     list scroll independently of one another (and of run-eval__body). */
  &__models-layout {
    display: grid;
    grid-template-columns: 15rem 1fr;
    gap: 1.5rem;
    margin-top: 1.5rem;
    align-items: stretch;
    height: 34rem;
    min-height: 0;
  }

  &__filters {
    border: 1px solid $border-subtle;
    border-radius: 0.75rem;
    padding: 1rem;
    display: flex;
    flex-direction: column;
    gap: 1.125rem;
    min-height: 0;
    overflow: hidden;
  }

  &__filters-head {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: space-between;
    font-size: 0.875rem;
    font-weight: 600;
    color: $text-primary;
  }

  // NEW: scrolls independently from the model card list on the right.
  &__filters-scroll {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    display: flex;
    flex-direction: column;
    gap: 1.125rem;
    padding-right: 4px;
    margin-right: -4px;
  }

  &__filter-section {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    flex-shrink: 0;
  }

  &__filter-title {
    font-family: $font-mono;
    font-size: 0.71875rem;
    font-weight: 600;
    letter-spacing: 0.08em;
    text-transform: uppercase;
    color: $text-tertiary;
  }

  &__filter-options {
    display: flex;
    flex-direction: column;
    gap: 0.4375rem;
  }

  &__filter-chip {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    font-size: 0.875rem;
    color: $text-secondary;
    cursor: pointer;

    input {
      accent-color: $primary;
    }
  }

  &__models-main {
    min-width: 0;
    min-height: 0;
    display: flex;
    flex-direction: column;
  }

  &__search-bar {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 0.5rem;
    border: 1px solid $border-default;
    border-radius: 0.5rem;
    padding: 0.5625rem 0.75rem;
    color: $text-tertiary;

    input {
      flex: 1;
      border: none;
      outline: none;
      font-size: 0.90625rem;
      color: $text-primary;
      background: transparent;
      font-family: $font-body;

      &::placeholder {
        color: $text-tertiary;
      }
    }
  }

  &__active-filters {
    flex-shrink: 0;
    display: flex;
    flex-wrap: wrap;
    gap: 0.375rem;
    margin-top: 0.75rem;
  }

  &__tag {
    display: inline-flex;
    align-items: center;
    gap: 0.375rem;
    font-size: 0.78125rem;
    color: $primary;
    background: $primary-light;
    border-radius: 0.375rem;
    padding: 0.25rem 0.25rem 0.25rem 0.5rem;

    button {
      display: grid;
      place-items: center;
      border: none;
      background: transparent;
      color: inherit;
      cursor: pointer;
    }
  }

  // NEW: scrolls independently from the filters panel on the left.
  &__models-scroll {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    margin-top: 1rem;
    padding-right: 4px;
    margin-right: -4px;
  }

  &__models-grid {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 0.75rem;
  }

  &__model-card {
    position: relative;
    text-align: left;
    padding: 0.875rem 1rem;
    border: 1px solid $border-default;
    border-radius: 0.75rem;
    background: $bg-main;
    cursor: pointer;
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    transition: border-color 0.14s ease, background 0.14s ease;

    &:hover {
      border-color: $primary;
    }

    &--selected {
      border-color: $primary;
      background: $primary-light;
    }
  }

  &__model-top {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 0.5rem;
  }

  &__model-name {
    font-size: 0.9375rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__model-provider {
    font-size: 0.84375rem;
    color: $text-tertiary;
    margin-top: -0.25rem;
  }

  &__model-caps {
    display: flex;
    flex-wrap: wrap;
    gap: 0.3125rem;
  }

  &__model-meta {
    display: flex;
    flex-wrap: wrap;
    gap: 0.625rem;
    font-size: 0.78125rem;
    color: $text-tertiary;
    margin-top: 0.125rem;
  }

  &__empty {
    grid-column: 1 / -1;
    padding: 2rem;
    text-align: center;
    color: $text-tertiary;
    font-size: 0.90625rem;
  }

  &__selected-bar {
    flex-shrink: 0;
    margin-top: 1rem;
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0.75rem 1rem;
    background: $primary-light;
    border-radius: 0.625rem;
    font-size: 0.90625rem;
    color: $text-primary;

    strong {
      color: $primary;
    }
  }

  /* ---------- dataset step ---------- */
  &__tabs {
    display: flex;
    gap: 0.375rem;
    margin-top: 1.5rem;
    border-bottom: 1px solid $border-subtle;
  }

  &__tab {
    padding: 0.5625rem 0.25rem;
    margin-right: 1.25rem;
    border: none;
    background: transparent;
    font-size: 0.90625rem;
    font-weight: 600;
    color: $text-tertiary;
    cursor: pointer;
    border-bottom: 2px solid transparent;
    transition: color 0.14s ease, border-color 0.14s ease;

    &--active {
      color: $primary;
      border-bottom-color: $primary;
    }
  }

  &__category-filters {
    display: flex;
    flex-wrap: wrap;
    gap: 0.5rem;
    margin-top: 1.125rem;
  }

  &__dataset-grid {
    margin-top: 1.125rem;
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 0.75rem;
  }

  &__dataset-card {
    position: relative;
    text-align: left;
    padding: 1rem 1.125rem;
    border: 1px solid $border-default;
    border-radius: 0.75rem;
    background: $bg-main;
    cursor: pointer;
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    transition: border-color 0.14s ease, background 0.14s ease;

    &:hover {
      border-color: $primary;
    }

    &--selected {
      border-color: $primary;
      background: $primary-light;
    }
  }

  &__dataset-top {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 0.5rem;
  }

  &__dataset-top-actions {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    flex-shrink: 0;

    .run-eval__type-check {
      position: static;
    }
  }

  &__subgroup-btn {
    display: inline-flex;
    align-items: center;
    gap: 0.25rem;
    font-size: 0.71875rem;
    font-weight: 600;
    color: $text-secondary;
    background: $bg-subtle;
    border: 1px solid $border-default;
    border-radius: 999px;
    padding: 0.25rem 0.5625rem;
    cursor: pointer;
    transition: border-color 0.14s ease, color 0.14s ease, background 0.14s ease;

    &:hover {
      border-color: $primary;
      color: $primary;
      background: $primary-light;
    }
  }

  &__dataset-name {
    font-size: 0.9375rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__dataset-desc {
    font-size: 0.875rem;
    color: $text-secondary;
    line-height: 1.5;
  }

  &__dataset-meta {
    display: flex;
    flex-wrap: wrap;
    gap: 0.625rem;
    font-size: 0.78125rem;
    color: $text-tertiary;
  }

  &__dataset-caps {
    display: flex;
    flex-wrap: wrap;
    gap: 0.3125rem;
  }

  &__empty-state,
  &__upload-zone {
    margin-top: 1.5rem;
    border: 1.5008px dashed $border-strong;
    border-radius: 0.75rem;
    padding: 2.75rem 1.5rem;
    text-align: center;
    color: $text-tertiary;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 0.5rem;

    svg {
      color: $text-tertiary;
      margin-bottom: 0.25rem;
    }

    h3 {
      font-size: 1rem;
      color: $text-primary;
    }

    p {
      font-size: 0.875rem;
    }
  }

  &__upload-zone {
    cursor: pointer;

    &:hover {
      border-color: $primary;
    }
  }

  &__format-chips {
    display: flex;
    gap: 0.375rem;
    margin-top: 0.375rem;
  }

  /* ---------- metrics step ---------- */
  &__metrics-count {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 0.625rem;
    font-size: 0.875rem;
    color: $text-secondary;
  }

  &__metrics-count-num {
    font-weight: 700;
    color: $primary;
  }

  &__metrics-bulk-actions {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    margin-left: 0.25rem;
  }

  &__metrics-bulk-divider {
    width: 1px;
    height: 12px;
    background: $border-strong;
  }

  &__metrics-toggle-all {
    font-family: $font-body;
    font-size: 0.8125rem;
    font-weight: 600;
    color: $primary;
    background: transparent;
    border: none;
    padding: 0;
    cursor: pointer;

    &:hover:not(:disabled) {
      text-decoration: underline;
    }

    &:disabled {
      color: $text-tertiary;
      cursor: not-allowed;
    }
  }

  &__metric-group {
    margin-top: 1.5rem;

    &:first-child {
      margin-top: 0;
    }

    display: flex;
    flex-direction: column;
    gap: 0.75rem;
  }

  &__metrics-grid {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 0.75rem;
  }

  /* ---------- redesigned metric card ---------- */
  &__metric-card {
    position: relative;
    display: flex;
    align-items: center;
    gap: 0.75rem;
    text-align: left;
    width: 100%;
    padding: 0.875rem 2.5rem 0.875rem 0.875rem;
    border: 1px solid $border-default;
    border-radius: 0.875rem;
    background: $bg-main;
    cursor: pointer;
    box-shadow: 0 1px 2px rgba(16, 24, 40, 0.03);
    transition: border-color 0.16s ease, background 0.16s ease, box-shadow 0.16s ease, transform 0.16s ease;

    &:hover {
      border-color: $primary;
      box-shadow: $shadow-sm;
      transform: translateY(-1px);
    }

    &--selected {
      border-color: $primary;
      background: linear-gradient(135deg, $primary-light 0%, rgba(255, 255, 255, 0) 120%);
      box-shadow: 0 0 0 1px $primary, $shadow-sm;
    }
  }

  &__metric-card-icon {
    flex-shrink: 0;
    width: 32px;
    height: 32px;
    border-radius: 0.625rem;
    display: grid;
    place-items: center;
    background: $bg-subtle;
    color: $text-tertiary;
    transition: background 0.16s ease, color 0.16s ease;
  }

  &__metric-card--selected &__metric-card-icon {
    background: $primary;
    color: #fff;
  }

  &__metric-card-check {
    position: absolute;
    top: 50%;
    right: 0.875rem;
    transform: translateY(-50%) scale(0.6);
    width: 20px;
    height: 20px;
    border-radius: 50%;
    display: grid;
    place-items: center;
    background: $bg-subtle;
    color: transparent;
    opacity: 0;
    transition: opacity 0.16s ease, transform 0.16s ease, background 0.16s ease, color 0.16s ease;

    &--on {
      opacity: 1;
      transform: translateY(-50%) scale(1);
      background: $primary;
      color: #fff;
    }
  }

  &__metric-name {
    display: block;
    font-size: 0.875rem;
    font-weight: 600;
    color: $text-primary;
    line-height: 1.3;
  }

  /* ---------- metrics + judge model split (two independent scroll columns) ---------- */
  &__metrics-layout {
    display: grid;
    grid-template-columns: 1fr 350px;
    align-items: stretch;
    gap: 1.5rem;
    margin-top: 0.5rem;
    height: 32rem;
    min-height: 0;
  }

  &__metrics-main {
    min-width: 0;
    min-height: 0;
    display: flex;
    flex-direction: column;
  }

  // Scrolls independently from the judge panel on the right.
  &__metrics-main-scroll {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding-right: 4px;
    margin-right: -4px;
  }

  &__judge-panel {
    flex-shrink: 0;
    background: $bg-subtle;
    border: 1px solid $border-subtle;
    border-radius: 0.75rem;
    padding: 1.125rem 1.25rem;
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
    min-height: 0;
    overflow: hidden;

    .run-eval__filter-title {
      display: flex;
      align-items: center;
      gap: 0.375rem;
      margin-top: 0;

      svg {
        color: $primary;
      }
    }
  }

  &__judge-hint {
    font-size: 0.78125rem;
    color: $text-tertiary;
    line-height: 1.5;
    margin-bottom: 0.75rem;
    flex-shrink: 0;
  }

  // Scrolls independently from the metrics list on the left.
  &__judge-panel-scroll {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
    padding-right: 4px;
    margin-right: -4px;
  }

  &__judge-empty {
    padding: 1.25rem 0.75rem;
    text-align: center;
    border: 1px dashed $border-strong;
    border-radius: 0.625rem;
    font-size: 0.8125rem;
    color: $text-tertiary;
  }

  &__judge-list {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    flex-shrink: 0;
  }

  &__judge-row {
    display: flex;
    align-items: center;
    gap: 0.625rem;
    width: 100%;
    text-align: left;
    padding: 0.625rem 0.75rem;
    border: 1px solid $border-default;
    border-radius: 0.625rem;
    background: $bg-main;
    cursor: pointer;
    transition: border-color 0.14s ease, background 0.14s ease;

    &:hover {
      border-color: $primary;
    }

    &--selected {
      border-color: $primary;
      background: $primary-light;
    }
  }

  &__judge-row-text {
    display: flex;
    flex-direction: column;
    gap: 1px;
    min-width: 0;
  }

  &__judge-row-name {
    font-size: 0.84375rem;
    font-weight: 600;
    color: $text-primary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__judge-row-meta {
    font-size: 0.71875rem;
    color: $text-tertiary;
  }

  &__radio {
    flex-shrink: 0;
    width: 16px;
    height: 16px;
    border-radius: 50%;
    border: 1.5px solid $border-strong;
    background: $bg-main;
    position: relative;
    transition: border-color 0.14s ease;

    &--checked {
      border-color: $primary;
      border-width: 5px;
    }
  }

  &__field--judge {
    max-width: none;
    margin-top: 0.875rem;

    .run-eval__label {
      display: flex;
      align-items: center;
      gap: 0.375rem;
    }
  }

  &__metric-tooltip {
    display: block;
    margin-top: 0.25rem;
    font-size: 0.78125rem;
    color: $text-tertiary;
    line-height: 1.4;
  }

  /* ---------- review step ---------- */
  &__review-stats {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 0.75rem;
    margin-top: 1.5rem;
  }

  &__review-stat {
    display: flex;
    align-items: center;
    gap: 0.625rem;
    padding: 0.875rem 1rem;
    border: 1px solid $border-subtle;
    border-radius: 0.75rem;
    background: $bg-subtle;
  }

  &__review-stat-icon {
    flex-shrink: 0;
    width: 34px;
    height: 34px;
    border-radius: 0.625rem;
    display: grid;
    place-items: center;
    background: $bg-main;
    color: $primary;
    box-shadow: 0 1px 2px rgba(16, 24, 40, 0.04);
  }

  &__review-stat-label {
    display: block;
    font-size: 0.6875rem;
    font-weight: 600;
    color: $text-tertiary;
    margin-bottom: 2px;
  }

  &__review-stat-value {
    display: block;
    font-size: 0.9375rem;
    font-weight: 800;
    color: $text-primary;
    letter-spacing: -0.01em;
  }

  &__review-section {
    margin-top: 1.75rem;

    &:first-of-type {
      margin-top: 1.75rem;
    }
  }

  &__review-model-list {
    display: flex;
    flex-wrap: wrap;
    gap: 0.5rem;
    margin-top: 0.75rem;
  }

  &__review-model-chip {
    display: flex;
    flex-direction: column;
    gap: 1px;
    padding: 0.5rem 0.75rem;
    border: 1px solid $border-default;
    border-radius: 0.625rem;
    background: $bg-main;
  }

  &__review-model-chip-name {
    font-size: 0.8125rem;
    font-weight: 700;
    color: $text-primary;
  }

  &__review-model-chip-provider {
    font-size: 0.6875rem;
    color: $text-tertiary;
  }

  &__review-caps {
    display: flex;
    flex-wrap: wrap;
    gap: 0.375rem;
    margin-top: 0.75rem;
  }

  &__review-metric-pill {
    display: inline-flex;
    align-items: center;
    gap: 0.3125rem;
    font-size: 0.75rem;
    font-weight: 600;
    color: $success;
    background: $success-subtle;
    border-radius: 999px;
    padding: 0.25rem 0.625rem 0.25rem 0.5rem;
  }

  &__review {
    margin-top: 0.75rem;
    border: 1px solid $border-subtle;
    border-radius: 0.75rem;
    overflow: hidden;
  }

  &__review-row {
    display: flex;
    align-items: baseline;
    justify-content: space-between;
    gap: 1rem;
    padding: 0.75rem 1rem;
    font-size: 0.90625rem;
    border-bottom: 1px solid $border-subtle;

    &:last-child {
      border-bottom: 0;
    }

    span:first-child {
      color: $text-tertiary;
      flex-shrink: 0;
    }

    span:last-child {
      color: $text-primary;
      font-weight: 500;
      text-align: right;
    }

    &--highlight span:last-child {
      color: $primary;
      font-weight: 700;
    }
  }

  &__review-divider {
    height: 1px;
    background: $border-subtle;
  }

  /* ---------- subgroup drawer ---------- */
  &__drawer-overlay {
    position: fixed;
    inset: 0;
    background: rgba(14, 21, 38, 0.32);
    opacity: 0;
    visibility: hidden;
    transition: opacity 0.2s ease, visibility 0.2s ease;
    z-index: 200;

    &--open {
      opacity: 1;
      visibility: visible;
    }
  }

  &__drawer {
    position: absolute;
    top: 0;
    right: 0;
    height: 100%;
    width: 400px;
    max-width: 92vw;
    background: $bg-main;
    box-shadow: $shadow-xl;
    display: flex;
    flex-direction: column;
    transform: translateX(100%);
    transition: transform 0.28s cubic-bezier(0.32, 0.72, 0, 1);

    &--open {
      transform: translateX(0);
    }
  }

  &__drawer-header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 1rem;
    padding: 20px 20px 16px;
    border-bottom: 1px solid $border-subtle;
  }

  &__drawer-eyebrow {
    font-family: $font-mono;
    font-size: 0.6875rem;
    font-weight: 700;
    letter-spacing: 0.1em;
    text-transform: uppercase;
    color: $primary;
    margin-bottom: 4px;
  }

  &__drawer-title {
    font-size: 1.0625rem;
    font-weight: 800;
    color: $text-primary;
    letter-spacing: -0.02em;
  }

  &__drawer-close {
    flex-shrink: 0;
    display: grid;
    place-items: center;
    width: 30px;
    height: 30px;
    border-radius: 0.5rem;
    border: 1px solid $border-default;
    background: $bg-main;
    color: $text-secondary;
    cursor: pointer;
    transition: border-color 0.14s ease, color 0.14s ease;

    &:hover {
      border-color: $text-primary;
      color: $text-primary;
    }
  }

  &__drawer-body {
    flex: 1;
    overflow-y: auto;
    padding: 16px 20px 24px;
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
  }

  &__drawer-task {
    position: relative;
    display: flex;
    align-items: center;
    gap: 0.75rem;
    text-align: left;
    width: 100%;
    padding: 0.75rem 0.875rem;
    border: 1px solid $border-default;
    border-radius: 0.625rem;
    background: $bg-main;
    cursor: pointer;
    transition: border-color 0.14s ease, background 0.14s ease;

    &:hover {
      border-color: $primary;
    }

    &--selected {
      border-color: $primary;
      background: $primary-light;
    }
  }

  &__drawer-task-name {
    font-size: 0.875rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__checkbox {
    flex-shrink: 0;
    width: 18px;
    height: 18px;
    border-radius: 5px;
    border: 1.5px solid $border-strong;
    background: $bg-main;
    display: grid;
    place-items: center;
    color: transparent;
    transition: background 0.14s ease, border-color 0.14s ease, color 0.14s ease;

    &--checked {
      background: $primary;
      border-color: $primary;
      color: #fff;
    }
  }

  /* ---------- shared feedback ---------- */
  &__error {
    margin-top: 1.25rem;
    font-size: 0.875rem;
    color: $danger;
    background: $danger-subtle;
    border-radius: 0.5rem;
    padding: 0.625rem 0.875rem;
  }

  &__nav {
    flex-shrink: 0;
    margin-top: 1.25rem;
    padding-top: 1.25rem;
    border-top: 1px solid $border-subtle;
    display: flex;
    align-items: center;
    justify-content: space-between;
  }

  /* ---------- responsive ---------- */
  @media (max-width: 896px) {
    &__provider-grid,
    &__models-grid,
    &__dataset-grid,
    &__metrics-grid {
      grid-template-columns: 1fr;
    }

    &__models-layout {
      grid-template-columns: 1fr;
      height: auto;
      max-height: none;
    }

    &__filters {
      max-height: 16rem;
    }

    &__models-scroll {
      margin-top: 0.75rem;
    }
  }

  @media (max-width: 900px) {
    &__metrics-layout {
      grid-template-columns: 1fr;
      height: auto;
    }

    &__metrics-main-scroll {
      max-height: 22rem;
    }

    &__judge-panel {
      width: 100%;
    }

    &__judge-panel-scroll {
      max-height: 18rem;
    }

    &__framework-grid {
      grid-template-columns: 1fr;
    }
  }

  @media (max-width: 640px) {
    &__metrics-grid {
      grid-template-columns: 1fr;
    }

    &__review-stats {
      grid-template-columns: repeat(2, 1fr);
    }
  }

  @media (max-width: 720px) {
    &__wizard {
      flex-direction: column;
    }

    &__sidebar {
      width: 100%;
      flex-direction: row;
      overflow-x: auto;
      border-right: none;
      border-bottom: 1px solid $border-subtle;
      padding: 14px;
      gap: 6px;
    }

    &__step {
      flex-direction: column;
      align-items: center;
      text-align: center;
      padding: 8px 10px;
      flex-shrink: 0;
      width: 96px;

      &::before {
        display: none;
      }
    }

    &__step-text {
      align-items: center;
      padding-top: 2px;
    }

    &__step-desc {
      display: none;
    }

    &__content {
      padding: 24px 20px;
    }
  }
}
