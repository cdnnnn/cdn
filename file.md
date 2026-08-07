import { useMemo, useState, type FC } from 'react';
import { Search, Check, X } from 'lucide-react';
import { MODELS } from '../data';
import type { ModelInfo } from '../types';

interface Props {
  providers: string[];
  selected: string[];
  onToggle: (id: string) => void;
  onClear: () => void;
}

const ModelsStep: FC<Props> = ({ providers, selected, onToggle, onClear }) => {
  const [query, setQuery] = useState('');
  const [capFilters, setCapFilters] = useState<string[]>([]);

  const pool = useMemo(
    () => (providers.length ? MODELS.filter((m) => providers.includes(m.providerId)) : MODELS),
    [providers]
  );

  const allCapabilities = useMemo(() => {
    const set = new Set<string>();
    pool.forEach((m) => m.capabilities.forEach((c) => set.add(c)));
    return Array.from(set).sort();
  }, [pool]);

  const filtered = useMemo(() => {
    return pool.filter((m: ModelInfo) => {
      if (query && !m.name.toLowerCase().includes(query.toLowerCase()) && !m.provider.toLowerCase().includes(query.toLowerCase())) {
        return false;
      }
      if (capFilters.length && !capFilters.every((c) => m.capabilities.includes(c))) return false;
      return true;
    });
  }, [pool, query, capFilters]);

  const toggleCap = (cap: string) =>
    setCapFilters((prev) => (prev.includes(cap) ? prev.filter((c) => c !== cap) : [...prev, cap]));
  const resetFilters = () => {
    setCapFilters([]);
    setQuery('');
  };

  return (
    <div className="run-eval__card run-eval__card--wide">
      <h2 className="run-eval__step-title">Choose models</h2>
      <p className="run-eval__step-desc">Select the models you want to compare. Use filters to narrow the list.</p>

      <div className="run-eval__models-layout">
        <aside className="run-eval__filters">
          <div className="run-eval__filters-head">
            <span>Filters</span>
            <button type="button" className="run-eval__link" onClick={resetFilters}>
              Reset all
            </button>
          </div>

          <div className="run-eval__filters-scroll">
            <div className="run-eval__filter-section">
              <p className="run-eval__filter-title">Capabilities</p>
              <div className="run-eval__filter-options">
                {allCapabilities.map((cap) => (
                  <label key={cap} className="run-eval__filter-chip">
                    <input type="checkbox" checked={capFilters.includes(cap)} onChange={() => toggleCap(cap)} />
                    {cap}
                  </label>
                ))}
              </div>
            </div>
          </div>
        </aside>

        <div className="run-eval__models-main">
          <div className="run-eval__search-bar">
            <Search size={15} />
            <input
              type="text"
              placeholder="Search models..."
              value={query}
              onChange={(e) => setQuery(e.target.value)}
            />
          </div>

          {capFilters.length > 0 && (
            <div className="run-eval__active-filters">
              {capFilters.map((c) => (
                <span key={c} className="run-eval__tag">
                  {c}
                  <button type="button" onClick={() => toggleCap(c)}>
                    <X size={11} />
                  </button>
                </span>
              ))}
            </div>
          )}

          <div className="run-eval__models-scroll">
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
                      {m.capabilities.slice(0, 3).map((c) => (
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
