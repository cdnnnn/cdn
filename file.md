import { useEffect, useMemo, useRef, useState } from 'react';
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
  RefreshCw,
  Search,
  Eye,
  X,
  Minus,
  type LucideIcon,
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchProviders } from '../../store/slices/providersSlice';
import { fetchModels, checkModelHealth } from '../../store/slices/modelsSlice';
import { fetchDatasets, uploadDataset, resetUploadStatus } from '../../store/slices/datasetsSlice';
import { SUPPORTED_UPLOAD_EXTENSIONS } from '../../api/endpoints/datasets';
// `evaluationsApi.previewDataset` — GET /datasets/{id}/preview?limit={limit} —
// is a thin, one-off read used only by the Test Suite preview slider below.
// It lives in the evaluations API module (not datasets) and is called
// directly rather than round-tripped through Redux.
import { evaluationsApi } from '../../api/endpoints/evaluations';
import { fetchMetrics } from '../../store/slices/metricsSlice';
import {
  launchEvaluation,
  runAgentBenchmark,
  runAgentBenchmarkMulti,
  setDraft,
  setDraftType,
} from '../../store/slices/evaluationsSlice';
import type { CreateEvaluationRequest, EvaluationDraft, DatasetPreviewResponse } from '../../types';
import styles from './NewEvaluation.module.scss';

// ─────────────────────────────────────────────────────────────────────────
// This component is built against the REAL evaluationsSlice draft shape:
//   { name, type, providers, models, dataset, subgroup, runSamplesMode,
//     runSamples, metrics, judgeModelId, agentFramework }
// `type` is lowercase: 'model' | 'agent' | 'rag' | null.
// setDraftType(type) — clears metrics, and clears agentFramework unless
// type === 'agent' (handled in the slice itself).
// runSamplesMode is 'custom' (default) or 'full' — see runSamplesControl
// below and the `launch` function for how 'full' maps to run_samples: 0.
//
// Other slice assumptions this component depends on:
//
// providersSlice / modelsSlice / datasetsSlice / metricsSlice — lazy fetch:
//   - fetchProviders() is dispatched once, the first time Step 2 is opened.
//   - fetchModels() is dispatched once, the first time Step 3 is opened.
//   - fetchDatasets(type) is dispatched the first time Step 4 is opened for
//     a given dataset type/framework combination.
//   - fetchMetrics(evalType) — GET /metrics?eval_type={type} — is
//     dispatched the first time Step 5 is opened for a given draft.type.
//     Response: { eval_type, metrics: string[], all_metrics: string[] }.
//     Only `all_metrics` is consumed (see metricsCatalog below).
//   - None of the above fire on mount or the instant their prerequisite
//     (e.g. draft.type) is set — only on actually navigating to the step.
//     Each step's "refresh" button bypasses this and calls the thunk
//     directly, any time it's clicked.
//
// modelsSlice — health checks:
//   - checkModelHealth(modelId: string) — GET /models/health/{model_id}
//     Response: { success, message, model_id, response }
//   - state.models.healthById: Record<string, 'idle'|'loading'|'success'|'failed'>
//   - Fired automatically, in parallel, for every model in the current
//     provider selection as soon as Step 3's model list is available (see
//     the auto health-check effect below) — a model can't be selected
//     until its check resolves to 'success'. The manual "Check health"
//     button on each card still works too, e.g. to retry a failure.
//
// datasetsSlice:
//   - fetchDatasets(type: string) — GET /datasets?type={type}
//     `type` is one of: 'model' | 'rag' | 'agent_benchmark' | 'agent_custom'
//     (the last two both represent draft.type === 'agent', distinguished by
//     whether draft.agentFramework is set).
//   - Dataset items carry `dataset_type` (used to detect "custom" datasets)
//     and `dataset_categories: string[]` (used for the subgroup rail).
//
// Search (client-side only, no new endpoints):
//   - Providers/Models/Test Suite/Metrics steps each get a small toolbar
//     with a search box that filters the already-fetched list by name, and
//     a refresh button that re-dispatches that step's existing fetch thunk.
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

// `value` matches draft.type exactly (lowercase); `label` is for display.
const TYPE_OPTIONS: {
  value: EvaluationDraft['type'];
  label: string;
  icon: LucideIcon;
  sub: string;
  variant: string;
  disabled: boolean;
}[] = [
  {
    value: 'model',
    label: 'Model',
    icon: Cpu,
    sub: 'Benchmark a general-purpose LLM on standard tasks like reasoning, coding, and knowledge — ideal for comparing raw model quality across providers.',
    variant: '',
    disabled: false,
  },
  {
    value: 'agent',
    label: 'Agent',
    icon: Bot,
    sub: 'Test an autonomous agent that plans, calls tools, and completes multi-step tasks — measures task completion, not just single-turn output.',
    variant: 'agent',
    disabled: false,
  },
  {
    value: 'rag',
    label: 'RAG',
    icon: Database,
    sub: 'Evaluate a retrieval-augmented pipeline for grounding accuracy — checks how well answers stay faithful to your retrieved context.',
    variant: 'rag',
    disabled: true,
  },
];

// Only Hermes remains as a selectable framework once "Agent" is chosen.
const AGENT_FRAMEWORKS = [
  { id: 'hermes', title: 'Hermes', desc: 'Lightweight tool-calling agent runtime' },
];

const SUGGESTED_NAMES = [
  'Q3 Model Selection',
  'Support Bot Regression',
  'RAG Accuracy v2',
  'GLM-4.6 vs Claude',
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

// Pulls a displayable message out of a rejected createAsyncThunk action,
// regardless of whether that slice uses rejectWithValue (payload is a
// string, or an object with a `message`) or lets RTK's default serializer
// handle it (action.error.message).
function getThunkErrorMessage(action: any, fallback: string): string {
  const payload = action?.payload;
  if (typeof payload === 'string' && payload) return payload;
  if (payload && typeof payload === 'object' && typeof payload.message === 'string' && payload.message) {
    return payload.message;
  }
  return action?.error?.message || fallback;
}

// Preview slider (Change-2): user picks how many sample questions to pull,
// clamped server-side-friendly at 1–20 inclusive.
const PREVIEW_LIMIT_MIN = 1;
const PREVIEW_LIMIT_MAX = 20;
const PREVIEW_LIMIT_DEFAULT = 5;

type DatasetTypeFilter = 'all' | 'custom' | 'deepeval';

const DATASET_TYPE_FILTERS: { value: DatasetTypeFilter; label: string }[] = [
  { value: 'all', label: 'All' },
  { value: 'custom', label: 'Custom' },
  { value: 'deepeval', label: 'Deepeval' },
];

// Skeleton placeholder counts — just enough to plausibly fill the grid
// while a refresh is in flight, not meant to match the real result count.
const SKELETON_CARD_COUNT = 6;
const SKELETON_CHIP_COUNT = 8;

export default function NewEvaluation() {
  const dispatch = useAppDispatch();
  const navigate = useNavigate();
  const [step, setStep] = useState(0);
  const [toast, setToast] = useState(false);
  const [datasetTab, setDatasetTab] = useState<'browse' | 'upload'>('browse');
  const [uploadName, setUploadName] = useState('');
  const [uploadDescription, setUploadDescription] = useState('');
  const [uploadFile, setUploadFile] = useState<File | null>(null);
  const [uploadFileError, setUploadFileError] = useState<string | null>(null);
  const totalSteps = STEPS.length;

  // ---- search (client-side filter) + refresh (re-fetch) state, one pair
  // per searchable step: Providers, Models, Test Suite, Metrics. ----------
  const [providerSearch, setProviderSearch] = useState('');
  const [modelSearch, setModelSearch] = useState('');
  const [datasetSearch, setDatasetSearch] = useState('');
  const [metricSearch, setMetricSearch] = useState('');

  const [providersLoading, setProvidersLoading] = useState(false);
  const [modelsLoading, setModelsLoading] = useState(false);
  const [datasetsRefreshing, setDatasetsRefreshing] = useState(false);
  const [metricsRefreshing, setMetricsRefreshing] = useState(false);
  // Tracked locally rather than read from the datasets slice's own `error`
  // field — that field wasn't reliably cleared on a subsequent successful
  // fetch, so a stale error from an earlier failed attempt kept showing
  // even after Refresh (or renavigating) succeeded. Deriving this purely
  // from the outcome of our own dispatch calls below fixes that.
  const [datasetsErrorLocal, setDatasetsErrorLocal] = useState<string | null>(null);

  // ---- Test Suite: All/Custom/Deepeval filter (Change-1) -----------------
  const [datasetTypeFilter, setDatasetTypeFilter] = useState<DatasetTypeFilter>('all');

  // ---- Test Suite: preview slider (Change-2) ------------------------------
  const [previewOpen, setPreviewOpen] = useState(false);
  const [previewDatasetId, setPreviewDatasetId] = useState<string | null>(null);
  const [previewDatasetName, setPreviewDatasetName] = useState<string>('');
  const [previewLimit, setPreviewLimit] = useState(PREVIEW_LIMIT_DEFAULT);
  const [previewLoading, setPreviewLoading] = useState(false);
  const [previewError, setPreviewError] = useState<string | null>(null);
  const [previewData, setPreviewData] = useState<DatasetPreviewResponse | null>(null);

  const draft = useAppSelector((s) => s.evaluations.draft);
  const launching = useAppSelector((s) => s.evaluations.launching);
  const launchError = useAppSelector((s) => s.evaluations.launchError);

  const providers = useAppSelector((s) => s.providers.items) ?? [];
  const models = useAppSelector((s) => s.models.items) ?? [];
  const healthById = useAppSelector((s) => (s.models as any).healthById) as Record<string, HealthStatus> | undefined;

  const metricsState = useAppSelector((s) => s.metrics) ?? { allMetrics: [], status: 'idle' as const, error: null };
  // Only `all_metrics` from the API response is used — it's the full
  // catalog rendered as selectable chips. Loading state for this step is
  // tracked locally (metricsRefreshing) rather than read from
  // metricsState.status, for the same staleness reason as datasets below.
  const metricsCatalog: string[] = (metricsState as any).allMetrics ?? [];

  const datasets = useAppSelector((s) => s.datasets.items) ?? [];
  const datasetUploading = useAppSelector((s) => s.datasets.uploadStatus === 'loading');
  const datasetUploadError = useAppSelector((s) => s.datasets.uploadError);

  // ---- lazy fetch guards: each API is only called the first time the
  // user actually navigates to the step that needs it, not on mount and
  // not the moment its prerequisite (e.g. draft.type) is set. Refresh
  // buttons bypass these guards entirely (they call the thunk directly).
  const providersFetchedRef = useRef(false);
  const modelsFetchedRef = useRef(false);
  const datasetsFetchedForTypeRef = useRef<string | null>(null);
  const metricsFetchedForTypeRef = useRef<string | null>(null);

  // GET /providers — fetched once, the first time Step 2 is reached. If
  // that fetch fails, the marker is rolled back so leaving and coming back
  // to Step 2 retries it automatically, instead of silently reusing a
  // failed attempt forever (the Refresh button already bypasses this).
  useEffect(() => {
    if (step !== 2 || providersFetchedRef.current) return;
    providersFetchedRef.current = true;
    (async () => {
      setProvidersLoading(true);
      const result = await dispatch(fetchProviders());
      setProvidersLoading(false);
      if (fetchProviders.rejected.match(result)) {
        providersFetchedRef.current = false;
      }
    })();
  }, [step, dispatch]);

  // GET /models — fetched once, the first time Step 3 is reached (by then
  // providers are already selected, since Step 2 requires it to advance).
  // Same retry-on-failure behavior as providers above.
  useEffect(() => {
    if (step !== 3 || modelsFetchedRef.current) return;
    modelsFetchedRef.current = true;
    (async () => {
      setModelsLoading(true);
      const result = await dispatch(fetchModels());
      setModelsLoading(false);
      if (fetchModels.rejected.match(result)) {
        modelsFetchedRef.current = false;
      }
    })();
  }, [step, dispatch]);

  // ---- (1) dataset "type" query param, split for Agent by framework -------
  // Model/RAG: type = draft.type
  // Agent, no framework chosen:  type = 'agent_benchmark'
  // Agent, framework chosen:     type = 'agent_custom'
  const datasetType = useMemo(() => {
    if (!draft.type) return '';
    if (draft.type === 'agent') {
      return draft.agentFramework ? 'agent_custom' : 'agent_benchmark';
    }
    return draft.type;
  }, [draft.type, draft.agentFramework]);

  // GET /datasets?type={type} — fetched the first time Step 4 is reached
  // for a given type/framework combination. If the user goes back and
  // changes type/framework, the pool is stale, so a different datasetType
  // triggers one refetch the next time Step 4 is (re)entered. If the fetch
  // itself fails, the marker is rolled back so re-selecting the *same*
  // type and coming back to Step 4 retries automatically — previously a
  // failed attempt permanently "used up" the marker for that type, so
  // going back and forward again silently reused the failure.
  useEffect(() => {
    if (step !== 4 || !datasetType) return;
    if (datasetsFetchedForTypeRef.current === datasetType) return;
    datasetsFetchedForTypeRef.current = datasetType;
    (async () => {
      setDatasetsRefreshing(true);
      const result = await dispatch(fetchDatasets(datasetType));
      setDatasetsRefreshing(false);
      if (fetchDatasets.rejected.match(result)) {
        datasetsFetchedForTypeRef.current = null;
        setDatasetsErrorLocal(getThunkErrorMessage(result, 'Failed to load test suites'));
      } else {
        setDatasetsErrorLocal(null);
      }
    })();
  }, [step, datasetType, dispatch]);

  // Any previously chosen dataset is invalid once the dataset "type" changes
  // (different type/framework combination = different dataset pool). Also
  // clear any in-progress dataset search since it applied to the old pool.
  // This is a pure UI-state reset, independent of the fetch timing above.
  useEffect(() => {
    if (!datasetType) return;
    dispatch(setDraft({ dataset: null }));
    setDatasetSearch('');
    setDatasetTypeFilter('all');
    setDatasetsErrorLocal(null);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [datasetType]);

  // GET /metrics?eval_type={type} — fetched the first time Step 5 is
  // reached for a given draft.type. Only `all_metrics` is consumed (see
  // metricsCatalog below) — `metrics` is ignored entirely. Same
  // retry-on-failure rollback as datasets above.
  useEffect(() => {
    if (step !== 5 || !draft.type) return;
    if (metricsFetchedForTypeRef.current === draft.type) return;
    metricsFetchedForTypeRef.current = draft.type;
    (async () => {
      setMetricsRefreshing(true);
      const result = await dispatch(fetchMetrics(draft.type));
      setMetricsRefreshing(false);
      if (fetchMetrics.rejected.match(result)) {
        metricsFetchedForTypeRef.current = null;
      }
    })();
  }, [step, draft.type, dispatch]);

  const suite = datasets.find((d) => d.id === draft.dataset);

  // ---- (6) auto-select every subgroup on dataset pick ----------------------
  useEffect(() => {
    const cats = (suite as any)?.dataset_categories ?? [];
    dispatch(setDraft({ subgroup: cats }));
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [draft.dataset]);

  const selectAllSubgroups = () => dispatch(setDraft({ subgroup: (suite as any)?.dataset_categories ?? [] }));
  const clearAllSubgroups = () => dispatch(setDraft({ subgroup: [] }));

  const connectedProviders = providers.filter((p) => p.status === 'connected');
  const filteredProviders = useMemo(
    () => connectedProviders.filter((p) => p.name.toLowerCase().includes(providerSearch.trim().toLowerCase())),
    [connectedProviders, providerSearch]
  );

  const availableModels = useMemo(
    () => models.filter((m) => draft.providers.includes(m.provider_id)),
    [models, draft.providers]
  );
  const filteredModels = useMemo(
    () => availableModels.filter((m) => m.name.toLowerCase().includes(modelSearch.trim().toLowerCase())),
    [availableModels, modelSearch]
  );

  // Counts per dataset_type, for the filter chip labels — computed off the
  // full (unsearched, unfiltered) list so the counts don't shift as the
  // user types in the search box.
  const datasetTypeCounts = useMemo(
    () => ({
      all: datasets.length,
      custom: datasets.filter((d) => (d as any).dataset_type === 'custom').length,
      deepeval: datasets.filter((d) => (d as any).dataset_type === 'deepeval').length,
    }),
    [datasets]
  );

  const filteredDatasets = useMemo(
    () =>
      datasets.filter((d) => {
        const matchesSearch = d.name.toLowerCase().includes(datasetSearch.trim().toLowerCase());
        const matchesType = datasetTypeFilter === 'all' || (d as any).dataset_type === datasetTypeFilter;
        return matchesSearch && matchesType;
      }),
    [datasets, datasetSearch, datasetTypeFilter]
  );

  const filteredMetrics = useMemo(
    () => metricsCatalog.filter((m) => m.toLowerCase().includes(metricSearch.trim().toLowerCase())),
    [metricsCatalog, metricSearch]
  );

  // ---- refresh handlers: re-dispatch each step's existing fetch thunk ----
  // Each sets its own `*Refreshing` flag so the step can swap its grid for
  // skeleton placeholders while the request is in flight (Change-3).
  const refreshProviders = async () => {
    setProvidersLoading(true);
    await dispatch(fetchProviders());
    setProvidersLoading(false);
  };

  const refreshModels = async () => {
    setModelsLoading(true);
    await dispatch(fetchModels());
    setModelsLoading(false);
  };

  const refreshDatasets = async () => {
    if (!datasetType) return;
    setDatasetsRefreshing(true);
    const result = await dispatch(fetchDatasets(datasetType));
    setDatasetsRefreshing(false);
    if (fetchDatasets.rejected.match(result)) {
      datasetsFetchedForTypeRef.current = null;
      setDatasetsErrorLocal(getThunkErrorMessage(result, 'Failed to load test suites'));
    } else {
      datasetsFetchedForTypeRef.current = datasetType;
      setDatasetsErrorLocal(null);
    }
  };

  const refreshMetrics = async () => {
    if (!draft.type) return;
    setMetricsRefreshing(true);
    const result = await dispatch(fetchMetrics(draft.type));
    setMetricsRefreshing(false);
    if (fetchMetrics.rejected.match(result)) {
      metricsFetchedForTypeRef.current = null;
    } else {
      metricsFetchedForTypeRef.current = draft.type;
    }
  };

  // ---- preview slider (Change-2): GET /datasets/{id}/preview?limit=N -----
  const openPreview = (datasetId: string, datasetName: string) => {
    setPreviewDatasetId(datasetId);
    setPreviewDatasetName(datasetName);
    setPreviewLimit(PREVIEW_LIMIT_DEFAULT);
    setPreviewData(null);
    setPreviewError(null);
    setPreviewOpen(true);
  };

  const closePreview = () => {
    setPreviewOpen(false);
  };

  // Clamp keystrokes/typed values to the 1–20 range instead of just
  // rejecting them, so the field never silently ignores input.
  const setClampedPreviewLimit = (raw: number) => {
    if (Number.isNaN(raw)) return;
    setPreviewLimit(Math.min(PREVIEW_LIMIT_MAX, Math.max(PREVIEW_LIMIT_MIN, Math.round(raw))));
  };

  const runPreview = async () => {
    if (!previewDatasetId) return;
    setPreviewLoading(true);
    setPreviewError(null);
    try {
      const data = await evaluationsApi.previewDataset(previewDatasetId, previewLimit);
      setPreviewData(data);
    } catch (err) {
      const detail =
        (err as { response?: { data?: { detail?: string } } })?.response?.data?.detail ||
        (err as Error)?.message ||
        'Failed to load preview';
      setPreviewError(detail);
    } finally {
      setPreviewLoading(false);
    }
  };

  // Manual, single-model health check — still available via the "Check
  // health" button on each card, alongside the automatic parallel check
  // below.
  const runHealthCheck = (modelId: string) => {
    dispatch(checkModelHealth(modelId));
  };

  // Auto health-check: as soon as the models list for the currently
  // selected providers is available (and while Step 3 is open), fire a
  // parallel health check for every visible model — the user isn't
  // expected to click "Check health" one by one. Guarded on the raw
  // `models` array reference so it re-runs once per fresh fetch (initial
  // navigation-triggered fetch, or a manual refresh) rather than on every
  // render or on unrelated state changes like typing in the search box.
  const autoHealthCheckedForModelsRef = useRef<typeof models | null>(null);
  useEffect(() => {
    if (step !== 3 || availableModels.length === 0) return;
    if (autoHealthCheckedForModelsRef.current === models) return;
    autoHealthCheckedForModelsRef.current = models;
    availableModels.forEach((m) => dispatch(checkModelHealth(m.id)));
  }, [step, models, availableModels, dispatch]);

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
    Boolean(uploadName.trim()) && Boolean(uploadFile) && !uploadFileError && Boolean(draft.type) && !datasetUploading;

  const submitUpload = async () => {
    if (!uploadFile || !canUpload) return;
    const result = await dispatch(
      uploadDataset({
        file: uploadFile,
        name: uploadName.trim(),
        description: uploadDescription.trim(),
        evalType: datasetType,
      })
    );
    if (uploadDataset.fulfilled.match(result)) {
      dispatch(setDraft({ dataset: result.payload.id }));
      setDatasetTab('browse');
    }
  };

  // ---- (5) Model type + non-custom dataset ⇒ hide metrics & judge ---------
  const isCustomDataset = (suite as any)?.dataset_type === 'custom';
  const modelHidesMetrics = draft.type === 'model' && Boolean(suite) && !isCustomDataset;

  // Clear any selected metrics the moment this simplified mode kicks in, so
  // neither the manifest nor the launch payload carries stale selections.
  useEffect(() => {
    if (modelHidesMetrics && draft.metrics.length > 0) {
      dispatch(setDraft({ metrics: [] }));
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [modelHidesMetrics]);

  const selectedModels = draft.models.map((id) => models.find((m) => m.id === id)).filter(Boolean) as typeof models;
  const judgeModel = draft.judgeModelId ? models.find((m) => m.id === draft.judgeModelId) : null;

  // The Judge Model panel — and picking a judge at all — is only relevant
  // when the LLM_Judge metric has been selected (and metrics aren't hidden
  // entirely per the rule above). In every other case judge_config must be
  // sent as {} on launch.
  const requiresJudge = !modelHidesMetrics && draft.metrics.includes('LLM_Judge');

  // If the user deselects LLM_Judge after having picked a judge, clear the
  // stale selection so it doesn't silently linger in the manifest/payload.
  useEffect(() => {
    if (!requiresJudge && draft.judgeModelId) {
      dispatch(setDraft({ judgeModelId: null }));
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [requiresJudge]);

  const isModelSelectable = (modelId: string) => healthById?.[modelId] === 'success';

  const toggleModel = (modelId: string) => {
    const alreadySelected = draft.models.includes(modelId);
    if (!alreadySelected && !isModelSelectable(modelId)) return;
    dispatch(setDraft({ models: toggle(draft.models, modelId) }));
  };

  const canGo = () => {
    if (step === 0) return Boolean(draft.name.trim());
    if (step === 1) return Boolean(draft.type);
    if (step === 2) return draft.providers.length > 0;
    if (step === 3) return draft.models.length > 0;
    if (step === 4) return Boolean(draft.dataset);
    if (step === 5) return modelHidesMetrics || !requiresJudge || Boolean(draft.judgeModelId);
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

  // ---- (3) & (4) launch: three different endpoints depending on type ------
  const launch = async () => {
    const dataset = datasets.find((d) => d.id === draft.dataset);
    const judgeModelObj = draft.judgeModelId ? models.find((m) => m.id === draft.judgeModelId) : undefined;
    // Change: "Full" mode means "use the whole dataset" — the backend
    // contract for that is run_samples: 0, regardless of whatever number
    // was last typed into the Custom field.
    const effectiveRunSamples = draft.runSamplesMode === 'full' ? 0 : draft.runSamples;

    let result: any;

    if (draft.type === 'agent' && !draft.agentFramework) {
      // POST /agent-benchmark/run
      result = await dispatch(
        runAgentBenchmark({
          dataset_id: dataset?.id || '',
          model_ids: draft.models,
          evaluation_name: draft.name,
          run_samples: effectiveRunSamples,
        })
      );
    } else if (draft.type === 'agent' && draft.agentFramework) {
      // POST /agent-benchmark/run-multi
      result = await dispatch(
        runAgentBenchmarkMulti({
          dataset_id: dataset?.id || '',
          model_ids: draft.models,
          evaluation_name: draft.name,
          selected_metrics: draft.metrics,
          selected_categories: draft.subgroup,
          run_samples: effectiveRunSamples,
        })
      );
    } else {
      // POST /evaluations then /evaluations/{id}/start — Model or RAG
      const payload: CreateEvaluationRequest = {
        name: draft.name,
        eval_type: draft.type || '',
        dataset_id: dataset?.id || '',
        benchmark: dataset?.name || undefined,
        model_ids: draft.models,
        selected_metrics: modelHidesMetrics ? [] : draft.metrics,
        run_samples: effectiveRunSamples,
        selected_category: draft.subgroup.length > 0 ? draft.subgroup : dataset ? [dataset.category] : undefined,
        // Only populated when the LLM_Judge metric is selected AND a judge
        // model has been chosen — every other case sends an empty object.
        judge_config:
          requiresJudge && draft.judgeModelId
            ? {
                model_id: draft.judgeModelId,
                base_url: judgeModelObj?.base_url || '',
                api_key: draft.judgeModelId,
              }
            : {},
      };
      result = await dispatch(launchEvaluation(payload));
    }

    const succeeded =
      launchEvaluation.fulfilled.match(result) ||
      runAgentBenchmark.fulfilled.match(result) ||
      runAgentBenchmarkMulti.fulfilled.match(result);

    if (succeeded) {
      setToast(true);
      setTimeout(() => {
        setToast(false);
        navigate('/app/history');
      }, 2000);
    }
  };

  const progressPct = Math.round((step / (totalSteps - 1)) * 100);

  // ---- live Run Manifest values (one per step) ----------------------------
  const providerNames = draft.providers.map((id) => providers.find((p) => p.id === id)?.name || id);
  const mf = (value: string, filled: boolean) => ({ value: filled ? value : '—', empty: !filled });
  const typeLabel = TYPE_OPTIONS.find((o) => o.value === draft.type)?.label ?? '';
  const frameworkTitle = draft.agentFramework ? AGENT_FRAMEWORKS.find((f) => f.id === draft.agentFramework)?.title : null;
  const manifest = [
    mf(draft.name, Boolean(draft.name)),
    mf(frameworkTitle ? `${typeLabel} · ${frameworkTitle}` : typeLabel, Boolean(draft.type)),
    mf(draft.providers.length === 1 ? providerNames[0] : `${draft.providers.length} providers`, draft.providers.length > 0),
    mf(`${draft.models.length} models`, draft.models.length > 0),
    mf(suite?.name || '', Boolean(suite)),
    mf(modelHidesMetrics ? 'Not required' : `${draft.metrics.length} metrics`, modelHidesMetrics || draft.metrics.length > 0),
    mf(
      modelHidesMetrics
        ? 'Ready to launch'
        : judgeModel
        ? `Judge · ${judgeModel.name}`
        : requiresJudge
        ? 'Judge required'
        : 'Ready to launch',
      modelHidesMetrics || !requiresJudge || Boolean(judgeModel)
    ),
  ];

  const CrumbIcon = STEP_ICONS[step];

  // ---- shared "Run samples" control (Custom / Full) ------------------------
  // Used by both branches of Step 5 (the simplified model-benchmark view and
  // the full metrics view) so the toggle behaves identically in either.
  // 'full' means "use the whole dataset" — run_samples is sent as 0 in that
  // case (see `launch` below), regardless of whatever number was last typed
  // into the Custom field.
  const runSamplesControl = (
    <div className={`${styles.ev__field} ${styles['ev__field--samples']}`}>
      <label className={styles.ev__label}>Run samples</label>
      <div className={styles['ev__radio-row']}>
        <button
          type="button"
          className={`${styles['ev__radio-opt']} ${draft.runSamplesMode === 'custom' ? styles['ev__radio-opt--on'] : ''}`}
          onClick={() => dispatch(setDraft({ runSamplesMode: 'custom' }))}
        >
          <span className={`${styles.ev__radio} ${draft.runSamplesMode === 'custom' ? styles['ev__radio--on'] : ''}`} />
          Custom
        </button>
        <button
          type="button"
          className={`${styles['ev__radio-opt']} ${draft.runSamplesMode === 'full' ? styles['ev__radio-opt--on'] : ''}`}
          onClick={() => dispatch(setDraft({ runSamplesMode: 'full' }))}
        >
          <span className={`${styles.ev__radio} ${draft.runSamplesMode === 'full' ? styles['ev__radio--on'] : ''}`} />
          Full
        </button>
      </div>
      {draft.runSamplesMode === 'custom' ? (
        <input
          type="number"
          min={0}
          className={styles.ev__input}
          style={{ marginTop: 10 }}
          value={draft.runSamples}
          onChange={(e) => {
            const val = e.target.value === '' ? 0 : Math.max(0, Number(e.target.value));
            dispatch(setDraft({ runSamples: Number.isNaN(val) ? 0 : val }));
          }}
        />
      ) : (
        <p className={styles['ev__radio-full-note']}>Every question in the suite will be used — no count needed.</p>
      )}
    </div>
  );

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
                          const on = draft.type === o.value;
                          return (
                            <button
                              key={o.value}
                              type="button"
                              className={`${styles.ev__option} ${on ? styles['ev__option--on'] : ''} ${
                                o.disabled ? styles['ev__option--off'] : ''
                              }`}
                              onClick={() => !o.disabled && dispatch(setDraftType(o.value))}
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
                                  {o.label}
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

                      {draft.type === 'agent' && (
                        <div className={styles.ev__section}>
                          <label className={styles.ev__label}>
                            <Waypoints size={13} /> Agent framework <span className="opt">optional</span>
                          </label>
                          <p className={styles['ev__section-hint']}>
                            Tell us which framework the agent runs on, if applicable. This also determines which test
                            suites are available in the next steps.
                          </p>
                          <div className={styles['ev__fw-grid']}>
                            {AGENT_FRAMEWORKS.map((f) => {
                              const on = draft.agentFramework === f.id;
                              return (
                                <button
                                  key={f.id}
                                  type="button"
                                  className={`${styles.ev__fw} ${on ? styles['ev__fw--on'] : ''}`}
                                  onClick={() => dispatch(setDraft({ agentFramework: on ? null : f.id }))}
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
                      <div className={styles['ev__step-toolbar']}>
                        <div className={styles['ev__toolbar-search']}>
                          <Search size={14} />
                          <input
                            placeholder="Search providers…"
                            value={providerSearch}
                            onChange={(e) => setProviderSearch(e.target.value)}
                          />
                        </div>
                        <button
                          type="button"
                          className={styles['ev__toolbar-refresh']}
                          onClick={refreshProviders}
                          disabled={providersLoading}
                          title="Refresh providers"
                        >
                          <RefreshCw size={14} className={providersLoading ? styles.ev__spin : ''} />
                        </button>
                      </div>
                      {providersLoading ? (
                        <div className={styles.ev__grid} aria-busy="true" aria-label="Refreshing providers">
                          {Array.from({ length: SKELETON_CARD_COUNT }).map((_, i) => (
                            <div key={i} className={styles['ev__skel-pcard']}>
                              <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--icon']}`} />
                              <span className={styles['ev__skel-lines']}>
                                <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--line']}`} style={{ width: '70%' }} />
                                <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--line']}`} style={{ width: '45%' }} />
                                <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--pill']}`} />
                              </span>
                            </div>
                          ))}
                        </div>
                      ) : (
                        <div className={styles.ev__grid}>
                          {filteredProviders.map((p) => {
                            const on = draft.providers.includes(p.id);
                            return (
                              <button
                                key={p.id}
                                type="button"
                                className={`${styles.ev__pcard} ${on ? styles['ev__pcard--on'] : ''}`}
                                onClick={() => dispatch(setDraft({ providers: toggle(draft.providers, p.id) }))}
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
                          {connectedProviders.length > 0 && filteredProviders.length === 0 && (
                            <p className={styles.ev__empty}>No providers match "{providerSearch}".</p>
                          )}
                        </div>
                      )}
                    </div>
                  )}

                  {/* STEP 3 — MODELS */}
                  {step === 3 &&
                    (availableModels.length > 0 ? (
                      <div className={styles.ev__scroll}>
                        <div className={styles['ev__step-toolbar']}>
                          <div className={styles['ev__toolbar-search']}>
                            <Search size={14} />
                            <input
                              placeholder="Search models…"
                              value={modelSearch}
                              onChange={(e) => setModelSearch(e.target.value)}
                            />
                          </div>
                          <button
                            type="button"
                            className={styles['ev__toolbar-refresh']}
                            onClick={refreshModels}
                            disabled={modelsLoading}
                            title="Refresh models"
                          >
                            <RefreshCw size={14} className={modelsLoading ? styles.ev__spin : ''} />
                          </button>
                        </div>
                        {modelsLoading ? (
                          <div className={`${styles.ev__grid} ${styles['ev__grid--wide']}`} aria-busy="true" aria-label="Refreshing models">
                            {Array.from({ length: SKELETON_CARD_COUNT }).map((_, i) => (
                              <div key={i} className={styles['ev__skel-mcard']}>
                                <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--line']}`} style={{ width: '60%', height: 14 }} />
                                <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--line']}`} style={{ width: '35%' }} />
                                <span className={styles['ev__skel-caps']}>
                                  <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--tag']}`} />
                                  <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--tag']}`} />
                                </span>
                                <span className={styles['ev__skel-stats']}>
                                  <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--stat']}`} />
                                  <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--stat']}`} />
                                </span>
                              </div>
                            ))}
                          </div>
                        ) : (
                        <div className={`${styles.ev__grid} ${styles['ev__grid--wide']}`}>
                          {filteredModels.map((m) => {
                            const on = draft.models.includes(m.id);
                            const health: HealthStatus = healthById?.[m.id] ?? 'idle';
                            const selectable = health === 'success';
                            const caps = (m as any).capabilities as string[] | undefined;
                            const inputPrice = (m as any).input_price as number | null | undefined;
                            const outputPrice = (m as any).output_price as number | null | undefined;
                            const accuracy = (m as any).accuracy_score as number | null | undefined;
                            const providerName = providers.find((p) => p.id === m.provider_id)?.name ?? m.provider_id;

                            return (
                              // Not a <button> — it contains a nested "Check health"
                              // control, so it's a clickable div with keyboard support
                              // instead (nested interactive elements aren't valid HTML).
                              <div
                                key={m.id}
                                role="button"
                                tabIndex={0}
                                className={`${styles.ev__mcard} ${on ? styles['ev__mcard--on'] : ''} ${
                                  !selectable && !on ? styles['ev__mcard--locked'] : ''
                                } ${health === 'loading' ? styles['ev__mcard--checking'] : ''}`}
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
                          {filteredModels.length === 0 && (
                            <p className={styles.ev__empty}>No models match "{modelSearch}".</p>
                          )}
                        </div>
                        )}
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
                            <div className={styles['ev__step-toolbar']}>
                              <div className={styles['ev__toolbar-search']}>
                                <Search size={14} />
                                <input
                                  placeholder="Search test suites…"
                                  value={datasetSearch}
                                  onChange={(e) => setDatasetSearch(e.target.value)}
                                />
                              </div>
                              <button
                                type="button"
                                className={styles['ev__toolbar-refresh']}
                                onClick={refreshDatasets}
                                disabled={datasetsRefreshing}
                                title="Refresh test suites"
                              >
                                <RefreshCw size={14} className={datasetsRefreshing ? styles.ev__spin : ''} />
                              </button>
                            </div>

                            {/* Change-1: All / Custom / Deepeval filter */}
                            <div className={styles['ev__filter-row']}>
                              {DATASET_TYPE_FILTERS.map((f) => {
                                const on = datasetTypeFilter === f.value;
                                const count = datasetTypeCounts[f.value];
                                return (
                                  <button
                                    key={f.value}
                                    type="button"
                                    className={`${styles['ev__filter-chip']} ${on ? styles['ev__filter-chip--on'] : ''}`}
                                    onClick={() => setDatasetTypeFilter(f.value)}
                                  >
                                    {f.label}
                                    <span className={styles['ev__filter-chip-count']}>{count}</span>
                                  </button>
                                );
                              })}
                            </div>

                            {(() => {
                              // Both the busy flag and the error message are
                              // tracked locally (set from the outcome of our
                              // own dispatch calls) rather than read from the
                              // datasets slice's own status/error fields —
                              // see datasetsErrorLocal above for why.
                              if (datasetsRefreshing) {
                                return (
                                  <div className={styles.ev__dgrid} aria-busy="true" aria-label="Loading test suites">
                                    {Array.from({ length: SKELETON_CARD_COUNT }).map((_, i) => (
                                      <div key={i} className={styles['ev__skel-dcard']}>
                                        <span className={styles['ev__skel-dcard-top']}>
                                          <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--icon-sm']}`} />
                                          <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--line']}`} style={{ width: '55%' }} />
                                        </span>
                                        <span className={styles['ev__skel-caps']}>
                                          <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--tag']}`} />
                                          <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--tag']}`} />
                                          <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--tag']}`} />
                                        </span>
                                      </div>
                                    ))}
                                  </div>
                                );
                              }

                              if (datasetsErrorLocal) {
                                return <p className={styles.ev__error}>{datasetsErrorLocal}</p>;
                              }

                              return (
                                <div className={styles.ev__dgrid}>
                                  {filteredDatasets.map((d) => {
                                    const on = draft.dataset === d.id;
                                    const isCustom = (d as any).dataset_type === 'custom';
                                    const isDeepeval = (d as any).dataset_type === 'deepeval';
                                    return (
                                      // Not a <button> — it now contains a nested "Preview"
                                      // control, so it's a clickable div with keyboard
                                      // support instead (mirrors the model card pattern;
                                      // nested interactive elements aren't valid HTML).
                                      <div
                                        key={d.id}
                                        role="button"
                                        tabIndex={0}
                                        className={`${styles.ev__dcard} ${on ? styles['ev__dcard--on'] : ''}`}
                                        onClick={() => dispatch(setDraft({ dataset: d.id }))}
                                        onKeyDown={(e) => {
                                          if (e.key === 'Enter' || e.key === ' ') {
                                            e.preventDefault();
                                            dispatch(setDraft({ dataset: d.id }));
                                          }
                                        }}
                                        aria-pressed={on}
                                      >
                                        <div className={styles['ev__dcard-top']}>
                                          <div className={styles['ev__dcard-id']}>
                                            <span className={styles['ev__dcard-icon']}>
                                              <Database size={15} />
                                            </span>
                                            <span className={styles['ev__dcard-name']}>{d.name}</span>
                                          </div>
                                          <div className={styles['ev__dcard-actions']}>
                                            <button
                                              type="button"
                                              className={styles['ev__dcard-preview-btn']}
                                              onClick={(e) => {
                                                e.stopPropagation();
                                                openPreview(d.id, d.name);
                                              }}
                                              title="Preview sample questions"
                                            >
                                              <Eye size={13} /> Preview
                                            </button>
                                            {on && (
                                              <span className={styles['ev__mcard-mark']}>
                                                <Check size={12} strokeWidth={3} />
                                              </span>
                                            )}
                                          </div>
                                        </div>
                                        <div className={styles['ev__dcard-tags']}>
                                          <span className={styles.ev__tag}>{d.category}</span>
                                          <span className={styles.ev__tag}>{d.eval_type}</span>
                                          {/* Change-1: both dataset_type values now get a tag —
                                              previously only 'custom' rendered one. */}
                                          {isCustom && (
                                            <span className={`${styles.ev__tag} ${styles['ev__tag--custom']}`}>Custom</span>
                                          )}
                                          {isDeepeval && (
                                            <span className={`${styles.ev__tag} ${styles['ev__tag--deepeval']}`}>Deepeval</span>
                                          )}
                                          <span className={`${styles.ev__tag} ${styles['ev__tag--count']}`}>
                                            {d.question_count.toLocaleString()} questions
                                          </span>
                                        </div>
                                      </div>
                                    );
                                  })}
                                  {datasets.length === 0 && (
                                    <p className={styles.ev__empty}>No test suites available for this type yet.</p>
                                  )}
                                  {datasets.length > 0 && filteredDatasets.length === 0 && (
                                    <p className={styles.ev__empty}>
                                      No test suites match {datasetSearch ? `"${datasetSearch}"` : 'this filter'}.
                                    </p>
                                  )}
                                </div>
                              );
                            })()}
                          </div>

                          <aside className={styles.ev__rail}>
                            <div className={styles['ev__rail-head']}>
                              <div className={styles['ev__rail-head-row']}>
                                <p className={styles['ev__rail-title']}>
                                  <Layers size={13} /> Subgroups
                                </p>
                                {suite && (suite as any).dataset_categories?.length > 0 && (
                                  <div className={styles['ev__rail-actions']}>
                                    <button type="button" className={styles.ev__link} onClick={selectAllSubgroups}>
                                      Select all
                                    </button>
                                    <button type="button" className={styles.ev__link} onClick={clearAllSubgroups}>
                                      Unselect all
                                    </button>
                                  </div>
                                )}
                              </div>
                              <p className={styles['ev__rail-sub']}>
                                {suite
                                  ? `All of "${suite.name}"'s categories are selected by default — narrow as needed.`
                                  : 'Select a suite to see its subgroups.'}
                              </p>
                            </div>
                            <div className={styles['ev__rail-scroll']}>
                              {!suite && <p className={styles['ev__rail-empty']}>No suite selected yet.</p>}
                              {suite && (suite as any).dataset_categories?.length === 0 && (
                                <p className={styles['ev__rail-empty']}>This suite has no subgroups.</p>
                              )}
                              {suite &&
                                ((suite as any).dataset_categories ?? []).map((cat: string) => {
                                  const on = draft.subgroup.includes(cat);
                                  return (
                                    <button
                                      key={cat}
                                      type="button"
                                      className={`${styles['ev__check-row']} ${on ? styles['ev__check-row--on'] : ''}`}
                                      onClick={() => dispatch(setDraft({ subgroup: toggle(draft.subgroup, cat) }))}
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
                            <input className={styles.ev__input} value={draft.type || '—'} disabled readOnly />
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
                    <>
                      {modelHidesMetrics ? (
                        // (5) Model type + non-custom dataset: metrics & judge model
                        // are not applicable — only run samples is configurable.
                        <div style={{ maxWidth: 300 }}>
                          {runSamplesControl}
                          <p className={styles['ev__samples-note']}>
                            Metrics and a judge model aren\u2019t configurable for standard (non-custom) model
                            benchmarks.
                          </p>
                        </div>
                      ) : (
                        <div className={`${styles.ev__metrics} ${!requiresJudge ? styles['ev__metrics--single'] : ''}`}>
                          <div className={styles['ev__metrics-main']}>
                            <div className={styles.ev__samples}>
                              {runSamplesControl}
                              <p className={styles['ev__samples-note']}>Questions sampled from the suite for each model.</p>
                            </div>

                            <div className={styles['ev__metrics-bar']}>
                              <span className={styles['ev__metrics-count']}>
                                <b>{draft.metrics.length}</b> of {metricsCatalog.length} selected
                              </span>
                              <div className={styles['ev__metrics-actions']}>
                                <button
                                  type="button"
                                  className={styles.ev__link}
                                  onClick={() => dispatch(setDraft({ metrics: [...metricsCatalog] }))}
                                >
                                  Select all
                                </button>
                                <button type="button" className={styles.ev__link} onClick={() => dispatch(setDraft({ metrics: [] }))}>
                                  Clear
                                </button>
                              </div>
                            </div>

                            <div className={styles['ev__step-toolbar']}>
                              <div className={styles['ev__toolbar-search']}>
                                <Search size={14} />
                                <input
                                  placeholder="Search metrics…"
                                  value={metricSearch}
                                  onChange={(e) => setMetricSearch(e.target.value)}
                                />
                              </div>
                              <button
                                type="button"
                                className={styles['ev__toolbar-refresh']}
                                onClick={refreshMetrics}
                                disabled={metricsRefreshing}
                                title="Refresh metrics"
                              >
                                <RefreshCw size={14} className={metricsRefreshing ? styles.ev__spin : ''} />
                              </button>
                            </div>

                            {metricsRefreshing ? (
                              <div className={styles.ev__chips} aria-busy="true" aria-label="Loading metrics">
                                {Array.from({ length: SKELETON_CHIP_COUNT }).map((_, i) => (
                                  <span
                                    key={i}
                                    className={`${styles['ev__skel-block']} ${styles['ev__skel-block--chip']}`}
                                    style={{ width: 64 + ((i * 37) % 90) }}
                                  />
                                ))}
                              </div>
                            ) : (
                              <div className={styles.ev__chips}>
                                {filteredMetrics.map((name: string) => {
                                  const on = draft.metrics.includes(name);
                                  return (
                                    <button
                                      key={name}
                                      type="button"
                                      className={`${styles.ev__chip} ${on ? styles['ev__chip--on'] : ''}`}
                                      onClick={() => dispatch(setDraft({ metrics: toggle(draft.metrics, name) }))}
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
                                {metricsCatalog.length === 0 && (
                                  <p className={styles.ev__empty}>No metrics available for this type.</p>
                                )}
                                {metricsCatalog.length > 0 && filteredMetrics.length === 0 && (
                                  <p className={styles.ev__empty}>No metrics match "{metricSearch}".</p>
                                )}
                              </div>
                            )}
                          </div>

                          {requiresJudge && (
                            <aside className={styles.ev__judge}>
                              <div className={styles['ev__judge-head']}>
                                <p className={styles['ev__judge-title']}>
                                  <Gavel size={13} /> Judge model
                                </p>
                                <p className={styles['ev__judge-sub']}>
                                  Required — the LLM_Judge metric needs a model to grade open-ended answers.
                                </p>
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
                                          onClick={() => dispatch(setDraft({ judgeModelId: on ? null : m.id }))}
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
                              {!draft.judgeModelId && (
                                <p className={styles['ev__judge-required']}>
                                  Select a judge model to continue — it's mandatory when LLM_Judge is selected.
                                </p>
                              )}
                            </aside>
                          )}
                        </div>
                      )}
                    </>
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
                          <div className={styles['ev__summary-v']}>{modelHidesMetrics ? '—' : draft.metrics.length}</div>
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
                            <span>{typeLabel || '—'}</span>
                          </div>
                          {draft.agentFramework && (
                            <div className={styles.ev__row}>
                              <span>Framework</span>
                              <span>{AGENT_FRAMEWORKS.find((f) => f.id === draft.agentFramework)?.title}</span>
                            </div>
                          )}
                          <div className={styles.ev__row}>
                            <span>Providers</span>
                            <span>{draft.providers.map((id) => providers.find((p) => p.id === id)?.name || id).join(', ') || '—'}</span>
                          </div>
                          <div className={styles.ev__row}>
                            <span>Run samples</span>
                            <span>{draft.runSamplesMode === 'full' ? 'Full dataset' : draft.runSamples}</span>
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
                          {draft.subgroup.length > 0 && (
                            <div className={styles.ev__row}>
                              <span>Subgroups</span>
                              <span>{draft.subgroup.join(', ')}</span>
                            </div>
                          )}
                        </div>
                      </div>

                      {!modelHidesMetrics && (
                        <div className={styles.ev__block}>
                          <p className={styles['ev__block-title']}>
                            <Target size={11} /> Metrics <b>({draft.metrics.length})</b>
                          </p>
                          {draft.metrics.length > 0 ? (
                            <div className={styles['ev__metric-tags']}>
                              {draft.metrics.map((m) => (
                                <span key={m} className={styles['ev__metric-tag']}>
                                  {m}
                                </span>
                              ))}
                            </div>
                          ) : (
                            <p className={styles.ev__empty}>No metrics selected.</p>
                          )}
                        </div>
                      )}

                      {requiresJudge && (
                        <div className={styles.ev__block}>
                          <p className={styles['ev__block-title']}>
                            <Gavel size={11} /> Judge model
                          </p>
                          <div className={styles.ev__rows}>
                            <div className={styles.ev__row}>
                              <span>Model</span>
                              <span>{judgeModel ? judgeModel.name : '—'}</span>
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

      {/* Change-2: dataset preview slider — right-to-left panel showing
          GET /datasets/{id}/preview?limit={limit} for the clicked suite. */}
      {previewOpen && (
        <div className={styles['ev-preview-overlay']} onClick={closePreview}>
          <aside
            className={styles['ev-preview-panel']}
            role="dialog"
            aria-modal="true"
            aria-label={`Preview ${previewDatasetName}`}
            onClick={(e) => e.stopPropagation()}
          >
            <div className={styles['ev-preview-head']}>
              <div className={styles['ev-preview-head-main']}>
                <p className={styles['ev-preview-eyebrow']}>Test suite preview</p>
                <h3 className={styles['ev-preview-title']}>{previewDatasetName || 'Dataset'}</h3>
              </div>
              <button type="button" className={styles['ev-preview-close']} onClick={closePreview} title="Close preview">
                <X size={16} />
              </button>
            </div>

            <div className={styles['ev-preview-controls']}>
              <div className={styles['ev-preview-limit']}>
                <label className={styles['ev-preview-limit-label']}>
                  Questions to preview <span className={styles['ev-preview-limit-range']}>({PREVIEW_LIMIT_MIN}–{PREVIEW_LIMIT_MAX})</span>
                </label>
                <div className={styles['ev-preview-limit-controls']}>
                  <button
                    type="button"
                    className={styles['ev-preview-stepper-btn']}
                    onClick={() => setClampedPreviewLimit(previewLimit - 1)}
                    disabled={previewLimit <= PREVIEW_LIMIT_MIN}
                    aria-label="Decrease"
                  >
                    <Minus size={14} />
                  </button>
                  <input
                    type="number"
                    className={styles['ev-preview-limit-input']}
                    min={PREVIEW_LIMIT_MIN}
                    max={PREVIEW_LIMIT_MAX}
                    value={previewLimit}
                    onChange={(e) => setClampedPreviewLimit(Number(e.target.value))}
                  />
                  <button
                    type="button"
                    className={styles['ev-preview-stepper-btn']}
                    onClick={() => setClampedPreviewLimit(previewLimit + 1)}
                    disabled={previewLimit >= PREVIEW_LIMIT_MAX}
                    aria-label="Increase"
                  >
                    <Plus size={14} />
                  </button>
                </div>
              </div>
              <button
                type="button"
                className={`${styles.ev__btn} ${styles['ev__btn--primary']}`}
                onClick={runPreview}
                disabled={previewLoading}
              >
                {previewLoading ? (
                  <>
                    <Loader2 size={15} className={styles.ev__spin} /> Loading…
                  </>
                ) : previewData ? (
                  <>
                    <RefreshCw size={15} /> Reload
                  </>
                ) : (
                  <>
                    <Eye size={15} /> Load preview
                  </>
                )}
              </button>
            </div>

            <div className={styles['ev-preview-body']}>
              {previewError && <p className={styles.ev__error}>{previewError}</p>}

              {previewLoading && (
                <div className={styles['ev-preview-skel-list']} aria-busy="true" aria-label="Loading preview">
                  {Array.from({ length: Math.min(previewLimit, 5) }).map((_, i) => (
                    <div key={i} className={styles['ev-preview-skel-card']}>
                      <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--line']}`} style={{ width: '85%' }} />
                      <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--line']}`} style={{ width: '60%' }} />
                      <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--line']}`} style={{ width: '40%' }} />
                    </div>
                  ))}
                </div>
              )}

              {!previewLoading && !previewError && !previewData && (
                <p className={styles.ev__empty}>Pick how many questions to sample, then load the preview.</p>
              )}

              {!previewLoading && !previewError && previewData && previewData.questions.length === 0 && (
                <p className={styles.ev__empty}>This suite returned no sample questions.</p>
              )}

              {!previewLoading &&
                !previewError &&
                previewData &&
                previewData.questions.length > 0 && (
                  <div className={styles['ev-preview-list']}>
                    {previewData.questions.map((q, i) => (
                      <div key={q.id ?? i} className={styles['ev-preview-q']}>
                        <div className={styles['ev-preview-q-head']}>
                          <span className={styles['ev-preview-q-index']}>Q{i + 1}</span>
                          {q.category && <span className={styles['ev-preview-q-cat']}>{q.category}</span>}
                        </div>
                        {q.input?.prompt && <p className={styles['ev-preview-q-prompt']}>{q.input.prompt}</p>}
                        {Array.isArray(q.choices) && q.choices.length > 0 && (
                          <ul className={styles['ev-preview-q-choices']}>
                            {q.choices.map((c, ci) => (
                              <li key={ci}>{c}</li>
                            ))}
                          </ul>
                        )}
                        {q.expected?.answer !== undefined && (
                          <p className={styles['ev-preview-q-answer']}>
                            <span>Expected</span> {String(q.expected.answer)}
                          </p>
                        )}
                      </div>
                    ))}
                  </div>
                )}
            </div>
          </aside>
        </div>
      )}

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
