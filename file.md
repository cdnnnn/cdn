//History.tsx
import { useMemo, useState, type FC, type MouseEvent } from 'react';
import { useNavigate } from 'react-router-dom';
import { Play, Search, Copy, Trash2, X, Bot, MessageSquare } from 'lucide-react';
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

const History: FC = () => {
  const navigate = useNavigate();
  const [items, setItems] = useState<RecentEvaluation[]>(RECENT_EVALUATIONS);
  const [query, setQuery] = useState('');
  const [typeFilter, setTypeFilter] = useState('all');
  const [dateFilter, setDateFilter] = useState(30);

  const filtered = useMemo(() => {
    return items.filter((ev) => {
      if (query && !ev.name.toLowerCase().includes(query.toLowerCase())) return false;
      if (!matchesType(ev.type, typeFilter)) return false;
      if (ev.daysAgo > dateFilter) return false;
      return true;
    });
  }, [items, query, typeFilter, dateFilter]);

  // Mirrors viewEvaluationResult(id): populate the results view and navigate there.
  const handleOpen = (ev: RecentEvaluation) => {
    navigate('/app/results', { state: { evaluation: ev } });
  };

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
        <div className="history__list">
          {filtered.map((ev) => (
            <div key={ev.id} className="history__item" onClick={() => handleOpen(ev)}>
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
          ))}
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
  gap: 20px;

  /* ---------- header ---------- */
  &__header {
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

    &--sm {
      padding: 6px;

      svg {
        width: 16px;
        height: 16px;
      }
    }
  }

  /* ---------- filters ---------- */
  &__filters {
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

  /* ---------- list ---------- */
  &__list {
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  &__item {
    display: flex;
    align-items: center;
    gap: 16px;
    padding: 18px 20px;
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: $radius-lg;
    cursor: pointer;
    transition: border-color 0.12s ease, box-shadow 0.12s ease;

    &:hover {
      border-color: $primary;
      box-shadow: $shadow-sm;
    }
  }

  &__icon {
    width: 44px;
    height: 44px;
    background: $primary-light;
    border-radius: $radius-md;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-shrink: 0;

    svg {
      width: 22px;
      height: 22px;
      color: $primary;
      stroke-width: 1.5;
    }
  }

  &__content {
    flex: 1;
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
  }

  &__type {
    padding: 2px 8px;
    background: $primary-light;
    color: $primary;
    border-radius: 4px;
    font-weight: 500;
  }

  &__results {
    display: flex;
    gap: 28px;
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

    &--highlight {
      color: $primary;
    }
  }

  &__actions {
    display: flex;
    gap: 6px;
    flex-shrink: 0;
  }

  /* ---------- empty state ---------- */
  &__empty {
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
  @media (max-width: 640px) {
    &__item {
      flex-wrap: wrap;
    }

    &__results {
      width: 100%;
      justify-content: space-between;
      gap: 12px;
    }

    &__actions {
      width: 100%;
      justify-content: flex-end;
    }
  }
}
