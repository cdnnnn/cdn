import type { FC } from 'react';
import { useNavigate } from 'react-router-dom';
import { Play, PlugZap, Upload, Search, Trophy, TrendingUp } from 'lucide-react';
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
        <div className="dash__hero-glow dash__hero-glow--a" aria-hidden="true" />
        <div className="dash__hero-glow dash__hero-glow--b" aria-hidden="true" />
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
            <button type="button" className="dash__link-btn" onClick={() => navigate('/app/history')}>
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
                    <span className="dash__eval-stat-value">{ev.topModel}</span>
                    <span className="dash__eval-stat-label">Top</span>
                  </span>
                  <span className="dash__eval-stat">
                    <span className="dash__eval-stat-value dash__eval-stat-value--highlight n">{ev.topScore}</span>
                    <span className="dash__eval-stat-label">Score</span>
                  </span>
                </div>
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
                <span className="dash__qa-text">
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






























@import '../../../styles/variables';

.dash {
  display: flex;
  flex-direction: column;
  gap: 22px;

  /* ---------- hero ---------- */
  &__hero {
    position: relative;
    overflow: hidden;
    background: linear-gradient(135deg, $primary 0%, #0f1d78 100%);
    border-radius: $radius-xl;
    padding: 36px 40px;
    color: #fff;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 24px;
  }

  &__hero-glow {
    position: absolute;
    border-radius: 50%;
    pointer-events: none;

    &--a {
      top: -60px;
      right: -60px;
      width: 280px;
      height: 280px;
      background: radial-gradient(circle, rgba(255, 255, 255, 0.1) 0%, transparent 70%);
    }

    &--b {
      bottom: -80px;
      right: 120px;
      width: 220px;
      height: 220px;
      background: radial-gradient(circle, rgba(255, 255, 255, 0.06) 0%, transparent 70%);
    }
  }

  &__hero-content {
    position: relative;
    z-index: 1;
    max-width: 600px;
  }

  &__hero-eyebrow {
    margin: 0 0 8px;
    font-size: 12px;
    font-weight: 700;
    color: rgba(255, 255, 255, 0.7);
    text-transform: uppercase;
    letter-spacing: 0.08em;
  }

  &__hero-title {
    margin: 0 0 10px;
    font-size: 26px;
    font-weight: 700;
    letter-spacing: -0.01em;
  }

  &__hero-sub {
    margin: 0 0 18px;
    font-size: 13.5px;
    color: rgba(255, 255, 255, 0.75);
    line-height: 1.5;
  }

  &__hero-highlight {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    background: rgba(255, 255, 255, 0.12);
    border: 1px solid rgba(255, 255, 255, 0.18);
    border-radius: 999px;
    padding: 8px 16px;
    font-size: 12.5px;

    strong {
      font-weight: 700;
    }
  }

  &__hero-highlight-score {
    background: #fff;
    color: $primary;
    font-weight: 700;
    border-radius: 999px;
    padding: 2px 10px;
    margin-left: 2px;
  }

  &__hero-btn {
    position: relative;
    z-index: 1;
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 8px;
    background: #fff;
    color: $primary;
    border: none;
    padding: 13px 22px;
    border-radius: $radius-md;
    font-size: 14px;
    font-weight: 700;
    cursor: pointer;
    box-shadow: 0 8px 20px rgba(0, 0, 0, 0.15);

    &:hover {
      background: $bg-subtle;
    }
  }

  /* ---------- stats ---------- */
  &__stats {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 14px;
  }

  &__stat-card {
    position: relative;
    overflow: hidden;
    background: $bg-main;
    border: 1px solid $border-default;
    border-radius: $radius-lg;
    padding: 20px 22px;
    display: flex;
    flex-direction: column;
    gap: 4px;

    &::before {
      content: '';
      position: absolute;
      left: 0;
      top: 0;
      bottom: 0;
      width: 3px;
      background: $primary;
    }
  }

  &__stat-value {
    font-size: 26px;
    font-weight: 700;
    letter-spacing: -0.01em;
    color: $text-primary;
  }

  &__stat-label {
    font-size: 12px;
    color: $text-tertiary;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.03em;
  }

  /* ---------- row: recent + quick actions ---------- */
  &__row {
    display: grid;
    grid-template-columns: 1.4fr 1fr;
    gap: 16px;
  }

  &__card {
    background: $bg-main;
    border: 1px solid $border-default;
    border-radius: $radius-lg;
    padding: 22px 24px;
  }

  &__card-head {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 16px;
  }

  &__card-title {
    margin: 0;
    font-size: 15px;
    font-weight: 700;
    color: $text-primary;
  }

  &__card-subtitle {
    margin: 2px 0 0;
    font-size: 12px;
    color: $text-tertiary;
  }

  &__link-btn {
    border: none;
    background: none;
    color: $primary;
    font-size: 12.5px;
    font-weight: 700;
    cursor: pointer;
    padding: 0;

    &:hover {
      color: $primary-hover;
    }
  }

  /* ---------- recent evaluations ---------- */
  &__eval-list {
    display: flex;
    flex-direction: column;
  }

  &__eval-item {
    width: 100%;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 14px;
    padding: 13px 4px;
    border: none;
    border-top: 1px solid $border-subtle;
    background: none;
    cursor: pointer;
    text-align: left;

    &:first-child {
      border-top: none;
    }

    &:hover {
      background: $bg-subtle;
    }
  }

  &__eval-info {
    min-width: 0;
  }

  &__eval-name {
    font-size: 13.5px;
    font-weight: 600;
    color: $text-primary;
  }

  &__eval-meta {
    display: flex;
    gap: 8px;
    align-items: center;
    margin-top: 3px;
  }

  &__eval-type-badge {
    font-size: 10.5px;
    font-weight: 700;
    color: $primary;
    background: $primary-light;
    padding: 2px 8px;
    border-radius: 999px;
  }

  &__eval-date {
    font-size: 11px;
    color: $text-tertiary;
  }

  &__eval-stats {
    display: flex;
    gap: 20px;
    align-items: center;
    flex-shrink: 0;
  }

  &__eval-stat {
    display: flex;
    flex-direction: column;
    align-items: flex-end;
    gap: 1px;
  }

  &__eval-stat-value {
    font-size: 13.5px;
    font-weight: 700;
    color: $text-primary;

    &--highlight {
      color: $primary;
    }
  }

  &__eval-stat-label {
    font-size: 9.5px;
    color: $text-tertiary;
    text-transform: uppercase;
    letter-spacing: 0.02em;
  }

  /* ---------- quick actions ---------- */
  &__qa-grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 10px;
  }

  &__qa-card {
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 16px;
    border: 1px solid $border-subtle;
    border-radius: $radius-md;
    cursor: pointer;
    background: $bg-subtle;
    text-align: left;
    transition: border-color 0.15s, background 0.15s;

    &:hover {
      border-color: $primary;
      background: $bg-main;
    }
  }

  &__qa-icon {
    width: 34px;
    height: 34px;
    border-radius: $radius-sm;
    display: flex;
    align-items: center;
    justify-content: center;

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
      background: #efeafc;
      color: #6d4fd1;
    }
  }

  &__qa-text {
    display: flex;
    flex-direction: column;
    gap: 2px;
  }

  &__qa-title {
    font-size: 13px;
    font-weight: 700;
    color: $text-primary;
  }

  &__qa-desc {
    font-size: 11px;
    color: $text-tertiary;
  }

  /* ---------- latest results ---------- */
  &__badge {
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

  &__table-wrap {
    overflow-x: auto;
    margin-top: 6px;
  }

  &__table {
    width: 100%;
    border-collapse: collapse;
    font-size: 13px;

    thead th {
      text-align: left;
      padding: 10px 14px;
      font-size: 11px;
      font-weight: 700;
      color: $text-tertiary;
      text-transform: uppercase;
      letter-spacing: 0.03em;
      border-bottom: 2px solid $border-default;
    }

    tbody td {
      padding: 13px 14px;
      border-bottom: 1px solid $border-subtle;
      color: $text-secondary;
    }

    tbody tr:last-child td {
      border-bottom: none;
    }
  }

  &__row-winner {
    background: linear-gradient(90deg, $primary-light, transparent 70%);
  }

  &__rank {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 26px;
    height: 26px;
    border-radius: 50%;
    background: $bg-inset;
    color: $text-secondary;
    font-weight: 700;
    font-size: 12px;

    &--1 {
      background: linear-gradient(135deg, #f5c542, #e0a91a);
      color: #fff;
    }

    &--2 {
      background: linear-gradient(135deg, #c6ccd6, #9aa3b3);
      color: #fff;
    }

    &--3 {
      background: linear-gradient(135deg, #d79a6a, #b9713b);
      color: #fff;
    }
  }

  &__cell-strong {
    font-weight: 700;
    color: $text-primary;
  }

  &__score {
    font-weight: 700;
    color: $primary;
    font-size: 14px;
  }
}
