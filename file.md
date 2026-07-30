import { Link } from 'react-router-dom';
import type { CSSProperties, FC } from 'react';
import './Landing.scss';

// Pip position/animation are driven by CSS custom properties, which the
// React CSSProperties type doesn't know about — cast per usage below.
type PipStyle = CSSProperties & { '--x': string; '--d': string };

const Landing: FC = () => {
  return (
    <div className="landing">
      {/* ==================== HERO ==================== */}
      <header className="landing__hero" id="top">
        <div className="landing__shell landing__hero-in">
          <div className="landing__hero-copy">
            <span className="landing__hero-badge">
              <span className="landing__hero-badge-dot" />
              Model Evaluation Platform · v1.0.0
            </span>
            <p className="landing__eyebrow">Model evaluation for enterprise AI</p>
            <h1>
              Stop choosing models <em>on vibes.</em>
            </h1>
            <p className="landing__hero-sub">
              SemcoEval runs the same test suite across every model you're considering and puts
              accuracy, latency and cost on one page — so the model you ship is the one the
              evidence picked.
            </p>
            <div className="landing__hero-cta">
              <Link className="landing__btn landing__btn--primary landing__btn--lg" to="/app">
                Run an evaluation
              </Link>
              <a className="landing__btn landing__btn--ghost landing__btn--lg" href="#run">
                See how it works
              </a>
            </div>
          </div>

          {/* SIGNATURE: the score rails */}
          <section className="landing__panel" aria-label="Sample evaluation result">
            <div className="landing__panel-head">
              <span className="landing__panel-title">Run 4127 — Enterprise QA suite</span>
              <span className="landing__panel-meta">240 prompts · 4 models · complete</span>
            </div>

            <div className="landing__rails">
              <div className="landing__rail">
                <div className="landing__rail-top">
                  <span className="landing__rail-label">Accuracy</span>
                  <span className="landing__rail-unit">% graded correct</span>
                  <span className="landing__rail-dir">higher is better →</span>
                </div>
                <div className="landing__axis">
                  <span className="landing__pip" style={{ '--x': '48.8%', '--d': '.05s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      Model Alpha <b className="n">79.5</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                  <span className="landing__pip landing__pip--low" style={{ '--x': '60.3%', '--d': '.12s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      Model Beta <b className="n">84.1</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                  <span className="landing__pip landing__pip--brand" style={{ '--x': '71.8%', '--d': '.19s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      Model Gamma <b className="n">88.7</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                  <span
                    className="landing__pip landing__pip--best landing__pip--low"
                    style={{ '--x': '78.0%', '--d': '.26s' } as PipStyle}
                  >
                    <span className="landing__pip-flag">
                      Model Delta <b className="n">91.2</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                </div>
                <div className="landing__axis-scale n">
                  <span>60</span>
                  <span>70</span>
                  <span>80</span>
                  <span>90</span>
                  <span>100</span>
                </div>
              </div>

              <div className="landing__rail">
                <div className="landing__rail-top">
                  <span className="landing__rail-label">Response time</span>
                  <span className="landing__rail-unit">p95, seconds</span>
                  <span className="landing__rail-dir">← lower is better</span>
                </div>
                <div className="landing__axis">
                  <span className="landing__pip landing__pip--best" style={{ '--x': '12.5%', '--d': '.10s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      Model Alpha <b className="n">0.4</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                  <span className="landing__pip landing__pip--brand" style={{ '--x': '37.5%', '--d': '.17s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      Model Gamma <b className="n">1.2</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                  <span className="landing__pip" style={{ '--x': '65.6%', '--d': '.24s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      Model Delta <b className="n">2.1</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                  <span className="landing__pip landing__pip--low" style={{ '--x': '90.6%', '--d': '.31s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      Model Beta <b className="n">2.9</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                </div>
                <div className="landing__axis-scale n">
                  <span>0.0</span>
                  <span>0.8</span>
                  <span>1.6</span>
                  <span>2.4</span>
                  <span>3.2</span>
                </div>
              </div>

              <div className="landing__rail">
                <div className="landing__rail-top">
                  <span className="landing__rail-label">Cost</span>
                  <span className="landing__rail-unit">USD per 1,000 runs</span>
                  <span className="landing__rail-dir">← lower is better</span>
                </div>
                <div className="landing__axis">
                  <span className="landing__pip landing__pip--best" style={{ '--x': '4.8%', '--d': '.15s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      Model Alpha <b className="n">$2.40</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                  <span className="landing__pip landing__pip--low" style={{ '--x': '13.6%', '--d': '.22s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      Model Beta <b className="n">$6.80</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                  <span className="landing__pip landing__pip--brand" style={{ '--x': '49.0%', '--d': '.29s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      Model Gamma <b className="n">$24.50</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                  <span className="landing__pip" style={{ '--x': '82.4%', '--d': '.36s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      Model Delta <b className="n">$41.20</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                </div>
                <div className="landing__axis-scale n">
                  <span>$0</span>
                  <span>$12.50</span>
                  <span>$25</span>
                  <span>$37.50</span>
                  <span>$50</span>
                </div>
              </div>
            </div>

            <div className="landing__panel-foot">
              <span>
                <span className="landing__k">Most accurate</span>
                <b>Model Delta</b>
              </span>
              <span>
                <span className="landing__k">Fastest</span>
                <b>Model Alpha</b>
              </span>
              <span>
                <span className="landing__k">Cheapest</span>
                <b>Model Alpha</b>
              </span>
            </div>
          </section>
        </div>
      </header>

      {/* ==================== PROOF STRIP ==================== */}
      <section className="landing__strip">
        <div className="landing__shell landing__strip-in">
          <span className="landing__stat">
            <span className="landing__stat-n n">9</span>
            <span className="landing__stat-l">providers</span>
          </span>
          <span className="landing__stat">
            <span className="landing__stat-n n">100+</span>
            <span className="landing__stat-l">models</span>
          </span>
          <span className="landing__stat">
            <span className="landing__stat-n n">14</span>
            <span className="landing__stat-l">benchmark suites</span>
          </span>
          <span className="landing__stat">
            <span className="landing__stat-n n">3</span>
            <span className="landing__stat-l">evaluation types</span>
          </span>
          <span className="landing__strip-note">
            OpenAI · Anthropic · Google · Groq · Together · OpenRouter · Azure · Fireworks · Ollama
          </span>
        </div>
      </section>

      {/* ==================== GRADING ==================== */}
      <section className="landing__band" id="grading">
        <div className="landing__shell">
          <div className="landing__band-head">
            <p className="landing__eyebrow">Grading</p>
            <h2>A score you can't audit is just a rumour.</h2>
            <p>
              Open any cell in the leaderboard and you get the prompt, the full response, and the
              reason it was marked. Here's prompt 112 from the run above.
            </p>
          </div>

          <div className="landing__band-body">
            <div className="landing__ask">
              <p className="landing__eyebrow">Prompt 112 · Billing policy</p>
              <p className="landing__ask-q">
                A customer on the Growth plan downgrades halfway through their billing cycle. What
                happens to the credits they've already paid for?
              </p>
              <p className="landing__ask-src">
                <b>Grading source — Billing Policy v4.2 §4.2:</b> unused credits carry over as
                account balance, no refund is issued, and the plan change applies from the next
                billing date.
              </p>
            </div>

            <div className="landing__answers">
              <article className="landing__ans landing__ans--pass">
                <div className="landing__ans-top">
                  <span className="landing__ans-name">Model Delta</span>
                  <span className="landing__ans-mark">Pass</span>
                </div>
                <p>
                  Their unused credits stay on the account as a balance and roll into the next
                  cycle. No refund is issued, and the downgrade takes effect on their next billing
                  date.
                </p>
                <div className="landing__ans-foot n">
                  <span>1.9s</span>
                  <span>412 tokens</span>
                  <span>3/3 facts matched</span>
                </div>
              </article>

              <article className="landing__ans landing__ans--pass">
                <div className="landing__ans-top">
                  <span className="landing__ans-name">Model Gamma</span>
                  <span className="landing__ans-mark">Pass</span>
                </div>
                <p>
                  Credits already purchased remain available as account balance. The downgrade
                  applies from the start of the next billing period rather than immediately.
                </p>
                <div className="landing__ans-foot n">
                  <span>1.1s</span>
                  <span>288 tokens</span>
                  <span>3/3 facts matched</span>
                </div>
              </article>

              <article className="landing__ans landing__ans--pass">
                <div className="landing__ans-top">
                  <span className="landing__ans-name">Model Beta</span>
                  <span className="landing__ans-mark">Pass</span>
                </div>
                <p>
                  The remaining credits carry over to the account. The plan downgrade is scheduled
                  for the next billing date.
                </p>
                <div className="landing__ans-foot n">
                  <span>2.7s</span>
                  <span>196 tokens</span>
                  <span>2/3 facts matched</span>
                </div>
              </article>

              <article className="landing__ans landing__ans--fail">
                <div className="landing__ans-top">
                  <span className="landing__ans-name">Model Alpha</span>
                  <span className="landing__ans-mark">Fail</span>
                </div>
                <p>
                  The customer receives a <span className="landing__hl">prorated refund</span> for
                  the unused portion of their Growth plan, processed back to their original
                  payment method within 5–7 business days.
                </p>
                <div className="landing__ans-note">
                  <b>Contradicts source.</b> §4.2 states no refund is issued. The refund window and
                  payment-method detail appear nowhere in the grading source.
                </div>
                <div className="landing__ans-foot n">
                  <span>0.4s</span>
                  <span>241 tokens</span>
                  <span>0/3 facts matched</span>
                </div>
              </article>
            </div>
          </div>
        </div>
      </section>

      {/* ==================== HOW A RUN WORKS ==================== */}
      <section className="landing__band landing__band--paper" id="run">
        <div className="landing__shell">
          <div className="landing__band-head">
            <p className="landing__eyebrow">How a run works</p>
            <h2>Four steps, and the answer is a table — not an opinion.</h2>
            <p>
              Same prompts, same grader, same conditions for every model in the run. Otherwise
              you're comparing two different experiments.
            </p>
          </div>

          <div className="landing__band-body landing__pipeline">
            <div className="landing__step">
              <p className="landing__step-k">STEP 01</p>
              <h3>Connect a provider</h3>
              <p>
                Paste an API key. SemcoEval reads the provider's model list and fills in context
                length, vision and tool-calling support for you.
              </p>
              <p className="landing__step-hint">No per-model setup</p>
            </div>
            <div className="landing__step">
              <p className="landing__step-k">STEP 02</p>
              <h3>Pick the shortlist</h3>
              <p>
                Filter every connected provider at once by price, speed, context window, modality
                or benchmark score, then select the models to compare.
              </p>
              <p className="landing__step-hint">One catalogue, all providers</p>
            </div>
            <div className="landing__step">
              <p className="landing__step-k">STEP 03</p>
              <h3>Choose the tests</h3>
              <p>Start from a standard benchmark suite, or upload your own prompts and expected answers.</p>
              <p className="landing__step-hint">CSV · JSON · JSONL</p>
            </div>
            <div className="landing__step">
              <p className="landing__step-k">STEP 04</p>
              <h3>Read the results</h3>
              <p>
                Scores, response times and cost land in one leaderboard, with every response kept
                so you can check the grading yourself.
              </p>
              <p className="landing__step-hint">Export as PDF or CSV</p>
            </div>
          </div>
        </div>
      </section>

      {/* ==================== MODES ==================== */}
      <section className="landing__band" id="modes">
        <div className="landing__shell">
          <div className="landing__band-head">
            <p className="landing__eyebrow">What you can test</p>
            <h2>Three kinds of system, three different scorecards.</h2>
            <p>
              A chat model and a tool-using agent fail in completely different ways, so they
              aren't graded the same way.
            </p>
          </div>

          <div className="landing__band-body landing__modes">
            <article className="landing__mode">
              <span className="landing__mode-tag">Fast evaluation</span>
              <h3>Chat &amp; text models</h3>
              <p>Grade base knowledge, summarisation quality and conversational tone against standardised suites.</p>
              <div className="landing__chips">
                <span className="landing__chip">accuracy</span>
                <span className="landing__chip">coherence</span>
                <span className="landing__chip">tone match</span>
                <span className="landing__chip">p95 latency</span>
              </div>
            </article>
            <article className="landing__mode">
              <span className="landing__mode-tag">For automation</span>
              <h3>Autonomous agents</h3>
              <p>
                Test multi-step tool execution — whether the agent picked the right tool, with the
                right arguments, in the right order.
              </p>
              <div className="landing__chips">
                <span className="landing__chip">task completion</span>
                <span className="landing__chip">tool accuracy</span>
                <span className="landing__chip">step count</span>
                <span className="landing__chip">cost per task</span>
              </div>
            </article>
            <article className="landing__mode">
              <span className="landing__mode-tag">High precision</span>
              <h3>Document search &amp; RAG</h3>
              <p>
                Measure how much of an answer is actually supported by the retrieved documents,
                and catch the answers that aren't.
              </p>
              <div className="landing__chips">
                <span className="landing__chip">groundedness</span>
                <span className="landing__chip">retrieval recall</span>
                <span className="landing__chip">citation match</span>
                <span className="landing__chip">refusal rate</span>
              </div>
            </article>
          </div>
        </div>
      </section>

      {/* ==================== PLATFORM ==================== */}
      <section className="landing__band landing__band--paper" id="platform">
        <div className="landing__shell">
          <div className="landing__band-head">
            <p className="landing__eyebrow">Platform</p>
            <h2>Built for the part after the demo.</h2>
          </div>

          <div className="landing__band-body landing__ledger">
            <div className="landing__row">
              <p className="landing__row-k n">01</p>
              <h3>Every response is kept</h3>
              <p>
                Scores are only as good as the grading behind them. Open any cell to read the
                exact prompt, the model's full response, and why it was marked right or wrong.
              </p>
            </div>
            <div className="landing__row">
              <p className="landing__row-k n">02</p>
              <h3>Re-run against a saved baseline</h3>
              <p>
                Pin a run as your baseline. When a provider ships a new version, re-run the same
                suite and see exactly which tests moved, and in which direction.
              </p>
            </div>
            <div className="landing__row">
              <p className="landing__row-k n">03</p>
              <h3>Cost projected to your real volume</h3>
              <p>
                Enter your expected monthly request count and each model's score sits next to what
                it would actually cost you at that scale — not per million tokens.
              </p>
            </div>
            <div className="landing__row">
              <p className="landing__row-k n">04</p>
              <h3>Keys stay in your workspace</h3>
              <p>
                Provider keys are stored per workspace with scoped team access. On-prem models
                running through Ollama never leave your network.
              </p>
            </div>
            <div className="landing__row">
              <p className="landing__row-k n">05</p>
              <h3>Reports your stakeholders will read</h3>
              <p>
                Turn a run into a shareable report with the recommendation, the trade-offs and the
                raw table — the thing you take into the architecture review.
              </p>
            </div>
          </div>
        </div>
      </section>

      {/* ==================== CLOSE ==================== */}
      <section className="landing__close">
        <div className="landing__shell landing__close-in">
          <p className="landing__eyebrow">Get started</p>
          <h2>Your next model decision, with the receipts.</h2>
          <p>
            Connect one provider, pick a standard suite, and have a defensible answer before your
            next architecture review.
          </p>
          <div className="landing__close-cta">
            <Link className="landing__btn landing__btn--primary landing__btn--lg" to="/app">
              Run an evaluation
            </Link>
            <a className="landing__btn landing__btn--ghost landing__btn--lg" href="#grading">
              See a sample report
            </a>
          </div>
        </div>
      </section>
    </div>
  );
};

export default Landing;



















@use '../../styles/variables' as *;

.landing {
  --blue: #{$primary};
  --blue-2: #{$primary-hover};
  --blue-wash: #{$primary-light};
  --jade: #{$success};
  --jade-w: #{$success-subtle};
  --red: #{$danger};
  --red-w: #{$danger-subtle};
  --amber-w: #{$warning-subtle};
  --ink: #{$text-primary};
  --ink-2: #{$text-secondary};
  --ink-3: #{$text-tertiary};
  --white: #fff;
  --paper: #{$bg-subtle};
  --rule: #{$border-default};
  --rule-2: #{$border-subtle};
  --gut: clamp(1.25rem, 5vw, 3.75rem);
  --band: clamp(4.5rem, 8.5vw, 7.75rem);

  background: var(--white);
  color: var(--ink);

  &__shell {
    max-width: 1200px;
    margin: 0 auto;
    padding: 0 var(--gut);
  }

  &__hero-badge {
    display: inline-flex;
    align-items: center;
    gap: 0.5rem;
    font-size: 0.8125rem;
    font-weight: 600;
    color: var(--blue);
    background: var(--blue-wash);
    border: 1px solid $primary-subtle;
    border-radius: 999px;
    padding: 0.375rem 0.75rem 0.375rem 0.625rem;
    margin-bottom: 1.125rem;
  }

  &__hero-badge-dot {
    width: 6px;
    height: 6px;
    border-radius: 50%;
    background: var(--jade);
    box-shadow: 0 0 0 0.1875rem var(--jade-w);
  }

  &__eyebrow {
    font-size: 0.7185rem;
    font-weight: 600;
    letter-spacing: 0.15em;
    text-transform: uppercase;
    color: var(--ink-3);
    display: flex;
    align-items: center;
    gap: 0.625rem;

    &::before {
      content: '';
      width: 20px;
      height: 1px;
      background: var(--blue);
    }
  }

  h1,
  h2,
  h3 {
    letter-spacing: -0.032em;
    line-height: 1.06;
    font-weight: 700;
  }

  /* ================= BUTTONS ================= */
  &__btn {
    font-family: $font-body;
    font-size: 0.9375rem;
    font-weight: 600;
    padding: 0.625rem 1.125rem;
    border-radius: 0.5rem;
    border: 1px solid transparent;
    cursor: pointer;
    text-decoration: none;
    display: inline-flex;
    align-items: center;
    gap: 0.5rem;
    transition: background 0.15s ease, border-color 0.15s ease, color 0.15s ease;

    &--lg {
      padding: 0.875rem 1.5rem;
      font-size: 1rem;
    }

    &--primary {
      background: var(--blue);
      color: var(--white);

      &:hover {
        background: var(--blue-2);
      }
    }

    &--ghost {
      background: var(--white);
      color: var(--ink);
      border-color: var(--rule);

      &:hover {
        border-color: var(--ink);
      }
    }

    &:focus-visible {
      outline: 0.125rem solid var(--blue);
      outline-offset: 0.1875rem;
    }
  }

  /* ================= HERO ================= */
  &__hero {
    position: relative;
    overflow: hidden;
    padding: clamp(3.125rem, 6.5vw, 5.5rem) 0 var(--band);

    &::before {
      content: '';
      position: absolute;
      inset: 0;
      background-image: linear-gradient(to right, var(--rule-2) 0.0625rem, transparent 0.0625rem),
        linear-gradient(to bottom, var(--rule-2) 0.0625rem, transparent 0.0625rem);
      background-size: 3.375rem 3.375rem;
      mask-image: radial-gradient(115% 80% at 45% 0%, #000 22%, transparent 72%);
      -webkit-mask-image: radial-gradient(115% 80% at 45% 0%, #000 22%, transparent 72%);
      pointer-events: none;
    }
  }

  &__hero-in {
    position: relative;
  }

  &__hero-copy {
    max-width: 720px;
  }

  &__hero h1 {
    font-size: clamp(2.5rem, 6.2vw, 4.625rem);
    margin: 1.25rem 0 0;

    em {
      font-style: normal;
      color: var(--blue);
      position: relative;
      white-space: nowrap;

      &::after {
        content: '';
        position: absolute;
        left: 0;
        right: 0;
        bottom: -0.13em;
        height: 6px;
        background: repeating-linear-gradient(to right, var(--blue) 0 0.09375rem, transparent 0.09375rem 0.4375rem);
        opacity: 0.45;
      }
    }
  }

  &__hero-sub {
    margin-top: 1.375rem;
    max-width: 560px;
    font-size: clamp(1rem, 1.5vw, 1.156rem);
    color: var(--ink-2);
  }

  &__hero-cta {
    margin-top: 2rem;
    display: flex;
    flex-wrap: wrap;
    gap: 0.75rem;
    align-items: center;
  }

  /* ================= SCORE PANEL ================= */
  &__panel {
    margin-top: clamp(2.625rem, 5.5vw, 4.25rem);
    background: var(--white);
    border: 1px solid var(--rule);
    border-radius: 0.875rem;
    box-shadow: 0 0.0625rem 0.125rem rgba(14, 21, 38, 0.04), 0 1.25rem 2.875rem -1.625rem rgba(14, 21, 38, 0.22);
    overflow: hidden;
  }

  &__panel-head {
    display: flex;
    align-items: center;
    flex-wrap: wrap;
    gap: 0.5rem 0.875rem;
    padding: 0.875rem 1.25rem;
    border-bottom: 1px solid var(--rule-2);
    background: var(--paper);
  }

  &__panel-title {
    font-size: 0.9375rem;
    font-weight: 600;
    letter-spacing: -0.01em;
  }

  &__panel-meta {
    margin-left: auto;
    font-size: 0.84375rem;
    color: var(--ink-3);
    display: inline-flex;
    align-items: center;
    gap: 0.4375rem;

    &::before {
      content: '';
      width: 6px;
      height: 6px;
      border-radius: 50%;
      background: var(--jade);
      box-shadow: 0 0 0 0.1875rem var(--jade-w);
    }
  }

  &__rails {
    padding: 0.375rem 2.5rem 1.25rem;
  }

  &__rail {
    padding: 1.25rem 0;
    border-bottom: 1px dashed var(--rule);

    &:last-child {
      border-bottom: 0;
    }
  }

  &__rail-top {
    display: flex;
    align-items: baseline;
    flex-wrap: wrap;
    gap: 0.25rem 0.75rem;
    margin-bottom: 1.75rem;
  }

  &__rail-label {
    font-size: 0.90625rem;
    font-weight: 600;
  }

  &__rail-unit {
    font-size: 0.8125rem;
    color: var(--ink-3);
  }

  &__rail-dir {
    margin-left: auto;
    font-size: 0.8125rem;
    color: var(--ink-3);
  }

  &__axis {
    position: relative;
    height: 2px;
    background: var(--rule);
    border-radius: 0.125rem;

    &::after {
      content: '';
      position: absolute;
      left: 0;
      right: 0;
      top: 0;
      height: 7px;
      background: repeating-linear-gradient(to right, var(--rule) 0 0.0625rem, transparent 0.0625rem 10%);
    }
  }

  &__axis-scale {
    display: flex;
    justify-content: space-between;
    font-size: 0.71875rem;
    color: var(--ink-3);
    margin-top: 0.75rem;
  }

  &__pip {
    position: absolute;
    top: 50%;
    left: var(--x);
    transform: translate(-50%, -50%);
    animation: landing-settle 0.85s cubic-bezier(0.2, 0.8, 0.2, 1) backwards;
    animation-delay: var(--d, 0s);
  }

  &__pip-dot {
    display: block;
    width: 13px;
    height: 13px;
    border-radius: 50%;
    background: var(--white);
    border: 3px solid var(--ink-3);
    transition: transform 0.15s ease;
  }

  &__pip:hover &__pip-dot {
    transform: scale(1.22);
  }

  &__pip-flag {
    position: absolute;
    bottom: 1.125rem;
    left: 50%;
    transform: translateX(-50%);
    white-space: nowrap;
    font-size: 0.78125rem;
    color: var(--ink-2);
    background: var(--white);
    padding: 0.0625rem 0.3125rem;
    border-radius: 0.25rem;

    b {
      color: var(--ink);
      font-weight: 700;
    }
  }

  &__pip--low &__pip-flag {
    bottom: auto;
    top: 1.125rem;
  }

  &__pip--best &__pip-dot {
    border-color: var(--jade);
  }

  &__pip--best &__pip-flag,
  &__pip--best &__pip-flag b {
    color: var(--jade);
  }

  &__pip--brand &__pip-dot {
    border-color: var(--blue);
  }

  &__panel-foot {
    display: flex;
    flex-wrap: wrap;
    gap: 0.5rem 1.625rem;
    padding: 0.875rem 1.25rem;
    border-top: 1px solid var(--rule-2);
    background: var(--paper);
    font-size: 0.875rem;
    color: var(--ink-2);

    b {
      color: var(--ink);
      font-weight: 600;
    }
  }

  &__k {
    font-size: 0.6875rem;
    font-weight: 600;
    letter-spacing: 0.13em;
    text-transform: uppercase;
    color: var(--ink-3);
    margin-right: 0.4375rem;
  }

  /* ================= BANDS ================= */
  &__band {
    padding: var(--band) 0;

    &--paper {
      background: var(--paper);
      border-block: 0.0625rem solid var(--rule-2);
    }
  }

  &__band-head {
    max-width: 660px;

    .landing__eyebrow {
      color: var(--blue);
    }

    h2 {
      font-size: clamp(1.75rem, 3.7vw, 2.75rem);
      margin: 1rem 0 0;
    }

    p {
      margin-top: 1rem;
      color: var(--ink-2);
      font-size: 1.125rem;
    }
  }

  &__band-body {
    margin-top: clamp(2.125rem, 4.5vw, 3.375rem);
  }

  /* ================= PROOF STRIP ================= */
  &__strip {
    background: var(--paper);
    border-block: 0.0625rem solid var(--rule-2);
  }

  &__strip-in {
    display: flex;
    flex-wrap: wrap;
    gap: 0.625rem 2.875rem;
    padding: 1.375rem 0;
    align-items: baseline;
  }

  &__stat {
    display: flex;
    align-items: baseline;
    gap: 0.5625rem;
  }

  &__stat-n {
    font-size: 1.375rem;
    font-weight: 700;
    letter-spacing: -0.03em;
  }

  &__stat-l {
    font-size: 0.875rem;
    color: var(--ink-2);
  }

  &__strip-note {
    margin-left: auto;
    font-size: 0.84375rem;
    color: var(--ink-3);
  }

  /* ================= GRADED TRANSCRIPT ================= */
  &__ask {
    background: var(--white);
    border: 1px solid var(--rule);
    border-left: 3px solid var(--blue);
    border-radius: 0.625rem;
    padding: 1.25rem 1.375rem;

    .landing__eyebrow {
      color: var(--ink-3);

      &::before {
        background: var(--ink-3);
      }
    }
  }

  &__ask-q {
    margin-top: 0.75rem;
    font-size: 1.15625rem;
    line-height: 1.45;
  }

  &__ask-src {
    margin-top: 1rem;
    padding: 0.6875rem 0.8125rem;
    background: var(--paper);
    border-radius: 0.4375rem;
    font-size: 0.875rem;
    color: var(--ink-2);

    b {
      color: var(--ink);
      font-weight: 600;
    }
  }

  &__answers {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 0.875rem;
    margin-top: 0.875rem;
  }

  &__ans {
    background: var(--white);
    border: 1px solid var(--rule);
    border-left: 3px solid var(--rule);
    border-radius: 0.625rem;
    padding: 1rem 1.125rem;
    display: flex;
    flex-direction: column;

    &--pass {
      border-left-color: var(--jade);

      .landing__ans-mark {
        background: var(--jade-w);
        color: var(--jade);
      }
    }

    &--fail {
      border-left-color: var(--red);

      .landing__ans-mark {
        background: var(--red-w);
        color: var(--red);
      }
    }

    p {
      font-size: 0.96875rem;
      color: var(--ink-2);
    }
  }

  &__ans-top {
    display: flex;
    align-items: center;
    gap: 0.625rem;
    margin-bottom: 0.6875rem;
  }

  &__ans-name {
    font-size: 0.90625rem;
    font-weight: 600;
  }

  &__ans-mark {
    margin-left: auto;
    font-size: 0.6875rem;
    font-weight: 700;
    letter-spacing: 0.11em;
    text-transform: uppercase;
    padding: 0.1875rem 0.5rem;
    border-radius: 0.3125rem;
  }

  &__ans-foot {
    margin-top: auto;
    padding-top: 0.8125rem;
    font-size: 0.84375rem;
    display: flex;
    gap: 1rem;
    color: var(--ink-3);
  }

  &__ans-note {
    margin-top: 0.75rem;
    background: var(--red-w);
    color: var(--red);
    border-radius: 0.4375rem;
    padding: 0.5625rem 0.6875rem;
    font-size: 0.84375rem;
    line-height: 1.45;
  }

  &__hl {
    background: var(--amber-w);
    padding: 0 0.1875rem;
    border-radius: 0.1875rem;
    color: var(--ink);
  }

  /* ================= PIPELINE ================= */
  &__pipeline {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 1rem;
  }

  &__step {
    padding: 1.5rem 1.375rem 1.625rem;
    border: 1px solid var(--rule);
    border-radius: 0.75rem;
    background: var(--white);
    position: relative;

    &::before {
      content: '';
      position: absolute;
      top: -0.0625rem;
      left: -0.0625rem;
      width: 40px;
      height: 3px;
      border-radius: 0.75rem 0 0 0;
      background: var(--blue);
    }

    h3 {
      font-size: 1.21875rem;
      margin: 0.75rem 0 0.5625rem;
    }

    p {
      font-size: 0.96875rem;
      color: var(--ink-2);
    }
  }

  &__step-k {
    font-size: 0.75rem;
    font-weight: 700;
    letter-spacing: 0.12em;
    color: var(--blue);
  }

  &__step-hint {
    margin-top: 0.875rem;
    padding-top: 0.75rem;
    border-top: 1px dashed var(--rule);
    font-size: 0.8125rem;
    color: var(--ink-3);
  }

  /* ================= MODES ================= */
  &__modes {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 1rem;
  }

  &__mode {
    background: var(--white);
    border: 1px solid var(--rule);
    border-radius: 0.75rem;
    padding: 1.5rem 1.375rem;
    display: flex;
    flex-direction: column;
    transition: border-color 0.15s ease, transform 0.15s ease;

    &:hover {
      border-color: var(--ink);
      transform: translateY(-0.125rem);
    }

    h3 {
      font-size: 1.25rem;
      margin: 1rem 0 0.625rem;
    }

    p {
      font-size: 0.96875rem;
      color: var(--ink-2);
    }
  }

  &__mode-tag {
    align-self: flex-start;
    font-size: 0.6875rem;
    font-weight: 600;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    color: var(--blue);
    background: var(--blue-wash);
    padding: 0.3125rem 0.5625rem;
    border-radius: 0.3125rem;
  }

  &__chips {
    margin-top: 1.25rem;
    padding-top: 0.875rem;
    border-top: 1px solid var(--rule-2);
    display: flex;
    flex-wrap: wrap;
    gap: 0.375rem;
  }

  &__chip {
    font-size: 0.78125rem;
    color: var(--ink-2);
    border: 1px solid var(--rule);
    border-radius: 0.3125rem;
    padding: 0.1875rem 0.5rem;
  }

  /* ================= LEDGER ================= */
  &__ledger {
    border-top: 1px solid var(--rule);
  }

  &__row {
    display: grid;
    grid-template-columns: 2.5rem 1.05fr 1.45fr;
    gap: 1.5rem;
    padding: 1.625rem 0;
    border-bottom: 1px solid var(--rule);
    align-items: start;

    h3 {
      font-size: 1.28125rem;
    }

    p {
      font-size: 1rem;
      color: var(--ink-2);
    }
  }

  &__row-k {
    font-size: 0.8125rem;
    color: var(--ink-3);
    padding-top: 0.3125rem;
  }

  /* ================= CLOSE ================= */
  &__close {
    padding: var(--band) 0;
  }

  &__close-in {
    max-width: 700px;

    .landing__eyebrow {
      color: var(--blue);
    }

    h2 {
      font-size: clamp(1.875rem, 4.4vw, 3.125rem);
      margin: 1rem 0 0;
    }

    p {
      margin-top: 1.125rem;
      color: var(--ink-2);
      font-size: 1.125rem;
    }
  }

  &__close-cta {
    margin-top: 1.875rem;
    display: flex;
    flex-wrap: wrap;
    gap: 0.75rem;
  }

  /* ================= RESPONSIVE ================= */
  @media (max-width: 1000px) {
    .landing__pipeline {
      grid-template-columns: 1fr 1fr;
    }
    .landing__modes {
      grid-template-columns: 1fr;
    }
  }

  @media (max-width: 860px) {
    .landing__answers {
      grid-template-columns: 1fr;
    }
    .landing__row {
      grid-template-columns: 1.875rem 1fr;
    }
    .landing__row p {
      grid-column: 2;
    }
  }

  @media (max-width: 620px) {
    .landing__pipeline {
      grid-template-columns: 1fr;
    }
    .landing__rails {
      padding-inline: 1.75rem;
    }
    .landing__rail-dir {
      display: none;
    }
    .landing__pip-flag {
      font-size: 0.71875rem;
    }
  }

  @media (prefers-reduced-motion: reduce) {
    .landing__pip {
      animation: none;
    }
    .landing__mode {
      transition: none;
    }
  }
}

@keyframes landing-settle {
  from {
    left: 0%;
    opacity: 0;
  }
  to {
    left: var(--x);
    opacity: 1;
  }
}
