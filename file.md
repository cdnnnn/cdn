import { useMemo, useState, type FC } from 'react';
import { Search, SlidersHorizontal, X } from 'lucide-react';
import { FILTERS, MODELS } from '../RunEvaluation/data';
import type { EvalTypeId, ModelDifficulty } from '../RunEvaluation/types';

const DIFFICULTY_LABELS: Record<ModelDifficulty, string> = { easy: 'Easy', medium: 'Medium', hard: 'Hard' };
const EVAL_TYPE_LABELS: Record<EvalTypeId, string> = { model: 'Model', agent: 'Agent', rag: 'RAG' };

const Models: FC = () => {
  const [query, setQuery] = useState('');
  const [categoryFilters, setCategoryFilters] = useState<string[]>([]);
  const [difficultyFilters, setDifficultyFilters] = useState<ModelDifficulty[]>([]);
  const [evalTypeFilters, setEvalTypeFilters] = useState<EvalTypeId[]>([]);
  const [showFilters, setShowFilters] = useState(true);

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

  const filtered = useMemo(() => {
    return MODELS.filter((m) => {
      if (query && !m.name.toLowerCase().includes(query.toLowerCase()) && !m.provider.toLowerCase().includes(query.toLowerCase())) {
        return false;
      }
      if (categoryFilters.length && !categoryFilters.includes(m.category)) return false;
      if (difficultyFilters.length && !difficultyFilters.includes(m.difficulty)) return false;
      if (evalTypeFilters.length && !evalTypeFilters.includes(m.eval_type)) return false;
      return true;
    });
  }, [query, categoryFilters, difficultyFilters, evalTypeFilters]);

  return (
    <div className="models-page">
      <div className="models-page__header">
        <div>
          <p className="models-page__eyebrow">Workspace</p>
          <h1 className="models-page__title">Models</h1>
          <p className="models-page__subtitle">Browse every model available across your connected providers.</p>
        </div>
      </div>

      <div className="models-page__layout">
        {showFilters && (
          <aside className="models-page__filters">
            <div className="models-page__filters-head">
              <span>Filters</span>
              <button type="button" className="models-page__link" onClick={resetFilters}>
                Reset all
              </button>
            </div>

            <div className="models-page__filter-section">
              <p className="models-page__filter-title">Category</p>
              <div className="models-page__filter-options">
                {FILTERS.category.map((cat) => (
                  <label key={cat} className="models-page__filter-chip">
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

            <div className="models-page__filter-section">
              <p className="models-page__filter-title">Difficulty</p>
              <div className="models-page__filter-options">
                {FILTERS.difficulty.map((d) => (
                  <label key={d} className="models-page__filter-chip">
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

            <div className="models-page__filter-section">
              <p className="models-page__filter-title">Evaluation Type</p>
              <div className="models-page__filter-options">
                {FILTERS.eval_type.map((t) => (
                  <label key={t} className="models-page__filter-chip">
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

        <div className="models-page__main">
          <div className="models-page__search-bar">
            <Search size={15} />
            <input
              type="text"
              placeholder="Search models…"
              value={query}
              onChange={(e) => setQuery(e.target.value)}
            />
            <button type="button" className="models-page__btn" onClick={() => setShowFilters((v) => !v)}>
              <SlidersHorizontal size={14} /> Filters{activeFilterCount > 0 ? ` (${activeFilterCount})` : ''}
            </button>
          </div>

          {activeFilterCount > 0 && (
            <div className="models-page__active-filters">
              {categoryFilters.map((c) => (
                <span key={`cat-${c}`} className="models-page__tag">
                  {c}
                  <button type="button" onClick={() => toggleFrom(categoryFilters, setCategoryFilters, c)}>
                    <X size={11} />
                  </button>
                </span>
              ))}
              {difficultyFilters.map((d) => (
                <span key={`diff-${d}`} className="models-page__tag">
                  {DIFFICULTY_LABELS[d]}
                  <button type="button" onClick={() => toggleFrom(difficultyFilters, setDifficultyFilters, d)}>
                    <X size={11} />
                  </button>
                </span>
              ))}
              {evalTypeFilters.map((t) => (
                <span key={`type-${t}`} className="models-page__tag">
                  {EVAL_TYPE_LABELS[t]}
                  <button type="button" onClick={() => toggleFrom(evalTypeFilters, setEvalTypeFilters, t)}>
                    <X size={11} />
                  </button>
                </span>
              ))}
            </div>
          )}

          <div className="models-page__grid">
            {filtered.map((m) => (
              <div key={m.id} className="models-page__card">
                <div className="models-page__card-top">
                  <span className="models-page__card-name">{m.name}</span>
                  <span className="models-page__card-eval-type">{EVAL_TYPE_LABELS[m.eval_type]}</span>
                </div>
                <span className="models-page__card-provider">{m.provider}</span>
                <div className="models-page__card-tags">
                  <span className="models-page__chip">{m.category}</span>
                  <span className="models-page__chip">{DIFFICULTY_LABELS[m.difficulty]}</span>
                  {m.capabilities.slice(0, 2).map((c) => (
                    <span key={c} className="models-page__chip">
                      {c}
                    </span>
                  ))}
                </div>
                <div className="models-page__card-meta n">
                  <span>{m.contextWindow}</span>
                  <span>{m.speedRating}</span>
                  <span>{m.pricing}</span>
                </div>
                <div className="models-page__card-scores">
                  <div className="models-page__score">
                    <span className="v n">{m.accuracyScore}</span>
                    <span className="l">Accuracy</span>
                  </div>
                  <div className="models-page__score">
                    <span className="v n">{m.agentScore}</span>
                    <span className="l">Agent</span>
                  </div>
                </div>
              </div>
            ))}
            {filtered.length === 0 && <p className="models-page__empty">No models match these filters.</p>}
          </div>
        </div>
      </div>
    </div>
  );
};

export default Models;




























@import '../../../styles/variables';

.models-page {
  display: flex;
  flex-direction: column;
  gap: 20px;

  &__header {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
  }

  &__eyebrow {
    margin: 0 0 2px;
    font-size: 12px;
    font-weight: 600;
    color: $text-tertiary;
    text-transform: uppercase;
    letter-spacing: 0.04em;
  }

  &__title {
    margin: 0 0 4px;
    font-size: 20px;
    font-weight: 700;
    color: $text-primary;
  }

  &__subtitle {
    margin: 0;
    font-size: 13.5px;
    color: $text-secondary;
  }

  &__layout {
    display: grid;
    grid-template-columns: 220px 1fr;
    gap: 20px;
    align-items: flex-start;
  }

  /* ---------- filters ---------- */
  &__filters {
    background: $bg-main;
    border: 1px solid $border-default;
    border-radius: $radius-lg;
    padding: 16px;
    display: flex;
    flex-direction: column;
    gap: 18px;
    position: sticky;
    top: 20px;
  }

  &__filters-head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    font-size: 12.5px;
    font-weight: 700;
    color: $text-primary;
  }

  &__link {
    border: none;
    background: none;
    color: $primary;
    font-size: 11.5px;
    font-weight: 600;
    cursor: pointer;
    padding: 0;

    &:hover {
      color: $primary-hover;
    }
  }

  &__filter-title {
    margin: 0 0 8px;
    font-size: 11px;
    font-weight: 700;
    color: $text-tertiary;
    text-transform: uppercase;
    letter-spacing: 0.03em;
  }

  &__filter-options {
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  &__filter-chip {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 12.5px;
    color: $text-secondary;
    cursor: pointer;

    input[type='checkbox'] {
      accent-color: $primary;
      width: 15px;
      height: 15px;
    }
  }

  /* ---------- main ---------- */
  &__main {
    display: flex;
    flex-direction: column;
    gap: 14px;
    min-width: 0;
  }

  &__search-bar {
    display: flex;
    align-items: center;
    gap: 10px;
    border: 1px solid $border-default;
    background: $bg-main;
    border-radius: $radius-sm;
    padding: 10px 14px;
    color: $text-tertiary;

    input {
      flex: 1;
      border: none;
      background: transparent;
      font-size: 13px;
      color: $text-primary;
      outline: none;

      &::placeholder {
        color: $text-tertiary;
      }
    }
  }

  &__btn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    border: 1px solid $border-default;
    background: $bg-main;
    color: $text-secondary;
    padding: 7px 13px;
    border-radius: $radius-sm;
    font-size: 12.5px;
    font-weight: 600;
    cursor: pointer;
    white-space: nowrap;

    &:hover {
      background: $bg-subtle;
    }
  }

  &__active-filters {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
  }

  &__tag {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    background: $primary-light;
    color: $primary;
    font-size: 11.5px;
    font-weight: 600;
    padding: 4px 6px 4px 10px;
    border-radius: 999px;

    button {
      border: none;
      background: none;
      color: $primary;
      display: flex;
      align-items: center;
      cursor: pointer;
      padding: 2px;
    }
  }

  /* ---------- grid ---------- */
  &__grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(260px, 1fr));
    gap: 14px;
  }

  &__card {
    background: $bg-main;
    border: 1px solid $border-default;
    border-radius: $radius-lg;
    padding: 16px 18px;
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  &__card-top {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 8px;
  }

  &__card-name {
    font-size: 13.5px;
    font-weight: 700;
    color: $text-primary;
    line-height: 1.3;
  }

  &__card-eval-type {
    flex-shrink: 0;
    font-size: 10px;
    font-weight: 700;
    color: $primary;
    background: $primary-light;
    padding: 2px 8px;
    border-radius: 999px;
    text-transform: uppercase;
  }

  &__card-provider {
    font-size: 11.5px;
    color: $text-tertiary;
  }

  &__card-tags {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
  }

  &__chip {
    font-size: 10.5px;
    font-weight: 600;
    color: $text-secondary;
    background: $bg-inset;
    padding: 3px 8px;
    border-radius: 999px;
  }

  &__card-meta {
    display: flex;
    gap: 10px;
    font-size: 11px;
    color: $text-tertiary;
  }

  &__card-scores {
    display: flex;
    gap: 0;
    margin-top: 4px;
    border-top: 1px solid $border-subtle;
    padding-top: 8px;
  }

  &__score {
    flex: 1;
    display: flex;
    flex-direction: column;
    gap: 1px;

    .v {
      font-size: 14px;
      font-weight: 700;
      color: $text-primary;
    }

    .l {
      font-size: 10px;
      color: $text-tertiary;
      text-transform: uppercase;
      letter-spacing: 0.02em;
    }
  }

  &__empty {
    grid-column: 1 / -1;
    text-align: center;
    padding: 40px;
    color: $text-tertiary;
    font-size: 13px;
  }
}

















import { useMemo, useState, type FC } from 'react';
import { Search, Trophy, Download } from 'lucide-react';
import { RECENT_EVALUATIONS } from '../shared/evaluations';

const Reports: FC = () => {
  const [query, setQuery] = useState('');
  const [activeId, setActiveId] = useState(RECENT_EVALUATIONS[0]?.id ?? null);

  const filtered = useMemo(() => {
    if (!query.trim()) return RECENT_EVALUATIONS;
    const q = query.toLowerCase();
    return RECENT_EVALUATIONS.filter((ev) => ev.name.toLowerCase().includes(q) || ev.topModel.toLowerCase().includes(q));
  }, [query]);

  const active = useMemo(
    () => RECENT_EVALUATIONS.find((ev) => ev.id === activeId) ?? filtered[0] ?? null,
    [activeId, filtered]
  );

  return (
    <div className="reports">
      <div className="reports__header">
        <div>
          <p className="reports__eyebrow">Workspace</p>
          <h1 className="reports__title">Reports</h1>
          <p className="reports__subtitle">Detailed leaderboards and exportable summaries for every evaluation.</p>
        </div>
        <div className="reports__search">
          <Search size={14} />
          <input
            type="text"
            placeholder="Search reports…"
            value={query}
            onChange={(e) => setQuery(e.target.value)}
          />
        </div>
      </div>

      <div className="reports__grid">
        {filtered.map((ev) => (
          <button
            type="button"
            key={ev.id}
            className={`reports__tile${active?.id === ev.id ? ' reports__tile--active' : ''}`}
            onClick={() => setActiveId(ev.id)}
          >
            <span className="reports__tile-type">{ev.type.split('(')[0].trim()}</span>
            <span className="reports__tile-name">{ev.name}</span>
            <span className="reports__tile-meta">
              <span>{ev.date}</span>
              <span className="reports__tile-score n">{ev.topScore}</span>
            </span>
          </button>
        ))}
      </div>

      {active && (
        <section className="reports__detail">
          <div className="reports__detail-head">
            <div>
              <h2 className="reports__detail-title">{active.name}</h2>
              <p className="reports__detail-sub">
                {active.type} &middot; {active.date}
              </p>
            </div>
            <div className="reports__detail-actions">
              <span className="reports__status-badge">
                <Trophy size={12} /> {active.status}
              </span>
              <button type="button" className="reports__export-btn">
                <Download size={13} /> Export
              </button>
            </div>
          </div>

          <div className="reports__stats">
            <div className="reports__stat">
              <span className="reports__stat-value n">{active.topScore}</span>
              <span className="reports__stat-label">Top Score</span>
            </div>
            <div className="reports__stat">
              <span className="reports__stat-value">{active.topModel}</span>
              <span className="reports__stat-label">Winner</span>
            </div>
            <div className="reports__stat">
              <span className="reports__stat-value n">{active.results.length}</span>
              <span className="reports__stat-label">Models Compared</span>
            </div>
          </div>

          <div className="reports__table-wrap">
            <table className="reports__table">
              <thead>
                <tr>
                  <th style={{ width: 56 }}>Rank</th>
                  <th>Model</th>
                  <th>Provider</th>
                  <th>Score</th>
                  <th>Accuracy</th>
                  <th>Speed</th>
                  <th>Cost</th>
                </tr>
              </thead>
              <tbody>
                {active.results.map((r) => (
                  <tr key={r.rank} className={r.rank === 1 ? 'reports__row-winner' : undefined}>
                    <td>
                      <span className={`reports__rank reports__rank--${r.rank}`}>{r.rank}</span>
                    </td>
                    <td className="reports__cell-strong">{r.model}</td>
                    <td>{r.provider}</td>
                    <td>
                      <span className="reports__score n">{r.score}</span>
                    </td>
                    <td className="n">{r.accuracy}</td>
                    <td className="n">{r.time}</td>
                    <td className="n">{r.cost}</td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        </section>
      )}
    </div>
  );
};

export default Reports;





















@import '../../../styles/variables';

.reports {
  display: flex;
  flex-direction: column;
  gap: 20px;

  &__header {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
    flex-wrap: wrap;
  }

  &__eyebrow {
    margin: 0 0 2px;
    font-size: 12px;
    font-weight: 600;
    color: $text-tertiary;
    text-transform: uppercase;
    letter-spacing: 0.04em;
  }

  &__title {
    margin: 0 0 4px;
    font-size: 20px;
    font-weight: 700;
    color: $text-primary;
  }

  &__subtitle {
    margin: 0;
    font-size: 13.5px;
    color: $text-secondary;
  }

  &__search {
    display: flex;
    align-items: center;
    gap: 8px;
    border: 1px solid $border-default;
    background: $bg-main;
    border-radius: $radius-sm;
    padding: 9px 13px;
    color: $text-tertiary;
    min-width: 240px;

    input {
      flex: 1;
      border: none;
      background: transparent;
      font-size: 13px;
      color: $text-primary;
      outline: none;

      &::placeholder {
        color: $text-tertiary;
      }
    }
  }

  /* ---------- report tiles ---------- */
  &__grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
    gap: 12px;
  }

  &__tile {
    display: flex;
    flex-direction: column;
    gap: 6px;
    text-align: left;
    background: $bg-main;
    border: 1px solid $border-default;
    border-radius: $radius-lg;
    padding: 14px 16px;
    cursor: pointer;
    transition: border-color 0.15s, box-shadow 0.15s;

    &:hover {
      border-color: $border-strong;
    }

    &--active {
      border-color: $primary;
      box-shadow: 0 0 0 1px $primary;
      background: $primary-light;
    }
  }

  &__tile-type {
    font-size: 10px;
    font-weight: 700;
    color: $primary;
    text-transform: uppercase;
    letter-spacing: 0.03em;
  }

  &__tile-name {
    font-size: 13px;
    font-weight: 700;
    color: $text-primary;
  }

  &__tile-meta {
    display: flex;
    justify-content: space-between;
    align-items: center;
    font-size: 11px;
    color: $text-tertiary;
  }

  &__tile-score {
    font-weight: 700;
    color: $primary;
    font-size: 12.5px;
  }

  /* ---------- detail ---------- */
  &__detail {
    background: $bg-main;
    border: 1px solid $border-default;
    border-radius: $radius-lg;
    padding: 22px 24px;
  }

  &__detail-head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
    margin-bottom: 18px;
    flex-wrap: wrap;
  }

  &__detail-title {
    margin: 0 0 4px;
    font-size: 15px;
    font-weight: 700;
    color: $text-primary;
  }

  &__detail-sub {
    margin: 0;
    font-size: 12px;
    color: $text-tertiary;
  }

  &__detail-actions {
    display: flex;
    align-items: center;
    gap: 10px;
  }

  &__status-badge {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    background: $success-subtle;
    color: $success;
    font-size: 11.5px;
    font-weight: 700;
    padding: 5px 12px;
    border-radius: 999px;
    white-space: nowrap;
  }

  &__export-btn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    border: 1px solid $border-default;
    background: $bg-main;
    color: $text-secondary;
    padding: 7px 13px;
    border-radius: $radius-sm;
    font-size: 12.5px;
    font-weight: 600;
    cursor: pointer;

    &:hover {
      background: $bg-subtle;
    }
  }

  &__stats {
    display: flex;
    border: 1px solid $border-default;
    border-radius: $radius-md;
    overflow: hidden;
    margin-bottom: 20px;
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
    font-size: 18px;
    font-weight: 700;
    color: $text-primary;
  }

  &__stat-label {
    font-size: 11px;
    color: $text-tertiary;
    text-transform: uppercase;
    letter-spacing: 0.03em;
  }

  /* ---------- table ---------- */
  &__table-wrap {
    overflow-x: auto;
  }

  &__table {
    width: 100%;
    border-collapse: collapse;
    font-size: 12.5px;

    th {
      text-align: left;
      padding: 9px 12px;
      color: $text-tertiary;
      font-size: 11px;
      font-weight: 600;
      text-transform: uppercase;
      letter-spacing: 0.03em;
      border-bottom: 1px solid $border-default;
    }

    td {
      padding: 10px 12px;
      border-bottom: 1px solid $border-subtle;
      color: $text-secondary;
    }

    tr:last-child td {
      border-bottom: none;
    }
  }

  &__row-winner {
    background: $primary-light;
  }

  &__cell-strong {
    font-weight: 600;
    color: $text-primary;
  }

  &__score {
    font-weight: 700;
    color: $primary;
  }

  &__rank {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 22px;
    height: 22px;
    border-radius: 50%;
    background: $bg-inset;
    color: $text-secondary;
    font-size: 11.5px;
    font-weight: 700;

    &--1 {
      background: $primary;
      color: #fff;
    }
  }
}
