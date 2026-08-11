import { useEffect } from 'react';
import { useNavigate } from 'react-router-dom';
import {
  Loader2,
  TrendingUp,
  Play,
  Plus,
  GitCompare,
  BookOpen,
  ChevronRight,
  Clock3,
  AlertCircle,
  Radar,
  ListChecks,
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchProviders } from '../../store/slices/providersSlice';
import { fetchModels } from '../../store/slices/modelsSlice';
import { fetchEvaluations } from '../../store/slices/evaluationsSlice';
import ScoreRing from '../common/ScoreRing';
import Sparkline from '../common/Sparkline';
import RadarChart from '../common/RadarChart';
import { useCounter } from '../common/useCounter';
import styles from './Dashboard.module.scss';

// Signal accent + status colors, matching the evaluation Run Console.
const SIGNAL = '#2B2BF5';
const LEGEND_COLORS = ['#2B2BF5', '#E08600', '#0FA968'];

export default function Dashboard() {
  const dispatch = useAppDispatch();
  const navigate = useNavigate();

  const providers = useAppSelector((s) => s.providers.items);
  const models = useAppSelector((s) => s.models.items);
  const runs = useAppSelector((s) => s.evaluations.list);

  useEffect(() => {
    dispatch(fetchProviders());
    dispatch(fetchModels());
    dispatch(fetchEvaluations());
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
    <div className="page-enter pg-shell">
      <div className={styles.db__header}>
        <div>
          <p className={styles.db__eyebrow}>Overview</p>
          <h1>Dashboard</h1>
          <p className={styles.db__sub}>Your evaluation activity at a glance</p>
        </div>
        <div className={styles.db__meta}>
          <Clock3 size={13} />
          {runs.length} evaluation{runs.length === 1 ? '' : 's'} tracked
        </div>
      </div>

      <div className={styles['db-body']}>
        <div className={styles['d-stats']}>
          {stats.map((s, i) => (
            <div className={styles['d-stat']} key={i}>
              <div className={styles['d-stat-top']}>
                <div>
                  <div className={styles['d-stat-label']}>{s.label}</div>
                  <div className={styles['d-stat-val']}>{s.value}</div>
                  <div className={styles['d-stat-change']}>
                    <TrendingUp size={12} /> {s.change}
                  </div>
                </div>
                <Sparkline data={s.spark} color={SIGNAL} width={72} height={32} />
              </div>
            </div>
          ))}
        </div>

        <div className={styles['dash__grid']}>
          <div className={styles['dash__panel']}>
            <div className={styles['dash__panel-hdr']}>
              <span className={styles['dash__panel-title']}>
                <ListChecks size={15} /> Recent Evaluations
              </span>
              <button className={styles['dash__link']} onClick={() => navigate('/app/history')}>
                View All <ChevronRight size={14} />
              </button>
            </div>
            {runs.length === 0 && (
              <div className={styles['dash__empty']}>No evaluations yet — launch one from Quick Actions below.</div>
            )}
            {runs.slice(0, 4).map((run) => (
              <div key={run.id} className={styles['dash__run-row']} onClick={() => navigate(`/app/history?id=${run.id}`)}>
                <div className={styles['dash__run-main']}>
                  {run.status === 'completed' ? (
                    <ScoreRing score={Math.round(run.top_score ?? 0)} size={40} stroke={4} color={SIGNAL} />
                  ) : run.status === 'failed' ? (
                    <div className={styles['dash__fail-icon']}>
                      <AlertCircle size={19} color="#DC2626" />
                    </div>
                  ) : (
                    <div className={styles['dash__spinner']}>
                      <Loader2 size={17} color={SIGNAL} style={{ animation: 'spin 1.5s linear infinite' }} />
                    </div>
                  )}
                  <div style={{ minWidth: 0 }}>
                    <div className={styles['dash__run-name']}>{run.name}</div>
                    <div className={styles['dash__run-meta']}>
                      {run.benchmark || '—'} &middot; {new Date(run.created_at).toLocaleDateString()}
                    </div>
                  </div>
                </div>
                <span className={`${styles['status-pill']} ${styles[`status-pill--${run.status === 'completed' ? 'completed' : run.status === 'failed' ? 'failed' : 'running'}`]}`}>
                  {run.status}
                </span>
              </div>
            ))}
          </div>

          <div className={styles['dash__panel']} style={{ padding: 22 }}>
            <div className={styles['section-title']}>
              <Radar size={15} /> Top Models
            </div>
            <div className={styles['section-sub']}>Strength comparison across 5 dimensions</div>
            {radarModels.length > 0 ? (
              <>
                <div className="radar-wrap">
                  <RadarChart models={radarModels} size={260} />
                </div>
                <div className={styles['dash__legend']}>
                  {radarModels.map((m, i) => (
                    <span key={i}>
                      <span className={styles['dash__dot']} style={{ background: LEGEND_COLORS[i] }} /> {m.name}
                    </span>
                  ))}
                </div>
              </>
            ) : (
              <div className={styles['dash__empty']}>Connect a provider to see model comparisons.</div>
            )}
          </div>
        </div>

        <div className={styles['dash__panel']} style={{ padding: 22 }}>
          <div className={styles['section-title']}>
            <Play size={15} /> Quick Actions
          </div>
          <div className={styles['section-sub']}>Jump straight into your next step.</div>
          <div className={styles['dash__actions']}>
            {[
              { icon: <Play size={20} />, label: 'New Evaluation', desc: 'Start a benchmark run', to: '/app/run-evaluation', cls: 'ind' },
              { icon: <Plus size={20} />, label: 'Add Provider', desc: 'Connect an API', to: '/app/providers', cls: 'em' },
              { icon: <GitCompare size={20} />, label: 'Compare Models', desc: 'Side-by-side analysis', to: '/app/comparison', cls: 'amb' },
              { icon: <BookOpen size={20} />, label: 'Datasets', desc: 'Browse benchmark suites', to: '/app/datasets', cls: 'sky' },
            ].map((a, i) => (
              <button key={i} onClick={() => navigate(a.to)} className={`${styles['qa-btn']} ${styles[`qa-btn--${a.cls}`]}`}>
                <div className={styles['qa-btn__icon']}>{a.icon}</div>
                <div>
                  <div className={styles['qa-btn__label']}>{a.label}</div>
                  <div className={styles['qa-btn__desc']}>{a.desc}</div>
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
// Dashboard — matches the Evaluation Run Console design system:
// ink/paper palette, ultramarine signal accent, mono instrument labels,
// hover-lift cards with a subtle inset ring on emphasis.
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

%micro {
  font-family: $mono;
  font-size: 0.6875rem;
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

// ===========================================================================
// Header
// ===========================================================================
.db {
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
      font-size: 1.5rem;
      font-weight: 800;
      letter-spacing: -0.02em;
      color: $ink;
      line-height: 1.2;
    }
  }

  &__eyebrow {
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

  &__sub {
    margin-top: 4px;
    font-size: 0.84375rem;
    color: $ink-2;
  }

  &__meta {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 7px;
    padding: 7px 13px;
    border-radius: 999px;
    border: 1px solid $line;
    background: $paper;
    font-family: $mono;
    font-size: 0.71875rem;
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ink-2;
    white-space: nowrap;
    margin-bottom: 3px;
  }
}

// ===========================================================================
// Body
// ===========================================================================
.db-body {
  padding: 0 32px 32px;
}

// ---- stat cards -----------------------------------------------------------
.d-stats {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 14px;
  margin-bottom: 18px;
}

.d-stat {
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

.d-stat-top {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
}

.d-stat-label {
  @extend %micro;
  font-size: 0.625rem;
  color: $ink-3;
}

.d-stat-val {
  font-family: $mono;
  font-size: 2rem;
  font-weight: 700;
  letter-spacing: -0.02em;
  line-height: 1;
  margin-top: 10px;
  color: $ink;
}

.d-stat-change {
  font-size: 0.71875rem;
  color: $ok;
  font-weight: 600;
  margin-top: 8px;
  display: flex;
  align-items: center;
  gap: 5px;
}

// ---- main grid --------------------------------------------------------------
.dash {
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
    font-size: 0.875rem;
    color: $ink;

    svg { color: $signal; }
  }

  &__panel-sub {
    font-size: 0.75rem;
    color: $ink-3;
    margin-top: 2px;
  }

  &__link {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    font-family: $sans;
    font-size: 0.78125rem;
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
    font-size: 0.84375rem;
    color: $ink;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__run-meta {
    font-size: 0.71875rem;
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
    font-size: 0.8125rem;
  }

  &__legend {
    display: flex;
    justify-content: center;
    gap: 18px;
    margin-top: 6px;
    flex-wrap: wrap;
  }

  &__legend span {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 0.71875rem;
    font-weight: 600;
    color: $ink-2;
  }

  &__dot {
    width: 8px;
    height: 8px;
    border-radius: 50%;
    display: inline-block;
  }

  &__actions {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 12px;
  }
}

// ---- status pill (mono, colored per run status) ----------------------------
.status-pill {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 4px 10px 4px 8px;
  border-radius: 999px;
  font-family: $mono;
  font-size: 0.625rem;
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

// ---- quick action tiles -----------------------------------------------------
.qa-btn {
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
}

.qa-btn__icon {
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

.qa-btn--ind .qa-btn__icon { background: $signal; }
.qa-btn--em  .qa-btn__icon { background: $ok; }
.qa-btn--amb .qa-btn__icon { background: $amber; }
.qa-btn--sky .qa-btn__icon { background: #0369A1; }

.qa-btn__label {
  font-size: 0.84375rem;
  font-weight: 700;
  font-family: $display;
  color: $ink;
}

.qa-btn__desc {
  font-size: 0.71875rem;
  color: $ink-3;
  margin-top: 2px;
}

// ---- panel section title used for Quick Actions card -----------------------
.section-title {
  display: flex;
  align-items: center;
  gap: 8px;
  font-family: $display;
  font-weight: 700;
  font-size: 0.875rem;
  color: $ink;
  margin-bottom: 4px;

  svg { color: $signal; }
}

.section-sub {
  font-size: 0.75rem;
  color: $ink-3;
  margin-bottom: 18px;
}

@keyframes db-pulse {
  0%, 100% { box-shadow: 0 0 0 0 rgba(43, 43, 245, 0.5); }
  50% { box-shadow: 0 0 0 4px rgba(43, 43, 245, 0); }
}

@media (max-width: 768px) {
  .d-stats { grid-template-columns: repeat(2, 1fr); }
  .dash__grid { grid-template-columns: 1fr; }
  .dash__actions { grid-template-columns: 1fr 1fr; }
  .db__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .db-body { padding: 0 18px 24px; }
}
