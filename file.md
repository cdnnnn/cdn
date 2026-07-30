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
  apiKey?: string;
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
  { id: 'openai', name: 'OpenAI', status: 'connected', modelCount: 8, logo: 'O', desc: 'Industry benchmark provider offering flagship chat and reasoning models.', apiKey: 'sk-****-a91f' },
  { id: 'anthropic', name: 'Anthropic', status: 'connected', modelCount: 5, logo: 'A', desc: 'Safety-first reasoning models optimized for long context and tool use.', apiKey: 'sk-****-77e2' },
  { id: 'together', name: 'Together AI', status: 'connected', modelCount: 14, logo: 'T', desc: 'High-performance infrastructure hosting open-weight agent models.', apiKey: 'sk-****-c410' },
  { id: 'groq', name: 'Groq Cloud', status: 'connected', modelCount: 6, logo: 'G', desc: 'Ultra-low response time LPU inference engine for instant generation.', apiKey: 'gsk-****-3b6d' },
  { id: 'gemini', name: 'Google Gemini', status: 'connected', modelCount: 6, logo: 'G', desc: 'Massive context window models with native multi-modal reasoning.', apiKey: 'AIza****-9f02' },
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



























//Providers.tsx
import { useState, type FC, type FormEvent } from 'react';
import { PlugZap, CheckCircle2, Boxes, Settings2, Unplug } from 'lucide-react';
import { PROVIDERS } from '../RunEvaluation/data';
import type { Provider } from '../RunEvaluation/types';
import './Providers.scss';

const Providers: FC = () => {
  const [providers, setProviders] = useState<Provider[]>(PROVIDERS);
  const [activePanel, setActivePanel] = useState<string | null>(null);
  const [keyInput, setKeyInput] = useState('');

  const connectedCount = providers.filter((p) => p.status === 'connected').length;

  const openPanel = (id: string, existingKey?: string) => {
    setActivePanel((prev) => (prev === id ? null : id));
    setKeyInput(existingKey ?? '');
  };

  const handleSubmit = (e: FormEvent, id: string) => {
    e.preventDefault();
    if (!keyInput.trim()) return;
    setProviders((prev) =>
      prev.map((p) => (p.id === id ? { ...p, status: 'connected', apiKey: `sk-****-${keyInput.slice(-4)}` } : p))
    );
    setActivePanel(null);
  };

  const handleDisconnect = (id: string) => {
    setProviders((prev) => prev.map((p) => (p.id === id ? { ...p, status: 'not_connected', apiKey: undefined } : p)));
    setActivePanel(null);
  };

  return (
    <div className="providers-page">
      <div className="providers-page__header">
        <div className="providers-page__header-left">
          <p className="providers-page__header-eyebrow">Provider connections</p>
          <h1 className="providers-page__title">Providers</h1>
          <p className="providers-page__subtitle">Manage AI service connections and API keys</p>
        </div>

        <div className="providers-page__header-meta">
          <PlugZap size={13} />
          {connectedCount} of {providers.length} connected
        </div>
      </div>

      <div className="providers-page__body">
        <div className="providers-page__grid">
          {providers.map((p) => {
            const connected = p.status === 'connected';
            const panelOpen = activePanel === p.id;

            return (
              <div key={p.id} className={`providers-page__card${connected ? ' providers-page__card--connected' : ''}`}>
                <div className="providers-page__card-top">
                  <span className="providers-page__logo">{p.logo}</span>
                  <span className={`providers-page__status${connected ? ' providers-page__status--on' : ''}`}>
                    {connected && <CheckCircle2 size={11} />}
                    {connected ? 'Connected' : 'Not connected'}
                  </span>
                </div>

                <h3 className="providers-page__name">{p.name}</h3>
                <p className="providers-page__desc">{p.desc}</p>

                <div className="providers-page__stat">
                  <Boxes size={13} />
                  <span className="n">{p.modelCount}</span> models
                </div>

                {connected && !panelOpen && (
                  <div className="providers-page__key-row">
                    <span className="providers-page__key-label">API Key</span>
                    <code className="providers-page__key-value">{p.apiKey}</code>
                  </div>
                )}

                {panelOpen && (
                  <form className="providers-page__connect-form" onSubmit={(e) => handleSubmit(e, p.id)}>
                    <label className="providers-page__field-label" htmlFor={`key-${p.id}`}>
                      API Key
                    </label>
                    <input
                      id={`key-${p.id}`}
                      type="password"
                      className="providers-page__input"
                      placeholder="Enter API key"
                      value={keyInput}
                      onChange={(e) => setKeyInput(e.target.value)}
                      autoFocus
                    />
                    <div className="providers-page__form-actions">
                      <button type="button" className="providers-page__btn providers-page__btn--outline" onClick={() => setActivePanel(null)}>
                        Cancel
                      </button>
                      <button type="submit" className="providers-page__btn providers-page__btn--primary">
                        Save
                      </button>
                    </div>
                  </form>
                )}

                {!panelOpen && (
                  <div className="providers-page__actions">
                    {connected ? (
                      <>
                        <button
                          type="button"
                          className="providers-page__btn providers-page__btn--outline"
                          onClick={() => openPanel(p.id, p.apiKey)}
                        >
                          <Settings2 size={13} /> Configure
                        </button>
                        <button
                          type="button"
                          className="providers-page__btn providers-page__btn--danger-outline"
                          onClick={() => handleDisconnect(p.id)}
                        >
                          <Unplug size={13} /> Disconnect
                        </button>
                      </>
                    ) : (
                      <button type="button" className="providers-page__btn providers-page__btn--primary providers-page__btn--full" onClick={() => openPanel(p.id)}>
                        <PlugZap size={13} /> Connect
                      </button>
                    )}
                  </div>
                )}
              </div>
            );
          })}
        </div>
      </div>
    </div>
  );
};

export default Providers;




















//Providers.scss
@use '../../../styles/variables' as *;

.providers-page {
  display: flex;
  flex-direction: column;
  height: calc(100vh - 166px);
  min-height: 0;
  gap: 18px;

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

  /* ---------- scrollable body ---------- */
  &__body {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding-right: 4px;
    margin-right: -4px;
  }

  /* ---------- card grid ---------- */
  &__grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
    gap: 14px;
  }

  &__card {
    display: flex;
    flex-direction: column;
    gap: 10px;
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

    &--connected {
      border-color: $success-subtle;
    }
  }

  &__card-top {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__logo {
    width: 38px;
    height: 38px;
    flex-shrink: 0;
    border-radius: 11px;
    background: $text-primary;
    color: #fff;
    font-weight: 700;
    font-size: 0.875rem;
    display: grid;
    place-items: center;
  }

  &__status {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    font-size: 0.6875rem;
    font-weight: 600;
    color: $text-tertiary;
    background: $bg-inset;
    border-radius: 999px;
    padding: 3px 9px;

    &--on {
      color: $success;
      background: $success-subtle;
    }
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

  &__stat {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-size: 0.78125rem;
    color: $text-secondary;

    svg {
      color: $text-tertiary;
    }

    .n {
      font-weight: 700;
      color: $text-primary;
    }
  }

  /* ---------- api key display ---------- */
  &__key-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
    margin-top: 2px;
    padding: 9px 12px;
    background: $bg-subtle;
    border-radius: 8px;
  }

  &__key-label {
    font-size: 0.65625rem;
    font-weight: 600;
    letter-spacing: 0.05em;
    text-transform: uppercase;
    color: $text-tertiary;
  }

  &__key-value {
    font-family: $font-mono;
    font-size: 0.75rem;
    color: $text-secondary;
  }

  /* ---------- inline connect form ---------- */
  &__connect-form {
    margin-top: 2px;
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  &__field-label {
    font-size: 0.71875rem;
    font-weight: 600;
    color: $text-secondary;
  }

  &__input {
    width: 100%;
    border: 1px solid $border-default;
    border-radius: 8px;
    padding: 8px 11px;
    font-size: 0.8125rem;
    font-family: $font-body;
    color: $text-primary;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &::placeholder {
      color: #a8b1bb;
    }

    &:focus {
      outline: none;
      border-color: $primary;
      box-shadow: 0 0 0 3px $primary-light;
    }
  }

  &__form-actions {
    display: flex;
    justify-content: flex-end;
    gap: 8px;
  }

  /* ---------- actions ---------- */
  &__actions {
    display: flex;
    gap: 8px;
    margin-top: 2px;
  }

  &__btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 6px;
    font-family: $font-body;
    font-size: 0.75rem;
    font-weight: 600;
    padding: 7px 12px;
    border-radius: 8px;
    border: 1px solid transparent;
    cursor: pointer;
    transition: background 0.14s ease, border-color 0.14s ease, color 0.14s ease;
    flex: 1;

    &--outline {
      background: $bg-main;
      border-color: $border-default;
      color: $text-primary;

      &:hover {
        border-color: $text-primary;
      }
    }

    &--danger-outline {
      background: $bg-main;
      border-color: $border-default;
      color: $text-tertiary;

      &:hover {
        border-color: $danger;
        color: $danger;
        background: $danger-subtle;
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

    &--full {
      flex: 1;
    }
  }
}
