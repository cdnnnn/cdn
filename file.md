import { useEffect } from 'react';
import { useNavigate } from 'react-router-dom';
import { Loader2, TrendingUp, Play, Plus, GitCompare, BookOpen, ChevronRight } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchProviders } from '../../store/slices/providersSlice';
import { fetchModels } from '../../store/slices/modelsSlice';
import ScoreRing from '../common/ScoreRing';
import Sparkline from '../common/Sparkline';
import RadarChart from '../common/RadarChart';
import { useCounter } from '../common/useCounter';
import styles from './Dashboard.module.scss';

export default function Dashboard() {
  const dispatch = useAppDispatch();
  const navigate = useNavigate();

  const providers = useAppSelector((s) => s.providers.items);
  const models = useAppSelector((s) => s.models.items);
  const runs = useAppSelector((s) => s.evaluations.runs);

  useEffect(() => {
    dispatch(fetchProviders());
    dispatch(fetchModels());
  }, [dispatch]);

  const connectedCount = providers.filter((p) => p.status === 'connected').length;
  const totalEvals = useCounter(runs.length, 900);
  const connectedAnim = useCounter(connectedCount, 900);
  const avgAccuracy = models.length
    ? (models.reduce((sum, m) => sum + (m.accuracy_score || 0), 0) / models.length).toFixed(1)
    : '0.0';

  const stats = [
    { label: 'Total Evaluations', value: totalEvals, change: `${runs.length} tracked locally`, spark: [2, 4, 3, 5, 6, 5, runs.length || 1] },
    { label: 'Models Connected', value: connectedAnim, change: `${models.length} in catalog`, spark: [1, 2, 2, 3, 3, 4, connectedCount || 1] },
    { label: 'Avg. Accuracy', value: `${avgAccuracy}%`, change: 'Across active models', spark: [85, 86, 87, 88, 89, 90, Number(avgAccuracy) || 90] },
  ];

  const radarModels = models.slice(0, 3).map((m) => ({
    name: m.name,
    values: [
      (m.accuracy_score || 0) / 100,
      0.8,
      m.input_price ? Math.max(0, 1 - m.input_price / 10) : 0.5,
      Math.min(1, m.context_window / 200000),
      (m.agent_score || m.accuracy_score || 0) / 100,
    ],
  }));

  return (
    <div className={`page-enter ${styles.dashboard}`}>
      <div className={styles.dashboard__header}>
        <div>
          <div className={styles['dashboard__header-eyebrow']}>Overview</div>
          <h1>Dashboard</h1>
          <p className={styles['dashboard__header-sub']}>Your evaluation activity at a glance</p>
        </div>
      </div>

      <div className={styles['dashboard__body']}>
        <div className={styles['dashboard__stats']}>
          {stats.map((s, i) => (
            <div className={styles['dashboard__stat']} key={i}>
              <div className={styles['dashboard__stat-top']}>
                <div>
                  <div className={styles['dashboard__stat-label']}>{s.label}</div>
                  <div className={styles['dashboard__stat-val']}>{s.value}</div>
                  <div className={styles['dashboard__stat-change']}><TrendingUp size={12} /> {s.change}</div>
                </div>
                <Sparkline data={s.spark} color="#6366F1" width={72} height={32} />
              </div>
            </div>
          ))}
        </div>

        <div className={styles['dashboard__grid']}>
          <div className={styles['dashboard__panel']}>
            <div className={styles['dashboard__panel-hdr']}>
              <span className={styles['dashboard__panel-title']}>Recent Evaluations</span>
              <button className={styles['dashboard__link']} onClick={() => navigate('/app/evaluations')}>
                View All <ChevronRight size={14} />
              </button>
            </div>
            {runs.length === 0 && (
              <div className={styles['dashboard__empty']}>No evaluations yet — launch one from Quick Actions below.</div>
            )}
            {runs.slice(0, 4).map((run) => (
              <div key={run.id} className={styles['dashboard__run-row']} onClick={() => navigate(`/app/evaluations/${run.id}`)}>
                <div className={styles['dashboard__run-main']}>
                  {run.status === 'completed' ? (
                    <ScoreRing score={Math.round((run.progress / (run.total || 1)) * 100)} size={44} stroke={4} color="#6366F1" />
                  ) : (
                    <div className={styles['dashboard__spinner']}><Loader2 size={18} color="#6366F1" style={{ animation: 'spin 1.5s linear infinite' }} /></div>
                  )}
                  <div>
                    <div className={styles['dashboard__run-name']}>{run.name}</div>
                    <div className={styles['dashboard__run-meta']}>{run.benchmark || '—'} &middot; {new Date(run.created_at).toLocaleDateString()}</div>
                  </div>
                </div>
                <span className={`${styles['dashboard__status-pill']} ${styles[`dashboard__status-pill--${run.status}`]}`}>
                  {run.status}
                </span>
              </div>
            ))}
          </div>

          <div className={styles['dashboard__panel']} style={{ padding: 24 }}>
            <div className={styles['dashboard__section-title']}>Top Models</div>
            <div className={styles['dashboard__section-sub']}>Strength comparison across 5 dimensions</div>
            {radarModels.length > 0 ? (
              <>
                <div className="radar-wrap"><RadarChart models={radarModels} size={260} /></div>
                <div className={styles['dashboard__legend']}>
                  {radarModels.map((m, i) => (
                    <span key={i}><span className={styles['dashboard__dot']} style={{ background: ['#6366F1', '#F59E0B', '#10B981'][i] }} /> {m.name}</span>
                  ))}
                </div>
              </>
            ) : (
              <div className={styles['dashboard__empty']}>Connect a provider to see model comparisons.</div>
            )}
          </div>
        </div>

        <div className={styles['dashboard__panel']} style={{ padding: 24 }}>
          <div className={styles['dashboard__section-title']}>Quick Actions</div>
          <div className={styles['dashboard__actions']}>
            {[
              { icon: <Play size={20} />, label: 'New Evaluation', desc: 'Start a benchmark run', to: '/app/new-eval', cls: 'ind' },
              { icon: <Plus size={20} />, label: 'Add Provider', desc: 'Connect an API', to: '/app/providers', cls: 'em' },
              { icon: <GitCompare size={20} />, label: 'Compare Models', desc: 'Side-by-side analysis', to: '/app/comparison', cls: 'amb' },
              { icon: <BookOpen size={20} />, label: 'Browse Suites', desc: 'Explore benchmarks', to: '/app/suites', cls: 'sky' },
            ].map((a, i) => (
              <button
                key={i}
                onClick={() => navigate(a.to)}
                className={`${styles['dashboard__qa-btn']} ${styles[`dashboard__qa-btn--${a.cls}`]}`}
              >
                <div className={styles['dashboard__qa-btn-icon']}>{a.icon}</div>
                <div>
                  <div className={styles['dashboard__qa-btn-label']}>{a.label}</div>
                  <div className={styles['dashboard__qa-btn-desc']}>{a.desc}</div>
                </div>
              </button>
            ))}
          </div>
        </div>
      </div>
    </div>
  );
}




























@use '../../styles/_variables' as *;

// ===========================================================================
// Dashboard — matches the Run Console / Model Catalog / Providers design
// system: ink/paper palette, ultramarine signal accent, mono instrument
// labels, hover-lift cards, mono numerals for data-dense cells.
//
// Font scaling: `.dashboard` sets a single base font-size. All descendant
// font-sizes are expressed in `em` (relative to that base), so bumping
// `.dashboard`'s font-size (e.g. on wide screens) scales the whole component
// proportionally from one place — same convention as Sidebar and Model Catalog.
// ===========================================================================

$ink:      #14161B;
$ink-2:    #565B66;
$ink-3:    #8A909B;
$paper:    #F5F6F8;
$card:     #FFFFFF;
$line:     #E6E8EC;
$line-2:   #EEF0F3;
$signal:   #2B2BF5;
$signal-2: #1C1CC7;
$wash:     #ECEDFF;
$ok:       #0FA968;
$ok-wash:  #E7F7EF;
$amber:    #E08600;
$amber-wash: #FDF3E3;
$danger:   #DC2626;
$danger-wash: #FDECEC;

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

$soft: 0 1px 2px rgba(20, 22, 27, 0.05);
$lift: 0 14px 30px -14px rgba(20, 22, 27, 0.22);

// base font-size the dashboard's internal `em` scale is built on
$dashboard-base-font: 0.875rem;

%micro {
  font-family: $mono;
  font-size: 0.7857em; // 0.6875rem / 0.875rem
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

.dashboard {
  // master scale control — every em-based font-size below responds to this
  font-size: $dashboard-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  // ---- header ---------------------------------------------------------------
  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding: 24px 32px 20px;
    margin-bottom: 24px;
    border-bottom: 1px solid $line;
    background: $card;

    h1 {
      font-family: $display;
      font-size: 1.7143em; // 1.5rem / 0.875rem
      font-weight: 800;
      letter-spacing: -0.02em;
      color: $ink;
      line-height: 1.2;
    }
  }

  &__header-eyebrow {
    @extend %micro;
    display: flex;
    align-items: center;
    gap: 8px;
    color: $signal;
    margin-bottom: 6px;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $signal;
    }
  }

  &__header-sub {
    margin-top: 4px;
    font-size: 0.9643em; // 0.84375rem / 0.875rem
    color: $ink-2;
  }

  &__header-meta {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 7px;
    padding: 7px 13px;
    border-radius: 999px;
    border: 1px solid $line;
    background: $paper;
    font-family: $mono;
    font-size: 0.8214em; // 0.71875rem / 0.875rem
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ink-2;
    white-space: nowrap;
    margin-bottom: 3px;
  }

  &__body {
    padding: 0 32px 32px;
  }

  // ---- stat cards -------------------------------------------------------------
  &__stats {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 14px;
    margin-bottom: 18px;
  }

  &__stat {
    position: relative;
    background: $card;
    border: 1px solid $line;
    border-radius: 16px;
    padding: 20px 22px;
    transition: border-color 0.16s ease, box-shadow 0.16s ease, transform 0.16s ease;

    &:hover {
      border-color: $ink-3;
      box-shadow: $lift;
      transform: translateY(-2px);
    }
  }

  &__stat-top {
    display: flex;
    justify-content: space-between;
    align-items: flex-start;
  }

  &__stat-label {
    @extend %micro;
    font-size: 0.7143em; // 0.625rem / 0.875rem
    color: $ink-3;
  }

  &__stat-val {
    font-family: $mono;
    font-size: 2.2857em; // 2rem / 0.875rem
    font-weight: 700;
    letter-spacing: -0.02em;
    line-height: 1;
    margin-top: 10px;
    color: $ink;
  }

  &__stat-change {
    font-size: 0.8214em; // 0.71875rem / 0.875rem
    color: $ok;
    font-weight: 600;
    margin-top: 8px;
    display: flex;
    align-items: center;
    gap: 5px;
  }

  // ---- main grid ----------------------------------------------------------------
  &__grid {
    display: grid;
    grid-template-columns: 1.2fr 1fr;
    gap: 14px;
    margin-bottom: 14px;
  }

  &__panel {
    background: $card;
    border: 1px solid $line;
    border-radius: 16px;
    overflow: hidden;
  }

  &__panel-hdr {
    padding: 17px 20px;
    border-bottom: 1px solid $line-2;
    display: flex;
    justify-content: space-between;
    align-items: center;
  }

  &__panel-title {
    display: flex;
    align-items: center;
    gap: 8px;
    font-family: $display;
    font-weight: 700;
    font-size: 1em; // 0.875rem / 0.875rem (base)
    color: $ink;

    svg { color: $signal; }
  }

  &__panel-sub {
    font-size: 0.8571em; // 0.75rem / 0.875rem
    color: $ink-3;
    margin-top: 2px;
  }

  &__link {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    font-family: $sans;
    font-size: 0.8929em; // 0.78125rem / 0.875rem
    font-weight: 650;
    color: $ink-2;
    background: transparent;
    border: 0;
    cursor: pointer;
    padding: 0;
    transition: color 0.15s ease;

    &:hover { color: $signal; }
  }

  &__run-row {
    padding: 14px 20px;
    border-bottom: 1px solid $line-2;
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 14px;
    cursor: pointer;
    transition: background 0.15s ease;

    &:last-child { border-bottom: 0; }
    &:hover { background: $paper; }
  }

  &__run-main {
    display: flex;
    align-items: center;
    gap: 13px;
    min-width: 0;
  }

  &__run-name {
    font-family: $display;
    font-weight: 700;
    font-size: 0.9643em; // 0.84375rem / 0.875rem
    color: $ink;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__run-meta {
    font-size: 0.8214em; // 0.71875rem / 0.875rem
    color: $ink-3;
    margin-top: 2px;
  }

  &__spinner {
    flex-shrink: 0;
    width: 40px;
    height: 40px;
    border-radius: 11px;
    background: $wash;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  &__fail-icon {
    flex-shrink: 0;
    width: 40px;
    height: 40px;
    border-radius: 11px;
    background: $danger-wash;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  &__empty {
    padding: 30px 20px;
    text-align: center;
    color: $ink-3;
    font-size: 0.9286em; // 0.8125rem / 0.875rem
  }

  &__legend {
    display: flex;
    justify-content: center;
    gap: 18px;
    margin-top: 6px;
    flex-wrap: wrap;

    span {
      display: flex;
      align-items: center;
      gap: 6px;
      font-size: 0.8214em; // 0.71875rem / 0.875rem
      font-weight: 600;
      color: $ink-2;
    }
  }

  &__dot {
    width: 8px;
    height: 8px;
    border-radius: 50%;
    display: inline-block;
  }

  // ---- status pill (mono, colored per run status) --------------------------------
  &__status-pill {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 4px 10px 4px 8px;
    border-radius: 999px;
    font-family: $mono;
    font-size: 0.7143em; // 0.625rem / 0.875rem
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;

    &::before {
      content: '';
      width: 5px;
      height: 5px;
      border-radius: 50%;
    }

    &--completed {
      color: $ok;
      background: $ok-wash;
      &::before { background: $ok; }
    }

    &--failed {
      color: $danger;
      background: $danger-wash;
      &::before { background: $danger; }
    }

    &--running {
      color: $signal;
      background: $wash;
      &::before { background: $signal; animation: db-pulse 1.1s ease-in-out infinite; }
    }
  }

  // ---- quick action tiles ----------------------------------------------------------
  &__actions {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 12px;
  }

  &__qa-btn {
    position: relative;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 12px;
    padding: 22px 18px;
    border: 1.5px solid $line;
    border-radius: 16px;
    background: $card;
    cursor: pointer;
    text-align: center;
    transition: border-color 0.18s ease, box-shadow 0.18s ease, transform 0.18s ease;

    &:hover {
      border-color: $ink-3;
      box-shadow: $lift;
      transform: translateY(-3px);
    }

    &--ind .dashboard__qa-btn-icon { background: $signal; }
    &--em  .dashboard__qa-btn-icon { background: $ok; }
    &--amb .dashboard__qa-btn-icon { background: $amber; }
    &--sky .dashboard__qa-btn-icon { background: #0369A1; }
  }

  &__qa-btn-icon {
    width: 46px;
    height: 46px;
    border-radius: 13px;
    display: flex;
    align-items: center;
    justify-content: center;
    color: #fff;
    background: $ink;
    position: relative;
    overflow: hidden;

    &::after {
      content: '';
      position: absolute;
      inset: 0;
      background: linear-gradient(140deg, transparent 45%, rgba(255, 255, 255, 0.18) 140%);
    }

    svg { position: relative; z-index: 1; }
  }

  &__qa-btn-label {
    font-size: 0.9643em; // 0.84375rem / 0.875rem
    font-weight: 700;
    font-family: $display;
    color: $ink;
  }

  &__qa-btn-desc {
    font-size: 0.8214em; // 0.71875rem / 0.875rem
    color: $ink-3;
    margin-top: 2px;
  }

  // ---- panel section title used for Quick Actions card ------------------------------
  &__section-title {
    display: flex;
    align-items: center;
    gap: 8px;
    font-family: $display;
    font-weight: 700;
    font-size: 1em; // 0.875rem / 0.875rem (base)
    color: $ink;
    margin-bottom: 4px;

    svg { color: $signal; }
  }

  &__section-sub {
    font-size: 0.8571em; // 0.75rem / 0.875rem
    color: $ink-3;
    margin-bottom: 18px;
  }
}

@keyframes db-pulse {
  0%, 100% { box-shadow: 0 0 0 0 rgba(43, 43, 245, 0.5); }
  50% { box-shadow: 0 0 0 4px rgba(43, 43, 245, 0); }
}

@media (max-width: 768px) {
  .dashboard__stats { grid-template-columns: repeat(2, 1fr); }
  .dashboard__grid { grid-template-columns: 1fr; }
  .dashboard__actions { grid-template-columns: 1fr 1fr; }
  .dashboard__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .dashboard__body { padding: 0 18px 24px; }
}
