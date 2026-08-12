import { useEffect, useMemo, useState } from 'react';
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
  Layers,
  Loader2,
  Waypoints,
  Lightbulb,
  Plus,
  Upload,
  FileText,
  HeartPulse,
  ShieldCheck,
  ShieldAlert,
  type LucideIcon,
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchProviders } from '../../store/slices/providersSlice';
import { fetchModels, checkModelHealth } from '../../store/slices/modelsSlice';
import { fetchDatasets, uploadDataset, resetUploadStatus } from '../../store/slices/datasetsSlice';
import { SUPPORTED_UPLOAD_EXTENSIONS } from '../../api/endpoints/datasets';
import { fetchMetrics } from '../../store/slices/metricsSlice';
import { launchEvaluation, setDraft } from '../../store/slices/evaluationsSlice';
import type { CreateEvaluationRequest } from '../../types';
import styles from './NewEvaluation.module.scss';

// ─────────────────────────────────────────────────────────────────────────
// Assumed slice shapes this component depends on:
//
// metricsSlice:
//   - fetchMetrics(evalType: string) — GET /metrics?eval_type={type}
//     Response: { eval_type, metrics: string[], all_metrics: string[] }
//   - state.metrics: { metrics: string[], allMetrics: string[],
//                       status: 'idle'|'loading'|'succeeded'|'failed', error }
//   - dispatched only once the user picks a type in Step 2 (not on mount).
//
// modelsSlice:
//   - checkModelHealth(modelId: string) — GET /models/health/{model_id}
//     Response: { success, message, model_id, response }
//   - state.models.healthById: Record<string, 'idle'|'loading'|'success'|'failed'>
//   - NEVER dispatched automatically — only in response to the user
//     explicitly clicking "Check health" on a model card.
// ─────────────────────────────────────────────────────────────────────────

const STEPS = [
  { label: 'Name' },
  { label: 'Type' },
  { label: 'Providers' },
  { label: 'Models' },
  { label: 'Test Suite' },
  { label: 'Metrics' },
  { label: 'Review' },
];

const STAGE = [
  { title: 'Name your run', sub: 'A recognizable name makes this run easy to find later in your history.' },
  { title: 'What are you evaluating?', sub: 'The system under test shapes which datasets and metrics you can pick.' },
  { title: 'Select providers', sub: 'Choose which connected providers to draw candidate models from.' },
  { title: 'Choose models', sub: 'Check a model\u2019s health before selecting it — only models that pass the check can be added to the run.' },
  { title: 'Pick a test suite', sub: 'Select a dataset to evaluate against, or upload your own.' },
  { title: 'Configure metrics', sub: 'Choose what to measure, and optionally a model to judge open-ended answers.' },
  { title: 'Review & launch', sub: 'Confirm the run manifest, then launch.' },
];

const STEP_ICONS: LucideIcon[] = [Tag, LayoutGrid, Plug, Cpu, Database, Target, ClipboardCheck];

const TYPE_OPTIONS = [
  {
    v: 'Model',
    icon: Cpu,
    sub: 'Benchmark a general-purpose LLM on standard tasks like reasoning, coding, and knowledge — ideal for comparing raw model quality across providers.',
    variant: '',
    disabled: false,
  },
  {
    v: 'Agent',
    icon: Bot,
    sub: 'Test an autonomous agent that plans, calls tools, and completes multi-step tasks — measures task completion, not just single-turn output.',
    variant: 'agent',
    disabled: false,
  },
  {
    v: 'RAG',
    icon: Database,
    sub: 'Evaluate a retrieval-augmented pipeline for grounding accuracy — checks how well answers stay faithful to your retrieved context.',
    variant: 'rag',
    disabled: true,
  },
];

// Optional agent frameworks, only shown once "Agent" is selected as the type.
const AGENT_FRAMEWORKS = [
  { id: 'hermes', title: 'Hermes', desc: 'Lightweight tool-calling agent runtime' },
  { id: 'langgraph', title: 'LangGraph', desc: 'Graph-based multi-step agent orchestration' },
];

const SUGGESTED_NAMES = [
  'Q3 Model Selection',
  'Support Bot Regression',
  'RAG Accuracy v2',
  'GPT-4o vs Claude',
];

const NAMING_TIPS = [
  "Include what you're testing — a model, a product feature, or a use case.",
  'Add a date or version so you can track changes over time (e.g. "Q3", "v2").',
  'Keep it specific enough to tell apart from similar past runs later.',
];

function formatContextWindow(tokens: number): string {
  if (tokens >= 1_000_000) return `${(tokens / 1_000_000).toLocaleString()}M`;
  if (tokens >= 1_000) return `${Math.round(tokens / 1000)}k`;
  return `${tokens}`;
}

function formatPrice(price: number | null | undefined): string {
  return price === null || price === undefined ? '—' : `$${price.toFixed(2)}`;
}

function providerInitials(name: string): string {
  const parts = name.replace(/[^a-zA-Z0-9 ]/g, '').split(' ').filter(Boolean);
  const letters = parts.slice(0, 2).map((w) => w[0]).join('');
  return (letters || name.slice(0, 2)).toUpperCase();
}

type HealthStatus = 'idle' | 'loading' | 'success' | 'failed';

export default function NewEvaluation() {
  const dispatch = useAppDispatch();
  const navigate = useNavigate();
  const [step, setStep] = useState(0);
  const [toast, setToast] = useState(false);
  const [agentFramework, setAgentFramework] = useState<string | null>(null);
  const [selSubgroup, setSelSubgroup] = useState<string[]>([]);
  const [runSamples, setRunSamples] = useState<number>(10);
  const [datasetTab, setDatasetTab] = useState<'browse' | 'upload'>('browse');
  const [uploadName, setUploadName] = useState('');
  const [uploadDescription, setUploadDescription] = useState('');
  const [uploadFile, setUploadFile] = useState<File | null>(null);
  const [uploadFileError, setUploadFileError] = useState<string | null>(null);
  const totalSteps = STEPS.length;

  const rawDraft = useAppSelector((s) => s.evaluations.draft);
  const launching = useAppSelector((s) => s.evaluations.launching);
  const launchError = useAppSelector((s) => s.evaluations.launchError);

  const providers = useAppSelector((s) => s.providers.items) ?? [];
  const models = useAppSelector((s) => s.models.items) ?? [];
  const healthById = useAppSelector((s) => (s.models as any).healthById) as Record<string, HealthStatus> | undefined;

  const metricsState = useAppSelector((s) => s.metrics) ?? { allMetrics: [], status: 'idle' as const, error: null };
  // Only `all_metrics` from the API response is used — it's the full
  // catalog rendered as selectable chips. The `metrics` field is ignored.
  const metricsCatalog: string[] = (metricsState as any).allMetrics ?? [];
  const metricsLoading = (metricsState as any).status === 'loading';

  const datasets = useAppSelector((s) => s.datasets.items) ?? [];
  const datasetsLoading = useAppSelector((s) => s.datasets.status === 'loading' || s.datasets.status === 'idle');
  const datasetsError = useAppSelector((s) => s.datasets.error);
  const datasetUploading = useAppSelector((s) => s.datasets.uploadStatus === 'loading');
  const datasetUploadError = useAppSelector((s) => s.datasets.uploadError);

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
  }, [dispatch]);

  // GET /datasets?eval_type={type} — refetched whenever the chosen type changes.
  useEffect(() => {
    if (!draft.eval_type) return;
    dispatch(fetchDatasets(draft.eval_type.toLowerCase()));
  }, [dispatch, draft.eval_type]);

  // GET /metrics?eval_type={type} — fetched only once a type is chosen in
  // Step 2, and re-fetched whenever the user changes type. Any metrics
  // selected under the previous type are cleared, since they may not be
  // valid for the new type. Only the response's `all_metrics` field is
  // consumed (see metricsCatalog above) — `metrics` is ignored entirely.
  useEffect(() => {
    if (!draft.eval_type) return;
    dispatch(fetchMetrics(draft.eval_type.toLowerCase()));
    dispatch(setDraft({ selMetrics: [] }));
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [dispatch, draft.eval_type]);

  // Reset the subgroup/task selection whenever the chosen dataset changes.
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

  // Health checks are NEVER fired automatically — the user must explicitly
  // click "Check health" on a model card. This runs that single check.
  const runHealthCheck = (modelId: string) => {
    dispatch(checkModelHealth(modelId));
  };

  const toggle = (list: string[], value: string) =>
    list.includes(value) ? list.filter((v) => v !== value) : [...list, value];

  const getFileExtension = (filename: string) => {
    const idx = filename.lastIndexOf('.');
    return idx >= 0 ? filename.slice(idx + 1).toLowerCase() : '';
  };

  const openUploadPanel = () => {
    dispatch(resetUploadStatus());
    setUploadName('');
    setUploadDescription('');
    setUploadFile(null);
    setUploadFileError(null);
    setDatasetTab('upload');
  };

  const handleUploadFileChange = (file: File | null) => {
    setUploadFile(file);
    if (!file) {
      setUploadFileError(null);
      return;
    }
    const ext = getFileExtension(file.name);
    if (!SUPPORTED_UPLOAD_EXTENSIONS.includes(ext)) {
      setUploadFileError('Unsupported file type. Please choose a .json, .jsonl, .arrow, or .parquet file.');
    } else {
      setUploadFileError(null);
    }
  };

  const canUpload =
    Boolean(uploadName.trim()) && Boolean(uploadFile) && !uploadFileError && Boolean(draft.eval_type) && !datasetUploading;

  const submitUpload = async () => {
    if (!uploadFile || !canUpload) return;
    const result = await dispatch(
      uploadDataset({
        file: uploadFile,
        name: uploadName.trim(),
        description: uploadDescription.trim(),
        evalType: draft.eval_type.toLowerCase(),
      })
    );
    if (uploadDataset.fulfilled.match(result)) {
      dispatch(setDraft({ selBenchmark: result.payload.id }));
      setDatasetTab('browse');
    }
  };

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

  const suite = datasets.find((d) => d.id === draft.selBenchmark);
  const selectedModels = draft.selModels.map((id) => models.find((m) => m.id === id)).filter(Boolean) as typeof models;
  const judgeModel = draft.judgeModelId ? models.find((m) => m.id === draft.judgeModelId) : null;

  // A model can only be added to the run once the user has manually run a
  // health check on it AND it came back successful.
  const isModelSelectable = (modelId: string) => healthById?.[modelId] === 'success';

  const toggleModel = (modelId: string) => {
    const alreadySelected = draft.selModels.includes(modelId);
    if (!alreadySelected && !isModelSelectable(modelId)) return;
    dispatch(setDraft({ selModels: toggle(draft.selModels, modelId) }));
  };

  const launch = async () => {
    const dataset = datasets.find((d) => d.id === draft.selBenchmark);
    const judgeModelObj = draft.judgeModelId ? models.find((m) => m.id === draft.judgeModelId) : undefined;

    const payload: CreateEvaluationRequest = {
      name: draft.name,
      eval_type: draft.eval_type.toLowerCase(),
      dataset_id: dataset?.id || '',
      benchmark: dataset?.name || undefined,
      model_ids: draft.selModels,
      selected_metrics: draft.selMetrics,
      run_samples: runSamples,
      selected_category: selSubgroup.length > 0 ? selSubgroup : dataset ? [dataset.category] : undefined,
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

  // ---- live Run Manifest values (one per step) ----------------------------
  const providerNames = draft.selProviders.map((id) => providers.find((p) => p.id === id)?.name || id);
  const mf = (value: string, filled: boolean) => ({ value: filled ? value : '—', empty: !filled });
  const frameworkTitle = agentFramework ? AGENT_FRAMEWORKS.find((f) => f.id === agentFramework)?.title : null;
  const manifest = [
    mf(draft.name, Boolean(draft.name)),
    mf(frameworkTitle ? `${draft.eval_type} · ${frameworkTitle}` : draft.eval_type, Boolean(draft.eval_type)),
    mf(draft.selProviders.length === 1 ? providerNames[0] : `${draft.selProviders.length} providers`, draft.selProviders.length > 0),
    mf(`${draft.selModels.length} models`, draft.selModels.length > 0),
    mf(suite?.name || '', Boolean(suite)),
    mf(`${draft.selMetrics.length} metrics`, draft.selMetrics.length > 0),
    mf(judgeModel ? `Judge · ${judgeModel.name}` : 'Ready to launch', true),
  ];

  const CrumbIcon = STEP_ICONS[step];

  return (
    <div className="page-enter" style={{ height: '100%', display: 'flex', flexDirection: 'column' }}>
      {/* ---- header (matches History/Reports/Comparison/Sidebar pattern) ---- */}
      <div className={styles['ev__header']}>
        <div>
          <p className={styles['ev__header-eyebrow']}>Evaluation console</p>
          <h1>New run</h1>
          <p className={styles['ev__header-sub']}>Assemble and launch a new evaluation run</p>
        </div>
        <div className={styles['ev__header-meta']}>
          <span className={styles['ev__header-status']} data-state={launching ? 'live' : 'draft'}>
            {launching ? 'Launching' : 'Draft'}
          </span>
          <span className={styles['ev__header-eta']}>
            <Clock3 size={13} /> ~5 min
          </span>
        </div>
      </div>

      <div className={styles.page}>
        <div className={styles.ev}>
          {/* ---- shell ---- */}
          <div className={styles.ev__shell}>
            {/* SIGNATURE: Run Manifest */}
            <aside className={styles.ev__manifest}>
              <div className={styles['ev__manifest-head']}>
                <div className={styles['ev__manifest-eyebrow']}>
                  <span>Run manifest</span>
                  <span className={styles['ev__manifest-pct']}>{progressPct}%</span>
                </div>
                <div className={styles['ev__manifest-title']} data-empty={!draft.name}>
                  {draft.name || 'Untitled run'}
                </div>
                <div className={styles.ev__meter}>
                  <div className={styles['ev__meter-fill']} style={{ width: `${progressPct}%` }} />
                </div>
              </div>
              <div className={styles.ev__spec}>
                {STEPS.map((s, i) => {
                  const state = i === step ? 'active' : i < step ? 'done' : 'todo';
                  const Icon = STEP_ICONS[i];
                  const row = manifest[i];
                  return (
                    <button
                      key={s.label}
                      type="button"
                      className={`${styles['ev__spec-row']} ${styles[`ev__spec-row--${state}`]}`}
                      onClick={() => goToStep(i)}
                      disabled={i > step}
                    >
                      <span className={styles['ev__spec-tick']}>
                        {state === 'done' ? <Check size={13} strokeWidth={3} /> : <Icon size={14} />}
                      </span>
                      <span className={styles['ev__spec-body']}>
                        <span className={styles['ev__spec-label']}>{s.label}</span>
                        <span className={styles['ev__spec-value']} data-empty={row.empty}>
                          {row.value}
                        </span>
                      </span>
                    </button>
                  );
                })}
              </div>
            </aside>

            {/* STAGE */}
            <section className={styles.ev__stage}>
              <div className={styles['ev__stage-head']}>
                <div className={styles.ev__crumb}>
                  <span>
                    <CrumbIcon size={13} /> Step
                  </span>
                  <span className={styles['ev__crumb-sep']} />
                  <span>
                    <b>{String(step + 1).padStart(2, '0')}</b> / {String(totalSteps).padStart(2, '0')}
                  </span>
                  <span className={styles['ev__crumb-sep']} />
                  <span>{STEPS[step].label}</span>
                </div>
                <h2 className={styles['ev__stage-title']}>{STAGE[step].title}</h2>
                <p className={styles['ev__stage-sub']}>{STAGE[step].sub}</p>
              </div>

              <div className={styles['ev__stage-body']}>
                <div key={step} className={styles.ev__anim} style={{ flex: 1, minHeight: 0, display: 'flex', flexDirection: 'column' }}>
                  {/* STEP 0 — NAME */}
                  {step === 0 && (
                    <>
                      <div className={styles.ev__field} style={{ maxWidth: 620 }}>
                        <label className={styles.ev__label}>Run name</label>
                        <input
                          className={styles['ev__name-input']}
                          placeholder="Untitled run"
                          value={draft.name}
                          onChange={(e) => dispatch(setDraft({ name: e.target.value }))}
                          onKeyDown={(e) => {
                            if (e.key === 'Enter' && canGo()) goNext();
                          }}
                          autoFocus
                        />
                        <p className={styles['ev__name-caption']}>
                          <Tag size={13} /> This is how the run appears in your history.
                        </p>
                      </div>

                      <div className={styles.ev__quick}>
                        <p className={styles['ev__quick-head']}>Presets</p>
                        <div className={styles['ev__quick-row']}>
                          {SUGGESTED_NAMES.map((s) => {
                            const on = draft.name === s;
                            return (
                              <button
                                key={s}
                                type="button"
                                className={`${styles.ev__preset} ${on ? styles['ev__preset--on'] : ''}`}
                                onClick={() => dispatch(setDraft({ name: s }))}
                              >
                                {on ? <Check size={13} strokeWidth={3} /> : <Plus size={13} strokeWidth={2.5} />} {s}
                              </button>
                            );
                          })}
                        </div>
                      </div>

                      <div className={styles.ev__note}>
                        <span className={styles['ev__note-icon']}>
                          <Lightbulb size={16} />
                        </span>
                        <div>
                          <p className={styles['ev__note-title']}>What makes a good name</p>
                          <ul className={styles['ev__note-list']}>
                            {NAMING_TIPS.map((tip) => (
                              <li key={tip}>{tip}</li>
                            ))}
                          </ul>
                        </div>
                      </div>
                    </>
                  )}

                  {/* STEP 1 — TYPE */}
                  {step === 1 && (
                    <>
                      <div className={styles.ev__options}>
                        {TYPE_OPTIONS.map((o) => {
                          const Icon = o.icon;
                          const on = draft.eval_type === o.v;
                          return (
                            <button
                              key={o.v}
                              type="button"
                              className={`${styles.ev__option} ${on ? styles['ev__option--on'] : ''} ${
                                o.disabled ? styles['ev__option--off'] : ''
                              }`}
                              onClick={() => !o.disabled && dispatch(setDraft({ eval_type: o.v }))}
                              disabled={o.disabled}
                            >
                              <span
                                className={`${styles['ev__option-icon']} ${
                                  o.variant ? styles[`ev__option-icon--${o.variant}`] : ''
                                }`}
                              >
                                <Icon size={20} />
                              </span>
                              <span className={styles['ev__option-main']}>
                                <span className={styles['ev__option-name']}>
                                  {o.v}
                                  {o.disabled && <span className={styles.ev__badge}>Soon</span>}
                                </span>
                                <span className={styles['ev__option-desc']}>{o.sub}</span>
                              </span>
                              {on && (
                                <span className={styles.ev__mark}>
                                  <Check size={13} strokeWidth={3} />
                                </span>
                              )}
                            </button>
                          );
                        })}
                      </div>

                      {draft.eval_type === 'Agent' && (
                        <div className={styles.ev__section}>
                          <label className={styles.ev__label}>
                            <Waypoints size={13} /> Agent framework <span className="opt">optional</span>
                          </label>
                          <p className={styles['ev__section-hint']}>Tell us which framework the agent runs on, if applicable.</p>
                          <div className={styles['ev__fw-grid']}>
                            {AGENT_FRAMEWORKS.map((f) => {
                              const on = agentFramework === f.id;
                              return (
                                <button
                                  key={f.id}
                                  type="button"
                                  className={`${styles.ev__fw} ${on ? styles['ev__fw--on'] : ''}`}
                                  onClick={() => setAgentFramework(on ? null : f.id)}
                                >
                                  <span className={styles['ev__fw-icon']}>
                                    <Waypoints size={16} />
                                  </span>
                                  <span style={{ display: 'flex', flexDirection: 'column', gap: 2, minWidth: 0 }}>
                                    <span className={styles['ev__fw-name']}>{f.title}</span>
                                    <span className={styles['ev__fw-desc']}>{f.desc}</span>
                                  </span>
                                  {on && (
                                    <span className={styles.ev__mark}>
                                      <Check size={12} strokeWidth={3} />
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

                  {/* STEP 2 — PROVIDERS */}
                  {step === 2 && (
                    <div className={styles.ev__scroll}>
                      <div className={styles.ev__grid}>
                        {connectedProviders.map((p) => {
                          const on = draft.selProviders.includes(p.id);
                          return (
                            <button
                              key={p.id}
                              type="button"
                              className={`${styles.ev__pcard} ${on ? styles['ev__pcard--on'] : ''}`}
                              onClick={() => dispatch(setDraft({ selProviders: toggle(draft.selProviders, p.id) }))}
                            >
                              <span className={styles['ev__pcard-icon']}>{providerInitials(p.name)}</span>
                              <span className={styles['ev__pcard-body']}>
                                <span className={styles['ev__pcard-name']}>{p.name}</span>
                                <span className={styles['ev__pcard-meta']}>{p.model_count} models available</span>
                                <span className={styles.ev__pill}>Connected</span>
                              </span>
                              {on && (
                                <span className={styles.ev__mark}>
                                  <Check size={12} strokeWidth={3} />
                                </span>
                              )}
                            </button>
                          );
                        })}
                        {connectedProviders.length === 0 && (
                          <p className={styles.ev__empty}>No connected providers yet. Connect one from the Providers page to continue.</p>
                        )}
                      </div>
                    </div>
                  )}

                  {/* STEP 3 — MODELS */}
                  {step === 3 &&
                    (availableModels.length > 0 ? (
                      <div className={styles.ev__scroll}>
                        <div className={`${styles.ev__grid} ${styles['ev__grid--wide']}`}>
                          {availableModels.map((m) => {
                            const on = draft.selModels.includes(m.id);
                            const health: HealthStatus = healthById?.[m.id] ?? 'idle';
                            const selectable = health === 'success';
                            const caps = (m as any).capabilities as string[] | undefined;
                            const inputPrice = (m as any).input_price as number | null | undefined;
                            const outputPrice = (m as any).output_price as number | null | undefined;
                            const accuracy = (m as any).accuracy_score as number | null | undefined;
                            const providerName = providers.find((p) => p.id === m.provider_id)?.name ?? m.provider_id;

                            return (
                              // Not a <button> — it now contains a nested "Check health"
                              // control, so it's a clickable div with keyboard support
                              // instead (nested interactive elements aren't valid HTML).
                              <div
                                key={m.id}
                                role="button"
                                tabIndex={0}
                                className={`${styles.ev__mcard} ${on ? styles['ev__mcard--on'] : ''} ${
                                  !selectable && !on ? styles['ev__mcard--locked'] : ''
                                }`}
                                onClick={() => toggleModel(m.id)}
                                onKeyDown={(e) => {
                                  if (e.key === 'Enter' || e.key === ' ') {
                                    e.preventDefault();
                                    toggleModel(m.id);
                                  }
                                }}
                                aria-pressed={on}
                                aria-disabled={!selectable && !on}
                              >
                                <div className={styles['ev__mcard-top']}>
                                  <div className={styles['ev__mcard-name']}>{m.name}</div>
                                  {on && (
                                    <span className={styles['ev__mcard-mark']}>
                                      <Check size={12} strokeWidth={3} />
                                    </span>
                                  )}
                                </div>

                                {/* Provider name + health badge, same line, badge on the right */}
                                <div className={styles['ev__mcard-provider-row']}>
                                  <span className={styles['ev__mcard-provider']}>{providerName}</span>

                                  {health === 'success' && (
                                    <span className={`${styles['ev__health-badge']} ${styles['ev__health-badge--success']}`}>
                                      <ShieldCheck size={12} /> Available
                                    </span>
                                  )}

                                  {health === 'failed' && (
                                    <button
                                      type="button"
                                      className={`${styles['ev__health-badge']} ${styles['ev__health-badge--failed']}`}
                                      onClick={(e) => {
                                        e.stopPropagation();
                                        runHealthCheck(m.id);
                                      }}
                                      title="Retry health check"
                                    >
                                      <ShieldAlert size={12} /> Unavailable
                                    </button>
                                  )}

                                  {health === 'loading' && (
                                    <span className={`${styles['ev__health-badge']} ${styles['ev__health-badge--loading']}`}>
                                      <Loader2 size={12} className={styles.ev__spin} /> Checking…
                                    </span>
                                  )}

                                  {health === 'idle' && (
                                    <button
                                      type="button"
                                      className={styles['ev__health-check-btn']}
                                      onClick={(e) => {
                                        e.stopPropagation();
                                        runHealthCheck(m.id);
                                      }}
                                    >
                                      <HeartPulse size={12} /> Check health
                                    </button>
                                  )}
                                </div>

                                {caps && caps.length > 0 && (
                                  <div className={styles.ev__caps}>
                                    {caps.slice(0, 3).map((c) => (
                                      <span key={c} className={styles.ev__cap}>
                                        {c}
                                      </span>
                                    ))}
                                  </div>
                                )}
                                <div className={styles['ev__mcard-stats']}>
                                  <span className={styles.ev__stat}>
                                    <span className={styles['ev__stat-k']}>Context</span>
                                    <span className={styles['ev__stat-v']}>{formatContextWindow(m.context_window)}</span>
                                  </span>
                                  {(inputPrice !== undefined || outputPrice !== undefined) && (
                                    <span className={styles.ev__stat}>
                                      <span className={styles['ev__stat-k']}>Price /1M</span>
                                      <span className={styles['ev__stat-v']}>
                                        {formatPrice(inputPrice)}/{formatPrice(outputPrice)}
                                      </span>
                                    </span>
                                  )}
                                  {accuracy !== undefined && accuracy !== null && (
                                    <span className={styles.ev__stat}>
                                      <span className={styles['ev__stat-k']}>Accuracy</span>
                                      <span className={styles['ev__stat-v']}>{accuracy.toFixed(1)}%</span>
                                    </span>
                                  )}
                                </div>

                                {!selectable && !on && (
                                  <p className={styles['ev__mcard-hint']}>
                                    {health === 'idle' && 'Run a health check to enable selection.'}
                                    {health === 'loading' && 'Waiting for health check to complete…'}
                                    {health === 'failed' && 'This model failed its health check and can\u2019t be selected.'}
                                  </p>
                                )}
                              </div>
                            );
                          })}
                        </div>
                      </div>
                    ) : (
                      <p className={styles.ev__empty}>Select providers first to see their available models.</p>
                    ))}

                  {/* STEP 4 — TEST SUITE */}
                  {step === 4 && (
                    <>
                      <div className={styles.ev__tabs}>
                        <button
                          type="button"
                          className={`${styles.ev__tab} ${datasetTab === 'browse' ? styles['ev__tab--on'] : ''}`}
                          onClick={() => setDatasetTab('browse')}
                        >
                          <LayoutGrid size={14} /> Browse
                        </button>
                        <button
                          type="button"
                          className={`${styles.ev__tab} ${datasetTab === 'upload' ? styles['ev__tab--on'] : ''}`}
                          onClick={openUploadPanel}
                        >
                          <Upload size={14} /> Upload
                        </button>
                      </div>

                      {datasetTab === 'browse' && (
                        <div className={styles.ev__suite}>
                          <div className={styles['ev__suite-scroll']}>
                            {datasetsLoading && <p className={styles.ev__empty}>Loading test suites…</p>}
                            {!datasetsLoading && datasetsError && <p className={styles.ev__error}>{datasetsError}</p>}
                            {!datasetsLoading && !datasetsError && (
                              <div className={styles.ev__dgrid}>
                                {datasets.map((d) => {
                                  const on = draft.selBenchmark === d.id;
                                  return (
                                    <button
                                      key={d.id}
                                      type="button"
                                      className={`${styles.ev__dcard} ${on ? styles['ev__dcard--on'] : ''}`}
                                      onClick={() => dispatch(setDraft({ selBenchmark: d.id }))}
                                    >
                                      <div className={styles['ev__dcard-top']}>
                                        <div className={styles['ev__dcard-id']}>
                                          <span className={styles['ev__dcard-icon']}>
                                            <Database size={15} />
                                          </span>
                                          <span className={styles['ev__dcard-name']}>{d.name}</span>
                                        </div>
                                        {on && (
                                          <span className={styles['ev__mcard-mark']}>
                                            <Check size={12} strokeWidth={3} />
                                          </span>
                                        )}
                                      </div>
                                      <div className={styles['ev__dcard-tags']}>
                                        <span className={styles.ev__tag}>{d.category}</span>
                                        <span className={styles.ev__tag}>{d.eval_type}</span>
                                        {d.dataset_type === 'custom' && (
                                          <span className={`${styles.ev__tag} ${styles['ev__tag--custom']}`}>Custom</span>
                                        )}
                                        <span className={`${styles.ev__tag} ${styles['ev__tag--count']}`}>
                                          {d.question_count.toLocaleString()} questions
                                        </span>
                                      </div>
                                    </button>
                                  );
                                })}
                                {datasets.length === 0 && (
                                  <p className={styles.ev__empty}>No test suites available for this type yet.</p>
                                )}
                              </div>
                            )}
                          </div>

                          <aside className={styles.ev__rail}>
                            <div className={styles['ev__rail-head']}>
                              <p className={styles['ev__rail-title']}>
                                <Layers size={13} /> Subgroups
                              </p>
                              <p className={styles['ev__rail-sub']}>
                                {suite ? `Narrow "${suite.name}" to specific categories.` : 'Select a suite to see its subgroups.'}
                              </p>
                            </div>
                            <div className={styles['ev__rail-scroll']}>
                              {!suite && <p className={styles['ev__rail-empty']}>No suite selected yet.</p>}
                              {suite && suite.dataset_categories.length === 0 && (
                                <p className={styles['ev__rail-empty']}>This suite has no subgroups.</p>
                              )}
                              {suite &&
                                suite.dataset_categories.map((cat) => {
                                  const on = selSubgroup.includes(cat);
                                  return (
                                    <button
                                      key={cat}
                                      type="button"
                                      className={`${styles['ev__check-row']} ${on ? styles['ev__check-row--on'] : ''}`}
                                      onClick={() => setSelSubgroup((prev) => toggle(prev, cat))}
                                    >
                                      <span className={`${styles.ev__check} ${on ? styles['ev__check--on'] : ''}`}>
                                        {on && <Check size={11} strokeWidth={3} />}
                                      </span>
                                      <span className={styles['ev__check-label']}>{cat}</span>
                                    </button>
                                  );
                                })}
                            </div>
                          </aside>
                        </div>
                      )}

                      {datasetTab === 'upload' && (
                        <div className={styles.ev__upload}>
                          <div className={styles.ev__field}>
                            <label className={styles.ev__label}>Name</label>
                            <input
                              className={styles.ev__input}
                              placeholder="e.g. Internal QA set v1"
                              value={uploadName}
                              onChange={(e) => setUploadName(e.target.value)}
                              disabled={datasetUploading}
                            />
                          </div>
                          <div className={styles.ev__field}>
                            <label className={styles.ev__label}>
                              Description <span className="opt">optional</span>
                            </label>
                            <input
                              className={styles.ev__input}
                              placeholder="What does this dataset cover?"
                              value={uploadDescription}
                              onChange={(e) => setUploadDescription(e.target.value)}
                              disabled={datasetUploading}
                            />
                          </div>
                          <div className={styles.ev__field}>
                            <label className={styles.ev__label}>Evaluation type</label>
                            <input className={styles.ev__input} value={draft.eval_type || '—'} disabled readOnly />
                          </div>
                          <div className={styles.ev__field}>
                            <label className={styles.ev__label}>File</label>
                            <label className={`${styles.ev__drop} ${uploadFile ? styles['ev__drop--has'] : ''}`}>
                              <input
                                type="file"
                                accept={SUPPORTED_UPLOAD_EXTENSIONS.map((e) => `.${e}`).join(',')}
                                onChange={(e) => handleUploadFileChange(e.target.files?.[0] ?? null)}
                                disabled={datasetUploading}
                                hidden
                              />
                              {uploadFile ? (
                                <span className={styles['ev__drop-file']}>
                                  <FileText size={15} /> {uploadFile.name}
                                </span>
                              ) : (
                                <>
                                  <FileText size={15} /> Choose a .json, .jsonl, .arrow or .parquet file
                                </>
                              )}
                            </label>
                            {uploadFileError && <p className={styles.ev__error}>{uploadFileError}</p>}
                          </div>
                          {datasetUploadError && <p className={styles.ev__error}>{datasetUploadError}</p>}
                          <div className={styles['ev__upload-actions']}>
                            <button
                              type="button"
                              className={`${styles.ev__btn} ${styles['ev__btn--ghost']}`}
                              onClick={() => setDatasetTab('browse')}
                              disabled={datasetUploading}
                            >
                              Cancel
                            </button>
                            <button
                              type="button"
                              className={`${styles.ev__btn} ${styles['ev__btn--primary']}`}
                              onClick={submitUpload}
                              disabled={!canUpload}
                            >
                              {datasetUploading ? (
                                <>
                                  <Loader2 size={15} className={styles.ev__spin} /> Uploading…
                                </>
                              ) : (
                                <>
                                  <Upload size={15} /> Upload &amp; use
                                </>
                              )}
                            </button>
                          </div>
                        </div>
                      )}
                    </>
                  )}

                  {/* STEP 5 — METRICS */}
                  {step === 5 && (
                    <div className={styles.ev__metrics}>
                      <div className={styles['ev__metrics-main']}>
                        <div className={styles.ev__samples}>
                          <div className={styles.ev__field}>
                            <label className={styles.ev__label}>Run samples</label>
                            <input
                              type="number"
                              min={0}
                              className={styles.ev__input}
                              value={runSamples}
                              onChange={(e) => {
                                const val = e.target.value === '' ? 0 : Math.max(0, Number(e.target.value));
                                setRunSamples(Number.isNaN(val) ? 0 : val);
                              }}
                            />
                          </div>
                          <p className={styles['ev__samples-note']}>Questions sampled from the suite for each model.</p>
                        </div>

                        <div className={styles['ev__metrics-bar']}>
                          <span className={styles['ev__metrics-count']}>
                            <b>{draft.selMetrics.length}</b> of {metricsCatalog.length} selected
                          </span>
                          <div className={styles['ev__metrics-actions']}>
                            <button
                              type="button"
                              className={styles.ev__link}
                              onClick={() => dispatch(setDraft({ selMetrics: [...metricsCatalog] }))}
                            >
                              Select all
                            </button>
                            <button type="button" className={styles.ev__link} onClick={() => dispatch(setDraft({ selMetrics: [] }))}>
                              Clear
                            </button>
                          </div>
                        </div>

                        <div className={styles.ev__chips}>
                          {metricsLoading && <p className={styles.ev__empty}>Loading metrics for {draft.eval_type || 'this type'}…</p>}
                          {!metricsLoading &&
                            metricsCatalog.map((name: string) => {
                              const on = draft.selMetrics.includes(name);
                              return (
                                <button
                                  key={name}
                                  type="button"
                                  className={`${styles.ev__chip} ${on ? styles['ev__chip--on'] : ''}`}
                                  onClick={() => dispatch(setDraft({ selMetrics: toggle(draft.selMetrics, name) }))}
                                >
                                  {on && (
                                    <span className={styles['ev__chip-tick']}>
                                      <Check size={12} strokeWidth={3} />
                                    </span>
                                  )}
                                  {name}
                                </button>
                              );
                            })}
                          {!metricsLoading && metricsCatalog.length === 0 && (
                            <p className={styles.ev__empty}>No metrics available for this type.</p>
                          )}
                        </div>
                      </div>

                      <aside className={styles.ev__judge}>
                        <div className={styles['ev__judge-head']}>
                          <p className={styles['ev__judge-title']}>
                            <Gavel size={13} /> Judge model
                          </p>
                          <p className={styles['ev__judge-sub']}>Optionally grade open-ended answers with a model.</p>
                        </div>
                        <div className={styles['ev__judge-scroll']}>
                          {models.filter((m) => m.is_active).length === 0 ? (
                            <div className={styles['ev__judge-empty']}>No models available yet.</div>
                          ) : (
                            models
                              .filter((m) => m.is_active)
                              .map((m) => {
                                const on = draft.judgeModelId === m.id;
                                return (
                                  <button
                                    key={m.id}
                                    type="button"
                                    className={`${styles['ev__judge-row']} ${on ? styles['ev__judge-row--on'] : ''}`}
                                    onClick={() => dispatch(setDraft({ judgeModelId: on ? undefined : m.id }))}
                                  >
                                    <span className={`${styles.ev__radio} ${on ? styles['ev__radio--on'] : ''}`} />
                                    <span style={{ display: 'flex', flexDirection: 'column', gap: 1, minWidth: 0 }}>
                                      <span className={styles['ev__judge-name']}>{m.name}</span>
                                      <span className={styles['ev__judge-meta']}>
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
                  )}

                  {/* STEP 6 — REVIEW */}
                  {step === 6 && (
                    <>
                      <div className={styles.ev__summary}>
                        <div className={styles['ev__summary-cell']}>
                          <div className={styles['ev__summary-k']}>
                            <Layers size={11} /> Questions
                          </div>
                          <div className={`${styles['ev__summary-v']} ${suite ? '' : styles['ev__summary-v--muted']}`}>
                            {suite ? suite.question_count.toLocaleString() : '—'}
                          </div>
                        </div>
                        <div className={styles['ev__summary-cell']}>
                          <div className={styles['ev__summary-k']}>
                            <Cpu size={11} /> Models
                          </div>
                          <div className={styles['ev__summary-v']}>{selectedModels.length}</div>
                        </div>
                        <div className={styles['ev__summary-cell']}>
                          <div className={styles['ev__summary-k']}>
                            <Target size={11} /> Metrics
                          </div>
                          <div className={styles['ev__summary-v']}>{draft.selMetrics.length}</div>
                        </div>
                      </div>

                      <div className={styles.ev__block}>
                        <p className={styles['ev__block-title']}>
                          <Tag size={11} /> Overview
                        </p>
                        <div className={styles.ev__rows}>
                          <div className={styles.ev__row}>
                            <span>Name</span>
                            <span>{draft.name || '—'}</span>
                          </div>
                          <div className={styles.ev__row}>
                            <span>Type</span>
                            <span>{draft.eval_type || '—'}</span>
                          </div>
                          {agentFramework && (
                            <div className={styles.ev__row}>
                              <span>Framework</span>
                              <span>{AGENT_FRAMEWORKS.find((f) => f.id === agentFramework)?.title}</span>
                            </div>
                          )}
                          <div className={styles.ev__row}>
                            <span>Providers</span>
                            <span>{draft.selProviders.map((id) => providers.find((p) => p.id === id)?.name || id).join(', ') || '—'}</span>
                          </div>
                          <div className={styles.ev__row}>
                            <span>Run samples</span>
                            <span>{runSamples}</span>
                          </div>
                        </div>
                      </div>

                      <div className={styles.ev__block}>
                        <p className={styles['ev__block-title']}>
                          <Cpu size={11} /> Models <b>({selectedModels.length})</b>
                        </p>
                        {selectedModels.length > 0 ? (
                          <div className={styles['ev__review-grid']}>
                            {selectedModels.map((m) => (
                              <div key={m!.id} className={styles['ev__review-card']}>
                                <span className={styles['ev__review-card-icon']}>
                                  <Cpu size={15} />
                                </span>
                                <span style={{ display: 'flex', flexDirection: 'column', gap: 1, minWidth: 0 }}>
                                  <span className={styles['ev__review-card-name']}>{m!.name}</span>
                                  <span className={styles['ev__review-card-sub']}>
                                    {providers.find((p) => p.id === m!.provider_id)?.name || m!.provider_id}
                                  </span>
                                </span>
                              </div>
                            ))}
                          </div>
                        ) : (
                          <p className={styles.ev__empty}>No models selected.</p>
                        )}
                      </div>

                      <div className={styles.ev__block}>
                        <p className={styles['ev__block-title']}>
                          <Database size={11} /> Test suite
                        </p>
                        <div className={styles.ev__rows}>
                          <div className={styles.ev__row}>
                            <span>Suite</span>
                            <span>{suite?.name ?? '—'}</span>
                          </div>
                          {suite?.category && (
                            <div className={styles.ev__row}>
                              <span>Category</span>
                              <span>{suite.category}</span>
                            </div>
                          )}
                          {selSubgroup.length > 0 && (
                            <div className={styles.ev__row}>
                              <span>Subgroups</span>
                              <span>{selSubgroup.join(', ')}</span>
                            </div>
                          )}
                        </div>
                      </div>

                      <div className={styles.ev__block}>
                        <p className={styles['ev__block-title']}>
                          <Target size={11} /> Metrics <b>({draft.selMetrics.length})</b>
                        </p>
                        {draft.selMetrics.length > 0 ? (
                          <div className={styles['ev__metric-tags']}>
                            {draft.selMetrics.map((m) => (
                              <span key={m} className={styles['ev__metric-tag']}>
                                {m}
                              </span>
                            ))}
                          </div>
                        ) : (
                          <p className={styles.ev__empty}>No metrics selected.</p>
                        )}
                      </div>

                      {judgeModel && (
                        <div className={styles.ev__block}>
                          <p className={styles['ev__block-title']}>
                            <Gavel size={11} /> Judge model
                          </p>
                          <div className={styles.ev__rows}>
                            <div className={styles.ev__row}>
                              <span>Model</span>
                              <span>{judgeModel.name}</span>
                            </div>
                          </div>
                        </div>
                      )}

                      {launchError && <p className={styles.ev__error}>{launchError}</p>}
                    </>
                  )}
                </div>
              </div>

              {/* ---- footer nav ---- */}
              <div className={styles.ev__footer}>
                <button
                  type="button"
                  className={`${styles.ev__btn} ${styles['ev__btn--ghost']}`}
                  onClick={() => (step > 0 ? goBack() : navigate('/app/dashboard'))}
                  disabled={launching}
                >
                  <ChevronLeft size={16} /> {step === 0 ? 'Cancel' : 'Back'}
                </button>

                <div style={{ display: 'flex', alignItems: 'center', gap: 16 }}>
                  {step === 0 && canGo() && (
                    <span className={styles.ev__hint}>
                      <kbd>↵</kbd> Enter to continue
                    </span>
                  )}
                  {step < totalSteps - 1 ? (
                    <button type="button" className={`${styles.ev__btn} ${styles['ev__btn--primary']}`} onClick={goNext} disabled={!canGo()}>
                      Continue <ChevronRight size={16} />
                    </button>
                  ) : (
                    <button type="button" className={`${styles.ev__btn} ${styles['ev__btn--launch']}`} onClick={launch} disabled={launching}>
                      {launching ? (
                        <>
                          <Loader2 size={16} className={styles.ev__spin} /> Launching…
                        </>
                      ) : (
                        <>
                          <Play size={16} /> Launch run
                        </>
                      )}
                    </button>
                  )}
                </div>
              </div>
            </section>
          </div>
        </div>
      </div>

      {toast && (
        <div className={styles['ev-toast']}>
          <div className={styles['ev-toast__icon']}>
            <Check size={18} />
          </div>
          <div>
            <div className={styles['ev-toast__title']}>Run launched</div>
            <div className={styles['ev-toast__sub']}>You'll find it in your history once it completes.</div>
          </div>
        </div>
      )}
    </div>
  );
}
