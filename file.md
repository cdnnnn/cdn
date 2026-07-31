//types.ts
export type EvalTypeId = 'model' | 'agent' | 'rag';

export interface EvalType {
  id: EvalTypeId;
  title: string;
  desc: string;
  badge: string;
}

export interface Provider {
  id: string;
  name: string;
  status: 'connected' | 'not_connected';
  modelCount: number;
  logo: string;
  desc: string;
}

export type ModelDifficulty = 'easy' | 'medium' | 'hard';

export interface ModelInfo {
  id: string;
  name: string;
  provider: string;
  providerId: string;
  capabilities: string[];
  contextWindow: string;
  pricing: string;
  speedRating: string;
  accuracyScore: number;
  agentScore: number;
  /** High-level grouping used by the Models filter panel, e.g. "Reasoning", "Coding" */
  category: string;
  /** Evaluation type this model is best suited for */
  eval_type: EvalTypeId;
  difficulty: ModelDifficulty;
}

export interface ModelFilters {
  category: string[];
  difficulty: ModelDifficulty[];
  eval_type: EvalTypeId[];
}

export interface DatasetSubgroup {
  id: string;
  name: string;
  count: number;
  sampleQuestion: string;
}

export interface TestSuite {
  id: string;
  name: string;
  category: string;
  questions: number;
  language: string;
  task: string;
  difficulty: string;
  version: string;
  maintainer: string;
  description: string;
  recommendedFor: EvalTypeId[];
  featured: boolean;
  subgroups: DatasetSubgroup[];
}

export interface Metric {
  id: string;
  name: string;
  tooltip: string;
  defaultChecked: boolean;
}

export type AgentFramework = 'hermes' | 'langgraph';

export interface EvaluationDraft {
  name: string;
  type: EvalTypeId | null;
  agentFramework: AgentFramework | null;
  providers: string[];
  models: string[];
  dataset: string | null;
  /** Selected subgroup ids for the chosen dataset, keyed by dataset id */
  datasetSubgroups: string[];
  metrics: string[];
}

export const WIZARD_STEPS = ['Name', 'Type', 'Providers', 'Models', 'Test Suite', 'Metrics', 'Review'] as const;














//data.ts
import type { EvalType, Metric, ModelFilters, ModelInfo, Provider, TestSuite } from './types';

export const EVAL_TYPES: EvalType[] = [
  {
    id: 'model',
    title: 'General Chat & Text (AI Model)',
    desc: 'Evaluate base model knowledge, summarization quality, and conversation tone across standardized test suites.',
    badge: 'Fast Evaluation',
  },
  {
    id: 'agent',
    title: 'Autonomous Workflow (Agent Evaluation)',
    desc: 'Test autonomous agents on multi-step tool execution, function calling, and programmatic workflow accuracy.',
    badge: 'Recommended for Automation',
  },
  {
    id: 'rag',
    title: 'Document Search & Answering (Knowledge / RAG)',
    desc: 'Measure how accurately AI models retrieve information from documents without generating incorrect answers.',
    badge: 'High Precision',
  },
];

export const PROVIDERS: Provider[] = [
  { id: 'openai', name: 'OpenAI', status: 'connected', modelCount: 8, logo: 'O', desc: 'Industry benchmark provider offering flagship chat and reasoning models.' },
  { id: 'anthropic', name: 'Anthropic', status: 'connected', modelCount: 5, logo: 'A', desc: 'Safety-first reasoning models optimized for long context and tool use.' },
  { id: 'together', name: 'Together AI', status: 'connected', modelCount: 14, logo: 'T', desc: 'High-performance infrastructure hosting open-weight agent models.' },
  { id: 'groq', name: 'Groq Cloud', status: 'connected', modelCount: 6, logo: 'G', desc: 'Ultra-low response time LPU inference engine for instant generation.' },
  { id: 'gemini', name: 'Google Gemini', status: 'connected', modelCount: 6, logo: 'G', desc: 'Massive context window models with native multi-modal reasoning.' },
  { id: 'openrouter', name: 'OpenRouter', status: 'not_connected', modelCount: 45, logo: 'R', desc: 'Unified routing engine with failover access to open and proprietary models.' },
  { id: 'pado', name: 'PADO Service', status: 'connected', modelCount: 5, logo: 'P', desc: 'Curated hosting for the GLM family of open-weight reasoning models.' },
  { id: 'custom', name: 'Custom Service', status: 'connected', modelCount: 6, logo: 'C', desc: 'Bring your own self-hosted or custom-deployed model endpoints.' },
];

export const MODELS: ModelInfo[] = [
  { id: 'm-1', name: 'Model Alpha Agent', provider: 'Together AI', providerId: 'together', capabilities: ['Tool Calling', 'Autonomous Agent', 'JSON Mode'], contextWindow: '128k tokens', pricing: '$0.70 / 1M tokens', speedRating: 'Ultra Fast (85 t/s)', accuracyScore: 94.8, agentScore: 97.5, category: 'Agentic', eval_type: 'agent', difficulty: 'hard' },
  { id: 'm-2', name: 'Model Delta Agent v2', provider: 'Anthropic', providerId: 'anthropic', capabilities: ['Tool Calling', 'Deep Reasoning', 'Vision'], contextWindow: '200k tokens', pricing: '$3.00 / 1M tokens', speedRating: 'Fast (55 t/s)', accuracyScore: 95.4, agentScore: 96.2, category: 'Agentic', eval_type: 'agent', difficulty: 'hard' },
  { id: 'm-3', name: 'Model Gamma Agent', provider: 'OpenAI', providerId: 'openai', capabilities: ['Tool Calling', 'Multimodal Vision', 'Reasoning'], contextWindow: '128k tokens', pricing: '$2.50 / 1M tokens', speedRating: 'Fast (65 t/s)', accuracyScore: 94.2, agentScore: 95.0, category: 'Agentic', eval_type: 'agent', difficulty: 'medium' },
  { id: 'm-4', name: 'Model Epsilon Reasoning', provider: 'Together AI', providerId: 'together', capabilities: ['Deep Math', 'Advanced Logic', 'Code Generation'], contextWindow: '64k tokens', pricing: '$0.55 / 1M tokens', speedRating: 'Medium (42 t/s)', accuracyScore: 96.1, agentScore: 91.8, category: 'Reasoning', eval_type: 'model', difficulty: 'hard' },
  { id: 'm-5', name: 'Model Zeta Instruct', provider: 'Groq Cloud', providerId: 'groq', capabilities: ['Instant Response', 'General Chat', 'Tool Calling'], contextWindow: '128k tokens', pricing: '$0.59 / 1M tokens', speedRating: 'Instant (280 t/s)', accuracyScore: 91.5, agentScore: 89.4, category: 'General Chat', eval_type: 'model', difficulty: 'easy' },
  { id: 'm-6', name: 'Model Theta Long-Context', provider: 'Google Gemini', providerId: 'gemini', capabilities: ['2M Tokens', 'Multi-modal Video', 'Document Analysis'], contextWindow: '2,000,000 tokens', pricing: '$2.50 / 1M tokens', speedRating: 'Fast (50 t/s)', accuracyScore: 93.7, agentScore: 91.0, category: 'Long Context', eval_type: 'rag', difficulty: 'medium' },
  { id: 'm-7', name: 'Model Theta Flash', provider: 'Google Gemini', providerId: 'gemini', capabilities: ['Vision', 'Tool Calling', 'Streaming'], contextWindow: '1,000,000 tokens', pricing: '$0.10 / 1M tokens', speedRating: 'Ultra Fast (120 t/s)', accuracyScore: 92.4, agentScore: 94.3, category: 'Agentic', eval_type: 'agent', difficulty: 'easy' },
  { id: 'm-8', name: 'Model Delta Opus', provider: 'Anthropic', providerId: 'anthropic', capabilities: ['Deep Reasoning', 'Long Context', 'Vision'], contextWindow: '200k tokens', pricing: '$15.00 / 1M tokens', speedRating: 'Medium (25 t/s)', accuracyScore: 97.2, agentScore: 94.8, category: 'Reasoning', eval_type: 'model', difficulty: 'hard' },
  { id: 'm-9', name: 'Model Eta Instruct', provider: 'Together AI', providerId: 'together', capabilities: ['Coding', 'Math', 'Multilingual'], contextWindow: '128k tokens', pricing: '$0.90 / 1M tokens', speedRating: 'Fast (65 t/s)', accuracyScore: 93.8, agentScore: 92.1, category: 'Coding', eval_type: 'model', difficulty: 'medium' },
  { id: 'm-10', name: 'Model Gamma Mini', provider: 'OpenAI', providerId: 'openai', capabilities: ['Vision', 'Tool Calling', 'Streaming'], contextWindow: '128k tokens', pricing: '$0.15 / 1M tokens', speedRating: 'Ultra Fast (90 t/s)', accuracyScore: 88.5, agentScore: 87.2, category: 'General Chat', eval_type: 'model', difficulty: 'easy' },
  { id: 'm-11', name: 'GLM5.1', provider: 'PADO Service', providerId: 'pado', capabilities: ['Deep Reasoning', 'Tool Calling', 'Multilingual'], contextWindow: '128k tokens', pricing: '$0.80 / 1M tokens', speedRating: 'Fast (58 t/s)', accuracyScore: 94.5, agentScore: 92.6, category: 'Reasoning', eval_type: 'model', difficulty: 'medium' },
  { id: 'm-12', name: 'Qwen3', provider: 'Custom Service', providerId: 'custom', capabilities: ['Coding', 'Math', 'Self-hosted'], contextWindow: '128k tokens', pricing: 'Self-hosted', speedRating: 'Fast (60 t/s)', accuracyScore: 92.9, agentScore: 90.3, category: 'Coding', eval_type: 'model', difficulty: 'medium' },
];

/** Filter option lists surfaced in the Models step filter panel. */
export const FILTERS: ModelFilters = {
  category: Array.from(new Set(MODELS.map((m) => m.category))),
  difficulty: ['easy', 'medium', 'hard'],
  eval_type: ['model', 'agent', 'rag'],
};

export const TEST_SUITES: TestSuite[] = [
  {
    id: 'ts-agent',
    name: 'Autonomous Tool & Workflow Suite',
    category: 'Agents',
    questions: 420,
    language: 'English / JSON',
    task: 'Multi-step API Execution & Self-Correction',
    difficulty: 'Expert',
    version: 'v3.2',
    maintainer: 'SemcoEval Labs',
    description: 'Evaluates multi-step tool calling, nested JSON schema generation, error recovery, and workflow execution.',
    recommendedFor: ['agent'],
    featured: true,
    subgroups: [
      { id: 'ts-agent-api', name: 'API Execution', count: 180, sampleQuestion: 'Given this OpenAPI spec, call the /orders endpoint to fetch all orders placed in the last 7 days and return them as JSON.' },
      { id: 'ts-agent-selfcorrect', name: 'Self-Correction', count: 140, sampleQuestion: 'Your last tool call failed with a 422 error because "customer_id" was missing. Diagnose the failure and retry with a corrected payload.' },
      { id: 'ts-agent-planning', name: 'Multi-step Planning', count: 100, sampleQuestion: 'Plan and execute the steps needed to onboard a new customer: create their account, provision an API key, and send a welcome email.' },
    ],
  },
  {
    id: 'ts-swe',
    name: 'SWE-bench Verified Software Engineer Suite',
    category: 'Coding',
    questions: 500,
    language: 'Python / JS / TS',
    task: 'Autonomous GitHub Issue Resolution',
    difficulty: 'Expert',
    version: '2026.2',
    maintainer: 'Open Source AI Labs',
    description: 'Measures ability to independently diagnose bug reports, run local unit tests, and commit working patches.',
    recommendedFor: ['agent', 'model'],
    featured: true,
    subgroups: [
      { id: 'ts-swe-diagnosis', name: 'Bug Diagnosis', count: 200, sampleQuestion: 'Users report that the checkout page crashes intermittently. Given the stack trace and recent commits, identify the root cause of the bug.' },
      { id: 'ts-swe-tests', name: 'Unit Test Writing', count: 150, sampleQuestion: 'Write unit tests covering the edge cases for this `calculate_discount()` function, including zero, negative, and boundary values.' },
      { id: 'ts-swe-patch', name: 'Patch Generation', count: 150, sampleQuestion: 'Given this failing test and the linked GitHub issue, generate a minimal patch that fixes the bug without breaking existing tests.' },
    ],
  },
  {
    id: 'ts-mmlu',
    name: 'MMLU-Pro General Knowledge & Reasoning Suite',
    category: 'General',
    questions: 1400,
    language: 'Multilingual (24 Languages)',
    task: 'Academic & Professional Problem Solving',
    difficulty: 'High',
    version: 'Pro-v2',
    maintainer: 'Stanford CRFM',
    description: 'Comprehensive multi-domain benchmark covering 57 disciplines including law, physics, and ethics.',
    recommendedFor: ['model'],
    featured: false,
    subgroups: [
      { id: 'ts-mmlu-stem', name: 'STEM', count: 500, sampleQuestion: 'A capacitor with capacitance 4µF is charged to 12V. How much energy, in joules, is stored in the capacitor?' },
      { id: 'ts-mmlu-humanities', name: 'Humanities', count: 450, sampleQuestion: 'Which philosophical movement most directly influenced the categorical imperative described by Immanuel Kant?' },
      { id: 'ts-mmlu-social', name: 'Social Sciences', count: 450, sampleQuestion: 'According to supply-side economics, how would a reduction in marginal tax rates be expected to affect long-run aggregate supply?' },
    ],
  },
  {
    id: 'ts-ragas',
    name: 'Ragas Document Factual Recall & Faithfulness',
    category: 'RAG',
    questions: 350,
    language: 'English',
    task: 'Context Precision & Hallucination Defense',
    difficulty: 'Medium',
    version: '1.4',
    maintainer: 'Ragas Ecosystem',
    description: 'Evaluates retrieval accuracy and ensures answers derive strictly from verified documents.',
    recommendedFor: ['rag'],
    featured: true,
    subgroups: [
      { id: 'ts-ragas-precision', name: 'Context Precision', count: 150, sampleQuestion: 'Given these 5 retrieved passages about our refund policy, which ones are actually relevant to answering "How long do refunds take?"' },
      { id: 'ts-ragas-faithfulness', name: 'Faithfulness', count: 120, sampleQuestion: 'Does the generated answer "Refunds are processed within 3-5 business days" accurately reflect what is stated in the retrieved policy document?' },
      { id: 'ts-ragas-hallucination', name: 'Hallucination Defense', count: 80, sampleQuestion: 'The retrieved documents do not mention international shipping costs. Does the model correctly decline to answer rather than fabricating a figure?' },
    ],
  },
  {
    id: 'ts-finance',
    name: 'Corporate Finance & Audit Math Suite',
    category: 'Finance',
    questions: 280,
    language: 'English / Numeric',
    task: 'Exact Mathematical Deduction & Regulation',
    difficulty: 'Advanced',
    version: '2026-Q1',
    maintainer: 'SemcoEval Labs',
    description: 'Validates exact calculation precision, formula interpretation, and compliance audit reasoning.',
    recommendedFor: ['model', 'agent'],
    featured: false,
    subgroups: [
      { id: 'ts-finance-audit', name: 'Audit Math', count: 120, sampleQuestion: 'A company reports $2.4M in revenue and $1.85M in COGS. Calculate the gross margin percentage and flag any rounding discrepancies.' },
      { id: 'ts-finance-regulation', name: 'Regulation', count: 100, sampleQuestion: 'Under SOX Section 404, what internal control documentation must a public company maintain for its financial reporting process?' },
      { id: 'ts-finance-formula', name: 'Formula Interpretation', count: 60, sampleQuestion: 'Given this Excel formula referencing quarterly revenue cells, explain what value it will return and identify any circular reference errors.' },
    ],
  },
  {
    id: 'ts-health',
    name: 'Clinical Diagnostic Safety & Care Benchmark',
    category: 'Healthcare',
    questions: 310,
    language: 'English / Medical',
    task: 'Diagnostic Logic & Patient Guardrails',
    difficulty: 'Expert',
    version: 'Med-2.1',
    maintainer: 'HealthAI Foundation',
    description: 'Assesses diagnostic recommendation accuracy and safety compliance in health triage.',
    recommendedFor: ['model'],
    featured: false,
    subgroups: [
      { id: 'ts-health-diagnosis', name: 'Diagnostic Logic', count: 150, sampleQuestion: 'A 54-year-old patient presents with sudden chest pain radiating to the left arm and shortness of breath. What is the most appropriate immediate next step?' },
      { id: 'ts-health-guardrails', name: 'Patient Guardrails', count: 100, sampleQuestion: 'A user asks for the exact dosage of a prescription-only medication for a symptom they describe. Does the model correctly decline and recommend consulting a clinician?' },
      { id: 'ts-health-triage', name: 'Triage Safety', count: 60, sampleQuestion: 'Given these reported symptoms, classify the case as emergency, urgent, or routine care, and justify the triage level.' },
    ],
  },
];

export const METRICS: { universal: Metric[]; model: Metric[]; agent: Metric[]; rag: Metric[] } = {
  universal: [
    { id: 'accuracy', name: 'Accuracy', tooltip: 'Percentage of correct answers or successful task completions.', defaultChecked: true },
    { id: 'latency', name: 'Response Time', tooltip: 'Average time to generate a complete response (seconds).', defaultChecked: true },
    { id: 'cost', name: 'Cost Efficiency', tooltip: 'Cost per 1,000 API calls at provider pricing.', defaultChecked: true },
    { id: 'safety', name: 'Safety Score', tooltip: 'Resistance to jailbreaks, prompt injection, and harmful outputs.', defaultChecked: false },
  ],
  model: [
    { id: 'fluency', name: 'Fluency & Coherence', tooltip: 'How natural and well-structured the generated text is.', defaultChecked: true },
    { id: 'instruction_following', name: 'Instruction Following', tooltip: 'How well the model adheres to specific instructions and constraints.', defaultChecked: true },
    { id: 'reasoning', name: 'Reasoning Quality', tooltip: 'Logical consistency, chain-of-thought clarity, and problem decomposition.', defaultChecked: true },
    { id: 'factuality', name: 'Factual Accuracy', tooltip: 'Correctness of factual claims based on world knowledge.', defaultChecked: false },
    { id: 'helpfulness', name: 'Helpfulness', tooltip: "How useful and relevant the response is to the user's query.", defaultChecked: false },
  ],
  agent: [
    { id: 'tool_success', name: 'Tool Calling Success', tooltip: 'Percentage of function/API calls with correct syntax and parameters.', defaultChecked: true },
    { id: 'task_completion', name: 'Task Completion Rate', tooltip: 'Percentage of multi-step tasks completed successfully end-to-end.', defaultChecked: true },
    { id: 'action_accuracy', name: 'Action Sequencing', tooltip: 'Correct ordering and logic of multi-step action plans.', defaultChecked: true },
    { id: 'error_recovery', name: 'Error Recovery', tooltip: 'Ability to detect failures and self-correct without human intervention.', defaultChecked: true },
    { id: 'planning', name: 'Planning Quality', tooltip: 'Quality of task decomposition and strategic planning.', defaultChecked: false },
  ],
  rag: [
    { id: 'faithfulness', name: 'Faithfulness', tooltip: 'Does the answer accurately reflect the retrieved context without adding unsupported claims?', defaultChecked: true },
    { id: 'answer_relevance', name: 'Answer Relevance', tooltip: 'How well the generated answer addresses the original question.', defaultChecked: true },
    { id: 'context_precision', name: 'Context Precision', tooltip: 'Are the retrieved documents relevant to the question?', defaultChecked: true },
    { id: 'groundedness', name: 'Groundedness', tooltip: 'Is every claim in the answer supported by the source documents?', defaultChecked: true },
    { id: 'hallucination', name: 'Hallucination Rate', tooltip: 'Percentage of fabricated facts not present in retrieved context.', defaultChecked: true },
  ],
};

export const SUGGESTED_NAMES = ['Agent Tool Calling Test', 'Support Bot Comparison', 'Code Generation Test'];




















//Datasets.tsx
import { useMemo, useState, type FC } from 'react';
import { useNavigate } from 'react-router-dom';
import {
  Upload,
  Database,
  Star,
  LayoutGrid,
  Bot,
  Code2,
  BookOpen,
  Search,
  DollarSign,
  Globe,
  User,
  X,
  ListChecks,
} from 'lucide-react';
import { TEST_SUITES } from '../RunEvaluation/data';
import './Datasets.scss';

const CATEGORIES = [
  { value: 'All', icon: LayoutGrid },
  { value: 'Agents', icon: Bot },
  { value: 'Coding', icon: Code2 },
  { value: 'General', icon: BookOpen },
  { value: 'RAG', icon: Search },
  { value: 'Finance', icon: DollarSign },
] as const;

const CATEGORY_TINTS: Record<string, 'blue' | 'violet' | 'amber' | 'jade' | 'rose'> = {
  Agents: 'violet',
  Coding: 'blue',
  General: 'jade',
  RAG: 'blue',
  Finance: 'amber',
};

const DIFFICULTY_TINTS: Record<string, 'blue' | 'violet' | 'amber' | 'jade' | 'rose'> = {
  Medium: 'jade',
  High: 'blue',
  Advanced: 'violet',
  Expert: 'rose',
};

const Datasets: FC = () => {
  const navigate = useNavigate();
  const [query, setQuery] = useState('');
  const [category, setCategory] = useState('All');
  const [activeId, setActiveId] = useState(TEST_SUITES[0]?.id ?? null);

  const filtered = useMemo(
    () =>
      TEST_SUITES.filter((d) => {
        if (category !== 'All' && d.category !== category) return false;
        if (query && !d.name.toLowerCase().includes(query.toLowerCase()) && !d.description.toLowerCase().includes(query.toLowerCase())) {
          return false;
        }
        return true;
      }),
    [query, category]
  );

  const active = useMemo(
    () => filtered.find((d) => d.id === activeId) ?? filtered[0] ?? null,
    [activeId, filtered]
  );

  return (
    <div className="datasets-page">
      <div className="datasets-page__header">
        <div className="datasets-page__header-left">
          <p className="datasets-page__header-eyebrow">Test suite library</p>
          <h1 className="datasets-page__title">Test Suites</h1>
          <p className="datasets-page__subtitle">Benchmark datasets and custom tests</p>
        </div>

        <div className="datasets-page__header-right">
          <div className="datasets-page__header-meta">
            <Database size={13} />
            {TEST_SUITES.length} suites available
          </div>
          <button type="button" className="datasets-page__btn datasets-page__btn--primary" onClick={() => navigate('/app/run-evaluation')}>
            <Upload size={14} strokeWidth={2.25} /> Upload
          </button>
        </div>
      </div>

      <div className="datasets-page__toolbar">
        <div className="datasets-page__search">
          <Search size={15} />
          <input type="text" placeholder="Search test suites..." value={query} onChange={(e) => setQuery(e.target.value)} />
          {query && (
            <button type="button" className="datasets-page__search-clear" onClick={() => setQuery('')} aria-label="Clear search">
              <X size={13} />
            </button>
          )}
        </div>

        <div className="datasets-page__seg">
          {CATEGORIES.map((c) => (
            <button
              key={c.value}
              type="button"
              className={`datasets-page__seg-item${category === c.value ? ' datasets-page__seg-item--active' : ''}`}
              onClick={() => setCategory(c.value)}
            >
              <c.icon size={13} strokeWidth={2.25} />
              {c.value}
            </button>
          ))}
        </div>
      </div>

      <div className="datasets-page__split">
        <aside className="datasets-page__list">
          {filtered.map((d) => {
            const isActive = active?.id === d.id;
            const catTint = CATEGORY_TINTS[d.category] ?? 'blue';
            return (
              <button
                key={d.id}
                type="button"
                className={`datasets-page__row${isActive ? ' datasets-page__row--active' : ''}`}
                onClick={() => setActiveId(d.id)}
              >
                <span className="datasets-page__row-name">{d.name}</span>
                <span className="datasets-page__row-meta">
                  <span className={`datasets-page__tag datasets-page__tag--${catTint}`}>{d.category}</span>
                  <span className="datasets-page__row-count n">{d.questions.toLocaleString()} q</span>
                </span>
              </button>
            );
          })}

          {filtered.length === 0 && (
            <div className="datasets-page__empty datasets-page__empty--list">
              <Database size={20} />
              <p>No test suites in this category.</p>
            </div>
          )}
        </aside>

        <section className="datasets-page__detail">
          {active ? (
            <>
              {(() => {
                const catIcon = CATEGORIES.find((c) => c.value === active.category)?.icon ?? Database;
                const catTint = CATEGORY_TINTS[active.category] ?? 'blue';
                const diffTint = DIFFICULTY_TINTS[active.difficulty] ?? 'blue';
                const CatIcon = catIcon;

                return (
                  <>
                    <div className="datasets-page__detail-head">
                      <div>
                        <h2 className="datasets-page__name">{active.name}</h2>
                        <p className="datasets-page__desc">{active.description}</p>
                      </div>
                    </div>

                    <div className="datasets-page__tags">
                      <span className={`datasets-page__tag datasets-page__tag--${catTint}`}>
                        <CatIcon size={11} strokeWidth={2.5} />
                        {active.category}
                      </span>
                      <span className={`datasets-page__tag datasets-page__tag--${diffTint}`}>{active.difficulty}</span>
                      {active.featured && (
                        <span className="datasets-page__tag datasets-page__tag--featured">
                          <Star size={11} strokeWidth={2.5} />
                          Featured
                        </span>
                      )}
                    </div>

                    <div className="datasets-page__stats">
                      <div className="datasets-page__stat">
                        <span className="datasets-page__stat-value n">{active.questions.toLocaleString()}</span>
                        <span className="datasets-page__stat-label">Questions</span>
                      </div>
                      <div className="datasets-page__stat">
                        <span className="datasets-page__stat-value n">{active.subgroups.length}</span>
                        <span className="datasets-page__stat-label">Subcategories</span>
                      </div>
                      <div className="datasets-page__stat">
                        <span className="datasets-page__stat-value">{active.version}</span>
                        <span className="datasets-page__stat-label">Version</span>
                      </div>
                    </div>

                    <div className="datasets-page__meta-row">
                      <span className="datasets-page__meta-item">
                        <Globe size={12} /> {active.language}
                      </span>
                      <span className="datasets-page__meta-item">
                        <User size={12} /> {active.maintainer}
                      </span>
                      <span className="datasets-page__meta-item">
                        <ListChecks size={12} /> {active.task}
                      </span>
                    </div>

                    {/* ---------- question preview ---------- */}
                    <div className="datasets-page__group-head">
                      <h3>Question breakdown</h3>
                      <div className="datasets-page__group-line" />
                    </div>

                    <div className="datasets-page__preview-list">
                      {active.subgroups.map((s) => {
                        const pct = Math.round((s.count / active.questions) * 100);
                        return (
                          <div className="datasets-page__preview-item" key={s.id}>
                            <div className="datasets-page__preview-item-top">
                              <span className="datasets-page__preview-item-name">{s.name}</span>
                              <span className="datasets-page__preview-item-count n">{s.count.toLocaleString()} questions</span>
                            </div>
                            <div className="datasets-page__preview-bar-wrap">
                              <div className="datasets-page__preview-bar" style={{ width: `${pct}%` }} />
                            </div>
                            <p className="datasets-page__preview-item-sample">
                              <span className="datasets-page__preview-item-sample-label">Sample</span>
                              &ldquo;{s.sampleQuestion}&rdquo;
                            </p>
                          </div>
                        );
                      })}
                    </div>
                  </>
                );
              })()}
            </>
          ) : (
            <p className="datasets-page__empty">Select a test suite to preview its questions.</p>
          )}
        </section>
      </div>
    </div>
  );
};

export default Datasets;























//Datasets.scss
@use '../../../styles/variables' as *;

.datasets-page {
  display: flex;
  flex-direction: column;
  height: calc(100vh - 166px);
  min-height: 0;
  gap: 16px;

  /* ---------- header ---------- */
  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding-bottom: 18px;
    margin-bottom: 2px;
    border-bottom: 1px solid $border-subtle;
  }

  &__header-left {
    display: flex;
    flex-direction: column;
  }

  &__header-right {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 10px;
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
  }

  &__title {
    font-size: 21px;
    font-weight: 800;
    letter-spacing: -0.02em;
    color: $text-primary;
  }

  &__subtitle {
    margin-top: 3px;
    color: $text-secondary;
    font-size: 0.84375rem;
  }

  &__btn {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    font-family: $font-body;
    font-size: 0.8125rem;
    font-weight: 600;
    padding: 9px 14px;
    border-radius: 8px;
    border: 1px solid transparent;
    cursor: pointer;
    white-space: nowrap;
    transition: background 0.14s ease, border-color 0.14s ease;

    &--primary {
      background: $primary;
      border-color: $primary;
      color: #fff;

      &:hover {
        background: $primary-hover;
        border-color: $primary-hover;
      }
    }
  }

  /* ---------- toolbar: search + category segment ---------- */
  &__toolbar {
    flex-shrink: 0;
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
  }

  &__search {
    display: flex;
    align-items: center;
    gap: 9px;
    width: 300px;
    max-width: 100%;
    border: 1px solid $border-default;
    border-radius: 10px;
    padding: 9px 12px;
    background: $bg-main;
    color: $text-tertiary;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &:focus-within {
      border-color: $primary;
      box-shadow: 0 0 0 3px $primary-light;
    }

    input {
      flex: 1;
      border: none;
      outline: none;
      font-size: 0.8125rem;
      color: $text-primary;
      background: transparent;
      font-family: $font-body;
      min-width: 0;

      &::placeholder {
        color: $text-tertiary;
      }
    }
  }

  &__search-clear {
    flex-shrink: 0;
    width: 18px;
    height: 18px;
    border-radius: 50%;
    border: none;
    background: $bg-inset;
    color: $text-tertiary;
    display: grid;
    place-items: center;
    cursor: pointer;
    transition: background 0.14s ease, color 0.14s ease;

    &:hover {
      background: $border-default;
      color: $text-primary;
    }
  }

  &__seg {
    display: inline-flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 2px;
    padding: 3px;
    border: 1px solid $border-subtle;
    border-radius: 11px;
    background: $bg-subtle;
  }

  &__seg-item {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-family: $font-body;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-tertiary;
    background: transparent;
    border: none;
    border-radius: 8px;
    padding: 7px 12px;
    cursor: pointer;
    transition: background 0.14s ease, color 0.14s ease, box-shadow 0.14s ease;

    svg {
      opacity: 0.8;
    }

    &:hover {
      color: $text-primary;
    }

    &--active {
      background: $bg-main;
      color: $primary;
      box-shadow: $shadow-xs;

      svg {
        opacity: 1;
      }
    }
  }

  /* ---------- master-detail split ---------- */
  &__split {
    flex: 1;
    min-height: 0;
    display: grid;
    grid-template-columns: 320px 1fr;
    border: 1px solid $border-default;
    border-radius: 14px;
    overflow: hidden;
    background: $bg-main;
  }

  /* ---------- left list ---------- */
  &__list {
    border-right: 1px solid $border-default;
    background: $bg-subtle;
    overflow-y: auto;
  }

  &__row {
    width: 100%;
    display: flex;
    flex-direction: column;
    gap: 6px;
    text-align: left;
    padding: 14px 16px;
    border: none;
    border-bottom: 1px solid $border-subtle;
    background: transparent;
    cursor: pointer;
    transition: background 0.14s ease;

    &:hover {
      background: $bg-inset;
    }

    &--active {
      background: $bg-main;
      border-left: 3px solid $primary;
      padding-left: 13px;

      &:hover {
        background: $bg-main;
      }
    }
  }

  &__row-name {
    font-size: 0.8125rem;
    font-weight: 700;
    letter-spacing: -0.01em;
    line-height: 1.4;
    color: $text-primary;
  }

  &__row-meta {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__row-count {
    font-size: 0.6875rem;
    font-weight: 600;
    color: $text-tertiary;
    flex-shrink: 0;
  }

  /* ---------- right detail ---------- */
  &__detail {
    overflow-y: auto;
    padding: 28px 32px;
  }

  &__detail-head {
    margin-bottom: 14px;
  }

  &__name {
    margin: 0 0 6px;
    font-size: 1.1875rem;
    font-weight: 800;
    letter-spacing: -0.015em;
    line-height: 1.35;
    color: $text-primary;
  }

  &__desc {
    margin: 0;
    font-size: 0.875rem;
    line-height: 1.6;
    color: $text-secondary;
    max-width: 720px;
  }

  &__tags {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    margin-bottom: 22px;
  }

  &__tag {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    font-size: 0.6875rem;
    font-weight: 600;
    border-radius: 999px;
    padding: 3px 10px;

    &--blue {
      color: $primary;
      background: $primary-light;
    }

    &--violet {
      color: #7c3aed;
      background: #f3e8ff;
    }

    &--amber {
      color: $warning;
      background: $warning-subtle;
    }

    &--jade {
      color: $success;
      background: $success-subtle;
    }

    &--rose {
      color: $danger;
      background: $danger-subtle;
    }

    &--featured {
      color: $warning;
      background: $warning-subtle;
    }
  }

  /* ---------- stats ---------- */
  &__stats {
    display: flex;
    border: 1px solid $border-default;
    border-radius: 10px;
    overflow: hidden;
    margin-bottom: 18px;
  }

  &__stat {
    flex: 1;
    padding: 12px 16px;
    border-right: 1px solid $border-subtle;
    display: flex;
    flex-direction: column;
    gap: 2px;

    &:last-child {
      border-right: none;
    }
  }

  &__stat-value {
    font-size: 1.25rem;
    font-weight: 800;
    letter-spacing: -0.02em;
    color: $text-primary;
    line-height: 1;
  }

  &__stat-label {
    font-size: 0.625rem;
    font-weight: 600;
    letter-spacing: 0.05em;
    text-transform: uppercase;
    color: $text-tertiary;
    margin-top: 2px;
  }

  /* ---------- meta row ---------- */
  &__meta-row {
    display: flex;
    flex-wrap: wrap;
    gap: 16px;
    margin-bottom: 26px;
  }

  &__meta-item {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 0.75rem;
    color: $text-secondary;

    svg {
      flex-shrink: 0;
      color: $text-tertiary;
    }
  }

  /* ---------- question breakdown / preview ---------- */
  &__group-head {
    display: flex;
    align-items: center;
    gap: 10px;
    margin: 0 0 14px;

    h3 {
      margin: 0;
      font-family: $font-mono;
      font-size: 0.6875rem;
      font-weight: 600;
      text-transform: uppercase;
      letter-spacing: 0.08em;
      color: $text-tertiary;
      white-space: nowrap;
    }
  }

  &__group-line {
    flex: 1;
    height: 1px;
    background: $border-default;
  }

  &__preview-list {
    display: flex;
    flex-direction: column;
    gap: 18px;
    max-width: 640px;
  }

  &__preview-item {
    display: flex;
    flex-direction: column;
    gap: 6px;
  }

  &__preview-item-top {
    display: flex;
    align-items: baseline;
    justify-content: space-between;
    gap: 10px;
  }

  &__preview-item-name {
    font-size: 0.8125rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__preview-item-count {
    font-size: 0.71875rem;
    font-weight: 600;
    color: $text-tertiary;
    flex-shrink: 0;
  }

  &__preview-bar-wrap {
    height: 6px;
    border-radius: 4px;
    background: $bg-inset;
    overflow: hidden;
  }

  &__preview-bar {
    height: 100%;
    border-radius: 4px;
    background: $primary;
  }

  &__preview-item-sample {
    margin: 2px 0 0;
    padding: 10px 12px;
    background: $bg-subtle;
    border: 1px solid $border-subtle;
    border-left: 2px solid $primary-subtle;
    border-radius: 8px;
    font-size: 0.8125rem;
    font-style: italic;
    line-height: 1.55;
    color: $text-secondary;
  }

  &__preview-item-sample-label {
    display: block;
    margin-bottom: 4px;
    font-family: $font-mono;
    font-size: 0.625rem;
    font-weight: 600;
    font-style: normal;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $text-tertiary;
  }

  /* ---------- empty state ---------- */
  &__empty {
    padding: 52px 20px;
    text-align: center;
    color: $text-tertiary;
    font-size: 0.84375rem;

    &--list {
      display: flex;
      flex-direction: column;
      align-items: center;
      gap: 10px;

      svg {
        color: $text-tertiary;
      }
    }
  }

  /* ---------- responsive ---------- */
  @media (max-width: 800px) {
    &__split {
      grid-template-columns: 1fr;
    }

    &__list {
      display: none;
    }
  }
}
