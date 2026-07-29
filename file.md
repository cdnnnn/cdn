//Dashboard.tsx
import type { FC } from 'react';
import { useNavigate } from 'react-router-dom';
import { Play, PlugZap, Upload, Search, ChevronRight, Trophy } from 'lucide-react';
import { MODELS, PROVIDERS, TEST_SUITES } from '../RunEvaluation/data';
import { RECENT_EVALUATIONS } from './data';
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
          <h1 className="dash__hero-title">Welcome back, Alex</h1>
          <p className="dash__hero-sub">Compare AI models, run standardized tests, and make data-driven decisions.</p>
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
            {RECENT_EVALUATIONS.map((ev) => (
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
          <h2 className="dash__card-title">Quick Actions</h2>
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


















//Dashboard.scss
@use '../../../styles/variables' as *;

.dash {
  display: flex;
  flex-direction: column;
  gap: 20px;

  /* ---------- hero ---------- */
  &__hero {
    position: relative;
    overflow: hidden;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 1.5rem;
    padding: 28px 30px;
    border-radius: 18px;
    border: 1px solid $border-subtle;
    background: $bg-main;
    box-shadow: $shadow-xs;
  }

  &__hero-grid {
    position: absolute;
    inset: 0;
    background-image: linear-gradient(to right, $border-subtle 1px, transparent 1px),
      linear-gradient(to bottom, $border-subtle 1px, transparent 1px);
    background-size: 40px 40px;
    mask-image: radial-gradient(80% 130% at 100% 0%, #000 5%, transparent 68%);
    -webkit-mask-image: radial-gradient(80% 130% at 100% 0%, #000 5%, transparent 68%);
    pointer-events: none;
  }

  &__hero-content {
    position: relative;
  }

  &__hero-title {
    font-size: 24px;
    font-weight: 800;
    letter-spacing: -0.02em;
    color: $text-primary;
  }

  &__hero-sub {
    margin-top: 6px;
    font-size: 0.875rem;
    color: $text-secondary;
    max-width: 34rem;
  }

  &__hero-btn {
    position: relative;
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 8px;
    padding: 11px 18px;
    border: 1px solid $primary;
    border-radius: 10px;
    background: $primary;
    color: #fff;
    font-size: 0.84375rem;
    font-weight: 600;
    font-family: $font-body;
    cursor: pointer;
    transition: background 0.14s ease, border-color 0.14s ease;

    &:hover {
      background: $primary-hover;
      border-color: $primary-hover;
    }
  }

  /* ---------- stats ---------- */
  &__stats {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 14px;
  }

  &__stat-card {
    display: flex;
    flex-direction: column;
    gap: 6px;
    padding: 18px 20px;
    border: 1px solid $border-subtle;
    border-radius: 14px;
    background: $bg-main;
    box-shadow: $shadow-xs;
  }

  &__stat-value {
    font-size: 26px;
    font-weight: 800;
    letter-spacing: -0.02em;
    color: $text-primary;
  }

  &__stat-label {
    font-family: $font-mono;
    font-size: 0.65625rem;
    font-weight: 600;
    letter-spacing: 0.09em;
    text-transform: uppercase;
    color: $text-tertiary;
  }

  /* ---------- two-column row ---------- */
  &__row {
    display: grid;
    grid-template-columns: 1.35fr 1fr;
    gap: 14px;
    align-items: start;
  }

  /* ---------- shared card ---------- */
  &__card {
    border: 1px solid $border-subtle;
    border-radius: 16px;
    background: $bg-main;
    box-shadow: $shadow-xs;
    padding: 20px 22px 22px;
  }

  &__card-head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
    margin-bottom: 14px;
  }

  &__card-title {
    font-size: 1rem;
    font-weight: 700;
    letter-spacing: -0.01em;
    color: $text-primary;
  }

  &__card-subtitle {
    margin-top: 3px;
    font-size: 0.78125rem;
    color: $text-tertiary;
  }

  &__btn {
    font-family: $font-body;
    border-radius: 8px;
    cursor: pointer;
    font-weight: 600;
    transition: border-color 0.14s ease, color 0.14s ease;

    &--sm {
      padding: 6px 12px;
      font-size: 0.78125rem;
      background: $bg-main;
      color: $text-secondary;
      border: 1px solid $border-default;

      &:hover {
        border-color: $text-primary;
        color: $text-primary;
      }
    }
  }

  /* ---------- recent evaluations ---------- */
  &__eval-list {
    display: flex;
    flex-direction: column;
    gap: 6px;
  }

  &__eval-item {
    display: flex;
    align-items: center;
    gap: 14px;
    width: 100%;
    text-align: left;
    padding: 12px 12px;
    border: 1px solid transparent;
    border-radius: 12px;
    background: transparent;
    cursor: pointer;
    transition: background 0.14s ease, border-color 0.14s ease;

    &:hover {
      background: $bg-subtle;
      border-color: $border-subtle;

      .dash__eval-arrow {
        transform: translateX(2px);
        color: $primary;
      }
    }
  }

  &__eval-info {
    display: flex;
    flex-direction: column;
    gap: 5px;
    min-width: 0;
    flex: 1;
  }

  &__eval-name {
    font-size: 0.84375rem;
    font-weight: 600;
    color: $text-primary;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  &__eval-meta {
    display: flex;
    align-items: center;
    gap: 8px;
  }

  &__eval-type-badge {
    font-size: 0.65625rem;
    font-weight: 600;
    color: $primary;
    background: $primary-light;
    border-radius: 5px;
    padding: 2px 7px;
  }

  &__eval-date {
    font-size: 0.71875rem;
    color: $text-tertiary;
  }

  &__eval-stats {
    display: flex;
    align-items: center;
    gap: 20px;
    flex-shrink: 0;
  }

  &__eval-stat {
    display: flex;
    flex-direction: column;
    align-items: flex-end;
    gap: 2px;
  }

  &__eval-stat-label {
    font-size: 0.625rem;
    font-weight: 600;
    letter-spacing: 0.05em;
    text-transform: uppercase;
    color: $text-tertiary;
  }

  &__eval-stat-value {
    font-size: 0.78125rem;
    font-weight: 600;
    color: $text-secondary;
    white-space: nowrap;

    &--highlight {
      color: $success;
      font-weight: 700;
    }
  }

  &__eval-arrow {
    flex-shrink: 0;
    color: $text-tertiary;
    transition: transform 0.14s ease, color 0.14s ease;
  }

  /* ---------- quick actions ---------- */
  &__qa-grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 10px;
  }

  &__qa-card {
    display: flex;
    align-items: center;
    gap: 12px;
    text-align: left;
    padding: 14px;
    border: 1px solid $border-subtle;
    border-radius: 12px;
    background: $bg-main;
    cursor: pointer;
    transition: border-color 0.14s ease, background 0.14s ease;

    &:hover {
      border-color: $border-strong;
      background: $bg-subtle;
    }
  }

  &__qa-icon {
    width: 36px;
    height: 36px;
    flex-shrink: 0;
    border-radius: 10px;
    display: grid;
    place-items: center;

    &--blue {
      background: $primary-light;
      color: $primary;
    }

    &--green {
      background: $success-subtle;
      color: $success;
    }

    &--amber {
      background: $warning-subtle;
      color: $warning;
    }

    &--violet {
      background: #f3e8ff;
      color: #7c3aed;
    }
  }

  &__qa-title {
    display: block;
    font-size: 0.8125rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__qa-desc {
    display: block;
    margin-top: 2px;
    font-size: 0.71875rem;
    color: $text-tertiary;
  }

  /* ---------- results table ---------- */
  &__badge {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 5px;
    font-size: 0.6875rem;
    font-weight: 600;
    color: $success;
    background: $success-subtle;
    border-radius: 999px;
    padding: 5px 10px;
    white-space: nowrap;
  }

  &__table-wrap {
    overflow-x: auto;
  }

  &__table {
    width: 100%;
    border-collapse: separate;
    border-spacing: 0;

    th {
      font-family: $font-mono;
      font-size: 0.625rem;
      font-weight: 600;
      letter-spacing: 0.07em;
      text-transform: uppercase;
      color: $text-tertiary;
      background: $bg-subtle;
      padding: 9px 12px;
      border-bottom: 1px solid $border-default;
      text-align: left;
      white-space: nowrap;
    }

    td {
      padding: 11px 12px;
      border-bottom: 1px solid $border-subtle;
      font-size: 0.8125rem;
      color: $text-secondary;
      white-space: nowrap;
    }

    tbody tr:last-child td {
      border-bottom: 0;
    }

    tbody tr:hover td {
      background: $bg-subtle;
    }
  }

  &__cell-strong {
    color: $text-primary;
    font-weight: 600;
  }

  &__row-winner td {
    background: $success-subtle;
  }

  &__rank {
    display: inline-grid;
    place-items: center;
    width: 24px;
    height: 24px;
    border-radius: 7px;
    font-size: 0.71875rem;
    font-weight: 700;
    background: $bg-inset;
    color: $text-tertiary;

    &--1 {
      background: $success;
      color: #fff;
    }

    &--2 {
      background: $border-default;
      color: $text-secondary;
    }

    &--3 {
      background: $warning-subtle;
      color: $warning;
    }
  }

  &__score {
    font-weight: 700;
    color: $primary;
  }

  /* ---------- responsive ---------- */
  @media (max-width: 900px) {
    &__stats {
      grid-template-columns: repeat(2, 1fr);
    }

    &__row {
      grid-template-columns: 1fr;
    }

    &__hero {
      flex-direction: column;
      align-items: flex-start;
    }
  }
}














//data.ts
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
    modelsTested: 2,
    topModel: 'Model Theta Long-Context',
    topScore: '95.3%',
    status: 'Completed',
    results: [
      { rank: 1, model: 'Model Theta Long-Context', provider: 'Google Gemini', score: '95.3%', accuracy: '96.5%', time: '1.20s', cost: '$0.31' },
      { rank: 2, model: 'Model Zeta Instruct', provider: 'Groq Cloud', score: '93.1%', accuracy: '91.2%', time: '0.28s', cost: '$0.07' },
    ],
  },
];
