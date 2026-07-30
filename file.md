import { Link } from 'react-router-dom';
import type { FC } from 'react';
import {
  ArrowRight,
  BarChart3,
  FileText,
  Gauge,
  Database,
  PlugZap,
  FileBarChart,
  Check,
  Sparkles,
} from 'lucide-react';
import './Landing.scss';

const STATS = [
  { value: '9', label: 'Providers' },
  { value: '100+', label: 'Models' },
  { value: '14', label: 'Test suites' },
  { value: '3', label: 'Evaluation types' },
];

const FEATURES = [
  {
    icon: BarChart3,
    title: 'Side-by-side scoring',
    desc: 'Every model runs the same prompts under the same conditions, so the leaderboard reflects the models — not the setup.',
  },
  {
    icon: FileText,
    title: 'Full response transcripts',
    desc: 'Open any score to see the exact prompt, response, and grading reason. Nothing is a black box.',
  },
  {
    icon: Gauge,
    title: 'Cost & latency at scale',
    desc: 'Enter your expected monthly volume and see what each model would actually cost you — not per-token pricing.',
  },
  {
    icon: Database,
    title: 'Standard or custom suites',
    desc: 'Start from a benchmark suite or upload your own prompts and expected answers in CSV, JSON, or JSONL.',
  },
  {
    icon: PlugZap,
    title: 'Every major provider',
    desc: 'Connect OpenAI, Anthropic, Google, Groq, Together, and more from one workspace — no per-model setup.',
  },
  {
    icon: FileBarChart,
    title: 'Reports that get read',
    desc: 'Turn a run into a shareable report with the recommendation, trade-offs, and raw data attached.',
  },
];

const STEPS = [
  { n: '01', title: 'Connect a provider', desc: 'Paste an API key and your model catalogue fills in automatically.' },
  { n: '02', title: 'Pick your models', desc: 'Filter by price, speed, context window, or benchmark score.' },
  { n: '03', title: 'Choose a test suite', desc: 'Use a standard benchmark or bring your own test questions.' },
  { n: '04', title: 'Read the results', desc: 'Scores, cost, and latency land in one leaderboard — instantly.' },
];

const COMPARISON = [
  { model: 'Model Alpha', accuracy: 79.5, cost: '$2.40', time: '0.4s', best: false },
  { model: 'Model Beta', accuracy: 84.1, cost: '$6.80', time: '2.9s', best: false },
  { model: 'Model Gamma', accuracy: 88.7, cost: '$24.50', time: '1.2s', best: false },
  { model: 'Model Delta', accuracy: 91.2, cost: '$41.20', time: '2.1s', best: true },
];

const Landing: FC = () => {
  return (
    <div className="land">
      {/* ==================== HERO ==================== */}
      <section className="land__hero">
        <div className="land__hero-glow" aria-hidden="true" />
        <div className="land__shell land__hero-grid">
          <div className="land__hero-copy">
            <span className="land__pill">
              <Sparkles size={13} strokeWidth={2.25} />
              Model Evaluation Platform
            </span>

            <h1 className="land__h1">
              Pick your next AI model with <span className="land__accent">evidence</span>, not guesswork
            </h1>

            <p className="land__lede">
              SemcoEval runs identical test suites across every model you're considering, then puts
              accuracy, latency, and cost side by side — so the model you ship is the one the data
              actually picked.
            </p>

            <div className="land__cta-row">
              <Link className="land__btn land__btn--primary" to="/app">
                Start an evaluation <ArrowRight size={16} strokeWidth={2.25} />
              </Link>
              <a className="land__btn land__btn--ghost" href="#how">
                See how it works
              </a>
            </div>

            <div className="land__stats">
              {STATS.map((s) => (
                <div className="land__stat" key={s.label}>
                  <span className="land__stat-value n">{s.value}</span>
                  <span className="land__stat-label">{s.label}</span>
                </div>
              ))}
            </div>
          </div>

          <div className="land__hero-visual">
            <div className="land__panel">
              <div className="land__panel-head">
                <span className="land__panel-dot" />
                Enterprise QA suite &middot; 240 prompts
              </div>
              <div className="land__bars">
                {COMPARISON.map((c) => (
                  <div className="land__bar-row" key={c.model}>
                    <span className="land__bar-label">{c.model}</span>
                    <div className="land__bar-track">
                      <div
                        className={`land__bar-fill${c.best ? ' land__bar-fill--best' : ''}`}
                        style={{ width: `${c.accuracy}%` }}
                      />
                    </div>
                    <span className="land__bar-value n">{c.accuracy}%</span>
                  </div>
                ))}
              </div>
              <div className="land__panel-foot">
                <span>
                  <b>Winner</b> Model Delta
                </span>
                <span className="n">$41.20 / 1K runs</span>
                <span className="n">2.1s avg</span>
              </div>
            </div>
          </div>
        </div>
      </section>

      {/* ==================== FEATURES ==================== */}
      <section className="land__section">
        <div className="land__shell">
          <div className="land__section-head">
            <p className="land__kicker">What you get</p>
            <h2 className="land__h2">Everything you need to compare models with confidence</h2>
          </div>

          <div className="land__feature-grid">
            {FEATURES.map((f) => (
              <div className="land__feature" key={f.title}>
                <span className="land__feature-icon">
                  <f.icon size={19} strokeWidth={2} />
                </span>
                <h3 className="land__feature-title">{f.title}</h3>
                <p className="land__feature-desc">{f.desc}</p>
              </div>
            ))}
          </div>
        </div>
      </section>

      {/* ==================== HOW IT WORKS ==================== */}
      <section className="land__section land__section--tint" id="how">
        <div className="land__shell">
          <div className="land__section-head">
            <p className="land__kicker">How it works</p>
            <h2 className="land__h2">Four steps, and the answer is a table — not an opinion</h2>
          </div>

          <div className="land__steps">
            <div className="land__steps-line" aria-hidden="true" />
            {STEPS.map((s) => (
              <div className="land__step" key={s.n}>
                <span className="land__step-num">{s.n}</span>
                <h3 className="land__step-title">{s.title}</h3>
                <p className="land__step-desc">{s.desc}</p>
              </div>
            ))}
          </div>
        </div>
      </section>

      {/* ==================== TRUST / GROUNDING ==================== */}
      <section className="land__section">
        <div className="land__shell land__grounded">
          <div className="land__grounded-copy">
            <p className="land__kicker">Every score, checked</p>
            <h2 className="land__h2">A score you can't audit is just a rumour</h2>
            <p className="land__lede land__lede--tight">
              Open any cell in the leaderboard and see the exact prompt, the model's full response,
              and the reason it was marked right or wrong — graded against a source you control.
            </p>
            <ul className="land__check-list">
              <li>
                <Check size={15} strokeWidth={2.5} /> Full prompt and response kept for every run
              </li>
              <li>
                <Check size={15} strokeWidth={2.5} /> Grading reason attached to every score
              </li>
              <li>
                <Check size={15} strokeWidth={2.5} /> Re-run against a pinned baseline anytime
              </li>
            </ul>
          </div>

          <div className="land__ask">
            <p className="land__ask-kicker">Prompt 112 &middot; Billing policy</p>
            <p className="land__ask-q">
              A customer downgrades their plan mid-cycle. What happens to their unused credits?
            </p>
            <div className="land__ask-a land__ask-a--pass">
              <span className="land__ask-tag land__ask-tag--pass">Pass</span>
              Unused credits carry over as account balance; no refund is issued.
            </div>
            <div className="land__ask-a land__ask-a--fail">
              <span className="land__ask-tag land__ask-tag--fail">Fail</span>
              Customer receives a prorated refund within 5–7 business days.
            </div>
          </div>
        </div>
      </section>

      {/* ==================== FINAL CTA ==================== */}
      <section className="land__section">
        <div className="land__shell">
          <div className="land__cta-banner">
            <div className="land__cta-glow" aria-hidden="true" />
            <p className="land__kicker land__kicker--light">Get started</p>
            <h2 className="land__cta-title">Your next model decision, with the receipts</h2>
            <p className="land__cta-sub">
              Connect one provider, pick a standard suite, and have a defensible answer before your
              next architecture review.
            </p>
            <Link className="land__btn land__btn--white" to="/app">
              Start an evaluation <ArrowRight size={16} strokeWidth={2.25} />
            </Link>
          </div>
        </div>
      </section>
    </div>
  );
};

export default Landing;





















@use '../../styles/variables' as *;

.land {
  background: #fff;
  color: $text-primary;

  &__shell {
    max-width: 1180px;
    margin: 0 auto;
    padding: 0 clamp(20px, 5vw, 60px);
  }

  &__kicker {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    font-family: $font-mono;
    font-size: 0.6875rem;
    font-weight: 700;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    color: $primary;
    margin-bottom: 10px;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $primary;
    }

    &--light {
      color: rgba(255, 255, 255, 0.85);

      &::before {
        background: rgba(255, 255, 255, 0.85);
      }
    }
  }

  &__h1 {
    font-size: clamp(2.25rem, 4.4vw, 3.4rem);
    font-weight: 800;
    letter-spacing: -0.03em;
    line-height: 1.08;
    margin-top: 16px;
  }

  &__accent {
    color: $primary;
  }

  &__h2 {
    font-size: clamp(1.625rem, 2.7vw, 2.25rem);
    font-weight: 800;
    letter-spacing: -0.025em;
    line-height: 1.15;
    max-width: 34rem;
  }

  &__lede {
    margin-top: 18px;
    font-size: 1.0625rem;
    line-height: 1.65;
    color: $text-secondary;
    max-width: 34rem;

    &--tight {
      margin-top: 14px;
      max-width: 30rem;
    }
  }

  /* ---------- buttons ---------- */
  &__btn {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    font-family: $font-body;
    font-size: 0.9375rem;
    font-weight: 600;
    padding: 13px 22px;
    border-radius: 11px;
    border: 1px solid transparent;
    cursor: pointer;
    text-decoration: none;
    transition: background 0.14s ease, border-color 0.14s ease, transform 0.14s ease;

    &--primary {
      background: $primary;
      color: #fff;
      border-color: $primary;
      box-shadow: 0 10px 24px -10px rgba(20, 40, 160, 0.45);

      &:hover {
        background: $primary-hover;
        border-color: $primary-hover;
        transform: translateY(-1px);
      }
    }

    &--ghost {
      background: #fff;
      color: $text-primary;
      border-color: $border-default;

      &:hover {
        border-color: $text-primary;
      }
    }

    &--white {
      background: #fff;
      color: $primary;
      border-color: #fff;

      &:hover {
        transform: translateY(-1px);
      }
    }
  }

  /* ==================== HERO ==================== */
  &__hero {
    position: relative;
    overflow: hidden;
    padding: clamp(56px, 8vw, 96px) 0 clamp(64px, 7vw, 92px);
  }

  &__hero-glow {
    position: absolute;
    top: -220px;
    left: 50%;
    transform: translateX(-50%);
    width: 900px;
    height: 560px;
    border-radius: 50%;
    background: radial-gradient(circle, rgba(20, 40, 160, 0.1) 0%, rgba(20, 40, 160, 0) 70%);
    pointer-events: none;
  }

  &__hero-grid {
    position: relative;
    display: grid;
    grid-template-columns: 1.05fr 0.95fr;
    gap: 48px;
    align-items: center;
  }

  &__pill {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    font-size: 0.78125rem;
    font-weight: 600;
    color: $primary;
    background: $primary-light;
    border: 1px solid $primary-subtle;
    border-radius: 999px;
    padding: 7px 14px;
  }

  &__cta-row {
    display: flex;
    flex-wrap: wrap;
    gap: 12px;
    margin-top: 30px;
  }

  &__stats {
    display: flex;
    flex-wrap: wrap;
    gap: 30px;
    margin-top: 44px;
    padding-top: 26px;
    border-top: 1px solid $border-subtle;
  }

  &__stat {
    display: flex;
    flex-direction: column;
    gap: 2px;
  }

  &__stat-value {
    font-size: 1.375rem;
    font-weight: 800;
    letter-spacing: -0.02em;
    color: $text-primary;
  }

  &__stat-label {
    font-size: 0.75rem;
    color: $text-tertiary;
  }

  /* ---------- hero visual panel ---------- */
  &__hero-visual {
    position: relative;
  }

  &__panel {
    position: relative;
    background: #fff;
    border: 1px solid $border-subtle;
    border-radius: 20px;
    box-shadow: 0 24px 60px -30px rgba(14, 21, 38, 0.28);
    padding: 22px 24px 20px;
    overflow: hidden;

    &::before {
      content: '';
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      height: 3px;
      background: linear-gradient(90deg, $primary, $primary-hover 60%, $success);
    }
  }

  &__panel-head {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 0.78125rem;
    font-weight: 600;
    color: $text-secondary;
    padding-bottom: 16px;
    border-bottom: 1px solid $border-subtle;
    margin-bottom: 18px;
  }

  &__panel-dot {
    width: 7px;
    height: 7px;
    border-radius: 50%;
    background: $success;
    box-shadow: 0 0 0 3px $success-subtle;
    flex-shrink: 0;
  }

  &__bars {
    display: flex;
    flex-direction: column;
    gap: 14px;
  }

  &__bar-row {
    display: grid;
    grid-template-columns: 84px 1fr 46px;
    align-items: center;
    gap: 12px;
  }

  &__bar-label {
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-secondary;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  &__bar-track {
    position: relative;
    height: 8px;
    background: $bg-inset;
    border-radius: 4px;
    overflow: hidden;
  }

  &__bar-fill {
    height: 100%;
    border-radius: 4px;
    background: $text-tertiary;

    &--best {
      background: linear-gradient(90deg, $primary, $primary-hover);
    }
  }

  &__bar-value {
    font-size: 0.75rem;
    font-weight: 700;
    color: $text-primary;
    text-align: right;
  }

  &__panel-foot {
    display: flex;
    flex-wrap: wrap;
    gap: 8px 18px;
    margin-top: 18px;
    padding-top: 16px;
    border-top: 1px solid $border-subtle;
    font-size: 0.75rem;
    color: $text-secondary;

    b {
      color: $success;
      font-weight: 700;
      margin-right: 4px;
    }
  }

  /* ==================== SECTIONS ==================== */
  &__section {
    padding: clamp(56px, 7vw, 88px) 0;

    &--tint {
      background: $bg-subtle;
      border-block: 1px solid $border-subtle;
    }
  }

  &__section-head {
    margin-bottom: clamp(32px, 4vw, 48px);
  }

  /* ---------- feature grid ---------- */
  &__feature-grid {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 20px;
  }

  &__feature {
    padding: 26px 24px;
    border: 1px solid $border-subtle;
    border-radius: 16px;
    background: #fff;
    transition: border-color 0.14s ease, transform 0.14s ease, box-shadow 0.14s ease;

    &:hover {
      border-color: $border-strong;
      transform: translateY(-2px);
      box-shadow: $shadow-md;
    }
  }

  &__feature-icon {
    width: 42px;
    height: 42px;
    display: grid;
    place-items: center;
    border-radius: 12px;
    background: $primary-light;
    color: $primary;
    margin-bottom: 16px;
  }

  &__feature-title {
    font-size: 1.03125rem;
    font-weight: 700;
    letter-spacing: -0.01em;
  }

  &__feature-desc {
    margin-top: 8px;
    font-size: 0.875rem;
    line-height: 1.55;
    color: $text-secondary;
  }

  /* ---------- steps ---------- */
  &__steps {
    position: relative;
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 24px;
  }

  &__steps-line {
    position: absolute;
    top: 19px;
    left: calc(12.5% - 20px);
    right: calc(12.5% - 20px);
    height: 2px;
    background: $border-default;
    z-index: 0;
  }

  &__step {
    position: relative;
    z-index: 1;
  }

  &__step-num {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 40px;
    height: 40px;
    border-radius: 50%;
    background: #fff;
    border: 2px solid $primary;
    color: $primary;
    font-weight: 800;
    font-size: 0.8125rem;
    margin-bottom: 18px;
  }

  &__step-title {
    font-size: 1.0625rem;
    font-weight: 700;
    letter-spacing: -0.01em;
  }

  &__step-desc {
    margin-top: 8px;
    font-size: 0.875rem;
    line-height: 1.55;
    color: $text-secondary;
  }

  /* ---------- grounded / trust ---------- */
  &__grounded {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 56px;
    align-items: center;
  }

  &__check-list {
    list-style: none;
    margin-top: 22px;
    display: flex;
    flex-direction: column;
    gap: 12px;

    li {
      display: flex;
      align-items: center;
      gap: 10px;
      font-size: 0.90625rem;
      color: $text-secondary;

      svg {
        flex-shrink: 0;
        color: $success;
      }
    }
  }

  &__ask {
    background: #fff;
    border: 1px solid $border-subtle;
    border-radius: 18px;
    box-shadow: $shadow-md;
    padding: 24px 26px;
  }

  &__ask-kicker {
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-tertiary;
  }

  &__ask-q {
    margin-top: 10px;
    font-size: 1.09375rem;
    font-weight: 600;
    line-height: 1.45;
    color: $text-primary;
  }

  &__ask-a {
    margin-top: 14px;
    display: flex;
    align-items: flex-start;
    gap: 10px;
    padding: 13px 15px;
    border-radius: 12px;
    font-size: 0.84375rem;
    line-height: 1.5;

    &--pass {
      background: $success-subtle;
      color: $text-primary;
    }

    &--fail {
      background: $danger-subtle;
      color: $text-primary;
    }
  }

  &__ask-tag {
    flex-shrink: 0;
    font-size: 0.625rem;
    font-weight: 700;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    border-radius: 6px;
    padding: 3px 8px;

    &--pass {
      background: $success;
      color: #fff;
    }

    &--fail {
      background: $danger;
      color: #fff;
    }
  }

  /* ---------- final CTA ---------- */
  &__cta-banner {
    position: relative;
    overflow: hidden;
    border-radius: 24px;
    background: linear-gradient(135deg, $primary 0%, $primary-hover 60%, #101d6b 100%);
    padding: clamp(44px, 6vw, 64px) clamp(28px, 6vw, 64px);
    text-align: center;
    display: flex;
    flex-direction: column;
    align-items: center;
  }

  &__cta-glow {
    position: absolute;
    top: -140px;
    right: -80px;
    width: 420px;
    height: 420px;
    border-radius: 50%;
    background: radial-gradient(circle, rgba(255, 255, 255, 0.16) 0%, rgba(255, 255, 255, 0) 70%);
    pointer-events: none;
  }

  &__cta-title {
    position: relative;
    font-size: clamp(1.625rem, 3vw, 2.25rem);
    font-weight: 800;
    letter-spacing: -0.025em;
    color: #fff;
    max-width: 32rem;
  }

  &__cta-sub {
    position: relative;
    margin-top: 14px;
    font-size: 1rem;
    line-height: 1.6;
    color: rgba(255, 255, 255, 0.85);
    max-width: 30rem;
  }

  &__cta-banner .land__btn {
    position: relative;
    margin-top: 26px;
  }

  /* ==================== RESPONSIVE ==================== */
  @media (max-width: 960px) {
    &__hero-grid {
      grid-template-columns: 1fr;
    }

    &__feature-grid {
      grid-template-columns: repeat(2, 1fr);
    }

    &__steps {
      grid-template-columns: repeat(2, 1fr);
    }

    &__steps-line {
      display: none;
    }

    &__grounded {
      grid-template-columns: 1fr;
      gap: 32px;
    }
  }

  @media (max-width: 620px) {
    &__feature-grid {
      grid-template-columns: 1fr;
    }

    &__steps {
      grid-template-columns: 1fr;
    }

    &__cta-row {
      flex-direction: column;
      align-items: stretch;
    }
  }
}
