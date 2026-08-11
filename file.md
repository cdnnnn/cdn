import { useEffect, useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { ArrowRight, Play, Award, Link2, Cpu, FlaskConical, BarChart3, GitCompare, Shield } from 'lucide-react';
import Footer from '../layout/Footer';
import ThemeToggle from '../common/ThemeToggle';
import styles from './Landing.module.scss';

const features = [
  { icon: <Link2 size={22} />, cls: 'signal', title: 'Provider Hub', desc: 'Connect any AI provider with API keys. Manage credentials, monitor status, and see available models instantly.' },
  { icon: <Cpu size={22} />, cls: 'amber', title: 'Model Catalog', desc: 'Browse all models across providers. Filter by capability, compare pricing, and register custom endpoints.' },
  { icon: <FlaskConical size={22} />, cls: 'ok', title: 'Guided Evaluations', desc: 'A step-by-step wizard for model selection, test suite choice, and metric configuration.' },
  { icon: <BarChart3 size={22} />, cls: 'sky', title: 'Results & History', desc: 'Every evaluation stored with full breakdowns. Duplicate past runs, track trends, export findings.' },
  { icon: <GitCompare size={22} />, cls: 'rose', title: 'Visual Comparison', desc: 'Radar charts and metric tables make it obvious where each model excels or falls short.' },
  { icon: <Shield size={22} />, cls: 'signal', title: 'SSO & Security', desc: 'Enterprise-grade sign-in. API keys encrypted and isolated to your environment.' },
];

export default function Landing() {
  const [animated, setAnimated] = useState(false);
  const navigate = useNavigate();

  useEffect(() => {
    const t = setTimeout(() => setAnimated(true), 200);
    return () => clearTimeout(t);
  }, []);

  // Landing is now gated by AuthGuard (wrapping "/"), so by the time this
  // page renders the user is already authenticated — these buttons are
  // plain navigation into the app, not sign-in triggers.
  const goToDashboard = () => navigate('/app/dashboard');

  const bars = [
    { label: 'Claude Sonnet 4', pct: animated ? 94 : 0, cls: 'primary' },
    { label: 'GPT-4o', pct: animated ? 91 : 0, cls: 'warm' },
    { label: 'Gemini 2.5 Pro', pct: animated ? 89 : 0, cls: 'cool' },
    { label: 'Mistral Large', pct: animated ? 85 : 0, cls: 'gray' },
  ];

  return (
    <div className={styles.landing}>
      <nav className={styles['l-nav']}>
        <div className={styles['l-logo']}><div className={styles.mark}>&#9670;</div>SemcoEval</div>
        <ThemeToggle />
      </nav>

      <section className={styles['hero-section']}>
        <div className={styles['hero-bg-grid']} />
        <div className={styles['hero-bg-glow']} />
        <div className={styles['hero-content']}>
          <div className={styles['hero-badge']}><div className={styles['badge-dot']} /> Now supporting 40+ models</div>
          <h1>Evaluate AI models<br />with <span className={styles.grad}>measured evidence</span></h1>
          <p>Stop guessing which model fits your use case. Run structured benchmarks, compare results side-by-side, and make selection decisions backed by real data.</p>
          <div className={styles['hero-actions']}>
            <button className={styles['btn-primary']} onClick={goToDashboard}>Open Dashboard <ArrowRight size={16} /></button>
            <button className={styles['btn-secondary']}><Play size={16} /> Watch Demo</button>
          </div>
        </div>
        <div className={styles['hero-visual']}>
          <div className={styles['hero-card']}>
            <div className={styles['hero-card-hdr']}>
              <span className={styles['hero-card-title']}>Live Benchmark</span>
              <span className={styles['hero-card-badge']}><div className={styles['pulse-dot']} /> Running</span>
            </div>
            {bars.map((b, i) => (
              <div className={styles['hero-bar']} key={i}>
                <span className={styles['hero-bar-label']}>{b.label}</span>
                <div className={styles['hero-bar-track']}>
                  <div className={`${styles['hero-bar-fill']} ${styles[`hero-bar-fill--${b.cls}`]}`} style={{ width: `${b.pct}%`, transitionDelay: `${i * 250}ms` }}>
                    {b.pct > 0 && <span>{b.pct}%</span>}
                  </div>
                </div>
              </div>
            ))}
          </div>
          <div className={`${styles['float-badge']} ${styles.tr}`}><Award size={16} className={styles['float-badge__icon']} /><span>Winner: <strong>Claude Sonnet 4</strong></span></div>
          <div className={`${styles['float-badge']} ${styles.bl}`}><div className={styles['pulse-dot']} /><span>3 evaluations running</span></div>
        </div>
      </section>

      <section className={styles.features}>
        <div className={styles['feat-header']}><h2>Everything you need to decide</h2><p>From connecting providers to comparing results — a complete evaluation workflow</p></div>
        <div className={styles['feat-grid']}>
          {features.map((f, i) => (
            <div className={styles['feat-card']} key={i}>
              <div className={`${styles['feat-icon']} ${styles[`feat-icon--${f.cls}`]}`}>{f.icon}</div>
              <h3>{f.title}</h3>
              <p>{f.desc}</p>
            </div>
          ))}
        </div>
      </section>

      <section className={styles['stats-section']}>
        {[{ v: '40+', l: 'Models supported' }, { v: '6', l: 'Benchmark suites' }, { v: '12K+', l: 'Evaluation tasks' }, { v: '<5min', l: 'Average eval time' }].map((s, i) => (
          <div className={styles['stat-box']} key={i}><div className={styles['stat-val']}>{s.v}</div><div className={styles['stat-lbl']}>{s.l}</div></div>
        ))}
      </section>

      <section className={styles['cta-section']}>
        <div className={styles['cta-box']}>
          <h2>Ready to evaluate with confidence?</h2>
          <p>Connect your first provider and run a benchmark in under five minutes.</p>
          <button className={styles['btn-primary']} onClick={goToDashboard}>Get Started Free <ArrowRight size={16} /></button>
        </div>
      </section>

      <Footer />
    </div>
  );
}




















@use '../../styles/_variables' as *;

// ===========================================================================
// Landing — mirrors the ink/paper/signal design system used across History,
// Reports, Comparison, New Evaluation and Sidebar. Gradients replaced with
// the flat ultramarine signal accent; mono numerals for stats/benchmarks.
// Neutrals resolve to theme CSS vars so the page is dark-mode aware.
// ===========================================================================

$ink:      var(--ink-1);
$ink-2:    var(--ink-2);
$ink-3:    var(--ink-3);
$paper:    var(--paper);
$card:     var(--card);
$line:     var(--line);
$line-2:   var(--line-2);
$signal:   #2B2BF5;
$signal-2: #1C1CC7;
$wash:     var(--signal-wash);
$ok:       #0FA968;
$ok-wash:  var(--ok-wash);
$amber:    #E08600;
$amber-wash: var(--amber-wash);
$danger:   #DC2626;
$danger-wash: var(--danger-wash);
$ink-wash: var(--ink-wash);
$sky:      #0369A1;
$sky-wash: var(--sky-wash);
$rose:     #DB2777;
$rose-wash: var(--rose-wash);

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

$soft: 0 1px 2px rgba(20, 22, 27, 0.05);
$lift: 0 14px 30px -14px rgba(20, 22, 27, 0.22);
$lift-lg: 0 26px 50px -20px rgba(20, 22, 27, 0.28);

.landing { min-height: 100vh; background: $card; overflow-x: hidden; }

.l-error {
  max-width: 1440px; margin: 0 auto; padding: 0 48px;
  color: $danger; font-size: 13px; font-weight: 600;
}

.l-nav {
  display: flex; align-items: center; justify-content: space-between; padding: 18px 48px;
  max-width: 1440px; margin: 0 auto; position: relative; z-index: 10;
}
.l-logo {
  font-family: $display; font-size: 22px; font-weight: 800; letter-spacing: -0.02em;
  display: flex; align-items: center; gap: 10px; color: $ink;

  .mark {
    width: 34px; height: 34px; background: $ink; border-radius: 10px;
    display: flex; align-items: center; justify-content: center; color: #fff; font-size: 18px;
    position: relative; overflow: hidden;

    &::after {
      content: '';
      position: absolute;
      inset: 0;
      background: linear-gradient(140deg, transparent 40%, rgba($signal, 0.9) 140%);
    }
  }
}
.l-links {
  display: flex; gap: 32px; align-items: center;
  a { color: $ink-2; text-decoration: none; font-size: 14px; font-weight: 500; transition: color 0.2s; cursor: pointer; }
  a:hover { color: $ink; }
}

.btn-primary {
  display: inline-flex; align-items: center; gap: 8px; background: $signal; color: #fff; border: none;
  padding: 14px 28px; border-radius: 14px; font-size: 15px; font-weight: 700; cursor: pointer; transition: all 0.2s ease;
  font-family: $display; box-shadow: 0 8px 20px -8px rgba($signal, 0.65);
}
.btn-primary:hover { background: $signal-2; transform: translateY(-2px); box-shadow: 0 12px 26px -8px rgba($signal, 0.7); }
.btn-primary:disabled { opacity: 0.6; cursor: default; transform: none; }

.btn-secondary {
  display: inline-flex; align-items: center; gap: 8px; background: $card; color: $ink;
  border: 1.5px solid $line; padding: 14px 28px; border-radius: 14px; font-size: 15px; font-weight: 700;
  cursor: pointer; transition: all 0.2s ease; font-family: $display;
}
.btn-secondary:hover { border-color: $signal; color: $signal; background: $wash; box-shadow: $soft; }

.hero-section {
  position: relative; max-width: 1440px; margin: 0 auto; padding: 64px 48px 80px;
  display: grid; grid-template-columns: 1fr 1fr; gap: 80px; align-items: center;
}

// Fine line grid with a radial fade-out mask — the grid is crisp behind the
// headline and dissolves toward the edges so it never competes with content.
.hero-bg-grid {
  position: absolute;
  inset: 0;
  pointer-events: none;
  z-index: 1;
  background-image:
    linear-gradient(to right, $line 1px, transparent 1px),
    linear-gradient(to bottom, $line 1px, transparent 1px);
  background-size: 44px 44px;
  -webkit-mask-image: radial-gradient(115% 115% at 20% 0%, #000 30%, transparent 72%);
  mask-image: radial-gradient(115% 115% at 20% 0%, #000 30%, transparent 72%);
  opacity: 0.7;
}

// Soft signal-tinted glow behind the headline, sitting over the grid.
.hero-bg-glow {
  position: absolute;
  top: -10%;
  left: 8%;
  width: 460px;
  height: 460px;
  pointer-events: none;
  z-index: 1;
  border-radius: 50%;
  background: radial-gradient(circle, rgba($signal, 0.14), transparent 68%);
  filter: blur(20px);
}

.hero-content { position: relative; z-index: 2; }
.hero-badge {
  display: inline-flex; align-items: center; gap: 8px; background: $wash;
  border: 1px solid rgba($signal, 0.18); border-radius: 100px; padding: 6px 16px 6px 8px;
  font-size: 13px; color: $signal; margin-bottom: 28px; font-weight: 700;
  font-family: $mono; letter-spacing: 0.01em;

  .badge-dot { width: 8px; height: 8px; border-radius: 50%; background: $signal; animation: pulse 2s infinite; }
}
.hero-content h1 {
  font-family: $display;
  font-size: 54px; font-weight: 800; line-height: 1.08; letter-spacing: -0.045em; margin-bottom: 24px;
  color: $ink;
  .grad { color: $signal; }
}
.hero-content > p { font-size: 17px; color: $ink-2; line-height: 1.7; margin-bottom: 40px; max-width: 480px; }
.hero-actions { display: flex; gap: 14px; }

.hero-visual { position: relative; z-index: 2; }
.hero-card { background: $card; border: 1px solid $line; border-radius: 20px; padding: 28px; box-shadow: $lift-lg; }
.hero-card-hdr { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; }
.hero-card-title {
  font-family: $mono; font-size: 0.6875rem; font-weight: 700; color: $ink-3;
  text-transform: uppercase; letter-spacing: 0.14em;
}
.hero-card-badge {
  display: inline-flex; align-items: center; gap: 6px;
  font-family: $mono; font-size: 0.625rem; font-weight: 700; letter-spacing: 0.06em; text-transform: uppercase;
  color: $ok; background: $ok-wash; border-radius: 999px; padding: 4px 10px 4px 8px;
}
.hero-bar { display: flex; align-items: center; gap: 12px; margin-bottom: 12px; }
.hero-bar:last-child { margin-bottom: 0; }
.hero-bar-label { font-size: 13px; color: $ink-2; width: 120px; flex-shrink: 0; font-weight: 650; }
.hero-bar-track { flex: 1; height: 34px; background: $paper; border-radius: 10px; overflow: hidden; }
.hero-bar-fill {
  height: 100%; border-radius: 10px; display: flex; align-items: center; justify-content: flex-end; padding: 0 14px;
  font-family: $mono; font-size: 12px; font-weight: 700; color: #fff; transition: width 1.8s cubic-bezier(0.16, 1, 0.3, 1);

  &--primary { background: $signal; }
  &--warm    { background: $amber; }
  &--cool    { background: $sky; }
  &--gray    { background: $ink-3; }
}

.float-badge {
  position: absolute; background: $card; border: 1px solid $line; border-radius: 14px;
  padding: 12px 18px; display: flex; align-items: center; gap: 10px; box-shadow: $lift;
  z-index: 3; animation: float 4s ease infinite; font-size: 13px; color: $ink-2; font-weight: 550;
  strong { color: $ink; font-weight: 700; }
}
.float-badge__icon { color: $amber; }
.float-badge.tr { top: -24px; right: -16px; animation-delay: 0.5s; }
.float-badge.bl { bottom: -20px; left: -16px; animation-delay: 1.5s; }
.pulse-dot { width: 8px; height: 8px; border-radius: 50%; background: $ok; animation: pulse 2s infinite; }

.features { max-width: 1440px; margin: 0 auto; padding: 96px 48px; background: $paper; }
.feat-header { text-align: center; margin-bottom: 72px; }
.feat-header h2 { font-family: $display; font-size: 38px; font-weight: 800; letter-spacing: -0.03em; margin-bottom: 14px; color: $ink; }
.feat-header p { color: $ink-2; font-size: 16px; max-width: 460px; margin: 0 auto; line-height: 1.6; }
.feat-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 20px; }
.feat-card {
  background: $card; border: 1px solid $line; border-radius: 18px; padding: 32px;
  transition: all 0.25s ease; cursor: default; position: relative; overflow: hidden;

  &::before { content: ''; position: absolute; top: 0; left: 0; right: 0; height: 3px; background: $signal; opacity: 0; transition: opacity 0.25s ease; }
  &:hover { border-color: transparent; box-shadow: $lift; transform: translateY(-6px); }
  &:hover::before { opacity: 1; }
  h3 { font-family: $display; font-size: 17px; font-weight: 700; margin-bottom: 8px; letter-spacing: -0.01em; color: $ink; }
  p { font-size: 14px; color: $ink-2; line-height: 1.65; }
}
.feat-icon {
  width: 50px; height: 50px; border-radius: 14px; display: flex; align-items: center; justify-content: center; margin-bottom: 22px;
  &--signal { background: $wash; color: $signal; }
  &--amber  { background: $amber-wash; color: $amber; }
  &--ok     { background: $ok-wash; color: $ok; }
  &--sky    { background: $sky-wash; color: $sky; }
  &--rose   { background: $rose-wash; color: $rose; }
}

.stats-section {
  max-width: 1440px; margin: 0 auto; padding: 64px 48px; display: grid; grid-template-columns: repeat(4, 1fr);
  gap: 24px; background: $card; border-top: 1px solid $line; border-bottom: 1px solid $line;
}
.stat-box { text-align: center; padding: 16px; }
.stat-val { font-family: $mono; font-size: 44px; font-weight: 700; letter-spacing: -0.03em; color: $signal; }
.stat-lbl { font-size: 14px; color: $ink-2; margin-top: 4px; font-weight: 600; }

.cta-section { max-width: 1440px; margin: 0 auto; padding: 96px 48px; text-align: center; background: $paper; }
.cta-box {
  background: $card; border: 1px solid $line; border-radius: 28px; padding: 72px;
  position: relative; overflow: hidden; box-shadow: $lift;
  &::before { content: ''; position: absolute; inset: -2px; background: $signal; border-radius: 30px; z-index: -1; opacity: 0.12; }
  h2 { font-family: $display; font-size: 38px; font-weight: 800; letter-spacing: -0.03em; margin-bottom: 14px; color: $ink; }
  p { color: $ink-2; font-size: 16px; margin-bottom: 36px; line-height: 1.6; }
}

.l-footer { text-align: center; padding: 32px 48px; color: $ink-3; font-size: 13px; border-top: 1px solid $line; background: $card; }

@keyframes pulse {
  0%, 100% { opacity: 1; transform: scale(1); }
  50% { opacity: 0.5; transform: scale(1.3); }
}
@keyframes float {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-8px); }
}

@media (max-width: 768px) {
  .hero-section { grid-template-columns: 1fr; padding: 48px 24px; gap: 40px; }
  .hero-content h1 { font-size: 36px; }
  .feat-grid { grid-template-columns: 1fr; }
  .stats-section { grid-template-columns: repeat(2, 1fr); }
}
