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
  Gauge,
  ClipboardCheck,
  Gavel,
  Wallet,
  Layers,
  Loader2,
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
  { label: 'Name & Type', description: 'Name it and pick what you\u2019re testing' },
  { label: 'Providers', description: 'Choose connected providers' },
  { label: 'Models', description: 'Pick models to compare' },
  { label: 'Test Suite', description: 'Select a benchmark or dataset' },
  { label: 'Metrics', description: 'Choose what to measure' },
  { label: 'Review', description: 'Confirm and launch the run' },
];

const STEP_ICONS: ComponentType<{ size?: number }>[] = [Tag, Plug, Cpu, Database, Gauge, ClipboardCheck];

const TYPE_OPTIONS = [
  { v: 'Model', icon: Cpu, sub: 'General-purpose model' },
  { v: 'Agent', icon: Bot, sub: 'Autonomous agent' },
  { v: 'RAG', icon: Database, sub: 'Retrieval-augmented' },
];

export default function NewEvaluation() {
  const dispatch = useAppDispatch();
  const navigate = useNavigate();
  const [step, setStep] = useState(0);
  const [toast, setToast] = useState(false);
  const totalSteps = STEPS.length;

  const rawDraft = useAppSelector((s) => s.evaluations.draft);
  const launching = useAppSelector((s) => s.evaluations.launching);
  const launchError = useAppSelector((s) => s.evaluations.launchError);

  const providers = useAppSelector((s) => s.providers.items) ?? [];
  const models = useAppSelector((s) => s.models.items) ?? [];
  const benchmarks = useAppSelector((s) => s.benchmarks.items) ?? [];
  const metrics = useAppSelector((s) => s.metrics) ?? { allMetrics: [], customAgentMetrics: [] };

  // Defensive defaults: guards the review-step calculations below, which now
  // run on every render (not just when step 5 is active), against a draft
  // that hasn't been fully hydrated yet.
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
    if (step === 0) return Boolean(draft.name.trim() && draft.eval_type);
    if (step === 1) return draft.selProviders.length > 0;
    if (step === 2) return draft.selModels.length > 0;
    if (step === 3) return Boolean(draft.selBenchmark);
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
      selected_category: benchmark ? [benchmark.type] : undefined,
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
      <div className={styles.wiz}>
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
                  <p className={styles['wiz-sub']}>Give it a recognizable name and choose what you're evaluating.</p>

                  <div className={styles.wiz__field}>
                    <label className={styles.wiz__label}>Evaluation Name</label>
                    <input
                      className={styles.wiz__input}
                      placeholder="e.g. Q3 Model Selection"
                      value={draft.name}
                      onChange={(e) => dispatch(setDraft({ name: e.target.value }))}
                    />
                  </div>

                  <div className={styles.wiz__field} style={{ maxWidth: 'none' }}>
                    <label className={styles.wiz__label}>Evaluation Type</label>
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
                  </div>
                </>
              )}

              {step === 1 && (
                <>
                  <h2>Select providers</h2>
                  <p className={styles['wiz-sub']}>Choose which connected providers to draw models from.</p>
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
                              <Check size={12} strokeWidth={2.75} />
                            </span>
                          )}
                        </button>
                      );
                    })}
                    {connectedProviders.length === 0 && (
                      <p className={styles.wiz__empty}>No connected providers yet — connect one from the Providers page first.</p>
                    )}
                  </div>
                </>
              )}

              {step === 2 && (
                <>
                  <h2>Choose models</h2>
                  <p className={styles['wiz-sub']}>Pick which models to include in this evaluation.</p>
                  {availableModels.length > 0 ? (
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
                                <Check size={12} strokeWidth={2.75} />
                              </span>
                            )}
                          </button>
                        );
                      })}
                    </div>
                  ) : (
                    <p className={styles.wiz__empty}>Select providers first to see available models.</p>
                  )}
                </>
              )}

              {step === 3 && (
                <>
                  <h2>Pick a test suite</h2>
                  <p className={styles['wiz-sub']}>Select the benchmark to evaluate against.</p>
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
                              <span className={styles['wiz__type-check-inline']}>
                                <Check size={12} strokeWidth={2.75} />
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
                  </div>
                </>
              )}

              {step === 4 && (
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
                            <Gauge size={15} />
                          </span>
                          <span className={styles['wiz__card-text']}>
                            <span className={styles['wiz__card-name']}>{m}</span>
                          </span>
                          {selected && (
                            <span className={styles['wiz__card-check']}>
                              <Check size={12} strokeWidth={2.75} />
                            </span>
                          )}
                        </button>
                      );
                    })}
                    {activeMetricsList.length === 0 && <p className={styles.wiz__empty}>No metrics available.</p>}
                  </div>

                  <div className={styles['wiz__field--judge']}>
                    <label className={styles.wiz__label}>
                      <Gavel size={13} strokeWidth={2.25} />
                      Judge Model <span className="opt">(optional — grades other models)</span>
                    </label>
                    <select
                      className={styles.wiz__select}
                      value={draft.judgeModelId || ''}
                      onChange={(e) => dispatch(setDraft({ judgeModelId: e.target.value || undefined }))}
                    >
                      <option value="">None</option>
                      {models
                        .filter((m) => m.is_active)
                        .map((m) => (
                          <option key={m.id} value={m.id}>
                            {m.name}
                          </option>
                        ))}
                    </select>
                  </div>
                </>
              )}

              {step === 5 && (
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
                    <p className={styles['wiz__review-section-title']}>Overview</p>
                    <div className={styles.wiz__review}>
                      <div className={styles['wiz__review-row']}>
                        <span>Name</span>
                        <span>{draft.name || '—'}</span>
                      </div>
                      <div className={styles['wiz__review-row']}>
                        <span>Type</span>
                        <span>{draft.eval_type || '—'}</span>
                      </div>
                      <div className={styles['wiz__review-row']}>
                        <span>Providers</span>
                        <span>{draft.selProviders.map((id) => providers.find((p) => p.id === id)?.name || id).join(', ') || '—'}</span>
                      </div>
                    </div>
                  </div>

                  <div className={styles['wiz__review-section']}>
                    <p className={styles['wiz__review-section-title']}>Models ({selectedModels.length})</p>
                    {selectedModels.length > 0 ? (
                      <div className={styles.wiz__grid} style={{ marginTop: '0.75rem' }}>
                        {selectedModels.map((m) => (
                          <div key={m!.id} className={styles.wiz__card} style={{ cursor: 'default' }}>
                            <span className={styles['wiz__card-icon']}>
                              <Cpu size={15} />
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
                    <p className={styles['wiz__review-section-title']}>Test Suite</p>
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
                    </div>
                  </div>

                  <div className={styles['wiz__review-section']}>
                    <p className={styles['wiz__review-section-title']}>Metrics ({draft.selMetrics.length})</p>
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
                        <Gavel size={11} strokeWidth={2.25} style={{ marginRight: 4, verticalAlign: -2 }} />
                        Judge Model
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
