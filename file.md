@use '../../../styles/variables' as *;

.history {
  display: flex;
  flex-direction: column;
  gap: 18px;
  // Caps the page at a comfortable reading/working width and centers it, so
  // on very wide viewports (1800px+) the sidebar/detail columns don't stretch
  // into an unusably wide layout — instead the extra space becomes gutters.
  max-width: 1680px;
  margin-left: auto;
  margin-right: auto;
  // .main-layout__body / .workspace-layout / .workspace-layout__content form an
  // unbroken flex chain sized to the viewport (header/footer are fixed and offset
  // via margin), so height: 100% here resolves to the visible content area.
  // workspace-layout__content also has a 3rem bottom padding, which would
  // otherwise leave a large gap below .history — pull most of that back in,
  // keeping just a small 0.75rem breathing-room strip at the true bottom edge.
  height: calc(100% + 3rem - 0.75rem);
  margin-bottom: calc(-3rem + 0.75rem);
  min-height: 0;

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

  /* ---------- generic buttons ---------- */
  &__btn {
    display: inline-flex;
    align-items: center;
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
    transition: background 0.14s ease, border-color 0.14s ease, color 0.14s ease;

    &:hover {
      border-color: $text-primary;
      color: $text-primary;
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

    &--push {
      margin-left: auto;
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

  /* ---------- master-detail layout ---------- */
  &__layout {
    flex: 1;
    display: grid;
    grid-template-columns: 340px 1fr;
    gap: 16px;
    min-height: 0;
    overflow: hidden;
  }

  /* ---------- sidebar list ---------- */
  &__sidebar {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 16px;
    box-shadow: $shadow-xs;
    height: 100%;
    min-height: 0;
    display: flex;
    flex-direction: column;
    overflow: hidden;
  }

  &__sidebar-list {
    flex: 1;
    display: flex;
    flex-direction: column;
    min-height: 0;
    overflow-y: auto;
  }

  &__item {
    display: flex;
    flex-direction: column;
    gap: 6px;
    padding: 14px 16px;
    text-align: left;
    border: none;
    border-bottom: 1px solid $border-subtle;
    border-left: 3px solid transparent;
    background: $bg-main;
    width: 100%;
    cursor: pointer;
    transition: background 0.12s ease, border-color 0.12s ease;

    &:last-child {
      border-bottom: none;
    }

    &:hover {
      background: $bg-subtle;
    }

    &--active {
      background: $primary-light;
      border-left-color: $primary;

      .history__item-name {
        color: $primary;
      }
    }
  }

  &__item-top {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__item-name {
    font-size: 0.84375rem;
    font-weight: 600;
    color: $text-primary;
    line-height: 1.35;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__item-meta {
    display: flex;
    align-items: center;
    justify-content: space-between;
    font-size: 0.75rem;
    color: $text-tertiary;
  }

  &__item-score {
    font-weight: 700;
    color: $success;
  }

  /* ---------- type badge (shared by sidebar + detail) ---------- */
  &__type-badge {
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

  /* ---------- detail panel ---------- */
  &__detail {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 16px;
    box-shadow: $shadow-xs;
    padding: 26px 28px;
    height: 100%;
    min-height: 0;
    overflow-y: auto;
  }

  &__detail-head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
    padding-bottom: 18px;
    margin-bottom: 20px;
    border-bottom: 1px solid $border-subtle;
    position: sticky;
    top: -26px;
    padding-top: 26px;
    margin-top: -26px;
    background: $bg-main;
    z-index: 1;
  }

  &__detail-head-left {
    display: flex;
    flex-direction: column;
    gap: 8px;

    .history__type-badge {
      width: fit-content;
    }
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

  /* ---------- stat cards ---------- */
  &__stat-row {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 12px;
    margin-bottom: 24px;
  }

  &__stat-card {
    background: $bg-subtle;
    border-radius: 12px;
    padding: 14px 16px;
    display: flex;
    flex-direction: column;
    gap: 4px;
    min-width: 0;
  }

  &__stat-card-label {
    font-size: 0.6875rem;
    font-weight: 600;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $text-tertiary;
  }

  &__stat-card-value {
    font-size: 1.25rem;
    font-weight: 800;
    color: $text-primary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;

    &--sm {
      font-size: 1rem;
      font-weight: 700;
    }

    &--accent {
      color: $success;
    }
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
    &__layout {
      grid-template-columns: 1fr;
      grid-template-rows: 16rem 1fr;
      overflow: visible;
    }

    &__sidebar,
    &__detail {
      height: auto;
      max-height: 22rem;
    }

    &__stat-row {
      grid-template-columns: repeat(2, 1fr);
    }
  }

  @media (max-width: 520px) {
    &__detail {
      padding: 18px 16px;
    }

    &__detail-head {
      flex-direction: column;
    }

    &__stat-row {
      grid-template-columns: 1fr;
    }
  }

  /* ---------- ultra-wide: nudge key text sizes up a touch ---------- */
  @media (min-width: 1800px) {
    &__title {
      font-size: 23px;
    }

    &__subtitle {
      font-size: 0.90625rem;
    }

    &__item-name {
      font-size: 0.90625rem;
    }

    &__detail-name {
      font-size: 1.375rem;
    }

    &__stat-card-value {
      font-size: 1.375rem;
    }

    &__stat-card-value--sm {
      font-size: 1.0625rem;
    }

    &__table {
      font-size: 0.90625rem;
    }

    &__cell-strong {
      font-size: 0.90625rem;
    }
  }
}



























import { useMemo, useState, type FC, type MouseEvent } from 'react';
import { useNavigate } from 'react-router-dom';
import { Play, Search, Copy, Trash2, X, Database, FileBarChart } from 'lucide-react';
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

function typeTint(type: string): 'violet' | 'blue' | 'amber' {
  if (type.includes('Agent')) return 'violet';
  if (type.includes('RAG')) return 'blue';
  return 'amber';
}

function matchesType(evType: string, filter: string) {
  if (filter === 'all') return true;
  if (filter === 'Agent') return evType.includes('Agent');
  if (filter === 'RAG') return evType.includes('RAG');
  return evType.includes('AI Model');
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

  const handleDuplicate = (e: MouseEvent, _id: string) => {
    e.stopPropagation();
    navigate('/app/run-evaluation');
  };

  const handleDelete = (e: MouseEvent, id: string) => {
    e.stopPropagation();
    setItems((prev) => prev.filter((ev) => ev.id !== id));
    if (selectedId === id) setSelectedId(null);
  };

  return (
    <div className="history">
      <div className="history__header">
        <div className="history__header-left">
          <p className="history__header-eyebrow">Evaluation records</p>
          <h1 className="history__title">History</h1>
          <p className="history__subtitle">Past evaluations</p>
        </div>

        <div className="history__header-meta">
          <Database size={13} />
          {items.length} evaluations logged
        </div>
      </div>

      <div className="history__filters">
        <div className="history__search">
          <Search size={15} />
          <input type="text" placeholder="Search evaluations..." value={query} onChange={(e) => setQuery(e.target.value)} />
          {query && (
            <button type="button" className="history__search-clear" onClick={() => setQuery('')} aria-label="Clear search">
              <X size={13} />
            </button>
          )}
        </div>
        <Select value={typeFilter} options={TYPE_FILTERS} onChange={setTypeFilter} width={150} />
        <Select value={dateFilter} options={DATE_FILTERS} onChange={setDateFilter} width={160} />

        <button type="button" className="history__btn history__btn--primary history__btn--push" onClick={() => navigate('/app/run-evaluation')}>
          <Play size={14} strokeWidth={2.25} /> New Evaluation
        </button>
      </div>

      {filtered.length === 0 ? (
        <div className="history__empty">
          <Search size={22} />
          <p>No evaluations match your filters.</p>
        </div>
      ) : (
        <div className="history__layout">
          <div className="history__sidebar">
            <div className="history__sidebar-list">
              {filtered.map((ev) => {
                const tint = typeTint(ev.type);
                const isActive = selected?.id === ev.id;
                return (
                  <button
                    type="button"
                    key={ev.id}
                    className={`history__item${isActive ? ' history__item--active' : ''}`}
                    onClick={() => setSelectedId(ev.id)}
                  >
                    <div className="history__item-top">
                      <span className="history__item-name">{ev.name}</span>
                      <span className={`history__type-badge history__type-badge--${tint}`}>
                        {ev.type.split('(')[0].trim()}
                      </span>
                    </div>
                    <div className="history__item-meta">
                      <span>{ev.date}</span>
                      <span className="history__item-score n">{ev.topScore}</span>
                    </div>
                  </button>
                );
              })}
            </div>
          </div>

          {selected && (
            <div className="history__detail">
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
                  <button
                    type="button"
                    className="history__btn history__btn--danger"
                    onClick={(e) => handleDelete(e, selected.id)}
                  >
                    <Trash2 size={13} /> Delete
                  </button>
                  <button type="button" className="history__btn history__btn--primary" onClick={() => navigate('/app/reports')}>
                    <FileBarChart size={13} /> View Report
                  </button>
                </div>
              </div>

              <div className="history__stat-row">
                <div className="history__stat-card">
                  <span className="history__stat-card-label">Models Tested</span>
                  <span className="history__stat-card-value n">{selected.modelsTested}</span>
                </div>
                <div className="history__stat-card">
                  <span className="history__stat-card-label">Top Model</span>
                  <span className="history__stat-card-value history__stat-card-value--sm">{selected.topModel}</span>
                </div>
                <div className="history__stat-card">
                  <span className="history__stat-card-label">Top Score</span>
                  <span className="history__stat-card-value history__stat-card-value--accent n">{selected.topScore}</span>
                </div>
                <div className="history__stat-card">
                  <span className="history__stat-card-label">Status</span>
                  <span className="history__stat-card-value history__stat-card-value--sm">{selected.status}</span>
                </div>
              </div>

              <p className="history__section-title">Full results</p>
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
