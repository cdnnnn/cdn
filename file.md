//Evaluations.tx
export interface EvalResult {
  rank: number;
  model: string;
  provider: string;
  score: string;
  accuracy: string;
  time: string;
  cost: string;
}

export interface RecentEvaluation {
  id: string;
  name: string;
  type: string;
  date: string;
  daysAgo: number;
  modelsTested: number;
  topModel: string;
  topScore: string;
  status: 'Completed' | 'Running';
  results: EvalResult[];
}

export const RECENT_EVALUATIONS: RecentEvaluation[] = [
  {
    id: 'eval-9041',
    name: 'Agent Tool Calling Duel',
    type: 'Autonomous Workflow (Agent)',
    date: '10 mins ago',
    daysAgo: 0,
    modelsTested: 3,
    topModel: 'Model Alpha Agent',
    topScore: '97.5%',
    status: 'Completed',
    results: [
      { rank: 1, model: 'Model Alpha Agent', provider: 'Together AI', score: '97.5%', accuracy: '96.2%', time: '0.75s', cost: '$0.08' },
      { rank: 2, model: 'Model Delta Agent v2', provider: 'Anthropic', score: '96.2%', accuracy: '97.0%', time: '1.12s', cost: '$0.42' },
      { rank: 3, model: 'Model Gamma Agent', provider: 'OpenAI', score: '94.0%', accuracy: '93.8%', time: '0.95s', cost: '$0.35' },
    ],
  },
  {
    id: 'eval-8820',
    name: 'Q3 Financial Audit Verification',
    type: 'General Chat & Text (AI Model)',
    date: 'Yesterday',
    daysAgo: 1,
    modelsTested: 4,
    topModel: 'Model Epsilon Reasoning',
    topScore: '96.8%',
    status: 'Completed',
    results: [
      { rank: 1, model: 'Model Epsilon Reasoning', provider: 'Together AI', score: '96.8%', accuracy: '98.2%', time: '1.85s', cost: '$0.09' },
      { rank: 2, model: 'Model Delta Opus', provider: 'Anthropic', score: '95.1%', accuracy: '96.0%', time: '1.10s', cost: '$0.45' },
      { rank: 3, model: 'Model Gamma Mini', provider: 'OpenAI', score: '92.4%', accuracy: '91.5%', time: '0.88s', cost: '$0.38' },
    ],
  },
  {
    id: 'eval-8715',
    name: 'Support KB Retrieval (RAG)',
    type: 'Document Search & Answering (RAG)',
    date: '3 days ago',
    daysAgo: 3,
    modelsTested: 2,
    topModel: 'Model Theta Long-Context',
    topScore: '95.3%',
    status: 'Completed',
    results: [
      { rank: 1, model: 'Model Theta Long-Context', provider: 'Google Gemini', score: '95.3%', accuracy: '96.5%', time: '1.20s', cost: '$0.31' },
      { rank: 2, model: 'Model Zeta Instruct', provider: 'Groq Cloud', score: '93.1%', accuracy: '91.2%', time: '0.28s', cost: '$0.07' },
    ],
  },
  {
    id: 'eval-8590',
    name: 'Support Bot Tone & Helpfulness',
    type: 'General Chat & Text (AI Model)',
    date: '6 days ago',
    daysAgo: 6,
    modelsTested: 3,
    topModel: 'Model Zeta Instruct',
    topScore: '93.4%',
    status: 'Completed',
    results: [
      { rank: 1, model: 'Model Zeta Instruct', provider: 'Groq Cloud', score: '93.4%', accuracy: '90.8%', time: '0.31s', cost: '$0.06' },
      { rank: 2, model: 'Model Gamma Mini', provider: 'OpenAI', score: '91.2%', accuracy: '89.4%', time: '0.72s', cost: '$0.11' },
      { rank: 3, model: 'Model Theta Flash', provider: 'Google Gemini', score: '90.5%', accuracy: '88.9%', time: '0.45s', cost: '$0.05' },
    ],
  },
  {
    id: 'eval-8402',
    name: 'SWE-bench Patch Accuracy Check',
    type: 'Autonomous Workflow (Agent)',
    date: '12 days ago',
    daysAgo: 12,
    modelsTested: 4,
    topModel: 'Model Delta Agent v2',
    topScore: '95.9%',
    status: 'Completed',
    results: [
      { rank: 1, model: 'Model Delta Agent v2', provider: 'Anthropic', score: '95.9%', accuracy: '97.5%', time: '2.10s', cost: '$0.51' },
      { rank: 2, model: 'Model Alpha Agent', provider: 'Together AI', score: '94.1%', accuracy: '95.0%', time: '0.98s', cost: '$0.14' },
      { rank: 3, model: 'Model Eta Instruct', provider: 'Together AI', score: '90.2%', accuracy: '91.1%', time: '1.30s', cost: '$0.17' },
      { rank: 4, model: 'Model Gamma Agent', provider: 'OpenAI', score: '88.7%', accuracy: '89.9%', time: '1.05s', cost: '$0.39' },
    ],
  },
  {
    id: 'eval-8177',
    name: 'Clinical Triage Safety Baseline',
    type: 'General Chat & Text (AI Model)',
    date: '21 days ago',
    daysAgo: 21,
    modelsTested: 2,
    topModel: 'Model Delta Opus',
    topScore: '97.9%',
    status: 'Completed',
    results: [
      { rank: 1, model: 'Model Delta Opus', provider: 'Anthropic', score: '97.9%', accuracy: '98.6%', time: '2.40s', cost: '$0.63' },
      { rank: 2, model: 'Model Epsilon Reasoning', provider: 'Together AI', score: '95.0%', accuracy: '96.1%', time: '1.90s', cost: '$0.10' },
    ],
  },
];





























//Dashboard.tsx
import type { FC } from 'react';
import { useNavigate } from 'react-router-dom';
import { Play, PlugZap, Upload, Search, ChevronRight, Trophy, TrendingUp } from 'lucide-react';
import { MODELS, PROVIDERS, TEST_SUITES } from '../RunEvaluation/data';
import { RECENT_EVALUATIONS } from '../shared/evaluations';
import './Dashboard.scss';

const QUICK_ACTIONS = [
  { icon: Play, label: 'Run Evaluation', desc: 'Test models on benchmarks', to: '/app/run-evaluation', tint: 'blue' },
  { icon: PlugZap, label: 'Add Provider', desc: 'Connect API keys', to: '/app/providers', tint: 'green' },
  { icon: Upload, label: 'Upload Dataset', desc: 'Custom test questions', to: '/app/datasets', tint: 'amber' },
  { icon: Search, label: 'Browse Models', desc: 'Explore 100+ models', to: '/app/models', tint: 'violet' },
] as const;

const Dashboard: FC = () => {
  const navigate = useNavigate();
  const connectedProviders = PROVIDERS.filter((p) => p.status === 'connected').length;
  const latest = RECENT_EVALUATIONS[0];

  const stats = [
    { value: MODELS.length, label: 'Models' },
    { value: connectedProviders, label: 'Providers' },
    { value: TEST_SUITES.length, label: 'Test Suites' },
    { value: RECENT_EVALUATIONS.length, label: 'Evaluations' },
  ];

  return (
    <div className="dash">
      {/* ---------- hero ---------- */}
      <section className="dash__hero">
        <div className="dash__hero-grid" aria-hidden="true" />
        <div className="dash__hero-content">
          <p className="dash__hero-eyebrow">Evaluation overview</p>
          <h1 className="dash__hero-title">Compare models with confidence</h1>
          <p className="dash__hero-sub">Run standardized tests across providers and let the results guide your next decision.</p>

          <div className="dash__hero-highlight">
            <TrendingUp size={14} />
            Top performer this week: <strong>{latest.topModel}</strong>
            <span className="dash__hero-highlight-score n">{latest.topScore}</span>
          </div>
        </div>
        <button type="button" className="dash__hero-btn" onClick={() => navigate('/app/run-evaluation')}>
          <Play size={15} strokeWidth={2.25} />
          New Evaluation
        </button>
      </section>

      {/* ---------- stats ---------- */}
      <section className="dash__stats">
        {stats.map((s) => (
          <div className="dash__stat-card" key={s.label}>
            <span className="dash__stat-value n">{s.value}</span>
            <span className="dash__stat-label">{s.label}</span>
          </div>
        ))}
      </section>

      {/* ---------- recent + quick actions ---------- */}
      <section className="dash__row">
        <div className="dash__card">
          <div className="dash__card-head">
            <h2 className="dash__card-title">Recent Evaluations</h2>
            <button type="button" className="dash__btn dash__btn--sm" onClick={() => navigate('/app/history')}>
              View All
            </button>
          </div>

          <div className="dash__eval-list">
            {RECENT_EVALUATIONS.slice(0, 3).map((ev) => (
              <button
                type="button"
                key={ev.id}
                className="dash__eval-item"
                onClick={() => navigate('/app/history')}
              >
                <div className="dash__eval-info">
                  <span className="dash__eval-name">{ev.name}</span>
                  <span className="dash__eval-meta">
                    <span className="dash__eval-type-badge">{ev.type.split('(')[0].trim()}</span>
                    <span className="dash__eval-date">{ev.date}</span>
                  </span>
                </div>
                <div className="dash__eval-stats">
                  <span className="dash__eval-stat">
                    <span className="dash__eval-stat-label">Top</span>
                    <span className="dash__eval-stat-value">{ev.topModel}</span>
                  </span>
                  <span className="dash__eval-stat">
                    <span className="dash__eval-stat-label">Score</span>
                    <span className="dash__eval-stat-value dash__eval-stat-value--highlight n">{ev.topScore}</span>
                  </span>
                </div>
                <ChevronRight size={16} className="dash__eval-arrow" />
              </button>
            ))}
          </div>
        </div>

        <div className="dash__card">
          <div className="dash__card-head">
            <h2 className="dash__card-title">Quick Actions</h2>
          </div>
          <div className="dash__qa-grid">
            {QUICK_ACTIONS.map((qa) => (
              <button type="button" key={qa.label} className="dash__qa-card" onClick={() => navigate(qa.to)}>
                <span className={`dash__qa-icon dash__qa-icon--${qa.tint}`}>
                  <qa.icon size={17} strokeWidth={2} />
                </span>
                <span>
                  <span className="dash__qa-title">{qa.label}</span>
                  <span className="dash__qa-desc">{qa.desc}</span>
                </span>
              </button>
            ))}
          </div>
        </div>
      </section>

      {/* ---------- latest results ---------- */}
      <section className="dash__card">
        <div className="dash__card-head">
          <div>
            <h2 className="dash__card-title">Latest Results</h2>
            <p className="dash__card-subtitle">From your most recent evaluation &middot; {latest.name}</p>
          </div>
          <span className="dash__badge">
            <Trophy size={12} /> {latest.status}
          </span>
        </div>

        <div className="dash__table-wrap">
          <table className="dash__table">
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
              {latest.results.map((r) => (
                <tr key={r.rank} className={r.rank === 1 ? 'dash__row-winner' : undefined}>
                  <td>
                    <span className={`dash__rank dash__rank--${r.rank}`}>{r.rank}</span>
                  </td>
                  <td className="dash__cell-strong">{r.model}</td>
                  <td>{r.provider}</td>
                  <td>
                    <span className="dash__score n">{r.score}</span>
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
    </div>
  );
};

export default Dashboard;




















//History.tsx
import { useMemo, useState, type FC, type MouseEvent } from 'react';
import { useNavigate } from 'react-router-dom';
import { Play, Search, Bot, MessageSquare, FileSearch, Copy, Trash2 } from 'lucide-react';
import { RECENT_EVALUATIONS, type RecentEvaluation } from '../shared/evaluations';
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

function typeIcon(type: string) {
  if (type.includes('Agent')) return Bot;
  if (type.includes('RAG')) return FileSearch;
  return MessageSquare;
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
          <Play size={15} strokeWidth={2.25} /> New Evaluation
        </button>
      </div>

      <div className="history__filters">
        <div className="history__search">
          <Search size={15} />
          <input type="text" placeholder="Search evaluations..." value={query} onChange={(e) => setQuery(e.target.value)} />
        </div>
        <select className="history__select" value={typeFilter} onChange={(e) => setTypeFilter(e.target.value)}>
          {TYPE_FILTERS.map((f) => (
            <option key={f.value} value={f.value}>
              {f.label}
            </option>
          ))}
        </select>
        <select className="history__select" value={dateFilter} onChange={(e) => setDateFilter(Number(e.target.value))}>
          {DATE_FILTERS.map((f) => (
            <option key={f.label} value={f.value}>
              {f.label}
            </option>
          ))}
        </select>
      </div>

      <div className="history__list">
        {filtered.map((ev) => {
          const Icon = typeIcon(ev.type);
          return (
            <div className="history__item" key={ev.id}>
              <span className="history__icon">
                <Icon size={17} strokeWidth={2} />
              </span>

              <div className="history__content">
                <h4 className="history__name">{ev.name}</h4>
                <div className="history__meta">
                  <span className="history__type-badge">{ev.type.split('(')[0].trim()}</span>
                  <span className="history__date">{ev.date}</span>
                </div>
              </div>

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

              <div className="history__actions">
                <button type="button" className="history__icon-btn" title="Duplicate" onClick={(e) => handleDuplicate(e, ev.id)}>
                  <Copy size={14} />
                </button>
                <button type="button" className="history__icon-btn history__icon-btn--danger" title="Delete" onClick={(e) => handleDelete(e, ev.id)}>
                  <Trash2 size={14} />
                </button>
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






















//History.scss
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
    font-size: 27px;
    font-weight: 700;
    color: $text-primary;
  }

  &__subtitle {
    margin-top: 0.375rem;
    color: $text-secondary;
    font-size: 0.90625rem;
  }

  &__btn {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    font-family: $font-body;
    font-size: 0.84375rem;
    font-weight: 600;
    padding: 10px 16px;
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
    gap: 8px;
    width: 280px;
    max-width: 100%;
    border: 1px solid $border-default;
    border-radius: 8px;
    padding: 9px 12px;
    background: $bg-main;
    color: $text-tertiary;

    input {
      flex: 1;
      border: none;
      outline: none;
      font-size: 0.84375rem;
      color: $text-primary;
      background: transparent;
      font-family: $font-body;

      &::placeholder {
        color: $text-tertiary;
      }
    }
  }

  &__select {
    width: 150px;
    border: 1px solid $border-default;
    border-radius: 8px;
    padding: 9px 12px;
    font-size: 0.8125rem;
    font-family: $font-body;
    color: $text-primary;
    background: $bg-main;
    cursor: pointer;

    &:focus {
      outline: none;
      border-color: $primary;
      box-shadow: 0 0 0 3px $primary-light;
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
    padding: 16px 18px;
    border: 1px solid $border-subtle;
    border-radius: 14px;
    background: $bg-main;
    box-shadow: $shadow-xs;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &:hover {
      border-color: $border-strong;
      box-shadow: $shadow-sm;
    }
  }

  &__icon {
    width: 40px;
    height: 40px;
    flex-shrink: 0;
    border-radius: 11px;
    background: $primary-light;
    color: $primary;
    display: grid;
    place-items: center;
  }

  &__content {
    display: flex;
    flex-direction: column;
    gap: 5px;
    min-width: 0;
    flex: 1.3;
  }

  &__name {
    font-size: 0.90625rem;
    font-weight: 600;
    color: $text-primary;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  &__meta {
    display: flex;
    align-items: center;
    gap: 8px;
  }

  &__type-badge {
    font-size: 0.6875rem;
    font-weight: 600;
    color: $text-secondary;
    background: $bg-subtle;
    border: 1px solid $border-subtle;
    border-radius: 5px;
    padding: 2px 7px;
  }

  &__date {
    font-size: 0.75rem;
    color: $text-tertiary;
  }

  &__results {
    display: flex;
    align-items: center;
    gap: 26px;
    flex: 1;
    justify-content: flex-end;
  }

  &__stat {
    display: flex;
    flex-direction: column;
    align-items: flex-end;
    gap: 3px;
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
    max-width: 11rem;
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
    gap: 6px;
  }

  &__icon-btn {
    width: 32px;
    height: 32px;
    border-radius: 8px;
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
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 10px;
    padding: 52px 20px;
    border: 1px dashed $border-strong;
    border-radius: 14px;
    color: $text-tertiary;
    font-size: 0.875rem;

    svg {
      color: $text-tertiary;
    }
  }

  /* ---------- responsive ---------- */
  @media (max-width: 860px) {
    &__item {
      flex-wrap: wrap;
    }

    &__results {
      justify-content: flex-start;
      width: 100%;
      padding-left: 56px;
    }

    &__actions {
      margin-left: auto;
    }
  }
}
