import { useMemo, useState, type FC, type MouseEvent } from 'react';
import { useNavigate } from 'react-router-dom';
import { Play, Search, FlaskConical, Copy, Trash2, X } from 'lucide-react';
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

  const filtered = useMemo(() => {
    return items.filter((ev) => {
      if (query && !ev.name.toLowerCase().includes(query.toLowerCase())) return false;
      if (!matchesType(ev.type, typeFilter)) return false;
      if (ev.daysAgo > dateFilter) return false;
      return true;
    });
  }, [items, query, typeFilter, dateFilter]);

  const handleDuplicate = (e: MouseEvent, _id: string) => {
    e.stopPropagation();
    navigate('/app/run-evaluation');
  };

  const handleDelete = (e: MouseEvent, id: string) => {
    e.stopPropagation();
    setItems((prev) => prev.filter((ev) => ev.id !== id));
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
          <input type="text" placeholder="Search evaluations..." value={query} onChange={(e) => setQuery(e.target.value)} />
          {query && (
            <button type="button" className="history__search-clear" onClick={() => setQuery('')} aria-label="Clear search">
              <X size={13} />
            </button>
          )}
        </div>
        <Select value={typeFilter} options={TYPE_FILTERS} onChange={setTypeFilter} width={150} />
        <Select value={dateFilter} options={DATE_FILTERS} onChange={setDateFilter} width={160} />
      </div>

      <div className="history__grid">
        {filtered.map((ev) => {
          const tint = typeTint(ev.type);
          return (
            <div className="history__card" key={ev.id}>
              <div className="history__card-top">
                <span className="history__icon">
                  <FlaskConical size={16} strokeWidth={2} />
                </span>
                <span className={`history__type-badge history__type-badge--${tint}`}>{ev.type.split('(')[0].trim()}</span>
                <div className="history__actions">
                  <button type="button" className="history__icon-btn" title="Duplicate" onClick={(e) => handleDuplicate(e, ev.id)}>
                    <Copy size={12.5} />
                  </button>
                  <button
                    type="button"
                    className="history__icon-btn history__icon-btn--danger"
                    title="Delete"
                    onClick={(e) => handleDelete(e, ev.id)}
                  >
                    <Trash2 size={12.5} />
                  </button>
                </div>
              </div>

              <h4 className="history__name">{ev.name}</h4>
              <span className="history__date">{ev.date}</span>

              <div className="history__results">
                <div className="history__stat">
                  <span className="history__stat-label">Winner</span>
                  <span className="history__stat-value">{ev.topModel}</span>
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
            </div>
          );
        })}

        {filtered.length === 0 && (
          <div className="history__empty">
            <Search size={22} />
            <p>No evaluations match your filters.</p>
          </div>
        )}
      </div>
    </div>
  );
};

export default History;
















@use '../../../styles/variables' as *;

.history {
  display: flex;
  flex-direction: column;
  gap: 18px;

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
    font-size: 0.8125rem;
  }

  &__btn {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    font-family: $font-body;
    font-size: 0.8125rem;
    font-weight: 600;
    padding: 9px 14px;
    border-radius: 8px;
    border: 1px solid transparent;
    cursor: pointer;
    white-space: nowrap;
    transition: background 0.14s ease, border-color 0.14s ease;

    &--primary {
      background: $primary;
      border-color: $primary;
      color: #fff;

      &:hover {
        background: $primary-hover;
        border-color: $primary-hover;
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

  /* ---------- custom dropdown ---------- */
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

  /* ---------- card grid ---------- */
  &__grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(350px, 1fr));
    gap: 14px;
  }

  &__card {
    display: flex;
    flex-direction: column;
    gap: 12px;
    padding: 18px 20px;
    border: 1px solid $border-subtle;
    border-radius: 14px;
    background: $bg-main;
    box-shadow: $shadow-xs;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &:hover {
      border-color: $border-strong;
      box-shadow: $shadow-sm;

      .history__actions {
        opacity: 1;
      }
    }
  }

  &__card-top {
    display: flex;
    align-items: center;
    gap: 10px;
  }

  &__icon {
    width: 34px;
    height: 34px;
    flex-shrink: 0;
    border-radius: 10px;
    background: $bg-inset;
    color: $text-secondary;
    display: grid;
    place-items: center;
  }

  &__type-badge {
    font-size: 0.6875rem;
    font-weight: 600;
    border-radius: 999px;
    padding: 3px 10px;

    &--violet {
      color: #7c3aed;
      background: #f3e8ff;
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

  &__name {
    font-size: 0.90625rem;
    font-weight: 600;
    color: $text-primary;
    line-height: 1.4;
    display: -webkit-box;
    -webkit-line-clamp: 2;
    -webkit-box-orient: vertical;
    overflow: hidden;
  }

  &__date {
    font-size: 0.75rem;
    color: $text-tertiary;
    margin-top: -6px;
  }

  &__results {
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 10px;
    margin-top: 2px;
    padding-top: 13px;
    border-top: 1px solid $border-subtle;
  }

  &__stat {
    display: flex;
    flex-direction: column;
    gap: 4px;
    min-width: 0;
  }

  &__stat-label {
    font-size: 0.625rem;
    font-weight: 600;
    letter-spacing: 0.05em;
    text-transform: uppercase;
    color: $text-tertiary;
  }

  &__stat-value {
    font-size: 0.8125rem;
    font-weight: 600;
    color: $text-secondary;
    white-space: nowrap;
    max-width: 7.5rem;
    overflow: hidden;
    text-overflow: ellipsis;

    &--highlight {
      color: $success;
      font-weight: 700;
    }
  }

  &__actions {
    flex-shrink: 0;
    display: flex;
    gap: 5px;
    margin-left: auto;
    opacity: 0;
    transition: opacity 0.14s ease;
  }

  &__icon-btn {
    width: 27px;
    height: 27px;
    border-radius: 7px;
    border: 1px solid $border-default;
    background: $bg-main;
    color: $text-tertiary;
    display: grid;
    place-items: center;
    cursor: pointer;
    transition: border-color 0.14s ease, color 0.14s ease, background 0.14s ease;

    &:hover {
      border-color: $text-primary;
      color: $text-primary;
    }

    &--danger:hover {
      border-color: $danger;
      color: $danger;
      background: $danger-subtle;
    }
  }

  /* ---------- empty state ---------- */
  &__empty {
    grid-column: 1 / -1;
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
  @media (max-width: 520px) {
    &__grid {
      grid-template-columns: 1fr;
    }
  }
}


















import { useEffect, useRef, useState } from 'react';
import { Check, ChevronDown } from 'lucide-react';

export interface SelectOption<T extends string | number> {
  value: T;
  label: string;
}

interface SelectProps<T extends string | number> {
  value: T;
  options: SelectOption<T>[];
  onChange: (value: T) => void;
  width?: number;
}

export default function Select<T extends string | number>({ value, options, onChange, width = 160 }: SelectProps<T>) {
  const [open, setOpen] = useState(false);
  const ref = useRef<HTMLDivElement>(null);
  const current = options.find((o) => o.value === value) ?? options[0];

  useEffect(() => {
    const handler = (e: globalThis.MouseEvent) => {
      if (ref.current && !ref.current.contains(e.target as Node)) setOpen(false);
    };
    document.addEventListener('mousedown', handler);
    return () => document.removeEventListener('mousedown', handler);
  }, []);

  return (
    <div className="history-select" ref={ref} style={{ width }}>
      <button type="button" className={`history-select__trigger${open ? ' history-select__trigger--open' : ''}`} onClick={() => setOpen((v) => !v)}>
        <span>{current?.label}</span>
        <ChevronDown size={14} className="history-select__chevron" />
      </button>

      {open && (
        <div className="history-select__menu">
          {options.map((o) => (
            <button
              key={String(o.value)}
              type="button"
              className={`history-select__option${o.value === value ? ' history-select__option--active' : ''}`}
              onClick={() => {
                onChange(o.value);
                setOpen(false);
              }}
            >
              {o.label}
              {o.value === value && <Check size={13} strokeWidth={2.5} />}
            </button>
          ))}
        </div>
      )}
    </div>
  );
}
