import { useEffect, useMemo, useState, type ComponentType } from 'react';
import { useNavigate } from 'react-router-dom';
import {
  ChevronLeft,
  ChevronRight,
  Check,
  Cpu,
  Bot,
  Database,
  Play,
  Clock3,
  Tag,
  LayoutGrid,
  Plug,
  Target,
  ClipboardCheck,
  Gavel,
  Wallet,
  Layers,
  Loader2,
  Workflow,
  Waypoints,
  Lightbulb,
  Wand2,
  AlertTriangle,
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchProviders } from '../../store/slices/providersSlice';
import { fetchModels } from '../../store/slices/modelsSlice';
import { fetchBenchmarks } from '../../store/slices/benchmarksSlice';
import { registerDataset, resetRegistration } from '../../store/slices/datasetsSlice';
import { fetchMetrics } from '../../store/slices/metricsSlice';
import { launchEvaluation, setDraft } from '../../store/slices/evaluationsSlice';
import type { CreateEvaluationRequest } from '../../types';
import styles from './NewEvaluation.module.scss';

const STEPS = [
  { label: 'Name', description: 'Give your evaluation a name' },
  { label: 'Type', description: 'What kind of AI are you testing' },
  { label: 'Providers', description: 'Choose connected providers' },
  { label: 'Models', description: 'Pick models to compare' },
  { label: 'Test Suite', description: 'Select a benchmark or dataset' },
  { label: 'Metrics', description: 'Choose what to measure' },
  { label: 'Review', description: 'Confirm and launch the run' },
];

const STEP_ICONS: ComponentType<{ size?: number }>[] = [Tag, LayoutGrid, Plug, Cpu, Database, Target, ClipboardCheck];

const TYPE_OPTIONS = [
  {
    v: 'Model',
    icon: Cpu,
    sub: 'Benchmark a general-purpose LLM on standard tasks like reasoning, coding, and knowledge — ideal for comparing raw model quality across providers.',
    variant: '',
  },
  {
    v: 'Agent',
    icon: Bot,
    sub: 'Test an autonomous agent that plans, calls tools, and completes multi-step tasks — measures task completion, not just single-turn output.',
    variant: 'agent',
  },
  {
    v: 'RAG',
    icon: Database,
    sub: 'Evaluate a retrieval-augmented pipeline for grounding accuracy — checks how well answers stay faithful to your retrieved context.',
    variant: 'rag',
  },
];

// Optional agent frameworks, only shown once "Agent" is selected as the type.
const AGENT_FRAMEWORKS = [
  { id: 'hermes', title: 'Hermes', desc: 'Lightweight tool-calling agent runtime' },
  { id: 'langgraph', title: 'LangGraph', desc: 'Graph-based multi-step agent orchestration' },
];

const SUGGESTED_NAMES = [
  'Q3 Model Selection',
  'Support Bot Regression Test',
  'RAG Accuracy Benchmark v2',
  'GPT-4o vs Claude Comparison',
];

const NAMING_TIPS = [
  "Include what you're testing, e.g. a model, a product feature, or a use case.",
  'Add a date or version so you can track changes over time (e.g. "Q3", "v2").',
  'Keep it specific enough to tell apart from similar past evaluations later.',
];

function formatContextWindow(tokens: number): string {
  if (tokens >= 1_000_000) return `${(tokens / 1_000_000).toLocaleString()}M tokens`;
  if (tokens >= 1_000) return `${Math.round(tokens / 1000)}k tokens`;
  return `${tokens} tokens`;
}

function formatPrice(price: number | null | undefined): string {
  return price === null || price === undefined ? '—' : `$${price.toFixed(2)}`;
}

export default function NewEvaluation() {
  const dispatch = useAppDispatch();
  const navigate = useNavigate();
  const [step, setStep] = useState(0);
  const [toast, setToast] = useState(false);
  const [agentFramework, setAgentFramework] = useState<string | null>(null);
  const [selSubgroup, setSelSubgroup] = useState<string[]>([]);
  const [runSamples, setRunSamples] = useState<number>(10);
  const totalSteps = STEPS.length;

  const rawDraft = useAppSelector((s) => s.evaluations.draft);
  const launching = useAppSelector((s) => s.evaluations.launching);
  const launchError = useAppSelector((s) => s.evaluations.launchError);

  const providers = useAppSelector((s) => s.providers.items) ?? [];
  const models = useAppSelector((s) => s.models.items) ?? [];
  const benchmarks = useAppSelector((s) => s.benchmarks.items) ?? [];
  const metrics = useAppSelector((s) => s.metrics) ?? { allMetrics: [], customAgentMetrics: [] };

  // Dataset registration (POST /benchmarks/{name}/register) — required before
  // the user can continue past the Test Suite step. Lives in datasetsSlice.
  const registering = useAppSelector((s) => s.datasets.registering);
  const registerError = useAppSelector((s) => s.datasets.registerError);
  const registeredDatasetId = useAppSelector((s) => s.datasets.registeredDatasetId);

  // Defensive defaults: guards calculations below that run on every render
  // against a draft that hasn't been fully hydrated yet.
  const draft = {
    name: '',
    eval_type: '',
    selProviders: [] as string[],
    selModels: [] as string[],
    selBenchmark: '' as string | undefined,
    selMetrics: [] as string[],
    judgeModelId: undefined as string | undefined,
    ...rawDraft,
  };

  useEffect(() => {
    dispatch(fetchProviders());
    dispatch(fetchModels());
    dispatch(fetchBenchmarks());
    dispatch(fetchMetrics());
  }, [dispatch]);

  // Reset the subgroup/task selection whenever the chosen benchmark changes.
  useEffect(() => {
    setSelSubgroup([]);
  }, [draft.selBenchmark]);

  // A change of benchmark or run-samples count invalidates any previous
  // registration — the user has to register again before continuing.
  useEffect(() => {
    dispatch(resetRegistration());
  }, [dispatch, draft.selBenchmark, runSamples]);

  // Reset the chosen framework if the user switches away from "Agent".
  useEffect(() => {
    if (draft.eval_type !== 'Agent') setAgentFramework(null);
  }, [draft.eval_type]);

  const connectedProviders = providers.filter((p) => p.status === 'connected');
  const availableModels = useMemo(
    () => models.filter((m) => draft.selProviders.includes(m.provider_id)),
    [models, draft.selProviders]
  );
  const activeMetricsList =
    draft.eval_type === 'agent' ? [...metrics.allMetrics, ...metrics.customAgentMetrics] : metrics.allMetrics;

  const toggle = (list: string[], value: string) =>
    list.includes(value) ? list.filter((v) => v !== value) : [...list, value];

  const canGo = () => {
    if (step === 0) return Boolean(draft.name.trim());
    if (step === 1) return Boolean(draft.eval_type);
    if (step === 2) return draft.selProviders.length > 0;
    if (step === 3) return draft.selModels.length > 0;
    if (step === 4) return Boolean(draft.selBenchmark) && Boolean(registeredDatasetId);
    return true;
  };

  const goNext = () => {
    if (!canGo()) return;
    setStep((s) => Math.min(totalSteps - 1, s + 1));
  };
  const goBack = () => setStep((s) => Math.max(0, s - 1));
  const goToStep = (target: number) => {
    if (target < step) setStep(target);
  };

  const suite = benchmarks.find((b) => b.name === draft.selBenchmark);
  const selectedModels = draft.selModels.map((id) => models.find((m) => m.id === id)).filter(Boolean) as typeof models;
  const judgeModel = draft.judgeModelId ? models.find((m) => m.id === draft.judgeModelId) : null;

  const { estCost, estMinutes } = useMemo(() => {
    const questions = suite?.task_count ?? 0;
    const modelCount = draft.selModels.length || 1;
    return {
      estCost: questions * modelCount * 0.0009,
      estMinutes: Math.max(1, Math.round((questions * modelCount) / 180)),
    };
  }, [suite, draft.selModels.length]);

  // POST /benchmarks/{benchmark_name}/register?run_samples={run_samples} (datasetsSlice)
  const handleRegister = () => {
    if (!draft.selBenchmark) return;
    dispatch(registerDataset({ benchmarkName: draft.selBenchmark, runSamples }));
  };

  const launch = async () => {
    const benchmark = benchmarks.find((b) => b.name === draft.selBenchmark);
    const judgeModelObj = draft.judgeModelId ? models.find((m) => m.id === draft.judgeModelId) : undefined;

    const payload: CreateEvaluationRequest & { datasets?: { dataset_id: string }[] } = {
      name: draft.name,
      eval_type: draft.eval_type.toLowerCase(),
      dataset_id: registeredDatasetId || '',
      datasets: registeredDatasetId ? [{ dataset_id: registeredDatasetId }] : [],
      benchmark: draft.selBenchmark || undefined,
      model_ids: draft.selModels,
      selected_metrics: draft.selMetrics,
      run_samples: runSamples,
      selected_category: selSubgroup.length > 0 ? selSubgroup : benchmark ? [benchmark.type] : undefined,
      ...(draft.judgeModelId
        ? {
            judge_config: {
              model_id: draft.judgeModelId,
              base_url: judgeModelObj?.base_url || '',
              api_key: draft.judgeModelId,
            },
          }
        : {}),
    };

    const result = await dispatch(launchEvaluation(payload));
    if (launchEvaluation.fulfilled.match(result)) {
      setToast(true);
      setTimeout(() => {
        setToast(false);
        navigate('/app/history');
      }, 2000);
    }
  };

  const progressPct = Math.round((step / (totalSteps - 1)) * 100);

  return (
    <div className="page-enter" style={{ height: '100%', display: 'flex', flexDirection: 'column' }}>
      <div className={styles.page}>
        <div className={styles.wiz__header}>
          <div>
            <p className={styles['wiz__header-eyebrow']}>Create evaluation</p>
            <h1>New Evaluation</h1>
            <p className={styles['wiz-sub']} style={{ marginBottom: 0 }}>
              Set up and launch a structured model evaluation
            </p>
          </div>
          <div className={styles['wiz__header-meta']}>
            <Clock3 size={13} />
            ~5 min guided setup
          </div>
        </div>

        <div className={styles['wiz-shell']}>
          <aside className={styles.wiz__sidebar}>
            <div className={styles['wiz__sidebar-progress']}>
              <div className={styles['wiz__sidebar-progress-head']}>
                <span>
                  Step {step + 1} of {totalSteps}
                </span>
                <span>{progressPct}%</span>
              </div>
              <div className={styles['wiz__sidebar-progress-track']}>
                <div className={styles['wiz__sidebar-progress-fill']} style={{ width: `${progressPct}%` }} />
              </div>
            </div>

            {STEPS.map((s, i) => {
              const state = i === step ? 'active' : i < step ? 'complete' : 'upcoming';
              const Icon = STEP_ICONS[i];
              return (
                <button
                  key={s.label}
                  type="button"
                  className={`${styles.wiz__step} ${styles[`wiz__step--${state}`]}`}
                  onClick={() => goToStep(i)}
                  disabled={i > step}
                >
                  <span className={styles['wiz__step-marker']}>
                    {state === 'complete' ? <Check size={14} strokeWidth={3} /> : <Icon size={15} />}
                  </span>
                  <span className={styles['wiz__step-text']}>
                    <span className={styles['wiz__step-label']}>{s.label}</span>
                    <span className={styles['wiz__step-desc']}>{s.description}</span>
                  </span>
                </button>
              );
            })}
          </aside>

          <div className={styles.wiz__content}>
            <p className={styles['wiz__step-kicker']}>
              Step {step + 1} of {totalSteps}
            </p>

            <div className={styles.wiz__body}>
              {step === 0 && (
                <>
                  <h2>Name your evaluation</h2>
                  <p className={styles['wiz-sub']}>Give it a recognizable name so you can find it later.</p>

                  <div className={styles.wiz__field}>
                    <label className={styles.wiz__label}>Evaluation Name</label>
                    <div className={styles['wiz__input-icon-wrap']}>
                      <Tag size={16} />
                      <input
                        className={styles.wiz__input}
                        placeholder="e.g. Q3 Model Selection"
                        value={draft.name}
                        onChange={(e) => dispatch(setDraft({ name: e.target.value }))}
                        autoFocus
                      />
                    </div>
                  </div>

                  <div className={styles['wiz__suggestions']}>
                    <p className={styles['wiz__suggestions-title']}>Quick start</p>
                    <p className={styles['wiz__suggestions-sub']}>Not sure what to call it? Start from one of these.</p>
                    <div className={styles['wiz__suggestions-grid']}>
                      {SUGGESTED_NAMES.map((s) => (
                        <button
                          key={s}
                          type="button"
                          className={styles['wiz__suggestion-card']}
                          onClick={() => dispatch(setDraft({ name: s }))}
                        >
                          <span className={styles['wiz__suggestion-icon']}>
                            <Wand2 size={14} />
                          </span>
                          <span className={styles['wiz__suggestion-text']}>{s}</span>
                          <span className={styles['wiz__suggestion-use']}>Use</span>
                        </button>
                      ))}
                    </div>
                  </div>

                  <div className={styles.wiz__tips}>
                    <div className={styles['wiz__tips-icon']}>
                      <Lightbulb size={16} strokeWidth={2} />
                    </div>
                    <div>
                      <p className={styles['wiz__tips-title']}>Tips for a good name</p>
                      <ul className={styles['wiz__tips-list']}>
                        {NAMING_TIPS.map((tip) => (
                          <li key={tip}>{tip}</li>
                        ))}
                      </ul>
                    </div>
                  </div>

                  <div className={styles.wiz__roadmap}>
                    <p className={styles['wiz__roadmap-title']}>What you'll set up next</p>
                    <p className={styles['wiz__roadmap-sub']}>A quick look at the rest of the flow before you continue.</p>
                    <div className={styles['wiz__roadmap-grid']}>
                      {STEPS.slice(1).map((s, i) => {
                        const Icon = STEP_ICONS[i + 1];
                        return (
                          <div className={styles['wiz__roadmap-card']} key={s.label}>
                            <span className={styles['wiz__roadmap-num']}>{String(i + 2).padStart(2, '0')}</span>
                            <span className={styles['wiz__roadmap-icon']}>
                              <Icon size={15} />
                            </span>
                            <span className={styles['wiz__roadmap-text']}>
                              <span className={styles['wiz__roadmap-label']}>{s.label}</span>
                              <span className={styles['wiz__roadmap-desc']}>{s.description}</span>
                            </span>
                          </div>
                        );
                      })}
                    </div>
                  </div>
                </>
              )}

              {step === 1 && (
                <>
                  <h2>Choose evaluation type</h2>
                  <p className={styles['wiz-sub']}>Pick what kind of AI you're testing.</p>

                  <div className={styles['wiz__type-grid']}>
                    {TYPE_OPTIONS.map((o) => {
                      const Icon = o.icon;
                      const selected = draft.eval_type === o.v;
                      return (
                        <button
                          key={o.v}
                          type="button"
                          className={`${styles['wiz__type-card']} ${selected ? styles['wiz__type-card--selected'] : ''}`}
                          onClick={() => dispatch(setDraft({ eval_type: o.v }))}
                        >
                          <span
                            className={`${styles['wiz__type-icon']} ${o.variant ? styles[`wiz__type-icon--${o.variant}`] : ''}`}
                          >
                            <Icon size={18} />
                          </span>
                          <span className={styles['wiz__type-content']}>
                            <span className={styles['wiz__type-title']}>{o.v}</span>
                            <span className={styles['wiz__type-desc']}>{o.sub}</span>
                          </span>
                          {selected && (
                            <span className={styles['wiz__type-check']}>
                              <Check size={13} strokeWidth={2.75} />
                            </span>
                          )}
                        </button>
                      );
                    })}
                  </div>

                  {draft.eval_type === 'Agent' && (
                    <div className={styles['wiz__framework-section']}>
                      <label className={styles.wiz__label}>
                        <Workflow size={13} strokeWidth={2.25} />
                        Agent Framework <span className="opt">(optional)</span>
                      </label>
                      <p className={styles['wiz__framework-hint']}>
                        Tell us which framework the agent runs on, if applicable.
                      </p>
                      <div className={styles['wiz__framework-grid']}>
                        {AGENT_FRAMEWORKS.map((f) => {
                          const selected = agentFramework === f.id;
                          return (
                            <button
                              key={f.id}
                              type="button"
                              className={`${styles['wiz__type-card']} ${styles['wiz__type-card--framework']} ${
                                selected ? styles['wiz__type-card--selected'] : ''
                              }`}
                              onClick={() => setAgentFramework(selected ? null : f.id)}
                            >
                              <span className={styles['wiz__type-icon']}>
                                <Waypoints size={16} />
                              </span>
                              <span className={styles['wiz__type-content']}>
                                <span className={styles['wiz__type-title']}>{f.title}</span>
                                <span className={styles['wiz__type-desc']}>{f.desc}</span>
                              </span>
                              {selected && (
                                <span className={styles['wiz__type-check']}>
                                  <Check size={12} strokeWidth={2.75} />
                                </span>
                              )}
                            </button>
                          );
                        })}
                      </div>
                    </div>
                  )}
                </>
              )}

              {step === 2 && (
                <>
                  <h2>Select providers</h2>
                  <p className={styles['wiz-sub']}>Choose which connected providers to draw models from.</p>
                  <div className={styles['wiz__grid-scroll']}>
                    <div className={styles.wiz__grid}>
                      {connectedProviders.map((p) => {
                        const selected = draft.selProviders.includes(p.id);
                        return (
                          <button
                            key={p.id}
                            type="button"
                            className={`${styles.wiz__card} ${selected ? styles['wiz__card--selected'] : ''}`}
                            onClick={() => dispatch(setDraft({ selProviders: toggle(draft.selProviders, p.id) }))}
                          >
                            <span className={styles['wiz__card-icon']}>
                              <Plug size={15} />
                            </span>
                            <span className={styles['wiz__card-text']}>
                              <span className={styles['wiz__card-name']}>{p.name}</span>
                              <span className={styles['wiz__card-sub']}>{p.model_count} models available</span>
                              <span className={styles['wiz__provider-status']}>Connected</span>
                            </span>
                            {selected && (
                              <span className={styles['wiz__card-check']}>
                                <Check size={11} strokeWidth={2.75} />
                              </span>
                            )}
                          </button>
                        );
                      })}
                      {connectedProviders.length === 0 && (
                        <p className={styles.wiz__empty}>No connected providers yet — connect one from the Providers page first.</p>
                      )}
                    </div>
                  </div>
                </>
              )}

              {step === 3 && (
                <>
                  <h2>Choose models</h2>
                  <p className={styles['wiz-sub']}>Pick which models to include in this evaluation.</p>
                  {availableModels.length > 0 ? (
                    <div className={styles['wiz__grid-scroll']}>
                      <div className={styles['wiz__models-grid']}>
                        {availableModels.map((m) => {
                          const selected = draft.selModels.includes(m.id);
                          const caps = (m as any).capabilities as string[] | undefined;
                          const inputPrice = (m as any).input_price as number | null | undefined;
                          const outputPrice = (m as any).output_price as number | null | undefined;
                          const accuracy = (m as any).accuracy_score as number | null | undefined;
                          return (
                            <button
                              key={m.id}
                              type="button"
                              className={`${styles['wiz__model-card']} ${selected ? styles['wiz__model-card--selected'] : ''}`}
                              onClick={() => dispatch(setDraft({ selModels: toggle(draft.selModels, m.id) }))}
                            >
                              <div className={styles['wiz__model-top']}>
                                <span className={styles['wiz__model-name']}>{m.name}</span>
                                {selected && (
                                  <span className={styles['wiz__card-check']} style={{ position: 'static' }}>
                                    <Check size={11} strokeWidth={2.75} />
                                  </span>
                                )}
                              </div>
                              <span className={styles['wiz__model-provider']}>
                                {providers.find((p) => p.id === m.provider_id)?.name ?? m.provider_id}
                              </span>
                              {caps && caps.length > 0 && (
                                <div className={styles['wiz__model-caps']}>
                                  {caps.slice(0, 3).map((c) => (
                                    <span key={c} className={styles['wiz__model-cap-chip']}>
                                      {c}
                                    </span>
                                  ))}
                                </div>
                              )}
                              <div className={styles['wiz__model-meta']}>
                                <span>{formatContextWindow(m.context_window)}</span>
                                {(inputPrice !== undefined || outputPrice !== undefined) && (
                                  <span>
                                    {formatPrice(inputPrice)} in · {formatPrice(outputPrice)} out /1M
                                  </span>
                                )}
                                {accuracy !== undefined && accuracy !== null && <span>Accuracy {accuracy.toFixed(1)}%</span>}
                              </div>
                            </button>
                          );
                        })}
                      </div>
                    </div>
                  ) : (
                    <p className={styles.wiz__empty}>Select providers first to see available models.</p>
                  )}
                </>
              )}

              {step === 4 && (
                <>
                  <h2>Pick a test suite</h2>
                  <p className={styles['wiz-sub']}>Select the benchmark to evaluate against.</p>

                  <div className={styles['wiz__dataset-layout']}>
                    <div className={styles['wiz__dataset-grid-scroll']}>
                      <div className={styles['wiz__dataset-grid']}>
                        {benchmarks.map((b) => {
                          const selected = draft.selBenchmark === b.name;
                          return (
                            <button
                              key={b.name}
                              type="button"
                              className={`${styles['wiz__dataset-card']} ${selected ? styles['wiz__dataset-card--selected'] : ''}`}
                              onClick={() => dispatch(setDraft({ selBenchmark: b.name }))}
                            >
                              <div className={styles['wiz__dataset-top']}>
                                <span className={styles['wiz__dataset-top-left']}>
                                  <span className={styles['wiz__dataset-icon']}>
                                    <Database size={14} />
                                  </span>
                                  <span className={styles['wiz__dataset-name']}>{b.name}</span>
                                </span>
                                {selected && (
                                  <span className={styles['wiz__card-check']} style={{ position: 'static' }}>
                                    <Check size={11} strokeWidth={2.75} />
                                  </span>
                                )}
                              </div>
                              <p className={styles['wiz__dataset-desc']}>{b.description}</p>
                              <div className={styles['wiz__dataset-meta']}>
                                <span className={`${styles.wiz__chip} ${styles['wiz__chip--static']}`}>{b.type}</span>
                                <span>{b.task_count.toLocaleString()} tasks</span>
                              </div>
                            </button>
                          );
                        })}
                        {benchmarks.length === 0 && <p className={styles.wiz__empty}>No test suites available.</p>}
                      </div>
                    </div>

                    <aside className={styles['wiz__subgroup-panel']}>
                      <div className={styles['wiz__subgroup-panel-head']}>
                        <p className={styles['wiz__subgroup-panel-title']}>
                          <Layers size={13} strokeWidth={2.25} /> Subgroups
                        </p>
                        <p className={styles['wiz__subgroup-panel-sub']}>
                          {suite ? `Optionally narrow "${suite.name}" to specific tasks.` : 'Select a test suite to see its subgroups.'}
                        </p>
                      </div>
                      <div className={styles['wiz__subgroup-panel-scroll']}>
                        {!suite && <p className={styles['wiz__subgroup-empty']}>No test suite selected yet.</p>}
                        {suite && suite.tasks.length === 0 && (
                          <p className={styles['wiz__subgroup-empty']}>This test suite has no subgroups.</p>
                        )}
                        {suite &&
                          suite.tasks.map((t) => {
                            const checked = selSubgroup.includes(t.value);
                            return (
                              <button
                                key={t.value}
                                type="button"
                                className={`${styles['wiz__subgroup-row']} ${checked ? styles['wiz__subgroup-row--selected'] : ''}`}
                                onClick={() => setSelSubgroup((prev) => toggle(prev, t.value))}
                              >
                                <span className={`${styles.wiz__checkbox} ${checked ? styles['wiz__checkbox--checked'] : ''}`}>
                                  {checked && <Check size={11} strokeWidth={3} />}
                                </span>
                                <span className={styles['wiz__subgroup-row-name']}>{t.name}</span>
                              </button>
                            );
                          })}
                      </div>
                    </aside>
                  </div>

                  {registeredDatasetId && (
                    <p className={styles['wiz__register-status']} data-state="success">
                      <Check size={13} strokeWidth={2.75} /> Dataset registered — you can continue.
                    </p>
                  )}
                  {registerError && (
                    <p className={styles['wiz__register-status']} data-state="error">
                      <AlertTriangle size={13} strokeWidth={2.25} /> {registerError}
                    </p>
                  )}
                  {!registeredDatasetId && !registerError && draft.selBenchmark && (
                    <p className={styles['wiz__register-status']} data-state="pending">
                      Register this dataset to continue.
                    </p>
                  )}
                </>
              )}

              {step === 5 && (
                <>
                  <h2>Configure metrics</h2>
                  <p className={styles['wiz-sub']}>Choose which metrics to measure.</p>

                  <div className={styles['wiz__field']} style={{ maxWidth: 220 }}>
                    <label className={styles.wiz__label}>Run Samples</label>
                    <input
                      type="number"
                      min={0}
                      className={styles.wiz__input}
                      value={runSamples}
                      onChange={(e) => {
                        const val = e.target.value === '' ? 0 : Math.max(0, Number(e.target.value));
                        setRunSamples(Number.isNaN(val) ? 0 : val);
                      }}
                    />
                  </div>

                  <div className={styles['wiz__metrics-toolbar']}>
                    <span className={styles['wiz__metrics-count']}>
                      <strong>{draft.selMetrics.length}</strong> selected
                    </span>
                    <div className={styles['wiz__metrics-actions']}>
                      <button
                        type="button"
                        className={styles['wiz__link-btn']}
                        onClick={() => dispatch(setDraft({ selMetrics: [...activeMetricsList] }))}
                      >
                        Select all
                      </button>
                      <button type="button" className={styles['wiz__link-btn']} onClick={() => dispatch(setDraft({ selMetrics: [] }))}>
                        Unselect all
                      </button>
                    </div>
                  </div>

                  <div className={styles['wiz__metrics-layout']}>
                    <div className={styles['wiz__metrics-main-scroll']}>
                      <div className={styles['wiz__metrics-grid']}>
                        {activeMetricsList.map((m) => {
                          const selected = draft.selMetrics.includes(m);
                          return (
                            <button
                              key={m}
                              type="button"
                              className={`${styles['wiz__metric-card']} ${selected ? styles['wiz__metric-card--selected'] : ''}`}
                              onClick={() => dispatch(setDraft({ selMetrics: toggle(draft.selMetrics, m) }))}
                            >
                              <span className={styles['wiz__metric-name']}>{m}</span>
                              {selected && (
                                <span className={styles['wiz__metric-check']}>
                                  <Check size={11} strokeWidth={2.75} />
                                </span>
                              )}
                            </button>
                          );
                        })}
                        {activeMetricsList.length === 0 && <p className={styles.wiz__empty}>No metrics available.</p>}
                      </div>
                    </div>

                    <aside className={styles['wiz__judge-panel']}>
                      <p className={styles['wiz__judge-title']}>
                        <Gavel size={13} strokeWidth={2.25} /> Judge Model
                      </p>
                      <p className={styles['wiz__judge-hint']}>Pick any available model to grade the other models' responses.</p>

                      <div className={styles['wiz__judge-panel-scroll']}>
                        {models.filter((m) => m.is_active).length === 0 ? (
                          <div className={styles['wiz__judge-empty']}>No models are available yet.</div>
                        ) : (
                          models
                            .filter((m) => m.is_active)
                            .map((m) => {
                              const isJudge = draft.judgeModelId === m.id;
                              return (
                                <button
                                  key={m.id}
                                  type="button"
                                  className={`${styles['wiz__judge-row']} ${isJudge ? styles['wiz__judge-row--selected'] : ''}`}
                                  onClick={() => dispatch(setDraft({ judgeModelId: isJudge ? undefined : m.id }))}
                                >
                                  <span className={`${styles.wiz__radio} ${isJudge ? styles['wiz__radio--checked'] : ''}`} />
                                  <span className={styles['wiz__judge-row-text']}>
                                    <span className={styles['wiz__judge-row-name']}>{m.name}</span>
                                    <span className={styles['wiz__judge-row-meta']}>
                                      {providers.find((p) => p.id === m.provider_id)?.name ?? m.provider_id}
                                    </span>
                                  </span>
                                </button>
                              );
                            })
                        )}
                      </div>
                    </aside>
                  </div>
                </>
              )}

              {step === 6 && (
                <>
                  <h2>Review &amp; Launch</h2>
                  <p className={styles['wiz-sub']}>Confirm your evaluation setup.</p>

                  <div className={styles['wiz__review-stats']}>
                    <div className={styles['wiz__review-stat']}>
                      <span className={styles['wiz__review-stat-label']}>
                        <Wallet size={12} strokeWidth={2} style={{ marginRight: 4, verticalAlign: -2 }} />
                        Est. Cost
                      </span>
                      <span className={styles['wiz__review-stat-value']}>~${estCost.toFixed(2)}</span>
                    </div>
                    <div className={styles['wiz__review-stat']}>
                      <span className={styles['wiz__review-stat-label']}>
                        <Clock3 size={12} strokeWidth={2} style={{ marginRight: 4, verticalAlign: -2 }} />
                        Est. Time
                      </span>
                      <span className={styles['wiz__review-stat-value']}>~{estMinutes} min</span>
                    </div>
                    <div className={styles['wiz__review-stat']}>
                      <span className={styles['wiz__review-stat-label']}>
                        <Layers size={12} strokeWidth={2} style={{ marginRight: 4, verticalAlign: -2 }} />
                        Questions
                      </span>
                      <span className={styles['wiz__review-stat-value']}>{suite ? suite.task_count.toLocaleString() : '—'}</span>
                    </div>
                  </div>

                  <div className={styles['wiz__review-section']}>
                    <p className={styles['wiz__review-section-title']}>
                      <Tag size={11} strokeWidth={2.25} /> Overview
                    </p>
                    <div className={styles.wiz__review}>
                      <div className={styles['wiz__review-row']}>
                        <span>Name</span>
                        <span>{draft.name || '—'}</span>
                      </div>
                      <div className={styles['wiz__review-row']}>
                        <span>Type</span>
                        <span>{draft.eval_type || '—'}</span>
                      </div>
                      {agentFramework && (
                        <div className={styles['wiz__review-row']}>
                          <span>Agent Framework</span>
                          <span>{AGENT_FRAMEWORKS.find((f) => f.id === agentFramework)?.title}</span>
                        </div>
                      )}
                      <div className={styles['wiz__review-row']}>
                        <span>Providers</span>
                        <span>{draft.selProviders.map((id) => providers.find((p) => p.id === id)?.name || id).join(', ') || '—'}</span>
                      </div>
                    </div>
                  </div>

                  <div className={styles['wiz__review-section']}>
                    <p className={styles['wiz__review-section-title']}>
                      <Cpu size={11} strokeWidth={2.25} /> Models ({selectedModels.length})
                    </p>
                    {selectedModels.length > 0 ? (
                      <div className={styles.wiz__grid} style={{ marginTop: '0.75rem' }}>
                        {selectedModels.map((m) => (
                          <div key={m!.id} className={styles.wiz__card} style={{ cursor: 'default' }}>
                            <span className={styles['wiz__card-icon']}>
                              <Cpu size={14} />
                            </span>
                            <span className={styles['wiz__card-text']}>
                              <span className={styles['wiz__card-name']}>{m!.name}</span>
                              <span className={styles['wiz__card-sub']}>{providers.find((p) => p.id === m!.provider_id)?.name || m!.provider_id}</span>
                            </span>
                          </div>
                        ))}
                      </div>
                    ) : (
                      <p className={styles.wiz__empty}>No models selected.</p>
                    )}
                  </div>

                  <div className={styles['wiz__review-section']}>
                    <p className={styles['wiz__review-section-title']}>
                      <Database size={11} strokeWidth={2.25} /> Test Suite
                    </p>
                    <div className={styles.wiz__review}>
                      <div className={styles['wiz__review-row']}>
                        <span>Suite</span>
                        <span>{suite?.name ?? '—'}</span>
                      </div>
                      <div className={styles['wiz__review-row']}>
                        <span>Run Samples</span>
                        <span>{runSamples}</span>
                      </div>
                      {suite?.description && (
                        <div className={styles['wiz__review-row']}>
                          <span>Description</span>
                          <span>{suite.description}</span>
                        </div>
                      )}
                      {selSubgroup.length > 0 && (
                        <div className={styles['wiz__review-row']}>
                          <span>Subgroups</span>
                          <span>{suite?.tasks.filter((t) => selSubgroup.includes(t.value)).map((t) => t.name).join(', ')}</span>
                        </div>
                      )}
                    </div>
                  </div>

                  <div className={styles['wiz__review-section']}>
                    <p className={styles['wiz__review-section-title']}>
                      <Target size={11} strokeWidth={2.25} /> Metrics ({draft.selMetrics.length})
                    </p>
                    {draft.selMetrics.length > 0 ? (
                      <div style={{ display: 'flex', flexWrap: 'wrap', gap: '0.5rem', marginTop: '0.75rem' }}>
                        {draft.selMetrics.map((m) => (
                          <span key={m} className={styles.wiz__chip}>
                            {m}
                          </span>
                        ))}
                      </div>
                    ) : (
                      <p className={styles.wiz__empty}>No metrics selected.</p>
                    )}
                  </div>

                  {judgeModel && (
                    <div className={styles['wiz__review-section']}>
                      <p className={styles['wiz__review-section-title']}>
                        <Gavel size={11} strokeWidth={2.25} /> Judge Model
                      </p>
                      <div className={styles.wiz__review}>
                        <div className={styles['wiz__review-row']}>
                          <span>Model</span>
                          <span>{judgeModel.name}</span>
                        </div>
                      </div>
                    </div>
                  )}

                  {launchError && <p className={styles.wiz__error}>{launchError}</p>}
                </>
              )}
            </div>

            <div className={styles.wiz__nav}>
              <button
                type="button"
                className="btn btn-ghost"
                onClick={() => (step > 0 ? goBack() : navigate('/app/dashboard'))}
                disabled={launching}
              >
                <ChevronLeft size={16} /> {step === 0 ? 'Cancel' : 'Back'}
              </button>

              <div style={{ display: 'flex', alignItems: 'center', gap: 10 }}>
                {step === 4 && (
                  <button
                    type="button"
                    className="btn btn-ghost"
                    onClick={handleRegister}
                    disabled={!draft.selBenchmark || registering}
                  >
                    {registering ? (
                      <>
                        <Loader2 size={16} className={styles.wiz__spin} /> Registering…
                      </>
                    ) : registeredDatasetId ? (
                      <>
                        <Check size={16} /> Registered
                      </>
                    ) : (
                      'Register'
                    )}
                  </button>
                )}

                {step < totalSteps - 1 ? (
                  <button type="button" className="btn btn-ind" onClick={goNext} disabled={!canGo()}>
                    Continue <ChevronRight size={16} />
                  </button>
                ) : (
                  <button type="button" className="btn btn-ind" onClick={launch} disabled={launching}>
                    {launching ? (
                      <>
                        <Loader2 size={16} className={styles.wiz__spin} /> Launching…
                      </>
                    ) : (
                      <>
                        <Play size={16} /> Launch Evaluation
                      </>
                    )}
                  </button>
                )}
              </div>
            </div>
          </div>
        </div>
      </div>

      {toast && (
        <div className="toast">
          <div className={styles['toast__icon']}>
            <Check size={18} color="#10B981" />
          </div>
          <div>
            <div style={{ fontWeight: 700, fontSize: 14 }}>Evaluation launched</div>
            <div style={{ fontSize: 12, color: '#6B7280' }}>You'll find it in Evaluations once it completes.</div>
          </div>
        </div>
      )}
    </div>
  );
}
