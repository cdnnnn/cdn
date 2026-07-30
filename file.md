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
      { id: 'ts-agent-api', name: 'API Execution', count: 180 },
      { id: 'ts-agent-selfcorrect', name: 'Self-Correction', count: 140 },
      { id: 'ts-agent-planning', name: 'Multi-step Planning', count: 100 },
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
      { id: 'ts-swe-diagnosis', name: 'Bug Diagnosis', count: 200 },
      { id: 'ts-swe-tests', name: 'Unit Test Writing', count: 150 },
      { id: 'ts-swe-patch', name: 'Patch Generation', count: 150 },
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
      { id: 'ts-mmlu-stem', name: 'STEM', count: 500 },
      { id: 'ts-mmlu-humanities', name: 'Humanities', count: 450 },
      { id: 'ts-mmlu-social', name: 'Social Sciences', count: 450 },
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
      { id: 'ts-ragas-precision', name: 'Context Precision', count: 150 },
      { id: 'ts-ragas-faithfulness', name: 'Faithfulness', count: 120 },
      { id: 'ts-ragas-hallucination', name: 'Hallucination Defense', count: 80 },
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
      { id: 'ts-finance-audit', name: 'Audit Math', count: 120 },
      { id: 'ts-finance-regulation', name: 'Regulation', count: 100 },
      { id: 'ts-finance-formula', name: 'Formula Interpretation', count: 60 },
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
      { id: 'ts-health-diagnosis', name: 'Diagnostic Logic', count: 150 },
      { id: 'ts-health-guardrails', name: 'Patient Guardrails', count: 100 },
      { id: 'ts-health-triage', name: 'Triage Safety', count: 60 },
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























import { useEffect, useMemo, useState, type FC } from 'react';
import { UploadCloud, Check, ListChecks, X } from 'lucide-react';
import { TEST_SUITES } from '../data';
import type { EvalTypeId } from '../types';

interface Props {
  evalType: EvalTypeId | null;
  selected: string | null;
  onSelect: (id: string) => void;
  subgroupSelections: string[];
  onSubgroupsChange: (ids: string[]) => void;
}

const CATEGORIES = ['All', 'Agents', 'Coding', 'General', 'RAG', 'Finance', 'Healthcare'];

const DatasetStep: FC<Props> = ({ evalType, selected, onSelect, subgroupSelections, onSubgroupsChange }) => {
  const [tab, setTab] = useState<'official' | 'private'>('official');
  const [category, setCategory] = useState('All');
  const [panelDatasetId, setPanelDatasetId] = useState<string | null>(null);
  const [panelSelection, setPanelSelection] = useState<string[]>([]);

  const filtered = useMemo(
    () => TEST_SUITES.filter((t) => category === 'All' || t.category === category),
    [category]
  );

  const panelDataset = useMemo(() => TEST_SUITES.find((t) => t.id === panelDatasetId) ?? null, [panelDatasetId]);

  const openPanel = (datasetId: string) => {
    const dataset = TEST_SUITES.find((t) => t.id === datasetId);
    // If this dataset is already selected, seed with its current subgroup selection.
    // Otherwise start fresh with every subgroup checked by default.
    const existing = selected === datasetId ? subgroupSelections : [];
    const seed = existing.length > 0 ? existing : (dataset?.subgroups.map((s) => s.id) ?? []);
    setPanelSelection(seed);
    setPanelDatasetId(datasetId);
  };

  const closePanel = () => {
    setPanelDatasetId(null);
  };

  const toggleSubgroup = (id: string) => {
    setPanelSelection((prev) => (prev.includes(id) ? prev.filter((x) => x !== id) : [...prev, id]));
  };

  const selectAll = () => {
    if (!panelDataset) return;
    setPanelSelection(panelDataset.subgroups.map((s) => s.id));
  };

  const selectNone = () => setPanelSelection([]);

  const applyPanel = () => {
    if (!panelDatasetId) return;
    onSelect(panelDatasetId);
    onSubgroupsChange(panelSelection);
    closePanel();
  };

  // close on escape
  useEffect(() => {
    if (!panelDatasetId) return;
    const handler = (e: KeyboardEvent) => {
      if (e.key === 'Escape') closePanel();
    };
    document.addEventListener('keydown', handler);
    return () => document.removeEventListener('keydown', handler);
  }, [panelDatasetId]);

  const selectedQuestionCount = (dataset: (typeof TEST_SUITES)[number], subgroupIds: string[]) =>
    dataset.subgroups.filter((s) => subgroupIds.includes(s.id)).reduce((sum, s) => sum + s.count, 0);

  return (
    <div className="run-eval__card">
      <h2 className="run-eval__step-title">Pick a test suite</h2>
      <p className="run-eval__step-desc">Test suites contain questions that measure AI capabilities.</p>

      <div className="run-eval__tabs">
        {(['official', 'private'] as const).map((t) => (
          <button
            key={t}
            type="button"
            className={`run-eval__tab${tab === t ? ' run-eval__tab--active' : ''}`}
            onClick={() => setTab(t)}
          >
            {t === 'official' ? 'Benchmarks' : 'Upload'}
          </button>
        ))}
      </div>

      {tab === 'official' && (
        <>
          <div className="run-eval__category-filters">
            {CATEGORIES.map((c) => (
              <button
                key={c}
                type="button"
                className={`run-eval__chip${category === c ? ' run-eval__chip--active' : ''}`}
                onClick={() => setCategory(c)}
              >
                {c}
              </button>
            ))}
          </div>

          <div className="run-eval__dataset-grid">
            {filtered.map((d) => {
              const isSelected = selected === d.id;
              const recommended = evalType ? d.recommendedFor.includes(evalType) : false;
              const activeSubgroupCount = isSelected && subgroupSelections.length > 0 ? subgroupSelections.length : d.subgroups.length;
              const activeQuestionCount = isSelected && subgroupSelections.length > 0
                ? selectedQuestionCount(d, subgroupSelections)
                : d.questions;

              return (
                <div
                  key={d.id}
                  role="button"
                  tabIndex={0}
                  className={`run-eval__dataset-card${isSelected ? ' run-eval__dataset-card--selected' : ''}`}
                  onClick={() => onSelect(d.id)}
                  onKeyDown={(e) => {
                    if (e.key === 'Enter' || e.key === ' ') {
                      e.preventDefault();
                      onSelect(d.id);
                    }
                  }}
                >
                  <div className="run-eval__dataset-top">
                    <span className="run-eval__dataset-name">{d.name}</span>
                    <span className="run-eval__dataset-top-actions">
                      <button
                        type="button"
                        className="run-eval__dataset-subgroups-btn"
                        title="Choose subcategories"
                        onClick={(e) => {
                          e.stopPropagation();
                          openPanel(d.id);
                        }}
                      >
                        <ListChecks size={15} />
                      </button>
                      {isSelected && (
                        <span className="run-eval__type-check">
                          <Check size={12} strokeWidth={2.75} />
                        </span>
                      )}
                    </span>
                  </div>
                  <p className="run-eval__dataset-desc">{d.description}</p>
                  <div className="run-eval__dataset-meta n">
                    <span>{activeQuestionCount} questions</span>
                    <span>{activeSubgroupCount}/{d.subgroups.length} subcategories</span>
                    <span>{d.difficulty}</span>
                  </div>
                  {recommended && <span className="run-eval__badge run-eval__badge--soft">Recommended</span>}
                </div>
              );
            })}
          </div>
        </>
      )}

      {tab === 'private' && (
        <div className="run-eval__upload-zone">
          <UploadCloud size={26} />
          <h3>Upload Test Data</h3>
          <p>Drag &amp; drop or click to browse</p>
          <div className="run-eval__format-chips">
            {['CSV', 'JSON', 'JSONL', 'HuggingFace'].map((f) => (
              <span key={f} className="run-eval__chip run-eval__chip--static">
                {f}
              </span>
            ))}
          </div>
        </div>
      )}

      {/* ---------- subgroup slide-over panel ---------- */}
      {panelDataset && (
        <>
          <div className="run-eval__panel-overlay" onClick={closePanel} />
          <aside className="run-eval__panel">
            <div className="run-eval__panel-head">
              <div>
                <p className="run-eval__panel-eyebrow">{panelDataset.category}</p>
                <h3 className="run-eval__panel-title">{panelDataset.name}</h3>
              </div>
              <button type="button" className="run-eval__panel-close" onClick={closePanel} aria-label="Close">
                <X size={16} />
              </button>
            </div>

            <div className="run-eval__panel-toolbar">
              <span>{panelSelection.length} of {panelDataset.subgroups.length} selected</span>
              <span className="run-eval__panel-toolbar-actions">
                <button type="button" className="run-eval__link" onClick={selectAll}>Select all</button>
                <button type="button" className="run-eval__link" onClick={selectNone}>Clear</button>
              </span>
            </div>

            <div className="run-eval__panel-list">
              {panelDataset.subgroups.map((s) => {
                const checked = panelSelection.includes(s.id);
                return (
                  <label key={s.id} className="run-eval__panel-item">
                    <span className="run-eval__panel-item-main">
                      <input type="checkbox" checked={checked} onChange={() => toggleSubgroup(s.id)} />
                      <span className="run-eval__panel-item-name">{s.name}</span>
                    </span>
                    <span className="run-eval__panel-item-count n">{s.count}</span>
                  </label>
                );
              })}
            </div>

            <div className="run-eval__panel-footer">
              <span className="run-eval__panel-footer-total n">
                {selectedQuestionCount(panelDataset, panelSelection)} questions
              </span>
              <button
                type="button"
                className="run-eval__panel-apply"
                disabled={panelSelection.length === 0}
                onClick={applyPanel}
              >
                Apply
              </button>
            </div>
          </aside>
        </>
      )}
    </div>
  );
};

export default DatasetStep;






















/* ============================================================
   Append these rules inside the top-level `.run-eval { ... }`
   block in RunEvaluation.scss (nest under `&__...` as usual).
   Replaces the previous panel-additions snippet — width is now
   400px and the visual detailing / Apply button are refined.
   ============================================================ */

&__dataset-top-actions {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-shrink: 0;
}

&__dataset-subgroups-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  border: 1px solid transparent;
  border-radius: $radius-sm;
  color: $text-tertiary;
  background: $bg-subtle;
  cursor: pointer;
  transition: background 0.15s, color 0.15s, border-color 0.15s;

  &:hover {
    background: $primary-light;
    border-color: $primary-subtle;
    color: $primary;
  }
}

/* ---------- slide-over panel ---------- */
&__panel-overlay {
  position: fixed;
  inset: 0;
  background: rgba(14, 21, 38, 0.4);
  backdrop-filter: blur(1px);
  z-index: 200;
  animation: run-eval-overlay-in 0.2s ease-out;
}

@keyframes run-eval-overlay-in {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

&__panel {
  position: fixed;
  top: 0;
  right: 0;
  bottom: 0;
  width: 400px;
  max-width: 100vw;
  background: $bg-main;
  border-left: 1px solid $border-default;
  box-shadow: $shadow-xl;
  z-index: 201;
  display: flex;
  flex-direction: column;
  animation: run-eval-panel-in 0.24s cubic-bezier(0.16, 1, 0.3, 1);
}

@keyframes run-eval-panel-in {
  from {
    transform: translateX(100%);
  }
  to {
    transform: translateX(0);
  }
}

&__panel-head {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 12px;
  padding: 22px 24px 18px;
  border-bottom: 1px solid $border-subtle;
  background: $bg-subtle;
}

&__panel-eyebrow {
  margin: 0 0 4px;
  font-size: 11px;
  font-weight: 700;
  color: $primary;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

&__panel-title {
  margin: 0;
  font-size: 16px;
  font-weight: 700;
  color: $text-primary;
  line-height: 1.35;
}

&__panel-close {
  flex-shrink: 0;
  width: 30px;
  height: 30px;
  display: flex;
  align-items: center;
  justify-content: center;
  border: 1px solid $border-default;
  background: $bg-main;
  border-radius: $radius-sm;
  color: $text-secondary;
  cursor: pointer;
  transition: background 0.15s, border-color 0.15s;

  &:hover {
    background: $bg-inset;
    border-color: $border-strong;
  }
}

&__panel-toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 14px 24px;
  font-size: 12.5px;
  font-weight: 600;
  color: $text-secondary;
  border-bottom: 1px solid $border-subtle;
}

&__panel-toolbar-actions {
  display: flex;
  gap: 16px;

  .run-eval__link {
    font-weight: 600;
    color: $primary;

    &:hover {
      color: $primary-hover;
      text-decoration: underline;
    }
  }
}

&__panel-list {
  flex: 1;
  overflow-y: auto;
  padding: 10px 14px;
}

&__panel-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  padding: 13px 14px;
  margin-bottom: 4px;
  border: 1px solid transparent;
  border-radius: $radius-md;
  cursor: pointer;
  transition: background 0.15s, border-color 0.15s;

  &:hover {
    background: $bg-subtle;
    border-color: $border-subtle;
  }
}

&__panel-item-main {
  display: flex;
  align-items: center;
  gap: 12px;

  input[type='checkbox'] {
    width: 17px;
    height: 17px;
    accent-color: $primary;
    cursor: pointer;
  }
}

&__panel-item-name {
  font-size: 13.5px;
  font-weight: 500;
  color: $text-primary;
}

&__panel-item-count {
  font-size: 12px;
  font-weight: 600;
  color: $text-tertiary;
  background: $bg-inset;
  padding: 2px 9px;
  border-radius: 999px;
}

&__panel-footer {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  padding: 16px 24px;
  border-top: 1px solid $border-subtle;
  background: $bg-subtle;
}

&__panel-footer-total {
  font-size: 13px;
  font-weight: 700;
  color: $text-primary;
}

&__panel-apply {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border: none;
  background: $primary;
  color: #ffffff;
  padding: 10px 22px;
  border-radius: $radius-sm;
  font-size: 13px;
  font-weight: 700;
  cursor: pointer;
  transition: background 0.15s, box-shadow 0.15s;

  &:hover:not(:disabled) {
    background: $primary-hover;
    color: #ffffff;
    box-shadow: 0 4px 12px rgba(20, 40, 160, 0.28);
  }

  &:disabled {
    background: $border-strong;
    color: $bg-main;
    cursor: not-allowed;
  }
}
