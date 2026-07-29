import { useState, type FC } from 'react';
import { useNavigate } from 'react-router-dom';
import { ArrowLeft, ArrowRight, Play, Check, FlaskConical, Clock3 } from 'lucide-react';
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
  providers: [],
  models: [],
  dataset: null,
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
      return { ...d, type: id, metrics: defaults };
    });
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
          <span className="run-eval__header-icon">
            <FlaskConical size={19} strokeWidth={2} />
          </span>
          <div>
            <h1 className="run-eval__title">New Evaluation</h1>
            <p className="run-eval__subtitle">Compare AI models with standardized tests</p>
          </div>
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
          {step === 2 && <TypeStep value={draft.type} onChange={setType} />}
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
              onSelect={(id) => setDraft((d) => ({ ...d, dataset: id }))}
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



















@use '../../../styles/variables' as *;

.run-eval {
  max-width: 1024px;
  margin: 0 auto;
  height: calc(100vh - 166px);
  display: flex;
  flex-direction: column;
  min-height: 0;

  @media (min-width: 1800px) {
    max-width: 1300px;
  }

  /* ---------- page header ---------- */
  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 1rem;
    padding-bottom: 20px;
    margin-bottom: 20px;
    border-bottom: 1px solid $border-subtle;
  }

  &__header-left {
    display: flex;
    align-items: center;
    gap: 14px;
  }

  &__header-icon {
    width: 42px;
    height: 42px;
    flex-shrink: 0;
    border-radius: 12px;
    background: $primary-light;
    border: 1px solid $primary-subtle;
    color: $primary;
    display: grid;
    place-items: center;
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
  }

  &__title {
    font-size: 25px;
    font-weight: 800;
    letter-spacing: -0.03em;
    color: $text-primary;
    line-height: 1.15;
  }

  &__subtitle {
    margin-top: 3px;
    color: $text-secondary;
    font-size: 0.84375rem;
  }

  /* ---------- buttons ---------- */
  &__btn {
    display: inline-flex;
    align-items: center;
    gap: 0.5rem;
    font-size: 0.90625rem;
    font-weight: 600;
    padding: 0.5625rem 0.9375rem;
    border-radius: 0.5rem;
    border: 1px solid transparent;
    cursor: pointer;
    transition: background 0.14s ease, border-color 0.14s ease, color 0.14s ease;
    font-family: $font-body;

    &--primary {
      background: $primary;
      color: #fff;
      border-color: $primary;

      &:hover {
        background: $primary-hover;
        border-color: $primary-hover;
      }
    }

    &--secondary {
      background: $bg-main;
      color: $text-primary;
      border-color: $border-default;

      &:hover {
        border-color: $text-primary;
      }
    }

    &--lg {
      padding: 0.625rem 1.125rem;
      font-size: 0.90625rem;
    }

    &--sm {
      padding: 0.375rem 0.6875rem;
      font-size: 0.84375rem;
      background: $bg-main;
      color: $text-secondary;
      border-color: $border-default;

      &:hover {
        border-color: $text-primary;
        color: $text-primary;
      }
    }

    &:disabled {
      opacity: 0.5;
      cursor: not-allowed;
    }
  }

  /* ---------- wizard shell ---------- */
  &__wizard {
    position: relative;
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 20px;
    box-shadow: $shadow-md;
    padding: 32px 36px 24px;
    overflow: hidden;
    flex: 1;
    min-height: 0;
    display: flex;
    flex-direction: column;

    &::before {
      content: '';
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      height: 3px;
      background: linear-gradient(90deg, $primary, $primary-hover 60%, $success);
    }
  }

  &__body {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding-right: 4px;
    margin-right: -4px;
  }

  /* ---------- progress tracker ---------- */
  &__tracker {
    position: relative;
    padding-bottom: 28px;
    margin-bottom: 4px;
    flex-shrink: 0;
  }

  &__tracker-bar {
    position: absolute;
    top: 18px;
    left: 18px;
    right: 18px;
    height: 2px;
    background: $border-default;
    border-radius: 2px;
  }

  &__tracker-fill {
    height: 100%;
    background: linear-gradient(90deg, $primary, $primary-hover);
    border-radius: 2px;
    transition: width 0.25s ease;
  }

  &__tracker-nodes {
    position: relative;
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
  }

  &__node {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 8px;
    border: none;
    background: transparent;
    cursor: pointer;
    padding: 0;
    flex: 1;

    &:disabled {
      cursor: default;
    }
  }

  &__node-dot {
    width: 36px;
    height: 36px;
    border-radius: 50%;
    display: grid;
    place-items: center;
    font-size: 0.8125rem;
    font-weight: 700;
    background: $bg-main;
    border: 2px solid $border-default;
    color: $text-tertiary;
    transition: background 0.16s ease, border-color 0.16s ease, color 0.16s ease, transform 0.16s ease;
  }

  &__node-label {
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-tertiary;
    text-align: center;
    max-width: 88px;
    line-height: 1.25;
    transition: color 0.16s ease;
  }

  &__node--active {
    .run-eval__node-dot {
      background: $primary;
      border-color: $primary;
      color: #fff;
      transform: scale(1.12);
      box-shadow: 0 0 0 4px $primary-light;
    }

    .run-eval__node-label {
      color: $primary;
    }
  }

  &__node--complete {
    .run-eval__node-dot {
      background: $success-subtle;
      border-color: $success;
      color: $success;
    }

    .run-eval__node-label {
      color: $text-secondary;
    }

    &:hover .run-eval__node-dot {
      border-color: $success;
      background: $success;
      color: #fff;
    }
  }

  /* ---------- step kicker + heading ---------- */
  &__step-kicker {
    display: flex;
    align-items: center;
    gap: 8px;
    font-family: $font-mono;
    font-size: 0.75rem;
    font-weight: 700;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    color: $primary;
    margin-bottom: 8px;
    flex-shrink: 0;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $primary;
    }
  }

  /* ---------- step cards ---------- */
  &__card {
    &--wide {
      max-width: none;
    }
  }

  &__step-title {
    font-size: 25px;
    font-weight: 800;
    letter-spacing: -0.02em;
    line-height: 1.2;
    color: $text-primary;
  }

  &__step-desc {
    margin-top: 6px;
    font-size: 0.9375rem;
    color: $text-secondary;
    max-width: 608px;
  }

  &__step-header-row {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 1rem;
  }

  &__field {
    max-width: 480px;
    margin-top: 1.75rem;
  }

  &__label {
    display: block;
    font-size: 0.84375rem;
    font-weight: 600;
    color: $text-secondary;
    margin-bottom: 0.4375rem;
  }

  &__input {
    width: 100%;
    border: 1px solid $border-default;
    border-radius: 0.5rem;
    padding: 0.625rem 0.75rem;
    font-size: 0.9375rem;
    font-family: $font-body;
    color: $text-primary;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &::placeholder {
      color: #a8b1bb;
    }

    &:focus {
      outline: none;
      border-color: $primary;
      box-shadow: 0 0 0 0.1875rem $primary-light;
    }

    &--lg {
      padding: 0.75rem 0.875rem;
      font-size: 1rem;
    }
  }

  /* ---------- suggestion / static chips ---------- */
  &__suggestions {
    margin-top: 1.125rem;
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 0.5rem;
  }

  &__suggestions-label {
    font-size: 0.84375rem;
    color: $text-tertiary;
    margin-right: 0.125rem;
  }

  &__chip {
    font-size: 0.8125rem;
    font-weight: 500;
    color: $text-secondary;
    background: $bg-subtle;
    border: 1px solid $border-default;
    border-radius: 999px;
    padding: 0.3125rem 0.75rem;
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

    &--static {
      cursor: default;
      font-size: 0.75rem;
      padding: 0.1875rem 0.5rem;

      &:hover {
        border-color: $border-default;
        color: $text-secondary;
      }
    }
  }

  /* ---------- eval type cards ---------- */
  &__type-grid {
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
    margin-top: 1.5rem;
  }

  &__type-card {
    position: relative;
    display: flex;
    align-items: flex-start;
    gap: 0.875rem;
    text-align: left;
    width: 100%;
    padding: 1.125rem 3rem 1.125rem 1.125rem;
    border: 1px solid $border-default;
    border-radius: 0.75rem;
    background: $bg-main;
    cursor: pointer;
    transition: border-color 0.14s ease, background 0.14s ease;

    &:hover {
      border-color: $primary;
    }

    &--selected {
      border-color: $primary;
      background: $primary-light;
    }
  }

  &__type-icon {
    width: 38px;
    height: 38px;
    flex-shrink: 0;
    border-radius: 0.5rem;
    background: $bg-subtle;
    color: $primary;
    display: grid;
    place-items: center;
  }

  &__type-content {
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
    flex: 1;
  }

  &__type-title {
    font-size: 1rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__type-desc {
    font-size: 0.875rem;
    color: $text-secondary;
    line-height: 1.5;
  }

  &__type-check {
    position: absolute;
    top: 1.125rem;
    right: 1.125rem;
    width: 20px;
    height: 20px;
    border-radius: 50%;
    background: $primary;
    color: #fff;
    display: grid;
    place-items: center;
  }

  &__badge {
    align-self: flex-start;
    flex-shrink: 0;
    font-size: 0.71875rem;
    font-weight: 600;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $primary;
    background: $primary-light;
    border-radius: 0.375rem;
    padding: 0.25rem 0.5rem;

    &--soft {
      margin-top: 0.625rem;
    }
  }

  /* ---------- providers ---------- */
  &__provider-grid {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 0.75rem;
    margin-top: 1.5rem;
  }

  &__provider-card {
    position: relative;
    display: flex;
    align-items: flex-start;
    gap: 0.75rem;
    text-align: left;
    padding: 0.875rem 2.5rem 0.875rem 0.875rem;
    border: 1px solid $border-default;
    border-radius: 0.75rem;
    background: $bg-main;
    cursor: pointer;
    transition: border-color 0.14s ease, background 0.14s ease;

    &:hover:not(&--disabled) {
      border-color: $primary;
    }

    &--selected {
      border-color: $primary;
      background: $primary-light;
    }

    &--disabled {
      cursor: not-allowed;
      opacity: 0.7;
    }
  }

  &__provider-logo {
    width: 34px;
    height: 34px;
    flex-shrink: 0;
    border-radius: 0.5rem;
    background: $text-primary;
    color: #fff;
    font-weight: 700;
    font-size: 0.875rem;
    display: grid;
    place-items: center;
    margin-top: 0.0625rem;
  }

  &__provider-info {
    display: flex;
    flex-direction: column;
    gap: 0.3125rem;
    min-width: 0;
    flex: 1;
  }

  &__provider-name-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 0.5rem;
  }

  &__provider-name {
    font-size: 0.9375rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__provider-desc {
    font-size: 0.8125rem;
    color: $text-tertiary;
  }

  &__status-badge {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 0.3125rem;
    font-size: 0.71875rem;
    font-weight: 600;
    letter-spacing: 0.01em;
    color: $text-tertiary;
    background: $bg-inset;
    border-radius: 999px;
    padding: 0.1875rem 0.5rem;

    &--on {
      color: $success;
      background: $success-subtle;
    }
  }

  &__hint {
    margin-top: 1.25rem;
    display: flex;
    align-items: center;
    gap: 0.5rem;
    font-size: 0.875rem;
    color: $text-tertiary;

    svg {
      flex-shrink: 0;
    }
  }

  &__link {
    background: none;
    border: none;
    padding: 0;
    color: $primary;
    font-weight: 600;
    font-size: inherit;
    cursor: pointer;

    &:hover {
      text-decoration: underline;
    }
  }

  /* ---------- models step ---------- */
  &__models-layout {
    display: grid;
    grid-template-columns: 15rem 1fr;
    gap: 1.5rem;
    margin-top: 1.5rem;
    align-items: start;
  }

  &__filters {
    border: 1px solid $border-subtle;
    border-radius: 0.75rem;
    padding: 1rem;
    display: flex;
    flex-direction: column;
    gap: 1.125rem;
  }

  &__filters-head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    font-size: 0.875rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__filter-section {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
  }

  &__filter-title {
    font-family: $font-mono;
    font-size: 0.71875rem;
    font-weight: 600;
    letter-spacing: 0.08em;
    text-transform: uppercase;
    color: $text-tertiary;
  }

  &__filter-options {
    display: flex;
    flex-direction: column;
    gap: 0.4375rem;
  }

  &__filter-chip {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    font-size: 0.875rem;
    color: $text-secondary;
    cursor: pointer;

    input {
      accent-color: $primary;
    }
  }

  &__models-main {
    min-width: 0;
  }

  &__search-bar {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    border: 1px solid $border-default;
    border-radius: 0.5rem;
    padding: 0.5625rem 0.75rem;
    color: $text-tertiary;

    input {
      flex: 1;
      border: none;
      outline: none;
      font-size: 0.90625rem;
      color: $text-primary;
      background: transparent;
      font-family: $font-body;

      &::placeholder {
        color: $text-tertiary;
      }
    }
  }

  &__active-filters {
    display: flex;
    flex-wrap: wrap;
    gap: 0.375rem;
    margin-top: 0.75rem;
  }

  &__tag {
    display: inline-flex;
    align-items: center;
    gap: 0.375rem;
    font-size: 0.78125rem;
    color: $primary;
    background: $primary-light;
    border-radius: 0.375rem;
    padding: 0.25rem 0.25rem 0.25rem 0.5rem;

    button {
      display: grid;
      place-items: center;
      border: none;
      background: transparent;
      color: inherit;
      cursor: pointer;
    }
  }

  &__models-grid {
    margin-top: 1rem;
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 0.75rem;
  }

  &__model-card {
    position: relative;
    text-align: left;
    padding: 0.875rem 1rem;
    border: 1px solid $border-default;
    border-radius: 0.75rem;
    background: $bg-main;
    cursor: pointer;
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    transition: border-color 0.14s ease, background 0.14s ease;

    &:hover {
      border-color: $primary;
    }

    &--selected {
      border-color: $primary;
      background: $primary-light;
    }
  }

  &__model-top {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 0.5rem;
  }

  &__model-name {
    font-size: 0.9375rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__model-provider {
    font-size: 0.84375rem;
    color: $text-tertiary;
    margin-top: -0.25rem;
  }

  &__model-caps {
    display: flex;
    flex-wrap: wrap;
    gap: 0.3125rem;
  }

  &__model-meta {
    display: flex;
    flex-wrap: wrap;
    gap: 0.625rem;
    font-size: 0.78125rem;
    color: $text-tertiary;
    margin-top: 0.125rem;
  }

  &__empty {
    grid-column: 1 / -1;
    padding: 2rem;
    text-align: center;
    color: $text-tertiary;
    font-size: 0.90625rem;
  }

  &__selected-bar {
    margin-top: 1rem;
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0.75rem 1rem;
    background: $primary-light;
    border-radius: 0.625rem;
    font-size: 0.90625rem;
    color: $text-primary;

    strong {
      color: $primary;
    }
  }

  /* ---------- dataset step ---------- */
  &__tabs {
    display: flex;
    gap: 0.375rem;
    margin-top: 1.5rem;
    border-bottom: 1px solid $border-subtle;
  }

  &__tab {
    padding: 0.5625rem 0.25rem;
    margin-right: 1.25rem;
    border: none;
    background: transparent;
    font-size: 0.90625rem;
    font-weight: 600;
    color: $text-tertiary;
    cursor: pointer;
    border-bottom: 2px solid transparent;
    transition: color 0.14s ease, border-color 0.14s ease;

    &--active {
      color: $primary;
      border-bottom-color: $primary;
    }
  }

  &__category-filters {
    display: flex;
    flex-wrap: wrap;
    gap: 0.5rem;
    margin-top: 1.125rem;
  }

  &__dataset-grid {
    margin-top: 1.125rem;
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 0.75rem;
  }

  &__dataset-card {
    position: relative;
    text-align: left;
    padding: 1rem 1.125rem;
    border: 1px solid $border-default;
    border-radius: 0.75rem;
    background: $bg-main;
    cursor: pointer;
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    transition: border-color 0.14s ease, background 0.14s ease;

    &:hover {
      border-color: $primary;
    }

    &--selected {
      border-color: $primary;
      background: $primary-light;
    }
  }

  &__dataset-top {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 0.5rem;
  }

  &__dataset-name {
    font-size: 0.9375rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__dataset-desc {
    font-size: 0.875rem;
    color: $text-secondary;
    line-height: 1.5;
  }

  &__dataset-meta {
    display: flex;
    flex-wrap: wrap;
    gap: 0.625rem;
    font-size: 0.78125rem;
    color: $text-tertiary;
  }

  &__empty-state,
  &__upload-zone {
    margin-top: 1.5rem;
    border: 1.5008px dashed $border-strong;
    border-radius: 0.75rem;
    padding: 2.75rem 1.5rem;
    text-align: center;
    color: $text-tertiary;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 0.5rem;

    svg {
      color: $text-tertiary;
      margin-bottom: 0.25rem;
    }

    h3 {
      font-size: 1rem;
      color: $text-primary;
    }

    p {
      font-size: 0.875rem;
    }
  }

  &__upload-zone {
    cursor: pointer;

    &:hover {
      border-color: $primary;
    }
  }

  &__format-chips {
    display: flex;
    gap: 0.375rem;
    margin-top: 0.375rem;
  }

  /* ---------- metrics step ---------- */
  &__metrics-count {
    flex-shrink: 0;
    font-size: 0.875rem;
    color: $text-secondary;

    span {
      font-weight: 700;
      color: $primary;
    }
  }

  &__metric-group {
    margin-top: 1.5rem;
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
  }

  &__metrics-grid {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 0.625rem;
  }

  &__metric-card {
    position: relative;
    text-align: left;
    padding: 0.75rem 2rem 0.75rem 0.875rem;
    border: 1px solid $border-default;
    border-radius: 0.625rem;
    background: $bg-main;
    cursor: pointer;
    transition: border-color 0.14s ease, background 0.14s ease;

    &:hover {
      border-color: $primary;
    }

    &--selected {
      border-color: $primary;
      background: $primary-light;
    }
  }

  &__metric-name {
    display: block;
    font-size: 0.875rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__metric-tooltip {
    display: block;
    margin-top: 0.25rem;
    font-size: 0.78125rem;
    color: $text-tertiary;
    line-height: 1.4;
  }

  /* ---------- review step ---------- */
  &__review {
    margin-top: 1.5rem;
    border: 1px solid $border-subtle;
    border-radius: 0.75rem;
    overflow: hidden;
  }

  &__review-row {
    display: flex;
    align-items: baseline;
    justify-content: space-between;
    gap: 1rem;
    padding: 0.75rem 1rem;
    font-size: 0.90625rem;
    border-bottom: 1px solid $border-subtle;

    &:last-child {
      border-bottom: 0;
    }

    span:first-child {
      color: $text-tertiary;
      flex-shrink: 0;
    }

    span:last-child {
      color: $text-primary;
      font-weight: 500;
      text-align: right;
    }

    &--highlight span:last-child {
      color: $primary;
      font-weight: 700;
    }
  }

  &__review-divider {
    height: 1px;
    background: $border-subtle;
  }

  /* ---------- shared feedback ---------- */
  &__error {
    margin-top: 1.25rem;
    font-size: 0.875rem;
    color: $danger;
    background: $danger-subtle;
    border-radius: 0.5rem;
    padding: 0.625rem 0.875rem;
  }

  &__nav {
    flex-shrink: 0;
    margin-top: 1.25rem;
    padding-top: 1.25rem;
    border-top: 1px solid $border-subtle;
    display: flex;
    align-items: center;
    justify-content: space-between;
  }

  /* ---------- responsive ---------- */
  @media (max-width: 896px) {
    &__provider-grid,
    &__models-grid,
    &__dataset-grid,
    &__metrics-grid {
      grid-template-columns: 1fr;
    }

    &__models-layout {
      grid-template-columns: 1fr;
    }

    &__node-label {
      display: none;
    }
  }
}
