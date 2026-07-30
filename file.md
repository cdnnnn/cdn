import { useMemo, useState, type FC } from 'react';
import { useNavigate } from 'react-router-dom';
import { Search, Play, Trophy } from 'lucide-react';
import { RECENT_EVALUATIONS } from '../shared/evaluations';
import './History.scss';

const History: FC = () => {
  const navigate = useNavigate();
  const [query, setQuery] = useState('');
  const [activeId, setActiveId] = useState(RECENT_EVALUATIONS[0]?.id ?? null);

  const filtered = useMemo(() => {
    if (!query.trim()) return RECENT_EVALUATIONS;
    const q = query.toLowerCase();
    return RECENT_EVALUATIONS.filter(
      (ev) => ev.name.toLowerCase().includes(q) || ev.topModel.toLowerCase().includes(q)
    );
  }, [query]);

  const active = useMemo(
    () => RECENT_EVALUATIONS.find((ev) => ev.id === activeId) ?? filtered[0] ?? null,
    [activeId, filtered]
  );

  return (
    <div className="history">
      <div className="history__header">
        <div>
          <p className="history__eyebrow">Workspace</p>
          <h1 className="history__title">Evaluation History</h1>
          <p className="history__subtitle">Browse past runs and drill into full leaderboards.</p>
        </div>
        <button type="button" className="history__new-btn" onClick={() => navigate('/app/run-evaluation')}>
          <Play size={15} strokeWidth={2.25} />
          New Evaluation
        </button>
      </div>

      <div className="history__split">
        <aside className="history__list">
          <div className="history__search">
            <Search size={14} />
            <input
              type="text"
              placeholder="Search evaluations…"
              value={query}
              onChange={(e) => setQuery(e.target.value)}
            />
          </div>

          <div className="history__rows">
            {filtered.map((ev) => (
              <button
                type="button"
                key={ev.id}
                className={`history__row${active?.id === ev.id ? ' history__row--active' : ''}`}
                onClick={() => setActiveId(ev.id)}
              >
                <span className="history__row-name">{ev.name}</span>
                <span className="history__row-meta">
                  <span className="history__row-type">{ev.type.split('(')[0].trim()}</span>
                  <span className="history__row-date n">{ev.date}</span>
                </span>
              </button>
            ))}

            {filtered.length === 0 && <p className="history__empty">No evaluations match “{query}”.</p>}
          </div>
        </aside>

        <section className="history__detail">
          {active ? (
            <>
              <div className="history__detail-head">
                <div>
                  <h2 className="history__detail-title">{active.name}</h2>
                  <p className="history__detail-sub">
                    {active.type} &middot; {active.date}
                  </p>
                </div>
                <span className="history__status-badge">
                  <Trophy size={12} /> {active.status}
                </span>
              </div>

              <div className="history__stats">
                <div className="history__stat">
                  <span className="history__stat-value n">{active.topScore}</span>
                  <span className="history__stat-label">Top Score</span>
                </div>
                <div className="history__stat">
                  <span className="history__stat-value">{active.topModel}</span>
                  <span className="history__stat-label">Winner</span>
                </div>
                <div className="history__stat">
                  <span className="history__stat-value n">{active.results.length}</span>
                  <span className="history__stat-label">Models Compared</span>
                </div>
              </div>

              <div className="history__table-wrap">
                <table className="history__table">
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
                      <tr key={r.rank} className={r.rank === 1 ? 'history__row-winner' : undefined}>
                        <td>
                          <span className={`history__rank history__rank--${r.rank}`}>{r.rank}</span>
                        </td>
                        <td className="history__cell-strong">{r.model}</td>
                        <td>{r.provider}</td>
                        <td>
                          <span className="history__score n">{r.score}</span>
                        </td>
                        <td className="n">{r.accuracy}</td>
                        <td className="n">{r.time}</td>
                        <td className="n">{r.cost}</td>
                      </tr>
                    ))}
                  </tbody>
                </table>
              </div>
            </>
          ) : (
            <p className="history__empty">Select an evaluation to see its results.</p>
          )}
        </section>
      </div>
    </div>
  );
};

export default History;





















@import '../../../styles/variables';

.history {
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

  &__new-btn {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    padding: 9px 16px;
    border-radius: $radius-sm;
    border: none;
    background: $primary;
    color: #fff;
    font-size: 13.5px;
    font-weight: 600;
    cursor: pointer;
    white-space: nowrap;

    &:hover {
      background: $primary-hover;
    }
  }

  &__split {
    display: grid;
    grid-template-columns: 340px 1fr;
    border: 1px solid $border-default;
    border-radius: $radius-lg;
    overflow: hidden;
    background: $bg-main;
    min-height: 560px;
  }

  /* ---------- left list ---------- */
  &__list {
    border-right: 1px solid $border-default;
    background: $bg-subtle;
    display: flex;
    flex-direction: column;
    min-height: 0;
  }

  &__search {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 12px 14px;
    border-bottom: 1px solid $border-default;
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

  &__rows {
    overflow-y: auto;
    flex: 1;
  }

  &__row {
    width: 100%;
    display: flex;
    flex-direction: column;
    gap: 4px;
    text-align: left;
    padding: 13px 16px;
    border: none;
    border-bottom: 1px solid $border-subtle;
    background: transparent;
    cursor: pointer;

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
    font-size: 13.5px;
    font-weight: 600;
    color: $text-primary;
  }

  &__row-meta {
    display: flex;
    align-items: center;
    justify-content: space-between;
    font-size: 11.5px;
    color: $text-tertiary;
  }

  &__row-type {
    background: $bg-inset;
    border-radius: 999px;
    padding: 2px 8px;
    font-weight: 600;
  }

  /* ---------- right detail ---------- */
  &__detail {
    padding: 24px 28px;
    overflow-y: auto;
    min-height: 0;
  }

  &__detail-head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
    margin-bottom: 18px;
  }

  &__detail-title {
    margin: 0 0 4px;
    font-size: 17px;
    font-weight: 700;
    color: $text-primary;
  }

  &__detail-sub {
    margin: 0;
    font-size: 12.5px;
    color: $text-tertiary;
  }

  &__status-badge {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 4px 10px;
    border-radius: 999px;
    background: $success-subtle;
    color: $success;
    font-size: 12px;
    font-weight: 600;
    white-space: nowrap;
  }

  &__stats {
    display: flex;
    border: 1px solid $border-default;
    border-radius: $radius-md;
    overflow: hidden;
    margin-bottom: 22px;
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

  /* ---------- results table ---------- */
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

  &__empty {
    padding: 24px;
    text-align: center;
    color: $text-tertiary;
    font-size: 13px;
  }
}
