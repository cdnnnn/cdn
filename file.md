import { Component, useEffect, useMemo, useRef, useState, type ErrorInfo, type ReactNode } from 'react';
import { useNavigate } from 'react-router-dom';
import {
  ChevronLeft,
  ChevronRight,
  Check,
  Minus,
  Timer,
  RotateCw,
  Cpu,
  Bot,
  Database,
  Tag,
  LayoutGrid,
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
  X,
  AlertTriangle,
  Info,
  Image as ImageIcon,
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
  runAgentHarness,
  setDraft,
  setDraftType,
  setLaunching,
  setLaunchError,
} from '../../store/slices/evaluationsSlice';
// Document-agent harness (Workstream 2) — agent catalog + use-case list.
// The suites/metrics/launch calls themselves go through evaluationsApi
// (mocked behind USE_HARNESS_MOCK, see api/mock/harnessMock.ts).
import { HARNESS_AGENTS, HARNESS_USE_CASES } from '../../api/mock/harnessMock';
import type {
  CreateEvaluationRequest,
  EvaluationDraft,
  DatasetPreviewResponse,
  CustomMetric,
  ModelRetryConfig,
  Model,
  Dataset,
  HarnessAdapterId,
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
//   - fetchMetrics({ evalType, useCodeExecution }) — normally GET /metrics?
//     eval_type={type}, dispatched the first time Step 5 is opened for a
//     given draft.type/dataset combination. When eval_type is 'model' and
//     the selected Test Suite dataset's name is 'Livecodebench_code_generation_lite'
//     or 'MdEval' (see CODE_EXECUTION_DATASET_NAMES), useCodeExecution is
//     true and it calls GET /code-execution-metrics instead — same
//     response shape.
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
  {
    title: 'Choose models',
    sub: 'Tick a model to add it. Health is checked automatically, and only models that pass can join the run.',
  },
  { title: 'Pick a test suite', sub: 'Select a dataset to evaluate against, or upload your own.' },
  { title: 'Set up scoring', sub: 'Work top to bottom. Each part shows what is still missing.' },
  { title: 'Review and launch', sub: 'This is exactly what will run. Use Edit to change any part.' },
];

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
const MAX_MAX_RETRIES = 30;
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

// Document-agent harness settings (Metrics step) — bounds for the two
// number fields. task_timeout reuses the same 60–900s range as the
// per-model timeout, so clampRetryField('timeout', …) is used for it.
const MIN_HARNESS_STEPS = 1;
const MAX_HARNESS_STEPS = 100;
const clampHarnessSteps = (value: number): number =>
  Math.min(MAX_HARNESS_STEPS, Math.max(MIN_HARNESS_STEPS, Number.isNaN(value) ? MIN_HARNESS_STEPS : value));

// Model-only Metrics-step "Retest on Wrong" — bounds/default for Retest
// Max Rounds.
const MIN_RETEST_ROUNDS = 1;
const MAX_RETEST_ROUNDS = 30;
const DEFAULT_RETEST_ROUNDS = 3;
const clampRetestRounds = (value: number): number =>
  Math.min(MAX_RETEST_ROUNDS, Math.max(MIN_RETEST_ROUNDS, Number.isNaN(value) ? MIN_RETEST_ROUNDS : value));

// Model-type datasets whose `name` (GET /datasets?eval_type=model) selects
// GET /code-execution-metrics instead of the normal GET /metrics?eval_type=
// model for the Metrics step — same response shape, different metric set.
const CODE_EXECUTION_DATASET_NAMES = new Set(['Livecodebench_code_generation_lite', 'MdEval']);

// Preview is normally hidden for agent_benchmark (agent type, no framework
// selected — see isAgentBenchmarkNoFramework), since that endpoint (POST
// /agent-benchmark/run) doesn't use the dataset preview/question-browsing
// flow at all. These three datasets are the exception — Preview is shown
// for them even under agent_benchmark.
const AGENT_BENCHMARK_PREVIEWABLE_DATASETS = new Set(['ToolBench', 'BFCLv3', 'GAIA']);

// A preview question's `metadata` is usually flat (e.g. { level: "1",
// source: "..." }), but some datasets nest an object inside it (e.g.
// { level: "1", annotator: { steps: "...", number_of_steps: "4", tools:
// "..." } }). Rendered naively with String(value), a nested object would
// show up as the useless literal "[object Object]" — this flattens any
// such nesting into dot-notation key/value pairs (annotator.steps,
// annotator.number_of_steps, ...) so every field is still visible as its
// own chip. Arrays are joined into a single readable string rather than
// flattened by index.
function flattenMetadata(obj: Record<string, unknown>, prefix = ''): [string, string][] {
  const entries: [string, string][] = [];
  for (const [key, value] of Object.entries(obj)) {
    if (value === null || value === undefined) continue;
    const label = prefix ? `${prefix}.${key}` : key;
    if (Array.isArray(value)) {
      entries.push([
        label,
        value.map((v) => (v !== null && typeof v === 'object' ? JSON.stringify(v) : String(v))).join(', '),
      ]);
    } else if (typeof value === 'object') {
      entries.push(...flattenMetadata(value as Record<string, unknown>, label));
    } else {
      entries.push([label, String(value)]);
    }
  }
  return entries;
}

// Number inputs for retries/timeout/retest rounds: an emptied field is
// held as NaN in the draft (so backspacing to clear it shows an empty box
// instead of snapping to 0), and clamped back into range on blur and at
// launch time (clampRetryField / clampRetestRounds both map NaN -> min).
const parseNumInput = (s: string): number => (s === '' ? NaN : Number(s));
const displayNumInput = (v: number | undefined): number | string =>
  v === undefined || Number.isNaN(v) ? '' : v;

const SUGGESTED_NAMES = [
  'Q3 Model Selection',
  'Support Bot Regression',
  'RAG Accuracy v2',
  'GLM-4.6 vs Claude',
];

// Type step: what each system under test is about — shown in the "About"
// panel under the three cards for whichever one is selected.
const TYPE_ABOUT: Record<string, { tagline: string; measures: string[]; goodFor: string[]; flow: string[] }> = {
  model: {
    tagline: 'Models answer the same questions directly, with no tools and no retrieval.',
    measures: [
      'Answer quality on every question, per model',
      'Pass and fail counts for each metric you pick',
      'A side-by-side ranking with a top model',
    ],
    goodFor: [
      'Choosing between models or providers',
      'Checking a new model version against the current one',
      'Benchmarking on standard suites or your own dataset',
    ],
    flow: [
      'Every selected model answers the same set of questions.',
      'Answers are scored by your metrics, with a judge model.',
      'Models are ranked side by side in History.',
    ],
  },
  agent: {
    tagline: 'An agent plans, calls tools and works through a task over several steps.',
    measures: [
      'Whether multi-step tasks are actually completed',
      'Tool use: the right tool with the right arguments',
      'Steps, cost and time per task in the document-agent harness',
    ],
    goodFor: [
      'Testing an agent before rolling it out',
      'Comparing agents on the same tasks, such as Hermes and Co-Work',
      'Comparing the models behind an agent',
    ],
    flow: [
      'The agent receives a task and plans its approach.',
      'It calls tools and works through the steps.',
      'The outcome is scored on task completion.',
    ],
  },
  rag: {
    tagline: 'A retrieval pipeline finds documents first, then an LLM answers from them.',
    measures: [
      'Whether answers stay faithful to the retrieved context',
      'How well retrieval surfaces the right documents',
      'The combined quality of embedding, reranker and LLM',
    ],
    goodFor: [
      'Comparing LLMs on the same retrieval setup',
      'Trying different Top K values',
      'Checking grounding for a knowledge assistant',
    ],
    flow: [
      'The embedding model and reranker fetch the Top K documents for each question.',
      'Each LLM answers using that retrieved context.',
      'Answers are scored for accuracy and grounding.',
    ],
  },
};

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
  // Models table: view-only provider filter (All / one provider). Doesn't
  // change draft.providers — that's still set on the Providers step.
  const [modelProviderFilter, setModelProviderFilter] = useState<string>('all');
  // RAG-only: one independent search box per category (Embedding/
  // Reranker/LLM) instead of the single global model search used by the
  // Model/Agent grid — see RAG_CATEGORIES below.
  const [ragCategorySearch, setRagCategorySearch] = useState<Record<string, string>>({
    Embedding: '',
    Reranker: '',
    LLM: '',
  });
  const [judgeSearch, setJudgeSearch] = useState('');
  const [datasetSearch, setDatasetSearch] = useState('');
  const [metricSearch, setMetricSearch] = useState('');

  const [providersLoading, setProvidersLoading] = useState(false);
  const [modelsLoading, setModelsLoading] = useState(false);
  // Which models have had a health check *complete during this visit* to
  // the page, and whether the model list itself has been (re)fetched this
  // visit. Redux keeps healthById/models/draft.models between visits, so a
  // model can come back already marked 'success' from last time — these
  // make sure Step 3 can't be continued on that stale result while this
  // visit's re-check/refetch is still in flight (see canGo / selectedModels
  // NotReady).
  const [verifiedModelIds, setVerifiedModelIds] = useState<Record<string, true>>({});
  const [modelsFetchedThisVisit, setModelsFetchedThisVisit] = useState(false);
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

  // ---- Test Suite (Document-agent harness mode) --------------------------
  // The harness suite pool is fetched straight from evaluationsApi
  // .getHarnessSuites() (mocked for now) into local state, rather than
  // through the datasets slice — keeps the existing dataset thunks/reducers
  // completely untouched. `useCaseFilter` replaces the All/Custom/Deepeval
  // chips in that mode.
  const [harnessSuites, setHarnessSuites] = useState<Dataset[]>([]);
  const [useCaseFilter, setUseCaseFilter] = useState<string>('all');

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

  // ---- Test Suite: image lightbox (MinIO-backed question images) --------
  // A question's input.images[] holds MinIO object keys, not usable
  // directly as <img src> — each one is resolved to a blob URL on demand
  // via evaluationsApi.getImageBlobUrl (POST /datasets/image) when its
  // thumbnail button is clicked, mirroring MinIO's own object-preview UI.
  const [imagePreviewOpen, setImagePreviewOpen] = useState(false);
  const [imagePreviewPaths, setImagePreviewPaths] = useState<string[]>([]);
  const [imagePreviewIndex, setImagePreviewIndex] = useState(0);
  const [imagePreviewUrl, setImagePreviewUrl] = useState<string | null>(null);
  const [imagePreviewLoading, setImagePreviewLoading] = useState(false);
  const [imagePreviewError, setImagePreviewError] = useState<string | null>(null);

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

  // Document-agent harness: Agent type + the "Document-agent harness" mode
  // switch on the Type step. Mutually exclusive with agentFramework. Every
  // harness-specific branch below is gated on this, so Model / RAG / Agent
  // (benchmark) / Agent (Hermes) behave exactly as before.
  const isHarness = draft.type === 'agent' && draft.agentHarness;

  const storeDatasets = useAppSelector((s) => s.datasets.items) ?? [];
  // In harness mode the Test Suite pool comes from the harness catalog
  // (local state) instead of the datasets slice.
  const datasets: Dataset[] = isHarness ? harnessSuites : (storeDatasets as Dataset[]);
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
      } else {
        setModelsFetchedThisVisit(true);
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
      if (draft.agentHarness) return 'agent_harness';
      return draft.agentFramework ? 'agent_custom' : 'agent_benchmark';
    }
    return draft.type;
  }, [draft.type, draft.agentFramework, draft.agentHarness]);

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

      // Document-agent harness: suites come from evaluationsApi
      // .getHarnessSuites() (mocked), not from the datasets slice.
      if (datasetType === 'agent_harness') {
        try {
          const items = await evaluationsApi.getHarnessSuites();
          setHarnessSuites(items);
          setDatasetsErrorLocal(null);
        } catch (err) {
          datasetsFetchedForTypeRef.current = null;
          setDatasetsErrorLocal((err as Error)?.message || 'Failed to load test suites');
        } finally {
          setDatasetsRefreshing(false);
        }
        return;
      }

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
    dispatch(setDraft({ dataset: null, harnessSuiteIds: [] }));
    setDatasetSearch('');
    setDatasetTypeFilter('all');
    setUseCaseFilter('all');
    setDatasetsErrorLocal(null);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [datasetType]);

  const suite = datasets.find((d) => d?.id === draft.dataset);

  // Model type + one of the two code-execution built-in datasets selected
  // in the Test Suite step -> Metrics step calls GET /code-execution-
  // metrics instead of the normal GET /metrics?eval_type=model.
  const isCodeExecutionDataset =
    draft.type === 'model' && Boolean(suite?.name) && CODE_EXECUTION_DATASET_NAMES.has(suite!.name);

  // GET /metrics?eval_type={type} (or GET /code-execution-metrics — see
  // isCodeExecutionDataset above) — fetched the first time Step 5 is
  // reached for a given draft.type/dataset combination. Only `all_metrics`
  // is consumed (see metricsCatalog below) — `metrics` is ignored
  // entirely. Same retry-on-failure rollback as datasets above.
  //
  // The cache key includes isCodeExecutionDataset (not just draft.type) —
  // switching between a code-execution dataset and a regular one without
  // changing draft.type would otherwise be treated as "already fetched"
  // and silently keep showing the wrong metric set.
  //
  // Harness mode fetches its own scorer list ('agent_harness' — metricsSlice
  // routes that to the harness endpoint, mocked for now), so it's part of
  // the key too.
  const metricsEvalType = isHarness ? 'agent_harness' : draft.type;
  const metricsCacheKey = `${metricsEvalType ?? ''}:${isCodeExecutionDataset}`;
  useEffect(() => {
    if (step !== 5 || !draft.type || !metricsEvalType) return;
    if (metricsFetchedForTypeRef.current === metricsCacheKey) return;
    metricsFetchedForTypeRef.current = metricsCacheKey;
    (async () => {
      setMetricsRefreshing(true);
      const result = await dispatch(fetchMetrics({ evalType: metricsEvalType, useCodeExecution: isCodeExecutionDataset }));
      setMetricsRefreshing(false);
      if (fetchMetrics.rejected.match(result)) {
        metricsFetchedForTypeRef.current = null;
      }
    })();
  }, [step, draft.type, metricsEvalType, metricsCacheKey, dispatch]);

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
  //
  // Harness mode: Run Samples applies per suite, so the cap is the largest
  // selected suite (a smaller suite simply runs in full). The suite key
  // below re-derives the default whenever the set of picked suites changes.
  const selectedHarnessSuites = useMemo(
    () => harnessSuites.filter((s) => draft.harnessSuiteIds.includes(s.id)),
    [harnessSuites, draft.harnessSuiteIds]
  );
  const selectedDatasetQuestionCount = isHarness
    ? Math.max(0, ...selectedHarnessSuites.map((s) => s.question_count ?? 0))
    : suite?.question_count ?? 0;
  const maxRunSamples = Math.max(1, selectedDatasetQuestionCount);
  const defaultRunSamples = selectedDatasetQuestionCount > 10 ? 10 : maxRunSamples;
  const harnessSuiteKey = draft.harnessSuiteIds.join(',');

  // Re-derive the default whenever the selected dataset itself changes
  // (not on every render) — matches the same trigger as the subgroup
  // auto-select effect above, and won't stomp on a value the user typed
  // in manually unless they've actually picked a different dataset since.
  useEffect(() => {
    if (isHarness ? draft.harnessSuiteIds.length === 0 : !draft.dataset) return;
    dispatch(setDraft({ runSamples: defaultRunSamples }));
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [draft.dataset, harnessSuiteKey]);

  const connectedProviders = providers.filter((p) => p?.status === 'connected');
  const filteredProviders = useMemo(
    () => connectedProviders.filter((p) => (p?.name ?? '').toLowerCase().includes(providerSearch.trim().toLowerCase())),
    [connectedProviders, providerSearch]
  );

  const availableModels = useMemo(
    () => models.filter((m) => draft.providers.includes(m?.provider_id)),
    [models, draft.providers]
  );
  // Model/Agent grid: LLM-category models only — Embedding and Reranker
  // models are only meaningful for RAG (which groups all three categories
  // into its own columns via modelsByCategory below, not through this).
  const filteredModels = useMemo(
    () =>
      availableModels
        .filter((m) => (m?.category ?? '').toLowerCase() === 'llm')
        .filter((m) => (m?.name ?? '').toLowerCase().includes(modelSearch.trim().toLowerCase())),
    [availableModels, modelSearch]
  );

  // RAG-only: exactly three model categories are possible — Embedding,
  // Reranker, LLM — always shown in that order (see STEP 3 — MODELS
  // below), regardless of what order the API returns models in.
  // modelsByCategory (full, unfiltered) drives category membership and
  // the per-category validation in canGo() below; filteredModelsByCategory
  // applies each column's own search box (ragCategorySearch) on top of
  // that — independent per column, unlike the Model/Agent grid's single
  // shared modelSearch.
  const RAG_CATEGORIES = ['Embedding', 'Reranker', 'LLM'] as const;
  const modelsByCategory = useMemo(() => {
    const map = new Map<string, Model[]>();
    RAG_CATEGORIES.forEach((cat) => map.set(cat, []));
    for (const m of availableModels) {
      if (m?.category && map.has(m.category)) map.get(m.category)!.push(m);
    }
    return map;
  }, [availableModels]);
  const filteredModelsByCategory = useMemo(() => {
    const map = new Map<string, Model[]>();
    RAG_CATEGORIES.forEach((cat) => {
      const term = (ragCategorySearch[cat] ?? '').trim().toLowerCase();
      map.set(cat, (modelsByCategory.get(cat) ?? []).filter((m) => (m?.name ?? '').toLowerCase().includes(term)));
    });
    return map;
  }, [modelsByCategory, ragCategorySearch]);

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
        // Harness mode filters by use case; every other mode by dataset_type.
        const matchesType = isHarness
          ? useCaseFilter === 'all' || d?.use_case === useCaseFilter
          : datasetTypeFilter === 'all' || (d as any)?.dataset_type === datasetTypeFilter;
        return matchesSearch && matchesType;
      }),
    [datasets, datasetSearch, datasetTypeFilter, useCaseFilter, isHarness]
  );

  // Harness suite selection: multi-select (harnessSuiteIds) instead of the
  // single draft.dataset used everywhere else.
  const isSuiteOn = (id: string) => (isHarness ? draft.harnessSuiteIds.includes(id) : draft.dataset === id);
  const pickSuite = (id: string) =>
    isHarness
      ? dispatch(setDraft({ harnessSuiteIds: toggle(draft.harnessSuiteIds, id) }))
      : dispatch(setDraft({ dataset: id }));

  // Use cases actually present in the fetched pool, in the spec's order.
  const harnessUseCases = useMemo(
    () => HARNESS_USE_CASES.filter((u) => harnessSuites.some((s) => s.use_case === u)),
    [harnessSuites]
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
    const result = await dispatch(fetchModels());
    setModelsLoading(false);
    if (!fetchModels.rejected.match(result)) setModelsFetchedThisVisit(true);
  };

  const refreshDatasets = async () => {
    if (!datasetType) return;
    setDatasetsRefreshing(true);

    if (datasetType === 'agent_harness') {
      try {
        setHarnessSuites(await evaluationsApi.getHarnessSuites());
        datasetsFetchedForTypeRef.current = datasetType;
        setDatasetsErrorLocal(null);
      } catch (err) {
        datasetsFetchedForTypeRef.current = null;
        setDatasetsErrorLocal((err as Error)?.message || 'Failed to load test suites');
      } finally {
        setDatasetsRefreshing(false);
      }
      return;
    }

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
    if (!draft.type || !metricsEvalType) return;
    setMetricsRefreshing(true);
    const result = await dispatch(fetchMetrics({ evalType: metricsEvalType, useCodeExecution: isCodeExecutionDataset }));
    setMetricsRefreshing(false);
    if (fetchMetrics.rejected.match(result)) {
      metricsFetchedForTypeRef.current = null;
    } else {
      metricsFetchedForTypeRef.current = metricsCacheKey;
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

  // ---- Test Suite: image lightbox (MinIO-backed question images) --------
  const openImagePreview = (paths: string[], index: number) => {
    setImagePreviewPaths(paths);
    setImagePreviewIndex(index);
    setImagePreviewOpen(true);
  };

  const closeImagePreview = () => {
    setImagePreviewOpen(false);
    setImagePreviewError(null);
  };

  const showPrevImage = () => setImagePreviewIndex((i) => Math.max(0, i - 1));
  const showNextImage = () =>
    setImagePreviewIndex((i) => Math.min(imagePreviewPaths.length - 1, i + 1));

  // Resolves the object key at the current index to a blob URL whenever
  // the lightbox opens or the user navigates between images. Cleans up
  // (revokes) the previous blob URL both between images and on close, so
  // an abandoned in-flight request never leaks — or overwrites — the URL
  // for whatever image is now showing.
  useEffect(() => {
    if (!imagePreviewOpen) {
      setImagePreviewUrl(null);
      return;
    }
    const path = imagePreviewPaths[imagePreviewIndex];
    if (!path) return;

    let cancelled = false;
    let objectUrl: string | null = null;
    setImagePreviewLoading(true);
    setImagePreviewError(null);
    setImagePreviewUrl(null);

    evaluationsApi
      .getImageBlobUrl(path)
      .then((url) => {
        if (cancelled) {
          URL.revokeObjectURL(url);
          return;
        }
        objectUrl = url;
        setImagePreviewUrl(url);
      })
      .catch((err) => {
        if (cancelled) return;
        const detail =
          (err as { response?: { data?: { detail?: string } } })?.response?.data?.detail ||
          (err as Error)?.message ||
          'Failed to load image';
        setImagePreviewError(detail);
      })
      .finally(() => {
        if (!cancelled) setImagePreviewLoading(false);
      });

    return () => {
      cancelled = true;
      if (objectUrl) URL.revokeObjectURL(objectUrl);
    };
  }, [imagePreviewOpen, imagePreviewIndex, imagePreviewPaths]);

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
  // Every health check goes through here so we know when one has actually
  // *completed this visit* (verifiedModelIds) — a stale 'success' left in
  // Redux from an earlier visit doesn't count until it's re-confirmed.
  const trackedHealthCheck = (modelId: string) => {
    setVerifiedModelIds((prev) => {
      if (!prev[modelId]) return prev;
      const next = { ...prev };
      delete next[modelId];
      return next;
    });
    Promise.resolve(dispatch(checkModelHealth(modelId))).then(() =>
      setVerifiedModelIds((prev) => ({ ...prev, [modelId]: true }))
    );
  };

  const runHealthCheck = (modelId: string) => {
    trackedHealthCheck(modelId);
  };

  // Auto health-check: as soon as the models list for the currently
  // selected providers is available (and while Step 3 is open), fire a
  // parallel health check for every visible model — the user isn't
  // expected to click "Check health" one by one. Guarded on the raw
  // `models` array reference so it re-runs once per fresh fetch (initial
  // navigation-triggered fetch, or a manual refresh) rather than on every
  // render or on unrelated state changes like typing in the search box.
  const autoHealthCheckedForModelsRef = useRef<typeof models | null>(null);
  const autoHealthCheckedIdsRef = useRef<Set<string>>(new Set());
  useEffect(() => {
    if (step !== 3 || availableModels.length === 0) return;
    if (autoHealthCheckedForModelsRef.current !== models) {
      autoHealthCheckedForModelsRef.current = models;
      autoHealthCheckedIdsRef.current = new Set();
    }
    availableModels.forEach((m) => {
      if (autoHealthCheckedIdsRef.current.has(m.id)) return;
      autoHealthCheckedIdsRef.current.add(m.id);
      trackedHealthCheck(m.id);
    });
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

  // Model-only "Retest on Wrong" — same free-typing-then-clamp-on-blur
  // pattern as the Models step's retry/timeout inputs, so clearing/
  // retyping the field isn't fought by immediate clamping.
  const updateRetestMaxRounds = (raw: number) => {
    dispatch(setDraft({ retestMaxRounds: raw }));
  };
  const clampRetestMaxRoundsOnBlur = () => {
    const clamped = clampRetestRounds(draft.retestMaxRounds);
    if (clamped !== draft.retestMaxRounds) dispatch(setDraft({ retestMaxRounds: clamped }));
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
  // Harness mode is its own path (POST /agent-harness/run) with full
  // metrics + judge, so it's excluded here.
  const isAgentBenchmarkNoFramework = draft.type === 'agent' && !draft.agentFramework && !draft.agentHarness;
  const hideMetricsStep = modelHidesMetrics || isAgentBenchmarkNoFramework;

  // RAG datasets are always pre-built retrieval corpora — there's no
  // "upload your own" flow for them (only Top K applies, further below).
  // Same for the harness: its suites are curated (the in-house set is
  // maintained by the team), so no upload there either.
  const isRag = draft.type === 'rag';
  const hideUploadTab = isAgentBenchmarkNoFramework || isRag || isHarness;

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
  const isAgentWithFramework = draft.type === 'agent' && Boolean(draft.agentFramework) && !draft.agentHarness;

  const isModelSelectable = (modelId: string) => healthById?.[modelId] === 'success';

  // Selected models (that are still on screen in Step 3) whose health isn't
  // confirmed 'success' right now — e.g. a selection carried over from an
  // earlier visit to this step whose check is re-running, never ran, or
  // failed. Step 3 can't be continued past until this is empty, so a
  // stale selection doesn't let someone proceed with an unverified model.
  const selectedModelsNotReady = draft.models.filter(
    (id) =>
      availableModels.some((m) => m.id === id) && !(healthById?.[id] === 'success' && verifiedModelIds[id])
  );
  const selectedModelsChecking = selectedModelsNotReady.some((id) => healthById?.[id] !== 'failed');

  // Judge model options — every model that's actually usable (health check
  // passed) AND belongs to a currently-selected provider, regardless of
  // whether the model itself was picked in Step 3 for evaluation (judge
  // selection is independent of that). Scoped to availableModels
  // (draft.providers-filtered) rather than the full model list — a model
  // whose provider gets unselected back in Step 2 needs to disappear from
  // here too, not linger as a stale option.
  //
  // Only LLM-category models can act as a judge — Embedding and Reranker
  // models (which RAG's Models step lists alongside LLMs) can't grade
  // anything, so they're excluded here for every eval type.
  const judgeCandidateModels = useMemo(
    () =>
      availableModels.filter(
        (m) => (m?.category ?? '').toLowerCase() === 'llm' && isModelSelectable(m?.id)
      ),
    [availableModels, healthById]
  );

  const filteredJudgeModels = useMemo(
    () => judgeCandidateModels.filter((m) => (m?.name ?? '').toLowerCase().includes(judgeSearch.trim().toLowerCase())),
    [judgeCandidateModels, judgeSearch]
  );

  // Auto-scrolls down to the launch error message on the Review & Launch
  // step whenever a launch attempt fails — the error renders at the very
  // bottom of that step's content, below the whole review summary, so
  // without this the person has to notice it needs scrolling themselves.
  const launchErrorRef = useRef<HTMLParagraphElement | null>(null);
  useEffect(() => {
    if (launchError && step === totalSteps - 1) {
      launchErrorRef.current?.scrollIntoView({ behavior: 'smooth', block: 'end' });
    }
  }, [launchError, step, totalSteps]);

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

  // RAG-only variant of toggleModel: models are grouped by category into
  // one column each (see modelsByCategory below), and exactly one model
  // per category is required — so selecting a model here also deselects
  // any other model already chosen from the SAME category (categoryIds),
  // while selections in every other category are left untouched. Clicking
  // the currently-selected model in a category deselects it, same as
  // toggleModel.
  const toggleModelForCategory = (modelId: string, categoryIds: string[]) => {
    const alreadySelected = draft.models.includes(modelId);
    if (!alreadySelected && !isModelSelectable(modelId)) return;

    const categoryIdSet = new Set(categoryIds);
    const nextModels = alreadySelected
      ? draft.models.filter((id) => id !== modelId)
      : [...draft.models.filter((id) => !categoryIdSet.has(id)), modelId];

    // Keep modelRetryConfig in sync with the new selection: drop entries
    // for anything this action removed from the category, seed a default
    // for the newly-picked model (if it doesn't already have one).
    const nextRetryConfig = { ...draft.modelRetryConfig };
    if (alreadySelected) {
      delete nextRetryConfig[modelId];
    } else {
      draft.models.filter((id) => categoryIdSet.has(id) && id !== modelId).forEach((id) => delete nextRetryConfig[id]);
      if (!nextRetryConfig[modelId]) nextRetryConfig[modelId] = { ...DEFAULT_MODEL_RETRY_CONFIG };
    }

    dispatch(setDraft({ models: nextModels, modelRetryConfig: nextRetryConfig }));
  };

  // Shared model-card renderer — used by both the normal grid (Model/Agent,
  // and RAG's own toggle behavior isn't baked in here) and the RAG
  // per-category columns below. `onToggle` is whichever selection rule
  // applies (toggleModel vs. toggleModelForCategory); everything else
  // about the card (health check, stats, retry/timeout panel) is
  // identical either way.
  // inlineRetry: show the per-model retry/timeout fields inside the card
  // (RAG columns). The Model/Agent view passes false — those fields live in
  // the "In this run" side panel instead.
  const renderModelCard = (m: Model, onToggle: () => void, inlineRetry = true) => {
    const on = draft.models.includes(m.id);
    const health: HealthStatus = healthById?.[m.id] ?? 'idle';
    const selectable = health === 'success';
    const caps = (m as any).capabilities as string[] | undefined;
    const inputPrice = (m as any).input_price as number | null | undefined;
    const outputPrice = (m as any).output_price as number | null | undefined;
    const accuracy = (m as any).accuracy_score as number | null | undefined;
    const providerName = providers.find((p) => p?.id === m.provider_id)?.name ?? m.provider_id;
    // Why this card can't be selected yet. Shown as a line inside the card in
    // the RAG columns; in the Model/Agent grid it's the card's tooltip
    // instead, so every card keeps the same content and the same size.
    const lockedHint =
      !selectable && !on
        ? health === 'loading'
          ? 'Waiting for health check to complete…'
          : health === 'failed'
            ? 'This model failed its health check and can\u2019t be selected.'
            : 'Run a health check to enable selection.'
        : undefined;

    // Health badge / action — shared by both card layouts below.
    const healthBadge = (
      <>
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
      </>
    );

    // Model / Agent grid: same card design as the Test Suite step — header
    // (selection control, icon tile, name), tags, a labelled chip section,
    // and a footer of figures. Retries and timeout live in the side panel.
    if (!inlineRetry) {
      return (
        <div
          key={m.id}
          role="button"
          tabIndex={0}
          className={`${styles.ev__srow} ${styles['ev__srow--model']} ${on ? styles['ev__srow--on'] : ''} ${
            !selectable && !on ? styles['ev__srow--locked'] : ''
          } ${health === 'loading' ? styles['ev__mcard--checking'] : ''}`}
          onClick={onToggle}
          onKeyDown={(e) => {
            if (e.key === 'Enter' || e.key === ' ') {
              e.preventDefault();
              onToggle();
            }
          }}
          aria-pressed={on}
          aria-disabled={!selectable && !on}
          title={lockedHint}
        >
          <div className={styles['ev__srow-head']}>
            <span className={`${styles.ev__check} ${on ? styles['ev__check--on'] : ''}`}>
              {on && <Check size={11} strokeWidth={3} />}
            </span>
            <span className={styles['ev__srow-icon']}>
              <Cpu size={15} />
            </span>
            <span className={styles['ev__srow-name']} title={m.name || 'Unnamed model'}>
              {m.name || 'Unnamed model'}
            </span>
          </div>

          <div className={styles['ev__srow-tags']}>
            <span className={`${styles.ev__pilltag} ${styles['ev__pilltag--muted']}`}>{providerName}</span>
            {m.category && <span className={`${styles.ev__pilltag} ${styles['ev__pilltag--muted']}`}>{m.category}</span>}
            <span className={styles['ev__srow-health']}>{healthBadge}</span>
          </div>

          <div className={styles['ev__srow-subs']}>
            <span className={styles['ev__srow-k']}>Capabilities</span>
            <div className={styles['ev__srow-chips']}>
              {caps && caps.length > 0 ? (
                <>
                  {caps.slice(0, 4).map((c) => (
                    <span key={c} className={styles.ev__cap}>
                      {c}
                    </span>
                  ))}
                  {caps.length > 4 && (
                    <span className={styles.ev__cap} title={caps.slice(4).join(', ')}>
                      +{caps.length - 4} more
                    </span>
                  )}
                </>
              ) : (
                <span className={styles['ev__srow-none']}>Not listed</span>
              )}
            </div>
          </div>

          <div className={styles['ev__srow-side']}>
            <div className={styles.ev__stat}>
              <span className={styles['ev__stat-k']}>Context</span>
              <span className={styles['ev__stat-v']}>{formatContextWindow(m.context_window)}</span>
            </div>
            <div className={styles.ev__stat}>
              <span className={styles['ev__stat-k']}>Input /1M</span>
              <span className={styles['ev__stat-v']}>{formatPrice(inputPrice)}</span>
            </div>
            <div className={styles.ev__stat}>
              <span className={styles['ev__stat-k']}>Output /1M</span>
              <span className={styles['ev__stat-v']}>{formatPrice(outputPrice)}</span>
            </div>
            {typeof accuracy === 'number' && Number.isFinite(accuracy) && (
              <div className={styles.ev__stat}>
                <span className={styles['ev__stat-k']}>Accuracy</span>
                <span className={styles['ev__stat-v']}>{accuracy.toFixed(1)}%</span>
              </div>
            )}
          </div>
        </div>
      );
    }

    return (
      // Not a <button> — it contains a nested "Check health" control, so
      // it's a clickable div with keyboard support instead (nested
      // interactive elements aren't valid HTML).
      <div
        key={m.id}
        role="button"
        tabIndex={0}
        className={`${styles.ev__mcard} ${on ? styles['ev__mcard--on'] : ''} ${
          !selectable && !on ? styles['ev__mcard--locked'] : ''
        } ${health === 'loading' ? styles['ev__mcard--checking'] : ''}`}
        onClick={onToggle}
        onKeyDown={(e) => {
          if (e.key === 'Enter' || e.key === ' ') {
            e.preventDefault();
            onToggle();
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
          {healthBadge}
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

        {inlineRetry && lockedHint && <p className={styles['ev__mcard-hint']}>{lockedHint}</p>}

        {/* Per-model retry/timeout — only rendered once the card is
            selected AND the user has chosen "Individually" mode (in
            "Apply to all" mode, the shared config above the grid covers
            every selected model, so no per-card panel is needed at all).
            Unselected cards stay compact since this only exists in the
            DOM for selected ones — &__grid's align-items: start keeps the
            taller selected card from stretching its unselected neighbors
            in the same row. stopPropagation on the wrapper (and the
            inputs themselves) keeps clicks here from also toggling the
            card's selection via the outer onClick. */}
        {inlineRetry && on && draft.retryConfigMode === 'individual' && (
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
                  value={displayNumInput(draft.modelRetryConfig[m.id]?.max_retries ?? DEFAULT_MAX_RETRIES)}
                  onChange={(e) => updateModelRetryConfig(m.id, 'max_retries', parseNumInput(e.target.value))}
                  onBlur={() => clampModelRetryConfig(m.id, 'max_retries')}
                />
              </div>
              <div className={styles['ev__mcard-retry-field']}>
                <label>Timeout (s)</label>
                <input
                  type="number"
                  min={MIN_TIMEOUT}
                  max={MAX_TIMEOUT}
                  value={displayNumInput(draft.modelRetryConfig[m.id]?.timeout ?? DEFAULT_TIMEOUT)}
                  onChange={(e) => updateModelRetryConfig(m.id, 'timeout', parseNumInput(e.target.value))}
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
    const safe = raw;
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
    const safe = raw;
    dispatch(setDraft({ retryConfigAll: { ...draft.retryConfigAll, [field]: safe } }));
  };

  const clampRetryConfigAll = (field: keyof ModelRetryConfig) => {
    const clamped = clampRetryField(field, draft.retryConfigAll[field]);
    if (clamped === draft.retryConfigAll[field]) return;
    dispatch(setDraft({ retryConfigAll: { ...draft.retryConfigAll, [field]: clamped } }));
  };

  // Why the current step can't be continued yet — null means good to go.
  // Same rules as the old canGo(), but each one carries its reason, shown
  // next to the (disabled) Next button.
  const stepBlocker = (): string | null => {
    if (step === 0) return draft.name.trim() ? null : 'Give the run a name.';
    if (step === 1) {
      if (!draft.type) return 'Choose a system under test.';
      if (isHarness && draft.harnessAgents.length === 0) return 'Select at least one agent.';
      return null;
    }
    if (step === 2) return draft.providers.length > 0 ? null : 'Select at least one provider.';
    if (step === 3) {
      if (modelsLoading || !modelsFetchedThisVisit) return 'Loading models…';
      if (draft.type === 'rag') {
        const countIn = (cat: string) =>
          (modelsByCategory.get(cat) ?? []).filter((m) => draft.models.includes(m.id)).length;
        if (countIn('Embedding') !== 1 || countIn('Reranker') !== 1 || countIn('LLM') === 0) {
          return 'Select one Embedding, one Reranker and at least one LLM.';
        }
      } else if (draft.models.length === 0) {
        return 'Tick at least one model.';
      }
      if (selectedModelsNotReady.length > 0) {
        const id = selectedModelsNotReady[0];
        const name = models.find((m) => m?.id === id)?.name || id;
        return healthById?.[id] === 'failed'
          ? `${name} failed its health check. Retry it or untick it.`
          : `Waiting for the health check on ${name}.`;
      }
      return null;
    }
    if (step === 4) {
      if (isHarness) {
        if (selectedHarnessSuites.length === 0) return 'Select at least one suite.';
        if (!selectedHarnessSuites.every((d) => (d.question_count ?? 0) > 0)) {
          return 'Untick the suites that have no questions.';
        }
        return null;
      }
      if (!draft.dataset) return 'Select a test suite.';
      if ((suite?.question_count ?? 0) <= 0) return 'This suite has no questions. Pick another.';
      return null;
    }
    if (step === 5) {
      const metricsAvailable = !hideMetricsStep && (metricsCatalog.length > 0 || customMetricsForType.length > 0);
      if (metricsAvailable && draft.metrics.length === 0) {
        return isHarness ? 'Select at least one scorer.' : 'Select at least one metric.';
      }
      if (!draft.judgeModelId) return 'Select a judge model.';
      const retestApplies =
        draft.type === 'model' || draft.type === 'rag' || (draft.type === 'agent' && Boolean(draft.agentFramework));
      if (retestApplies && draft.retestOnWrong && !draft.retestVerifyMetric) {
        return 'Pick a verify metric for Retest on wrong.';
      }
      if (isHarness && (Number.isNaN(draft.harnessSettings.max_steps) || Number.isNaN(draft.harnessSettings.task_timeout))) {
        return 'Fill in the limits per task.';
      }
      return null;
    }
    return null;
  };
  const blocker = stepBlocker();
  const canGo = () => blocker === null;

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

    // Multimodal pre-flight — only when the selected dataset carries
    // images (has_images, from GET /datasets?eval_type={type}). Checks
    // every selected model PLUS the judge model against POST /models/
    // check-multimodal before creating the evaluation at all: a non-2xx
    // response means at least one of them can't handle image input, so
    // the create/run call below is skipped entirely and the error is
    // surfaced the same way a launch failure normally is. Applies equally
    // to Model, Agent, and RAG — done once here, before branching into
    // whichever of the three endpoints this draft.type ends up calling.
    // Harness: any of the picked suites carrying images triggers the check.
    const selectedHasImages = isHarness
      ? selectedHarnessSuites.some((s) => s.has_images)
      : Boolean((dataset as any)?.has_images);
    if (selectedHasImages) {
      const multimodalModelIds = Array.from(
        new Set([...draft.models, ...(draft.judgeModelId ? [draft.judgeModelId] : [])])
      );
      dispatch(setLaunchError(null));
      dispatch(setLaunching(true));
      try {
        await evaluationsApi.checkMultimodal(multimodalModelIds);
      } catch (err) {
        const detail =
          (err as { response?: { data?: { detail?: string } } })?.response?.data?.detail ||
          (err as Error)?.message ||
          'One or more selected models cannot handle this dataset\u2019s images.';
        dispatch(setLaunchError(detail));
        dispatch(setLaunching(false));
        return;
      }
    }

    // Change: "Full" mode means "use the whole dataset" — for the agent
    // benchmark endpoints the backend contract for that is run_samples: 0.
    const effectiveRunSamples =
      draft.runSamplesMode === 'full' || Number.isNaN(draft.runSamples) ? 0 : draft.runSamples;
    // For POST /evaluations (Model/RAG) specifically, "Full" sends
    // run_samples as null — the backend's contract for "use the whole
    // dataset" on this endpoint, distinct from the agent-benchmark
    // endpoints above which use 0 for the same concept.
    const createEvalRunSamples: number | null =
      draft.runSamplesMode === 'full' ? null : Number.isNaN(draft.runSamples) ? 0 : draft.runSamples;

    let result: any;

    // Custom metrics (their `id`s) are a subset of whichever metrics list
    // applies — draft.metrics mixes built-in names and custom ids together
    // (see customMetricIdSet above), so this pulls out just the custom
    // ones to send as the additive selected_metric_ids field. Only added to
    // the payload when non-empty.
    const selectedMetricIdsFor = (effectiveMetrics: string[]): Pick<CreateEvaluationRequest, 'selected_metric_ids'> => {
      const ids = effectiveMetrics.filter((m) => customMetricIdSet.has(m));
      return ids.length > 0 ? { selected_metric_ids: ids } : {};
    };

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

    // Retest on Wrong — Model and RAG always; Agent only when a framework
    // is selected (POST /agent-benchmark/run has no metrics-related
    // payload at all, so this stays {} there). retest_on_wrong is always
    // sent once applicable; retest_max_rounds/retest_verify_metric only
    // when it's true. Computed once here and reused for both the
    // Model/RAG create-eval payload and the run-multi payload below, so
    // the two stay in sync.
    const retestApplies =
      draft.type === 'model' || draft.type === 'rag' || (draft.type === 'agent' && Boolean(draft.agentFramework));
    const retestPayload: Pick<CreateEvaluationRequest, 'retest_on_wrong' | 'retest_max_rounds' | 'retest_verify_metric'> =
      retestApplies
        ? {
            retest_on_wrong: draft.retestOnWrong,
            ...(draft.retestOnWrong
              ? {
                  retest_max_rounds: clampRetestRounds(draft.retestMaxRounds),
                  ...(draft.retestVerifyMetric ? { retest_verify_metric: draft.retestVerifyMetric } : {}),
                }
              : {}),
          }
        : {};

    if (isHarness) {
      // POST /agent-harness/run — Document-agent harness. Same retry shape
      // and judge config as the Model/RAG create-evaluation path; suites
      // each carry their own run_samples (same "0 = full" convention as the
      // agent-benchmark endpoints).
      const harnessJudgeRetry: ModelRetryConfig = clampModelRetryConfigValue(
        draft.retryConfigMode === 'all'
          ? draft.retryConfigAll
          : (draft.judgeModelId && draft.modelRetryConfig[draft.judgeModelId]) || DEFAULT_MODEL_RETRY_CONFIG
      );
      result = await dispatch(
        runAgentHarness({
          evaluation_name: draft.name,
          agents: draft.harnessAgents,
          model_ids: draft.models,
          suites: draft.harnessSuiteIds.map((id) => ({ dataset_id: id, run_samples: effectiveRunSamples })),
          selected_metrics: draft.metrics,
          ...selectedMetricIdsFor(draft.metrics),
          judge_config: {
            model_id: draft.judgeModelId || '',
            base_url: judgeModelObj?.base_url || '',
            api_key: draft.judgeModelId || '',
            max_retries: harnessJudgeRetry.max_retries,
            timeout: harnessJudgeRetry.timeout,
          },
          harness: {
            max_steps: clampHarnessSteps(draft.harnessSettings.max_steps),
            task_timeout: clampRetryField('timeout', draft.harnessSettings.task_timeout),
            save_traces: draft.harnessSettings.save_traces,
          },
          ...retryConfigPayload,
        })
      );
    } else if (draft.type === 'agent' && !draft.agentFramework) {
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
          ...selectedMetricIdsFor(draft.metrics),
          selected_categories: draft.subgroup,
          run_samples: effectiveRunSamples,
          ...retryConfigPayload,
          ...retestPayload,
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

      // RAG-only — split draft.models (which holds the Embedding, Reranker,
      // AND LLM selections all together, same as any other type) back out
      // by category for the payload: model_ids carries only the LLM
      // selection(s), Embedding/Reranker each go in their own dedicated
      // field instead. canGo's step-3 check already guarantees exactly one
      // Embedding and one Reranker are selected by the time launch() runs.
      const ragEmbeddingId = isRag
        ? draft.models.find((id) => (modelsByCategory.get('Embedding') ?? []).some((m) => m.id === id))
        : undefined;
      const ragRerankerId = isRag
        ? draft.models.find((id) => (modelsByCategory.get('Reranker') ?? []).some((m) => m.id === id))
        : undefined;
      const ragLlmIds = isRag
        ? draft.models.filter((id) => (modelsByCategory.get('LLM') ?? []).some((m) => m.id === id))
        : draft.models;

      const payload: CreateEvaluationRequest = {
        name: draft.name,
        eval_type: draft.type || '',
        dataset_id: dataset?.id || '',
        // RAG doesn't use `benchmark` — omitted entirely for that type
        // rather than sent as the dataset name (or null), matching how
        // top_k is likewise only ever included for RAG below.
        ...(isRag ? {} : { benchmark: dataset?.name || undefined }),
        model_ids: ragLlmIds,
        ...(isRag ? { embedding_model_id: ragEmbeddingId, reranker_model_id: ragRerankerId } : {}),
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
        ...selectedMetricIdsFor(hideMetricsStep ? [] : draft.metrics),
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
        // Model and RAG — Metrics step's "Retest on Wrong" control (see
        // retestApplies above). Same retestPayload used for the run-multi
        // call above, so Agent-with-framework gets it too.
        ...retestPayload,
      };
      result = await dispatch(launchEvaluation(payload));
    }

    const succeeded =
      launchEvaluation.fulfilled.match(result) ||
      runAgentBenchmark.fulfilled.match(result) ||
      runAgentBenchmarkMulti.fulfilled.match(result) ||
      runAgentHarness.fulfilled.match(result);

    if (succeeded) {
      setToast(true);
      setTimeout(() => {
        setToast(false);
        navigate('/app/history');
      }, 2000);
    }
  };


  // ---- live Run Manifest values (one per step) ----------------------------
  const providerNames = draft.providers.map((id) => providers.find((p) => p?.id === id)?.name || id);
  const mf = (value: string, filled: boolean) => ({ value: filled ? value : 'Not set', empty: !filled });
  const typeLabel = TYPE_OPTIONS.find((o) => o.value === draft.type)?.label ?? '';
  const frameworkTitle = draft.agentFramework ? AGENT_FRAMEWORKS.find((f) => f.id === draft.agentFramework)?.title : null;
  const harnessAgentTitles = draft.harnessAgents.map((id) => HARNESS_AGENTS.find((a) => a.id === id)?.title || id);
  // Run matrix (Review step + manifest): every agent runs every selected
  // model on every selected suite — that's the scorecard's grain.
  const harnessRunCount = draft.harnessAgents.length * draft.models.length * draft.harnessSuiteIds.length;
  const typeManifestValue = isHarness
    ? `${typeLabel} · Harness${harnessAgentTitles.length > 0 ? ` (${harnessAgentTitles.join(', ')})` : ''}`
    : frameworkTitle
      ? `${typeLabel} · ${frameworkTitle}`
      : typeLabel;
  const scoringManifestValue = [
    hideMetricsStep || draft.metrics.length === 0
      ? null
      : `${draft.metrics.length} ${isHarness ? 'scorer' : 'metric'}${draft.metrics.length === 1 ? '' : 's'}`,
    judgeModel ? `Judge: ${judgeModel.name || 'Unnamed model'}` : null,
  ]
    .filter(Boolean)
    .join(', ');
  const manifest = [
    mf(draft.name, Boolean(draft.name)),
    mf(typeManifestValue, Boolean(draft.type)),
    mf(providerNames.join(', '), draft.providers.length > 0),
    mf(`${draft.models.length} model${draft.models.length === 1 ? '' : 's'}`, draft.models.length > 0),
    isHarness
      ? mf(
          selectedHarnessSuites.length === 1 ? selectedHarnessSuites[0].name : `${selectedHarnessSuites.length} suites`,
          selectedHarnessSuites.length > 0
        )
      : mf(suite?.name || '', Boolean(suite)),
    mf(scoringManifestValue, scoringManifestValue !== ''),
    mf('Ready to launch', step === totalSteps - 1),
  ];

  const stageTitle = step === 4 && isHarness ? 'Pick test suites' : STAGE[step].title;
  const stageSub =
    step === 4 && isHarness
      ? 'Each suite covers one kind of work. Pick one per use case for a full scorecard.'
      : STAGE[step].sub;
  const scorerWord = isHarness ? 'scorers' : 'metrics';

  // ---- run size (Scoring step formula, Review banner + run matrix) ---------
  // sampleN null = every question. Per suite, a sample larger than the suite
  // simply runs the whole suite.
  const sampleN = draft.runSamplesMode === 'full' || Number.isNaN(draft.runSamples) ? null : draft.runSamples;
  const suiteQuestions = (count: number) => (sampleN === null ? count : Math.min(sampleN, count));
  const harnessQuestionsPerPass = selectedHarnessSuites.reduce((n, s) => n + suiteQuestions(s.question_count ?? 0), 0);
  const harnessTaskCount = harnessQuestionsPerPass * draft.models.length * draft.harnessAgents.length;
  const coveredUseCases = harnessUseCases.filter((u) => selectedHarnessSuites.some((s) => s.use_case === u));
  const uncoveredUseCases = harnessUseCases.filter((u) => !coveredUseCases.includes(u));

  // ---- Models step (Model / Agent): table + "In this run" panel ------------
  const llmAvailable = availableModels.filter((m) => (m?.category ?? '').toLowerCase() === 'llm');
  const tableModels = filteredModels.filter(
    (m) => modelProviderFilter === 'all' || m.provider_id === modelProviderFilter
  );
  // ---- Type step: "About" panel -------------------------------------------
  // What the selected type measures, what it's good for, and how a run of
  // that type works (copy in TYPE_ABOUT).
  const typeAboutPanel = (() => {
    const opt = TYPE_OPTIONS.find((o) => o.value === draft.type);
    const about = draft.type ? TYPE_ABOUT[draft.type] : undefined;
    if (!opt || !about) {
      return (
        <p className={styles['ev__about-empty']}>
          <Info size={14} /> Select a system under test to see what it measures and when to use it.
        </p>
      );
    }
    const Icon = opt.icon;
    return (
      <div className={`${styles.ev__about} ${styles.ev__anim}`} key={opt.label}>
        <div className={styles['ev__about-head']}>
          <span
            className={`${styles['ev__option-icon']} ${
              opt.variant ? styles[`ev__option-icon--${opt.variant}`] : ''
            }`}
          >
            <Icon size={18} />
          </span>
          <div>
            <h3>About {opt.label} evaluation</h3>
            <p>{about.tagline}</p>
          </div>
        </div>
        <div className={styles['ev__about-cols']}>
          <div className={styles['ev__about-col']}>
            <p className={styles['ev__about-k']}>What it measures</p>
            <ul className={styles['ev__about-list']}>
              {about.measures.map((t) => (
                <li key={t}>
                  <Check size={13} strokeWidth={3} /> {t}
                </li>
              ))}
            </ul>
          </div>
          <div className={styles['ev__about-col']}>
            <p className={styles['ev__about-k']}>Good for</p>
            <ul className={styles['ev__about-list']}>
              {about.goodFor.map((t) => (
                <li key={t}>
                  <ChevronRight size={13} strokeWidth={3} /> {t}
                </li>
              ))}
            </ul>
          </div>
          <div className={styles['ev__about-col']}>
            <p className={styles['ev__about-k']}>How a run works</p>
            <ol className={styles['ev__about-flow']}>
              {about.flow.map((t, n) => (
                <li key={t}>
                  <span>{n + 1}</span> {t}
                </li>
              ))}
            </ol>
          </div>
        </div>
      </div>
    );
  })();

  // ---- Name step: quick-add tokens ----------------------------------------
  // One-click suffixes for the run name: this quarter, a version, today's
  // date. Clicking a token that's already in the name takes it back out.
  const nameTokens = (() => {
    const now = new Date();
    return [
      `Q${Math.floor(now.getMonth() / 3) + 1} ${now.getFullYear()}`,
      'v1',
      now.toLocaleDateString(undefined, { day: 'numeric', month: 'short' }),
    ];
  })();
  const toggleNameToken = (token: string) => {
    const at = draft.name.toLowerCase().indexOf(token.toLowerCase());
    const next =
      at === -1
        ? `${draft.name.trim()} ${token}`.trim()
        : `${draft.name.slice(0, at)}${draft.name.slice(at + token.length)}`.replace(/\s{2,}/g, ' ').trim();
    dispatch(setDraft({ name: next }));
  };

  // Search field of whichever list header is on screen (one step at a time).
  // Pressing "/" anywhere outside a text field jumps to it.
  const listSearchRef = useRef<HTMLInputElement>(null);
  useEffect(() => {
    const onKey = (e: KeyboardEvent) => {
      if (e.key !== '/' || e.metaKey || e.ctrlKey || e.altKey) return;
      const t = e.target as HTMLElement | null;
      if (t && (t.tagName === 'INPUT' || t.tagName === 'TEXTAREA' || t.isContentEditable)) return;
      if (!listSearchRef.current) return;
      e.preventDefault();
      listSearchRef.current.focus();
    };
    window.addEventListener('keydown', onKey);
    return () => window.removeEventListener('keydown', onKey);
  }, []);

  // ---- list header ---------------------------------------------------------
  // Shared header for the Providers, Models and Test Suite lists: a title
  // with a live count on the left, search + refresh on the right, an optional
  // row of filter chips underneath, and a rule separating it from the list.
  const listHead = (o: {
    title: string;
    meta: ReactNode;
    // how many items the list is showing right now (drives the result chip)
    count: number;
    search: { placeholder: string; value: string; onChange: (v: string) => void };
    refresh: { onClick: () => void; busy: boolean; title: string };
    filters?: ReactNode;
    sticky?: boolean;
  }) => {
    const hasQuery = o.search.value.trim() !== '';
    return (
      <div className={`${styles.ev__lhead} ${o.sticky ? styles['ev__lhead--sticky'] : ''}`}>
        <div className={styles['ev__lhead-row']}>
        <div className={styles['ev__lhead-title']}>
          <h3>{o.title}</h3>
          <span>{o.meta}</span>
        </div>

        {/* Search bar: one compact control holding the icon, the field, a
            live result count, the "/" hint or clear button, and refresh. */}
        <div
          className={`${styles.ev__sbar} ${hasQuery ? styles['ev__sbar--active'] : ''}`}
          onMouseDown={(e) => {
            // clicking anywhere on the bar (not on its buttons) focuses the field
            if ((e.target as HTMLElement).closest('button, input')) return;
            e.preventDefault();
            listSearchRef.current?.focus();
          }}
        >
          <span className={styles['ev__sbar-icon']}>
            <Search size={13} strokeWidth={2.6} />
          </span>
          <input
            ref={listSearchRef}
            type="text"
            placeholder={o.search.placeholder}
            aria-label={o.search.placeholder}
            value={o.search.value}
            onChange={(e) => o.search.onChange(e.target.value)}
            onKeyDown={(e) => {
              // Esc clears first, then leaves the field
              if (e.key !== 'Escape') return;
              if (o.search.value) o.search.onChange('');
              else e.currentTarget.blur();
            }}
          />
          {hasQuery && (
            <span
              className={`${styles['ev__sbar-count']} ${o.count === 0 ? styles['ev__sbar-count--none'] : ''}`}
              aria-live="polite"
            >
              {o.count === 0 ? 'No matches' : `${o.count} result${o.count === 1 ? '' : 's'}`}
            </span>
          )}
          {o.search.value ? (
            <button
              type="button"
              className={styles['ev__sbar-clear']}
              onClick={() => {
                o.search.onChange('');
                listSearchRef.current?.focus();
              }}
              aria-label="Clear search"
            >
              <X size={12} strokeWidth={2.5} />
            </button>
          ) : (
            <kbd className={styles['ev__sbar-kbd']} title="Press / to search">
              /
            </kbd>
          )}
          <span className={styles['ev__sbar-sep']} />
          <button
            type="button"
            className={styles['ev__sbar-refresh']}
            onClick={o.refresh.onClick}
            disabled={o.refresh.busy}
            title={o.refresh.title}
            aria-label={o.refresh.title}
          >
            <RefreshCw size={14} className={o.refresh.busy ? styles.ev__spin : ''} />
          </button>
        </div>
        </div>

        {o.filters && <div className={styles['ev__lhead-filters']}>{o.filters}</div>}
      </div>
    );
  };

  // One − / value / + control. The buttons step by `step` and clamp to the
  // field's range straight away; typing still works as before (free typing,
  // clamped on blur).
  const retryStepper = (
    field: keyof ModelRetryConfig,
    label: string,
    icon: ReactNode,
    step: number,
    value: ModelRetryConfig | undefined,
    onChange: (field: keyof ModelRetryConfig, raw: number) => void,
    onBlur: (field: keyof ModelRetryConfig) => void
  ) => {
    const min = field === 'max_retries' ? MIN_MAX_RETRIES : MIN_TIMEOUT;
    const max = field === 'max_retries' ? MAX_MAX_RETRIES : MAX_TIMEOUT;
    const fallback = field === 'max_retries' ? DEFAULT_MAX_RETRIES : DEFAULT_TIMEOUT;
    const raw = value?.[field] ?? fallback;
    const current = Number.isNaN(raw) ? fallback : raw;
    const bump = (dir: 1 | -1) => onChange(field, clampRetryField(field, current + dir * step));
    return (
      <div className={styles.ev__stepfield}>
        <span className={styles['ev__stepfield-label']}>
          {icon} {label}
        </span>
        <div className={styles.ev__stepper}>
          <button type="button" onClick={() => bump(-1)} disabled={current <= min} aria-label={`Decrease ${label}`}>
            <Minus size={13} />
          </button>
          <input
            type="number"
            min={min}
            max={max}
            aria-label={label}
            value={displayNumInput(raw)}
            onChange={(e) => onChange(field, parseNumInput(e.target.value))}
            onBlur={() => onBlur(field)}
          />
          <button type="button" onClick={() => bump(1)} disabled={current >= max} aria-label={`Increase ${label}`}>
            <Plus size={13} />
          </button>
        </div>
      </div>
    );
  };

  const retryFields = (
    value: ModelRetryConfig | undefined,
    onChange: (field: keyof ModelRetryConfig, raw: number) => void,
    onBlur: (field: keyof ModelRetryConfig) => void
  ) => (
    <div className={styles.ev__stepfields}>
      {retryStepper('max_retries', 'Max retries', <RotateCw size={12} />, 1, value, onChange, onBlur)}
      {retryStepper('timeout', 'Timeout (s)', <Timer size={12} />, 30, value, onChange, onBlur)}
    </div>
  );

  const modelsTableView = (
    <div className={styles.ev__split}>
      <div className={styles['ev__split-main']}>
        {listHead({
          title: 'Available models',
          meta: (
            <>
              {tableModels.length} of {llmAvailable.length} shown · <b>{draft.models.length}</b> selected
            </>
          ),
          count: tableModels.length,
          search: { placeholder: 'Search models', value: modelSearch, onChange: setModelSearch },
          refresh: { onClick: refreshModels, busy: modelsLoading, title: 'Refresh models' },
          filters: (
            <>
            <button
              type="button"
              className={`${styles['ev__filter-chip']} ${modelProviderFilter === 'all' ? styles['ev__filter-chip--on'] : ''}`}
              onClick={() => setModelProviderFilter('all')}
            >
              All <span className={styles['ev__filter-chip-count']}>{llmAvailable.length}</span>
            </button>
            {draft.providers.map((id) => {
              const on = modelProviderFilter === id;
              return (
                <button
                  key={id}
                  type="button"
                  className={`${styles['ev__filter-chip']} ${on ? styles['ev__filter-chip--on'] : ''}`}
                  onClick={() => setModelProviderFilter(on ? 'all' : id)}
                >
                  {providers.find((p) => p?.id === id)?.name || id}
                  <span className={styles['ev__filter-chip-count']}>
                    {llmAvailable.filter((m) => m.provider_id === id).length}
                  </span>
                </button>
              );
            })}
            </>
          ),
        })}

        {/* Model cards (original card design). Retries and timeout for the
            selected models are managed in the side panel, so the cards
            don't carry those fields here. */}
        <div className={styles['ev__models-scroll']}>
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
              {tableModels.map((m) => renderModelCard(m, () => toggleModel(m.id), false))}
              {tableModels.length === 0 && (
                <p className={styles.ev__empty}>
                  {modelSearch ? `No models match "${modelSearch}".` : 'No models for this filter.'}
                </p>
              )}
            </div>
          )}
        </div>
      </div>

      {/* Retries and timeout stay pinned at the top of the panel; the list
          of selected models scrolls underneath. */}
      <aside className={`${styles.ev__panel} ${styles['ev__panel--pinned']}`}>
        <div className={styles['ev__panel-fixed']}>
          <p className={styles['ev__panel-title']}>Retries and timeout</p>
          <div className={`${styles.ev__segctl} ${styles['ev__segctl--full']}`}>
            {[
              { v: 'all' as const, l: 'Same for all' },
              { v: 'individual' as const, l: 'Per model' },
            ].map((o) => (
              <button
                key={o.v}
                type="button"
                className={`${styles.ev__segbtn} ${draft.retryConfigMode === o.v ? styles['ev__segbtn--on'] : ''}`}
                onClick={() => dispatch(setDraft({ retryConfigMode: o.v }))}
              >
                {o.l}
              </button>
            ))}
          </div>
          {draft.retryConfigMode === 'all' && retryFields(draft.retryConfigAll, updateRetryConfigAll, clampRetryConfigAll)}
          <p className={styles['ev__panel-note']}>
            {draft.retryConfigMode === 'individual' ? 'Set on each model below. ' : ''}Retries {MIN_MAX_RETRIES} to{' '}
            {MAX_MAX_RETRIES}. Timeout {MIN_TIMEOUT} to {MAX_TIMEOUT} seconds.
          </p>
        </div>
        <div className={styles['ev__panel-scroll']}>
          <p className={styles['ev__panel-title']}>
            In this run: {selectedModels.length} model{selectedModels.length === 1 ? '' : 's'}
          </p>
          {selectedModels.length === 0 ? (
            <p className={styles['ev__panel-note']}>Nothing selected yet. Pick a healthy model card.</p>
          ) : (
            <div className={styles['ev__panel-list']}>
              {selectedModels.map((m) => {
                const health: HealthStatus = healthById?.[m.id] ?? 'idle';
                const ready = health === 'success' && Boolean(verifiedModelIds[m.id]);
                return (
                  <div key={m.id} className={styles.ev__pm}>
                    <div className={styles['ev__pm-head']}>
                      <span
                        className={`${styles['ev__pm-dot']} ${
                          ready ? styles['ev__pm-dot--ok'] : health === 'failed' ? styles['ev__pm-dot--fail'] : ''
                        }`}
                        title={ready ? 'Healthy' : health === 'failed' ? 'Health check failed' : 'Waiting for health check'}
                      />
                      <span className={styles['ev__pm-name']}>
                        {m.name || 'Unnamed model'}
                        <small>
                          {providers.find((p) => p?.id === m.provider_id)?.name ?? m.provider_id}
                          {!ready && (health === 'failed' ? ' · health check failed' : ' · waiting for health check')}
                        </small>
                      </span>
                      <button
                        type="button"
                        className={styles['ev__panel-x']}
                        onClick={() => toggleModel(m.id)}
                        aria-label={`Remove ${m.name || 'model'}`}
                      >
                        <X size={14} />
                      </button>
                    </div>
                    {draft.retryConfigMode === 'individual' && (
                      <div className={styles['ev__pm-body']}>
                        {retryFields(
                          draft.modelRetryConfig[m.id],
                          (field, raw) => updateModelRetryConfig(m.id, field, raw),
                          (field) => clampModelRetryConfig(m.id, field)
                        )}
                      </div>
                    )}
                  </div>
                );
              })}
            </div>
          )}
        </div>
      </aside>
    </div>
  );

  // ---- Test Suite step (harness): scorecard coverage panel ----------------
  const coveragePanel = (
    <aside className={styles.ev__panel}>
      <p className={styles['ev__panel-title']}>Scorecard coverage</p>
      <p className={styles['ev__panel-sub']}>
        {coveredUseCases.length} of {harnessUseCases.length} use cases covered
      </p>
      <div className={styles['ev__panel-list']}>
        {harnessUseCases.map((u) => {
          const on = coveredUseCases.includes(u);
          return (
            <div key={u} className={`${styles['ev__panel-item']} ${on ? '' : styles['ev__panel-item--off']}`}>
              <div className={styles['ev__panel-cov']}>
                <span className={`${styles.ev__dot} ${on ? styles['ev__dot--on'] : ''}`}>
                  {on && <Check size={10} strokeWidth={3.5} />}
                </span>
                {on ? u : `${u}: not covered`}
              </div>
            </div>
          );
        })}
      </div>
      <div className={styles['ev__panel-stats']}>
        <div>
          <span>Suites selected</span>
          <b>{selectedHarnessSuites.length}</b>
        </div>
        <div>
          <span>Questions available</span>
          <b>{selectedHarnessSuites.reduce((n, d) => n + (d.question_count ?? 0), 0).toLocaleString()}</b>
        </div>
      </div>
      <p className={styles['ev__panel-note']}>You choose how many questions to run in the next step.</p>
    </aside>
  );

  // ---- Review step: banner + settings rows --------------------------------
  const metricDisplayNames = draft.metrics.map((m) => customMetricsCatalog.find((c) => c.id === m)?.name ?? m);
  const reviewStats = isHarness
    ? [
        { k: 'Runs', v: harnessRunCount.toLocaleString() },
        { k: 'Tasks', v: harnessTaskCount.toLocaleString() },
        { k: 'Scorers', v: String(draft.metrics.length) },
      ]
    : [
        { k: 'Questions', v: suite ? suiteQuestions(suite.question_count ?? 0).toLocaleString() : '—' },
        { k: 'Models', v: String(selectedModels.length) },
        { k: 'Metrics', v: hideMetricsStep ? '—' : String(draft.metrics.length) },
      ];
  const reviewSummaryLine = isHarness
    ? `Agent harness comparing ${harnessAgentTitles.join(' and ') || 'no agents yet'}`
    : `${typeLabel || 'Evaluation'}${frameworkTitle ? ` (${frameworkTitle})` : ''} on ${suite?.name ?? 'no suite yet'}`;
  const retryText =
    draft.retryConfigMode === 'all'
      ? `${clampRetryField('max_retries', draft.retryConfigAll.max_retries)} ${
          clampRetryField('max_retries', draft.retryConfigAll.max_retries) === 1 ? 'retry' : 'retries'
        }, ${clampRetryField('timeout', draft.retryConfigAll.timeout)} second timeout`
      : 'Retries and timeout set per model';
  const retestShown =
    draft.type === 'model' || draft.type === 'rag' || (draft.type === 'agent' && Boolean(draft.agentFramework));
  const reviewRows: { k: string; v: string; step: number }[] = [
    { k: 'Type', v: typeManifestValue || '—', step: 1 },
    { k: 'Providers', v: providerNames.join(', ') || '—', step: 2 },
    {
      k: 'Models',
      v: selectedModels.length > 0 ? `${selectedModels.map((m) => m.name || 'Unnamed model').join(', ')}. ${retryText}` : '—',
      step: 3,
    },
    {
      k: isHarness ? 'Test suites' : 'Test suite',
      v: isHarness
        ? selectedHarnessSuites.map((d) => d.name).join(', ') || '—'
        : [
            suite?.name ?? '—',
            draft.subgroup.length > 0 ? `${draft.subgroup.length} subgroup${draft.subgroup.length === 1 ? '' : 's'}` : null,
            isRag ? `Top K ${draft.topK}` : null,
          ]
            .filter(Boolean)
            .join(', '),
      step: 4,
    },
    {
      k: 'Sample size',
      v: sampleN === null ? 'Every question' : `${sampleN} questions per ${isHarness ? 'suite' : 'model'}`,
      step: 5,
    },
    ...(hideMetricsStep
      ? []
      : [{ k: isHarness ? 'Scorers' : 'Metrics', v: metricDisplayNames.join(', ') || '—', step: 5 }]),
    { k: 'Judge model', v: judgeModel ? judgeModel.name || 'Unnamed model' : '—', step: 5 },
    ...(isHarness
      ? [
          {
            k: 'Limits per task',
            v: `${clampHarnessSteps(draft.harnessSettings.max_steps)} steps, ${clampRetryField(
              'timeout',
              draft.harnessSettings.task_timeout
            )} seconds, ${draft.harnessSettings.save_traces ? 'traces saved for replay' : 'traces not saved'}`,
            step: 5,
          },
        ]
      : []),
    ...(retestShown && draft.retestOnWrong
      ? [
          {
            k: 'Retest on wrong',
            v: `Up to ${clampRetestRounds(draft.retestMaxRounds)} rounds, verified by ${draft.retestVerifyMetric ?? '—'}`,
            step: 5,
          },
        ]
      : []),
    ...(draft.instruction.trim() ? [{ k: 'Instruction', v: draft.instruction.trim(), step: 5 }] : []),
  ];

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
                  <span className={styles['ev__manifest-pct']}>{Math.round((step / totalSteps) * 100)}%</span>
                </div>
                <div className={styles['ev__manifest-title']} data-empty={!draft.name}>
                  {draft.name || 'Untitled run'}
                </div>
                <div className={styles.ev__segs}>
                  {STEPS.map((st, i) => (
                    <span
                      key={st.label}
                      className={`${styles.ev__seg} ${
                        i < step ? styles['ev__seg--done'] : i === step ? styles['ev__seg--active'] : ''
                      }`}
                    />
                  ))}
                </div>
                <p className={styles['ev__manifest-done']}>
                  {step} of {totalSteps} steps done
                </p>
              </div>
              <div className={styles.ev__spec}>
                {STEPS.map((st, i) => {
                  const state = i === step ? 'active' : i < step ? 'done' : 'todo';
                  const row = manifest[i];
                  return (
                    <button
                      key={st.label}
                      type="button"
                      className={`${styles['ev__spec-row']} ${styles[`ev__spec-row--${state}`]}`}
                      onClick={() => goToStep(i)}
                      disabled={i > step}
                    >
                      <span className={styles['ev__spec-tick']}>
                        {state === 'done' ? <Check size={13} strokeWidth={3} /> : i + 1}
                      </span>
                      <span className={styles['ev__spec-body']}>
                        <span className={styles['ev__spec-label']}>{st.label}</span>
                        <span className={styles['ev__spec-value']} data-empty={row.empty}>
                          {row.value}
                        </span>
                      </span>
                      {state === 'done' && <span className={styles['ev__spec-edit']}>Edit</span>}
                    </button>
                  );
                })}
              </div>
              <p className={styles['ev__manifest-foot']}>Select a completed step to go back and edit it.</p>
            </aside>

            {/* STAGE */}
            <section className={styles.ev__stage}>
              <div className={styles['ev__stage-head']}>
                <p className={styles['ev__stage-step']}>
                  Step {step + 1} of {totalSteps}
                </p>
                <h2 className={styles['ev__stage-title']}>{stageTitle}</h2>
                <p className={styles['ev__stage-sub']}>{stageSub}</p>
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
                  {/* Left: the name field, quick-add tokens, presets and naming
                      tips. Right: a live preview of the row this run will get in
                      History, so "how it appears in your history" is something
                      you can see while typing. */}
                  {step === 0 && (
                    <div className={styles.ev__namegrid}>
                      <div className={styles['ev__namegrid-main']}>
                        <div className={styles.ev__field}>
                          <label className={styles.ev__label}>Run name</label>
                          <div className={styles['ev__name-wrap']}>
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
                            {draft.name && (
                              <button
                                type="button"
                                className={styles['ev__name-clear']}
                                onClick={() => dispatch(setDraft({ name: '' }))}
                                aria-label="Clear name"
                              >
                                <X size={14} />
                              </button>
                            )}
                          </div>
                          {draft.name.trim() ? (
                            <p className={`${styles['ev__name-caption']} ${styles['ev__name-caption--ok']}`}>
                              <Check size={13} strokeWidth={3} /> Ready. Press Enter to continue, or keep refining.
                            </p>
                          ) : (
                            <p className={styles['ev__name-caption']}>
                              <Tag size={13} /> This is how the run appears in your history.
                            </p>
                          )}
                        </div>

                        {/* Quick add: append a period, version or date to the name
                            in one click (click again to take it back off). */}
                        <div className={styles.ev__quick}>
                          <p className={styles['ev__quick-head']}>Quick add</p>
                          <div className={styles['ev__quick-row']}>
                            {nameTokens.map((t) => {
                              const on = draft.name.toLowerCase().includes(t.toLowerCase());
                              return (
                                <button
                                  key={t}
                                  type="button"
                                  className={`${styles.ev__preset} ${on ? styles['ev__preset--on'] : ''}`}
                                  aria-pressed={on}
                                  onClick={() => toggleNameToken(t)}
                                >
                                  {on ? <Check size={13} strokeWidth={3} /> : <Plus size={13} strokeWidth={2.5} />} {t}
                                </button>
                              );
                            })}
                          </div>
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
                      </div>

                      <aside className={styles.ev__hprev}>
                        <p className={styles['ev__hprev-head']}>Preview in History</p>
                        <div className={`${styles['ev__hprev-card']} ${draft.name.trim() ? styles['ev__hprev-card--live'] : ''}`}>
                          <div className={styles['ev__hprev-top']}>
                            <span className={styles['ev__hprev-icon']}>
                              <Tag size={15} />
                            </span>
                            <span className={styles['ev__hprev-name']} data-empty={!draft.name.trim()}>
                              {draft.name.trim() || 'Untitled run'}
                            </span>
                          </div>
                          <div className={styles['ev__hprev-badges']}>
                            <span className={styles.ev__tag}>{typeLabel || 'Type not set'}</span>
                            <span className={styles['ev__hprev-status']}>Draft</span>
                          </div>
                          <div className={styles['ev__hprev-meta']}>
                            <span>{new Date().toLocaleDateString()}</span>
                            <span>
                              {draft.models.length} model{draft.models.length === 1 ? '' : 's'}
                            </span>
                          </div>
                        </div>
                        <p className={styles['ev__hprev-note']}>
                          Updates as you type. The type and models fill in as you complete the next steps.
                        </p>
                      </aside>
                    </div>
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

                      {/* About the selected type. For Agent it goes below the
                          agent options instead, so those stay near the top. */}
                      {draft.type !== 'agent' && typeAboutPanel}

                      {/* Agent setup — the two follow-up choices that only apply to
                          Agent, as one panel with numbered steps:
                            1. how to test (Standard benchmark / Document-agent harness)
                            2. harness: which agents to compare (required)
                               standard: which framework (optional)
                          The two modes are mutually exclusive, so switching clears
                          agentFramework (harness) / harness picks (standard). */}
                      {draft.type === 'agent' && (
                        <div className={`${styles.ev__asetup} ${styles.ev__anim}`}>
                          <div className={styles['ev__asetup-head']}>
                            <span className={`${styles['ev__option-icon']} ${styles['ev__option-icon--agent']}`}>
                              <Bot size={18} />
                            </span>
                            <div className={styles['ev__asetup-title']}>
                              <h3>Agent setup</h3>
                              <p>Two more choices, because you picked Agent.</p>
                            </div>
                            {isHarness && draft.harnessAgents.length === 0 ? (
                              <span className={`${styles['ev__asetup-status']} ${styles['ev__asetup-status--todo']}`}>
                                1 choice left
                              </span>
                            ) : (
                              <span className={styles['ev__asetup-status']}>
                                <Check size={12} strokeWidth={3} /> All set
                              </span>
                            )}
                          </div>

                          {/* 1 — how to test */}
                          <div className={styles['ev__asetup-step']}>
                            <span className={`${styles['ev__asetup-n']} ${styles['ev__asetup-n--done']}`}>
                              <Check size={12} strokeWidth={3} />
                            </span>
                            <div className={styles['ev__asetup-body']}>
                              <p className={styles['ev__asetup-q']}>How should the agent be tested?</p>
                              <div className={styles['ev__asetup-grid']}>
                                {[
                                  {
                                    v: false,
                                    icon: Bot,
                                    l: 'Standard benchmark',
                                    d: 'Run one agent on a single agent benchmark. You can add the framework it runs on.',
                                    tags: ['One benchmark', 'Optional framework'],
                                  },
                                  {
                                    v: true,
                                    icon: Layers,
                                    l: 'Document-agent harness',
                                    d: 'Compare agents on the same document tasks, each in a clean sandbox, with one scorecard.',
                                    tags: [...HARNESS_USE_CASES],
                                  },
                                ].map((o) => {
                                  const on = draft.agentHarness === o.v;
                                  const ModeIcon = o.icon;
                                  return (
                                    <button
                                      key={String(o.v)}
                                      type="button"
                                      className={`${styles.ev__acard} ${on ? styles['ev__acard--on'] : ''}`}
                                      aria-pressed={on}
                                      onClick={() => {
                                        if (on) return;
                                        dispatch(
                                          setDraft({
                                            agentHarness: o.v,
                                            agentFramework: o.v ? null : draft.agentFramework,
                                            harnessAgents: o.v ? draft.harnessAgents : [],
                                            harnessSuiteIds: [],
                                            // the scorer catalog differs per mode, so drop stale picks
                                            metrics: [],
                                            retestOnWrong: false,
                                            retestVerifyMetric: null,
                                          })
                                        );
                                      }}
                                    >
                                      <span className={styles['ev__acard-head']}>
                                        <span className={`${styles.ev__radio} ${on ? styles['ev__radio--on'] : ''}`} />
                                        <span className={styles['ev__srow-icon']}>
                                          <ModeIcon size={15} />
                                        </span>
                                        <span className={styles['ev__acard-name']}>{o.l}</span>
                                      </span>
                                      <span className={styles['ev__acard-desc']}>{o.d}</span>
                                      <span className={styles['ev__acard-tags']}>
                                        {o.tags.map((t) => (
                                          <span key={t} className={`${styles.ev__pilltag} ${on ? '' : styles['ev__pilltag--muted']}`}>
                                            {t}
                                          </span>
                                        ))}
                                      </span>
                                    </button>
                                  );
                                })}
                              </div>
                            </div>
                          </div>

                          {/* 2 — harness: agents to compare (required) */}
                          {isHarness && (
                            <div className={styles['ev__asetup-step']}>
                              <span
                                className={`${styles['ev__asetup-n']} ${
                                  draft.harnessAgents.length > 0 ? styles['ev__asetup-n--done'] : styles['ev__asetup-n--active']
                                }`}
                              >
                                {draft.harnessAgents.length > 0 ? <Check size={12} strokeWidth={3} /> : 2}
                              </span>
                              <div className={styles['ev__asetup-body']}>
                                <p className={styles['ev__asetup-q']}>
                                  Which agents should be compared? <span>Pick one or more</span>
                                </p>
                                <div className={styles['ev__asetup-grid']}>
                                  {HARNESS_AGENTS.map((a) => {
                                    const on = draft.harnessAgents.includes(a.id);
                                    return (
                                      <button
                                        key={a.id}
                                        type="button"
                                        className={`${styles.ev__acard} ${on ? styles['ev__acard--on'] : ''}`}
                                        aria-pressed={on}
                                        onClick={() =>
                                          dispatch(
                                            setDraft({ harnessAgents: toggle(draft.harnessAgents, a.id) as HarnessAdapterId[] })
                                          )
                                        }
                                      >
                                        <span className={styles['ev__acard-head']}>
                                          <span className={`${styles.ev__check} ${on ? styles['ev__check--on'] : ''}`}>
                                            {on && <Check size={11} strokeWidth={3} />}
                                          </span>
                                          <span className={styles['ev__srow-icon']}>
                                            <Waypoints size={15} />
                                          </span>
                                          <span className={styles['ev__acard-name']}>{a.title}</span>
                                        </span>
                                        <span className={styles['ev__acard-desc']}>{a.desc}</span>
                                      </button>
                                    );
                                  })}
                                </div>
                                {draft.harnessAgents.length === 0 ? (
                                  <p className={`${styles['ev__asetup-sum']} ${styles['ev__asetup-sum--req']}`}>
                                    <AlertTriangle size={13} /> Select at least one agent to continue.
                                  </p>
                                ) : (
                                  <p className={styles['ev__asetup-sum']}>
                                    <Info size={13} />
                                    {draft.harnessAgents.length === 1 ? (
                                      <span>
                                        <b>{harnessAgentTitles[0]}</b> runs alone. Add another agent to compare them side by side.
                                      </span>
                                    ) : (
                                      <span>
                                        <b>{harnessAgentTitles.join(' vs ')}</b>: same tasks, same sandbox, one scorecard.
                                      </span>
                                    )}
                                  </p>
                                )}
                              </div>
                            </div>
                          )}

                          {/* 2 — standard: framework (optional) */}
                          {!isHarness && (
                            <div className={styles['ev__asetup-step']}>
                              <span
                                className={`${styles['ev__asetup-n']} ${draft.agentFramework ? styles['ev__asetup-n--done'] : ''}`}
                              >
                                {draft.agentFramework ? <Check size={12} strokeWidth={3} /> : 2}
                              </span>
                              <div className={styles['ev__asetup-body']}>
                                <p className={styles['ev__asetup-q']}>
                                  Which framework does the agent run on? <span>Optional</span>
                                </p>
                                <div className={styles['ev__asetup-grid']}>
                                  {AGENT_FRAMEWORKS.map((f) => {
                                    const on = draft.agentFramework === f.id;
                                    return (
                                      <button
                                        key={f.id}
                                        type="button"
                                        className={`${styles.ev__acard} ${on ? styles['ev__acard--on'] : ''}`}
                                        aria-pressed={on}
                                        onClick={() => dispatch(setDraft({ agentFramework: on ? null : f.id }))}
                                      >
                                        <span className={styles['ev__acard-head']}>
                                          <span className={`${styles.ev__check} ${on ? styles['ev__check--on'] : ''}`}>
                                            {on && <Check size={11} strokeWidth={3} />}
                                          </span>
                                          <span className={styles['ev__srow-icon']}>
                                            <Waypoints size={15} />
                                          </span>
                                          <span className={styles['ev__acard-name']}>{f.title}</span>
                                        </span>
                                        <span className={styles['ev__acard-desc']}>{f.desc}</span>
                                      </button>
                                    );
                                  })}
                                </div>
                                <p className={styles['ev__asetup-sum']}>
                                  <Info size={13} />
                                  {draft.agentFramework ? (
                                    <span>
                                      Runs on <b>{frameworkTitle}</b>. This also decides which test suites the next steps offer.
                                    </span>
                                  ) : (
                                    <span>No framework selected. The run uses an agent benchmark, and scoring needs only a judge model.</span>
                                  )}
                                </p>
                              </div>
                            </div>
                          )}
                        </div>
                      )}

                      {draft.type === 'agent' && typeAboutPanel}
                    </>
                  )}

                  {/* STEP 2 — PROVIDERS */}
                  {step === 2 && (
                    <div className={styles.ev__scroll}>
                      {listHead({
                        title: 'Connected providers',
                        meta: (
                          <>
                            {filteredProviders.length} of {connectedProviders.length} shown · <b>{draft.providers.length}</b> selected
                          </>
                        ),
                        count: filteredProviders.length,
                        search: { placeholder: 'Search providers', value: providerSearch, onChange: setProviderSearch },
                        refresh: { onClick: refreshProviders, busy: providersLoading, title: 'Refresh providers' },
                      })}
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
                        <div className={`${styles.ev__grid} ${styles['ev__grid--providers']}`}>
                          {filteredProviders.map((p) => {
                            const on = draft.providers.includes(p.id);
                            return (
                              <button
                                key={p.id}
                                type="button"
                                className={`${styles.ev__option} ${styles['ev__option--provider']} ${
                                  on ? styles['ev__option--on'] : ''
                                }`}
                                onClick={() => dispatch(setDraft({ providers: toggle(draft.providers, p.id) }))}
                                aria-pressed={on}
                              >
                                {/* Same card as the Type step: icon tile, name,
                                    description, selection mark on the right. */}
                                <span className={styles['ev__option-icon']}>{providerInitials(p.name)}</span>
                                <span className={styles['ev__option-main']}>
                                  <span className={styles['ev__option-name']}>
                                    {p.name || 'Unnamed provider'}
                                    <span className={styles['ev__pv-status']}>Connected</span>
                                  </span>
                                  {p.description && (
                                    <span className={styles['ev__option-desc']} title={p.description}>
                                      {p.description}
                                    </span>
                                  )}
                                  <span className={styles['ev__option-meta']}>
                                    <Cpu size={13} />
                                    <b>{p.model_count ?? 0}</b> model{(p.model_count ?? 0) === 1 ? '' : 's'} available
                                  </span>
                                </span>
                                {on && (
                                  <span className={styles.ev__mark}>
                                    <Check size={13} strokeWidth={3} />
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
                      !isRag ? (
                        modelsTableView
                      ) : (
                      <div className={styles['ev__models-panel']}>
                        <div className={styles['ev__step-toolbar']}>
                          {/* RAG uses per-category search boxes instead (see
                              the Embedding/Reranker/LLM columns below) — no
                              shared global search box for it. */}
                          {!isRag && (
                            <div className={styles['ev__toolbar-search']}>
                              <Search size={14} />
                              <input
                                placeholder="Search models…"
                                value={modelSearch}
                                onChange={(e) => setModelSearch(e.target.value)}
                              />
                            </div>
                          )}
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

                        {(!modelsFetchedThisVisit || selectedModelsNotReady.length > 0) && (
                          <p className={styles['ev__metrics-required']}>
                            {!modelsFetchedThisVisit || selectedModelsChecking
                              ? 'Checking the health of your selected models — you can continue once they\u2019re all available.'
                              : 'Some selected models aren\u2019t available yet — run a health check on them (or deselect them) to continue.'}
                          </p>
                        )}

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
                                    value={displayNumInput(draft.retryConfigAll.max_retries)}
                                    onChange={(e) => updateRetryConfigAll('max_retries', parseNumInput(e.target.value))}
                                    onBlur={() => clampRetryConfigAll('max_retries')}
                                  />
                                </div>
                                <div className={styles['ev__mcard-retry-field']}>
                                  <label>Timeout (s)</label>
                                  <input
                                    type="number"
                                    min={MIN_TIMEOUT}
                                    max={MAX_TIMEOUT}
                                    value={displayNumInput(draft.retryConfigAll.timeout)}
                                    onChange={(e) => updateRetryConfigAll('timeout', parseNumInput(e.target.value))}
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
                        <div className={styles['ev__models-scroll']}>
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
                        ) : draft.type === 'rag' ? (
                          // RAG: exactly three columns, always in this order
                          // — Embedding, Reranker, LLM. Embedding/Reranker
                          // are single-select (toggleModelForCategory);
                          // LLM is multi-select (plain toggleModel). Each
                          // column has its own search box (ragCategorySearch)
                          // instead of the shared global search used by the
                          // Model/Agent grid.
                          <div className={styles['ev__rag-columns']}>
                            {RAG_CATEGORIES.map((category) => {
                              const allCategoryModels = modelsByCategory.get(category) ?? [];
                              const categoryModels = filteredModelsByCategory.get(category) ?? [];
                              const allCategoryIds = allCategoryModels.map((m) => m.id);
                              const selectedCount = allCategoryModels.filter((m) => draft.models.includes(m.id)).length;
                              const isMultiSelect = category === 'LLM';
                              return (
                                <div key={category} className={styles['ev__rag-col']}>
                                  <div className={styles['ev__rag-col-head']}>
                                    <span className={styles['ev__rag-col-title']}>{category}</span>
                                    <span
                                      className={`${styles['ev__rag-col-status']} ${
                                        selectedCount > 0 ? styles['ev__rag-col-status--on'] : ''
                                      }`}
                                    >
                                      {selectedCount > 0 ? (
                                        <>
                                          <Check size={11} strokeWidth={3} /> {selectedCount} selected
                                        </>
                                      ) : isMultiSelect ? (
                                        'Select one or more'
                                      ) : (
                                        'Select one'
                                      )}
                                    </span>
                                  </div>

                                  <div className={styles['ev__rag-col-search']}>
                                    <Search size={12} />
                                    <input
                                      placeholder={`Search ${category.toLowerCase()}…`}
                                      value={ragCategorySearch[category] ?? ''}
                                      onChange={(e) =>
                                        setRagCategorySearch((prev) => ({ ...prev, [category]: e.target.value }))
                                      }
                                    />
                                  </div>

                                  <div className={styles['ev__rag-col-list']}>
                                    {categoryModels.map((m) =>
                                      renderModelCard(
                                        m,
                                        isMultiSelect
                                          ? () => toggleModel(m.id)
                                          : () => toggleModelForCategory(m.id, allCategoryIds)
                                      )
                                    )}
                                    {categoryModels.length === 0 && allCategoryModels.length > 0 && (
                                      <p className={styles.ev__empty}>
                                        No models match "{ragCategorySearch[category]}".
                                      </p>
                                    )}
                                    {allCategoryModels.length === 0 && (
                                      <p className={styles.ev__empty}>No {category} models available.</p>
                                    )}
                                  </div>
                                </div>
                              );
                            })}
                          </div>
                        ) : (
                        <div className={`${styles.ev__grid} ${styles['ev__grid--wide']}`}>
                          {filteredModels.map((m) => renderModelCard(m, () => toggleModel(m.id)))}
                          {filteredModels.length === 0 && (
                            <p className={styles.ev__empty}>No models match "{modelSearch}".</p>
                          )}
                        </div>
                        )}
                        </div>
                      </div>
                      )
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
                            {listHead({
                              title: 'Test suites',
                              meta: (
                                <>
                                  {filteredDatasets.length} shown ·{' '}
                                  <b>{isHarness ? draft.harnessSuiteIds.length : draft.dataset ? 1 : 0}</b> selected
                                </>
                              ),
                              count: filteredDatasets.length,
                              search: { placeholder: 'Search test suites', value: datasetSearch, onChange: setDatasetSearch },
                              refresh: { onClick: refreshDatasets, busy: datasetsRefreshing, title: 'Refresh test suites' },
                              sticky: true,
                              // Harness mode filters by use case; every other mode keeps
                              // the All / Custom / Deepeval chips.
                              filters: isHarness ? (
                                <>
                                {['all', ...harnessUseCases].map((u) => {
                                  const on = useCaseFilter === u;
                                  const count =
                                    u === 'all' ? harnessSuites.length : harnessSuites.filter((s) => s.use_case === u).length;
                                  return (
                                    <button
                                      key={u}
                                      type="button"
                                      className={`${styles['ev__filter-chip']} ${on ? styles['ev__filter-chip--on'] : ''}`}
                                      onClick={() => setUseCaseFilter(u)}
                                    >
                                      {u === 'all' ? 'All' : u}
                                      <span className={styles['ev__filter-chip-count']}>{count}</span>
                                    </button>
                                  );
                                })}
                                </>
                              ) : (
                                <>
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
                                </>
                              ),
                            })}

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
                                <div className={styles.ev__slist}>
                                  {filteredDatasets.map((d) => {
                                    const on = isSuiteOn(d.id);
                                    const isCustom = d?.dataset_type === 'custom';
                                    const isDeepeval = d?.dataset_type === 'deepeval';
                                    const hasNoQuestions = (d.question_count ?? 0) <= 0;
                                    const subgroups = d.dataset_categories || [];
                                    // Preview: hidden for harness suites while the catalog is
                                    // mocked, and for agent_benchmark except the three
                                    // previewable datasets (same rules as before).
                                    const canPreview =
                                      !isHarness &&
                                      (!isAgentBenchmarkNoFramework || AGENT_BENCHMARK_PREVIEWABLE_DATASETS.has(d.name));
                                    return (
                                      <div
                                        key={d.id}
                                        role="button"
                                        tabIndex={0}
                                        className={`${styles.ev__srow} ${on ? styles['ev__srow--on'] : ''} ${
                                          hasNoQuestions ? styles['ev__srow--empty'] : ''
                                        }`}
                                        onClick={() => pickSuite(d.id)}
                                        onKeyDown={(e) => {
                                          if (e.key === 'Enter' || e.key === ' ') {
                                            e.preventDefault();
                                            pickSuite(d.id);
                                          }
                                        }}
                                        aria-pressed={on}
                                      >
                                        {/* Header: selection control + name */}
                                        <div className={styles['ev__srow-head']}>
                                          {isHarness ? (
                                            <span className={`${styles.ev__check} ${on ? styles['ev__check--on'] : ''}`}>
                                              {on && <Check size={11} strokeWidth={3} />}
                                            </span>
                                          ) : (
                                            <span className={`${styles.ev__radio} ${on ? styles['ev__radio--on'] : ''}`} />
                                          )}
                                          <span className={styles['ev__srow-icon']}>
                                            <Database size={15} />
                                          </span>
                                          <span className={styles['ev__srow-name']} title={d.name || 'Untitled dataset'}>
                                            {d.name || 'Untitled dataset'}
                                          </span>
                                        </div>

                                        {/* Tags: use case / category, source, images */}
                                        <div className={styles['ev__srow-tags']}>
                                          {isHarness && d.use_case && (
                                            <span className={`${styles.ev__pilltag} ${on ? '' : styles['ev__pilltag--muted']}`}>
                                              {d.use_case}
                                            </span>
                                          )}
                                          {/* category tag, unless it would just repeat the Custom / Deepeval tag */}
                                          {!isHarness && d.category && d.category.toLowerCase() !== String(d.dataset_type ?? '').toLowerCase() && (
                                            <span className={`${styles.ev__pilltag} ${styles['ev__pilltag--muted']}`}>
                                              {d.category}
                                            </span>
                                          )}
                                          {isCustom && <span className={styles.ev__pilltag}>Custom</span>}
                                          {isDeepeval && (
                                            <span className={`${styles.ev__pilltag} ${styles['ev__pilltag--muted']}`}>Deepeval</span>
                                          )}
                                          {d.has_images && (
                                            <span
                                              className={`${styles.ev__pilltag} ${styles['ev__pilltag--muted']}`}
                                              title="Questions include images, so every selected model must accept image input"
                                            >
                                              <ImageIcon size={11} /> Images
                                            </span>
                                          )}
                                        </div>

                                        <p
                                          className={`${styles['ev__srow-desc']} ${
                                            d.description ? '' : styles['ev__srow-desc--none']
                                          }`}
                                          title={d.description || undefined}
                                        >
                                          {d.description || 'No description provided.'}
                                        </p>
                                        {d.scoring && (
                                          <p className={styles['ev__srow-scoring']}>
                                            <b>Scored by</b> {d.scoring}
                                          </p>
                                        )}

                                        {/* Subgroups this suite is split into (first few) */}
                                        {subgroups.length > 0 && (
                                          <div className={styles['ev__srow-subs']}>
                                            <span className={styles['ev__srow-k']}>Subgroups</span>
                                            <div className={styles['ev__srow-chips']}>
                                              {subgroups.slice(0, 3).map((g) => (
                                                <span key={g} className={styles.ev__cap}>
                                                  {g}
                                                </span>
                                              ))}
                                              {subgroups.length > 3 && (
                                                <span className={styles.ev__cap} title={subgroups.slice(3).join(', ')}>
                                                  +{subgroups.length - 3} more
                                                </span>
                                              )}
                                            </div>
                                          </div>
                                        )}

                                        {/* Footer: numbers on the left, Preview on the right */}
                                        <div className={styles['ev__srow-side']}>
                                          <div className={styles.ev__stat}>
                                            <span className={styles['ev__stat-k']}>Questions</span>
                                            <span
                                              className={`${styles['ev__stat-v']} ${
                                                hasNoQuestions ? styles['ev__srow-count--empty'] : ''
                                              }`}
                                              title={
                                                hasNoQuestions
                                                  ? "This test suite has no questions yet, so it can't be used."
                                                  : undefined
                                              }
                                            >
                                              {(d.question_count ?? 0).toLocaleString()}
                                            </span>
                                          </div>
                                          <div className={styles.ev__stat}>
                                            <span className={styles['ev__stat-k']}>Subgroups</span>
                                            <span className={styles['ev__stat-v']}>{subgroups.length || '—'}</span>
                                          </div>
                                          {canPreview && (
                                            <button
                                              type="button"
                                              className={styles['ev__srow-preview']}
                                              onClick={(e) => {
                                                e.stopPropagation();
                                                openPreview(d.id, d.name, d.question_count ?? 0);
                                              }}
                                            >
                                              <FileText size={13} /> Preview
                                            </button>
                                          )}
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

                          {/* Subgroups apply to a single dataset — not shown for the
                              multi-suite harness. */}
                          {isHarness && coveragePanel}
                          {!isHarness && (
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
                          )}
                        </div>
                      )}

                      {isHarness && selectedHarnessSuites.some((s) => (s.question_count ?? 0) <= 0) && (
                        <div className={styles['ev__dataset-empty-warning']}>
                          <Info size={14} />
                          <span>
                            One or more selected suites have no questions, so they can't be used for a run. Deselect
                            them to continue.
                          </span>
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

                  {/* STEP 5 — SCORING */}
                  {step === 5 && (
                    <div className={styles.ev__split}>
                    <div className={`${styles.ev__form} ${styles['ev__form--scroll']}`}>
                      {/* How much to run */}
                      <section className={styles.ev__frow}>
                        <div className={styles['ev__frow-side']}>
                          <h3>How much to run</h3>
                          <p>
                            {isHarness
                              ? 'Questions taken from each suite, for every agent and model.'
                              : 'Questions taken from the suite, for every model.'}
                          </p>
                        </div>
                        <div className={styles['ev__frow-main']}>
                          <div className={styles.ev__inline}>
                            <div className={styles.ev__segctl}>
                              {[
                                { v: 'custom' as const, l: 'A sample' },
                                { v: 'full' as const, l: 'Every question' },
                              ].map((o) => (
                                <button
                                  key={o.v}
                                  type="button"
                                  className={`${styles.ev__segbtn} ${
                                    draft.runSamplesMode === o.v ? styles['ev__segbtn--on'] : ''
                                  }`}
                                  onClick={() => dispatch(setDraft({ runSamplesMode: o.v }))}
                                >
                                  {o.l}
                                </button>
                              ))}
                            </div>
                            {draft.runSamplesMode === 'custom' && (
                              <>
                                <input
                                  type="number"
                                  min={0}
                                  max={maxRunSamples}
                                  className={styles.ev__num}
                                  value={displayNumInput(draft.runSamples)}
                                  onChange={(e) => {
                                    // An emptied field is held as NaN so it shows blank instead
                                    // of snapping to 0; onBlur below (and launch) resolve it.
                                    const raw = parseNumInput(e.target.value);
                                    const val = Number.isNaN(raw) ? NaN : Math.min(maxRunSamples, Math.max(0, raw));
                                    dispatch(setDraft({ runSamples: val }));
                                  }}
                                  onBlur={() => {
                                    if (Number.isNaN(draft.runSamples)) dispatch(setDraft({ runSamples: 0 }));
                                  }}
                                />
                                <span className={styles['ev__inline-note']}>
                                  questions per {isHarness ? 'suite' : 'model'}
                                </span>
                              </>
                            )}
                          </div>
                          {isHarness ? (
                            <p className={styles.ev__formula}>
                              {sampleN !== null &&
                              selectedHarnessSuites.every((d) => (d.question_count ?? 0) >= sampleN)
                                ? `${sampleN} questions × ${selectedHarnessSuites.length} suites`
                                : `${harnessQuestionsPerPass.toLocaleString()} questions across ${
                                    selectedHarnessSuites.length
                                  } suites`}{' '}
                              × {draft.models.length} models × {draft.harnessAgents.length} agents ={' '}
                              <b>{harnessTaskCount.toLocaleString()} tasks</b>
                            </p>
                          ) : (
                            <p className={styles['ev__inline-note']}>
                              {draft.runSamplesMode === 'full'
                                ? 'Every question in the suite will be used.'
                                : `Up to ${maxRunSamples.toLocaleString()} questions available in this suite.`}
                            </p>
                          )}
                        </div>
                      </section>

                      {/* What to score */}
                      <section className={styles.ev__frow}>
                        <div className={styles['ev__frow-side']}>
                          <h3>What to score</h3>
                          {hideMetricsStep ? (
                            <p>Fixed for this run.</p>
                          ) : (
                            <>
                              <p>
                                {draft.metrics.length} of {metricsCatalog.length + customMetricsForType.length} {scorerWord}{' '}
                                selected.
                              </p>
                              <div className={styles['ev__metrics-actions']}>
                                <button type="button" className={styles.ev__link} onClick={selectAllBuiltinMetrics}>
                                  Select all
                                </button>
                                <button type="button" className={styles.ev__link} onClick={clearBuiltinMetrics}>
                                  Clear
                                </button>
                              </div>
                              {(metricsCatalog.length > 0 || customMetricsForType.length > 0) &&
                                draft.metrics.length === 0 && (
                                  <p className={styles['ev__frow-req']}>Required: pick at least one</p>
                                )}
                            </>
                          )}
                        </div>
                        <div className={styles['ev__frow-main']}>
                          {hideMetricsStep ? (
                            <p className={styles['ev__inline-note']}>
                              {isAgentBenchmarkNoFramework
                                ? 'Metrics are not configurable without a selected framework. A judge model is still required.'
                                : 'Metrics are not configurable for standard (non-custom) benchmarks. A judge model is still required.'}
                            </p>
                          ) : (
                            <>
                              <div className={styles['ev__step-toolbar']} style={{ marginBottom: 0 }}>
                                <div className={styles['ev__toolbar-search']}>
                                  <Search size={14} />
                                  <input
                                    placeholder={`Search ${scorerWord}`}
                                    value={metricSearch}
                                    onChange={(e) => setMetricSearch(e.target.value)}
                                  />
                                </div>
                                <button
                                  type="button"
                                  className={styles['ev__toolbar-refresh']}
                                  onClick={refreshMetrics}
                                  disabled={metricsRefreshing}
                                  title={`Refresh ${scorerWord}`}
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
                                        aria-pressed={on}
                                        onClick={() => dispatch(setDraft({ metrics: toggle(draft.metrics, name) }))}
                                      >
                                        {name}
                                      </button>
                                    );
                                  })}
                                  {metricsCatalog.length === 0 && (
                                    <p className={styles['ev__inline-note']}>No {scorerWord} available for this type.</p>
                                  )}
                                  {metricsCatalog.length > 0 && filteredMetrics.length === 0 && (
                                    <p className={styles['ev__inline-note']}>Nothing matches "{metricSearch}".</p>
                                  )}
                                </div>
                              )}

                              {/* Custom metrics: stored by id in draft.metrics, alongside
                                  the built-in names. Shown whenever the catalog has any. */}
                              {customMetricsCatalog.length > 0 && (
                                <>
                                  <p className={styles.ev__sublabel}>
                                    Custom
                                    <button type="button" className={styles.ev__link} onClick={selectAllCustomMetrics}>
                                      Select all
                                    </button>
                                    <button type="button" className={styles.ev__link} onClick={clearCustomMetrics}>
                                      Clear
                                    </button>
                                  </p>
                                  <div className={styles.ev__chips}>
                                    {filteredCustomMetrics.map((c) => {
                                      const on = draft.metrics.includes(c.id);
                                      return (
                                        <button
                                          key={c.id}
                                          type="button"
                                          className={`${styles.ev__chip} ${styles['ev__chip--custom']} ${
                                            on ? styles['ev__chip--on'] : ''
                                          }`}
                                          aria-pressed={on}
                                          onClick={() => dispatch(setDraft({ metrics: toggle(draft.metrics, c.id) }))}
                                          title={c.description || c.name}
                                        >
                                          {c.name}
                                          {c.required_judge && <Gavel size={11} className={styles['ev__chip-judge-icon']} />}
                                        </button>
                                      );
                                    })}
                                    {customMetricsForType.length === 0 && (
                                      <p className={styles['ev__inline-note']}>No custom metrics for this type.</p>
                                    )}
                                    {customMetricsForType.length > 0 && filteredCustomMetrics.length === 0 && (
                                      <p className={styles['ev__inline-note']}>Nothing matches "{metricSearch}".</p>
                                    )}
                                  </div>
                                </>
                              )}
                            </>
                          )}
                        </div>
                      </section>

                      {/* Limits per task — harness only */}
                      {isHarness && (
                        <section className={styles.ev__frow}>
                          <div className={styles['ev__frow-side']}>
                            <h3>Limits per task</h3>
                            <p>When the harness stops an agent, and what it keeps afterwards.</p>
                          </div>
                          <div className={styles['ev__frow-main']}>
                            <div className={styles.ev__inline} style={{ alignItems: 'flex-end' }}>
                              <div className={styles.ev__numfield}>
                                <label>Max steps</label>
                                <input
                                  type="number"
                                  className={styles.ev__num}
                                  min={MIN_HARNESS_STEPS}
                                  max={MAX_HARNESS_STEPS}
                                  value={displayNumInput(draft.harnessSettings.max_steps)}
                                  onChange={(e) =>
                                    dispatch(
                                      setDraft({
                                        harnessSettings: { ...draft.harnessSettings, max_steps: parseNumInput(e.target.value) },
                                      })
                                    )
                                  }
                                  onBlur={() =>
                                    dispatch(
                                      setDraft({
                                        harnessSettings: {
                                          ...draft.harnessSettings,
                                          max_steps: clampHarnessSteps(draft.harnessSettings.max_steps),
                                        },
                                      })
                                    )
                                  }
                                />
                              </div>
                              <div className={styles.ev__numfield}>
                                <label>Timeout (seconds)</label>
                                <input
                                  type="number"
                                  className={styles.ev__num}
                                  min={MIN_TIMEOUT}
                                  max={MAX_TIMEOUT}
                                  value={displayNumInput(draft.harnessSettings.task_timeout)}
                                  onChange={(e) =>
                                    dispatch(
                                      setDraft({
                                        harnessSettings: {
                                          ...draft.harnessSettings,
                                          task_timeout: parseNumInput(e.target.value),
                                        },
                                      })
                                    )
                                  }
                                  onBlur={() =>
                                    dispatch(
                                      setDraft({
                                        harnessSettings: {
                                          ...draft.harnessSettings,
                                          task_timeout: clampRetryField('timeout', draft.harnessSettings.task_timeout),
                                        },
                                      })
                                    )
                                  }
                                />
                              </div>
                              <button
                                type="button"
                                role="checkbox"
                                aria-checked={draft.harnessSettings.save_traces}
                                className={styles.ev__checkline}
                                onClick={() =>
                                  dispatch(
                                    setDraft({
                                      harnessSettings: {
                                        ...draft.harnessSettings,
                                        save_traces: !draft.harnessSettings.save_traces,
                                      },
                                    })
                                  )
                                }
                              >
                                <span
                                  className={`${styles.ev__check} ${
                                    draft.harnessSettings.save_traces ? styles['ev__check--on'] : ''
                                  }`}
                                >
                                  {draft.harnessSettings.save_traces && <Check size={11} strokeWidth={3} />}
                                </span>
                                Save traces for replay
                              </button>
                            </div>
                            <p className={styles['ev__inline-note']}>
                              Steps {MIN_HARNESS_STEPS} to {MAX_HARNESS_STEPS}. Timeout {MIN_TIMEOUT} to {MAX_TIMEOUT} seconds.
                            </p>
                          </div>
                        </section>
                      )}

                      {/* Retest on wrong — Model and RAG always; Agent only with a framework */}
                      {retestShown && (
                        <section className={styles.ev__frow}>
                          <div className={styles['ev__frow-side']}>
                            <h3>Retest on wrong</h3>
                            <p>Re-grade a wrong answer before marking it failed.</p>
                            {draft.retestOnWrong && !draft.retestVerifyMetric && (
                              <p className={styles['ev__frow-req']}>Required: pick a verify metric</p>
                            )}
                          </div>
                          <div className={styles['ev__frow-main']}>
                            <div className={styles.ev__inline}>
                              <div className={styles.ev__segctl}>
                                <button
                                  type="button"
                                  className={`${styles.ev__segbtn} ${!draft.retestOnWrong ? styles['ev__segbtn--on'] : ''}`}
                                  onClick={() =>
                                    dispatch(
                                      setDraft({
                                        retestOnWrong: false,
                                        retestMaxRounds: DEFAULT_RETEST_ROUNDS,
                                        retestVerifyMetric: null,
                                      })
                                    )
                                  }
                                >
                                  Off
                                </button>
                                <button
                                  type="button"
                                  className={`${styles.ev__segbtn} ${draft.retestOnWrong ? styles['ev__segbtn--on'] : ''}`}
                                  onClick={() => dispatch(setDraft({ retestOnWrong: true }))}
                                >
                                  On
                                </button>
                              </div>
                              {draft.retestOnWrong && (
                                <>
                                  <input
                                    type="number"
                                    className={styles.ev__num}
                                    min={MIN_RETEST_ROUNDS}
                                    max={MAX_RETEST_ROUNDS}
                                    value={displayNumInput(draft.retestMaxRounds)}
                                    onChange={(e) => updateRetestMaxRounds(parseNumInput(e.target.value))}
                                    onBlur={clampRetestMaxRoundsOnBlur}
                                  />
                                  <span className={styles['ev__inline-note']}>
                                    rounds at most ({MIN_RETEST_ROUNDS} to {MAX_RETEST_ROUNDS})
                                  </span>
                                </>
                              )}
                            </div>
                            {draft.retestOnWrong && (
                              <>
                                <p className={styles.ev__sublabel}>Verify each retest with one metric</p>
                                <div className={styles.ev__chips}>
                                  {metricsCatalog.map((name: string) => {
                                    const on = draft.retestVerifyMetric === name;
                                    return (
                                      <button
                                        key={name}
                                        type="button"
                                        className={`${styles.ev__chip} ${on ? styles['ev__chip--on'] : ''}`}
                                        aria-pressed={on}
                                        onClick={() => dispatch(setDraft({ retestVerifyMetric: on ? null : name }))}
                                      >
                                        {name}
                                      </button>
                                    );
                                  })}
                                  {metricsCatalog.length === 0 && (
                                    <p className={styles['ev__inline-note']}>No metrics available to verify against.</p>
                                  )}
                                </div>
                              </>
                            )}
                          </div>
                        </section>
                      )}

                      {/* Instruction — optional; harness suites carry their own */}
                      {!isHarness && (
                        <section className={styles.ev__frow}>
                          <div className={styles['ev__frow-side']}>
                            <h3>Instruction</h3>
                            <p>Optional. Extra context for how the models should approach this run.</p>
                            <div className={styles['ev__metrics-actions']}>
                              <button
                                type="button"
                                className={styles.ev__link}
                                onClick={generateInstruction}
                                disabled={instructionGenerating || !draft.dataset || draft.models.length === 0}
                                title="Generate a default instruction from a sample of this dataset"
                              >
                                {instructionGenerating ? 'Generating…' : 'Generate a draft'}
                              </button>
                            </div>
                          </div>
                          <div className={styles['ev__frow-main']}>
                            <textarea
                              className={styles['ev__instruction-textarea']}
                              placeholder="Describe how the models should approach this evaluation, or generate a draft to edit."
                              value={draft.instruction}
                              onChange={(e) => dispatch(setDraft({ instruction: e.target.value }))}
                              rows={4}
                            />
                            {instructionError && <p className={styles.ev__error}>{instructionError}</p>}
                          </div>
                        </section>
                      )}
                    </div>

                    {/* Who judges — mandatory for every type. Its own side panel,
                        same pattern as the Models step's "In this run". */}
                    <aside className={`${styles.ev__panel} ${styles['ev__panel--pinned']}`}>
                      {/* Pinned: heading, guidance, required note and search.
                          Only the list of candidate models below scrolls. */}
                      <div className={styles['ev__panel-fixed']}>
                        <div className={styles['ev__panel-headrow']}>
                          <p className={styles['ev__panel-title']}>Who judges</p>
                          {judgeModel ? (
                            <span className={styles['ev__panel-chip']}>
                              <Gavel size={11} /> {judgeModel.name || 'Unnamed model'}
                            </span>
                          ) : (
                            <span className={`${styles['ev__panel-chip']} ${styles['ev__panel-chip--req']}`}>Required</span>
                          )}
                        </div>
                        <p className={styles['ev__panel-note']}>
                          One model grades open-ended answers. Pick one that is not under test if you can.
                        </p>
                        <div className={styles.ev__sbar}>
                          <span className={styles['ev__sbar-icon']}>
                            <Search size={13} strokeWidth={2.6} />
                          </span>
                          <input
                            type="text"
                            placeholder="Search judge models"
                            aria-label="Search judge models"
                            value={judgeSearch}
                            onChange={(e) => setJudgeSearch(e.target.value)}
                          />
                          {judgeSearch && (
                            <>
                              <span
                                className={`${styles['ev__sbar-count']} ${
                                  filteredJudgeModels.length === 0 ? styles['ev__sbar-count--none'] : ''
                                }`}
                              >
                                {filteredJudgeModels.length === 0
                                  ? 'No matches'
                                  : `${filteredJudgeModels.length} result${filteredJudgeModels.length === 1 ? '' : 's'}`}
                              </span>
                              <button
                                type="button"
                                className={styles['ev__sbar-clear']}
                                onClick={() => setJudgeSearch('')}
                                aria-label="Clear search"
                              >
                                <X size={12} strokeWidth={2.5} />
                              </button>
                            </>
                          )}
                        </div>
                      </div>

                      <div className={styles['ev__panel-scroll']}>
                        {judgeCandidateModels.length === 0 ? (
                          <p className={styles['ev__panel-note']}>No healthy models available yet.</p>
                        ) : filteredJudgeModels.length === 0 ? (
                          <p className={styles['ev__panel-note']}>No models match "{judgeSearch}".</p>
                        ) : (
                          filteredJudgeModels.map((m) => {
                            const on = draft.judgeModelId === m.id;
                            const providerName = providers.find((p) => p?.id === m.provider_id)?.name ?? m.provider_id;
                            const underTest = draft.models.includes(m.id);
                            return (
                              <button
                                key={m.id}
                                type="button"
                                className={`${styles.ev__jcard} ${on ? styles['ev__jcard--on'] : ''}`}
                                aria-pressed={on}
                                onClick={() => dispatch(setDraft({ judgeModelId: on ? null : m.id }))}
                              >
                                <span className={styles['ev__jcard-head']}>
                                  <span className={`${styles.ev__radio} ${on ? styles['ev__radio--on'] : ''}`} />
                                  <span className={styles['ev__srow-icon']}>
                                    <Gavel size={14} />
                                  </span>
                                  <span className={styles['ev__jcard-id']}>
                                    <span className={styles['ev__jcard-name']} title={m.name || 'Unnamed model'}>
                                      {m.name || 'Unnamed model'}
                                    </span>
                                    <span className={styles['ev__jcard-provider']}>{providerName}</span>
                                  </span>
                                  {underTest && (
                                    <span
                                      className={styles['ev__jcard-flag']}
                                      title="This model is also being evaluated in this run"
                                    >
                                      Under test
                                    </span>
                                  )}
                                </span>
                                <span className={styles['ev__jcard-stats']}>
                                  <span className={styles.ev__stat}>
                                    <span className={styles['ev__stat-k']}>Context</span>
                                    <span className={styles['ev__stat-v']}>{formatContextWindow(m.context_window)}</span>
                                  </span>
                                  <span className={styles.ev__stat}>
                                    <span className={styles['ev__stat-k']}>Input /1M</span>
                                    <span className={styles['ev__stat-v']}>{formatPrice(m.input_price)}</span>
                                  </span>
                                  <span className={styles.ev__stat}>
                                    <span className={styles['ev__stat-k']}>Output /1M</span>
                                    <span className={styles['ev__stat-v']}>{formatPrice(m.output_price)}</span>
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
                      <div className={styles.ev__banner}>
                        <div style={{ minWidth: 0 }}>
                          <p className={styles['ev__banner-title']}>{draft.name || 'Untitled run'}</p>
                          <p className={styles['ev__banner-sub']}>{reviewSummaryLine}</p>
                        </div>
                        <div className={styles['ev__banner-stats']}>
                          {reviewStats.map((st) => (
                            <div key={st.k}>
                              <span>{st.k}</span>
                              <b>{st.v}</b>
                            </div>
                          ))}
                        </div>
                      </div>

                      {/* Run matrix — every agent runs every selected model on every
                          selected suite; each cell is that agent's task count. */}
                      {isHarness && (
                        <>
                          <div className={styles.ev__sechead}>
                            <h3>Run matrix</h3>
                            <span>
                              Each cell runs on{' '}
                              {selectedModels.map((m) => m.name || 'Unnamed model').join(' and ') || 'no models yet'}
                            </span>
                          </div>
                          <div className={`${styles['ev__table-wrap']} ${styles['ev__table-wrap--static']}`}>
                            <table className={`${styles.ev__table} ${styles['ev__table--static']}`}>
                              <thead>
                                <tr>
                                  <th>Suite</th>
                                  <th>Use case</th>
                                  {harnessAgentTitles.map((t) => (
                                    <th key={t} className={styles['ev__table-num']}>
                                      {t}
                                    </th>
                                  ))}
                                </tr>
                              </thead>
                              <tbody>
                                {selectedHarnessSuites.map((d) => (
                                  <tr key={d.id}>
                                    <td>
                                      <b>{d.name}</b>
                                    </td>
                                    <td>{d.use_case || '—'}</td>
                                    {harnessAgentTitles.map((t) => (
                                      <td key={t} className={styles['ev__table-num']}>
                                        {(suiteQuestions(d.question_count ?? 0) * draft.models.length).toLocaleString()} tasks
                                      </td>
                                    ))}
                                  </tr>
                                ))}
                              </tbody>
                            </table>
                          </div>
                          {uncoveredUseCases.length > 0 && (
                            <p className={styles.ev__secnote}>
                              {uncoveredUseCases.join(', ')} {uncoveredUseCases.length === 1 ? 'is' : 'are'} not covered by
                              this run.
                            </p>
                          )}
                        </>
                      )}

                      <div className={styles.ev__sechead}>
                        <h3>Settings</h3>
                      </div>
                      <div className={styles.ev__settings}>
                        {reviewRows.map((r) => (
                          <div key={r.k} className={styles['ev__settings-row']}>
                            <span className={styles['ev__settings-k']}>{r.k}</span>
                            <span className={styles['ev__settings-v']}>{r.v}</span>
                            <button type="button" className={styles.ev__link} onClick={() => goToStep(r.step)} disabled={launching}>
                              Edit
                            </button>
                          </div>
                        ))}
                      </div>

                      {launchError && (
                        <p className={styles.ev__error} ref={launchErrorRef}>
                          {launchError}
                        </p>
                      )}
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
                  {step === 0 ? 'Cancel' : 'Back'}
                </button>

                <div style={{ display: 'flex', alignItems: 'center', gap: 16 }}>
                  {step === 0 && canGo() && (
                    <span className={styles.ev__hint}>
                      <kbd>↵</kbd> Enter to continue
                    </span>
                  )}
                  {step < totalSteps - 1 ? (
                    <>
                      {blocker && <span className={styles.ev__blocker}>{blocker}</span>}
                      <button
                        type="button"
                        className={`${styles.ev__btn} ${styles['ev__btn--primary']}`}
                        onClick={goNext}
                        disabled={!canGo()}
                      >
                        Next
                      </button>
                    </>
                  ) : (
                    <>
                      <span className={styles.ev__blocker}>You can follow progress in History after launch.</span>
                      <button
                        type="button"
                        className={`${styles.ev__btn} ${styles['ev__btn--launch']}`}
                        onClick={launch}
                        disabled={launching}
                      >
                        {launching ? (
                          <>
                            <Loader2 size={16} className={styles.ev__spin} /> Launching…
                          </>
                        ) : (
                          'Launch run'
                        )}
                      </button>
                    </>
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
                        const inputLanguage = q?.input?.language;
                        const expectedAnswer = q?.expected?.answer;
                        const expectedDocId = q?.expected?.doc_id;
                        const expectedSectionId = q?.expected?.section_id;
                        const metadataEntries =
                          q?.metadata && typeof q.metadata === 'object' ? flattenMetadata(q.metadata) : [];
                        const questionImages = Array.isArray(q?.input?.images) ? (q.input!.images as string[]) : [];

                        return (
                          <div key={q?.id ?? i} className={styles['ev-preview-q']}>
                            <div className={styles['ev-preview-q-head']}>
                              <span className={styles['ev-preview-q-index']}>Q{previewOffset + i + 1}</span>
                              {q?.category && <span className={styles['ev-preview-q-cat']}>{q.category}</span>}
                              {q?.subgroup && <span className={styles['ev-preview-q-cat']}>{q.subgroup}</span>}
                              {inputType && <span className={styles['ev-preview-q-cat']}>{String(inputType)}</span>}
                              {inputLanguage && <span className={styles['ev-preview-q-cat']}>{String(inputLanguage)}</span>}
                              {q?.question_type && <span className={styles['ev-preview-q-cat']}>{String(q.question_type)}</span>}
                            </div>

                            {promptText != null && (
                              <p className={styles['ev-preview-q-prompt']}>{String(promptText)}</p>
                            )}
                            {inputSource != null && (
                              <p className={styles['ev-preview-q-source']}>
                                <span>Source</span> {String(inputSource)}
                              </p>
                            )}

                            {questionImages.length > 0 && (
                              <div className={styles['ev-preview-q-images']}>
                                {questionImages.map((imgPath, imgIdx) => (
                                  <button
                                    key={`${imgPath}-${imgIdx}`}
                                    type="button"
                                    className={styles['ev-preview-q-image-btn']}
                                    onClick={() => openImagePreview(questionImages, imgIdx)}
                                    title={imgPath}
                                  >
                                    <ImageIcon size={13} />
                                    {imgPath.split('/').pop() || `Image ${imgIdx + 1}`}
                                  </button>
                                ))}
                              </div>
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

      {/* Image lightbox — resolves a MinIO object key from a preview
          question's input.images[] to a blob URL on demand and shows it
          full-size, mirroring MinIO's own object-preview panel: dark
          overlay, centered image, filename header, close button, and
          prev/next navigation when a question has more than one image. */}
      {imagePreviewOpen && (
        <div className={styles['ev-imgview-overlay']} onClick={closeImagePreview}>
          <div
            className={styles['ev-imgview-panel']}
            onClick={(e) => e.stopPropagation()}
            role="dialog"
            aria-modal="true"
            aria-label="Image preview"
          >
            <div className={styles['ev-imgview-head']}>
              <span className={styles['ev-imgview-path']} title={imagePreviewPaths[imagePreviewIndex]}>
                <ImageIcon size={14} />
                {imagePreviewPaths[imagePreviewIndex]?.split('/').pop() || 'Image'}
              </span>
              {imagePreviewPaths.length > 1 && (
                <span className={styles['ev-imgview-count']}>
                  {imagePreviewIndex + 1} / {imagePreviewPaths.length}
                </span>
              )}
              <button type="button" className={styles['ev-imgview-close']} onClick={closeImagePreview} title="Close">
                <X size={16} />
              </button>
            </div>

            <div className={styles['ev-imgview-body']}>
              {imagePreviewPaths.length > 1 && (
                <button
                  type="button"
                  className={`${styles['ev-imgview-nav']} ${styles['ev-imgview-nav--prev']}`}
                  onClick={showPrevImage}
                  disabled={imagePreviewIndex === 0}
                  title="Previous image"
                >
                  <ChevronLeft size={18} />
                </button>
              )}

              {imagePreviewLoading && (
                <div className={styles['ev-imgview-status']}>
                  <Loader2 size={22} className={styles.ev__spin} />
                  <span>Loading image…</span>
                </div>
              )}

              {!imagePreviewLoading && imagePreviewError && (
                <div className={styles['ev-imgview-status']}>
                  <p className={styles.ev__error}>{imagePreviewError}</p>
                </div>
              )}

              {!imagePreviewLoading && !imagePreviewError && imagePreviewUrl && (
                <img
                  src={imagePreviewUrl}
                  alt={imagePreviewPaths[imagePreviewIndex] || 'Question image'}
                  className={styles['ev-imgview-img']}
                />
              )}

              {imagePreviewPaths.length > 1 && (
                <button
                  type="button"
                  className={`${styles['ev-imgview-nav']} ${styles['ev-imgview-nav--next']}`}
                  onClick={showNextImage}
                  disabled={imagePreviewIndex === imagePreviewPaths.length - 1}
                  title="Next image"
                >
                  <ChevronRight size={18} />
                </button>
              )}
            </div>
          </div>
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





























@use '../../styles/_variables' as *;

// ===========================================================================
// SemcoEval — Run Console
// A precision "instrument panel" for assembling and launching an evaluation.
// Signature: a live Run Manifest (mono spec sheet) threaded by a signal rail.
// Header matches the History/Reports/Comparison/Sidebar design standard.
//
// All color tokens ($ink, $paper, $signal, $ok, $danger, $violet-ink,
// $ink-solid, etc.) come from the shared "ink" design system in
// ../../styles/_variables.scss (imported above) — this file no longer
// redeclares them locally. Neutrals resolve to theme CSS vars (see
// _theme.scss) for dark-mode support; $ink-solid/$ink-solid-hover are a
// FIXED near-black used only for "always dark" chips/buttons (option
// icons, the Continue button, the launch toast) — using themed $ink
// there would turn them near-white (and invisible) in dark mode. See the
// note above those two tokens in _theme.scss for why they're still CSS
// vars despite not actually varying by theme.
// ===========================================================================

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

$soft:  0 1px 2px rgba(20, 22, 27, 0.05);
$lift:  0 14px 30px -14px rgba(20, 22, 27, 0.22);
$ring:  0 0 0 3px rgba(43, 43, 245, 0.16);

// Base font-size every `em` font-size below is expressed relative to — same
// convention as Model Catalog / Sidebar. Bumping this (e.g. on wide
// screens, see the `@media (min-width: 1800px)` rules below) scales every
// descendant font-size proportionally from one place. This value has four
// independent application points since the component renders four separate
// DOM subtrees (.ev__header, .page → .ev, .ev-toast, .ev-preview-overlay)
// rather than one single wrapper — each sets `font-size: $ev-base-font;`
// itself so inheritance covers everything nested inside it.
$ev-base-font: 0.8125rem;

%micro {
  font-family: $mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

// ===========================================================================
// Header — matches History / Reports / Comparison / Sidebar header pattern
// ===========================================================================
.ev__header {
  // master scale control for this subtree — see $ev-base-font above
  font-size: $ev-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  flex-shrink: 0;
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
  gap: 1rem;
  padding: 24px 32px 20px;
  border-bottom: 1px solid $line;
  background: $card;

  h1 {
    font-family: $display;
    font-size: 1.8462em; // 1.5rem / 0.8125rem
    font-weight: 800;
    letter-spacing: -0.02em;
    color: $ink;
    line-height: 1.2;
  }
}

.ev__header-eyebrow {
  @extend %micro;
  display: flex;
  align-items: center;
  gap: 8px;
  color: $signal;
  margin-bottom: 6px;

  &::before {
    content: '';
    width: 16px;
    height: 2px;
    border-radius: 2px;
    background: $signal;
  }
}

.ev__header-sub {
  margin-top: 4px;
  font-size: 1.0385em; color: $ink-2; } // 0.84375rem / 0.8125rem

.ev__header-meta {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 3px;
}

.ev__header-status {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  padding: 7px 13px;
  border-radius: 999px;
  border: 1px solid $line;
  background: $paper;
  font-family: $mono;
  font-size: 0.8846em; // 0.71875rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.06em;
  text-transform: uppercase;
  color: $ink-2;
  white-space: nowrap;

  &::before {
    content: '';
    width: 6px;
    height: 6px;
    border-radius: 50%;
    background: $ink-3;
  }

  &[data-state='draft']::before { background: $signal; box-shadow: 0 0 0 3px $wash; }
  &[data-state='live'] { color: $signal; border-color: rgba($signal, 0.35); background: $wash; }
  &[data-state='live']::before { background: $signal; animation: ev-pulse 1.1s ease-in-out infinite; }
}

.ev__header-eta {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  font-size: 0.9231em; // 0.75rem / 0.8125rem
  font-weight: 600;
  color: $ink-3;
  white-space: nowrap;
}

.page {
  // master scale control for this subtree — cascades down through .ev and
  // every nested &__ selector below, since none of them reset font-size
  // themselves (they're all expressed in em relative to whatever ancestor
  // font-size is in effect). See $ev-base-font above.
  font-size: $ev-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  flex: 1;
  height: 100%;
  min-height: 0;
  padding: 22px 30px 26px;
  display: flex;
  flex-direction: column;
  background: $paper;
}

// ===========================================================================
// Root
// ===========================================================================
.ev {
  flex: 1;
  min-height: 0;
  display: flex;
  flex-direction: column;

  // ---- shell (manifest + stage) ------------------------------------------
  &__shell {
    flex: 1;
    min-height: 0;
    display: grid;
    grid-template-columns: 288px 1fr;
    gap: 16px;
  }

  // ========================================================================
  // SIGNATURE: Run Manifest
  // ========================================================================
  &__manifest {
    background: $card;
    border: 1px solid $line;
    border-radius: 16px;
    box-shadow: $soft;
    display: flex;
    flex-direction: column;
    min-height: 0;
    overflow: hidden;
  }

  &__manifest-head {
    flex-shrink: 0;
    padding: 18px 20px 16px;
    border-bottom: 1px solid $line-2;
  }

  &__manifest-eyebrow {
    @extend %micro;
    color: $ink-3;
    display: flex;
    align-items: center;
    justify-content: space-between;
  }

  &__manifest-pct {
    color: $signal;
    font-size: 0.9231em; letter-spacing: 0.06em; } // 0.75rem / 0.8125rem

  &__manifest-title {
    margin-top: 8px;
    font-family: $display;
    font-size: 1.2308em; // 1rem / 0.8125rem
    font-weight: 800;
    letter-spacing: -0.015em;
    color: $ink;
    line-height: 1.15;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;

    &[data-empty='true'] { color: $ink-3; font-style: normal; }
  }

  &__meter {
    margin-top: 12px;
    height: 4px;
    border-radius: 999px;
    background: $line;
    overflow: hidden;
  }

  &__meter-fill {
    height: 100%;
    border-radius: 999px;
    background: linear-gradient(90deg, $signal, $signal-2);
    transition: width 0.4s cubic-bezier(0.32, 0.72, 0, 1);
  }

  // ---- the spec list (each row = a step, with its live value) -------------
  &__spec {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding: 14px 12px 16px;
    display: flex;
    flex-direction: column;
    gap: 24px;
  }

  &__spec-row {
    position: relative;
    display: grid;
    grid-template-columns: 30px 1fr;
    align-items: start;
    gap: 12px;
    width: 100%;
    text-align: left;
    border: 0;
    background: transparent;
    padding: 12px 12px 12px 4px;
    border-radius: 12px;
    cursor: pointer;
    transition: background 0.15s ease;

    // Extends past this row's own bottom edge by exactly the flex `gap`
    // above (24px), so the line still reaches the next row's tick — the
    // larger the gap, the more this needs to stretch to stay unbroken.
    &::before {
      content: '';
      position: absolute;
      left: 18px;
      top: 38px;
      bottom: -24px;
      width: 2px;
      background: $line;
      transition: background 0.2s ease;
    }
    &:last-child::before { display: none; }

    &:disabled { cursor: default; }
    &:not(:disabled):hover { background: $paper; }
  }

  &__spec-tick {
    position: relative;
    z-index: 1;
    width: 28px;
    height: 28px;
    border-radius: 9px;
    display: grid;
    place-items: center;
    background: $card;
    border: 1.5px solid $line;
    color: $ink-3;
    font-family: $mono;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    font-weight: 700;
    transition: all 0.18s ease;
  }

  &__spec-body {
    min-width: 0;
    padding-top: 1px;
    display: flex;
    flex-direction: column;
    gap: 2px;
  }

  &__spec-label {
    @extend %micro;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    color: $ink-3;
    transition: color 0.18s ease;
  }

  &__spec-value {
    font-size: 1.1538em; // 0.9375rem / 0.8125rem
    font-weight: 600;
    color: $ink;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;

    &[data-empty='true'] {
      color: $ink-3;
      font-weight: 500;
      font-family: $mono;
    }
  }

  &__spec-row--done {
    &::before { background: $signal; }
    .ev__spec-tick { background: $signal; border-color: $signal; color: #fff; }
    .ev__spec-label { color: $ink-3; }
  }

  &__spec-row--active {
    background: $wash;
    .ev__spec-tick {
      background: $card;
      border-color: $signal;
      color: $signal;
      box-shadow: 0 0 0 4px rgba($signal, 0.14);
    }
    .ev__spec-label { color: $signal; }
    &:not(:disabled):hover { background: $wash; }
  }

  &__spec-row--todo { opacity: 0.9; }

  // ========================================================================
  // Stage (the working area for the current step)
  // ========================================================================
  &__stage {
    background: $card;
    border: 1px solid $line;
    border-radius: 16px;
    box-shadow: $soft;
    display: flex;
    flex-direction: column;
    min-height: 0;
    overflow: hidden;
  }

  &__stage-head {
    flex-shrink: 0;
    padding: 22px 28px 18px;
    border-bottom: 1px solid $line-2;
  }

  &__crumb {
    display: flex;
    align-items: center;
    gap: 9px;
    @extend %micro;
    color: $ink-3;

    b { color: $signal; font-weight: 700; }

    span:first-child {
      display: inline-flex;
      align-items: center;
      gap: 7px;
      color: $signal;
    }
  }

  &__crumb-sep {
    width: 3px;
    height: 3px;
    border-radius: 50%;
    background: $ink-3;
  }

  &__stage-title {
    margin-top: 12px;
    font-family: $display;
    font-size: 1.6923em; // 1.375rem / 0.8125rem
    font-weight: 800;
    letter-spacing: -0.025em;
    color: $ink;
    line-height: 1.1;
  }

  &__stage-sub {
    margin-top: 6px;
    font-size: 1.0385em; // 0.84375rem / 0.8125rem
    color: $ink-2;
    line-height: 1.5;
    max-width: 60ch;
  }

  &__stage-body {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding: 22px 28px 26px;
    display: flex;
    flex-direction: column;
  }

  &__anim {
    animation: ev-rise 0.34s cubic-bezier(0.22, 0.72, 0.16, 1) both;
  }

  // ---- footer nav ---------------------------------------------------------
  &__footer {
    flex-shrink: 0;
    padding: 16px 28px;
    border-top: 1px solid $line-2;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
  }

  &__hint {
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    color: $ink-3;
    display: flex;
    align-items: center;
    gap: 7px;
    min-width: 0;

    kbd {
      font-family: $mono;
      font-size: 0.8462em; // 0.6875rem / 0.8125rem
      color: $ink-2;
      background: $paper;
      border: 1px solid $line;
      border-bottom-width: 2px;
      border-radius: 5px;
      padding: 1px 6px;
    }
  }

  &__btn {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    font-family: $sans;
    font-size: 1.0385em; // 0.84375rem / 0.8125rem
    font-weight: 650;
    border-radius: 10px;
    padding: 10px 16px;
    cursor: pointer;
    border: 1px solid transparent;
    transition: background 0.16s ease, border-color 0.16s ease, color 0.16s ease, box-shadow 0.16s ease, transform 0.12s ease;

    &:disabled { cursor: not-allowed; opacity: 0.5; }

    &--ghost {
      background: transparent;
      border-color: $line;
      color: $ink-2;
      &:not(:disabled):hover { border-color: $ink-3; color: $ink; background: $paper; }
    }

    // fixed-dark chip — do NOT switch to $ink here, it would go near-white
    // (and invisible) in dark mode since $ink is theme-aware.
    &--primary {
      background: $ink-solid;
      color: #fff;
      box-shadow: $soft;
      &:not(:disabled):hover { background: $ink-solid-hover; transform: translateY(-1px); box-shadow: $lift; }
    }

    &--launch {
      background: $signal;
      color: #fff;
      box-shadow: 0 8px 20px -8px rgba($signal, 0.7);
      &:not(:disabled):hover { background: $signal-2; transform: translateY(-1px); }
    }
  }

  // ========================================================================
  // Shared field primitives
  // ========================================================================
  &__field {
    max-width: 620px;

    & + & { margin-top: 20px; }
  }

  &__label {
    display: flex;
    align-items: center;
    gap: 7px;
    @extend %micro;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    color: $ink-2;
    margin-bottom: 9px;

    .opt {
      font-family: $sans;
      letter-spacing: 0;
      text-transform: none;
      font-weight: 500;
      font-size: 0.9231em; color: $ink-3; } // 0.75rem / 0.8125rem
  }

  &__input {
    width: 100%;
    border: 1.5px solid $line;
    border-radius: 11px;
    padding: 12px 14px;
    font-size: 1.1538em; // 0.9375rem / 0.8125rem
    font-weight: 500;
    font-family: $sans;
    color: $ink;
    background: $card;
    transition: border-color 0.15s ease, box-shadow 0.15s ease;

    &::placeholder { color: $ink-3; font-weight: 400; }
    &:focus { outline: none; border-color: $signal; box-shadow: $ring; }
    &:disabled { background: $paper; color: $ink-2; }
  }

  &__input-wrap {
    position: relative;
    svg {
      position: absolute;
      top: 50%;
      left: 15px;
      transform: translateY(-50%);
      color: $ink-3;
      pointer-events: none;
    }
    input { padding-left: 42px; }
  }

  // ---- big "name your run" input -----------------------------------------
  &__name-input {
    width: 100%;
    border: 0;
    border-bottom: 2px solid $line;
    border-radius: 0;
    padding: 8px 2px 12px;
    background: transparent;
    font-family: $display;
    font-size: 1.6923em; // 1.375rem / 0.8125rem
    font-weight: 800;
    letter-spacing: -0.03em;
    color: $ink;
    transition: border-color 0.16s ease;

    &::placeholder { color: $ink-3; font-weight: 700; }
    &:focus { outline: none; border-color: $signal; }
  }

  &__name-caption {
    margin-top: 10px;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    color: $ink-3;
    display: flex;
    align-items: center;
    gap: 7px;
  }

  // ---- quick-start presets (mono chips) ----------------------------------
  &__quick {
    margin-top: 30px;
    max-width: 620px;
  }

  &__quick-head {
    @extend %micro;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    color: $ink-3;
    margin-bottom: 11px;
  }

  &__quick-row {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
  }

  &__preset {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    padding: 8px 12px 8px 11px;
    border: 1px solid $line;
    border-radius: 999px;
    background: $card;
    cursor: pointer;
    font-family: $mono;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 600;
    color: $ink-2;
    transition: all 0.15s ease;

    svg { color: $ink-3; transition: color 0.15s ease; }

    &:hover {
      border-color: $ink;
      color: $ink;
      transform: translateY(-1px);
      svg { color: $signal; }
    }

    &--on {
      border-color: $signal;
      background: $wash;
      color: $signal;
      svg { color: $signal; }
    }
  }

  // ---- tips note ----------------------------------------------------------
  &__note {
    margin-top: 28px;
    max-width: 620px;
    display: flex;
    gap: 12px;
    padding: 14px 16px;
    border: 1px solid $line;
    border-left: 2.5px solid $signal;
    border-radius: 12px;
    background: $card;
  }

  &__note-icon {
    flex-shrink: 0;
    color: $signal;
    margin-top: 1px;
  }

  &__note-title {
    font-size: 1em; // 0.8125rem / 0.8125rem (base)
    font-weight: 700;
    color: $ink;
    margin-bottom: 6px;
  }

  &__note-list {
    display: flex;
    flex-direction: column;
    gap: 5px;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    color: $ink-2;
    line-height: 1.5;

    li { display: flex; gap: 8px; }
    li::before {
      content: '—';
      color: $signal;
      flex-shrink: 0;
    }
  }

  // ========================================================================
  // Option rows (Type step) & framework
  // ========================================================================
  &__options {
    display: flex;
    gap: 10px;
  }

  &__option {
    position: relative;
    display: flex;
    align-items: center;
    gap: 15px;
    width: 100%;
    text-align: left;
    padding: 16px 52px 16px 16px;
    border: 1.5px solid $line;
    border-radius: 14px;
    background: $card;
    cursor: pointer;
    transition: border-color 0.18s ease, box-shadow 0.18s ease, transform 0.18s ease, background 0.18s ease;

    &:hover {
      border-color: $ink-3;
      box-shadow: $lift;
      transform: translateY(-2px);
    }

    &--on {
      border-color: $signal;
      background: $wash;
      &:hover { border-color: $signal; }
    }

    &--off {
      opacity: 0.55;
      cursor: not-allowed;
      &:hover { border-color: $line; box-shadow: none; transform: none; }
    }
  }

  // fixed-dark chip — icon glyph is always white-on-dark regardless of theme
  &__option-icon {
    flex-shrink: 0;
    width: 48px;
    height: 48px;
    border-radius: 13px;
    display: grid;
    place-items: center;
    background: $ink-solid;
    color: #fff;
    position: relative;
    overflow: hidden;

    &::after {
      content: '';
      position: absolute;
      inset: 0;
      background: linear-gradient(140deg, transparent 45%, rgba(255,255,255,0.16) 140%);
    }
    svg { position: relative; z-index: 1; }
  }
  &__option-icon--agent { background: $violet-ink; }
  &__option-icon--rag   { background: $sky-ink; }

  &__option-main {
    flex: 1;
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 4px;
  }

  &__option-name {
    display: flex;
    align-items: center;
    gap: 9px;
    font-family: $display;
    font-size: 1.3846em; // 1.125rem / 0.8125rem
    font-weight: 700;
    color: $ink;
  }

  &__badge {
    @extend %micro;
    font-size: 0.6923em; // 0.5625rem / 0.8125rem
    color: $ink-3;
    background: $paper;
    border: 1px solid $line;
    border-radius: 999px;
    padding: 2px 8px;
  }

  &__option-desc {
    font-size: 1.1538em; // 0.9375rem / 0.8125rem
    color: $ink-2;
    line-height: 1.5;
  }

  // selection marker (shared)
  &__mark {
    position: absolute;
    top: 50%;
    right: 16px;
    transform: translateY(-50%);
    width: 22px;
    height: 22px;
    border-radius: 50%;
    display: grid;
    place-items: center;
    background: $signal;
    color: #fff;
    box-shadow: 0 2px 6px rgba($signal, 0.4);
  }

  &__section {
    margin-top: 26px;
    padding-top: 22px;
    border-top: 1px solid $line-2;
    max-width: 720px;
  }

  &__section-hint {
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    color: $ink-3;
    margin: 4px 0 14px;
  }

  &__fw-grid {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 10px;
  }

  &__fw {
    position: relative;
    display: flex;
    align-items: center;
    gap: 12px;
    text-align: left;
    padding: 13px 42px 13px 13px;
    border: 1.5px solid $line;
    border-radius: 12px;
    background: $card;
    cursor: pointer;
    transition: border-color 0.16s ease, box-shadow 0.16s ease, transform 0.16s ease, background 0.16s ease;

    &:hover { border-color: $ink-3; transform: translateY(-2px); box-shadow: $lift; }
    &--on { border-color: $signal; background: $wash; }
  }

  &__fw-icon {
    flex-shrink: 0;
    width: 36px;
    height: 36px;
    border-radius: 10px;
    display: grid;
    place-items: center;
    background: $wash;
    color: $signal;
  }

  &__fw-name { font-family: $display; font-size: 1.1538em; font-weight: 700; color: $ink; } // 0.9375rem / 0.8125rem
  &__fw-desc { font-size: 1em; color: $ink-2; margin-top: 2px; line-height: 1.4; } // 0.8125rem / 0.8125rem (base)

  // ========================================================================
  // Per-step toolbar: search + refresh
  // Used on the Providers, Models, Test Suite (browse tab), and Metrics
  // steps. Search filters the already-fetched list client-side; refresh
  // re-dispatches that step's existing fetch thunk.
  // ========================================================================
  &__step-toolbar {
    display: flex;
    align-items: center;
    gap: 10px;
    margin-bottom: 14px;
  }

  &__toolbar-search {
    flex: 1;
    min-width: 0;
    max-width: 320px;
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 8px 12px;
    border: 1.5px solid $line;
    border-radius: 10px;
    background: $card;
    color: $ink-3;
    transition: border-color 0.15s ease, box-shadow 0.15s ease, color 0.15s ease;

    &:focus-within {
      border-color: $signal;
      box-shadow: $ring;
      color: $signal;
    }

    svg { flex-shrink: 0; }

    input {
      flex: 1;
      min-width: 0;
      border: 0;
      background: transparent;
      font-family: $sans;
      font-size: 1em; // 0.8125rem / 0.8125rem (base)
      font-weight: 500;
      color: $ink;
      outline: none;

      &::placeholder { color: $ink-3; font-weight: 400; }
    }
  }

  &__toolbar-refresh {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 34px;
    height: 34px;
    border: 1.5px solid $line;
    border-radius: 10px;
    background: $card;
    color: $ink-2;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover:not(:disabled) { border-color: $signal; color: $signal; background: $wash; }
    &:disabled { cursor: default; color: $signal; }
  }

  // ---- Test Suite: All / Custom / Deepeval filter (Change-1) -------------
  &__filter-row {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    margin-bottom: 14px;
  }

  &__filter-chip {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    padding: 7px 8px 7px 13px;
    border: 1.5px solid $line;
    border-radius: 999px;
    background: $card;
    color: $ink-2;
    font-family: $sans;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $ink-3; color: $ink; }

    &--on {
      border-color: $signal;
      background: $wash;
      color: $signal;
    }
  }

  &__filter-chip-count {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    min-width: 20px;
    height: 20px;
    padding: 0 6px;
    border-radius: 999px;
    background: $paper;
    color: $ink-3;
    font-family: $mono;
    font-size: 0.8077em; font-weight: 700; } // 0.65625rem / 0.8125rem
  &__filter-chip--on &__filter-chip-count {
    background: $signal;
    color: #fff;
  }

  // ========================================================================
  // Skeleton loaders (Change-3) — shown in place of a step's grid/chips
  // while its "refresh" request is in flight. Shapes loosely echo each
  // step's real card so the layout doesn't jump when data arrives.
  // ========================================================================
  &__skel-block {
    display: block;
    border-radius: 6px;
    background: linear-gradient(90deg, $line 25%, $line-2 37%, $line 63%);
    background-size: 400% 100%;
    animation: ev-shimmer 1.4s ease infinite;

    &--icon { width: 40px; height: 40px; border-radius: 11px; flex-shrink: 0; }
    &--icon-sm { width: 36px; height: 36px; border-radius: 10px; flex-shrink: 0; }
    &--line { height: 11px; border-radius: 5px; }
    &--pill { width: 72px; height: 16px; border-radius: 999px; margin-top: 2px; }
    &--tag { width: 54px; height: 18px; border-radius: 6px; }
    &--stat { width: 58px; height: 26px; border-radius: 7px; }
    &--chip { height: 34px; border-radius: 999px; }
  }

  &__skel-pcard {
    display: flex;
    align-items: flex-start;
    gap: 13px;
    padding: 15px 42px 15px 15px;
    border: 1.5px solid $line;
    border-radius: 14px;
    background: $card;
  }
  &__skel-lines {
    flex: 1;
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  &__skel-mcard {
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 15px 16px;
    border: 1.5px solid $line;
    border-radius: 14px;
    background: $card;
  }
  &__skel-caps { display: flex; gap: 6px; }
  &__skel-stats {
    display: flex;
    gap: 10px;
    padding-top: 10px;
    border-top: 1px solid $line-2;
  }

  &__skel-dcard {
    display: flex;
    flex-direction: column;
    gap: 12px;
    padding: 15px 16px;
    border: 1.5px solid $line;
    border-radius: 14px;
    background: $card;
  }
  &__skel-dcard-top {
    display: flex;
    align-items: center;
    gap: 11px;
  }

  // ========================================================================
  // Card grids (providers / models / datasets)
  // ========================================================================
  &__scroll {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    margin: 0 -6px;
    padding: 4px 6px;
  }

  // ---- Models step (Step 3): search toolbar + Retries & timeout panel
  // stay fixed; only the card grid below them scrolls. -------------------
  &__models-panel {
    flex: 1;
    min-height: 0;
    display: flex;
    flex-direction: column;
  }

  &__models-scroll {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    margin: 8px -6px 0;
    padding: 4px 6px;
  }

  // ---- RAG Models step: one column per model category, single-select
  // within each column (see toggleModelForCategory). --------------------
  &__rag-columns {
    display: flex;
    align-items: stretch;
    gap: 14px;
    height: 100%;
    min-height: 0;
    overflow-x: auto;
  }

  &__rag-col {
    // No fixed width — each column grows to share the available space
    // equally; min-width just keeps a column from getting crushed if
    // there are many categories (overflow-x on &__rag-columns takes over
    // once they can't all fit at that minimum).
    flex: 1 1 0;
    min-width: 240px;
    min-height: 0;
    display: flex;
    flex-direction: column;
    border: 1px solid $line;
    border-radius: 14px;
    background: $paper;
    overflow: hidden;
  }

  &__rag-col-head {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
    padding: 12px 14px;
    border-bottom: 1px solid $line;
    background: $card;
  }

  // Per-category search box (Embedding/Reranker/LLM each get their own,
  // independent of the others and of the Model/Agent grid's shared
  // search).
  &__rag-col-search {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 7px;
    margin: 10px 12px 0;
    padding: 6px 10px;
    border: 1px solid $line;
    border-radius: 9px;
    background: $card;

    svg { flex-shrink: 0; color: $ink-3; }

    input {
      flex: 1;
      min-width: 0;
      border: 0;
      background: transparent;
      font-family: $sans;
      font-size: 0.8846em; // 0.71875rem / 0.8125rem
      color: $ink;

      &:focus { outline: none; }
      &::placeholder { color: $ink-3; }
    }
  }

  &__rag-col-title {
    min-width: 0;
    font-family: $display;
    font-size: 1.0em;
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $ink;
    overflow: hidden;
    white-space: nowrap;
    text-overflow: ellipsis;
  }

  &__rag-col-status {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 4px;
    padding: 3px 8px;
    border-radius: 999px;
    background: $wash;
    color: $ink-3;
    font-family: $sans;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.03em;
  }
  &__rag-col-status--on { background: rgba($ok, 0.14); color: $ok; }

  // Own scrollbar per category — independent of the other columns and of
  // the page's own scroll, so a long list in one category never pushes
  // another category's cards out of view.
  &__rag-col-list {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 12px;

    // Cards inside a RAG column stack in a single column instead of
    // wrapping into a multi-column grid like &__grid does.
    .ev__mcard { width: 100%; }
  }

  &__grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(258px, 1fr));
    gap: 12px;

    @media (min-width: 1800px) {
      grid-template-columns: repeat(auto-fill, minmax(310px, 1fr));
    }
  }

  &__grid--wide {
    grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  }

  // ---- provider card ------------------------------------------------------
  &__pcard {
    position: relative;
    display: flex;
    align-items: flex-start;
    gap: 13px;
    text-align: left;
    padding: 15px 42px 15px 15px;
    border: 1.5px solid $line;
    border-radius: 14px;
    background: $card;
    cursor: pointer;
    transition: border-color 0.16s ease, box-shadow 0.16s ease, transform 0.16s ease, background 0.16s ease;

    &:hover { border-color: $ink-3; box-shadow: $lift; transform: translateY(-2px); }
    &--on { border-color: $signal; background: $wash; &:hover { border-color: $signal; } }
  }

  &__pcard-icon {
    flex-shrink: 0;
    width: 40px;
    height: 40px;
    border-radius: 11px;
    display: grid;
    place-items: center;
    background: $paper;
    border: 1px solid $line;
    color: $ink;
    font-family: $display;
    font-weight: 800;
    font-size: 1.2308em; transition: all 0.16s ease; } // 1rem / 0.8125rem
  &__pcard--on &__pcard-icon { background: $signal; border-color: $signal; color: #fff; }

  &__pcard-body { min-width: 0; display: flex; flex-direction: column; gap: 3px; }
  &__pcard-name { font-family: $display; font-size: 1.3846em; font-weight: 700; color: $ink; } // 1.125rem / 0.8125rem
  &__pcard-meta { font-size: 1.1538em; color: $ink-3; } // 0.9375rem / 0.8125rem

  &__pill {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    margin-top: 4px;
    width: fit-content;
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ok;
    background: $ok-wash;
    border-radius: 999px;
    padding: 3px 8px 3px 6px;

    &::before { content: ''; width: 5px; height: 5px; border-radius: 50%; background: $ok; }
  }

  // ---- model card ---------------------------------------------------------
  &__mcard {
    position: relative;
    display: flex;
    flex-direction: column;
    gap: 10px;
    text-align: left;
    padding: 15px 16px;
    border: 1.5px solid $line;
    border-radius: 14px;
    background: $card;
    cursor: pointer;
    transition: border-color 0.16s ease, box-shadow 0.16s ease, transform 0.16s ease, background 0.16s ease;

    &:hover { border-color: $ink-3; box-shadow: $lift; transform: translateY(-2px); }
    &--on { border-color: $signal; background: $wash; &:hover { border-color: $signal; } }
  }

  // locked = provider chosen but health not yet confirmed successful;
  // dims the interaction affordance so it doesn't read as clickable-to-select.
  &__mcard--locked {
    cursor: default;
    &:hover { border-color: $line; box-shadow: none; transform: none; }
  }

  // checking = a health check (auto or manual) is currently in flight for
  // this card — a soft pulsing border + sheen so it reads as "in progress"
  // at a glance, on top of the "Checking…" badge text.
  &__mcard--checking {
    position: relative;
    overflow: hidden;
    border-color: rgba($signal, 0.35);
    animation: ev-mcard-pulse 1.6s ease-in-out infinite;

    &::after {
      content: '';
      position: absolute;
      inset: 0;
      background: linear-gradient(100deg, transparent 30%, rgba($signal, 0.08) 50%, transparent 70%);
      background-size: 200% 100%;
      animation: ev-mcard-sheen 1.6s ease-in-out infinite;
      pointer-events: none;
    }
  }

  &__mcard-top {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 10px;
  }

  &__mcard-name { font-family: $display; font-size: 1.1154em; font-weight: 700; color: $ink; line-height: 1.25; } // 0.90625rem / 0.8125rem
  &__mcard-provider { font-size: 0.8846em; color: $ink-3; } // 0.71875rem / 0.8125rem

  // ---- provider row + manual health check (Step 3) -------------------------
  &__mcard-provider-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
    margin-top: 2px;
  }

  &__mcard-hint {
    margin-top: 10px;
    padding-top: 10px;
    border-top: 1px dashed $line-2;
    font-size: 0.8846em; // 0.71875rem / 0.8125rem
    color: $ink-3;
    line-height: 1.4;
  }

  &__health-badge {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 4px;
    padding: 3px 8px;
    border-radius: 999px;
    border: 1px solid transparent;
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.02em;
    white-space: nowrap;

    &--success {
      color: $ok;
      background: $ok-wash;
    }

    &--failed {
      color: $danger;
      background: $danger-wash;
      cursor: pointer;
      border: 0;
      &:hover { background: rgba($danger, 0.16); }
    }

    &--loading {
      color: $ink-3;
      background: $paper;
    }
  }

  &__health-check-btn {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 4px;
    padding: 3px 9px;
    border-radius: 999px;
    border: 1px solid $signal;
    background: $card;
    color: $signal;
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.02em;
    white-space: nowrap;
    cursor: pointer;
    transition: background 0.15s ease, color 0.15s ease;

    &:hover { background: $wash; }
  }

  &__mcard-mark {
    flex-shrink: 0;
    width: 20px;
    height: 20px;
    border-radius: 50%;
    display: grid;
    place-items: center;
    background: $signal;
    color: #fff;
  }

  &__caps { display: flex; flex-wrap: wrap; gap: 5px; }
  &__cap {
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 600;
    letter-spacing: 0.02em;
    color: $ink-2;
    background: $paper;
    border: 1px solid $line;
    border-radius: 6px;
    padding: 2px 7px;
  }

  &__mcard-stats {
    display: flex;
    flex-wrap: wrap;
    gap: 14px;
    padding-top: 10px;
    border-top: 1px solid $line-2;
  }

  &__stat { display: flex; flex-direction: column; gap: 1px; }
  &__stat-k { @extend %micro; font-size: 0.6923em; color: $ink-3; } // 0.5625rem / 0.8125rem
  &__stat-v { font-family: $mono; font-size: 0.9615em; font-weight: 700; color: $ink; letter-spacing: -0.01em; } // 0.78125rem / 0.8125rem

  // Per-model retry/timeout — shown on a model card once it's selected.
  // Wrapped so clicks on the inputs (stopPropagation'd in the component)
  // don't also toggle the card's selection.
  &__mcard-retry {
    display: flex;
    flex-direction: column;
    gap: 8px;
    margin-top: 10px;
    padding-top: 10px;
    border-top: 1px solid $line-2;
    cursor: default;
  }
  &__mcard-retry-fields {
    display: flex;
    gap: 10px;
  }
  &__mcard-retry-hint {
    display: flex;
    align-items: flex-start;
    gap: 5px;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    color: $ink-3;
    line-height: 1.3;

    svg { flex-shrink: 0; margin-top: 2px; }
  }
  &__mcard-retry-field {
    flex: 1;
    display: flex;
    flex-direction: column;
    gap: 4px;

    label {
      @extend %micro;
      font-size: 0.6923em; // 0.5625rem / 0.8125rem
      color: $ink-3;
    }

    input {
      width: 100%;
      border: 1.5px solid $line;
      border-radius: 8px;
      padding: 5px 8px;
      font-family: $mono;
      font-size: 0.8846em; // 0.71875rem / 0.8125rem
      font-weight: 700;
      color: $ink;
      background: $card;
      transition: border-color 0.15s ease, box-shadow 0.15s ease;

      &:focus { outline: none; border-color: $signal; box-shadow: $ring; }
    }
  }

  // ---- Retry/timeout mode: "Apply to all" vs "Individually" (Models step) -
  &__retry-mode {
    margin-top: 14px;
    margin-bottom: 4px;
    padding: 12px 14px;
    border: 1px solid $line;
    border-radius: 12px;
    background: $paper;
  }

  &__retry-mode-head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    flex-wrap: wrap;
    gap: 10px;
  }

  &__retry-mode-all {
    display: flex;
    align-items: flex-end;
    flex-wrap: wrap;
    gap: 12px;
    margin-top: 12px;

    .ev__mcard-retry-field { flex: 0 0 130px; }
  }

  &__retry-mode-note {
    display: flex;
    align-items: flex-start;
    gap: 5px;
    flex: 1 1 100%;
    margin-top: 10px;
    font-size: 0.8846em; // 0.71875rem / 0.8125rem
    color: $ink-3;
    line-height: 1.4;

    svg { flex-shrink: 0; margin-top: 2px; }
  }

  // ---- Retest on Wrong body (card wrapper is &__msec; header lives there
  // too, with the Yes/No radio row as a head-action). The two controls
  // (Rounds / Verify Metric) each get their own bordered, tinted panel so
  // they read as clearly separate settings rather than two fields
  // floating in the same open space. -------------------------------------
  &__retest-fields {
    display: flex;
    flex-wrap: wrap;
    align-items: stretch;
    gap: 16px;
  }

  &__retest-panel {
    flex: 0 0 200px;
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 14px 16px;
    border: 1px solid $line;
    border-radius: 12px;
    background: $card;
    // Thin colored top edge distinguishes the two panels from each other
    // at a glance (rounds vs. verify metric), on top of the shared card
    // border/background separation.
    border-top: 3px solid $signal;

    input[type='number'] {
      width: 100%;
      border: 1.5px solid $line;
      border-radius: 8px;
      padding: 7px 10px;
      font-family: $mono;
      font-size: 1.0385em; // 0.84375rem / 0.8125rem
      font-weight: 700;
      color: $ink;
      background: $paper;
      transition: border-color 0.15s ease, box-shadow 0.15s ease;

      &:focus { outline: none; border-color: $signal; box-shadow: $ring; }
    }
  }

  &__retest-panel--verify {
    flex: 1 1 280px;
  }

  &__retest-panel-label {
    display: flex;
    align-items: center;
    gap: 6px;
    @extend %micro;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    color: $ink-2;

    svg { flex-shrink: 0; color: $signal; }
  }

  &__retest-panel-hint {
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    color: $ink-3;
    line-height: 1.4;
  }

  // ========================================================================
  // Test-suite step: tabs, dataset grid, subgroup rail, upload
  // ========================================================================
  &__tabs {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    padding: 4px;
    border: 1px solid $line;
    border-radius: 12px;
    background: $paper;
    margin-bottom: 18px;
  }

  &__tab {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    padding: 8px 16px;
    border: 0;
    border-radius: 9px;
    background: transparent;
    color: $ink-2;
    font-family: $sans;
    font-size: 1em; // 0.8125rem / 0.8125rem (base)
    font-weight: 650;
    cursor: pointer;
    transition: all 0.16s ease;

    &:hover { color: $ink; }
    &--on { background: $card; color: $signal; box-shadow: $soft; }
  }

  &__suite {
    flex: 1;
    min-height: 0;
    display: grid;
    grid-template-columns: minmax(0, 1fr) 390px;
    grid-template-rows: minmax(0, 1fr);
    gap: 20px;
  }

  // "N suites selected" counter, pushed to the right of the use-case chips.
  &__suite-selected {
    margin-left: auto;
    align-self: center;
    font-family: $mono;
    font-size: 0.9615em;
    color: $ink-2;
    b { color: $signal; font-weight: 700; }
  }

  // Harness suite card: description + "Scored by" line under the tags.
  &__dcard-desc {
    padding-top: 10px;
    border-top: 1px solid $line-2;
    font-size: 0.9231em;
    color: $ink-2;
    line-height: 1.45;

    p + p { margin-top: 5px; }
    p span {
      @extend %micro;
      font-size: 0.7692em;
      color: $ink-3;
      margin-right: 6px;
    }
  }

  &__suite-scroll {
    min-width: 0;
    min-height: 0;
    overflow-y: auto;
    margin: 0 -6px;
    padding: 2px 6px 6px;
  }

  &__dgrid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
    gap: 12px;

    @media (min-width: 1800px) {
      grid-template-columns: repeat(auto-fill, minmax(310px, 1fr));
    }
  }

  &__dcard {
    position: relative;
    display: flex;
    flex-direction: column;
    gap: 11px;
    text-align: left;
    padding: 15px 16px;
    border: 1.5px solid $line;
    border-radius: 14px;
    background: $card;
    cursor: pointer;
    transition: border-color 0.16s ease, box-shadow 0.16s ease, transform 0.16s ease, background 0.16s ease;

    &:hover { border-color: $ink-3; box-shadow: $lift; transform: translateY(-2px); }
    &--on { border-color: $signal; background: $wash; &:hover { border-color: $signal; } }

    // A dataset with 0 questions can still be picked (so the info icon
    // and warning banner are reachable), but reads visibly muted/flagged
    // so it doesn't look identical to a usable suite at a glance.
    &--empty {
      border-color: rgba($danger, 0.3);

      &:hover { border-color: rgba($danger, 0.5); }
    }
  }

  &__dcard-top { display: flex; align-items: center; justify-content: space-between; gap: 10px; }
  &__dcard-id { display: flex; align-items: center; gap: 11px; min-width: 0; }

  &__dcard-icon {
    flex-shrink: 0;
    width: 36px;
    height: 36px;
    border-radius: 10px;
    display: grid;
    place-items: center;
    background: $paper;
    border: 1px solid $line;
    color: $ink;
    transition: all 0.16s ease;
  }
  &__dcard--on &__dcard-icon { background: $signal; border-color: $signal; color: #fff; }

  &__dcard-name { font-family: $display; font-size: 1.0769em; font-weight: 700; color: $ink; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; } // 0.875rem / 0.8125rem

  &__dcard-warn-icon {
    flex-shrink: 0;
    display: inline-flex;
    color: $danger;
  }

  &__dcard-tags { display: flex; flex-wrap: wrap; align-items: center; gap: 6px; }

  &__tag {
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 600;
    letter-spacing: 0.02em;
    color: $ink-2;
    background: $paper;
    border: 1px solid $line;
    border-radius: 6px;
    padding: 2px 7px;
  }
  &__tag--custom { color: $signal; background: $wash; border-color: rgba($signal, 0.25); }
  &__tag--deepeval { color: $ok; background: $ok-wash; border-color: rgba($ok, 0.25); }
  &__tag--count { border: 0; background: transparent; color: $ink-3; padding-left: 0; }
  &__tag--count-empty { color: $danger; font-weight: 700; }

  // Full-width banner shown below the grid+subgroup-rail layout whenever
  // the currently selected dataset has 0 questions — explains why
  // Continue is disabled without the user having to guess.
  &__dataset-empty-warning {
    display: flex;
    align-items: flex-start;
    gap: 9px;
    margin-top: 14px;
    padding: 11px 14px;
    border: 1px solid rgba($danger, 0.3);
    border-radius: 12px;
    background: $danger-wash;
    color: $danger;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    line-height: 1.5;

    svg { flex-shrink: 0; margin-top: 2px; }
    strong { font-weight: 700; }
  }

  // ---- dcard header actions: preview button + selection mark (Change-2) --
  &__dcard-actions {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 8px;
  }

  &__dcard-preview-btn {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 5px 10px;
    border: 1px solid $line;
    border-radius: 999px;
    background: $paper;
    color: $ink-2;
    font-family: $sans;
    font-size: 0.8846em; // 0.71875rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $signal; color: $signal; background: $wash; }
  }

  // ---- subgroup rail ------------------------------------------------------
  &__rail {
    flex-shrink: 0;
    display: flex;
    flex-direction: column;
    border: 1px solid $line;
    border-radius: 14px;
    background: $paper;
    min-height: 0;
    overflow: hidden;
  }

  &__rail-head { flex-shrink: 0; padding: 15px 16px 13px; border-bottom: 1px solid $line; }
  &__rail-head-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
  }
  &__rail-title {
    display: flex;
    align-items: center;
    gap: 7px;
    font-family: $display;
    font-size: 1em; // 0.8125rem / 0.8125rem (base)
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $ink;
    svg { color: $signal; }
  }
  &__rail-actions {
    flex-shrink: 0;
    display: flex;
    gap: 10px;
  }
  &__rail-sub { margin-top: 4px; font-size: 0.8846em; color: $ink-3; line-height: 1.45; } // 0.71875rem / 0.8125rem

  &__rail-scroll {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding: 10px;
    display: flex;
    flex-direction: column;
    gap: 6px;
  }

  &__rail-empty {
    margin: 10px;
    padding: 18px 12px;
    text-align: center;
    border: 1px dashed $line;
    border-radius: 10px;
    font-size: 0.9231em; color: $ink-3; } // 0.75rem / 0.8125rem

  &__check-row {
    display: flex;
    align-items: center;
    gap: 10px;
    width: 100%;
    text-align: left;
    padding: 9px 11px;
    border: 1px solid $line;
    border-radius: 10px;
    background: $card;
    cursor: pointer;
    transition: border-color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $ink-3; }
    &--on { border-color: $signal; background: $wash; }
  }

  &__check {
    flex-shrink: 0;
    width: 17px;
    height: 17px;
    border-radius: 5px;
    border: 1.5px solid $ink-3;
    background: $card;
    display: grid;
    place-items: center;
    color: transparent;
    transition: all 0.14s ease;

    &--on { background: $signal; border-color: $signal; color: #fff; }
  }

  &__check-label {
    font-size: 1em; // 0.8125rem / 0.8125rem (base)
    font-weight: 600;
    color: $ink;

    // Extra boost beyond the automatic base-font scale-up (see
    // $ev-base-font), since at this width the subgroup cards still read
    // smaller than the surrounding provider/model/dataset cards otherwise.
    @media (min-width: 1800px) {
      font-size: 1.1538em; // 0.9375rem / 0.8125rem
    }
  }

  // ---- upload panel -------------------------------------------------------
  &__upload {
    border: 1.5px solid $line;
    border-radius: 14px;
    background: $paper;
    padding: 20px;
    max-width: 560px;
  }

  &__drop {
    display: flex;
    align-items: center;
    gap: 11px;
    width: 100%;
    border: 1.5px dashed $ink-3;
    border-radius: 12px;
    padding: 16px;
    background: $card;
    color: $ink-3;
    font-size: 1.0385em; // 0.84375rem / 0.8125rem
    font-weight: 500;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $signal; color: $signal; background: $wash; }

    svg { flex-shrink: 0; }
  }
  &__drop-file { color: $ink; font-weight: 600; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
  &__drop--has { border-style: solid; border-color: $signal; color: $ink; }

  &__upload-actions {
    display: flex;
    justify-content: flex-end;
    gap: 10px;
    margin-top: 20px;
  }

  // ========================================================================
  // Metrics step: chips + judge rail
  // ========================================================================
  &__metrics {
    flex: 1;
    min-height: 0;
    display: grid;
    grid-template-columns: 1fr 300px;
    // Without an explicit row height, the implicit single row sizes to
    // its tallest column's content (auto/max-content) instead of being
    // constrained to the space actually available — which is what was
    // letting &__metrics-main's content grow past the bottom of the
    // stage with no scrollbar appearing anywhere. minmax(0, 1fr) forces
    // the row to the container's real height so &__metrics-main's own
    // overflow-y: auto (below) can actually kick in.
    grid-template-rows: minmax(0, 1fr);
    gap: 16px;
  }

  &__metrics-main {
    min-width: 0;
    min-height: 0;
    display: flex;
    flex-direction: column;
    gap: 16px;
    // Independent scrollbar for the cards column — without this, overflow
    // bubbles up to &__stage-body and scrolls the whole step (cards +
    // Judge Model rail) as one unit. With it, only this column scrolls;
    // &__judge keeps its own separate internal scroll region
    // (&__judge-scroll) and its head/footer stay pinned in place.
    overflow-y: auto;
    padding-right: 4px;
    margin-right: -4px;
  }

  // ---- Metrics step: one card per section -----------------------------
  // Same visual language as &__judge (border/radius/paper background,
  // header+body split) so the whole step reads as a consistent stack of
  // cards instead of one long undifferentiated column.
  &__msec {
    border: 1px solid $line;
    border-radius: 14px;
    background: $paper;
  }

  &__msec-head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
    padding: 14px 16px 12px;
    border-bottom: 1px solid $line;
  }

  &__msec-head-text { min-width: 0; }

  &__msec-title {
    display: flex;
    align-items: center;
    gap: 7px;
    font-family: $display;
    font-size: 1.0769em; // 0.875rem / 0.8125rem
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $ink;

    svg { flex-shrink: 0; color: $signal; }
  }

  &__msec-sub { margin-top: 3px; font-size: 0.9231em; color: $ink-3; line-height: 1.4; } // 0.75rem / 0.8125rem

  &__msec-head-action { flex-shrink: 0; }

  &__msec-body { padding: 16px; }

  // ---- Run Samples + Instruction side-by-side (top of Metrics step) -------
  &__samples-instruction-row {
    display: grid;
    grid-template-columns: 1fr 1fr;
    align-items: start;
    gap: 16px;
  }

  // Run Samples card body wraps the standalone runSamplesControl field
  // directly — .ev__field--samples below sets its width; no wrapper class
  // needed now that it isn't laid out side-by-side with a note anymore.
  &__field--samples { max-width: 260px; }
  // Top K (RAG-only, Test Suite step) — a simple number field, doesn't
  // need the wide layout the samples control does.
  // Top K (RAG-only, Test Suite step) — the field row is wide enough for
  // the label's long hint text to stay on one line; the actual number
  // input is kept narrow separately via &__input--topk below.
  &__field--topk {
    max-width: 480px;
    margin-bottom: 20px;

    .opt {
      white-space: nowrap;
      font-size: 0.85em;
    }
  }
  &__input--topk { max-width: 140px; }

  // Shown above the metrics chips when the catalog has entries but nothing
  // is selected yet — mirrors .ev__judge-required's styling/tone.
  &__metrics-required {
    margin-bottom: 12px;
    padding: 9px 11px;
    border: 1px dashed rgba($danger, 0.35);
    border-radius: 10px;
    background: $danger-wash;
    color: $danger;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    line-height: 1.4;
  }

  // ---- Run samples: Custom / Full radio row -------------------------------
  &__radio-row {
    display: inline-flex;
    align-items: center;
    gap: 16px;
  }

  &__radio-opt {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    border: 0;
    background: transparent;
    padding: 0;
    cursor: pointer;
    font-family: $sans;
    font-size: 1em; // 0.8125rem / 0.8125rem (base)
    font-weight: 600;
    color: $ink-2;
    transition: color 0.15s ease;

    &:hover { color: $ink; }
    &--on { color: $ink; }
  }

  &__radio-full-note {
    margin-top: 10px;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    color: $ink-3;
    line-height: 1.4;
  }

  &__metrics-count {
    font-family: $mono;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    color: $ink-2;
    b { color: $signal; font-weight: 700; }
  }

  &__metrics-actions { display: flex; gap: 14px; }

  &__link {
    font-family: $sans;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 600;
    color: $signal;
    background: none;
    border: 0;
    padding: 0;
    cursor: pointer;
    &:hover { text-decoration: underline; }
  }

  &__chips {
    flex: 1;
    margin: 0 -6px;
    padding: 4px 6px;
    align-content: flex-start;
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
  }

  &__chip {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    padding: 9px 14px;
    border: 1.5px solid $line;
    border-radius: 999px;
    background: $card;
    color: $ink-2;
    font-size: 0.95em;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.15s ease;
    height: fit-content;

    &:hover { border-color: $ink-3; color: $ink; }

    &--on {
      border-color: $signal;
      background: $signal;
      color: #fff;
    }

    // Custom metrics (from GET /metrics's `custom` array) get a dashed
    // border at rest so they read as a distinct category from the plain
    // built-in metric chips above, even before the section header makes
    // that explicit.
    &--custom {
      border-style: dashed;

      &.ev__chip--on { border-style: solid; }
    }
  }

  &__chip-tick {
    display: grid;
    place-items: center;
    width: 14px;
    height: 14px;
  }

  // Small gavel glyph on a custom metric chip that needs a judge model to
  // grade it — purely informational, doesn't affect the always-shown/
  // mandatory Judge Model rail itself.
  &__chip-judge-icon {
    flex-shrink: 0;
    opacity: 0.6;
  }
  &__chip--on &__chip-judge-icon { opacity: 0.85; }

  // ---- instruction (Metrics-step card; see &__msec for the header/body
  // wrapper) — generate button sits in the card head as a head-action,
  // textarea in the card body. ------------------------------------------
  &__instruction-generate-btn {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 6px 12px;
    border: 1px solid $signal;
    border-radius: 999px;
    background: $card;
    color: $signal;
    font-family: $sans;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: background 0.15s ease, color 0.15s ease;

    &:hover:not(:disabled) { background: $wash; }
    &:disabled { cursor: not-allowed; opacity: 0.5; }
  }

  &__instruction-textarea {
    width: 100%;
    min-height: 110px;
    border: 1.5px solid $line;
    border-radius: 11px;
    padding: 11px 13px;
    font-family: $sans;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 500;
    line-height: 1.55;
    color: $ink;
    background: $card;
    resize: vertical;
    transition: border-color 0.15s ease, box-shadow 0.15s ease;

    &::placeholder { color: $ink-3; font-weight: 400; }
    &:focus { outline: none; border-color: $signal; box-shadow: $ring; }
  }

  // ---- judge rail ---------------------------------------------------------
  &__judge {
    flex-shrink: 0;
    display: flex;
    flex-direction: column;
    border: 1px solid $line;
    border-radius: 14px;
    background: $paper;
    min-height: 0;
    overflow: hidden;
  }

  &__judge-head { flex-shrink: 0; padding: 15px 16px 13px; border-bottom: 1px solid $line; }

  &__judge-head-row {
    position: relative;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__judge-title {
    display: flex;
    align-items: center;
    gap: 7px;
    font-family: $display;
    font-size: 1.0769em; // 0.875rem / 0.8125rem
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $ink;
    svg { color: $signal; }
  }
  &__judge-sub { margin-top: 4px; font-size: 0.9615em; color: $ink-3; line-height: 1.45; } // 0.78125rem / 0.8125rem

  // Small toggle button + popover replacing the old always-open info card
  // — same explanatory copy, but collapsed by default so it doesn't eat
  // into the rail's limited vertical space.
  &__judge-info-btn {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 24px;
    height: 24px;
    border: 1px solid $line;
    border-radius: 999px;
    background: $paper;
    color: $ink-3;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $signal; color: $signal; }
  }
  &__judge-info-btn--on { border-color: $signal; color: $signal; background: $wash; }

  &__judge-info-popover {
    position: absolute;
    top: calc(100% + 8px);
    right: 0;
    z-index: 5;
    width: 240px;
    padding: 10px 11px;
    border: 1px solid rgba($signal, 0.2);
    border-radius: 10px;
    background: $wash;
    color: $ink-2;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    line-height: 1.45;
    box-shadow: 0 12px 28px -12px rgba(0, 0, 0, 0.25);
    animation: ev-fade-in 0.15s ease both;
  }

  &__judge-search {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 8px;
    margin: 12px 12px 0;
    padding: 8px 11px;
    border: 1px solid $line;
    border-radius: 10px;
    background: $card;

    svg { flex-shrink: 0; color: $ink-3; }

    input {
      flex: 1;
      min-width: 0;
      border: 0;
      background: transparent;
      font-family: $sans;
      font-size: 0.9231em; // 0.75rem / 0.8125rem
      color: $ink;

      &:focus { outline: none; }
      &::placeholder { color: $ink-3; }
    }
  }

  // Shown at the bottom of the judge rail whenever no judge model has been
  // picked yet — mandatory for every evaluation type now, not gated on
  // LLM_Judge or the agent framework anymore.
  &__judge-required {
    flex-shrink: 0;
    margin: 0 10px 10px;
    padding: 9px 11px;
    border: 1px dashed rgba($danger, 0.35);
    border-radius: 10px;
    background: $danger-wash;
    color: $danger;
    font-size: 0.8846em; line-height: 1.4; } // 0.71875rem / 0.8125rem

  &__judge-scroll {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding: 10px;
    display: flex;
    flex-direction: column;
    gap: 6px;
  }

  &__judge-empty {
    margin: 10px;
    padding: 18px 12px;
    text-align: center;
    border: 1px dashed $line;
    border-radius: 10px;
    font-size: 0.9231em; color: $ink-3; } // 0.75rem / 0.8125rem

  &__judge-row {
    display: flex;
    align-items: center;
    gap: 10px;
    width: 100%;
    text-align: left;
    padding: 10px 11px;
    border: 1px solid $line;
    border-radius: 10px;
    background: $card;
    cursor: pointer;
    transition: border-color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $ink-3; }
    &--on { border-color: $signal; background: $wash; }
  }

  &__radio {
    flex-shrink: 0;
    width: 15px;
    height: 15px;
    border-radius: 50%;
    border: 1.5px solid $ink-3;
    background: $card;
    transition: border-width 0.14s ease, border-color 0.14s ease;
    &--on { border-color: $signal; border-width: 5px; }
  }

  &__judge-name {
    font-size: 1.0769em; // 0.875rem / 0.8125rem
    font-weight: 600;
    color: $ink;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;

    // Same extra wide-screen boost as the subgroup cards (.ev__check-label)
    // above — otherwise the judge model list reads smaller than the rest
    // of the metrics step at this width.
    @media (min-width: 1800px) {
      font-size: 1.2308em; // 1rem / 0.8125rem
    }
  }
  &__judge-meta {
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    color: $ink-3;

    @media (min-width: 1800px) {
      font-size: 1.0769em; // 0.875rem / 0.8125rem
    }
  }

  // Input/output price per 1M tokens — same data + formatting (formatPrice)
  // as the Models step's card stats, just laid out as a single line on the
  // right of the row instead of a labeled stat block.
  &__judge-price {
    flex-shrink: 0;
    font-family: $mono;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 700;
    color: $ink-2;

    @media (min-width: 1800px) {
      font-size: 1.0385em; // 0.84375rem / 0.8125rem
    }
  }

  // ========================================================================
  // Review step
  // ========================================================================
  &__summary {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 12px;
  }

  &__summary-cell {
    padding: 16px;
    border: 1px solid $line;
    border-radius: 14px;
    background: $paper;
  }

  &__summary-k {
    display: flex;
    align-items: center;
    gap: 6px;
    @extend %micro;
    font-size: 0.6923em; // 0.5625rem / 0.8125rem
    color: $ink-3;
    margin-bottom: 8px;
    svg { color: $signal; }
  }

  &__summary-v { font-family: $mono; font-size: 1.8462em; font-weight: 700; color: $ink; letter-spacing: -0.02em; line-height: 1; } // 1.5rem / 0.8125rem
  &__summary-v--muted { color: $ink-3; }

  &__block { margin-top: 26px; }

  &__block-title {
    display: flex;
    align-items: center;
    gap: 7px;
    @extend %micro;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    color: $ink-2;
    margin-bottom: 12px;
    svg { color: $signal; }
    b { color: $ink-3; font-weight: 700; }
  }

  &__rows {
    border: 1px solid $line;
    border-radius: 12px;
    overflow: hidden;
  }

  &__row {
    display: flex;
    align-items: baseline;
    justify-content: space-between;
    gap: 16px;
    padding: 12px 15px;
    border-bottom: 1px solid $line-2;
    font-size: 1.0385em; &:last-child { border-bottom: 0; } // 0.84375rem / 0.8125rem

    span:first-child { @extend %micro; font-size: 0.7692em; color: $ink-3; } // 0.625rem / 0.8125rem
    span:last-child { color: $ink; font-weight: 600; text-align: right; }
  }

  &__review-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
    gap: 10px;
  }

  &__review-card {
    display: flex;
    align-items: center;
    gap: 11px;
    padding: 12px 14px;
    border: 1px solid $line;
    border-radius: 12px;
    background: $paper;
  }
  &__review-card-icon {
    flex-shrink: 0;
    width: 32px;
    height: 32px;
    border-radius: 9px;
    display: grid;
    place-items: center;
    background: $card;
    border: 1px solid $line;
    color: $signal;
  }
  &__review-card-name { font-family: $display; font-size: 1em; font-weight: 700; color: $ink; } // 0.8125rem / 0.8125rem (base)
  &__review-card-sub { font-size: 0.8846em; color: $ink-3; margin-top: 1px; } // 0.71875rem / 0.8125rem

  &__metric-tags { display: flex; flex-wrap: wrap; gap: 7px; }
  &__metric-tag {
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 600;
    color: $signal;
    background: $wash;
    border: 1px solid rgba($signal, 0.2);
    border-radius: 8px;
    padding: 5px 11px;
  }

  &__empty {
    padding: 20px;
    text-align: center;
    border: 1px dashed $line;
    border-radius: 12px;
    background: $paper;
    color: $ink-3;
    font-size: 1.0385em; } // 0.84375rem / 0.8125rem

  &__error {
    margin-top: 18px;
    display: flex;
    align-items: center;
    gap: 9px;
    font-size: 1em; // 0.8125rem / 0.8125rem (base)
    font-weight: 500;
    color: $danger;
    background: $danger-wash;
    border: 1px solid rgba($danger, 0.2);
    border-radius: 10px;
    padding: 11px 14px;
  }

  &__spin { animation: ev-spin 0.8s linear infinite; }

  // ========================================================================
  // REDESIGN LAYER
  // Everything below restyles the wizard to the reference screens: numbered
  // stepper, radio/checkbox cards, table + side panel, sectioned form rows,
  // review banner. Only existing tokens are used ($signal, $wash, $ink*,
  // $paper, $card, $line*, $ok, $danger, $sans/$display/$mono). Rules here
  // sit last in the block so they win over the earlier ones they refine.
  // ========================================================================

  // ---- sidebar stepper ----------------------------------------------------
  // Quiet timeline: completed steps recede (soft tick, their value still
  // readable), the current step is the only filled element, upcoming steps
  // are muted. The connector is a short segment between ticks rather than
  // one continuous rule, coloured only where the path has been walked.
  &__manifest-head { padding: 18px 20px 16px; }
  &__manifest-title { margin-top: 8px; }
  &__manifest-done { margin-top: 8px; font-size: 1.0385em; color: $ink-3; }

  &__segs { display: flex; gap: 4px; margin-top: 14px; }
  &__seg {
    flex: 1;
    height: 4px;
    border-radius: 999px;
    background: $line;
    transition: background 0.2s ease;

    &--done { background: $signal; }
    &--active { background: rgba($signal, 0.4); }
  }

  // shell: tighter gutter; the sidebar (aside) and stage (section) keep
  // their original 16px radius (set on &__manifest / &__stage above)
  &__shell { gap: 6px; }

  &__spec { gap: 4px; padding: 12px 10px 14px; }

  &__spec-row {
    grid-template-columns: 28px minmax(0, 1fr) auto;
    align-items: center;
    gap: 12px;
    padding: 10px 12px 10px 10px;
    border-radius: 10px;

    // segment from this tick to the next one, with a 3px breath at each end
    // (rows share one height, so the percentages line up)
    &::before {
      left: 23px;
      top: calc(50% + 17px);
      bottom: calc(-50% + 13px);
      width: 2px;
      border-radius: 2px;
      background: $line;
    }
  }

  &__spec-tick {
    width: 28px;
    height: 28px;
    border-radius: 50%;
    border: 1.5px solid $line;
    background: $card;
    color: $ink-3;
    font-size: 1em;
  }

  &__spec-label {
    font-family: $sans;
    font-size: 1.0769em;
    font-weight: 600;
    letter-spacing: 0;
    text-transform: none;
    color: $ink-3;
  }

  &__spec-value {
    font-size: 1.2692em;
    font-weight: 650;

    &[data-empty='true'] { font-family: $sans; font-weight: 500; color: $ink-3; }
  }

  &__spec-row--done {
    &::before { background: rgba($signal, 0.45); }
    .ev__spec-tick { background: $wash; border-color: transparent; color: $signal; }
    .ev__spec-label { color: $ink-3; }
    &:not(:disabled):hover { background: $paper; }
  }

  &__spec-row--active {
    background: $wash;
    box-shadow: none;
    .ev__spec-tick {
      background: $signal;
      border-color: $signal;
      color: #fff;
      box-shadow: 0 0 0 4px rgba($signal, 0.16);
    }
    .ev__spec-label { color: $signal; }
    &:not(:disabled):hover { background: $wash; }
  }

  &__spec-row--todo { opacity: 1; }

  // "Edit" pill, revealed on hover / keyboard focus of a completed step
  &__spec-edit {
    align-self: center;
    padding: 3px 9px;
    border: 1px solid rgba($signal, 0.25);
    border-radius: 999px;
    background: $card;
    font-family: $sans;
    font-size: 0.7692em;
    font-weight: 650;
    color: $signal;
    opacity: 0;
    transition: opacity 0.15s ease;
  }
  &__spec-row:hover &__spec-edit,
  &__spec-row:focus-visible &__spec-edit { opacity: 1; }

  &__manifest-foot {
    flex-shrink: 0;
    padding: 12px 20px;
    border-top: 1px solid $line-2;
    font-size: 0.9615em;
    color: $ink-3;
    line-height: 1.4;
  }

  // ---- stage head / footer ------------------------------------------------
  &__stage-step {
    font-family: $sans;
    font-size: 0.9231em;
    font-weight: 700;
    color: $signal;
  }
  &__stage-title { margin-top: 4px; }

  &__blocker { font-size: 1em; color: $ink-2; }

  // ---- question label -----------------------------------------------------
  &__q {
    font-family: $sans;
    font-size: 1.0769em;
    font-weight: 700;
    color: $ink;
    margin-bottom: 10px;

    span { font-weight: 500; color: $ink-2; }
  }

  // ---- radio / checkbox cards --------------------------------------------
  &__choice-row { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 12px; }
  &__choice-col { display: flex; flex-direction: column; gap: 10px; }
  &__choice-grid { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 10px; }

  &__choice {
    display: flex;
    flex-direction: column;
    gap: 8px;
    width: 100%;
    text-align: left;
    padding: 16px;
    border: 1.5px solid $line;
    border-radius: 14px;
    background: $card;
    font-family: $sans;
    cursor: pointer;
    transition: border-color 0.16s ease, background 0.16s ease;

    &:hover { border-color: $ink-3; }
    &:disabled { opacity: 0.55; cursor: not-allowed; }

    &--on {
      border-color: $signal;
      background: $wash;
      &:hover { border-color: $signal; }
      .ev__pilltag { background: $card; }
    }

    &--row {
      flex-direction: row;
      align-items: flex-start;
      gap: 12px;
      padding: 13px 16px;

      > .ev__radio, > .ev__check { margin-top: 3px; }
    }
  }

  &__choice-head { display: flex; align-items: center; gap: 10px; }
  &__choice-body { flex: 1; min-width: 0; display: flex; flex-direction: column; gap: 3px; }
  &__choice-name {
    font-family: $display;
    font-size: 1.1538em;
    font-weight: 700;
    color: $ink;
  }
  &__choice-desc { font-size: 1em; color: $ink-2; line-height: 1.45; }
  &__choice-tags { display: flex; flex-wrap: wrap; gap: 6px; margin-top: 6px; }
  &__choice-price {
    flex-shrink: 0;
    align-self: center;
    font-family: $mono;
    font-size: 0.9231em;
    font-weight: 600;
    color: $ink-2;
    white-space: nowrap;
  }

  &__pilltag {
    font-family: $sans;
    font-size: 0.8462em;
    font-weight: 650;
    color: $signal;
    background: $wash;
    border-radius: 6px;
    padding: 2px 8px;
    white-space: nowrap;

    &--muted { color: $ink-2; background: $paper; }
  }

  // "Because you chose …" follow-up block
  &__followup {
    margin-top: 20px;
    padding: 18px 20px 20px;
    border-radius: 14px;
    background: $paper;

    .ev__q { margin-top: 16px; }
  }
  &__followup-eyebrow { font-size: 0.9231em; font-weight: 650; color: $ink-2; }

  // checkbox used as a real <button> (models table, "save traces")
  &__check {
    padding: 0;
    &:disabled { opacity: 0.45; cursor: not-allowed; }
  }
  button#{&}__check { cursor: pointer; }

  // ---- main + side panel split (Models step) ------------------------------
  &__split {
    flex: 1;
    min-height: 0;
    display: grid;
    grid-template-columns: minmax(0, 1fr) 390px;
    grid-template-rows: minmax(0, 1fr);
    gap: 20px;
  }
  &__split-main { min-width: 0; min-height: 0; display: flex; flex-direction: column; }
  // model cards inside the split: slightly narrower minimum so two or more
  // fit beside the 390px panel, and every card is the same size — equal
  // row heights, cards stretched to fill, stats pinned to the bottom edge
  &__split &__models-scroll { margin-top: 0; }
  &__split &__grid--wide {
    grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
    grid-auto-rows: 1fr;
    align-items: stretch;
  }
  &__split &__mcard { height: 100%; }
  &__split &__mcard-stats { margin-top: auto; }

  &__step-toolbar { flex-wrap: wrap; }
  &__viewchips { flex: 0 1 auto; display: flex; flex-wrap: wrap; gap: 8px; }
  &__split &__toolbar-search { flex: 1 1 200px; }
  &__split &__toolbar-refresh { margin-left: auto; }

  // Side panel: square corners, bordered on all sides, runs the full height
  // of the step.
  &__panel {
    align-self: stretch;
    min-height: 0;
    overflow-y: auto;
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 18px;
    border: 1px solid $line;
    border-radius: 0;
    background: $paper;
  }
  // Pinned variant (Models step): the top block stays put, the rest scrolls.
  &__panel--pinned { overflow: hidden; padding: 0; gap: 0; }
  &__panel-fixed {
    flex-shrink: 0;
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 18px;
    border-bottom: 1px solid $line;
    background: $paper;
  }
  &__panel-scroll {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 18px;
  }
  // ---- per-model item ("In this run") ------------------------------------
  // Header: health dot, model name + provider, remove. In "Per model" mode a
  // tinted body underneath holds that model's two steppers, so each model
  // reads as one self-contained unit.
  &__pm {
    border: 1px solid $line;
    border-radius: 10px;
    background: $card;
    overflow: hidden;
  }
  &__pm-head { display: flex; align-items: center; gap: 10px; padding: 10px 12px; }
  &__pm-dot {
    flex-shrink: 0;
    width: 8px;
    height: 8px;
    border-radius: 50%;
    background: $ink-3;

    &--ok { background: $ok; }
    &--fail { background: $danger; }
  }
  &__pm-name {
    flex: 1;
    min-width: 0;
    font-size: 1em;
    font-weight: 700;
    color: $ink;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;

    small { display: block; font-size: 0.8846em; font-weight: 500; color: $ink-2; overflow: hidden; text-overflow: ellipsis; }
  }
  &__pm-body { padding: 10px 12px 12px; border-top: 1px solid $line-2; background: $paper; }

  // ---- stepper fields (retries / timeout) --------------------------------
  &__stepfields { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 10px; }
  &__stepfield { min-width: 0; display: flex; flex-direction: column; gap: 6px; }
  &__stepfield-label {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    font-size: 0.8846em;
    font-weight: 650;
    color: $ink-2;

    svg { color: $signal; }
  }

  &__stepper {
    display: flex;
    align-items: stretch;
    height: 34px;
    border: 1.5px solid $line;
    border-radius: 9px;
    background: $card;
    overflow: hidden;
    transition: border-color 0.15s ease, box-shadow 0.15s ease;

    &:focus-within { border-color: $signal; box-shadow: $ring; }

    button {
      flex: 0 0 30px;
      display: grid;
      place-items: center;
      border: 0;
      background: $paper;
      color: $ink-2;
      cursor: pointer;
      transition: background 0.12s ease, color 0.12s ease;

      &:hover:not(:disabled) { background: $wash; color: $signal; }
      &:disabled { opacity: 0.35; cursor: not-allowed; }
    }

    input {
      flex: 1;
      min-width: 0;
      width: 100%;
      border: 0;
      border-left: 1px solid $line-2;
      border-right: 1px solid $line-2;
      background: transparent;
      text-align: center;
      font-family: $mono;
      font-size: 1em;
      font-weight: 700;
      color: $ink;
      -moz-appearance: textfield;
      appearance: textfield;

      &:focus { outline: none; }
      &::-webkit-outer-spin-button,
      &::-webkit-inner-spin-button { -webkit-appearance: none; margin: 0; }
    }
  }

  // ---- judge panel (Metrics step) ----------------------------------------
  // Heading row with a chip showing the current judge (or "Required").
  &__panel-headrow { display: flex; align-items: center; justify-content: space-between; gap: 10px; }
  &__panel-chip {
    min-width: 0;
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 3px 9px;
    border-radius: 999px;
    background: $wash;
    font-size: 0.8846em;
    font-weight: 700;
    color: $signal;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;

    svg { flex-shrink: 0; }
    &--req { background: $danger-wash; color: $danger; }
  }
  &__panel-fixed .ev__sbar { flex: none; min-width: 0; }

  // Judge candidate card: radio, icon tile, name + provider, an "Under test"
  // flag where it applies, and a row of figures underneath.
  &__jcard {
    flex-shrink: 0;
    display: flex;
    flex-direction: column;
    gap: 10px;
    width: 100%;
    text-align: left;
    padding: 12px 14px;
    border: 1.5px solid $line;
    border-radius: 12px;
    background: $card;
    font-family: $sans;
    cursor: pointer;
    transition: border-color 0.16s ease, background 0.16s ease, box-shadow 0.16s ease;

    &:hover { border-color: $ink-3; box-shadow: $soft; }

    &--on {
      border-color: $signal;
      background: $wash;
      box-shadow: 0 8px 18px -12px rgba($signal, 0.6);
      &:hover { border-color: $signal; }
      .ev__srow-icon { background: $signal; border-color: $signal; color: #fff; }
      .ev__jcard-stats { border-top-color: rgba($signal, 0.18); }
      .ev__jcard-flag { background: $card; }
    }
  }
  &__jcard-head { display: flex; align-items: center; gap: 10px; }
  &__jcard-id { flex: 1; min-width: 0; display: flex; flex-direction: column; gap: 1px; }
  &__jcard-name {
    font-family: $display;
    font-size: 1.0769em;
    font-weight: 700;
    color: $ink;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
  &__jcard-provider { font-size: 0.9231em; color: $ink-2; }
  &__jcard-flag {
    flex-shrink: 0;
    padding: 2px 8px;
    border: 1px solid $line;
    border-radius: 999px;
    background: $paper;
    font-size: 0.8077em;
    font-weight: 650;
    color: $ink-2;
    white-space: nowrap;
  }
  &__jcard-stats {
    display: flex;
    gap: 18px;
    padding-top: 10px;
    border-top: 1px solid $line-2;
  }

  &__panel-title { font-family: $display; font-size: 1.0769em; font-weight: 800; color: $ink; }
  &__panel-sub { margin-top: -6px; font-size: 0.9231em; color: $ink-2; }
  &__panel-list { display: flex; flex-direction: column; gap: 8px; }
  &__panel-item {
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 10px 12px;
    border: 1px solid $line;
    border-radius: 10px;
    background: $card;
    font-size: 1em;
    font-weight: 600;
    color: $ink;

    small { display: block; font-size: 0.9231em; font-weight: 500; color: $ink-2; }

    &--off { background: transparent; border-style: dashed; font-weight: 500; color: $ink-2; }
  }
  &__panel-item-row { display: flex; align-items: center; justify-content: space-between; gap: 10px; }
  &__panel-cov { display: flex; align-items: center; gap: 10px; }
  &__panel-x {
    flex-shrink: 0;
    display: inline-flex;
    padding: 2px;
    border: 0;
    background: transparent;
    color: $ink-2;
    cursor: pointer;
    &:hover { color: $ink; }
  }
  &__panel-divider { height: 1px; margin: 6px 0; background: $line; }
  &__panel-stats {
    display: flex;
    flex-direction: column;
    gap: 6px;
    margin-top: 4px;
    padding-top: 12px;
    border-top: 1px solid $line;

    div { display: flex; justify-content: space-between; gap: 12px; font-size: 1em; color: $ink-2; }
    b { font-family: $mono; font-weight: 700; color: $ink; }
  }
  &__panel-note { font-size: 0.9231em; color: $ink-2; line-height: 1.45; }

  &__dot {
    flex-shrink: 0;
    width: 16px;
    height: 16px;
    border-radius: 50%;
    border: 1.5px solid $ink-3;
    display: grid;
    place-items: center;
    color: #fff;

    &--on { background: $signal; border-color: $signal; }
  }

  // ---- Type step: "Agent setup" panel -------------------------------------
  // The follow-up choices for Agent, as one panel: a header with a status
  // chip, then numbered steps joined by a rail (1 how to test, 2 agents or
  // framework). Choice cards use the same anatomy as the suite/model cards.
  &__asetup {
    flex-shrink: 0;
    margin-top: 16px;
    border: 1px solid $line;
    border-radius: 14px;
    background: $paper;
    overflow: hidden;
  }
  &__asetup-head {
    display: flex;
    align-items: center;
    gap: 14px;
    padding: 16px 20px;
    border-bottom: 1px solid $line;
    background: $card;

    .ev__option-icon { width: 40px; height: 40px; border-radius: 11px; }
  }
  &__asetup-title {
    flex: 1;
    min-width: 0;

    h3 { font-family: $display; font-size: 1.2308em; font-weight: 800; letter-spacing: -0.01em; color: $ink; }
    p { margin-top: 2px; font-size: 1.0385em; color: $ink-2; }
  }
  &__asetup-status {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 4px 11px;
    border-radius: 999px;
    background: $ok-wash;
    font-size: 0.9231em;
    font-weight: 700;
    color: $ok;

    &--todo { background: $danger-wash; color: $danger; }
  }

  &__asetup-step {
    position: relative;
    display: grid;
    grid-template-columns: 26px minmax(0, 1fr);
    gap: 14px;
    padding: 18px 20px 0;

    &:last-child { padding-bottom: 20px; }

    // rail down to the next step's marker
    &:not(:last-child)::before {
      content: '';
      position: absolute;
      left: 32px;
      top: 50px;
      bottom: -14px;
      width: 2px;
      border-radius: 2px;
      background: $line;
    }
  }
  &__asetup-n {
    position: relative;
    z-index: 1;
    width: 26px;
    height: 26px;
    border-radius: 50%;
    display: grid;
    place-items: center;
    border: 1.5px solid $line;
    background: $card;
    font-family: $mono;
    font-size: 0.9231em;
    font-weight: 700;
    color: $ink-3;

    &--active { border-color: $signal; color: $signal; box-shadow: 0 0 0 4px rgba($signal, 0.14); }
    &--done { border-color: $signal; background: $signal; color: #fff; }
  }
  &__asetup-body { min-width: 0; display: flex; flex-direction: column; gap: 12px; }
  &__asetup-q {
    padding-top: 3px;
    font-family: $display;
    font-size: 1.1538em;
    font-weight: 700;
    color: $ink;

    span {
      margin-left: 8px;
      padding: 2px 8px;
      border-radius: 999px;
      background: $card;
      border: 1px solid $line;
      font-family: $sans;
      font-size: 0.7667em;
      font-weight: 650;
      color: $ink-2;
      vertical-align: 2px;
    }
  }
  &__asetup-grid { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 12px; }

  &__acard {
    display: flex;
    flex-direction: column;
    gap: 10px;
    text-align: left;
    padding: 14px 16px;
    border: 1.5px solid $line;
    border-radius: 12px;
    background: $card;
    font-family: $sans;
    cursor: pointer;
    transition: border-color 0.16s ease, box-shadow 0.16s ease, transform 0.16s ease, background 0.16s ease;

    &:hover { border-color: $ink-3; box-shadow: $lift; transform: translateY(-2px); }

    &--on {
      border-color: $signal;
      background: $wash;
      &:hover { border-color: $signal; }
      .ev__srow-icon { background: $signal; border-color: $signal; color: #fff; }
      .ev__pilltag { background: $card; }
    }
  }
  &__acard-head { display: flex; align-items: center; gap: 10px; }
  &__acard-name { font-family: $display; font-size: 1.1538em; font-weight: 700; color: $ink; }
  &__acard-desc { font-size: 1.0385em; color: $ink-2; line-height: 1.5; }
  &__acard-tags { display: flex; flex-wrap: wrap; gap: 6px; margin-top: auto; }

  // one-line consequence of the current choice
  &__asetup-sum {
    display: flex;
    align-items: flex-start;
    gap: 8px;
    padding: 10px 12px;
    border-radius: 10px;
    background: $card;
    border: 1px solid $line;
    font-size: 1em;
    color: $ink-2;
    line-height: 1.45;

    svg { flex-shrink: 0; margin-top: 2px; color: $signal; }
    b { font-weight: 700; color: $ink; }

    &--req {
      border-color: rgba($danger, 0.3);
      background: $danger-wash;
      color: $danger;
      font-weight: 650;
      svg { color: $danger; }
    }
  }
  &__asetup + &__about { margin-top: 16px; margin-bottom: 22px; }

  // ---- Type step: "About" panel -------------------------------------------
  // Sits under the three type cards and describes the selected one: what it
  // measures, what it's good for, and how a run of that type works.
  &__about {
    // the stage body is a flex column; without this the panel (overflow:
    // hidden) would be squeezed to nothing when the step's content is tall
    flex-shrink: 0;
    margin-top: 16px;
    border: 1px solid $line;
    border-radius: 14px;
    background: $paper;
    overflow: hidden;
  }
  &__about-head {
    display: flex;
    align-items: center;
    gap: 14px;
    padding: 16px 20px;
    border-bottom: 1px solid $line;
    background: $card;

    .ev__option-icon { width: 40px; height: 40px; border-radius: 11px; }
    h3 { font-family: $display; font-size: 1.2308em; font-weight: 800; letter-spacing: -0.01em; color: $ink; }
    p { margin-top: 2px; font-size: 1.0385em; color: $ink-2; }
  }
  &__about-cols { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); }
  &__about-col {
    padding: 18px 20px 20px;
    & + & { border-left: 1px solid $line; }
  }
  &__about-k { @extend %micro; font-size: 0.7692em; color: $ink-3; margin-bottom: 12px; }

  &__about-list,
  &__about-flow {
    display: flex;
    flex-direction: column;
    gap: 10px;
    list-style: none;
    font-size: 1.0385em;
    color: $ink;
    line-height: 1.45;

    li { display: grid; grid-template-columns: 20px minmax(0, 1fr); align-items: start; gap: 8px; }
    svg { margin-top: 3px; color: $signal; }
  }
  &__about-flow span {
    width: 20px;
    height: 20px;
    border-radius: 50%;
    display: grid;
    place-items: center;
    background: $signal;
    color: #fff;
    font-family: $mono;
    font-size: 0.7692em;
    font-weight: 700;
  }
  // below the agent options it needs a little more room above
  &__section + &__about { margin-top: 26px; margin-bottom: 22px; }

  &__about-empty {
    flex-shrink: 0;
    margin-top: 16px;
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 16px 18px;
    border: 1px dashed $line;
    border-radius: 12px;
    background: $paper;
    font-size: 1em;
    color: $ink-2;

    svg { flex-shrink: 0; color: $signal; }
  }

  // ---- Name step ----------------------------------------------------------
  // Two columns: the form on the left, a live "Preview in History" card on
  // the right.
  &__namegrid {
    display: grid;
    grid-template-columns: minmax(0, 620px) minmax(260px, 340px);
    justify-content: space-between;
    align-items: start;
    gap: 40px;

    .ev__quick { margin-top: 24px; }
    .ev__quick + .ev__quick { margin-top: 18px; }
  }
  &__namegrid-main { min-width: 0; }

  &__name-wrap {
    position: relative;

    .ev__name-input { padding-right: 34px; }
  }
  &__name-clear {
    position: absolute;
    right: 2px;
    top: 50%;
    transform: translateY(-60%);
    width: 26px;
    height: 26px;
    display: grid;
    place-items: center;
    border: 0;
    border-radius: 50%;
    background: $paper;
    color: $ink-2;
    cursor: pointer;
    transition: background 0.12s ease, color 0.12s ease;

    &:hover { background: $line; color: $ink; }
  }
  &__name-caption--ok { color: $ok; font-weight: 600; }

  &__hprev {
    position: sticky;
    top: 0;
    padding: 18px;
    border: 1px solid $line;
    border-radius: 14px;
    background: $paper;
  }
  &__hprev-head { @extend %micro; font-size: 0.7692em; color: $ink-3; margin-bottom: 12px; }

  &__hprev-card {
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 14px;
    border: 1.5px solid $line;
    border-radius: 12px;
    background: $card;
    box-shadow: $soft;
    transition: border-color 0.2s ease, box-shadow 0.2s ease;

    &--live { border-color: $signal; box-shadow: 0 10px 22px -14px rgba($signal, 0.6); }
  }
  &__hprev-top { display: flex; align-items: center; gap: 10px; }
  &__hprev-icon {
    flex-shrink: 0;
    width: 32px;
    height: 32px;
    border-radius: 9px;
    display: grid;
    place-items: center;
    background: $wash;
    color: $signal;
  }
  &__hprev-name {
    min-width: 0;
    font-family: $display;
    font-size: 1.1538em;
    font-weight: 700;
    color: $ink;
    overflow-wrap: anywhere;

    &[data-empty='true'] { color: $ink-3; font-weight: 600; }
  }
  &__hprev-badges { display: flex; flex-wrap: wrap; align-items: center; gap: 6px; }
  &__hprev-status {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 2px 8px;
    border-radius: 999px;
    background: $wash;
    font-family: $mono;
    font-size: 0.7692em;
    font-weight: 700;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $signal;

    &::before { content: ''; width: 5px; height: 5px; border-radius: 50%; background: $signal; }
  }
  &__hprev-meta {
    display: flex;
    justify-content: space-between;
    gap: 10px;
    padding-top: 10px;
    border-top: 1px solid $line-2;
    font-size: 0.9231em;
    color: $ink-2;
  }
  &__hprev-note { margin-top: 12px; font-size: 0.9231em; color: $ink-3; line-height: 1.45; }

  // ---- list header (Providers / Models / Test Suite) ---------------------
  // Title + live count on the left, search and refresh on the right, filter
  // chips on a second row, and a rule underneath separating it from the list.
  &__lhead {
    flex-shrink: 0;
    display: flex;
    flex-direction: column;
    gap: 12px;
    margin-bottom: 16px;
    padding-bottom: 14px;
    border-bottom: 1px solid $line;

    // inside a scrolling list (Test Suite) the header stays in view
    &--sticky {
      position: sticky;
      top: -2px;
      z-index: 2;
      padding-top: 2px;
      background: $card;
      // soft edge so the rule still reads when cards scroll underneath
      box-shadow: 0 8px 10px -10px rgba(20, 22, 27, 0.28);
    }


  }

  &__lhead-row { display: flex; align-items: center; gap: 16px; }

  &__lhead-title {
    flex: 1;
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 2px;

    h3 { font-family: $display; font-size: 1.1538em; font-weight: 800; letter-spacing: -0.01em; color: $ink; }
    span { font-size: 0.9231em; color: $ink-2; }
    b { font-weight: 700; color: $signal; }
  }

  // ---- search bar ---------------------------------------------------------
  // A compact "command bar" on the right of the list title: gradient icon
  // tile, field, live result count, "/" hint or clear button, then refresh
  // behind a divider. Resting
  // it's a white bar with a soft shadow; on focus the border becomes a
  // signal gradient, the bar lifts and a glow spreads under it.
  &__sbar {
    position: relative;
    display: flex;
    align-items: center;
    flex: 0 1 340px;
    min-width: 220px;
    gap: 9px;
    height: 38px;
    padding: 0 5px 0 5px;
    border: 1.5px solid transparent;
    border-radius: 11px;
    // two layers: the fill (padding-box) and the border colour (border-box),
    // so the border can be a gradient on focus
    background:
      linear-gradient($card, $card) padding-box,
      linear-gradient($line, $line) border-box;
    box-shadow: 0 1px 2px rgba(20, 22, 27, 0.05), 0 6px 16px -10px rgba(20, 22, 27, 0.18);
    cursor: text;
    transition: box-shadow 0.2s ease, transform 0.2s ease;

    &:hover {
      background:
        linear-gradient($card, $card) padding-box,
        linear-gradient($ink-3, $ink-3) border-box;
    }

    &:focus-within,
    &--active {
      background:
        linear-gradient($card, $card) padding-box,
        linear-gradient(120deg, $signal, $signal-2 55%, rgba($signal, 0.35)) border-box;
    }

    &:focus-within {
      transform: translateY(-1px);
      box-shadow: 0 0 0 3px rgba($signal, 0.12), 0 10px 20px -12px rgba($signal, 0.55);
    }

    input {
      flex: 1;
      min-width: 0;
      height: 100%;
      border: 0;
      background: transparent;
      font-family: $sans;
      font-size: 1em;
      font-weight: 500;
      color: $ink;
      outline: none;

      &::placeholder { color: $ink-3; font-weight: 400; }
    }
  }

  &__sbar-icon {
    flex-shrink: 0;
    width: 26px;
    height: 26px;
    border-radius: 8px;
    display: grid;
    place-items: center;
    background: linear-gradient(140deg, $signal, $signal-2);
    color: #fff;
    box-shadow: 0 4px 8px -4px rgba($signal, 0.7);
    transition: transform 0.2s ease;
  }
  &__sbar:focus-within &__sbar-icon { transform: scale(1.06) rotate(-6deg); }

  // live result count, shown while there's a query
  &__sbar-count {
    flex-shrink: 0;
    padding: 2px 8px;
    border-radius: 999px;
    background: $wash;
    font-family: $sans;
    font-size: 0.8077em;
    font-weight: 700;
    color: $signal;
    white-space: nowrap;
    animation: ev-rise 0.2s ease both;

    &--none { background: $danger-wash; color: $danger; }
  }

  &__sbar-kbd {
    flex-shrink: 0;
    min-width: 20px;
    padding: 1px 5px;
    border: 1px solid $line;
    border-bottom-width: 2px;
    border-radius: 6px;
    background: $paper;
    text-align: center;
    font-family: $mono;
    font-size: 0.7692em;
    font-weight: 700;
    color: $ink-3;
  }
  &__sbar:focus-within &__sbar-kbd { display: none; }

  &__sbar-clear {
    flex-shrink: 0;
    width: 20px;
    height: 20px;
    display: grid;
    place-items: center;
    border: 0;
    border-radius: 50%;
    background: $line;
    color: $ink-2;
    cursor: pointer;
    transition: background 0.12s ease, color 0.12s ease;

    &:hover { background: $ink-2; color: #fff; }
  }

  &__sbar-sep { flex-shrink: 0; width: 1px; height: 18px; background: $line; }

  &__sbar-refresh {
    flex-shrink: 0;
    width: 28px;
    height: 28px;
    display: grid;
    place-items: center;
    border: 0;
    border-radius: 8px;
    background: transparent;
    color: $ink-2;
    cursor: pointer;
    transition: background 0.15s ease, color 0.15s ease;

    &:hover:not(:disabled) { background: $wash; color: $signal; }
    &:disabled { cursor: default; color: $signal; }
  }

  &__lhead-filters { display: flex; flex-wrap: wrap; align-items: center; gap: 8px; }

  // ---- provider cards (Providers step) -----------------------------------
  // Same card as the Type step's options (&__option: icon tile on the left,
  // name + description, selection mark on the right), with the provider's
  // initials in the tile, a "Connected" status beside the name and the model
  // count as a last line.
  &__grid--providers {
    grid-template-columns: repeat(auto-fill, minmax(340px, 1fr));
    grid-auto-rows: 1fr;
    gap: 10px;
  }

  &__option--provider {
    height: 100%;

    .ev__option-icon {
      font-family: $display;
      font-size: 1.3846em;
      font-weight: 800;
      letter-spacing: -0.02em;
    }
    .ev__option-name { flex-wrap: wrap; gap: 4px 10px; }
    .ev__option-desc {
      display: -webkit-box;
      -webkit-box-orient: vertical;
      -webkit-line-clamp: 3;
      line-clamp: 3;
      overflow: hidden;
    }
  }
  &__option--provider.ev__option--on .ev__option-icon { background: $signal; }

  &__pv-status {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-family: $sans;
    font-size: 0.6944em; // back to ~0.96em of the base inside the larger name
    font-weight: 650;
    color: $ok;

    &::before { content: ''; width: 6px; height: 6px; border-radius: 50%; background: $ok; }
  }

  &__option-meta {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    margin-top: 4px;
    font-size: 1em;
    color: $ink-2;

    svg { color: $ink-3; }
    b { font-family: $mono; font-weight: 700; color: $ink; }
  }

  // ---- segmented control --------------------------------------------------
  &__segctl {
    display: inline-flex;
    gap: 2px;
    padding: 3px;
    border: 1px solid $line;
    border-radius: 10px;
    background: $paper;

    &--full { display: flex; background: $card; .ev__segbtn { flex: 1; } }
  }
  &__segbtn {
    border: 0;
    border-radius: 8px;
    padding: 8px 14px;
    background: transparent;
    font-family: $sans;
    font-size: 1em;
    font-weight: 650;
    color: $ink-2;
    cursor: pointer;
    transition: background 0.15s ease, color 0.15s ease;

    &:hover { color: $ink; }
    &--on { background: $signal; color: #fff; &:hover { color: #fff; } }
  }

  // ---- data table ---------------------------------------------------------
  &__table-wrap {
    flex: 0 1 auto;
    min-height: 0;
    overflow: auto;
    border: 1px solid $line;
    border-radius: 0;

    &--static { flex: none; overflow: visible; }
  }

  &__table {
    width: 100%;
    border-collapse: collapse;
    font-size: 1em;

    th {
      position: sticky;
      top: 0;
      z-index: 1;
      padding: 10px 14px;
      background: $paper;
      border-bottom: 1px solid $line;
      text-align: left;
      font-family: $sans;
      font-size: 0.9231em;
      font-weight: 650;
      color: $ink-2;
      white-space: nowrap;
    }

    td {
      padding: 11px 14px;
      border-bottom: 1px solid $line-2;
      color: $ink;
      vertical-align: middle;
    }
    tbody tr:last-child td { border-bottom: 0; }
    tbody tr { cursor: pointer; transition: background 0.12s ease; }
    tbody tr:hover td { background: $paper; }

    th.ev__table-num,
    td.ev__table-num { text-align: right; white-space: nowrap; }
    td.ev__table-num { font-family: $mono; font-size: 0.9615em; }

    tbody tr.ev__table-row--on td { background: $wash; }
    tbody tr.ev__table-row--locked { cursor: default; }

    // Health check in flight: a soft signal sheen sweeps across the whole
    // row and a bar pulses on its left edge. The sheen lives on the <tr>
    // (cells go transparent) so it travels as one band, not per cell.
    tbody tr.ev__table-row--checking {
      background-image: linear-gradient(100deg, transparent 30%, rgba($signal, 0.1) 50%, transparent 70%);
      background-size: 200% 100%;
      background-repeat: no-repeat;
      animation: ev-mcard-sheen 1.6s ease-in-out infinite;

      td { background: transparent; }
      td:first-child { animation: ev-row-bar 1.6s ease-in-out infinite; }
      &:hover td { background: transparent; }
    }
    tbody tr.ev__table-row--checking.ev__table-row--on { background-color: $wash; }

    &--static {
      th { position: static; }
      tbody tr { cursor: default; }
      tbody tr:hover td { background: transparent; }
      td.ev__table-num { font-family: $sans; font-size: 1em; }
    }
  }

  &__table-model {
    display: flex;
    align-items: center;
    gap: 12px;

    b { display: block; font-family: $display; font-weight: 700; color: $ink; }
    small { display: block; font-size: 0.9231em; color: $ink-2; }
  }
  &__table-health { display: flex; align-items: center; gap: 8px; color: $ink-2; white-space: nowrap; }

  // ---- suite rows (Test Suite step) --------------------------------------
  // Suite cards: as many per row as fit, all the same height. Each card
  // shows everything the dataset carries — name, tags, description, how it's
  // scored (harness), its subgroups, and question / subgroup counts.
  &__slist {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
    grid-auto-rows: 1fr;
    gap: 12px;
  }

  &__srow {
    display: flex;
    flex-direction: column;
    gap: 10px;
    height: 100%;
    min-height: 236px;
    padding: 16px;
    border: 1.5px solid $line;
    border-radius: 14px;
    background: $card;
    cursor: pointer;
    transition: border-color 0.16s ease, box-shadow 0.16s ease, transform 0.16s ease, background 0.16s ease;

    &:hover { border-color: $ink-3; box-shadow: $lift; transform: translateY(-2px); }

    &--on {
      border-color: $signal;
      background: $wash;
      &:hover { border-color: $signal; }
      .ev__pilltag { background: $card; }
      .ev__srow-icon { background: $signal; border-color: $signal; color: #fff; }
      .ev__srow-side { border-top-color: rgba($signal, 0.18); }
    }
    &--empty { border-color: rgba($danger, 0.3); }
  }

  &__srow-head { display: flex; align-items: center; gap: 10px; }
  &__srow-icon {
    flex-shrink: 0;
    width: 32px;
    height: 32px;
    border-radius: 9px;
    display: grid;
    place-items: center;
    border: 1px solid $line;
    background: $paper;
    color: $ink;
    transition: background 0.16s ease, border-color 0.16s ease, color 0.16s ease;
  }
  &__srow-name {
    min-width: 0;
    font-family: $display;
    font-size: 1.2308em;
    font-weight: 700;
    color: $ink;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__srow-tags { display: flex; flex-wrap: wrap; gap: 6px; }
  &__srow-tags .ev__pilltag { display: inline-flex; align-items: center; gap: 4px; }

  &__srow-desc {
    font-size: 1.0385em;
    color: $ink-2;
    line-height: 1.5;
    display: -webkit-box;
    -webkit-box-orient: vertical;
    -webkit-line-clamp: 3;
    line-clamp: 3;
    overflow: hidden;

    &--none { color: $ink-3; font-style: italic; }
  }
  &__srow-scoring {
    font-size: 0.9615em;
    color: $ink-2;
    line-height: 1.45;
    b { font-weight: 700; color: $ink; }
  }

  &__srow-subs { display: flex; flex-direction: column; gap: 6px; }
  &__srow-k { @extend %micro; font-size: 0.6923em; color: $ink-3; }
  &__srow-chips { display: flex; flex-wrap: wrap; gap: 5px; }

  // Model cards (Models step) use this same card at the same size (they
  // inherit &__srow's minimum height; the grid below matches the suite
  // grid's column width). Locked = can't be selected until its health check
  // passes.
  &__srow--model { gap: 12px; }
  &__srow--locked {
    cursor: default;
    &:hover { border-color: $line; box-shadow: none; transform: none; }
  }
  // health badge sits at the end of the tags row
  &__srow-health { margin-left: auto; display: inline-flex; }
  &__srow-none { font-size: 0.9231em; color: $ink-3; }

  // footer: stats on the left, Preview pushed to the right
  &__srow-side {
    margin-top: auto;
    display: flex;
    align-items: flex-end;
    gap: 18px;
    padding-top: 12px;
    border-top: 1px solid $line-2;

    .ev__stat-v { font-size: 1.1538em; }
  }
  &__srow-side .ev__stat-v.ev__srow-count--empty { color: $danger; }
  &__srow-preview {
    margin-left: auto;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 6px 12px;
    border: 1px solid $line;
    border-radius: 999px;
    background: $card;
    font-family: $sans;
    font-size: 0.9231em;
    font-weight: 650;
    color: $ink-2;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $signal; color: $signal; background: $wash; }
  }

  // the Subgroups rail matches the side panel: square, bordered on all sides
  &__rail { border: 1px solid $line; border-radius: 0; }

  // ---- sectioned form rows (Scoring step) --------------------------------
  &__form { display: flex; flex-direction: column; }

  // Scoring step: the form is the main column of an &__split (judge panel
  // on the right), so it scrolls on its own.
  &__form--scroll { min-width: 0; min-height: 0; overflow-y: auto; padding-right: 8px; }
  &__split &__frow { grid-template-columns: 180px minmax(0, 1fr); gap: 22px; }
  &__form &__toolbar-refresh { margin-left: 0; }
  &__panel &__toolbar-search { flex: none; max-width: none; }
  &__panel-req { font-size: 0.9231em; font-weight: 700; color: $danger; }

  &__frow {
    display: grid;
    grid-template-columns: 200px minmax(0, 1fr);
    gap: 28px;
    padding: 22px 0;
    border-bottom: 1px solid $line-2;

    &:first-child { padding-top: 0; }
    &:last-child { border-bottom: 0; padding-bottom: 0; }
  }

  &__frow-side {
    h3 { font-family: $display; font-size: 1.1538em; font-weight: 800; color: $ink; }
    p { margin-top: 4px; font-size: 1em; color: $ink-2; line-height: 1.45; }
    .ev__metrics-actions { margin-top: 10px; }
    .ev__frow-req { margin-top: 8px; font-weight: 700; color: $danger; }
    .ev__link:disabled { opacity: 0.5; cursor: not-allowed; text-decoration: none; }
  }

  &__frow-main { min-width: 0; display: flex; flex-direction: column; gap: 12px; }

  &__inline { display: flex; flex-wrap: wrap; align-items: center; gap: 12px; }
  &__inline-note { font-size: 1em; color: $ink-2; line-height: 1.45; }

  &__num {
    width: 88px;
    border: 1.5px solid $line;
    border-radius: 10px;
    padding: 9px 12px;
    font-family: $sans;
    font-size: 1.0769em;
    font-weight: 650;
    color: $ink;
    background: $card;
    transition: border-color 0.15s ease, box-shadow 0.15s ease;

    &:focus { outline: none; border-color: $signal; box-shadow: $ring; }
  }
  &__numfield {
    flex: 1;
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 6px;

    label { font-size: 0.9231em; font-weight: 650; color: $ink; }
    .ev__num { width: 100%; }
  }
  &__inline &__numfield { flex: 0 0 120px; }

  &__formula {
    padding: 10px 12px;
    border-radius: 10px;
    background: $wash;
    font-size: 1em;
    color: $ink;
    b { font-weight: 800; }
  }

  &__sublabel {
    display: flex;
    align-items: center;
    gap: 14px;
    font-size: 0.9231em;
    font-weight: 650;
    color: $ink-2;
  }

  &__chips { flex: none; margin: 0; padding: 0; }

  &__jgrid { display: grid; grid-template-columns: repeat(auto-fill, minmax(270px, 1fr)); gap: 8px; }

  &__checkline {
    display: inline-flex;
    align-items: center;
    gap: 9px;
    padding: 0 0 10px;
    border: 0;
    background: transparent;
    font-family: $sans;
    font-size: 1em;
    font-weight: 650;
    color: $ink;
    cursor: pointer;
  }

  // ---- review -------------------------------------------------------------
  &__banner {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 24px;
    padding: 18px 20px;
    border-radius: 14px;
    background: $wash;
  }
  &__banner-title { font-family: $display; font-size: 1.3846em; font-weight: 800; color: $ink; overflow-wrap: anywhere; }
  &__banner-sub { margin-top: 4px; font-size: 1em; color: $ink-2; }
  &__banner-stats {
    flex-shrink: 0;
    display: flex;
    gap: 28px;

    div { display: flex; flex-direction: column; gap: 2px; }
    span { font-size: 0.9231em; color: $ink-2; }
    b { font-family: $display; font-size: 1.6923em; font-weight: 800; line-height: 1.1; color: $ink; }
  }

  &__sechead {
    display: flex;
    align-items: baseline;
    justify-content: space-between;
    gap: 16px;
    margin: 24px 0 10px;

    h3 { font-family: $display; font-size: 1.1538em; font-weight: 800; color: $ink; }
    span { font-size: 1em; color: $ink-2; text-align: right; }
  }
  &__secnote { margin-top: 10px; font-size: 1em; color: $ink-2; }

  &__settings { border: 1px solid $line; border-radius: 12px; overflow: hidden; }
  &__settings-row {
    display: grid;
    grid-template-columns: 140px minmax(0, 1fr) auto;
    align-items: baseline;
    gap: 16px;
    padding: 14px 16px;
    border-bottom: 1px solid $line-2;

    &:last-child { border-bottom: 0; }
  }
  &__settings-k { font-size: 1em; color: $ink-2; }
  &__settings-v { font-size: 1em; font-weight: 700; color: $ink; line-height: 1.45; overflow-wrap: anywhere; }
}

// ===========================================================================
// Dataset preview slider (Change-2) — right-to-left panel opened from a
// Test Suite card's "Preview" button. Kept as top-level (non-&__) classes
// since it's an overlay outside the .ev tree, same pattern as .ev-toast.
// ===========================================================================
.ev-preview-overlay {
  // master scale control for this subtree — cascades to .ev-preview-panel
  // and its descendants below. See $ev-base-font above.
  font-size: $ev-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  position: fixed;
  inset: 0;
  z-index: 70;
  background: rgba(10, 11, 15, 0.38);
  display: flex;
  justify-content: flex-end;
  animation: ev-fade-in 0.2s ease both;
}

.ev-preview-panel {
  width: min(440px, 100vw);
  height: 100%;
  background: $card;
  border-left: 1px solid $line;
  box-shadow: -20px 0 40px -16px rgba(0, 0, 0, 0.28);
  display: flex;
  flex-direction: column;
  min-height: 0;
  animation: ev-slide-in 0.28s cubic-bezier(0.22, 0.72, 0.16, 1) both;
}

.ev-preview-head {
  flex-shrink: 0;
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 12px;
  padding: 20px 20px 16px;
  border-bottom: 1px solid $line-2;
}

.ev-preview-head-main { min-width: 0; }

.ev-preview-eyebrow {
  @extend %micro;
  font-size: 0.7692em; color: $signal; } // 0.625rem / 0.8125rem

.ev-preview-title {
  margin-top: 6px;
  font-family: $display;
  font-size: 1.3077em; // 1.0625rem / 0.8125rem
  font-weight: 800;
  letter-spacing: -0.015em;
  color: $ink;
  line-height: 1.25;
  overflow-wrap: anywhere;
}

.ev-preview-close {
  flex-shrink: 0;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 30px;
  height: 30px;
  border: 1px solid $line;
  border-radius: 9px;
  background: $paper;
  color: $ink-2;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

  &:hover { border-color: $ink-3; color: $ink; }
}

// ---- image lightbox (MinIO object-preview style) — a centered modal for
// viewing a question's input.images[], distinct from the slide-in
// .ev-preview-* panel it's launched from. ------------------------------
.ev-imgview-overlay {
  font-size: $ev-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  position: fixed;
  inset: 0;
  z-index: 80;
  background: rgba(10, 11, 15, 0.62);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 32px;
  animation: ev-fade-in 0.18s ease both;
}

.ev-imgview-panel {
  width: min(720px, 100%);
  max-height: 100%;
  background: $ink-solid;
  border-radius: 16px;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  box-shadow: 0 30px 60px -20px rgba(0, 0, 0, 0.55);
  animation: ev-toast-in 0.22s cubic-bezier(0.22, 0.72, 0.16, 1) both;
}

.ev-imgview-head {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 12px 12px 16px;
  background: rgba(255, 255, 255, 0.04);
  border-bottom: 1px solid rgba(255, 255, 255, 0.08);
}

.ev-imgview-path {
  flex: 1;
  min-width: 0;
  display: flex;
  align-items: center;
  gap: 7px;
  font-family: $mono;
  font-size: 0.9231em; // 0.75rem / 0.8125rem
  font-weight: 600;
  color: #fff;
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;

  svg { flex-shrink: 0; color: rgba(255, 255, 255, 0.55); }
}

.ev-imgview-count {
  flex-shrink: 0;
  font-family: $mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  color: rgba(255, 255, 255, 0.55);
}

.ev-imgview-close {
  flex-shrink: 0;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  border: 1px solid rgba(255, 255, 255, 0.16);
  border-radius: 8px;
  background: transparent;
  color: rgba(255, 255, 255, 0.75);
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

  &:hover { border-color: rgba(255, 255, 255, 0.4); color: #fff; background: rgba(255, 255, 255, 0.08); }
}

.ev-imgview-body {
  position: relative;
  min-height: 260px;
  max-height: calc(100vh - 180px);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 16px;
  overflow: auto;
}

.ev-imgview-img {
  display: block;
  max-width: 100%;
  max-height: calc(100vh - 220px);
  border-radius: 6px;
  object-fit: contain;
}

.ev-imgview-status {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
  color: rgba(255, 255, 255, 0.7);
  font-size: 0.9231em; // 0.75rem / 0.8125rem

  .ev__error { color: $danger; }
}

.ev-imgview-nav {
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  z-index: 1;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 36px;
  height: 36px;
  border: 1px solid rgba(255, 255, 255, 0.16);
  border-radius: 999px;
  background: rgba(0, 0, 0, 0.35);
  color: #fff;
  cursor: pointer;
  transition: border-color 0.15s ease, background 0.15s ease, opacity 0.15s ease;

  &:hover:not(:disabled) { border-color: rgba(255, 255, 255, 0.4); background: rgba(0, 0, 0, 0.55); }
  &:disabled { opacity: 0.3; cursor: not-allowed; }
}
.ev-imgview-nav--prev { left: 12px; }
.ev-imgview-nav--next { right: 12px; }

.ev-preview-controls {
  flex-shrink: 0;
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
  gap: 14px;
  padding: 16px 20px;
  border-bottom: 1px solid $line-2;
  background: $paper;
}

.ev-preview-limit { min-width: 0; }

.ev-preview-limit-label {
  @extend %micro;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  color: $ink-2;
  display: block;
  margin-bottom: 8px;
}

.ev-preview-limit-range {
  font-family: $sans;
  letter-spacing: 0;
  text-transform: none;
  font-weight: 500;
  color: $ink-3;
}

.ev-preview-limit-controls {
  display: inline-flex;
  align-items: center;
  gap: 6px;
}

.ev-preview-stepper-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  border: 1.5px solid $line;
  border-radius: 8px;
  background: $card;
  color: $ink-2;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease;

  &:hover:not(:disabled) { border-color: $signal; color: $signal; }
  &:disabled { cursor: not-allowed; opacity: 0.4; }
}

.ev-preview-body {
  flex: 1;
  min-height: 0;
  overflow-y: auto;
  padding: 18px 20px 24px;
}

.ev-preview-skel-list,
.ev-preview-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.ev-preview-skel-card {
  display: flex;
  flex-direction: column;
  gap: 9px;
  padding: 14px 15px;
  border: 1px solid $line;
  border-radius: 12px;
  background: $paper;
}

.ev-preview-q {
  padding: 14px 15px;
  border: 1px solid $line;
  border-radius: 12px;
  background: $paper;
}

.ev-preview-q-head {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
}

.ev-preview-q-index {
  font-family: $mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  color: $signal;
  background: $wash;
  border-radius: 6px;
  padding: 2px 7px;
}

.ev-preview-q-cat {
  font-family: $mono;
  font-size: 0.8077em; // 0.65625rem / 0.8125rem
  font-weight: 600;
  color: $ink-3;
  background: $card;
  border: 1px solid $line;
  border-radius: 6px;
  padding: 2px 7px;
}

.ev-preview-q-prompt {
  font-size: 1.0385em; // 0.84375rem / 0.8125rem
  color: $ink;
  line-height: 1.5;
}

.ev-preview-q-source {
  margin-top: 4px;
  font-size: 0.9231em; // 0.75rem / 0.8125rem
  color: $ink-2;
  line-height: 1.4;

  span {
    @extend %micro;
    font-size: 0.7308em; // 0.59375rem / 0.8125rem
    color: $ink-3;
    margin-right: 6px;
  }
}

// Thumbnail-style trigger buttons for a question's input.images[] — click
// opens the MinIO image lightbox (.ev-imgview-* below) for that image.
.ev-preview-q-images {
  margin-top: 9px;
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.ev-preview-q-image-btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  max-width: 220px;
  padding: 5px 10px;
  border: 1px solid $line;
  border-radius: 8px;
  background: $card;
  color: $ink-2;
  font-family: $sans;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 600;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

  svg { flex-shrink: 0; color: $signal; }

  // Truncate a long filename rather than wrapping/overflowing the pill.
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;

  &:hover { border-color: $signal; color: $signal; background: $wash; }
}

.ev-preview-q-choices {
  margin-top: 9px;
  display: flex;
  flex-direction: column;
  gap: 5px;

  li {
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    color: $ink-2;
    padding: 6px 9px;
    border: 1px solid $line;
    border-radius: 8px;
    background: $card;
  }
}

.ev-preview-q-answer {
  margin-top: 10px;
  padding-top: 10px;
  border-top: 1px dashed $line-2;
  font-size: 1em; // 0.8125rem / 0.8125rem (base)
  font-weight: 600;
  color: $ok;

  span {
    @extend %micro;
    font-size: 0.7308em; // 0.59375rem / 0.8125rem
    color: $ink-3;
    margin-right: 7px;
  }

  // A second .ev-preview-q-answer block (e.g. Expected followed by
  // Reference) shouldn't repeat the divider directly under the first one.
  & + & {
    margin-top: 6px;
    padding-top: 0;
    border-top: 0;
  }
}

.ev-preview-q-meta {
  margin-top: 10px;
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.ev-preview-q-meta-item {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  color: $ink-2;
  background: $paper;
  border: 1px solid $line;
  border-radius: 6px;
  padding: 3px 8px;

  b {
    @extend %micro;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    color: $ink-3;
    font-weight: 700;
  }
}

@keyframes ev-fade-in {
  from { opacity: 0; }
  to { opacity: 1; }
}
@keyframes ev-slide-in {
  from { transform: translateX(100%); }
  to { transform: translateX(0); }
}

// ---- toast (fixed-dark chip, same reasoning as .ev__btn--primary) ---------
.ev-toast {
  // master scale control for this subtree — see $ev-base-font above
  font-size: $ev-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  position: fixed;
  right: 24px;
  bottom: 24px;
  z-index: 60;
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 14px 18px 14px 14px;
  background: $ink-solid;
  color: #fff;
  border-radius: 14px;
  box-shadow: 0 20px 40px -16px rgba(0, 0, 0, 0.5);
  animation: ev-toast-in 0.32s cubic-bezier(0.22, 0.72, 0.16, 1) both;

  &__icon {
    width: 34px;
    height: 34px;
    border-radius: 9px;
    display: grid;
    place-items: center;
    background: rgba($ok, 0.2);
    color: $ink-solid-ok;
  }
  &__title { font-family: $display; font-weight: 700; font-size: 1.0385em; } // 0.84375rem / 0.8125rem
  &__sub { font-size: 0.9231em; color: rgba(255, 255, 255, 0.6); margin-top: 1px; } // 0.75rem / 0.8125rem
}

// ---- keyframes ------------------------------------------------------------
@keyframes ev-spin { to { transform: rotate(360deg); } }
@keyframes ev-shimmer {
  0% { background-position: 100% 50%; }
  100% { background-position: 0 50%; }
}
@keyframes ev-pulse {
  0%, 100% { box-shadow: 0 0 0 0 rgba(43, 43, 245, 0.5); }
  50% { box-shadow: 0 0 0 4px rgba(43, 43, 245, 0); }
}
@keyframes ev-mcard-pulse {
  0%, 100% { box-shadow: 0 0 0 0 rgba(43, 43, 245, 0.16); }
  50% { box-shadow: 0 0 0 5px rgba(43, 43, 245, 0); }
}
@keyframes ev-row-bar {
  0%, 100% { box-shadow: inset 3px 0 0 rgba(43, 43, 245, 0.25); }
  50% { box-shadow: inset 3px 0 0 rgba(43, 43, 245, 0.9); }
}
@keyframes ev-mcard-sheen {
  0% { background-position: 140% 0; }
  100% { background-position: -40% 0; }
}
@keyframes ev-rise {
  from { opacity: 0; transform: translateY(8px); }
  to { opacity: 1; transform: none; }
}
@keyframes ev-toast-in {
  from { opacity: 0; transform: translateY(12px) scale(0.98); }
  to { opacity: 1; transform: none; }
}

// ---- responsive -----------------------------------------------------------
@media (max-width: 1040px) {
  .ev__shell { grid-template-columns: 1fr; }
  .ev__manifest { display: none; }
  .ev__suite, .ev__metrics { grid-template-columns: 1fr; }
  .ev__samples-instruction-row { grid-template-columns: 1fr; }
  .ev__split, .ev__frow, .ev__choice-row, .ev__namegrid { grid-template-columns: 1fr; }
  .ev__asetup-grid { grid-template-columns: 1fr; }
  .ev__about-cols { grid-template-columns: 1fr; }
  .ev__about-col + .ev__about-col { border-left: 0; border-top: 1px solid $line; }
  .ev__hprev { position: static; }
  .ev__split { grid-template-rows: none; }
  .ev__frow { gap: 12px; }
  .ev__rail, .ev__judge { max-height: 15rem; }
}

@media (max-width: 640px) {
  .ev__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .page { padding: 16px 14px 22px; }
  .ev__stage-head { padding: 18px 18px 15px; }
  .ev__stage-body { padding: 18px; }
  .ev__footer { padding: 14px 18px; }
  .ev__fw-grid, .ev__choice-grid { grid-template-columns: 1fr; }
  .ev__blocker { display: none; }
  .ev__banner { flex-direction: column; align-items: flex-start; }
  .ev__settings-row { grid-template-columns: minmax(0, 1fr) auto; }
  .ev__settings-k { grid-column: 1 / -1; }
  .ev__summary { grid-template-columns: 1fr; }
  .ev__hint { display: none; }
  .ev__name-input { font-size: 1.6923em; } // 1.375rem / 0.8125rem
  .ev__step-toolbar { flex-direction: column; align-items: stretch; }
  .ev__toolbar-search { max-width: none; }
  .ev-preview-panel { width: 100vw; }
  .ev-preview-controls { flex-direction: column; align-items: stretch; }
}

@media (prefers-reduced-motion: reduce) {
  .ev-preview-overlay, .ev-preview-panel { animation: none !important; }
  .ev__skel-block { animation: none !important; }
}

@media (prefers-reduced-motion: reduce) {
  .ev *, .ev-toast { animation: none !important; transition: none !important; }
}
