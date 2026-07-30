//Typestep.tsx
import type { FC } from 'react';
import { MessageSquare, Bot, Search, Check, Waypoints, Workflow } from 'lucide-react';
import { EVAL_TYPES } from '../data';
import type { AgentFramework, EvalTypeId } from '../types';

interface Props {
  value: EvalTypeId | null;
  onChange: (id: EvalTypeId) => void;
  agentFramework: AgentFramework | null;
  onAgentFrameworkChange: (framework: AgentFramework) => void;
}

const ICONS: Record<EvalTypeId, FC<{ size?: number }>> = {
  model: MessageSquare,
  agent: Bot,
  rag: Search,
};

const AGENT_FRAMEWORKS: { id: AgentFramework; name: string; desc: string; icon: FC<{ size?: number }> }[] = [
  { id: 'hermes', name: 'Hermes', desc: 'Lightweight single-agent tool-calling runtime.', icon: Waypoints },
  { id: 'langgraph', name: 'LangGraph', desc: 'Graph-based orchestration for multi-agent workflows.', icon: Workflow },
];

const TypeStep: FC<Props> = ({ value, onChange, agentFramework, onAgentFrameworkChange }) => {
  return (
    <div className="run-eval__card">
      <h2 className="run-eval__step-title">What are you testing?</h2>
      <p className="run-eval__step-desc">Different AI types need different evaluation methods.</p>

      <div className="run-eval__type-grid">
        {EVAL_TYPES.map((t) => {
          const Icon = ICONS[t.id];
          const selected = value === t.id;
          return (
            <div key={t.id} className={`run-eval__type-card-wrap${selected ? ' run-eval__type-card-wrap--selected' : ''}`}>
              <button
                type="button"
                className={`run-eval__type-card${selected ? ' run-eval__type-card--selected' : ''}`}
                onClick={() => onChange(t.id)}
              >
                <span className="run-eval__type-icon">
                  <Icon size={20} />
                </span>
                <span className="run-eval__type-content">
                  <span className="run-eval__type-title">{t.title}</span>
                  <span className="run-eval__type-desc">{t.desc}</span>
                </span>
                <span className="run-eval__badge">{t.badge}</span>
                {selected && (
                  <span className="run-eval__type-check">
                    <Check size={13} strokeWidth={2.75} />
                  </span>
                )}
              </button>

              {selected && t.id === 'agent' && (
                <div className="run-eval__agent-framework">
                  <p className="run-eval__agent-framework-label">Choose an agent framework</p>
                  <div className="run-eval__agent-framework-grid">
                    {AGENT_FRAMEWORKS.map((fw) => {
                      const FwIcon = fw.icon;
                      const fwSelected = agentFramework === fw.id;
                      return (
                        <button
                          key={fw.id}
                          type="button"
                          className={`run-eval__agent-framework-card${fwSelected ? ' run-eval__agent-framework-card--selected' : ''}`}
                          onClick={(e) => {
                            e.stopPropagation();
                            onAgentFrameworkChange(fw.id);
                          }}
                        >
                          <span className="run-eval__agent-framework-icon">
                            <FwIcon size={16} />
                          </span>
                          <span className="run-eval__agent-framework-info">
                            <span className="run-eval__agent-framework-name">{fw.name}</span>
                            <span className="run-eval__agent-framework-desc">{fw.desc}</span>
                          </span>
                          {fwSelected && (
                            <span className="run-eval__type-check">
                              <Check size={11} strokeWidth={2.75} />
                            </span>
                          )}
                        </button>
                      );
                    })}
                  </div>
                </div>
              )}
            </div>
          );
        })}
      </div>
    </div>
  );
};

export default TypeStep;














//Runevaluation.tsx
import { useState, type FC } from 'react';
import { useNavigate } from 'react-router-dom';
import { ArrowLeft, ArrowRight, Play, Check, Clock3 } from 'lucide-react';
import NameStep from './steps/NameStep';
import TypeStep from './steps/TypeStep';
import ProvidersStep from './steps/ProvidersStep';
import ModelsStep from './steps/ModelsStep';
import DatasetStep from './steps/DatasetStep';
import MetricsStep from './steps/MetricsStep';
import ReviewStep from './steps/ReviewStep';
import { METRICS } from './data';
import { WIZARD_STEPS, type EvaluationDraft } from './types';
import './RunEvaluation.scss';

const EMPTY_DRAFT: EvaluationDraft = {
  name: '',
  type: null,
  agentFramework: null,
  providers: [],
  models: [],
  dataset: null,
  datasetSubgroups: [],
  metrics: [],
};

const RunEvaluation: FC = () => {
  const navigate = useNavigate();
  const [step, setStep] = useState(1);
  const [draft, setDraft] = useState<EvaluationDraft>(EMPTY_DRAFT);
  const [error, setError] = useState<string | null>(null);
  const totalSteps = WIZARD_STEPS.length;

  const toggleInArray = (key: 'providers' | 'models' | 'metrics', id: string) => {
    setDraft((d) => {
      const arr = d[key];
      const next = arr.includes(id) ? arr.filter((x) => x !== id) : [...arr, id];
      return { ...d, [key]: next };
    });
  };

  const setType = (id: EvaluationDraft['type']) => {
    setDraft((d) => {
      if (d.type === id) return d;
      const defaults = [
        ...METRICS.universal.filter((m) => m.defaultChecked).map((m) => m.id),
        ...(id === 'agent' ? METRICS.agent : id === 'rag' ? METRICS.rag : METRICS.model)
          .filter((m) => m.defaultChecked)
          .map((m) => m.id),
      ];
      return {
        ...d,
        type: id,
        metrics: defaults,
        agentFramework: id === 'agent' ? d.agentFramework ?? 'hermes' : null,
      };
    });
  };

  const setAgentFramework = (framework: EvaluationDraft['agentFramework']) => {
    setDraft((d) => ({ ...d, agentFramework: framework }));
  };

  const selectDataset = (id: string) => {
    setDraft((d) => (d.dataset === id ? d : { ...d, dataset: id, datasetSubgroups: [] }));
  };

  const setDatasetSubgroups = (ids: string[]) => {
    setDraft((d) => ({ ...d, datasetSubgroups: ids }));
  };

  const validate = (): boolean => {
    setError(null);
    if (step === 1 && !draft.name.trim()) {
      setError('Enter an evaluation name to continue.');
      return false;
    }
    if (step === 2 && !draft.type) {
      setError('Select an evaluation type to continue.');
      return false;
    }
    if (step === 3 && draft.providers.length === 0) {
      setError('Select at least one provider to continue.');
      return false;
    }
    if (step === 4 && draft.models.length === 0) {
      setError('Select at least one model to continue.');
      return false;
    }
    if (step === 5 && !draft.dataset) {
      setError('Select a test suite to continue.');
      return false;
    }
    if (step === 6 && draft.metrics.length === 0) {
      setError('Select at least one metric to continue.');
      return false;
    }
    return true;
  };

  const goNext = () => {
    if (!validate()) return;
    setStep((s) => Math.min(totalSteps, s + 1));
  };
  const goBack = () => setStep((s) => Math.max(1, s - 1));
  const goToStep = (target: number) => {
    if (target < step) setStep(target);
  };

  const startEvaluation = () => {
    if (!validate()) return;
    navigate('/app/history');
  };

  return (
    <div className="run-eval">
      <div className="run-eval__header">
        <div className="run-eval__header-left">
          <p className="run-eval__header-eyebrow">Create evaluation</p>
          <h1 className="run-eval__title">New Evaluation</h1>
          <p className="run-eval__subtitle">Compare AI models with standardized tests</p>
        </div>

        <div className="run-eval__header-meta">
          <Clock3 size={13} />
          ~5 min guided setup
        </div>
      </div>

      <div className="run-eval__wizard">
        <div className="run-eval__tracker">
          <div className="run-eval__tracker-bar">
            <div
              className="run-eval__tracker-fill"
              style={{ width: `${((step - 1) / (totalSteps - 1)) * 100}%` }}
            />
          </div>
          <div className="run-eval__tracker-nodes">
            {WIZARD_STEPS.map((label, i) => {
              const num = i + 1;
              const state = num === step ? 'active' : num < step ? 'complete' : 'upcoming';
              return (
                <button
                  key={label}
                  type="button"
                  className={`run-eval__node run-eval__node--${state}`}
                  onClick={() => goToStep(num)}
                  disabled={num > step}
                >
                  <span className="run-eval__node-dot">
                    {state === 'complete' ? <Check size={12} strokeWidth={3} /> : num}
                  </span>
                  <span className="run-eval__node-label">{label}</span>
                </button>
              );
            })}
          </div>
        </div>

        <p className="run-eval__step-kicker">
          Step {step} of {totalSteps}
        </p>

        <div className="run-eval__body">
          {step === 1 && <NameStep name={draft.name} onChange={(name) => setDraft((d) => ({ ...d, name }))} />}
          {step === 2 && (
            <TypeStep
              value={draft.type}
              onChange={setType}
              agentFramework={draft.agentFramework}
              onAgentFrameworkChange={setAgentFramework}
            />
          )}
          {step === 3 && (
            <ProvidersStep
              selected={draft.providers}
              onToggle={(id) => toggleInArray('providers', id)}
              onGoToProviders={() => navigate('/app/providers')}
            />
          )}
          {step === 4 && (
            <ModelsStep
              providers={draft.providers}
              selected={draft.models}
              onToggle={(id) => toggleInArray('models', id)}
              onClear={() => setDraft((d) => ({ ...d, models: [] }))}
            />
          )}
          {step === 5 && (
            <DatasetStep
              evalType={draft.type}
              selected={draft.dataset}
              onSelect={selectDataset}
              subgroupSelections={draft.datasetSubgroups}
              onSubgroupsChange={setDatasetSubgroups}
            />
          )}
          {step === 6 && (
            <MetricsStep evalType={draft.type} selected={draft.metrics} onToggle={(id) => toggleInArray('metrics', id)} />
          )}
          {step === 7 && <ReviewStep draft={draft} />}

          {error && <p className="run-eval__error">{error}</p>}
        </div>

        <div className="run-eval__nav">
          {step > 1 ? (
            <button type="button" className="run-eval__btn run-eval__btn--secondary run-eval__btn--lg" onClick={goBack}>
              <ArrowLeft size={16} /> Back
            </button>
          ) : (
            <span />
          )}

          {step < totalSteps ? (
            <button type="button" className="run-eval__btn run-eval__btn--primary run-eval__btn--lg" onClick={goNext}>
              Continue <ArrowRight size={16} />
            </button>
          ) : (
            <button type="button" className="run-eval__btn run-eval__btn--primary run-eval__btn--lg" onClick={startEvaluation}>
              <Play size={16} /> Start Evaluation
            </button>
          )}
        </div>
      </div>
    </div>
  );
};

export default RunEvaluation;















//Reviewstep.tsx
import { useMemo, type FC } from 'react';
import { Info } from 'lucide-react';
import { EVAL_TYPES, MODELS, TEST_SUITES } from '../data';
import type { EvaluationDraft } from '../types';

interface Props {
  draft: EvaluationDraft;
}

const ReviewStep: FC<Props> = ({ draft }) => {
  const typeInfo = EVAL_TYPES.find((t) => t.id === draft.type);
  const agentFrameworkLabel = draft.agentFramework === 'hermes' ? 'Hermes' : draft.agentFramework === 'langgraph' ? 'LangGraph' : null;
  const modelNames = draft.models.map((id) => MODELS.find((m) => m.id === id)?.name).filter(Boolean);
  const dataset = TEST_SUITES.find((d) => d.id === draft.dataset);

  const questionCount = useMemo(() => {
    if (!dataset) return 0;
    if (draft.datasetSubgroups.length === 0) return dataset.questions;
    return dataset.subgroups
      .filter((s) => draft.datasetSubgroups.includes(s.id))
      .reduce((sum, s) => sum + s.count, 0);
  }, [dataset, draft.datasetSubgroups]);

  const subgroupNames = useMemo(() => {
    if (!dataset || draft.datasetSubgroups.length === 0) return null;
    return dataset.subgroups
      .filter((s) => draft.datasetSubgroups.includes(s.id))
      .map((s) => s.name)
      .join(', ');
  }, [dataset, draft.datasetSubgroups]);

  const { cost, minutes } = useMemo(() => {
    const modelCount = draft.models.length || 1;
    const estCost = questionCount * modelCount * 0.0009;
    const estMinutes = Math.max(1, Math.round((questionCount * modelCount) / 180));
    return { cost: estCost, minutes: estMinutes };
  }, [questionCount, draft.models.length]);

  return (
    <div className="run-eval__card">
      <h2 className="run-eval__step-title">Review &amp; Run</h2>
      <p className="run-eval__step-desc">Confirm your settings before starting.</p>

      <div className="run-eval__review">
        <div className="run-eval__review-row">
          <span>Name</span>
          <span>{draft.name || '—'}</span>
        </div>
        <div className="run-eval__review-row">
          <span>Type</span>
          <span>{typeInfo?.title ?? '—'}</span>
        </div>
        {agentFrameworkLabel && (
          <div className="run-eval__review-row">
            <span>Agent Framework</span>
            <span>{agentFrameworkLabel}</span>
          </div>
        )}
        <div className="run-eval__review-row">
          <span>Models</span>
          <span>{modelNames.length ? modelNames.join(', ') : '—'}</span>
        </div>
        <div className="run-eval__review-row">
          <span>Test Suite</span>
          <span>{dataset?.name ?? '—'}</span>
        </div>
        {subgroupNames && (
          <div className="run-eval__review-row">
            <span>Subcategories</span>
            <span>{subgroupNames}</span>
          </div>
        )}
        <div className="run-eval__review-row">
          <span>Questions</span>
          <span>{dataset ? questionCount : '—'}</span>
        </div>
        <div className="run-eval__review-divider" />
        <div className="run-eval__review-row run-eval__review-row--highlight">
          <span>Est. Cost</span>
          <span>~${cost.toFixed(2)}</span>
        </div>
        <div className="run-eval__review-row run-eval__review-row--highlight">
          <span>Est. Time</span>
          <span>~{minutes} min</span>
        </div>
      </div>

      <div className="run-eval__hint">
        <Info size={14} />
        <span>Costs are estimates. Actual costs depend on provider pricing.</span>
      </div>
    </div>
  );
};

export default ReviewStep;













//Types.ts
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





















/* ============================================================
   Append these rules inside the top-level `.run-eval { ... }`
   block in RunEvaluation.scss (nest under `&__...` as usual).
   Replaces the previous panel-additions snippet — width is now
   400px and the visual detailing / Apply button are refined.
   ============================================================ */

&__filter-chip--toggle {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 9px 12px;
  margin-bottom: 16px;
  border: 1px solid $border-default;
  border-radius: $radius-sm;
  background: $bg-subtle;
  font-size: 12.5px;
  font-weight: 600;
  color: $text-secondary;
  cursor: pointer;

  input[type='checkbox'] {
    accent-color: $primary;
  }
}

/* ---------- nested agent framework picker (TypeStep) ---------- */
&__type-card-wrap {
  display: flex;
  flex-direction: column;
}

&__agent-framework {
  margin-top: 10px;
  padding: 14px 16px 16px;
  border: 1px dashed $border-default;
  border-radius: $radius-md;
  background: $bg-subtle;
}

&__agent-framework-label {
  margin: 0 0 10px;
  font-size: 11.5px;
  font-weight: 700;
  color: $text-tertiary;
  text-transform: uppercase;
  letter-spacing: 0.04em;
}

&__agent-framework-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
}

&__agent-framework-card {
  position: relative;
  display: flex;
  align-items: flex-start;
  gap: 10px;
  padding: 12px 14px;
  border: 1px solid $border-default;
  border-radius: $radius-sm;
  background: $bg-main;
  cursor: pointer;
  text-align: left;
  transition: border-color 0.15s, box-shadow 0.15s;

  &:hover {
    border-color: $primary-subtle;
  }

  &--selected {
    border-color: $primary;
    box-shadow: 0 0 0 1px $primary;
    background: $primary-light;
  }

  .run-eval__type-check {
    position: absolute;
    top: 8px;
    right: 8px;
  }
}

&__agent-framework-icon {
  flex-shrink: 0;
  width: 30px;
  height: 30px;
  border-radius: $radius-sm;
  background: $primary-light;
  color: $primary;
  display: flex;
  align-items: center;
  justify-content: center;
}

&__agent-framework-info {
  display: flex;
  flex-direction: column;
  gap: 2px;
  padding-right: 14px;
}

&__agent-framework-name {
  font-size: 13px;
  font-weight: 700;
  color: $text-primary;
}

&__agent-framework-desc {
  font-size: 11px;
  color: $text-tertiary;
  line-height: 1.35;
}

&__dataset-top-actions {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-shrink: 0;
  position: relative;
  z-index: 2;

  /* The shared .run-eval__type-check badge is absolutely positioned in the
     corner on other steps (Type/Providers/Models), which makes it overlap
     the subgroup icon here. Force it back into normal flow inside this
     specific container so both are clickable/visible side by side. */
  .run-eval__type-check {
    position: static;
    transform: none;
  }
}

&__dataset-subgroups-btn {
  position: relative;
  z-index: 3;
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
  pointer-events: auto;
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
