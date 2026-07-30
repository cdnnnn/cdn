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

export interface ModelInfo {
  id: string;
  name: string;
  provider: string;
  providerId: string;
  description: string;
  version: string;
  capabilities: string[];
  contextWindow: string;
  pricing: string;
  speedRating: string;
  accuracyScore: number;
  agentScore: number;
  category: EvalTypeId;
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
}

export interface Metric {
  id: string;
  name: string;
  tooltip: string;
  defaultChecked: boolean;
}

export interface EvaluationDraft {
  name: string;
  type: EvalTypeId | null;
  providers: string[];
  models: string[];
  dataset: string | null;
  metrics: string[];
}

export const WIZARD_STEPS = ['Name', 'Type', 'Providers', 'Models', 'Test Suite', 'Metrics', 'Review'] as const;

















//data.ts
import type { EvalType, Metric, ModelInfo, Provider, TestSuite } from './types';

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
  { id: 'azure', name: 'Azure OpenAI Service', status: 'not_connected', modelCount: 8, logo: 'Az', desc: 'Enterprise tenant hosting with zero data-retention SLAs.' },
  { id: 'ollama', name: 'Ollama Local (On-Prem)', status: 'not_connected', modelCount: 12, logo: 'OL', desc: 'Locally running quantized models inside your own network.' },
];

export const MODELS: ModelInfo[] = [
  { id: 'm-1', name: 'Model Alpha Agent', provider: 'Together AI', providerId: 'together', description: 'Purpose-built for autonomous multi-step tool execution with strong JSON schema adherence.', version: 'v2.1', capabilities: ['Tool Calling', 'Autonomous Agent', 'JSON Mode'], contextWindow: '128k tokens', pricing: '$0.70 / 1M tokens', speedRating: 'Ultra Fast (85 t/s)', accuracyScore: 94.8, agentScore: 97.5, category: 'agent' },
  { id: 'm-2', name: 'Model Delta Agent v2', provider: 'Anthropic', providerId: 'anthropic', description: 'Balances deep reasoning with reliable tool use, ideal for complex multi-turn workflows.', version: 'v2.0', capabilities: ['Tool Calling', 'Deep Reasoning', 'Vision'], contextWindow: '200k tokens', pricing: '$3.00 / 1M tokens', speedRating: 'Fast (55 t/s)', accuracyScore: 95.4, agentScore: 96.2, category: 'agent' },
  { id: 'm-3', name: 'Model Gamma Agent', provider: 'OpenAI', providerId: 'openai', description: 'General-purpose agent model with strong multimodal understanding and tool calling.', version: 'v4', capabilities: ['Tool Calling', 'Multimodal Vision', 'Reasoning'], contextWindow: '128k tokens', pricing: '$2.50 / 1M tokens', speedRating: 'Fast (65 t/s)', accuracyScore: 94.2, agentScore: 95.0, category: 'agent' },
  { id: 'm-4', name: 'Model Epsilon Reasoning', provider: 'Together AI', providerId: 'together', description: 'Optimized for mathematical reasoning, formal logic, and multi-step code generation.', version: 'v1.4', capabilities: ['Deep Math', 'Advanced Logic', 'Code Generation'], contextWindow: '64k tokens', pricing: '$0.55 / 1M tokens', speedRating: 'Medium (42 t/s)', accuracyScore: 96.1, agentScore: 91.8, category: 'model' },
  { id: 'm-5', name: 'Model Zeta Instruct', provider: 'Groq Cloud', providerId: 'groq', description: 'LPU-accelerated instruction model tuned for near-instant conversational responses.', version: 'v1.1', capabilities: ['Instant Response', 'General Chat', 'Tool Calling'], contextWindow: '128k tokens', pricing: '$0.59 / 1M tokens', speedRating: 'Instant (280 t/s)', accuracyScore: 91.5, agentScore: 89.4, category: 'model' },
  { id: 'm-6', name: 'Model Theta Long-Context', provider: 'Google Gemini', providerId: 'gemini', description: 'Massive 2M-token context window built for document-heavy retrieval and analysis.', version: 'v1.5', capabilities: ['2M Tokens', 'Multi-modal Video', 'Document Analysis'], contextWindow: '2,000,000 tokens', pricing: '$2.50 / 1M tokens', speedRating: 'Fast (50 t/s)', accuracyScore: 93.7, agentScore: 91.0, category: 'rag' },
  { id: 'm-7', name: 'Model Theta Flash', provider: 'Google Gemini', providerId: 'gemini', description: 'Lightweight, low-latency variant tuned for high-throughput streaming workloads.', version: 'v1.5 Flash', capabilities: ['Vision', 'Tool Calling', 'Streaming'], contextWindow: '1,000,000 tokens', pricing: '$0.10 / 1M tokens', speedRating: 'Ultra Fast (120 t/s)', accuracyScore: 92.4, agentScore: 94.3, category: 'agent' },
  { id: 'm-8', name: 'Model Delta Opus', provider: 'Anthropic', providerId: 'anthropic', description: 'Flagship reasoning model for the highest-stakes, highest-accuracy evaluation tasks.', version: 'v3', capabilities: ['Deep Reasoning', 'Long Context', 'Vision'], contextWindow: '200k tokens', pricing: '$15.00 / 1M tokens', speedRating: 'Medium (25 t/s)', accuracyScore: 97.2, agentScore: 94.8, category: 'model' },
  { id: 'm-9', name: 'Model Eta Instruct', provider: 'Together AI', providerId: 'together', description: 'Strong all-round coding and multilingual model at a competitive price point.', version: 'v1.2', capabilities: ['Coding', 'Math', 'Multilingual'], contextWindow: '128k tokens', pricing: '$0.90 / 1M tokens', speedRating: 'Fast (65 t/s)', accuracyScore: 93.8, agentScore: 92.1, category: 'model' },
  { id: 'm-10', name: 'Model Gamma Mini', provider: 'OpenAI', providerId: 'openai', description: 'Compact, cost-efficient model for high-volume everyday chat and vision tasks.', version: 'v4 Mini', capabilities: ['Vision', 'Tool Calling', 'Streaming'], contextWindow: '128k tokens', pricing: '$0.15 / 1M tokens', speedRating: 'Ultra Fast (90 t/s)', accuracyScore: 88.5, agentScore: 87.2, category: 'model' },
];

export const TEST_SUITES: TestSuite[] = [
  { id: 'ts-agent', name: 'Autonomous Tool & Workflow Suite', category: 'Agents', questions: 420, language: 'English / JSON', task: 'Multi-step API Execution & Self-Correction', difficulty: 'Expert', version: 'v3.2', maintainer: 'SemcoEval Labs', description: 'Evaluates multi-step tool calling, nested JSON schema generation, error recovery, and workflow execution.', recommendedFor: ['agent'], featured: true },
  { id: 'ts-swe', name: 'SWE-bench Verified Software Engineer Suite', category: 'Coding', questions: 500, language: 'Python / JS / TS', task: 'Autonomous GitHub Issue Resolution', difficulty: 'Expert', version: '2026.2', maintainer: 'Open Source AI Labs', description: 'Measures ability to independently diagnose bug reports, run local unit tests, and commit working patches.', recommendedFor: ['agent', 'model'], featured: true },
  { id: 'ts-mmlu', name: 'MMLU-Pro General Knowledge & Reasoning Suite', category: 'General', questions: 1400, language: 'Multilingual (24 Languages)', task: 'Academic & Professional Problem Solving', difficulty: 'High', version: 'Pro-v2', maintainer: 'Stanford CRFM', description: 'Comprehensive multi-domain benchmark covering 57 disciplines including law, physics, and ethics.', recommendedFor: ['model'], featured: false },
  { id: 'ts-ragas', name: 'Ragas Document Factual Recall & Faithfulness', category: 'RAG', questions: 350, language: 'English', task: 'Context Precision & Hallucination Defense', difficulty: 'Medium', version: '1.4', maintainer: 'Ragas Ecosystem', description: 'Evaluates retrieval accuracy and ensures answers derive strictly from verified documents.', recommendedFor: ['rag'], featured: true },
  { id: 'ts-finance', name: 'Corporate Finance & Audit Math Suite', category: 'Finance', questions: 280, language: 'English / Numeric', task: 'Exact Mathematical Deduction & Regulation', difficulty: 'Advanced', version: '2026-Q1', maintainer: 'SemcoEval Labs', description: 'Validates exact calculation precision, formula interpretation, and compliance audit reasoning.', recommendedFor: ['model', 'agent'], featured: false },
  { id: 'ts-health', name: 'Clinical Diagnostic Safety & Care Benchmark', category: 'Healthcare', questions: 310, language: 'English / Medical', task: 'Diagnostic Logic & Patient Guardrails', difficulty: 'Expert', version: 'Med-2.1', maintainer: 'HealthAI Foundation', description: 'Assesses diagnostic recommendation accuracy and safety compliance in health triage.', recommendedFor: ['model'], featured: false },
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



























//Models.tsx
import { useMemo, useState, type FC } from 'react';
import { useNavigate } from 'react-router-dom';
import { Search, X, Boxes } from 'lucide-react';
import { MODELS } from '../RunEvaluation/data';
import './Models.scss';

const CAPABILITY_FILTERS = ['All', 'Tool Calling', 'Vision', 'Reasoning', 'Coding'];

const Models: FC = () => {
  const navigate = useNavigate();
  const [query, setQuery] = useState('');
  const [capability, setCapability] = useState('All');

  const filtered = useMemo(() => {
    return MODELS.filter((m) => {
      if (query && !m.name.toLowerCase().includes(query.toLowerCase()) && !m.provider.toLowerCase().includes(query.toLowerCase())) {
        return false;
      }
      if (capability !== 'All') {
        const matches = m.capabilities.some((c) => c.toLowerCase().includes(capability.toLowerCase()));
        if (!matches) return false;
      }
      return true;
    });
  }, [query, capability]);

  return (
    <div className="models-page">
      <div className="models-page__header">
        <div className="models-page__header-left">
          <p className="models-page__header-eyebrow">Model catalog</p>
          <h1 className="models-page__title">Models</h1>
          <p className="models-page__subtitle">Browse available AI models across every connected provider</p>
        </div>

        <div className="models-page__header-meta">
          <Boxes size={13} />
          {MODELS.length} models available
        </div>
      </div>

      <div className="models-page__filters">
        <div className="models-page__search">
          <Search size={15} />
          <input type="text" placeholder="Search models..." value={query} onChange={(e) => setQuery(e.target.value)} />
          {query && (
            <button type="button" className="models-page__search-clear" onClick={() => setQuery('')} aria-label="Clear search">
              <X size={13} />
            </button>
          )}
        </div>

        <div className="models-page__chips">
          {CAPABILITY_FILTERS.map((c) => (
            <button
              key={c}
              type="button"
              className={`models-page__chip${capability === c ? ' models-page__chip--active' : ''}`}
              onClick={() => setCapability(c)}
            >
              {c}
            </button>
          ))}
        </div>
      </div>

      <div className="models-page__grid">
        {filtered.map((m) => (
          <div className="models-page__card" key={m.id}>
            <div className="models-page__card-top">
              <span className="models-page__provider-badge">{m.provider}</span>
              <span className="models-page__version">{m.version}</span>
            </div>

            <h3 className="models-page__name">{m.name}</h3>
            <p className="models-page__desc">{m.description}</p>

            <div className="models-page__caps">
              {m.capabilities.map((c) => (
                <span key={c} className="models-page__cap-pill">
                  {c}
                </span>
              ))}
            </div>

            <div className="models-page__specs">
              <div className="models-page__spec">
                <span className="models-page__spec-label">Context</span>
                <span className="models-page__spec-value n">{m.contextWindow}</span>
              </div>
              <div className="models-page__spec">
                <span className="models-page__spec-label">Price</span>
                <span className="models-page__spec-value n">{m.pricing}</span>
              </div>
              <div className="models-page__spec">
                <span className="models-page__spec-label">Speed</span>
                <span className="models-page__spec-value n">{m.speedRating}</span>
              </div>
              <div className="models-page__spec">
                <span className="models-page__spec-label">Accuracy</span>
                <span className="models-page__spec-value models-page__spec-value--highlight n">{m.accuracyScore}%</span>
              </div>
            </div>

            <div className="models-page__card-actions">
              <button type="button" className="models-page__btn models-page__btn--outline">
                Details
              </button>
              <button type="button" className="models-page__btn models-page__btn--primary" onClick={() => navigate('/app/run-evaluation')}>
                Test
              </button>
            </div>
          </div>
        ))}

        {filtered.length === 0 && (
          <div className="models-page__empty">
            <Search size={22} />
            <p>No models match your filters.</p>
          </div>
        )}
      </div>
    </div>
  );
};

export default Models;

















//Models.scss
@use '../../../styles/variables' as *;

.models-page {
  display: flex;
  flex-direction: column;
  gap: 18px;

  /* ---------- header ---------- */
  &__header {
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
    letter-spacing: -0.02em;
    color: $text-primary;
  }

  &__subtitle {
    margin-top: 3px;
    color: $text-secondary;
    font-size: 0.84375rem;
  }

  /* ---------- filters ---------- */
  &__filters {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
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

  &__chips {
    display: flex;
    flex-wrap: wrap;
    gap: 7px;
  }

  &__chip {
    font-size: 0.78125rem;
    font-weight: 500;
    color: $text-secondary;
    background: $bg-main;
    border: 1px solid $border-default;
    border-radius: 999px;
    padding: 7px 14px;
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
  }

  /* ---------- card grid ---------- */
  &__grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
    gap: 14px;
  }

  &__card {
    display: flex;
    flex-direction: column;
    gap: 12px;
    padding: 18px 20px;
    border: 1px solid $border-subtle;
    border-radius: 14px;
    background: $bg-main;
    box-shadow: $shadow-xs;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &:hover {
      border-color: $border-strong;
      box-shadow: $shadow-sm;
    }
  }

  &__card-top {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__provider-badge {
    font-size: 0.6875rem;
    font-weight: 600;
    color: $primary;
    background: $primary-light;
    border-radius: 999px;
    padding: 3px 10px;
  }

  &__version {
    font-family: $font-mono;
    font-size: 0.6875rem;
    color: $text-tertiary;
  }

  &__name {
    font-size: 0.9375rem;
    font-weight: 700;
    letter-spacing: -0.01em;
    color: $text-primary;
  }

  &__desc {
    margin-top: -6px;
    font-size: 0.8125rem;
    line-height: 1.5;
    color: $text-secondary;
  }

  &__caps {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
  }

  &__cap-pill {
    font-size: 0.71875rem;
    color: $text-secondary;
    background: $bg-subtle;
    border: 1px solid $border-subtle;
    border-radius: 6px;
    padding: 3px 8px;
  }

  &__specs {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 8px;
    margin-top: 2px;
    padding-top: 13px;
    border-top: 1px solid $border-subtle;
  }

  &__spec {
    display: flex;
    flex-direction: column;
    gap: 4px;
    min-width: 0;
  }

  &__spec-label {
    font-size: 0.59375rem;
    font-weight: 600;
    letter-spacing: 0.05em;
    text-transform: uppercase;
    color: $text-tertiary;
  }

  &__spec-value {
    font-size: 0.71875rem;
    font-weight: 600;
    color: $text-secondary;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;

    &--highlight {
      color: $success;
      font-weight: 700;
    }
  }

  &__card-actions {
    display: flex;
    gap: 8px;
    margin-top: 2px;
  }

  &__btn {
    flex: 1;
    text-align: center;
    font-family: $font-body;
    font-size: 0.78125rem;
    font-weight: 600;
    padding: 8px 12px;
    border-radius: 8px;
    border: 1px solid transparent;
    cursor: pointer;
    transition: background 0.14s ease, border-color 0.14s ease, color 0.14s ease;

    &--outline {
      background: $bg-main;
      border-color: $border-default;
      color: $text-primary;

      &:hover {
        border-color: $text-primary;
      }
    }

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

  /* ---------- empty state ---------- */
  &__empty {
    grid-column: 1 / -1;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 10px;
    padding: 52px 20px;
    border: 1px dashed $border-strong;
    border-radius: 14px;
    color: $text-tertiary;
    font-size: 0.84375rem;

    svg {
      color: $text-tertiary;
    }
  }

  /* ---------- responsive ---------- */
  @media (max-width: 480px) {
    &__specs {
      grid-template-columns: repeat(2, 1fr);
    }
  }
}
