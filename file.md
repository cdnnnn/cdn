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
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchProviders } from '../../store/slices/providersSlice';
import { fetchModels } from '../../store/slices/modelsSlice';
import { fetchBenchmarks } from '../../store/slices/benchmarksSlice';
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
  { v: 'Model', icon: Cpu, sub: 'General-purpose model' },
  { v: 'Agent', icon: Bot, sub: 'Autonomous agent' },
  { v: 'RAG', icon: Database, sub: 'Retrieval-augmented' },
];

// Optional agent frameworks, only shown once "Agent" is selected as the type.
const AGENT_FRAMEWORKS = [
  { id: 'hermes', title: 'Hermes', desc: 'Lightweight tool-calling agent runtime' },
  { id: 'langgraph', title: 'LangGraph', desc: 'Graph-based multi-step agent orchestration' },
];

export default function NewEvaluation() {
  const dispatch = useAppDispatch();
  const navigate = useNavigate();
  const [step, setStep] = useState(0);
  const [toast, setToast] = useState(false);
  const [agentFramework, setAgentFramework] = useState<string | null>(null);
  const [selSubgroup, setSelSubgroup] = useState<string[]>([]);
  const totalSteps = STEPS.length;

  const rawDraft = useAppSelector((s) => s.evaluations.draft);
  const launching = useAppSelector((s) => s.evaluations.launching);
  const launchError = useAppSelector((s) => s.evaluations.launchError);

  const providers = useAppSelector((s) => s.providers.items) ?? [];
  const models = useAppSelector((s) => s.models.items) ?? [];
  const benchmarks = useAppSelector((s) => s.benchmarks.items) ?? [];
  const metrics = useAppSelector((s) => s.metrics) ?? { allMetrics: [], customAgentMetrics: [] };

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
    if (step === 4) return Boolean(draft.selBenchmark);
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

  const launch = async () => {
    const benchmark = benchmarks.find((b) => b.name === draft.selBenchmark);
    const payload: CreateEvaluationRequest = {
      name: draft.name,
      eval_type: draft.eval_type.toLowerCase(),
      dataset_id: benchmark?.huggingface_dataset || '',
      benchmark: draft.selBenchmark || undefined,
      model_ids: draft.selModels,
      selected_metrics: draft.selMetrics,
      selected_category: selSubgroup.length > 0 ? selSubgroup : benchmark ? [benchmark.type] : undefined,
      ...(draft.judgeModelId ? { judge_config: { model_id: draft.judgeModelId, base_url: '', api_key: '' } } : {}),
    };

    const result = await dispatch(launchEvaluation(payload));
    if (launchEvaluation.fulfilled.match(result)) {
      setToast(true);
      setTimeout(() => {
        setToast(false);
        navigate('/app/evaluations');
      }, 2000);
    }
  };

  const progressPct = Math.round((step / (totalSteps - 1)) * 100);

  return (
    <div className="page-enter">
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
                    <input
                      className={styles.wiz__input}
                      placeholder="e.g. Q3 Model Selection"
                      value={draft.name}
                      onChange={(e) => dispatch(setDraft({ name: e.target.value }))}
                      autoFocus
                    />
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
                          <span className={styles['wiz__type-icon']}>
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
                      <div className={styles.wiz__grid}>
                        {availableModels.map((m) => {
                          const selected = draft.selModels.includes(m.id);
                          return (
                            <button
                              key={m.id}
                              type="button"
                              className={`${styles.wiz__card} ${selected ? styles['wiz__card--selected'] : ''}`}
                              onClick={() => dispatch(setDraft({ selModels: toggle(draft.selModels, m.id) }))}
                            >
                              <span className={styles['wiz__card-icon']}>
                                <Cpu size={15} />
                              </span>
                              <span className={styles['wiz__card-text']}>
                                <span className={styles['wiz__card-name']}>{m.name}</span>
                                <span className={styles['wiz__card-sub']}>{m.context_window.toLocaleString()} ctx</span>
                              </span>
                              {selected && (
                                <span className={styles['wiz__card-check']}>
                                  <Check size={11} strokeWidth={2.75} />
                                </span>
                              )}
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
                                <span className={styles['wiz__dataset-name']}>{b.name}</span>
                                {selected && (
                                  <span className={styles['wiz__card-check']} style={{ position: 'static' }}>
                                    <Check size={11} strokeWidth={2.75} />
                                  </span>
                                )}
                              </div>
                              <p className={styles['wiz__dataset-desc']}>{b.description}</p>
                              <div className={styles['wiz__dataset-meta']}>
                                <span className={styles.wiz__chip}>{b.type}</span>
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
                </>
              )}

              {step === 5 && (
                <>
                  <h2>Configure metrics</h2>
                  <p className={styles['wiz-sub']}>Choose which metrics to measure.</p>

                  <div className={styles['wiz__metrics-toolbar']}>
                    <span className={styles['wiz__metrics-count']}>
                      <strong>{draft.selMetrics.length}</strong> selected
                    </span>
                    <div style={{ display: 'flex', gap: 12 }}>
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
                      <div className={styles.wiz__grid}>
                        {activeMetricsList.map((m) => {
                          const selected = draft.selMetrics.includes(m);
                          return (
                            <button
                              key={m}
                              type="button"
                              className={`${styles.wiz__card} ${selected ? styles['wiz__card--selected'] : ''}`}
                              onClick={() => dispatch(setDraft({ selMetrics: toggle(draft.selMetrics, m) }))}
                            >
                              <span className={styles['wiz__card-icon']}>
                                <Target size={14} />
                              </span>
                              <span className={styles['wiz__card-text']}>
                                <span className={styles['wiz__card-name']}>{m}</span>
                              </span>
                              {selected && (
                                <span className={styles['wiz__card-check']}>
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





























@use '../../styles/_variables' as *;

// ---------------------------------------------------------------------------
// Local aliases: map this component's design tokens onto the shared theme
// tokens defined in _variables.scss.
// ---------------------------------------------------------------------------
$primary: $indigo;
$primary-hover: $indigo-dark;
$primary-light: $indigo-pale;
$bg-main: $surface;
$bg-subtle: $surface-alt;
$bg-inset: $surface-hover;
$border-subtle: $border-light;
$border-default: $border;
$border-strong: rgba(17, 24, 39, 0.16);
$text-tertiary: $text-muted;
$danger: $red;
$danger-subtle: $red-pale;
$success: $emerald;
$success-subtle: $emerald-pale;
$shadow-sm: $shadow-2;
$shadow-md: $shadow-3;

.page {
  padding: 28px 40px 40px;
}

.wiz {
  &__header {
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding-bottom: 18px;
    margin-bottom: 20px;
    border-bottom: 1px solid $border-subtle;
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

  /* ---------- wizard shell ---------- */
  &-shell {
    position: relative;
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 20px;
    box-shadow: $shadow-md;
    overflow: hidden;
    display: flex;
    min-height: 600px;
  }

  /* ---------- sidebar / vertical stepper ---------- */
  &__sidebar {
    flex-shrink: 0;
    width: 280px;
    background: $bg-subtle;
    border-right: 1px solid $border-subtle;
    padding: 24px 14px 28px;
    display: flex;
    flex-direction: column;
    gap: 2px;
    overflow-y: auto;
  }

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
    padding: 10px 12px 20px 12px;
    cursor: pointer;
    transition: background 0.16s ease, transform 0.16s ease;

    &::before {
      content: '';
      position: absolute;
      top: 40px;
      left: 27px;
      width: 2px;
      height: calc(100% - 32px);
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
  }

  &__step-marker {
    position: relative;
    z-index: 1;
    flex-shrink: 0;
    width: 32px;
    height: 32px;
    border-radius: 50%;
    display: grid;
    place-items: center;
    background: $bg-main;
    border: 1.5px solid $border-default;
    color: $text-tertiary;
    font-family: $font-mono;
    font-size: 0.75rem;
    font-weight: 700;
    transition: background 0.18s ease, border-color 0.18s ease, color 0.18s ease, box-shadow 0.18s ease;
  }

  &__step-text {
    display: flex;
    flex-direction: column;
    gap: 2px;
    padding-top: 4px;
    min-width: 0;
  }

  &__step-label {
    font-size: 0.84375rem;
    font-weight: 700;
    color: $text-primary;
    transition: color 0.18s ease;
  }

  &__step-desc {
    font-size: 0.71875rem;
    color: $text-tertiary;
    line-height: 1.35;
  }

  &__step--active {
    background: $bg-main;
    box-shadow: $shadow-sm;

    .wiz__step-marker {
      background: $primary;
      border-color: $primary;
      color: #fff;
      box-shadow: 0 0 0 5px $primary-light;
    }

    .wiz__step-label {
      color: $primary;
    }
  }

  &__step--complete {
    &::before {
      background: linear-gradient(180deg, $primary 0%, $primary-hover 100%);
    }

    .wiz__step-marker {
      background: $primary-light;
      border-color: $primary;
      color: $primary;
    }

    &:not(:disabled):hover {
      background: rgba(0, 0, 0, 0.02);
    }
  }

  &__step--upcoming {
    .wiz__step-label {
      color: $text-secondary;
    }

    .wiz__step-desc {
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
    min-height: 0;
  }

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

  &__body {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding-right: 4px;
    margin-right: -4px;
  }

  &__body h2 {
    font-size: 19px;
    font-weight: 800;
    letter-spacing: -0.02em;
    line-height: 1.2;
    color: $text-primary;
  }

  &-sub {
    margin-top: 6px;
    font-size: 0.9375rem;
    color: $text-secondary;
    max-width: 608px;
    margin-bottom: 1.5rem;
  }

  /* ---------- fields ---------- */
  &__field {
    max-width: 480px;
    margin-top: 1.75rem;

    &:first-child {
      margin-top: 0;
    }
  }

  &__label {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 0.84375rem;
    font-weight: 600;
    color: $text-secondary;
    margin-bottom: 0.4375rem;

    .opt {
      color: $text-tertiary;
      font-weight: 400;
      font-size: 0.75rem;
    }
  }

  &__input {
    width: 100%;
    border: 1px solid $border-default;
    border-radius: 0.5rem;
    padding: 0.75rem 0.875rem;
    font-size: 1rem;
    font-family: $font-body;
    color: $text-primary;
    background: $bg-main;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &::placeholder {
      color: #a8b1bb;
    }

    &:focus {
      outline: none;
      border-color: $primary;
      box-shadow: 0 0 0 0.1875rem $primary-light;
    }
  }

  &__select {
    width: 100%;
    border: 1px solid $border-default;
    border-radius: 0.5rem;
    padding: 0.625rem 0.75rem;
    font-size: 0.9375rem;
    font-family: $font-body;
    color: $text-primary;
    background: $bg-main;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &:focus {
      outline: none;
      border-color: $primary;
      box-shadow: 0 0 0 0.1875rem $primary-light;
    }
  }

  /* ---------- type cards (Model / Agent / RAG) ---------- */
  &__type-grid {
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
    margin-top: 1.5rem;
    max-width: 560px;
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
    max-width: 560px;
  }

  &__framework-hint {
    font-size: 0.8125rem;
    color: $text-tertiary;
    margin-bottom: 0.875rem;
  }

  &__framework-grid {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 0.75rem;
  }

  &__type-card--framework {
    padding: 0.875rem 2.75rem 0.875rem 0.875rem;
    gap: 0.75rem;

    .wiz__type-icon {
      width: 32px;
      height: 32px;
    }

    .wiz__type-title {
      font-size: 0.9375rem;
    }

    .wiz__type-desc {
      font-size: 0.8125rem;
    }
  }

  /* ---------- generic selectable card grid (providers / models / metrics) ---------- */
  &__grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(230px, 1fr));
    gap: 0.75rem;
    margin-top: 1.5rem;
  }

  &__card {
    position: relative;
    text-align: left;
    display: flex;
    align-items: flex-start;
    gap: 0.75rem;
    padding: 0.875rem 2.25rem 0.875rem 0.875rem;
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

  &__card-icon {
    flex-shrink: 0;
    width: 30px;
    height: 30px;
    border-radius: 0.5625rem;
    display: grid;
    place-items: center;
    background: $bg-subtle;
    color: $text-tertiary;
    transition: background 0.16s ease, color 0.16s ease;
  }

  &__card--selected &__card-icon {
    background: $primary;
    color: #fff;
  }

  &__card-text {
    display: flex;
    flex-direction: column;
    gap: 2px;
    min-width: 0;
  }

  &__card-name {
    font-size: 0.875rem;
    font-weight: 600;
    color: $text-primary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__card-sub {
    font-size: 0.75rem;
    color: $text-tertiary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__card-check {
    position: absolute;
    top: 50%;
    right: 0.75rem;
    transform: translateY(-50%);
    width: 18px;
    height: 18px;
    border-radius: 50%;
    display: grid;
    place-items: center;
    background: $primary;
    color: #fff;
    flex-shrink: 0;
  }

  &__empty {
    grid-column: 1 / -1;
    padding: 2rem;
    text-align: center;
    color: $text-tertiary;
    font-size: 0.90625rem;
    background: $bg-subtle;
    border-radius: 0.875rem;
  }

  // scrollable wrapper used for the providers / models card grids
  &__grid-scroll {
    margin-top: 1.5rem;
    max-height: 26rem;
    overflow-y: auto;
    padding-right: 4px;
    margin-right: -4px;

    .wiz__grid {
      margin-top: 0;
    }
  }

  /* ---------- test suite: dataset grid + subgroup panel ---------- */
  &__dataset-layout {
    display: flex;
    align-items: stretch;
    gap: 1.25rem;
    margin-top: 1.5rem;
    height: 28rem;
    min-height: 0;
  }

  &__dataset-grid-scroll {
    flex: 1;
    min-width: 0;
    min-height: 0;
    overflow-y: auto;
    padding-right: 4px;
    margin-right: -4px;
  }

  &__dataset-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
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

  &__dataset-name {
    font-size: 0.9375rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__dataset-desc {
    font-size: 0.8125rem;
    color: $text-secondary;
    line-height: 1.5;
    display: -webkit-box;
    -webkit-line-clamp: 2;
    -webkit-box-orient: vertical;
    overflow: hidden;
  }

  &__dataset-meta {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 0.625rem;
    font-size: 0.78125rem;
    color: $text-tertiary;
  }

  // ---- subgroup panel (persistent column beside the dataset grid) ----
  &__subgroup-panel {
    flex-shrink: 0;
    width: 320px;
    min-height: 0;
    display: flex;
    flex-direction: column;
    border: 1px solid $border-subtle;
    border-radius: 0.875rem;
    background: $bg-subtle;
    overflow: hidden;
  }

  &__subgroup-panel-head {
    flex-shrink: 0;
    padding: 14px 16px 12px;
    border-bottom: 1px solid $border-subtle;
  }

  &__subgroup-panel-title {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 0.8125rem;
    font-weight: 800;
    color: $text-primary;
    letter-spacing: -0.01em;
  }

  &__subgroup-panel-sub {
    margin-top: 3px;
    font-size: 0.71875rem;
    color: $text-tertiary;
  }

  &__subgroup-panel-scroll {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding: 10px;
    display: flex;
    flex-direction: column;
    gap: 0.375rem;
  }

  &__subgroup-empty {
    padding: 1.25rem 0.75rem;
    text-align: center;
    border: 1px dashed $border-strong;
    border-radius: 0.625rem;
    font-size: 0.78125rem;
    color: $text-tertiary;
    margin: 10px;
  }

  &__subgroup-row {
    display: flex;
    align-items: center;
    gap: 0.625rem;
    width: 100%;
    text-align: left;
    padding: 0.5625rem 0.6875rem;
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

  &__subgroup-row-name {
    font-size: 0.8125rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__checkbox {
    flex-shrink: 0;
    width: 17px;
    height: 17px;
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

  /* ---------- static chips / tags ---------- */
  &__chip {
    font-size: 0.75rem;
    font-weight: 600;
    color: $primary;
    background: $primary-light;
    border-radius: 0.375rem;
    padding: 0.1875rem 0.5rem;
    display: inline-block;
  }

  /* ---------- metrics: cards + judge panel side by side ---------- */
  &__metrics-toolbar {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 1rem;
    margin-top: 1.5rem;
  }

  &__metrics-count {
    display: flex;
    align-items: center;
    gap: 0.625rem;
    font-size: 0.875rem;
    color: $text-secondary;

    strong {
      font-weight: 700;
      color: $primary;
    }
  }

  &__link-btn {
    font-family: $font-body;
    font-size: 0.8125rem;
    font-weight: 600;
    color: $primary;
    background: transparent;
    border: none;
    padding: 0;
    cursor: pointer;

    &:hover {
      text-decoration: underline;
    }
  }

  &__metrics-layout {
    display: grid;
    grid-template-columns: 1fr 300px;
    align-items: stretch;
    gap: 1.25rem;
    margin-top: 0.875rem;
    height: 26rem;
    min-height: 0;
  }

  &__metrics-main-scroll {
    min-width: 0;
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
    padding: 1rem 1.125rem;
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
    min-height: 0;
    overflow: hidden;
  }

  &__judge-title {
    display: flex;
    align-items: center;
    gap: 0.375rem;
    font-family: $font-mono;
    font-size: 0.71875rem;
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $text-tertiary;

    svg {
      color: $primary;
    }
  }

  &__judge-hint {
    font-size: 0.75rem;
    color: $text-tertiary;
    line-height: 1.5;
    margin: 0.5rem 0 0.75rem;
    flex-shrink: 0;
  }

  &__judge-panel-scroll {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
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

  &__judge-row {
    display: flex;
    align-items: center;
    gap: 0.625rem;
    width: 100%;
    text-align: left;
    padding: 0.5625rem 0.6875rem;
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
    font-size: 0.8125rem;
    font-weight: 600;
    color: $text-primary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__judge-row-meta {
    font-size: 0.6875rem;
    color: $text-tertiary;
  }

  &__radio {
    flex-shrink: 0;
    width: 15px;
    height: 15px;
    border-radius: 50%;
    border: 1.5px solid $border-strong;
    background: $bg-main;
    transition: border-color 0.14s ease;

    &--checked {
      border-color: $primary;
      border-width: 4.5px;
    }
  }

  /* ---------- review step ---------- */
  &__review-stats {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 0.75rem;
    margin-top: 1.5rem;
  }

  &__review-stat {
    padding: 0.875rem 1rem;
    border: 1px solid $border-subtle;
    border-radius: 0.75rem;
    background: $bg-subtle;
  }

  &__review-stat-label {
    display: block;
    font-size: 0.6875rem;
    font-weight: 700;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $text-tertiary;
    margin-bottom: 4px;
  }

  &__review-stat-value {
    display: block;
    font-family: $font-mono;
    font-size: 1.375rem;
    font-weight: 800;
    color: $text-primary;
    letter-spacing: -0.01em;
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
  }

  &__review-section {
    margin-top: 1.75rem;
  }

  &__review-section-title {
    display: flex;
    align-items: center;
    gap: 5px;
    font-family: $font-mono;
    font-size: 0.71875rem;
    font-weight: 700;
    letter-spacing: 0.08em;
    text-transform: uppercase;
    color: $text-tertiary;
  }

  &__error {
    margin-top: 1.25rem;
    font-size: 0.875rem;
    color: $danger;
    background: $danger-subtle;
    border-radius: 0.5rem;
    padding: 0.625rem 0.875rem;
  }

  /* ---------- nav footer ---------- */
  &__nav {
    flex-shrink: 0;
    margin-top: 1.5rem;
    padding-top: 1.25rem;
    border-top: 1px solid $border-subtle;
    display: flex;
    align-items: center;
    justify-content: space-between;
  }

  &__spin {
    animation: wiz-spin 0.8s linear infinite;
  }

  @keyframes wiz-spin {
    to {
      transform: rotate(360deg);
    }
  }
}

.toast__icon {
  width: 36px;
  height: 36px;
  border-radius: 10px;
  background: $success-subtle;
  display: flex;
  align-items: center;
  justify-content: center;
}

@media (max-width: 1100px) {
  .wiz__dataset-layout {
    flex-direction: column;
    height: auto;
  }

  .wiz__dataset-grid-scroll {
    max-height: 22rem;
  }

  .wiz__subgroup-panel {
    width: 100%;
    max-height: 18rem;
  }
}

@media (max-width: 900px) {
  .wiz__metrics-layout {
    grid-template-columns: 1fr;
    height: auto;
  }

  .wiz__metrics-main-scroll {
    max-height: 20rem;
  }

  .wiz__judge-panel-scroll {
    max-height: 16rem;
  }

  .wiz__framework-grid {
    grid-template-columns: 1fr;
  }
}

@media (max-width: 720px) {
  .wiz-shell {
    flex-direction: column;
  }

  .wiz__sidebar {
    width: 100%;
    flex-direction: row;
    overflow-x: auto;
    border-right: none;
    border-bottom: 1px solid $border-subtle;
    padding: 14px;
    gap: 6px;
  }

  .wiz__step {
    flex-direction: column;
    align-items: center;
    text-align: center;
    padding: 8px 10px;
    flex-shrink: 0;
    width: 92px;

    &::before {
      display: none;
    }
  }

  .wiz__step-text {
    align-items: center;
    padding-top: 2px;
  }

  .wiz__step-desc {
    display: none;
  }

  .wiz__content {
    padding: 24px 20px;
  }

  .page {
    padding: 20px 16px 32px;
  }
}
