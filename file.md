import { useMemo, useState, type FC } from 'react';
import { Search, SlidersHorizontal, Check, X } from 'lucide-react';
import { FILTERS, MODELS } from '../data';
import type { ModelDifficulty, ModelInfo, EvalTypeId } from '../types';

interface Props {
  providers: string[];
  selected: string[];
  onToggle: (id: string) => void;
  onClear: () => void;
}

const DIFFICULTY_LABELS: Record<ModelDifficulty, string> = { easy: 'Easy', medium: 'Medium', hard: 'Hard' };
const EVAL_TYPE_LABELS: Record<EvalTypeId, string> = { model: 'Model', agent: 'Agent', rag: 'RAG' };

const ModelsStep: FC<Props> = ({ providers, selected, onToggle, onClear }) => {
  const [query, setQuery] = useState('');
  const [categoryFilters, setCategoryFilters] = useState<string[]>([]);
  const [difficultyFilters, setDifficultyFilters] = useState<ModelDifficulty[]>([]);
  const [evalTypeFilters, setEvalTypeFilters] = useState<EvalTypeId[]>([]);
  const [providersOnly, setProvidersOnly] = useState(false);
  const [showFilters, setShowFilters] = useState(true);

  // All connected models are shown by default. "Providers only" is an optional,
  // explicit toggle rather than an implicit hard filter — otherwise models from
  // providers you haven't yet ticked in Step 3 silently disappear here.
  const pool = useMemo(
    () => (providersOnly && providers.length ? MODELS.filter((m) => providers.includes(m.providerId)) : MODELS),
    [providersOnly, providers]
  );

  const filtered = useMemo(() => {
    return pool.filter((m: ModelInfo) => {
      if (query && !m.name.toLowerCase().includes(query.toLowerCase()) && !m.provider.toLowerCase().includes(query.toLowerCase())) {
        return false;
      }
      if (categoryFilters.length && !categoryFilters.includes(m.category)) return false;
      if (difficultyFilters.length && !difficultyFilters.includes(m.difficulty)) return false;
      if (evalTypeFilters.length && !evalTypeFilters.includes(m.eval_type)) return false;
      return true;
    });
  }, [pool, query, categoryFilters, difficultyFilters, evalTypeFilters]);

  const toggleFrom = <T,>(list: T[], setList: (v: T[]) => void, value: T) => {
    setList(list.includes(value) ? list.filter((v) => v !== value) : [...list, value]);
  };

  const resetFilters = () => {
    setCategoryFilters([]);
    setDifficultyFilters([]);
    setEvalTypeFilters([]);
    setQuery('');
  };

  const activeFilterCount = categoryFilters.length + difficultyFilters.length + evalTypeFilters.length;

  return (
    <div className="run-eval__card run-eval__card--wide">
      <h2 className="run-eval__step-title">Choose models</h2>
      <p className="run-eval__step-desc">Select the models you want to compare. Use filters to narrow the list.</p>

      <div className="run-eval__models-layout">
        {showFilters && (
          <aside className="run-eval__filters">
            <div className="run-eval__filters-head">
              <span>Filters</span>
              <button type="button" className="run-eval__link" onClick={resetFilters}>
                Reset all
              </button>
            </div>

            {providers.length > 0 && (
              <label className="run-eval__filter-chip run-eval__filter-chip--toggle">
                <input
                  type="checkbox"
                  checked={providersOnly}
                  onChange={(e) => setProvidersOnly(e.target.checked)}
                />
                Selected providers only
              </label>
            )}

            <div className="run-eval__filter-section">
              <p className="run-eval__filter-title">Category</p>
              <div className="run-eval__filter-options">
                {FILTERS.category.map((cat) => (
                  <label key={cat} className="run-eval__filter-chip">
                    <input
                      type="checkbox"
                      checked={categoryFilters.includes(cat)}
                      onChange={() => toggleFrom(categoryFilters, setCategoryFilters, cat)}
                    />
                    {cat}
                  </label>
                ))}
              </div>
            </div>

            <div className="run-eval__filter-section">
              <p className="run-eval__filter-title">Difficulty</p>
              <div className="run-eval__filter-options">
                {FILTERS.difficulty.map((d) => (
                  <label key={d} className="run-eval__filter-chip">
                    <input
                      type="checkbox"
                      checked={difficultyFilters.includes(d)}
                      onChange={() => toggleFrom(difficultyFilters, setDifficultyFilters, d)}
                    />
                    {DIFFICULTY_LABELS[d]}
                  </label>
                ))}
              </div>
            </div>

            <div className="run-eval__filter-section">
              <p className="run-eval__filter-title">Evaluation Type</p>
              <div className="run-eval__filter-options">
                {FILTERS.eval_type.map((t) => (
                  <label key={t} className="run-eval__filter-chip">
                    <input
                      type="checkbox"
                      checked={evalTypeFilters.includes(t)}
                      onChange={() => toggleFrom(evalTypeFilters, setEvalTypeFilters, t)}
                    />
                    {EVAL_TYPE_LABELS[t]}
                  </label>
                ))}
              </div>
            </div>
          </aside>
        )}

        <div className="run-eval__models-main">
          <div className="run-eval__search-bar">
            <Search size={15} />
            <input
              type="text"
              placeholder="Search models..."
              value={query}
              onChange={(e) => setQuery(e.target.value)}
            />
            <button type="button" className="run-eval__btn run-eval__btn--sm" onClick={() => setShowFilters((v) => !v)}>
              <SlidersHorizontal size={14} /> Filters{activeFilterCount > 0 ? ` (${activeFilterCount})` : ''}
            </button>
          </div>

          {activeFilterCount > 0 && (
            <div className="run-eval__active-filters">
              {categoryFilters.map((c) => (
                <span key={`cat-${c}`} className="run-eval__tag">
                  {c}
                  <button type="button" onClick={() => toggleFrom(categoryFilters, setCategoryFilters, c)}>
                    <X size={11} />
                  </button>
                </span>
              ))}
              {difficultyFilters.map((d) => (
                <span key={`diff-${d}`} className="run-eval__tag">
                  {DIFFICULTY_LABELS[d]}
                  <button type="button" onClick={() => toggleFrom(difficultyFilters, setDifficultyFilters, d)}>
                    <X size={11} />
                  </button>
                </span>
              ))}
              {evalTypeFilters.map((t) => (
                <span key={`type-${t}`} className="run-eval__tag">
                  {EVAL_TYPE_LABELS[t]}
                  <button type="button" onClick={() => toggleFrom(evalTypeFilters, setEvalTypeFilters, t)}>
                    <X size={11} />
                  </button>
                </span>
              ))}
            </div>
          )}

          <div className="run-eval__models-grid">
            {filtered.map((m) => {
              const isSelected = selected.includes(m.id);
              return (
                <button
                  key={m.id}
                  type="button"
                  className={`run-eval__model-card${isSelected ? ' run-eval__model-card--selected' : ''}`}
                  onClick={() => onToggle(m.id)}
                >
                  <div className="run-eval__model-top">
                    <span className="run-eval__model-name">{m.name}</span>
                    {isSelected && (
                      <span className="run-eval__type-check">
                        <Check size={12} strokeWidth={2.75} />
                      </span>
                    )}
                  </div>
                  <span className="run-eval__model-provider">{m.provider}</span>
                  <div className="run-eval__model-caps">
                    <span className="run-eval__chip run-eval__chip--static">{m.category}</span>
                    <span className="run-eval__chip run-eval__chip--static">{DIFFICULTY_LABELS[m.difficulty]}</span>
                    {m.capabilities.slice(0, 2).map((c) => (
                      <span key={c} className="run-eval__chip run-eval__chip--static">
                        {c}
                      </span>
                    ))}
                  </div>
                  <div className="run-eval__model-meta n">
                    <span>{m.contextWindow}</span>
                    <span>{m.speedRating}</span>
                    <span>{m.pricing}</span>
                  </div>
                </button>
              );
            })}
            {filtered.length === 0 && <p className="run-eval__empty">No models match these filters.</p>}
          </div>

          {selected.length > 0 && (
            <div className="run-eval__selected-bar">
              <span>
                <strong>{selected.length}</strong> models selected
              </span>
              <button type="button" className="run-eval__btn run-eval__btn--sm" onClick={onClear}>
                Clear
              </button>
            </div>
          )}
        </div>
      </div>
    </div>
  );
};

export default ModelsStep;


















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
