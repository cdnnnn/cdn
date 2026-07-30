import { Link } from 'react-router-dom';
import type { FC } from 'react';
import { ArrowUpRight } from 'lucide-react';
import './Landing.scss';

const LEDGER = [
  { k: 'Setup time', v: '~5 min' },
  { k: 'Providers', v: '9' },
  { k: 'Models', v: '100+' },
  { k: 'Test suites', v: '14' },
];

const INDEX_ITEMS = [
  {
    n: '01',
    title: 'Identical conditions, every model',
    desc: 'Same prompts, same grader, same test suite for every candidate — so the leaderboard measures the models, not the setup.',
  },
  {
    n: '02',
    title: 'Nothing hidden behind a score',
    desc: 'Open any result to read the exact prompt, the full response, and the reason it was marked right or wrong.',
  },
  {
    n: '03',
    title: 'Cost at your actual volume',
    desc: 'Enter your expected monthly request count and see what each model would cost you — not a per-token rate card.',
  },
  {
    n: '04',
    title: 'Bring your own questions',
    desc: 'Start from a standard benchmark, or upload your own prompts and expected answers as CSV, JSON, or JSONL.',
  },
  {
    n: '05',
    title: 'One workspace, every provider',
    desc: 'OpenAI, Anthropic, Google, Groq, Together, and more — connected once, compared together.',
  },
  {
    n: '06',
    title: 'A report someone will read',
    desc: 'Every run can become a shareable summary with the recommendation, the trade-offs, and the raw table attached.',
  },
];

const METHOD = [
  { n: '01', label: 'Connect', desc: 'Paste an API key' },
  { n: '02', label: 'Select', desc: 'Choose your models' },
  { n: '03', label: 'Test', desc: 'Pick a question set' },
  { n: '04', label: 'Read', desc: 'One ranked table' },
];

const Landing: FC = () => {
  return (
    <div className="land">
      {/* ==================== HERO ==================== */}
      <section className="land__hero">
        <div className="land__shell">
          <span className="land__tag">Model evaluation, in-house</span>

          <h1 className="land__h1">
            Choose the model
            <br />
            the evidence supports.
          </h1>

          <p className="land__lede">
            SemcoEval runs the same test suite across every model on your shortlist and lines up
            accuracy, latency, and cost in one table — before you commit to anything in production.
          </p>

          <div className="land__cta-row">
            <Link className="land__btn" to="/app">
              Start an evaluation
            </Link>
            <a className="land__text-link" href="#index">
              What it measures <ArrowUpRight size={14} strokeWidth={2.25} />
            </a>
          </div>
        </div>

        <div className="land__ledger">
          <div className="land__shell land__ledger-row">
            {LEDGER.map((l) => (
              <div className="land__ledger-cell" key={l.k}>
                <span className="land__ledger-v n">{l.v}</span>
                <span className="land__ledger-k">{l.k}</span>
              </div>
            ))}
          </div>
        </div>
      </section>

      {/* ==================== INDEX ==================== */}
      <section className="land__section" id="index">
        <div className="land__shell">
          <p className="land__eyebrow">01 — What it measures</p>

          <div className="land__index">
            {INDEX_ITEMS.map((item) => (
              <div className="land__index-row" key={item.n}>
                <span className="land__index-n n">{item.n}</span>
                <div className="land__index-body">
                  <h3 className="land__index-title">{item.title}</h3>
                  <p className="land__index-desc">{item.desc}</p>
                </div>
                <ArrowUpRight size={16} className="land__index-arrow" strokeWidth={2} />
              </div>
            ))}
          </div>
        </div>
      </section>

      {/* ==================== METHOD ==================== */}
      <section className="land__section land__section--ink">
        <div className="land__shell">
          <p className="land__eyebrow land__eyebrow--light">02 — The method</p>
          <h2 className="land__h2 land__h2--light">Four steps. The output is a table, not an opinion.</h2>

          <div className="land__axis">
            <div className="land__axis-line" aria-hidden="true" />
            {METHOD.map((m) => (
              <div className="land__tick" key={m.n}>
                <span className="land__tick-mark" aria-hidden="true" />
                <span className="land__tick-n n">{m.n}</span>
                <span className="land__tick-label">{m.label}</span>
                <span className="land__tick-desc">{m.desc}</span>
              </div>
            ))}
          </div>
        </div>
      </section>

      {/* ==================== TRANSCRIPT ==================== */}
      <section className="land__section">
        <div className="land__shell land__transcript-wrap">
          <div>
            <p className="land__eyebrow">03 — Graded, not guessed</p>
            <h2 className="land__h2">A score you can't open isn't a score you can trust.</h2>
            <p className="land__lede land__lede--tight">
              Every answer in every run is kept in full, next to the reason it passed or failed — so
              the leaderboard is something you can defend, not just cite.
            </p>
          </div>

          <div className="land__log">
            <div className="land__log-line land__log-line--q">
              <span className="land__log-tag">PROMPT 112</span>
              A customer downgrades their plan mid-cycle. What happens to unused credits?
            </div>
            <div className="land__log-line land__log-line--pass">
              <span className="land__log-tag land__log-tag--pass">[PASS]</span>
              Unused credits carry over as account balance; no refund is issued.
            </div>
            <div className="land__log-line land__log-line--fail">
              <span className="land__log-tag land__log-tag--fail">[FAIL]</span>
              Customer receives a prorated refund within 5–7 business days.
            </div>
          </div>
        </div>
      </section>

      {/* ==================== CLOSE ==================== */}
      <section className="land__close">
        <div className="land__shell land__close-in">
          <p className="land__eyebrow land__eyebrow--light">Get started</p>
          <h2 className="land__close-title">Your next model decision, with the receipts.</h2>
          <Link className="land__btn land__btn--outline-light" to="/app">
            Start an evaluation
          </Link>
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
    max-width: 980px;
    margin: 0 auto;
    padding: 0 clamp(20px, 5vw, 48px);
  }

  &__eyebrow {
    font-family: $font-mono;
    font-size: 0.75rem;
    font-weight: 600;
    letter-spacing: 0.06em;
    color: $text-tertiary;
    padding-bottom: 14px;
    margin-bottom: 30px;
    border-bottom: 1px solid $border-default;

    &--light {
      color: rgba(255, 255, 255, 0.55);
      border-bottom-color: rgba(255, 255, 255, 0.16);
    }
  }

  &__h1 {
    font-size: clamp(2.375rem, 5.4vw, 4rem);
    font-weight: 800;
    letter-spacing: -0.035em;
    line-height: 1.03;
    margin-top: 22px;
    max-width: 22ch;
  }

  &__h2 {
    font-size: clamp(1.5rem, 2.6vw, 2.125rem);
    font-weight: 800;
    letter-spacing: -0.025em;
    line-height: 1.2;
    max-width: 30rem;

    &--light {
      color: #fff;
    }
  }

  &__lede {
    margin-top: 20px;
    font-size: 1.0625rem;
    line-height: 1.65;
    color: $text-secondary;
    max-width: 34rem;

    &--tight {
      margin-top: 14px;
      max-width: 28rem;
    }
  }

  /* ---------- buttons / links ---------- */
  &__btn {
    display: inline-flex;
    align-items: center;
    font-family: $font-body;
    font-size: 0.90625rem;
    font-weight: 700;
    padding: 14px 26px;
    border-radius: 3px;
    background: $text-primary;
    color: #fff;
    text-decoration: none;
    border: 1px solid $text-primary;
    transition: background 0.14s ease, color 0.14s ease;

    &:hover {
      background: $primary;
      border-color: $primary;
    }

    &--outline-light {
      background: transparent;
      color: #fff;
      border-color: rgba(255, 255, 255, 0.4);

      &:hover {
        background: #fff;
        color: $text-primary;
        border-color: #fff;
      }
    }
  }

  &__text-link {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    font-size: 0.90625rem;
    font-weight: 600;
    color: $text-primary;
    text-decoration: none;
    border-bottom: 1px solid $border-strong;
    padding-bottom: 2px;

    &:hover {
      border-color: $text-primary;
    }
  }

  &__cta-row {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 28px;
    margin-top: 34px;
  }

  /* ==================== HERO ==================== */
  &__hero {
    padding-top: clamp(64px, 9vw, 108px);
  }

  &__tag {
    display: inline-block;
    font-family: $font-mono;
    font-size: 0.71875rem;
    font-weight: 600;
    letter-spacing: 0.04em;
    color: $primary;
    padding-bottom: 8px;
    border-bottom: 2px solid $primary;
  }

  &__ledger {
    margin-top: clamp(56px, 7vw, 84px);
    border-top: 1px solid $border-default;
    border-bottom: 1px solid $border-default;
    background: $bg-subtle;
  }

  &__ledger-row {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
  }

  &__ledger-cell {
    display: flex;
    flex-direction: column;
    gap: 4px;
    padding: 22px 0;
    border-right: 1px solid $border-default;

    &:last-child {
      border-right: 0;
    }
  }

  &__ledger-v {
    font-size: 1.375rem;
    font-weight: 800;
    letter-spacing: -0.02em;
    color: $text-primary;
  }

  &__ledger-k {
    font-family: $font-mono;
    font-size: 0.6875rem;
    letter-spacing: 0.05em;
    text-transform: uppercase;
    color: $text-tertiary;
  }

  /* ==================== SECTIONS ==================== */
  &__section {
    padding: clamp(64px, 8vw, 96px) 0;

    &--ink {
      background: $text-primary;
      color: #fff;
    }
  }

  /* ---------- index list ---------- */
  &__index {
    display: flex;
    flex-direction: column;
  }

  &__index-row {
    display: grid;
    grid-template-columns: 56px 1fr 20px;
    align-items: flex-start;
    gap: 20px;
    padding: 26px 0;
    border-bottom: 1px solid $border-default;
    transition: padding-left 0.16s ease;

    &:first-child {
      border-top: 1px solid $border-default;
    }

    &:hover {
      padding-left: 10px;

      .land__index-arrow {
        opacity: 1;
        transform: translate(2px, -2px);
      }
    }
  }

  &__index-n {
    font-size: 0.90625rem;
    font-weight: 700;
    color: $text-tertiary;
    padding-top: 3px;
  }

  &__index-title {
    font-size: 1.15625rem;
    font-weight: 700;
    letter-spacing: -0.015em;
  }

  &__index-desc {
    margin-top: 8px;
    font-size: 0.90625rem;
    line-height: 1.6;
    color: $text-secondary;
    max-width: 38rem;
  }

  &__index-arrow {
    margin-top: 4px;
    color: $text-tertiary;
    opacity: 0;
    transition: opacity 0.16s ease, transform 0.16s ease;
  }

  /* ---------- method axis ---------- */
  &__axis {
    position: relative;
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    margin-top: clamp(40px, 5vw, 60px);
    padding-top: 26px;
  }

  &__axis-line {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    height: 1px;
    background: rgba(255, 255, 255, 0.18);
  }

  &__tick {
    position: relative;
    display: flex;
    flex-direction: column;
    padding-right: 20px;
  }

  &__tick-mark {
    position: absolute;
    top: -26px;
    left: 0;
    width: 1px;
    height: 12px;
    background: $primary-hover;
  }

  &__tick-n {
    font-size: 0.71875rem;
    color: rgba(255, 255, 255, 0.4);
    margin-bottom: 10px;
  }

  &__tick-label {
    font-size: 1.03125rem;
    font-weight: 700;
    color: #fff;
  }

  &__tick-desc {
    margin-top: 8px;
    font-size: 0.8125rem;
    line-height: 1.55;
    color: rgba(255, 255, 255, 0.6);
  }

  /* ---------- transcript ---------- */
  &__transcript-wrap {
    display: grid;
    grid-template-columns: 0.9fr 1.1fr;
    gap: 56px;
    align-items: start;
  }

  &__log {
    font-family: $font-mono;
    border: 1px solid $border-default;
  }

  &__log-line {
    display: flex;
    align-items: baseline;
    gap: 12px;
    padding: 16px 18px;
    font-size: 0.8125rem;
    line-height: 1.6;
    border-bottom: 1px solid $border-default;
    color: $text-secondary;

    &:last-child {
      border-bottom: 0;
    }

    &--q {
      color: $text-primary;
      font-weight: 600;
      background: $bg-subtle;
    }
  }

  &__log-tag {
    flex-shrink: 0;
    font-weight: 700;
    font-size: 0.6875rem;
    letter-spacing: 0.03em;
    color: $text-tertiary;

    &--pass {
      color: $success;
    }

    &--fail {
      color: $danger;
    }
  }

  /* ==================== CLOSE ==================== */
  &__close {
    background: $text-primary;
    padding: clamp(72px, 9vw, 108px) 0;
  }

  &__close-in {
    max-width: 640px;
  }

  &__close-title {
    margin-top: 10px;
    font-size: clamp(1.75rem, 3.4vw, 2.5rem);
    font-weight: 800;
    letter-spacing: -0.03em;
    line-height: 1.15;
    color: #fff;
  }

  &__close .land__btn {
    margin-top: 32px;
  }

  /* ==================== RESPONSIVE ==================== */
  @media (max-width: 760px) {
    &__ledger-row {
      grid-template-columns: repeat(2, 1fr);
    }

    &__ledger-cell:nth-child(2) {
      border-right: 0;
    }

    &__index-row {
      grid-template-columns: 40px 1fr;
    }

    &__index-arrow {
      display: none;
    }

    &__axis {
      grid-template-columns: 1fr;
      row-gap: 32px;
    }

    &__axis-line {
      display: none;
    }

    &__transcript-wrap {
      grid-template-columns: 1fr;
      gap: 32px;
    }
  }
}
