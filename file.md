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
  const runs = useAppSelector((s) => s.evaluations.runs) ?? [];

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
      <div className={`pg-hdr ${styles.dash__header}`}>
        <div>
          <p className={styles['dash__header-eyebrow']}>Overview</p>
          <h1>Dashboard</h1>
          <p style={{ marginBottom: 0 }}>Your evaluation activity at a glance</p>
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
