//History.tsx
import { useMemo, useState, type FC, type MouseEvent } from 'react';
import { useNavigate } from 'react-router-dom';
import { Play, Search, Copy, Trash2, X, Bot, MessageSquare, Trophy, Zap, Wallet, FileBarChart } from 'lucide-react';
import { RECENT_EVALUATIONS, type RecentEvaluation } from '../shared/evaluations';
import Select from './Select';
import './History.scss';

const TYPE_FILTERS = [
  { value: 'all', label: 'All Types' },
  { value: 'AI Model', label: 'AI Model' },
  { value: 'Agent', label: 'Agent' },
  { value: 'RAG', label: 'RAG' },
];

const DATE_FILTERS = [
  { value: 30, label: 'Last 30 days' },
  { value: 7, label: 'Last 7 days' },
  { value: Infinity, label: 'All time' },
];

function matchesType(evType: string, filter: string) {
  if (filter === 'all') return true;
  if (filter === 'Agent') return evType.includes('Agent');
  if (filter === 'RAG') return evType.includes('RAG');
  return evType.includes('AI Model');
}

// Mirrors the icon logic from renderHistory() in the original app.js:
// Agent -> bot, RAG -> search, everything else -> message-square
function HistoryIcon({ type }: { type: string }) {
  if (type.includes('Agent')) return <Bot />;
  if (type.includes('RAG')) return <Search />;
  return <MessageSquare />;
}

function typeTint(type: string): 'violet' | 'blue' | 'amber' {
  if (type.includes('Agent')) return 'violet';
  if (type.includes('RAG')) return 'blue';
  return 'amber';
}

const History: FC = () => {
  const navigate = useNavigate();
  const [items, setItems] = useState<RecentEvaluation[]>(RECENT_EVALUATIONS);
  const [query, setQuery] = useState('');
  const [typeFilter, setTypeFilter] = useState('all');
  const [dateFilter, setDateFilter] = useState(30);
  const [selectedId, setSelectedId] = useState<string | null>(RECENT_EVALUATIONS[0]?.id ?? null);

  const filtered = useMemo(() => {
    return items.filter((ev) => {
      if (query && !ev.name.toLowerCase().includes(query.toLowerCase())) return false;
      if (!matchesType(ev.type, typeFilter)) return false;
      if (ev.daysAgo > dateFilter) return false;
      return true;
    });
  }, [items, query, typeFilter, dateFilter]);

  const selected = useMemo(
    () => filtered.find((ev) => ev.id === selectedId) ?? filtered[0] ?? null,
    [filtered, selectedId]
  );

  // Mirrors duplicateEval(id): send the user into a fresh Run Evaluation flow.
  const handleDuplicate = (e: MouseEvent, _id: string) => {
    e.stopPropagation();
    navigate('/app/run-evaluation');
  };

  // Mirrors deleteEval(id): confirm, then remove from the list.
  const handleDelete = (e: MouseEvent, id: string) => {
    e.stopPropagation();
    if (window.confirm('Delete this evaluation?')) {
      setItems((prev) => prev.filter((ev) => ev.id !== id));
      if (selectedId === id) setSelectedId(null);
    }
  };

  return (
    <div className="history">
      <div className="history__header">
        <div>
          <h1 className="history__title">History</h1>
          <p className="history__subtitle">Past evaluations</p>
        </div>
        <button type="button" className="history__btn history__btn--primary" onClick={() => navigate('/app/run-evaluation')}>
          <Play size={14} strokeWidth={2.25} /> New Evaluation
        </button>
      </div>

      <div className="history__filters">
        <div className="history__search">
          <Search size={15} />
          <input type="text" placeholder="Search..." value={query} onChange={(e) => setQuery(e.target.value)} />
          {query && (
            <button type="button" className="history__search-clear" onClick={() => setQuery('')} aria-label="Clear search">
              <X size={13} />
            </button>
          )}
        </div>
        <Select value={typeFilter} options={TYPE_FILTERS} onChange={setTypeFilter} width={140} />
        <Select value={dateFilter} options={DATE_FILTERS} onChange={setDateFilter} width={140} />
      </div>

      {filtered.length === 0 ? (
        <div className="history__empty">
          <Search size={22} />
          <p>No evaluations match your filters.</p>
        </div>
      ) : (
        <div className="history__body">
          <div className="history__list-panel">
            <div className="history__list">
              {filtered.map((ev) => {
                const isActive = selected?.id === ev.id;
                return (
                  <div
                    key={ev.id}
                    className={`history__item${isActive ? ' history__item--active' : ''}`}
                    onClick={() => setSelectedId(ev.id)}
                  >
                    <div className="history__icon">
                      <HistoryIcon type={ev.type} />
                    </div>

                    <div className="history__content">
                      <h4>{ev.name}</h4>
                      <div className="history__meta">
                        <span className="history__type">{ev.type.split('(')[0].trim()}</span>
                        <span>{ev.date}</span>
                      </div>
                    </div>

                    <div className="history__results">
                      <div className="history__stat">
                        <span className="history__stat-label">Winner</span>
                        <span className="history__stat-value">{ev.topModel.split(' - ')[0]}</span>
                      </div>
                      <div className="history__stat">
                        <span className="history__stat-label">Score</span>
                        <span className="history__stat-value history__stat-value--highlight n">{ev.topScore}</span>
                      </div>
                      <div className="history__stat">
                        <span className="history__stat-label">Models</span>
                        <span className="history__stat-value n">{ev.modelsTested}</span>
                      </div>
                    </div>

                    <div className="history__actions">
                      <button type="button" className="history__btn history__btn--sm" onClick={(e) => handleDuplicate(e, ev.id)} aria-label="Duplicate evaluation">
                        <Copy size={16} />
                      </button>
                      <button type="button" className="history__btn history__btn--sm" onClick={(e) => handleDelete(e, ev.id)} aria-label="Delete evaluation">
                        <Trash2 size={16} />
                      </button>
                    </div>
                  </div>
                );
              })}
            </div>
          </div>

          {selected && (
            <div className="history__detail-panel">
              <div className="history__detail-head">
                <div className="history__detail-head-left">
                  <span className={`history__type-badge history__type-badge--${typeTint(selected.type)}`}>
                    {selected.type.split('(')[0].trim()}
                  </span>
                  <h2 className="history__detail-name">{selected.name}</h2>
                  <span className="history__detail-date">
                    {selected.status === 'Running' ? 'Running' : 'Completed'} &middot; {selected.date}
                  </span>
                </div>

                <div className="history__detail-actions">
                  <button type="button" className="history__btn" onClick={(e) => handleDuplicate(e, selected.id)}>
                    <Copy size={13} /> Duplicate
                  </button>
                  <button type="button" className="history__btn history__btn--danger" onClick={(e) => handleDelete(e, selected.id)}>
                    <Trash2 size={13} /> Delete
                  </button>
                  <button type="button" className="history__btn history__btn--primary" onClick={() => navigate('/app/reports')}>
                    <FileBarChart size={13} /> View Report
                  </button>
                </div>
              </div>

              {/* Summary cards — matches the reference .results-summary-cards / .summary-card design */}
              <div className="history__summary-cards">
                <div className="history__summary-card history__summary-card--winner">
                  <div className="history__summary-icon">
                    <Trophy />
                  </div>
                  <div className="history__summary-content">
                    <div className="history__summary-label">Winner</div>
                    <div className="history__summary-value">{selected.topModel}</div>
                    <div className="history__summary-score">{selected.topScore}</div>
                  </div>
                </div>
                <div className="history__summary-card">
                  <div className="history__summary-icon">
                    <Zap />
                  </div>
                  <div className="history__summary-content">
                    <div className="history__summary-label">Avg Response Time</div>
                    <div className="history__summary-value">{selected.avgResponseTime}</div>
                    <div className="history__summary-score">{selected.modelsTested} models tested</div>
                  </div>
                </div>
                <div className="history__summary-card">
                  <div className="history__summary-icon">
                    <Wallet />
                  </div>
                  <div className="history__summary-content">
                    <div className="history__summary-label">Total Cost</div>
                    <div className="history__summary-value">{selected.costSpent}</div>
                    <div className="history__summary-score">{selected.status}</div>
                  </div>
                </div>
              </div>

              <p className="history__section-title">Full results</p>
              <div className="history__table-wrap">
                <table className="history__table">
                  <thead>
                    <tr>
                      <th style={{ width: 48 }}>Rank</th>
                      <th>Model</th>
                      <th>Provider</th>
                      <th>Score</th>
                      <th>Accuracy</th>
                      <th>Speed</th>
                      <th>Cost</th>
                    </tr>
                  </thead>
                  <tbody>
                    {selected.results.map((r) => (
                      <tr key={r.rank}>
                        <td>
                          <span className={`history__rank-pill${r.rank === 1 ? ' history__rank-pill--1' : ''}`}>{r.rank}</span>
                        </td>
                        <td className="history__cell-strong">{r.model}</td>
                        <td>{r.provider}</td>
                        <td className={`n${r.rank === 1 ? ' history__score-cell' : ''}`}>{r.score}</td>
                        <td className="n">{r.accuracy}</td>
                        <td className="n">{r.time}</td>
                        <td className="n">{r.cost}</td>
                      </tr>
                    ))}
                  </tbody>
                </table>
              </div>
            </div>
          )}
        </div>
      )}
    </div>
  );
};

export default History;
















//History.scss
@use '../../../styles/variables' as *;

.history {
  display: flex;
  flex-direction: column;
  gap: 18px;
  min-height: 0;
  // .main-layout__body / .workspace-layout / .workspace-layout__content form an
  // unbroken flex chain sized to the viewport (header/footer are fixed and offset
  // via margin), so height: 100% here resolves to the visible content area.
  // workspace-layout__content also has a 3rem bottom padding, which would
  // otherwise leave a large gap below .history — pull most of that back in,
  // keeping just a small 0.75rem breathing-room strip at the true bottom edge,
  // and giving the two scrollable panels below a real height to scroll within.
  height: calc(100% + 3rem - 0.75rem);
  margin-bottom: calc(-3rem + 0.75rem);

  /* ---------- header ---------- */
  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 1rem;
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

  /* ---------- generic buttons ---------- */
  &__btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 7px;
    font-family: $font-body;
    font-size: 0.8125rem;
    font-weight: 600;
    padding: 9px 14px;
    border-radius: 8px;
    border: 1px solid $border-default;
    background: $bg-main;
    color: $text-secondary;
    cursor: pointer;
    white-space: nowrap;
    transition: background 0.12s ease, border-color 0.12s ease, color 0.12s ease, box-shadow 0.12s ease;

    &:hover {
      border-color: $primary;
      box-shadow: $shadow-sm;
    }

    &--primary {
      background: $primary;
      border-color: $primary;
      color: #fff;

      &:hover {
        background: $primary-hover;
        border-color: $primary-hover;
        color: #fff;
      }
    }

    &--danger:hover {
      border-color: $danger;
      color: $danger;
      background: $danger-subtle;
    }

    &--sm {
      padding: 6px;

      svg {
        width: 15px;
        height: 15px;
      }
    }
  }

  /* ---------- filters ---------- */
  &__filters {
    flex-shrink: 0;
    display: flex;
    flex-wrap: wrap;
    gap: 10px;
  }

  &__search {
    display: flex;
    align-items: center;
    gap: 9px;
    width: 280px;
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

  /* ---------- custom dropdown (used by <Select />) ---------- */
  &-select {
    position: relative;
    flex-shrink: 0;

    &__trigger {
      width: 100%;
      display: flex;
      align-items: center;
      justify-content: space-between;
      gap: 8px;
      border: 1px solid $border-default;
      border-radius: 10px;
      padding: 9px 12px;
      background: $bg-main;
      font-size: 0.8125rem;
      font-weight: 500;
      font-family: $font-body;
      color: $text-primary;
      cursor: pointer;
      transition: border-color 0.14s ease, box-shadow 0.14s ease;

      &:hover {
        border-color: $border-strong;
      }

      &--open {
        border-color: $primary;
        box-shadow: 0 0 0 3px $primary-light;
      }
    }

    &__chevron {
      flex-shrink: 0;
      color: $text-tertiary;
      transition: transform 0.16s ease;
    }

    &__trigger--open &__chevron {
      transform: rotate(180deg);
    }

    &__menu {
      position: absolute;
      top: calc(100% + 6px);
      left: 0;
      right: 0;
      z-index: 20;
      background: $bg-main;
      border: 1px solid $border-subtle;
      border-radius: 10px;
      box-shadow: $shadow-lg;
      padding: 5px;
      display: flex;
      flex-direction: column;
      gap: 1px;
    }

    &__option {
      display: flex;
      align-items: center;
      justify-content: space-between;
      gap: 8px;
      width: 100%;
      text-align: left;
      padding: 8px 10px;
      border: none;
      border-radius: 7px;
      background: transparent;
      font-size: 0.8125rem;
      font-family: $font-body;
      color: $text-secondary;
      cursor: pointer;
      transition: background 0.12s ease, color 0.12s ease;

      &:hover {
        background: $bg-subtle;
        color: $text-primary;
      }

      &--active {
        color: $primary;
        font-weight: 600;

        svg {
          color: $primary;
        }
      }
    }
  }

  /* ---------- default two-column layout: 400px list + detail, each independently scrollable ---------- */
  &__body {
    flex: 1;
    display: flex;
    gap: 16px;
    min-height: 0;
  }

  &__list-panel {
    flex: 0 0 400px;
    max-width: 400px;
    min-height: 0;
    overflow-y: auto;
  }

  /* ---------- list ---------- */
  &__list {
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  &__item {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 10px;
    padding: 14px 16px;
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: $radius-lg;
    cursor: pointer;
    transition: border-color 0.12s ease, box-shadow 0.12s ease, background 0.12s ease;

    &:hover {
      border-color: $primary;
      box-shadow: $shadow-sm;
    }

    &--active {
      border-color: $primary;
      background: $primary-light;
      box-shadow: $shadow-sm;
    }
  }

  &__icon {
    width: 36px;
    height: 36px;
    background: $primary-light;
    border-radius: $radius-md;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-shrink: 0;

    svg {
      width: 18px;
      height: 18px;
      color: $primary;
      stroke-width: 1.5;
    }
  }

  &__item--active &__icon {
    background: $bg-main;
  }

  &__content {
    flex-basis: calc(100% - 46px);
    flex-grow: 1;
    min-width: 0;

    h4 {
      font-size: 14px;
      font-weight: 500;
      margin-bottom: 4px;
      color: $text-primary;
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
    }
  }

  &__meta {
    display: flex;
    align-items: center;
    gap: 10px;
    font-size: 12px;
    color: $text-tertiary;
    overflow: hidden;

    span:last-child {
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
    }
  }

  &__type {
    flex-shrink: 0;
    padding: 2px 8px;
    background: $primary-light;
    color: $primary;
    border-radius: 4px;
    font-weight: 500;
  }

  &__item--active &__type {
    background: $bg-main;
  }

  &__results {
    display: flex;
    gap: 16px;
    flex-shrink: 0;
  }

  &__stat {
    display: flex;
    flex-direction: column;
  }

  &__stat-label {
    font-size: 10px;
    text-transform: uppercase;
    letter-spacing: 0.4px;
    color: $text-tertiary;
    margin-bottom: 2px;
  }

  &__stat-value {
    font-size: 13px;
    font-weight: 500;
    color: $text-primary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    max-width: 84px;

    &--highlight {
      color: $primary;
    }
  }

  &__actions {
    display: flex;
    gap: 6px;
    flex-shrink: 0;
    margin-left: auto;
  }

  /* ---------- detail panel ---------- */
  &__detail-panel {
    flex: 1;
    min-width: 0;
    min-height: 0;
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 16px;
    box-shadow: $shadow-xs;
    padding: 26px 28px;
    overflow-y: auto;
  }

  &__detail-head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
    padding-bottom: 18px;
    margin-bottom: 22px;
    border-bottom: 1px solid $border-subtle;
  }

  &__detail-head-left {
    display: flex;
    flex-direction: column;
    gap: 8px;
    min-width: 0;
  }

  &__detail-name {
    font-size: 1.25rem;
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $text-primary;
  }

  &__detail-date {
    font-size: 0.8125rem;
    color: $text-tertiary;
  }

  &__detail-actions {
    display: flex;
    gap: 8px;
    flex-shrink: 0;
    flex-wrap: wrap;
  }

  /* ---------- type badge (sidebar + detail) ---------- */
  &__type-badge {
    width: fit-content;
    flex-shrink: 0;
    font-size: 0.625rem;
    font-weight: 700;
    border-radius: 999px;
    padding: 2px 8px;

    &--violet {
      color: $violet;
      background: $violet-light;
    }

    &--blue {
      color: $primary;
      background: $primary-light;
    }

    &--amber {
      color: $warning;
      background: $warning-subtle;
    }
  }

  /* ---------- summary cards — matches reference .results-summary-cards / .summary-card ---------- */
  &__summary-cards {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 16px;
    margin-bottom: 24px;
  }

  &__summary-card {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: $radius-lg;
    padding: 20px;
    display: flex;
    gap: 14px;
    min-width: 0;

    &--winner {
      // Reference uses a fixed warm amber gradient/border for the winner
      // card in both themes (it's a celebratory accent, not a surface
      // token), so this intentionally does not switch with dark mode.
      background: linear-gradient(135deg, #fffbeb 0%, #fef3c7 100%);
      border-color: #fde68a;
    }
  }

  &__summary-icon {
    width: 44px;
    height: 44px;
    background: $bg-subtle;
    border-radius: $radius-md;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-shrink: 0;

    svg {
      width: 22px;
      height: 22px;
      color: $text-secondary;
      stroke-width: 1.5;
    }
  }

  &__summary-card--winner &__summary-icon {
    background: rgba(180, 83, 9, 0.12);

    svg {
      color: #b45309;
    }
  }

  &__summary-content {
    min-width: 0;
  }

  &__summary-label {
    font-size: 11px;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    color: $text-tertiary;
    margin-bottom: 4px;
  }

  &__summary-value {
    font-size: 16px;
    font-weight: 600;
    color: $text-primary;
    margin-bottom: 2px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__summary-score {
    font-size: 13px;
    color: $text-secondary;
  }

  /* ---------- results table ---------- */
  &__section-title {
    font-size: 0.8125rem;
    font-weight: 700;
    color: $text-primary;
    margin-bottom: 12px;
  }

  &__table-wrap {
    overflow-x: auto;
  }

  &__table {
    width: 100%;
    border-collapse: collapse;

    thead th {
      text-align: left;
      font-size: 0.6875rem;
      font-weight: 700;
      letter-spacing: 0.05em;
      text-transform: uppercase;
      color: $text-tertiary;
      padding: 10px 14px;
      background: $bg-subtle;
      white-space: nowrap;

      &:first-child {
        border-radius: 8px 0 0 8px;
      }

      &:last-child {
        border-radius: 0 8px 8px 0;
      }
    }

    tbody td {
      padding: 12px 14px;
      font-size: 0.84375rem;
      color: $text-secondary;
      border-bottom: 1px solid $border-subtle;
      white-space: nowrap;
    }

    tbody tr:last-child td {
      border-bottom: none;
    }
  }

  &__cell-strong {
    font-weight: 600;
    color: $text-primary;
  }

  &__rank-pill {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 22px;
    height: 22px;
    border-radius: 6px;
    background: $bg-inset;
    color: $text-tertiary;
    font-size: 0.6875rem;
    font-weight: 700;

    &--1 {
      background: $primary-light;
      color: $primary;
    }
  }

  &__score-cell {
    color: $success;
    font-weight: 700;
  }

  /* ---------- empty state ---------- */
  &__empty {
    flex-shrink: 0;
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
  @media (max-width: 900px) {
    height: auto;
    margin-bottom: 0;

    &__body {
      flex-direction: column;
    }

    &__list-panel {
      flex: 0 0 auto;
      max-width: none;
      max-height: 16rem;
    }

    &__detail-panel {
      max-height: none;
    }

    &__summary-cards {
      grid-template-columns: 1fr;
    }
  }

  @media (max-width: 520px) {
    &__detail-panel {
      padding: 18px 16px;
    }

    &__detail-head {
      flex-direction: column;
    }
  }
}
