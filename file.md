import { Component, useEffect, useMemo, useRef, useState, type ErrorInfo, type ReactNode } from 'react';
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
  AlertTriangle,
  Info,
  Sparkles,
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
import type {
  CreateEvaluationRequest,
  EvaluationDraft,
  DatasetPreviewResponse,
  CustomMetric,
  ModelRetryConfig,
} from '../../types';
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
//   - fetchDatasets(type: string) — GET /datasets?eval_type={type}
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
    disabled: false,
  },
];

// Only Hermes remains as a selectable framework once "Agent" is chosen.
const AGENT_FRAMEWORKS = [
  { id: 'hermes', title: 'Hermes', desc: 'Lightweight tool-calling agent runtime' },
];

// Per-model retry/timeout config (Models step) — bounds and defaults for
// the two fields shown on each selected model card.
const MIN_MAX_RETRIES = 1;
const MAX_MAX_RETRIES = 20;
const DEFAULT_MAX_RETRIES = 1;
const MIN_TIMEOUT = 60;
const MAX_TIMEOUT = 900;
const DEFAULT_TIMEOUT = 60;
const DEFAULT_MODEL_RETRY_CONFIG: ModelRetryConfig = { max_retries: DEFAULT_MAX_RETRIES, timeout: DEFAULT_TIMEOUT };
// Pure clamp helper — used both on blur (see updateModelRetryConfig/
// updateRetryConfigAll below) and as a final safety net in `launch()` in
// case the payload is built while a field is still mid-edit (e.g. the user
// clicks "Launch" without blurring the input first).
const clampRetryField = (field: keyof ModelRetryConfig, value: number): number => {
  const min = field === 'max_retries' ? MIN_MAX_RETRIES : MIN_TIMEOUT;
  const max = field === 'max_retries' ? MAX_MAX_RETRIES : MAX_TIMEOUT;
  return Math.min(max, Math.max(min, Number.isNaN(value) ? min : value));
};
const clampModelRetryConfigValue = (config: ModelRetryConfig): ModelRetryConfig => ({
  max_retries: clampRetryField('max_retries', config.max_retries),
  timeout: clampRetryField('timeout', config.timeout),
});

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

function formatContextWindow(tokens: number | null | undefined): string {
  if (tokens === null || tokens === undefined || Number.isNaN(tokens)) return '—';
  if (tokens >= 1_000_000) return `${(tokens / 1_000_000).toLocaleString()}M`;
  if (tokens >= 1_000) return `${Math.round(tokens / 1000)}k`;
  return `${tokens}`;
}

function formatPrice(price: number | null | undefined): string {
  return price === null || price === undefined || Number.isNaN(price) ? '—' : `$${price.toFixed(2)}`;
}

function providerInitials(name: string | null | undefined): string {
  const safeName = name?.trim() || '';
  if (!safeName) return '??';
  const parts = safeName.replace(/[^a-zA-Z0-9 ]/g, '').split(' ').filter(Boolean);
  const letters = parts.slice(0, 2).map((w) => w[0]).join('');
  return (letters || safeName.slice(0, 2)).toUpperCase();
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
// Preview slider (Change-2, later moved to offset-based pagination): fixed
// page size of 20 questions per page, starting at offset 0. Total page
// count is derived from the selected dataset's own `question_count`
// (surfaced in the /datasets list) rather than from the preview response.
const PREVIEW_PAGE_SIZE = 20;

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

// ---------------------------------------------------------------------------
// StepErrorBoundary — catches render-time exceptions thrown while rendering
// a single wizard step (e.g. an unexpected null/undefined field from the
// API) and shows a small recoverable card in place of just that step's
// content, instead of the whole page going blank. Must be a class component
// — React only supports error boundaries via getDerivedStateFromError /
// componentDidCatch, there's no hook equivalent.
//
// Rendered with `key={step}-${retryKey}` by the caller, so navigating to a
// different step (or clicking "Try again", which bumps retryKey) always
// remounts a fresh instance with hasError reset — no manual reset wiring
// needed here.
// ---------------------------------------------------------------------------
interface StepErrorBoundaryProps {
  children: ReactNode;
  onRetry: () => void;
  onBack: () => void;
  canGoBack: boolean;
}
interface StepErrorBoundaryState {
  hasError: boolean;
}
class StepErrorBoundary extends Component<StepErrorBoundaryProps, StepErrorBoundaryState> {
  state: StepErrorBoundaryState = { hasError: false };

  static getDerivedStateFromError(): StepErrorBoundaryState {
    return { hasError: true };
  }

  componentDidCatch(error: Error, info: ErrorInfo) {
    // eslint-disable-next-line no-console
    console.error('[NewEvaluation] step failed to render:', error, info.componentStack);
  }

  render() {
    if (!this.state.hasError) return this.props.children;
    return (
      <div className={styles.ev__error} style={{ flexDirection: 'column', alignItems: 'flex-start', gap: 10 }}>
        <div style={{ display: 'flex', alignItems: 'center', gap: 8 }}>
          <AlertTriangle size={16} />
          <strong>This step hit an unexpected error.</strong>
        </div>
        <p style={{ margin: 0, fontWeight: 400 }}>
          Your progress on earlier steps is safe. Try again, or go back and retry from there.
        </p>
        <div style={{ display: 'flex', gap: 10 }}>
          <button type="button" className={`${styles.ev__btn} ${styles['ev__btn--primary']}`} onClick={this.props.onRetry}>
            Try again
          </button>
          {this.props.canGoBack && (
            <button type="button" className={`${styles.ev__btn} ${styles['ev__btn--ghost']}`} onClick={this.props.onBack}>
              Back
            </button>
          )}
        </div>
      </div>
    );
  }
}

export default function NewEvaluation() {
  const dispatch = useAppDispatch();
  const navigate = useNavigate();
  const [step, setStep] = useState(0);
  // Bumped by StepErrorBoundary's "Try again" button to force a fresh
  // remount of the current step's content without changing `step` itself.
  const [stepRetryKey, setStepRetryKey] = useState(0);
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

  // ---- Test Suite: preview slider (Change-2, now paginated) --------------
  const [previewOpen, setPreviewOpen] = useState(false);
  const [previewDatasetId, setPreviewDatasetId] = useState<string | null>(null);
  const [previewDatasetName, setPreviewDatasetName] = useState<string>('');
  // Total known question count for the dataset being previewed — comes
  // from the dataset card (Dataset.question_count), not from the preview
  // response itself. Used only to compute how many pages to offer.
  const [previewQuestionCount, setPreviewQuestionCount] = useState(0);
  const [previewOffset, setPreviewOffset] = useState(0);
  const [previewLoading, setPreviewLoading] = useState(false);
  const [previewError, setPreviewError] = useState<string | null>(null);
  const [previewData, setPreviewData] = useState<DatasetPreviewResponse | null>(null);

  // ---- Metrics step: "Generate Instruction" ------------------------------
  const [instructionGenerating, setInstructionGenerating] = useState(false);
  const [instructionError, setInstructionError] = useState<string | null>(null);

  const draft = useAppSelector((s) => s.evaluations.draft);
  const launching = useAppSelector((s) => s.evaluations.launching);
  const launchError = useAppSelector((s) => s.evaluations.launchError);

  const providers = useAppSelector((s) => s.providers.items) ?? [];
  const models = useAppSelector((s) => s.models.items) ?? [];
  const healthById = useAppSelector((s) => (s.models as any).healthById) as Record<string, HealthStatus> | undefined;

  const metricsState = useAppSelector((s) => s.metrics) ?? { allMetrics: [], customMetrics: [], status: 'idle' as const, error: null };
  // Only `all_metrics` from the API response is used for the built-in
  // catalog — it's the full list rendered as selectable chips. Loading
  // state for this step is tracked locally (metricsRefreshing) rather than
  // read from metricsState.status, for the same staleness reason as
  // datasets below.
  const metricsCatalog: string[] = (metricsState as any).allMetrics ?? [];
  // `custom` from the same response, normalized by metricsSlice into
  // state.metrics.customMetrics — richer objects (id/name/description/
  // required_judge/etc.), rendered as their own section below the built-in
  // metrics chips. Selection stores the metric's `id` (not its `name`) in
  // draft.metrics, alongside the plain metric-name strings from
  // metricsCatalog — both end up in the same selected_metrics array on
  // launch, the backend only cares that each entry resolves to a metric.
  const customMetricsCatalog: CustomMetric[] = (metricsState as any).customMetrics ?? [];
  const customMetricsForType = useMemo(
    () => customMetricsCatalog.filter((c) => !Array.isArray(c?.eval_types) || c.eval_types.length === 0 || (draft.type && c.eval_types.includes(draft.type))),
    [customMetricsCatalog, draft.type]
  );

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

  // GET /datasets?eval_type={type} — fetched the first time Step 4 is reached
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

  const suite = datasets.find((d) => d?.id === draft.dataset);

  // ---- (6) auto-select every subgroup on dataset pick ----------------------
  useEffect(() => {
    const cats = (suite as any)?.dataset_categories ?? [];
    dispatch(setDraft({ subgroup: cats }));
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [draft.dataset]);

  const selectAllSubgroups = () => dispatch(setDraft({ subgroup: (suite as any)?.dataset_categories ?? [] }));
  const clearAllSubgroups = () => dispatch(setDraft({ subgroup: [] }));

  // ---- Run Samples default/max, driven by the selected dataset's size ----
  // More than 10 questions available: default to 10, cap the input at the
  // dataset's total. 10 or fewer available: default (and cap) at the
  // dataset's total, since there's nothing more to sample.
  const selectedDatasetQuestionCount = suite?.question_count ?? 0;
  const maxRunSamples = Math.max(1, selectedDatasetQuestionCount);
  const defaultRunSamples = selectedDatasetQuestionCount > 10 ? 10 : maxRunSamples;

  // Re-derive the default whenever the selected dataset itself changes
  // (not on every render) — matches the same trigger as the subgroup
  // auto-select effect above, and won't stomp on a value the user typed
  // in manually unless they've actually picked a different dataset since.
  useEffect(() => {
    if (!draft.dataset) return;
    dispatch(setDraft({ runSamples: defaultRunSamples }));
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [draft.dataset]);

  const connectedProviders = providers.filter((p) => p?.status === 'connected');
  const filteredProviders = useMemo(
    () => connectedProviders.filter((p) => (p?.name ?? '').toLowerCase().includes(providerSearch.trim().toLowerCase())),
    [connectedProviders, providerSearch]
  );

  const availableModels = useMemo(
    () => models.filter((m) => draft.providers.includes(m?.provider_id)),
    [models, draft.providers]
  );
  const filteredModels = useMemo(
    () => availableModels.filter((m) => (m?.name ?? '').toLowerCase().includes(modelSearch.trim().toLowerCase())),
    [availableModels, modelSearch]
  );

  // Counts per dataset_type, for the filter chip labels — computed off the
  // full (unsearched, unfiltered) list so the counts don't shift as the
  // user types in the search box.
  const datasetTypeCounts = useMemo(
    () => ({
      all: datasets.length,
      custom: datasets.filter((d) => (d as any)?.dataset_type === 'custom').length,
      deepeval: datasets.filter((d) => (d as any)?.dataset_type === 'deepeval').length,
    }),
    [datasets]
  );

  const filteredDatasets = useMemo(
    () =>
      datasets.filter((d) => {
        const matchesSearch = (d?.name ?? '').toLowerCase().includes(datasetSearch.trim().toLowerCase());
        const matchesType = datasetTypeFilter === 'all' || (d as any)?.dataset_type === datasetTypeFilter;
        return matchesSearch && matchesType;
      }),
    [datasets, datasetSearch, datasetTypeFilter]
  );

  const filteredMetrics = useMemo(
    () => metricsCatalog.filter((m) => (m ?? '').toLowerCase().includes(metricSearch.trim().toLowerCase())),
    [metricsCatalog, metricSearch]
  );

  const filteredCustomMetrics = useMemo(
    () => customMetricsForType.filter((c) => (c?.name ?? '').toLowerCase().includes(metricSearch.trim().toLowerCase())),
    [customMetricsForType, metricSearch]
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

  // ---- preview slider (Change-2, paginated): GET /datasets/{id}/preview?
  // limit=20&offset={offset} ---------------------------------------------
  const previewTotalPages = Math.max(1, Math.ceil(previewQuestionCount / PREVIEW_PAGE_SIZE));
  const previewCurrentPage = Math.floor(previewOffset / PREVIEW_PAGE_SIZE) + 1;
  const previewHasPrevPage = previewOffset > 0;
  const previewHasNextPage = previewCurrentPage < previewTotalPages;

  const openPreview = (datasetId: string, datasetName: string, questionCount: number) => {
    setPreviewDatasetId(datasetId);
    setPreviewDatasetName(datasetName);
    setPreviewQuestionCount(questionCount || 0);
    setPreviewOffset(0);
    setPreviewData(null);
    setPreviewError(null);
    setPreviewOpen(true);
  };

  const closePreview = () => {
    setPreviewOpen(false);
  };

  const goToPreviewPrevPage = () => {
    setPreviewOffset((o) => Math.max(0, o - PREVIEW_PAGE_SIZE));
  };

  const goToPreviewNextPage = () => {
    setPreviewOffset((o) => {
      const maxOffset = Math.max(0, (previewTotalPages - 1) * PREVIEW_PAGE_SIZE);
      return Math.min(maxOffset, o + PREVIEW_PAGE_SIZE);
    });
  };

  // Fetches whatever page `previewOffset` currently points at. Called both
  // by the auto-fetch effect below (on open / page change) and by the
  // manual reload button (same page, fresh data).
  const fetchPreviewPage = async (datasetId: string, offset: number) => {
    setPreviewLoading(true);
    setPreviewError(null);
    try {
      const data = await evaluationsApi.previewDataset(datasetId, PREVIEW_PAGE_SIZE, offset);
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

  // Auto-loads a page whenever the slider opens or the page changes — no
  // separate "Load preview" click needed, Prev/Next just work.
  useEffect(() => {
    if (!previewOpen || !previewDatasetId) return;
    fetchPreviewPage(previewDatasetId, previewOffset);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [previewOpen, previewDatasetId, previewOffset]);

  // ---- Metrics step: "Generate Instruction" -------------------------------
  // POST /evaluations/generate-instruction — optional, pre-fills the
  // instruction textarea; the user can edit or overwrite it freely
  // afterward either way.
  const generateInstruction = async () => {
    if (!draft.type || !draft.dataset || draft.models.length === 0) return;

    setInstructionGenerating(true);
    setInstructionError(null);
    try {
      // 1. Model with the largest context window among the ones selected
      // in Step 3 — not necessarily the judge model, just whichever
      // selected model can hold the most context for this call.
      const selectedModelObjs = draft.models
        .map((id) => models.find((m) => m?.id === id))
        .filter(Boolean) as typeof models;
      let bestModel: (typeof models)[number] | null = null;
      for (const m of selectedModelObjs) {
        const mCtx = m.context_window ?? 0;
        const bestCtx = bestModel?.context_window ?? -1;
        if (mCtx > bestCtx) bestModel = m;
      }
      if (!bestModel) {
        setInstructionError('None of the selected models could be found — try reselecting them in Step 4.');
        return;
      }

      // 2. Always a fresh 5-question, offset-0 sample — independent of
      // whatever page the preview slider (if ever opened) last showed.
      const sample = await evaluationsApi.previewDataset(draft.dataset, 5, 0);
      const questions = (Array.isArray(sample?.questions) ? sample.questions : []).map((q) => ({
        input: q?.input,
        expected: q?.expected,
      }));

      // 3. eval_type is just draft.type, chosen back in Step 2.
      const response = await evaluationsApi.generateInstruction({
        model_id: bestModel.id,
        eval_type: draft.type,
        questions,
      });
      dispatch(setDraft({ instruction: response.instruction || '' }));
    } catch (err) {
      const detail =
        (err as { response?: { data?: { detail?: string } } })?.response?.data?.detail ||
        (err as Error)?.message ||
        'Failed to generate an instruction';
      setInstructionError(detail);
    } finally {
      setInstructionGenerating(false);
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

  // Built-in metrics (plain name strings) and custom metrics (ids) share
  // the same draft.metrics array, so "select all" / "clear" for one
  // section must only touch its own entries — otherwise selecting all
  // built-in metrics would silently wipe out any chosen custom metrics
  // (and vice versa).
  const customMetricIdSet = new Set(customMetricsForType.map((c) => c.id));
  const selectAllBuiltinMetrics = () => {
    const keptCustom = draft.metrics.filter((m) => customMetricIdSet.has(m));
    dispatch(setDraft({ metrics: [...keptCustom, ...metricsCatalog] }));
  };
  const clearBuiltinMetrics = () => {
    dispatch(setDraft({ metrics: draft.metrics.filter((m) => customMetricIdSet.has(m)) }));
  };
  const selectAllCustomMetrics = () => {
    const builtinSet = new Set(metricsCatalog);
    const keptBuiltin = draft.metrics.filter((m) => builtinSet.has(m));
    dispatch(setDraft({ metrics: [...keptBuiltin, ...customMetricsForType.map((c) => c.id)] }));
  };
  const clearCustomMetrics = () => {
    dispatch(setDraft({ metrics: draft.metrics.filter((m) => !customMetricIdSet.has(m)) }));
  };

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

    // Hermes uploads (Agent type, agentFramework selected) go through
    // /upload-jsonl (for .jsonl files, which also needs category: 'Agents')
    // or /upload (any other supported extension) — both variants expect
    // eval_type: 'agent' rather than the wizard's internal 'agent_custom'
    // dataset-type discriminator. Every other case (Model, RAG, Agent
    // benchmark with no framework) keeps sending datasetType unchanged.
    const isHermesUpload = isAgentWithFramework;
    const uploadFileExt = getFileExtension(uploadFile.name);

    const result = await dispatch(
      uploadDataset({
        file: uploadFile,
        name: uploadName.trim(),
        description: uploadDescription.trim(),
        evalType: isHermesUpload ? 'agent' : datasetType,
        ...(isHermesUpload && uploadFileExt === 'jsonl' ? { category: 'Agents' } : {}),
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

  // Agent type with no framework selected (draft.agentFramework null) maps
  // to POST /agent-benchmark/run, which only ever accepts dataset_id,
  // model_ids, evaluation_name, run_samples — there's no metrics or judge
  // concept for this path at all, so the Metrics step (and the Test
  // Suite step's Upload tab, further below) simplify down the same way
  // the Model + standard-dataset case does.
  const isAgentBenchmarkNoFramework = draft.type === 'agent' && !draft.agentFramework;
  const hideMetricsStep = modelHidesMetrics || isAgentBenchmarkNoFramework;

  // RAG datasets are always pre-built retrieval corpora — there's no
  // "upload your own" flow for them (only Top K applies, further below).
  const isRag = draft.type === 'rag';
  const hideUploadTab = isAgentBenchmarkNoFramework || isRag;

  // The Upload tab isn't offered for Agent benchmarks with no framework
  // selected, or for RAG (see Test Suite step below) — if the user had it
  // open and then goes back and changes type/framework into one of those
  // states, snap back to Browse so there's no dangling reference to a
  // hidden tab.
  useEffect(() => {
    if (hideUploadTab && datasetTab === 'upload') {
      setDatasetTab('browse');
    }
  }, [hideUploadTab, datasetTab]);

  // Clear any selected metrics the moment this simplified mode kicks in, so
  // neither the manifest nor the launch payload carries stale selections.
  useEffect(() => {
    if (hideMetricsStep && draft.metrics.length > 0) {
      dispatch(setDraft({ metrics: [] }));
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [hideMetricsStep]);

  const selectedModels = draft.models.map((id) => models.find((m) => m?.id === id)).filter(Boolean) as typeof models;
  const judgeModel = draft.judgeModelId ? models.find((m) => m?.id === draft.judgeModelId) : null;

  // Agent type WITH a framework selected (draft.agentFramework truthy, e.g.
  // Hermes) maps to POST /agent-benchmark/run-multi — used below to pick
  // the right upload payload shape, not for judge-related logic anymore
  // (see the always-on judge model requirement, further below).
  const isAgentWithFramework = draft.type === 'agent' && Boolean(draft.agentFramework);

  const isModelSelectable = (modelId: string) => healthById?.[modelId] === 'success';

  // Judge model options — reuse the exact same "available" definition as
  // Step 3 (health check passed), scoped to the same provider selection,
  // instead of filtering by is_active. Previously this used
  // `models.filter(is_active)`, which could surface models that never
  // passed (or never ran) a health check, or exclude ones that did but
  // happen to have is_active === false.
  const judgeCandidateModels = useMemo(
    () => availableModels.filter((m) => isModelSelectable(m.id)),
    [availableModels, healthById]
  );

  useEffect(() => {
    if (draft.judgeModelId && !judgeCandidateModels.some((m) => m.id === draft.judgeModelId)) {
      dispatch(setDraft({ judgeModelId: null }));
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [judgeCandidateModels, draft.judgeModelId]);

  const toggleModel = (modelId: string) => {
    const alreadySelected = draft.models.includes(modelId);
    if (!alreadySelected && !isModelSelectable(modelId)) return;
    dispatch(setDraft({ models: toggle(draft.models, modelId) }));
    if (alreadySelected) {
      // Deselecting — drop its retry config so draft.modelRetryConfig
      // always mirrors draft.models exactly.
      const next = { ...draft.modelRetryConfig };
      delete next[modelId];
      dispatch(setDraft({ modelRetryConfig: next }));
    } else if (!draft.modelRetryConfig[modelId]) {
      // Selecting — seed defaults so the payload always has an entry for
      // every selected model, even if the user never touches the fields.
      dispatch(
        setDraft({
          modelRetryConfig: { ...draft.modelRetryConfig, [modelId]: { ...DEFAULT_MODEL_RETRY_CONFIG } },
        })
      );
    }
  };

  // Update a selected model's retry config field while the user is typing —
  // stores exactly what's on screen (no clamping) so intermediate/out-of-
  // range values while typing (e.g. clearing the field, or typing "6" on
  // the way to "65") aren't immediately overwritten and don't fight the
  // user's keystrokes. Range is enforced on blur (see below). Clicks/
  // keystrokes inside the config panel call e.stopPropagation() so they
  // don't also toggle the card's selection (see the card's onClick).
  const updateModelRetryConfig = (modelId: string, field: keyof ModelRetryConfig, raw: number) => {
    const current = draft.modelRetryConfig[modelId] || { ...DEFAULT_MODEL_RETRY_CONFIG };
    const safe = Number.isNaN(raw) ? 0 : raw;
    dispatch(
      setDraft({
        modelRetryConfig: { ...draft.modelRetryConfig, [modelId]: { ...current, [field]: safe } },
      })
    );
  };

  // Clamps a selected model's retry config field into its valid range —
  // called on blur, once the user has finished typing.
  const clampModelRetryConfig = (modelId: string, field: keyof ModelRetryConfig) => {
    const current = draft.modelRetryConfig[modelId] || { ...DEFAULT_MODEL_RETRY_CONFIG };
    const clamped = clampRetryField(field, current[field]);
    if (clamped === current[field]) return;
    dispatch(
      setDraft({
        modelRetryConfig: { ...draft.modelRetryConfig, [modelId]: { ...current, [field]: clamped } },
      })
    );
  };

  // Same pattern as above, but for the single shared "apply to all" config
  // used when retryConfigMode === 'all'.
  const updateRetryConfigAll = (field: keyof ModelRetryConfig, raw: number) => {
    const safe = Number.isNaN(raw) ? 0 : raw;
    dispatch(setDraft({ retryConfigAll: { ...draft.retryConfigAll, [field]: safe } }));
  };

  const clampRetryConfigAll = (field: keyof ModelRetryConfig) => {
    const clamped = clampRetryField(field, draft.retryConfigAll[field]);
    if (clamped === draft.retryConfigAll[field]) return;
    dispatch(setDraft({ retryConfigAll: { ...draft.retryConfigAll, [field]: clamped } }));
  };

  const canGo = () => {
    if (step === 0) return Boolean(draft.name.trim());
    if (step === 1) return Boolean(draft.type);
    if (step === 2) return draft.providers.length > 0;
    if (step === 3) return draft.models.length > 0;
    // A dataset with 0 questions can't actually be run against — block
    // continuing until one with at least one question is picked.
    if (step === 4) return Boolean(draft.dataset) && (suite?.question_count ?? 0) > 0;
    if (step === 5) {
      // Judge model is mandatory for every type now — no more LLM_Judge /
      // hideMetricsStep / agent-framework gating.
      if (!draft.judgeModelId) return false;
      // If the metrics catalog (built-in or custom) actually has anything
      // in it for this type, picking at least one is mandatory too. When
      // the whole metrics section is hidden (hideMetricsStep) or the
      // catalog is genuinely empty, there's nothing to require.
      const metricsAvailable = !hideMetricsStep && (metricsCatalog.length > 0 || customMetricsForType.length > 0);
      if (metricsAvailable && draft.metrics.length === 0) return false;
      return true;
    }
    return true;
  };

  const goNext = () => {
    if (!canGo()) return;
    setStep((s) => Math.min(totalSteps - 1, s + 1));
  };
  // Navigating backward away from the Metrics step clears any instruction
  // that was there — whether typed or generated — so coming back to it
  // later (after e.g. changing the dataset or models on an earlier step)
  // never shows a stale instruction that no longer matches the current
  // selections. Forward navigation (goNext, and reaching Review) leaves it
  // untouched, since that's the normal "continue with what I have" path.
  const goBack = () => {
    if (step === 5) dispatch(setDraft({ instruction: '' }));
    setStep((s) => Math.max(0, s - 1));
  };
  const goToStep = (target: number) => {
    if (target < step) {
      if (step === 5) dispatch(setDraft({ instruction: '' }));
      setStep(target);
    }
  };

  // ---- (3) & (4) launch: three different endpoints depending on type ------
  const launch = async () => {
    const dataset = datasets.find((d) => d?.id === draft.dataset);
    const judgeModelObj = draft.judgeModelId ? models.find((m) => m?.id === draft.judgeModelId) : undefined;
    // Change: "Full" mode means "use the whole dataset" — for the agent
    // benchmark endpoints the backend contract for that is run_samples: 0.
    const effectiveRunSamples = draft.runSamplesMode === 'full' ? 0 : draft.runSamples;
    // For POST /evaluations (Model/RAG) specifically, "Full" sends
    // run_samples as null — the backend's contract for "use the whole
    // dataset" on this endpoint, distinct from the agent-benchmark
    // endpoints above which use 0 for the same concept.
    const createEvalRunSamples: number | null = draft.runSamplesMode === 'full' ? null : draft.runSamples;

    let result: any;

    // Same "Apply to all" / "Individually" shape used for the Model/RAG
    // create-evaluation payload below — reused here for the two
    // agent-benchmark endpoints so retry/timeout behaves identically
    // regardless of which API call the Agent step ends up making.
    // Clamped defensively in case a field is still mid-edit when Launch
    // is clicked.
    const retryConfigPayload: Pick<CreateEvaluationRequest, 'max_retries' | 'timeout' | 'model_retry_config'> =
      draft.retryConfigMode === 'all'
        ? clampModelRetryConfigValue(draft.retryConfigAll)
        : {
            model_retry_config: draft.models.reduce<Record<string, ModelRetryConfig>>((acc, id) => {
              acc[id] = clampModelRetryConfigValue(draft.modelRetryConfig[id] || DEFAULT_MODEL_RETRY_CONFIG);
              return acc;
            }, {}),
          };

    if (draft.type === 'agent' && !draft.agentFramework) {
      // POST /agent-benchmark/run
      result = await dispatch(
        runAgentBenchmark({
          dataset_id: dataset?.id || '',
          model_ids: draft.models,
          evaluation_name: draft.name,
          run_samples: effectiveRunSamples,
          ...retryConfigPayload,
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
          ...retryConfigPayload,
        })
      );
    } else {
      // POST /evaluations then /evaluations/{id}/start — Model or RAG

      // The Models step's "Apply to all" / "Individually" toggle decides
      // whether retry/timeout for the judge model comes from the shared
      // config or from that specific model's own entry (falling back to
      // defaults if the judge model was never one of the selected models,
      // so it never got its own individual entry). Clamped defensively in
      // case a field is still mid-edit (not yet blurred) when Launch is
      // clicked.
      const judgeRetryConfig: ModelRetryConfig = clampModelRetryConfigValue(
        draft.retryConfigMode === 'all'
          ? draft.retryConfigAll
          : (draft.judgeModelId && draft.modelRetryConfig[draft.judgeModelId]) || DEFAULT_MODEL_RETRY_CONFIG
      );

      const payload: CreateEvaluationRequest = {
        name: draft.name,
        eval_type: draft.type || '',
        dataset_id: dataset?.id || '',
        // RAG doesn't use `benchmark` — omitted entirely for that type
        // rather than sent as the dataset name (or null), matching how
        // top_k is likewise only ever included for RAG below.
        ...(isRag ? {} : { benchmark: dataset?.name || undefined }),
        model_ids: draft.models,
        // Exactly one of these two shapes, matching retryConfigMode:
        //   'all'        -> shared top-level max_retries/timeout
        //   'individual' -> model_retry_config, one entry per model, seeded
        //                   with defaults as soon as a model is selected
        //                   (see toggleModel) so it's always fully populated.
        // Same retryConfigPayload used for the agent-benchmark endpoints
        // above, so retry/timeout behaves identically across all launch
        // paths (Model/RAG create-eval, and both agent-benchmark calls).
        ...retryConfigPayload,
        selected_metrics: hideMetricsStep ? [] : draft.metrics,
        run_samples: createEvalRunSamples,
        selected_category: draft.subgroup.length > 0 ? draft.subgroup : dataset ? [dataset.category] : undefined,
        // Judge model is mandatory now (see canGo's step-5 check), so
        // draft.judgeModelId is always set by the time launch() runs — the
        // {} fallback is just defensive. max_retries/timeout mirror
        // whichever value applies per judgeRetryConfig above.
        judge_config: draft.judgeModelId
          ? {
              model_id: draft.judgeModelId,
              base_url: judgeModelObj?.base_url || '',
              api_key: draft.judgeModelId,
              max_retries: judgeRetryConfig.max_retries,
              timeout: judgeRetryConfig.timeout,
            }
          : {},
        // RAG-only — how many retrieved documents to consider per query.
        // Omitted entirely for Model/Agent rather than sent as undefined,
        // so it doesn't show up as a spurious key in the request body.
        ...(isRag ? { top_k: draft.topK } : {}),
        // Optional — omitted entirely (rather than sent as an empty
        // string) when the user hasn't typed or generated one.
        ...(draft.instruction.trim() ? { instruction: draft.instruction.trim() } : {}),
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
  const providerNames = draft.providers.map((id) => providers.find((p) => p?.id === id)?.name || id);
  const mf = (value: string, filled: boolean) => ({ value: filled ? value : '—', empty: !filled });
  const typeLabel = TYPE_OPTIONS.find((o) => o.value === draft.type)?.label ?? '';
  const frameworkTitle = draft.agentFramework ? AGENT_FRAMEWORKS.find((f) => f.id === draft.agentFramework)?.title : null;
  const manifest = [
    mf(draft.name, Boolean(draft.name)),
    mf(frameworkTitle ? `${typeLabel} · ${frameworkTitle}` : typeLabel, Boolean(draft.type)),
    mf(draft.providers.length === 1 ? providerNames[0] : `${draft.providers.length} providers`, draft.providers.length > 0),
    mf(`${draft.models.length} models`, draft.models.length > 0),
    mf(suite?.name || '', Boolean(suite)),
    mf(hideMetricsStep ? 'Not required' : `${draft.metrics.length} metrics`, hideMetricsStep || draft.metrics.length > 0),
    mf(
      judgeModel ? `Judge · ${judgeModel.name || 'Unnamed model'}` : 'Judge required',
      Boolean(judgeModel)
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
        <>
          <input
            type="number"
            min={0}
            max={maxRunSamples}
            className={styles.ev__input}
            style={{ marginTop: 10 }}
            value={draft.runSamples}
            onChange={(e) => {
              const raw = e.target.value === '' ? 0 : Number(e.target.value);
              const val = Number.isNaN(raw) ? 0 : Math.min(maxRunSamples, Math.max(0, raw));
              dispatch(setDraft({ runSamples: val }));
            }}
          />
          <p className={styles['ev__radio-full-note']}>Up to {maxRunSamples.toLocaleString()} questions available in this suite.</p>
        </>
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
                <StepErrorBoundary
                  key={`${step}-${stepRetryKey}`}
                  onRetry={() => setStepRetryKey((k) => k + 1)}
                  onBack={goBack}
                  canGoBack={step > 0}
                >
                <div className={styles.ev__anim} style={{ flex: 1, minHeight: 0, display: 'flex', flexDirection: 'column' }}>
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
                                  <span className={styles['ev__pcard-name']}>{p.name || 'Unnamed provider'}</span>
                                  <span className={styles['ev__pcard-meta']}>{p.model_count ?? 0} models available</span>
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

                        {/* Retry/timeout mode — lets the user apply one shared
                            max_retries/timeout to every selected model at once
                            ("Apply to all"), or set it per model on each card
                            below ("Individually"). Only relevant once at least
                            one model is selected. */}
                        {draft.models.length > 0 && (
                          <div className={styles['ev__retry-mode']}>
                            <div className={styles['ev__retry-mode-head']}>
                              <label className={styles.ev__label}>Retries &amp; timeout</label>
                              <div className={styles['ev__radio-row']}>
                                <button
                                  type="button"
                                  className={`${styles['ev__radio-opt']} ${
                                    draft.retryConfigMode === 'all' ? styles['ev__radio-opt--on'] : ''
                                  }`}
                                  onClick={() => dispatch(setDraft({ retryConfigMode: 'all' }))}
                                >
                                  <span
                                    className={`${styles.ev__radio} ${
                                      draft.retryConfigMode === 'all' ? styles['ev__radio--on'] : ''
                                    }`}
                                  />
                                  Apply to all
                                </button>
                                <button
                                  type="button"
                                  className={`${styles['ev__radio-opt']} ${
                                    draft.retryConfigMode === 'individual' ? styles['ev__radio-opt--on'] : ''
                                  }`}
                                  onClick={() => dispatch(setDraft({ retryConfigMode: 'individual' }))}
                                >
                                  <span
                                    className={`${styles.ev__radio} ${
                                      draft.retryConfigMode === 'individual' ? styles['ev__radio--on'] : ''
                                    }`}
                                  />
                                  Individually
                                </button>
                              </div>
                            </div>

                            {draft.retryConfigMode === 'all' ? (
                              <div className={styles['ev__retry-mode-all']}>
                                <div className={styles['ev__mcard-retry-field']}>
                                  <label>Max retries</label>
                                  <input
                                    type="number"
                                    min={MIN_MAX_RETRIES}
                                    max={MAX_MAX_RETRIES}
                                    value={draft.retryConfigAll.max_retries}
                                    onChange={(e) => updateRetryConfigAll('max_retries', Number(e.target.value))}
                                    onBlur={() => clampRetryConfigAll('max_retries')}
                                  />
                                </div>
                                <div className={styles['ev__mcard-retry-field']}>
                                  <label>Timeout (s)</label>
                                  <input
                                    type="number"
                                    min={MIN_TIMEOUT}
                                    max={MAX_TIMEOUT}
                                    value={draft.retryConfigAll.timeout}
                                    onChange={(e) => updateRetryConfigAll('timeout', Number(e.target.value))}
                                    onBlur={() => clampRetryConfigAll('timeout')}
                                  />
                                </div>
                                <p className={styles['ev__retry-mode-note']}>
                                  <Info size={11} /> Retries {MIN_MAX_RETRIES}–{MAX_MAX_RETRIES} · Timeout {MIN_TIMEOUT}
                                  –{MAX_TIMEOUT}s — applied to all {draft.models.length} selected model
                                  {draft.models.length === 1 ? '' : 's'}.
                                </p>
                              </div>
                            ) : (
                              <p className={styles['ev__retry-mode-note']}>
                                <Info size={11} /> Retries {MIN_MAX_RETRIES}–{MAX_MAX_RETRIES} · Timeout {MIN_TIMEOUT}
                                –{MAX_TIMEOUT}s — set on each selected model's card below.
                              </p>
                            )}
                          </div>
                        )}
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
                            const providerName = providers.find((p) => p?.id === m.provider_id)?.name ?? m.provider_id;

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
                                  <div className={styles['ev__mcard-name']}>{m.name || 'Unnamed model'}</div>
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
                                  {typeof accuracy === 'number' && Number.isFinite(accuracy) && (
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

                                {/* Per-model retry/timeout — only shown once the model is
                                    selected AND the user has chosen "Individually" mode
                                    (in "Apply to all" mode, the shared config above the
                                    grid covers every selected model). stopPropagation on
                                    the wrapper (and the inputs themselves) keeps clicks
                                    here from also toggling the card's selection via the
                                    outer onClick. */}
                                {on && draft.retryConfigMode === 'individual' && (
                                  <div
                                    className={styles['ev__mcard-retry']}
                                    onClick={(e) => e.stopPropagation()}
                                    onKeyDown={(e) => e.stopPropagation()}
                                  >
                                    <div className={styles['ev__mcard-retry-fields']}>
                                      <div className={styles['ev__mcard-retry-field']}>
                                        <label>Max retries</label>
                                        <input
                                          type="number"
                                          min={MIN_MAX_RETRIES}
                                          max={MAX_MAX_RETRIES}
                                          value={draft.modelRetryConfig[m.id]?.max_retries ?? DEFAULT_MAX_RETRIES}
                                          onChange={(e) =>
                                            updateModelRetryConfig(m.id, 'max_retries', Number(e.target.value))
                                          }
                                          onBlur={() => clampModelRetryConfig(m.id, 'max_retries')}
                                        />
                                      </div>
                                      <div className={styles['ev__mcard-retry-field']}>
                                        <label>Timeout (s)</label>
                                        <input
                                          type="number"
                                          min={MIN_TIMEOUT}
                                          max={MAX_TIMEOUT}
                                          value={draft.modelRetryConfig[m.id]?.timeout ?? DEFAULT_TIMEOUT}
                                          onChange={(e) => updateModelRetryConfig(m.id, 'timeout', Number(e.target.value))}
                                          onBlur={() => clampModelRetryConfig(m.id, 'timeout')}
                                        />
                                      </div>
                                    </div>
                                    <p className={styles['ev__mcard-retry-hint']}>
                                      <Info size={11} /> Retries {MIN_MAX_RETRIES}–{MAX_MAX_RETRIES} · Timeout {MIN_TIMEOUT}
                                      –{MAX_TIMEOUT}s
                                    </p>
                                  </div>
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
                      {/* Upload isn't offered for Agent benchmarks with no
                          framework selected (POST /agent-benchmark/run only
                          accepts existing datasets), or for RAG (fixed
                          retrieval corpora, no upload-your-own flow) — so
                          there's nothing to switch between and the tab bar
                          itself is hidden, not just the Upload button. */}
                      {!hideUploadTab && (
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
                      )}

                      {isRag && (
                        <div className={`${styles.ev__field} ${styles['ev__field--topk']}`}>
                          <label className={styles.ev__label}>
                            Top K <span className="opt">how many retrieved documents to consider per query</span>
                          </label>
                          <input
                            type="number"
                            min={1}
                            max={50}
                            className={`${styles.ev__input} ${styles['ev__input--topk']}`}
                            value={draft.topK}
                            onChange={(e) => {
                              const raw = e.target.value === '' ? 5 : Number(e.target.value);
                              const val = Number.isNaN(raw) ? 5 : Math.min(50, Math.max(1, Math.round(raw)));
                              dispatch(setDraft({ topK: val }));
                            }}
                          />
                        </div>
                      )}

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
                                    const isCustom = (d as any)?.dataset_type === 'custom';
                                    const isDeepeval = (d as any)?.dataset_type === 'deepeval';
                                    const hasNoQuestions = (d.question_count ?? 0) <= 0;
                                    return (
                                      // Not a <button> — it now contains a nested "Preview"
                                      // control, so it's a clickable div with keyboard
                                      // support instead (mirrors the model card pattern;
                                      // nested interactive elements aren't valid HTML).
                                      <div
                                        key={d.id}
                                        role="button"
                                        tabIndex={0}
                                        className={`${styles.ev__dcard} ${on ? styles['ev__dcard--on'] : ''} ${
                                          hasNoQuestions ? styles['ev__dcard--empty'] : ''
                                        }`}
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
                                            <span className={styles['ev__dcard-name']}>{d.name || 'Untitled dataset'}</span>
                                            {hasNoQuestions && (
                                              <span
                                                className={styles['ev__dcard-warn-icon']}
                                                title="This test suite has no questions yet — it can't be used until it does."
                                              >
                                                <Info size={13} />
                                              </span>
                                            )}
                                          </div>
                                          <div className={styles['ev__dcard-actions']}>
                                            <button
                                              type="button"
                                              className={styles['ev__dcard-preview-btn']}
                                              onClick={(e) => {
                                                e.stopPropagation();
                                                openPreview(d.id, d.name, d.question_count ?? 0);
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
                                          {d.category && <span className={styles.ev__tag}>{d.category}</span>}
                                          {d.eval_type && <span className={styles.ev__tag}>{d.eval_type}</span>}
                                          {/* Change-1: both dataset_type values now get a tag —
                                              previously only 'custom' rendered one. */}
                                          {isCustom && (
                                            <span className={`${styles.ev__tag} ${styles['ev__tag--custom']}`}>Custom</span>
                                          )}
                                          {isDeepeval && (
                                            <span className={`${styles.ev__tag} ${styles['ev__tag--deepeval']}`}>Deepeval</span>
                                          )}
                                          <span
                                            className={`${styles.ev__tag} ${styles['ev__tag--count']} ${
                                              hasNoQuestions ? styles['ev__tag--count-empty'] : ''
                                            }`}
                                          >
                                            {(d.question_count ?? 0).toLocaleString()} questions
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

                      {suite && (suite.question_count ?? 0) <= 0 && (
                        <div className={styles['ev__dataset-empty-warning']}>
                          <Info size={14} />
                          <span>
                            <strong>{suite.name || 'This test suite'}</strong> has no questions, so it can't be used for a
                            run. Pick a different test suite to continue.
                          </span>
                        </div>
                      )}

                      {datasetTab === 'upload' && !hideUploadTab && (
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
                    <div className={styles.ev__metrics}>
                      <div className={styles['ev__metrics-main']}>
                        <div className={styles['ev__samples-instruction-row']}>
                          <div className={styles.ev__samples}>
                            {runSamplesControl}
                            <p className={styles['ev__samples-note']}>
                              {hideMetricsStep
                                ? isAgentBenchmarkNoFramework
                                  ? "Metrics aren\u2019t configurable for agent benchmarks without a selected framework — a judge model is still required, though."
                                  : "Metrics aren\u2019t configurable for standard (non-custom) model benchmarks — a judge model is still required, though."
                                : 'Questions sampled from the suite for each model.'}
                            </p>
                          </div>

                          {/* Instruction — optional, independent of the metrics/judge
                              gating below. Always available since it's just extra
                              context for the run, not tied to any specific metric.
                              Moved up next to Run Samples so both sit in one 2-column
                              row at the top of the Metrics step. */}
                          <div className={`${styles['ev__instruction']} ${styles['ev__instruction--col']}`}>
                            <div className={styles['ev__instruction-head']}>
                              <label className={styles.ev__label}>
                                Instruction (Optional)
                              </label>
                              <button
                                type="button"
                                className={styles['ev__instruction-generate-btn']}
                                onClick={generateInstruction}
                                disabled={instructionGenerating || !draft.dataset || draft.models.length === 0}
                                title="Generate a default instruction from a sample of this dataset"
                              >
                                {instructionGenerating ? (
                                  <>
                                    <Loader2 size={13} className={styles.ev__spin} /> Generating…
                                  </>
                                ) : (
                                  <>
                                    <Sparkles size={13} /> Generate Instruction
                                  </>
                                )}
                              </button>
                            </div>
                            <textarea
                              className={styles['ev__instruction-textarea']}
                              placeholder="Describe how the model(s) should approach this evaluation — or click Generate Instruction for a starting point you can edit."
                              value={draft.instruction}
                              onChange={(e) => dispatch(setDraft({ instruction: e.target.value }))}
                              rows={5}
                            />
                            {instructionError && <p className={styles.ev__error}>{instructionError}</p>}
                          </div>
                        </div>

                        {!hideMetricsStep && (
                          <>
                            {(metricsCatalog.length > 0 || customMetricsForType.length > 0) && draft.metrics.length === 0 && (
                              <p className={styles['ev__metrics-required']}>
                                Select at least one metric (built-in or custom) to continue.
                              </p>
                            )}

                            <div className={styles['ev__metrics-bar']}>
                              <span className={styles['ev__metrics-count']}>
                                <b>{draft.metrics.filter((m) => metricsCatalog.includes(m)).length}</b> of {metricsCatalog.length}{' '}
                                selected
                              </span>
                              <div className={styles['ev__metrics-actions']}>
                                <button type="button" className={styles.ev__link} onClick={selectAllBuiltinMetrics}>
                                  Select all
                                </button>
                                <button type="button" className={styles.ev__link} onClick={clearBuiltinMetrics}>
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

                            {/* Custom metrics — same layout as built-in, separate section since
                                they carry richer metadata (id/description/required_judge) and
                                store the metric's `id` in draft.metrics rather than its name.
                                Shown whenever the raw catalog has *any* entries — even if none
                                match this eval type or the current search — so an empty result
                                is visibly "0 for this type" rather than indistinguishable from
                                the section never having loaded at all. */}
                            {customMetricsCatalog.length > 0 && (
                              <div className={styles['ev__custom-metrics']}>
                                <div className={styles['ev__metrics-bar']}>
                                  <span className={styles['ev__metrics-count']}>
                                    <b>{draft.metrics.filter((m) => customMetricIdSet.has(m)).length}</b> of{' '}
                                    {customMetricsForType.length} custom selected
                                  </span>
                                  <div className={styles['ev__metrics-actions']}>
                                    <button type="button" className={styles.ev__link} onClick={selectAllCustomMetrics}>
                                      Select all
                                    </button>
                                    <button type="button" className={styles.ev__link} onClick={clearCustomMetrics}>
                                      Clear
                                    </button>
                                  </div>
                                </div>

                                <div className={styles.ev__chips}>
                                  {filteredCustomMetrics.map((c) => {
                                    const on = draft.metrics.includes(c.id);
                                    return (
                                      <button
                                        key={c.id}
                                        type="button"
                                        className={`${styles.ev__chip} ${styles['ev__chip--custom']} ${on ? styles['ev__chip--on'] : ''}`}
                                        onClick={() => dispatch(setDraft({ metrics: toggle(draft.metrics, c.id) }))}
                                        title={c.description || c.name}
                                      >
                                        {on && (
                                          <span className={styles['ev__chip-tick']}>
                                            <Check size={12} strokeWidth={3} />
                                          </span>
                                        )}
                                        {c.name}
                                        {c.required_judge && <Gavel size={11} className={styles['ev__chip-judge-icon']} />}
                                      </button>
                                    );
                                  })}
                                  {customMetricsForType.length === 0 && (
                                    <p className={styles.ev__empty}>No custom metrics available for this type.</p>
                                  )}
                                  {customMetricsForType.length > 0 && filteredCustomMetrics.length === 0 && (
                                    <p className={styles.ev__empty}>No custom metrics match "{metricSearch}".</p>
                                  )}
                                </div>
                              </div>
                            )}
                          </>
                        )}
                      </div>

                      {/* Judge model — mandatory for every type now (Model, Agent, RAG),
                          regardless of which metrics are selected or whether the metrics
                          section itself is shown. */}
                      <aside className={styles.ev__judge}>
                        <div className={styles['ev__judge-head']}>
                          <p className={styles['ev__judge-title']}>
                            <Gavel size={13} /> Judge model
                          </p>
                          <p className={styles['ev__judge-sub']}>Required to launch, for every evaluation type.</p>
                        </div>

                        <div className={styles['ev__judge-info']}>
                          <Info size={13} />
                          <span>
                            The judge model only actually grades metrics that need one (e.g. LLM-graded metrics) — for
                            everything else it's simply ignored, but a selection is still required to launch.
                          </span>
                        </div>

                        <div className={styles['ev__judge-scroll']}>
                          {judgeCandidateModels.length === 0 ? (
                            <div className={styles['ev__judge-empty']}>No available models yet.</div>
                          ) : (
                            judgeCandidateModels.map((m) => {
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
                                      <span className={styles['ev__judge-name']}>{m.name || 'Unnamed model'}</span>
                                      <span className={styles['ev__judge-meta']}>
                                        {providers.find((p) => p?.id === m.provider_id)?.name ?? m.provider_id}
                                      </span>
                                    </span>
                                  </button>
                                );
                              })
                          )}
                        </div>
                        {!draft.judgeModelId && (
                          <p className={styles['ev__judge-required']}>
                            Select a judge model to continue — it's required for every evaluation.
                          </p>
                        )}
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
                            {suite ? (suite.question_count ?? 0).toLocaleString() : '—'}
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
                          <div className={styles['ev__summary-v']}>{hideMetricsStep ? '—' : draft.metrics.length}</div>
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
                            <span>{draft.providers.map((id) => providers.find((p) => p?.id === id)?.name || id).join(', ') || '—'}</span>
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
                                  <span className={styles['ev__review-card-name']}>{m!.name || 'Unnamed model'}</span>
                                  <span className={styles['ev__review-card-sub']}>
                                    {providers.find((p) => p?.id === m!.provider_id)?.name || m!.provider_id}
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
                          {isRag && (
                            <div className={styles.ev__row}>
                              <span>Top K</span>
                              <span>{draft.topK}</span>
                            </div>
                          )}
                        </div>
                      </div>

                      {!hideMetricsStep && (
                        <div className={styles.ev__block}>
                          <p className={styles['ev__block-title']}>
                            <Target size={11} /> Metrics <b>({draft.metrics.length})</b>
                          </p>
                          {draft.metrics.length > 0 ? (
                            <div className={styles['ev__metric-tags']}>
                              {draft.metrics.map((m) => {
                                // draft.metrics mixes built-in metric names with custom
                                // metric ids — resolve the id back to its display name.
                                const custom = customMetricsCatalog.find((c) => c.id === m);
                                return (
                                  <span key={m} className={styles['ev__metric-tag']}>
                                    {custom ? custom.name : m}
                                  </span>
                                );
                              })}
                            </div>
                          ) : (
                            <p className={styles.ev__empty}>No metrics selected.</p>
                          )}
                        </div>
                      )}

                      <div className={styles.ev__block}>
                        <p className={styles['ev__block-title']}>
                          <Gavel size={11} /> Judge model
                        </p>
                        <div className={styles.ev__rows}>
                          <div className={styles.ev__row}>
                            <span>Model</span>
                            <span>{judgeModel ? judgeModel.name || 'Unnamed model' : '—'}</span>
                          </div>
                        </div>
                      </div>

                      {launchError && <p className={styles.ev__error}>{launchError}</p>}
                    </>
                  )}
                </div>
                </StepErrorBoundary>
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
                  Page {previewCurrentPage} of {previewTotalPages}
                  <span className={styles['ev-preview-limit-range']}>
                    {' '}
                    ({PREVIEW_PAGE_SIZE} per page
                    {previewQuestionCount > 0 ? `, ${previewQuestionCount.toLocaleString()} total` : ''})
                  </span>
                </label>
                <div className={styles['ev-preview-limit-controls']}>
                  <button
                    type="button"
                    className={styles['ev-preview-stepper-btn']}
                    onClick={goToPreviewPrevPage}
                    disabled={previewLoading || !previewHasPrevPage}
                    aria-label="Previous page"
                    title="Previous page"
                  >
                    <ChevronLeft size={14} />
                  </button>
                  <button
                    type="button"
                    className={styles['ev-preview-stepper-btn']}
                    onClick={goToPreviewNextPage}
                    disabled={previewLoading || !previewHasNextPage}
                    aria-label="Next page"
                    title="Next page"
                  >
                    <ChevronRight size={14} />
                  </button>
                </div>
              </div>
              <button
                type="button"
                className={`${styles.ev__btn} ${styles['ev__btn--primary']}`}
                onClick={() => previewDatasetId && fetchPreviewPage(previewDatasetId, previewOffset)}
                disabled={previewLoading}
                title="Reload this page"
              >
                {previewLoading ? (
                  <>
                    <Loader2 size={15} className={styles.ev__spin} /> Loading…
                  </>
                ) : (
                  <>
                    <RefreshCw size={15} /> Reload
                  </>
                )}
              </button>
            </div>

            <div className={styles['ev-preview-body']}>
              {previewError && <p className={styles.ev__error}>{previewError}</p>}

              {previewLoading && (
                <div className={styles['ev-preview-skel-list']} aria-busy="true" aria-label="Loading preview">
                  {Array.from({ length: 5 }).map((_, i) => (
                    <div key={i} className={styles['ev-preview-skel-card']}>
                      <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--line']}`} style={{ width: '85%' }} />
                      <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--line']}`} style={{ width: '60%' }} />
                      <span className={`${styles['ev__skel-block']} ${styles['ev__skel-block--line']}`} style={{ width: '40%' }} />
                    </div>
                  ))}
                </div>
              )}

              {!previewLoading &&
                !previewError &&
                previewData &&
                (() => {
                  // Guard against `questions` being missing/non-array on the
                  // response — accessing .length/.map directly on that would
                  // throw if the API ever omits or nulls the field.
                  const previewQuestions = Array.isArray(previewData.questions) ? previewData.questions : [];

                  if (previewQuestions.length === 0) {
                    return <p className={styles.ev__empty}>This suite returned no sample questions.</p>;
                  }

                  return (
                    <div className={styles['ev-preview-list']}>
                      {previewQuestions.map((q, i) => {
                        // Two response shapes are both seen in practice —
                        // resolve whichever fields are actually present
                        // rather than assuming one fixed shape (see
                        // DatasetPreviewQuestion in types).
                        const promptText = q?.input?.prompt ?? q?.input?.question;
                        const inputSource = q?.input?.source;
                        const inputType = q?.input?.type;
                        const expectedAnswer = q?.expected?.answer;
                        const expectedDocId = q?.expected?.doc_id;
                        const expectedSectionId = q?.expected?.section_id;
                        const metadataEntries =
                          q?.metadata && typeof q.metadata === 'object' ? Object.entries(q.metadata) : [];

                        return (
                          <div key={q?.id ?? i} className={styles['ev-preview-q']}>
                            <div className={styles['ev-preview-q-head']}>
                              <span className={styles['ev-preview-q-index']}>Q{previewOffset + i + 1}</span>
                              {q?.category && <span className={styles['ev-preview-q-cat']}>{q.category}</span>}
                              {q?.subgroup && <span className={styles['ev-preview-q-cat']}>{q.subgroup}</span>}
                              {inputType && <span className={styles['ev-preview-q-cat']}>{String(inputType)}</span>}
                            </div>

                            {promptText != null && (
                              <p className={styles['ev-preview-q-prompt']}>{String(promptText)}</p>
                            )}
                            {inputSource != null && (
                              <p className={styles['ev-preview-q-source']}>
                                <span>Source</span> {String(inputSource)}
                              </p>
                            )}

                            {Array.isArray(q?.choices) && q.choices.length > 0 && (
                              <ul className={styles['ev-preview-q-choices']}>
                                {q.choices.map((c, ci) => (
                                  <li key={ci}>{String(c)}</li>
                                ))}
                              </ul>
                            )}

                            {expectedAnswer !== undefined && expectedAnswer !== null && (
                              <p className={styles['ev-preview-q-answer']}>
                                <span>Expected</span> {String(expectedAnswer)}
                              </p>
                            )}
                            {(expectedDocId != null || expectedSectionId != null) && (
                              <p className={styles['ev-preview-q-answer']}>
                                <span>Reference</span>{' '}
                                {[expectedDocId != null ? `doc ${expectedDocId}` : null, expectedSectionId != null ? `section ${expectedSectionId}` : null]
                                  .filter(Boolean)
                                  .join(' · ')}
                              </p>
                            )}

                            {metadataEntries.length > 0 && (
                              <div className={styles['ev-preview-q-meta']}>
                                {metadataEntries.map(([key, value]) => (
                                  <span key={key} className={styles['ev-preview-q-meta-item']}>
                                    <b>{key}</b>
                                    {String(value)}
                                  </span>
                                ))}
                              </div>
                            )}
                          </div>
                        );
                      })}
                    </div>
                  );
                })()}
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
