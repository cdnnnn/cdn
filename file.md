import { useEffect, useMemo, useState, type FC } from 'react';
import { useNavigate } from 'react-router-dom';
import {
  Play,
  PlugZap,
  Upload,
  Search,
  ChevronRight,
  Trophy,
  Cpu,
  Database,
  Gauge,
  CheckCircle2,
  Sparkles,
  Loader2,
  AlertCircle,
  RefreshCw,
  BarChart3,
  PieChart as PieChartIcon,
} from 'lucide-react';
import {
  ResponsiveContainer,
  BarChart,
  Bar,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  PieChart,
  Pie,
  Cell,
} from 'recharts';
import { fetchDashboard } from './api';
import type { DashboardResponse, RecentEval } from './types';
import './Dashboard.scss';

const QUICK_ACTIONS = [
  { icon: Play, label: 'Run Evaluation', desc: 'Test models on benchmarks', to: '/app/run-evaluation', tint: 'blue' },
  { icon: PlugZap, label: 'Add Provider', desc: 'Connect API keys', to: '/app/providers', tint: 'green' },
  { icon: Upload, label: 'Upload Dataset', desc: 'Custom test questions', to: '/app/datasets', tint: 'amber' },
  { icon: Search, label: 'Browse Models', desc: 'Explore 100+ models', to: '/app/models', tint: 'violet' },
] as const;

// Maps the icon keywords the API sends to actual Lucide components.
const STAT_ICON_MAP: Record<string, typeof Cpu> = {
  microchip: Cpu,
  database: Database,
  gauge: Gauge,
  check: CheckCircle2,
};

function statIcon(icon: string) {
  return STAT_ICON_MAP[icon] ?? Sparkles;
}

function statTint(index: number): 'blue' | 'amber' | 'violet' | 'green' {
  const tints = ['blue', 'amber', 'violet', 'green'] as const;
  return tints[index % tints.length];
}

function statusTint(status: string): 'violet' | 'blue' | 'amber' | 'green' | 'danger' {
  switch (status) {
    case 'completed':
      return 'green';
    case 'running':
      return 'blue';
    case 'pending':
      return 'amber';
    case 'failed':
    case 'canceled':
      return 'danger';
    default:
      return 'violet';
  }
}

function statusLabel(status: string): string {
  switch (status) {
    case 'completed':
      return 'Completed';
    case 'running':
      return 'Running';
    case 'pending':
      return 'Pending';
    case 'failed':
      return 'Failed';
    case 'canceled':
      return 'Canceled';
    default:
      return status;
  }
}

function formatDate(dateStr: string): string {
  const d = new Date(dateStr);
  if (Number.isNaN(d.getTime())) return dateStr;
  return d.toLocaleDateString(undefined, { year: 'numeric', month: 'short', day: 'numeric' });
}

// NOTE: these are plain hex values chosen to visually match the app's
// $primary / $success / $warning / $danger / violet tokens used elsewhere.
// SCSS variables aren't available to inline JS/SVG styling, so if your
// actual design tokens differ, adjust CHART_COLORS to match.
const CHART_COLORS = {
  primary: '#4f46e5',
  primaryLight: '#eef2ff',
  success: '#16a34a',
  successLight: '#dcfce7',
  warning: '#d97706',
  warningLight: '#fef3c7',
  danger: '#dc2626',
  dangerLight: '#fee2e2',
  violet: '#7c3aed',
  violetLight: '#f3e8ff',
  textTertiary: '#94a3b8',
  border: '#e5e7eb',
};

const STATUS_COLORS: Record<string, string> = {
  completed: CHART_COLORS.success,
  running: CHART_COLORS.primary,
  pending: CHART_COLORS.warning,
  failed: CHART_COLORS.danger,
  canceled: CHART_COLORS.danger,
};

interface BarTooltipProps {
  active?: boolean;
  payload?: { value: number; payload: { name: string; desc: string } }[];
}

const MetricBarTooltip: FC<BarTooltipProps> = ({ active, payload }) => {
  if (!active || !payload || payload.length === 0) return null;
  const { name, desc } = payload[0].payload;
  return (
    <div className="dash__chart-tooltip">
      <p className="dash__chart-tooltip-title">{name}</p>
      <p className="dash__chart-tooltip-value">{payload[0].value.toFixed(2)}</p>
      {desc && <p className="dash__chart-tooltip-desc">{desc}</p>}
    </div>
  );
};

interface PieTooltipProps {
  active?: boolean;
  payload?: { value: number; name: string; payload: { fill: string } }[];
}

const StatusPieTooltip: FC<PieTooltipProps> = ({ active, payload }) => {
  if (!active || !payload || payload.length === 0) return null;
  const entry = payload[0];
  return (
    <div className="dash__chart-tooltip">
      <p className="dash__chart-tooltip-title">
        <span className="dash__chart-tooltip-swatch" style={{ background: entry.payload.fill }} />
        {entry.name}
      </p>
      <p className="dash__chart-tooltip-value">{entry.value}</p>
    </div>
  );
};

const Dashboard: FC = () => {
  const navigate = useNavigate();

  const [data, setData] = useState<DashboardResponse | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const load = () => {
    setLoading(true);
    setError(null);
    fetchDashboard()
      .then(setData)
      .catch((err) => {
        setError(err instanceof Error ? err.message : 'Failed to load the dashboard.');
      })
      .finally(() => setLoading(false));
  };

  useEffect(() => {
    load();
  }, []);

  // Prefer the "Best Score" stat's top model if present, otherwise fall
  // back to whichever metric card has one — used for the hero highlight.
  const heroHighlight = useMemo(() => {
    if (!data) return null;
    const bestScoreStat = data.overall_data.find((s) => s.title.toLowerCase().includes('score') && s.top_model);
    if (bestScoreStat?.top_model) return { model: bestScoreStat.top_model, score: bestScoreStat.count };
    const withTopModel = data.metrics_overview_cards.find((m) => m.top_model);
    if (withTopModel?.top_model) return { model: withTopModel.top_model, score: withTopModel.best_avg_score };
    return null;
  }, [data]);

  const metricChartData = useMemo(
    () =>
      (data?.metrics_overview_cards ?? []).map((m) => ({
        name: m.metric_name,
        desc: m.metric_desc,
        score: m.best_avg_score,
      })),
    [data]
  );

  const statusChartData = useMemo(() => {
    if (!data) return [];
    const counts = new Map<string, number>();
    data.recent_evals.forEach((ev) => {
      counts.set(ev.eval_status, (counts.get(ev.eval_status) ?? 0) + 1);
    });
    return Array.from(counts.entries()).map(([status, count]) => ({
      name: statusLabel(status),
      value: count,
      fill: STATUS_COLORS[status] ?? CHART_COLORS.violet,
    }));
  }, [data]);

  const totalRecentEvals = data?.recent_evals.length ?? 0;

  return (
    <div className="dash">
      {/* ---------- hero ---------- */}
      <section className="dash__hero">
        <div className="dash__hero-grid" aria-hidden="true" />
        <div className="dash__hero-content">
          <p className="dash__hero-eyebrow">Evaluation overview</p>
          <h1 className="dash__hero-title">Compare models with confidence</h1>
          <p className="dash__hero-sub">Run standardized tests across providers and let the results guide your next decision.</p>

          {heroHighlight && (
            <div className="dash__hero-highlight">
              <Trophy size={14} />
              Top performer: <strong>{heroHighlight.model}</strong>
              <span className="dash__hero-highlight-score n">{heroHighlight.score}</span>
            </div>
          )}
        </div>
        <button type="button" className="dash__hero-btn" onClick={() => navigate('/app/run-evaluation')}>
          <Play size={15} strokeWidth={2.25} />
          New Evaluation
        </button>
      </section>

      {loading && (
        <section className="dash__card dash__loading">
          <Loader2 size={20} className="dash__spin" />
          <p>Loading dashboard…</p>
        </section>
      )}

      {!loading && error && (
        <section className="dash__card dash__loading">
          <AlertCircle size={20} />
          <p>{error}</p>
          <button type="button" className="dash__btn dash__btn--sm" onClick={load}>
            <RefreshCw size={13} /> Try again
          </button>
        </section>
      )}

      {!loading && !error && data && (
        <>
          {/* ---------- stats ---------- */}
          <section className="dash__stats">
            {data.overall_data.map((s, i) => {
              const Icon = statIcon(s.icon);
              return (
                <div className="dash__stat-card" key={s.id}>
                  <span className={`dash__stat-icon dash__stat-icon--${statTint(i)}`}>
                    <Icon size={16} strokeWidth={2} />
                  </span>
                  <div>
                    <span className="dash__stat-value n">{s.count}</span>
                    <span className="dash__stat-label">{s.title}</span>
                    {s.top_model && <span className="dash__stat-sub">Top: {s.top_model}</span>}
                  </div>
                </div>
              );
            })}
          </section>

          {/* ---------- charts row ---------- */}
          <section className="dash__charts-row">
            <div className="dash__card dash__chart-card">
              <div className="dash__card-head">
                <div>
                  <h2 className="dash__card-title">Metrics Performance</h2>
                  <p className="dash__card-subtitle">Best average score per metric</p>
                </div>
                <span className="dash__badge">
                  <BarChart3 size={12} /> {metricChartData.length}
                </span>
              </div>

              {metricChartData.length === 0 ? (
                <p className="dash__empty">No metrics data yet.</p>
              ) : (
                <div className="dash__chart-wrap">
                  <ResponsiveContainer width="100%" height={260}>
                    <BarChart data={metricChartData} margin={{ top: 8, right: 8, left: -12, bottom: 0 }} barCategoryGap="28%">
                      <defs>
                        <linearGradient id="dashBarGradient" x1="0" y1="0" x2="0" y2="1">
                          <stop offset="0%" stopColor={CHART_COLORS.primary} stopOpacity={0.95} />
                          <stop offset="100%" stopColor={CHART_COLORS.primary} stopOpacity={0.55} />
                        </linearGradient>
                      </defs>
                      <CartesianGrid vertical={false} stroke={CHART_COLORS.border} strokeDasharray="3 4" />
                      <XAxis
                        dataKey="name"
                        tick={{ fontSize: 11, fill: CHART_COLORS.textTertiary }}
                        axisLine={{ stroke: CHART_COLORS.border }}
                        tickLine={false}
                        interval={0}
                        angle={metricChartData.length > 5 ? -20 : 0}
                        textAnchor={metricChartData.length > 5 ? 'end' : 'middle'}
                        height={metricChartData.length > 5 ? 44 : 24}
                      />
                      <YAxis
                        tick={{ fontSize: 11, fill: CHART_COLORS.textTertiary }}
                        axisLine={false}
                        tickLine={false}
                        width={34}
                      />
                      <Tooltip content={<MetricBarTooltip />} cursor={{ fill: CHART_COLORS.primaryLight }} />
                      <Bar dataKey="score" fill="url(#dashBarGradient)" radius={[8, 8, 0, 0]} maxBarSize={44} isAnimationActive />
                    </BarChart>
                  </ResponsiveContainer>
                </div>
              )}
            </div>

            <div className="dash__card dash__chart-card">
              <div className="dash__card-head">
                <div>
                  <h2 className="dash__card-title">Evaluation Status</h2>
                  <p className="dash__card-subtitle">Breakdown of recent evaluations</p>
                </div>
                <span className="dash__badge">
                  <PieChartIcon size={12} /> {totalRecentEvals}
                </span>
              </div>

              {statusChartData.length === 0 ? (
                <p className="dash__empty">No recent evaluations yet.</p>
              ) : (
                <div className="dash__donut-wrap">
                  <ResponsiveContainer width="100%" height={220}>
                    <PieChart>
                      <Tooltip content={<StatusPieTooltip />} />
                      <Pie
                        data={statusChartData}
                        dataKey="value"
                        nameKey="name"
                        innerRadius={62}
                        outerRadius={88}
                        paddingAngle={3}
                        cornerRadius={6}
                        isAnimationActive
                      >
                        {statusChartData.map((entry) => (
                          <Cell key={entry.name} fill={entry.fill} stroke="none" />
                        ))}
                      </Pie>
                    </PieChart>
                  </ResponsiveContainer>
                  <div className="dash__donut-center">
                    <span className="dash__donut-center-value">{totalRecentEvals}</span>
                    <span className="dash__donut-center-label">Evaluations</span>
                  </div>
                </div>
              )}

              {statusChartData.length > 0 && (
                <div className="dash__donut-legend">
                  {statusChartData.map((entry) => (
                    <span className="dash__donut-legend-item" key={entry.name}>
                      <span className="dash__donut-legend-swatch" style={{ background: entry.fill }} />
                      {entry.name}
                      <span className="dash__donut-legend-count">{entry.value}</span>
                    </span>
                  ))}
                </div>
              )}
            </div>
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

              {data.recent_evals.length === 0 ? (
                <p className="dash__empty">No evaluations yet. Run your first one to see it here.</p>
              ) : (
                <div className="dash__eval-list">
                  {data.recent_evals.slice(0, 5).map((ev: RecentEval) => (
                    <button
                      type="button"
                      key={ev.id}
                      className="dash__eval-item"
                      onClick={() => navigate('/app/history', { state: { evaluationId: ev.id } })}
                    >
                      <div className="dash__eval-info">
                        <span className="dash__eval-name">{ev.eval_name}</span>
                        <span className="dash__eval-meta">
                          <span className="dash__eval-type-badge">{ev.provider}</span>
                          <span className="dash__eval-date">{formatDate(ev.created_at)}</span>
                        </span>
                      </div>
                      <span className={`dash__status-badge dash__status-badge--${statusTint(ev.eval_status)}`}>
                        {statusLabel(ev.eval_status)}
                      </span>
                      <ChevronRight size={16} className="dash__eval-arrow" />
                    </button>
                  ))}
                </div>
              )}
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
                    <span>
                      <span className="dash__qa-title">{qa.label}</span>
                      <span className="dash__qa-desc">{qa.desc}</span>
                    </span>
                  </button>
                ))}
              </div>
            </div>
          </section>

          {/* ---------- metrics overview ---------- */}
          <section className="dash__card">
            <div className="dash__card-head">
              <div>
                <h2 className="dash__card-title">Metrics Overview</h2>
                <p className="dash__card-subtitle">Best average score per metric across all evaluations</p>
              </div>
              <span className="dash__badge">
                <BarChart3 size={12} /> {data.metrics_overview_cards.length} metrics
              </span>
            </div>

            {data.metrics_overview_cards.length === 0 ? (
              <p className="dash__empty">No metrics data yet.</p>
            ) : (
              <div className="dash__metric-grid">
                {data.metrics_overview_cards.map((m) => (
                  <div className="dash__metric-card" key={m.id}>
                    <div className="dash__metric-card-head">
                      <span className="dash__metric-card-name">{m.metric_name}</span>
                      <span className="dash__metric-card-score n">{m.best_avg_score.toFixed(2)}</span>
                    </div>
                    {m.metric_desc && <p className="dash__metric-card-desc">{m.metric_desc}</p>}
                    {m.top_model && (
                      <span className="dash__metric-card-top">
                        <Trophy size={11} strokeWidth={2.5} /> {m.top_model}
                      </span>
                    )}
                  </div>
                ))}
              </div>
            )}
          </section>
        </>
      )}
    </div>
  );
};

export default Dashboard;














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
    align-items: flex-start;
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

  &__hero-eyebrow {
    display: flex;
    align-items: center;
    gap: 8px;
    font-family: $font-mono;
    font-size: 0.6875rem;
    font-weight: 700;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    color: $primary;
    margin-bottom: 8px;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $primary;
    }
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

  &__hero-highlight {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    margin-top: 16px;
    padding: 7px 14px;
    border-radius: 999px;
    background: $success-subtle;
    color: $text-secondary;
    font-size: 0.78125rem;

    svg {
      color: $success;
      flex-shrink: 0;
    }

    strong {
      color: $text-primary;
      font-weight: 700;
    }
  }

  &__hero-highlight-score {
    font-weight: 700;
    color: $success;
  }

  &__hero-btn {
    position: relative;
    flex-shrink: 0;
    margin-top: 2px;
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

  /* ---------- loading / error state ---------- */
  &__spin {
    animation: dash-spin 0.8s linear infinite;
  }

  @keyframes dash-spin {
    to {
      transform: rotate(360deg);
    }
  }

  &__loading {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 10px;
    padding: 44px 20px;
    color: $text-tertiary;
    font-size: 0.84375rem;
    text-align: center;
  }

  &__empty {
    padding: 20px 4px;
    color: $text-tertiary;
    font-size: 0.8125rem;
  }

  /* ---------- charts row ---------- */
  &__charts-row {
    display: grid;
    grid-template-columns: 1.4fr 1fr;
    gap: 14px;
    align-items: stretch;
  }

  &__chart-card {
    display: flex;
    flex-direction: column;
  }

  &__chart-wrap {
    margin-top: 4px;
  }

  // Recharts renders its own SVG text/tooltips outside our component tree,
  // so this custom tooltip is a plain floating card recharts positions for us.
  &__chart-tooltip {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 10px;
    box-shadow: $shadow-md;
    padding: 8px 12px;
    min-width: 120px;
  }

  &__chart-tooltip-title {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 0.75rem;
    font-weight: 700;
    color: $text-primary;
    margin-bottom: 2px;
  }

  &__chart-tooltip-swatch {
    flex-shrink: 0;
    width: 8px;
    height: 8px;
    border-radius: 50%;
  }

  &__chart-tooltip-value {
    font-size: 0.9375rem;
    font-weight: 800;
    color: $primary;
    letter-spacing: -0.01em;
  }

  &__chart-tooltip-desc {
    margin-top: 2px;
    font-size: 0.6875rem;
    color: $text-tertiary;
    max-width: 180px;
    line-height: 1.4;
  }

  /* ---------- donut chart ---------- */
  &__donut-wrap {
    position: relative;
    margin-top: 4px;
  }

  &__donut-center {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    display: flex;
    flex-direction: column;
    align-items: center;
    pointer-events: none;
  }

  &__donut-center-value {
    font-size: 1.5rem;
    font-weight: 800;
    color: $text-primary;
    letter-spacing: -0.02em;
    line-height: 1.1;
  }

  &__donut-center-label {
    font-size: 0.65625rem;
    font-weight: 600;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $text-tertiary;
    margin-top: 2px;
  }

  &__donut-legend {
    display: flex;
    flex-wrap: wrap;
    gap: 8px 16px;
    margin-top: 12px;
    padding-top: 12px;
    border-top: 1px solid $border-subtle;
  }

  &__donut-legend-item {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-secondary;
  }

  &__donut-legend-swatch {
    flex-shrink: 0;
    width: 8px;
    height: 8px;
    border-radius: 2px;
  }

  &__donut-legend-count {
    font-weight: 700;
    color: $text-primary;
  }

  /* ---------- stats ---------- */
  &__stats {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 14px;
  }

  &__stat-card {
    display: flex;
    align-items: flex-start;
    gap: 12px;
    padding: 18px 20px;
    border: 1px solid $border-subtle;
    border-radius: 14px;
    background: $bg-main;
    box-shadow: $shadow-xs;
  }

  &__stat-icon {
    flex-shrink: 0;
    width: 36px;
    height: 36px;
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

  &__stat-value {
    display: block;
    font-size: 26px;
    font-weight: 800;
    letter-spacing: -0.02em;
    color: $text-primary;
    line-height: 1.1;
  }

  &__stat-label {
    display: block;
    font-family: $font-mono;
    font-size: 0.65625rem;
    font-weight: 600;
    letter-spacing: 0.09em;
    text-transform: uppercase;
    color: $text-tertiary;
    margin-top: 4px;
  }

  &__stat-sub {
    display: block;
    font-size: 0.71875rem;
    color: $text-tertiary;
    margin-top: 4px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
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
    display: inline-flex;
    align-items: center;
    gap: 6px;

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

  &__eval-arrow {
    flex-shrink: 0;
    color: $text-tertiary;
    transition: transform 0.14s ease, color 0.14s ease;
  }

  /* status badge on each recent-eval row */
  &__status-badge {
    flex-shrink: 0;
    font-size: 0.65625rem;
    font-weight: 700;
    letter-spacing: 0.02em;
    border-radius: 999px;
    padding: 4px 10px;
    white-space: nowrap;

    &--green {
      color: $success;
      background: $success-subtle;
    }

    &--blue {
      color: $primary;
      background: $primary-light;
    }

    &--amber {
      color: $warning;
      background: $warning-subtle;
    }

    &--violet {
      color: #7c3aed;
      background: #f3e8ff;
    }

    &--danger {
      color: $danger;
      background: $danger-subtle;
    }
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

  /* ---------- metrics overview ---------- */
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

  &__metric-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
    gap: 12px;
  }

  &__metric-card {
    display: flex;
    flex-direction: column;
    gap: 6px;
    padding: 14px 16px;
    border: 1px solid $border-subtle;
    border-radius: 12px;
    background: $bg-subtle;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &:hover {
      border-color: $primary;
      box-shadow: $shadow-xs;
    }
  }

  &__metric-card-head {
    display: flex;
    align-items: baseline;
    justify-content: space-between;
    gap: 8px;
  }

  &__metric-card-name {
    font-size: 0.84375rem;
    font-weight: 700;
    color: $text-primary;
  }

  &__metric-card-score {
    font-size: 1.0625rem;
    font-weight: 800;
    color: $primary;
    letter-spacing: -0.01em;
  }

  &__metric-card-desc {
    font-size: 0.75rem;
    color: $text-tertiary;
    line-height: 1.45;
  }

  &__metric-card-top {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    margin-top: 2px;
    font-size: 0.71875rem;
    font-weight: 600;
    color: $success;
  }

  /* ---------- responsive ---------- */
  @media (max-width: 900px) {
    &__stats {
      grid-template-columns: repeat(2, 1fr);
    }

    &__row {
      grid-template-columns: 1fr;
    }

    &__charts-row {
      grid-template-columns: 1fr;
    }

    &__hero {
      flex-direction: column;
      align-items: flex-start;
    }
  }

  @media (max-width: 560px) {
    &__stats {
      grid-template-columns: 1fr;
    }

    &__metric-grid {
      grid-template-columns: 1fr;
    }
  }
}
