//Datasetstep.tsx
import { useMemo, useState, type FC, type MouseEvent } from 'react';
import { UploadCloud, Check, Loader2, AlertTriangle, Layers, X } from 'lucide-react';
import type { Benchmark, EvalTypeId } from '../types';

interface Props {
  evalType: EvalTypeId | null;
  benchmarks: Benchmark[];
  loading: boolean;
  error: string | null;
  selected: string | null;
  onSelect: (id: string) => void;
  subgroup: string[];
  onToggleSubgroup: (value: string) => void;
}

const DatasetStep: FC<Props> = ({
  evalType,
  benchmarks,
  loading,
  error,
  selected,
  onSelect,
  subgroup,
  onToggleSubgroup,
}) => {
  const [tab, setTab] = useState<'official' | 'private'>('official');
  const [category, setCategory] = useState('All');
  const [drawerBenchmark, setDrawerBenchmark] = useState<Benchmark | null>(null);

  const categories = useMemo(() => {
    const set = new Set<string>();
    benchmarks.forEach((b) => set.add(b.type));
    return ['All', ...Array.from(set).sort()];
  }, [benchmarks]);

  const filtered = useMemo(
    () => benchmarks.filter((b) => category === 'All' || b.type === category),
    [benchmarks, category]
  );

  const openSubgroupDrawer = (b: Benchmark, e: MouseEvent) => {
    e.stopPropagation();
    setDrawerBenchmark(b);
  };

  const closeDrawer = () => setDrawerBenchmark(null);

  const handleTaskToggle = (taskValue: string) => {
    if (drawerBenchmark && selected !== drawerBenchmark.name) {
      onSelect(drawerBenchmark.name);
    }
    onToggleSubgroup(taskValue);
  };

  return (
    <div className="run-eval__card">
      <h2 className="run-eval__step-title">Pick a test suite</h2>
      <p className="run-eval__step-desc">Test suites contain questions that measure AI capabilities.</p>

      <div className="run-eval__tabs">
        {(['official', 'private'] as const).map((t) => (
          <button
            key={t}
            type="button"
            className={`run-eval__tab${tab === t ? ' run-eval__tab--active' : ''}`}
            onClick={() => setTab(t)}
          >
            {t === 'official' ? 'Benchmarks' : 'Upload'}
          </button>
        ))}
      </div>

      {tab === 'official' && (
        <>
          {loading && (
            <div className="run-eval__loading-state">
              <Loader2 size={18} className="run-eval__spin" />
              Loading test suites…
            </div>
          )}

          {!loading && error && (
            <div className="run-eval__inline-error">
              <AlertTriangle size={15} />
              {error}
            </div>
          )}

          {!loading && !error && (
            <>
              <div className="run-eval__category-filters">
                {categories.map((c) => (
                  <button
                    key={c}
                    type="button"
                    className={`run-eval__chip${category === c ? ' run-eval__chip--active' : ''}`}
                    onClick={() => setCategory(c)}
                  >
                    {c}
                  </button>
                ))}
              </div>

              <div className="run-eval__dataset-grid">
                {filtered.map((b) => {
                  const isSelected = selected === b.name;
                  const recommended = evalType ? b.type === evalType : false;
                  const hasSubgroupSelected = isSelected && subgroup.length > 0;
                  return (
                    <button
                      key={b.name}
                      type="button"
                      className={`run-eval__dataset-card${isSelected ? ' run-eval__dataset-card--selected' : ''}`}
                      onClick={() => onSelect(b.name)}
                    >
                      <div className="run-eval__dataset-top">
                        <span className="run-eval__dataset-name">{b.name}</span>
                        <span className="run-eval__dataset-top-actions">
                          {b.tasks.length > 0 && (
                            <span
                              role="button"
                              tabIndex={0}
                              className="run-eval__subgroup-btn"
                              title="View subgroups"
                              onClick={(e) => openSubgroupDrawer(b, e)}
                              onKeyDown={(e) => {
                                if (e.key === 'Enter' || e.key === ' ') openSubgroupDrawer(b, e as never);
                              }}
                            >
                              <Layers size={12} />
                              Subgroup
                            </span>
                          )}
                          {isSelected && (
                            <span className="run-eval__type-check">
                              <Check size={12} strokeWidth={2.75} />
                            </span>
                          )}
                        </span>
                      </div>
                      <p className="run-eval__dataset-desc">{b.description}</p>
                      <div className="run-eval__dataset-meta n">
                        <span>{b.task_count} tasks</span>
                        <span>{b.type}</span>
                      </div>
                      {b.required_capabilities.length > 0 && (
                        <div className="run-eval__dataset-caps">
                          {b.required_capabilities.slice(0, 4).map((c) => (
                            <span key={c} className="run-eval__chip run-eval__chip--static">
                              {c}
                            </span>
                          ))}
                        </div>
                      )}
                      {hasSubgroupSelected && (
                        <span className="run-eval__badge run-eval__badge--soft">
                          {subgroup.length} subgroup{subgroup.length === 1 ? '' : 's'} selected
                        </span>
                      )}
                      {!hasSubgroupSelected && recommended && (
                        <span className="run-eval__badge run-eval__badge--soft">Recommended</span>
                      )}
                    </button>
                  );
                })}
                {filtered.length === 0 && <p className="run-eval__empty">No test suites match this category.</p>}
              </div>
            </>
          )}
        </>
      )}

      {tab === 'private' && (
        <div className="run-eval__upload-zone">
          <UploadCloud size={26} />
          <h3>Upload Test Data</h3>
          <p>Drag &amp; drop or click to browse</p>
          <div className="run-eval__format-chips">
            {['CSV', 'JSON', 'JSONL', 'HuggingFace'].map((f) => (
              <span key={f} className="run-eval__chip run-eval__chip--static">
                {f}
              </span>
            ))}
          </div>
        </div>
      )}

      <div
        className={`run-eval__drawer-overlay${drawerBenchmark ? ' run-eval__drawer-overlay--open' : ''}`}
        onClick={closeDrawer}
      >
        <aside
          className={`run-eval__drawer${drawerBenchmark ? ' run-eval__drawer--open' : ''}`}
          onClick={(e) => e.stopPropagation()}
        >
          <div className="run-eval__drawer-header">
            <div>
              <p className="run-eval__drawer-eyebrow">Subgroups</p>
              <h3 className="run-eval__drawer-title">{drawerBenchmark?.name}</h3>
            </div>
            <button type="button" className="run-eval__drawer-close" onClick={closeDrawer} aria-label="Close">
              <X size={16} />
            </button>
          </div>

          <div className="run-eval__drawer-body">
            {drawerBenchmark && drawerBenchmark.tasks.length === 0 && (
              <p className="run-eval__empty">This test suite has no subgroups.</p>
            )}
            {drawerBenchmark?.tasks.map((task) => {
              const taskSelected = subgroup.includes(task.value);
              return (
                <button
                  key={task.value}
                  type="button"
                  className={`run-eval__drawer-task${taskSelected ? ' run-eval__drawer-task--selected' : ''}`}
                  onClick={() => handleTaskToggle(task.value)}
                  role="checkbox"
                  aria-checked={taskSelected}
                >
                  <span className={`run-eval__checkbox${taskSelected ? ' run-eval__checkbox--checked' : ''}`}>
                    {taskSelected && <Check size={12} strokeWidth={3} />}
                  </span>
                  <span className="run-eval__drawer-task-name">{task.name}</span>
                </button>
              );
            })}
          </div>
        </aside>
      </div>
    </div>
  );
};

export default DatasetStep;























//types.ts
export type EvalTypeId = 'model' | 'agent' | 'rag';

export interface EvalType {
  id: EvalTypeId;
  title: string;
  desc: string;
  badge: string;
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

/* ---------- API: evaluations ---------- */

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
  subgroup?: string[];
  metrics_config?: Record<string, unknown>;
  selected_metrics: string[];
  dataset_limit?: number;
  selected_category?: string;
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
    setDraft((d) => {
      if (d.type === id) return d;
      const defaults = [...allMetrics, ...(id === 'agent' ? customAgentMetrics : [])];
      return { ...d, type: id, metrics: defaults };
    });
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
    subgroup: draft.subgroup.length > 0 ? draft.subgroup : undefined,
    model_ids: draft.models,
    selected_metrics: draft.metrics,
    dataset_limit: 1,
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
            {step === 2 && <TypeStep value={draft.type} onChange={setType} />}
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
                loading={catalogLoading}
                error={catalogError}
              />
            )}
            {step === 7 && <ReviewStep draft={draft} models={models} benchmarks={benchmarks} />}

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
import { Info } from 'lucide-react';
import { EVAL_TYPES } from '../data';
import type { Benchmark, EvaluationDraft, ModelApi } from '../types';

interface Props {
  draft: EvaluationDraft;
  models: ModelApi[];
  benchmarks: Benchmark[];
}

const ReviewStep: FC<Props> = ({ draft, models, benchmarks }) => {
  const typeInfo = EVAL_TYPES.find((t) => t.id === draft.type);
  const modelNames = draft.models.map((id) => models.find((m) => m.id === id)?.name).filter(Boolean);
  const dataset = benchmarks.find((b) => b.name === draft.dataset);
  const subgroupTasks = dataset?.tasks.filter((t) => draft.subgroup.includes(t.value)) ?? [];

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

      <div className="run-eval__review">
        <div className="run-eval__review-row">
          <span>Name</span>
          <span>{draft.name || '—'}</span>
        </div>
        <div className="run-eval__review-row">
          <span>Type</span>
          <span>{typeInfo?.title ?? '—'}</span>
        </div>
        <div className="run-eval__review-row">
          <span>Models</span>
          <span>{modelNames.length ? modelNames.join(', ') : '—'}</span>
        </div>
        <div className="run-eval__review-row">
          <span>Test Suite</span>
          <span>{dataset?.name ?? '—'}</span>
        </div>
        {subgroupTasks.length > 0 && (
          <div className="run-eval__review-row">
            <span>Subgroup{subgroupTasks.length === 1 ? '' : 's'}</span>
            <span>{subgroupTasks.map((t) => t.name).join(', ')}</span>
          </div>
        )}
        <div className="run-eval__review-row">
          <span>Questions</span>
          <span>{dataset ? dataset.task_count : '—'}</span>
        </div>
        <div className="run-eval__review-divider" />
        <div className="run-eval__review-row run-eval__review-row--highlight">
          <span>Est. Cost</span>
          <span>~${cost.toFixed(2)}</span>
        </div>
        <div className="run-eval__review-row run-eval__review-row--highlight">
          <span>Est. Time</span>
          <span>~{minutes} min</span>
        </div>
      </div>

      <div className="run-eval__hint">
        <Info size={14} />
        <span>Costs are estimates. Actual costs depend on provider pricing.</span>
      </div>
    </div>
  );
};

export default ReviewStep;

















//Runevaluation.tsx
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
    width: 268px;
    background: $bg-subtle;
    border-right: 1px solid $border-subtle;
    padding: 28px 14px;
    display: flex;
    flex-direction: column;
    gap: 2px;
    overflow-y: auto;
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
    border-radius: 0.625rem;
    padding: 10px 12px 22px 12px;
    cursor: pointer;
    transition: background 0.14s ease;

    &::before {
      content: '';
      position: absolute;
      top: 42px;
      left: 29px;
      width: 2px;
      height: calc(100% - 34px);
      background: $border-default;
      transition: background 0.16s ease;
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
    transition: background 0.16s ease, border-color 0.16s ease, color 0.16s ease, box-shadow 0.16s ease;
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
    transition: color 0.16s ease;
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
      box-shadow: 0 0 0 4px $primary-light;
    }

    .run-eval__step-label {
      color: $primary;
    }
  }

  &__step--complete {
    &::before {
      background: $primary;
    }

    .run-eval__step-marker {
      background: $primary-light;
      border-color: $primary;
      color: $primary;
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
    top: 1.125rem;
    right: 1.125rem;
    width: 20px;
    height: 20px;
    border-radius: 50%;
    background: $primary;
    color: #fff;
    display: grid;
    place-items: center;
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
  &__models-layout {
    display: grid;
    grid-template-columns: 15rem 1fr;
    gap: 1.5rem;
    margin-top: 1.5rem;
    align-items: start;
  }

  &__filters {
    border: 1px solid $border-subtle;
    border-radius: 0.75rem;
    padding: 1rem;
    display: flex;
    flex-direction: column;
    gap: 1.125rem;
  }

  &__filters-head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    font-size: 0.875rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__filter-section {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
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
  }

  &__search-bar {
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

  &__models-grid {
    margin-top: 1rem;
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
    font-size: 0.875rem;
    color: $text-secondary;

    span {
      font-weight: 700;
      color: $primary;
    }
  }

  &__metric-group {
    margin-top: 1.5rem;
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
  }

  &__metrics-grid {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 0.625rem;
  }

  &__metric-card {
    position: relative;
    text-align: left;
    padding: 0.75rem 2rem 0.75rem 0.875rem;
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

  &__metric-name {
    display: block;
    font-size: 0.875rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__metric-tooltip {
    display: block;
    margin-top: 0.25rem;
    font-size: 0.78125rem;
    color: $text-tertiary;
    line-height: 1.4;
  }

  /* ---------- review step ---------- */
  &__review {
    margin-top: 1.5rem;
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
