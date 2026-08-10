import { useEffect } from 'react';
import { useNavigate } from 'react-router-dom';
import { Loader2, TrendingUp, Play, Plus, GitCompare, BookOpen, ChevronRight, Clock3 } from 'lucide-react';
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
    <div className="page-enter">
      <div className={styles.dash__header}>
        <div>
          <p className={styles['dash__header-eyebrow']}>Overview</p>
          <h1>Dashboard</h1>
          <p className={styles['dash__header-sub']}>Your evaluation activity at a glance</p>
        </div>
        <div className={styles['dash__header-meta']}>
          <Clock3 size={13} />
          {runs.length} evaluation{runs.length === 1 ? '' : 's'} tracked
        </div>
      </div>
      <div className="pg-body">
        <div className={styles['d-stats']}>
          {stats.map((s, i) => (
            <div className={styles['d-stat']} key={i}>
              <div className={styles['d-stat-top']}>
                <div>
                  <div className={styles['d-stat-label']}>{s.label}</div>
                  <div className={styles['d-stat-val']}>{s.value}</div>
                  <div className={styles['d-stat-change']}><TrendingUp size={12} /> {s.change}</div>
                </div>
                <Sparkline data={s.spark} color="#6366F1" width={72} height={32} />
              </div>
            </div>
          ))}
        </div>

        <div className={styles['dash__grid']}>
          <div className="card" style={{ padding: 0 }}>
            <div className={styles['dash__panel-hdr']}>
              <span>Recent Evaluations</span>
              <button className="btn btn-ghost btn-sm" onClick={() => navigate('/app/evaluations')}>View All <ChevronRight size={14} /></button>
            </div>
            {runs.length === 0 && (
              <div className={styles['dash__empty']}>No evaluations yet — launch one from Quick Actions below.</div>
            )}
            {runs.slice(0, 4).map((run) => (
              <div key={run.id} className={styles['dash__run-row']} onClick={() => navigate(`/app/evaluations/${run.id}`)}>
                <div style={{ display: 'flex', alignItems: 'center', gap: 14 }}>
                  {run.status === 'completed' ? (
                    <ScoreRing score={Math.round((run.progress / (run.total || 1)) * 100)} size={44} stroke={4} color="#6366F1" />
                  ) : (
                    <div className={styles['dash__spinner']}><Loader2 size={18} color="#6366F1" style={{ animation: 'spin 1.5s linear infinite' }} /></div>
                  )}
                  <div>
                    <div style={{ fontWeight: 600, fontSize: 14 }}>{run.name}</div>
                    <div style={{ fontSize: 12, color: '#6B7280' }}>{run.benchmark || '—'} &middot; {new Date(run.created_at).toLocaleDateString()}</div>
                  </div>
                </div>
                <span className={`badge ${run.status === 'completed' ? 'badge-green' : 'badge-run'}`}>{run.status}</span>
              </div>
            ))}
          </div>

          <div className="card">
            <div style={{ fontFamily: "'JetBrains Mono', monospace", fontWeight: 700, fontSize: 15, marginBottom: 4 }}>Top Models</div>
            <div style={{ fontSize: 13, color: '#6B7280', marginBottom: 12 }}>Strength comparison across 5 dimensions</div>
            {radarModels.length > 0 ? (
              <>
                <div className="radar-wrap"><RadarChart models={radarModels} size={260} /></div>
                <div className={styles['dash__legend']}>
                  {radarModels.map((m, i) => (
                    <span key={i}><span className={styles['dash__dot']} style={{ background: ['#6366F1', '#F59E0B', '#10B981'][i] }} /> {m.name}</span>
                  ))}
                </div>
              </>
            ) : (
              <div className={styles['dash__empty']}>Connect a provider to see model comparisons.</div>
            )}
          </div>
        </div>

        <div className="card" style={{ padding: 24 }}>
          <div style={{ fontFamily: "'JetBrains Mono', monospace", fontWeight: 700, fontSize: 15, marginBottom: 20 }}>Quick Actions</div>
          <div className={styles['dash__actions']}>
            {[
              { icon: <Play size={20} />, label: 'New Evaluation', desc: 'Start a benchmark run', to: '/app/new-eval', cls: 'ind' },
              { icon: <Plus size={20} />, label: 'Add Provider', desc: 'Connect an API', to: '/app/providers', cls: 'em' },
              { icon: <GitCompare size={20} />, label: 'Compare Models', desc: 'Side-by-side analysis', to: '/app/comparison', cls: 'amb' },
              { icon: <BookOpen size={20} />, label: 'Browse Suites', desc: 'Explore benchmarks', to: '/app/suites', cls: 'sky' },
            ].map((a, i) => (
              <button key={i} onClick={() => navigate(a.to)} className={`${styles['qa-btn']} ${styles[`qa-btn--${a.cls}`]}`}>
                <div className={styles['qa-btn__icon']}>{a.icon}</div>
                <div><div className={styles['qa-btn__label']}>{a.label}</div><div className={styles['qa-btn__desc']}>{a.desc}</div></div>
              </button>
            ))}
          </div>
        </div>
      </div>
    </div>
  );
}
















@use '../../styles/_variables' as *;

.dash {
  &__header {
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding-bottom: 18px;
    margin-bottom: 24px;
    border-bottom: 1px solid $border-light;

    h1 {
      font-family: $font-display;
      font-size: 1.5rem;
      font-weight: 800;
      letter-spacing: -0.02em;
      color: $text-primary;
      line-height: 1.2;
    }
  }

  &__header-eyebrow {
    display: flex;
    align-items: center;
    gap: 8px;
    font-family: $font-mono;
    font-size: 0.6875rem;
    font-weight: 700;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    color: $indigo;
    margin-bottom: 6px;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $indigo;
    }
  }

  &__header-sub {
    margin-top: 4px;
    font-size: 0.875rem;
    color: $text-secondary;
  }

  &__header-meta {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-secondary;
    background: $surface-alt;
    border: 1px solid $border-light;
    border-radius: 999px;
    padding: 7px 13px;
    white-space: nowrap;
    margin-bottom: 3px;
  }
}

.d-stats { display: grid; grid-template-columns: repeat(3, 1fr); gap: 16px; margin-bottom: 24px; }
.d-stat {
  background: $surface; border: 1px solid $border; border-radius: 16px; padding: 22px 24px; transition: all .2s;
}
.d-stat:hover { box-shadow: $shadow-2; transform: translateY(-2px); }
.d-stat-top { display: flex; justify-content: space-between; align-items: flex-start; }
.d-stat-label { font-size: 12px; color: $text-secondary; font-weight: 600; text-transform: uppercase; letter-spacing: .5px; }
.d-stat-val { font-family: $font-mono; font-size: 34px; font-weight: 700; letter-spacing: -1px; line-height: 1; margin-top: 8px; }
.d-stat-change { font-size: 12px; color: $emerald; font-weight: 600; margin-top: 6px; display: flex; align-items: center; gap: 4px; }

.dash {
  &__grid { display: grid; grid-template-columns: 1.2fr 1fr; gap: 16px; margin-bottom: 24px; }
  &__panel-hdr {
    padding: 20px 24px; border-bottom: 1px solid $border-light; display: flex; justify-content: space-between;
    align-items: center; font-family: $font-display; font-weight: 700; font-size: 15px;
  }
  &__run-row {
    padding: 16px 24px; border-bottom: 1px solid $border-light; display: flex; justify-content: space-between;
    align-items: center; transition: background .15s; cursor: pointer;
  }
  &__run-row:hover { background: $surface-hover; }
  &__spinner {
    width: 44px; height: 44px; border-radius: 50%; background: $indigo-pale;
    display: flex; align-items: center; justify-content: center;
  }
  &__empty { padding: 32px 24px; color: $text-secondary; font-size: 13px; }
  &__legend { display: flex; justify-content: center; gap: 20px; margin-top: 4px; flex-wrap: wrap; }
  &__legend span { display: flex; align-items: center; gap: 6px; font-size: 12px; font-weight: 600; color: $text-secondary; }
  &__dot { width: 8px; height: 8px; border-radius: 50%; display: inline-block; }
  &__actions { display: grid; grid-template-columns: repeat(4, 1fr); gap: 14px; }
}

// Note: .radar-wrap moved to src/styles/global.scss — it's shared with Comparison.

.qa-btn {
  display: flex; flex-direction: column; align-items: center; gap: 12px; padding: 24px 20px;
  border: 1px solid $border; border-radius: 16px; background: $surface; cursor: pointer; transition: all .25s;
  text-align: center;
}
.qa-btn:hover { transform: translateY(-4px); box-shadow: $shadow-3; border-color: transparent; }
.qa-btn__icon { width: 48px; height: 48px; border-radius: 14px; display: flex; align-items: center; justify-content: center; }
.qa-btn--ind .qa-btn__icon { background: $indigo-pale; color: $indigo; }
.qa-btn--em .qa-btn__icon { background: $emerald-pale; color: $emerald; }
.qa-btn--amb .qa-btn__icon { background: $amber-pale; color: $amber; }
.qa-btn--sky .qa-btn__icon { background: $sky-pale; color: $sky; }
.qa-btn__label { font-size: 14px; font-weight: 700; font-family: $font-display; }
.qa-btn__desc { font-size: 12px; color: $text-secondary; margin-top: 2px; }

@media (max-width: 768px) {
  .d-stats { grid-template-columns: repeat(2, 1fr); }
  .dash__grid { grid-template-columns: 1fr; }
  .dash__actions { grid-template-columns: 1fr 1fr; }
}
