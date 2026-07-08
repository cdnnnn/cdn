//ResultsPanel.module.scss

@use '../../../styles/tokens' as t;

.db-analytics-results-panel {
  background: t.$surface-2;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  flex: 1;
  min-height: 0;
}

.db-analytics-results-panel__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 53px;
  padding: 0 16px;
  border-bottom: 1px solid t.$border-strong;
  background: t.$surface-1;
  flex-shrink: 0;
}

.db-analytics-results-panel__header-text {
  display: flex;
  flex-direction: column;
  justify-content: center;
  gap: 1px;
  min-width: 0;
}

.db-analytics-results-panel__title {
  font-size: 14px;
  font-weight: 500;
  margin: 0;
  color: t.$text-primary;
  line-height: 1.3;
}

.db-analytics-results-panel__subtitle {
  font-size: 11px;
  color: t.$text-muted;
  margin: 0;
  line-height: 1.3;
}

.db-analytics-results-panel__body {
  flex: 1;
  min-height: 0;
  overflow-y: auto;
}

.db-analytics-results-panel__error {
  margin: 12px;
  font-size: 12px;
  color: #791f1f;
  background: #fbe9e9;
  padding: 8px 11px;
  border-radius: t.$radius;
}

.db-analytics-chart-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 12px;
  padding: 12px;
}

.db-analytics-chart-card {
  border: 1px solid t.$border-strong;
  border-radius: t.$radius-lg;
  padding: 14px 16px 16px;
  min-width: 0;
  background: t.$surface-2;
  box-shadow: 0 1px 3px rgba(11, 11, 11, 0.05);
  transition: border-color 0.18s ease, box-shadow 0.18s ease, transform 0.18s ease;

  &:hover {
    border-color: t.$border-stronger;
    box-shadow: 0 6px 18px rgba(11, 11, 11, 0.08);
    transform: translateY(-1px);
  }
}

.db-analytics-chart-card--span-2 {
  grid-column: 1 / -1;
}

.db-analytics-chart-card__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 14px;
  padding-bottom: 11px;
  border-bottom: 0.5px solid t.$border;
}

.db-analytics-chart-card__heading {
  display: flex;
  align-items: center;
  gap: 9px;
  min-width: 0;
}

.db-analytics-chart-card__icon {
  width: 28px;
  height: 28px;
  border-radius: 8px;
  background: t.$gradient-primary;
  color: #fff;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  box-shadow: 0 2px 6px rgba(79, 70, 229, 0.25);
}

.db-analytics-chart-card__title {
  font-size: 13.5px;
  font-weight: 600;
  color: t.$text-primary;
  margin: 0;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.db-analytics-chart-card__badge {
  font-size: 10px;
  font-weight: 500;
  color: t.$text-secondary;
  background: t.$surface-0;
  border: 0.5px solid t.$border;
  padding: 2px 7px;
  border-radius: 999px;
  text-transform: capitalize;
  flex-shrink: 0;
}

.db-analytics-chart-legend {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  row-gap: 8px;
  margin-top: 12px;
  padding-top: 12px;
  border-top: 0.5px solid t.$border;
}

.db-analytics-chart-legend__divider {
  width: 1px;
  align-self: stretch;
  min-height: 16px;
  background: t.$border-strong;
  margin: 0 12px;
  flex-shrink: 0;
}

.db-analytics-chart-legend__item {
  display: flex;
  align-items: center;
  gap: 7px;
  font-size: 12.5px;
  color: t.$text-secondary;
  white-space: nowrap;
}

.db-analytics-chart-legend__swatch {
  width: 9px;
  height: 9px;
  border-radius: 3px;
  flex-shrink: 0;
}

.db-analytics-chart-legend__name {
  max-width: 140px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  color: t.$text-primary;
  font-weight: 500;
}

.db-analytics-chart-legend__pct {
  font-weight: 700;
  color: t.$text-primary;
  font-variant-numeric: tabular-nums;
  flex-shrink: 0;
}

.db-analytics-empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 8px;
  padding: 48px 16px;
  color: t.$text-muted;
  text-align: center;
  min-height: 100%;
  box-sizing: border-box;

  p {
    font-size: 13px;
    margin: 0;
    max-width: 220px;
  }
}

.db-analytics-empty-state__icon--generating {
  color: t.$accent;
  animation: db-analytics-empty-state-draw 1.6s ease-in-out infinite;
}

@keyframes db-analytics-empty-state-draw {
  0% {
    transform: scale(0.9) rotate(-4deg);
    opacity: 0.55;
  }
  50% {
    transform: scale(1.08) rotate(3deg);
    opacity: 1;
  }
  100% {
    transform: scale(0.9) rotate(-4deg);
    opacity: 0.55;
  }
}

.db-analytics-kpi-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(140px, 1fr));
  gap: 12px;
}

.db-analytics-kpi-card {
  position: relative;
  display: flex;
  flex-direction: column;
  gap: 6px;
  padding: 16px 16px 14px;
  border-radius: t.$radius-lg;
  background: t.$surface-0;
  border: 1px solid t.$border;
  overflow: hidden;
  transition: transform 0.15s ease, box-shadow 0.15s ease, border-color 0.15s ease;

  &:hover {
    transform: translateY(-2px);
    box-shadow: 0 6px 16px rgba(11, 11, 11, 0.07);
    border-color: t.$border-strong;
  }
}

.db-analytics-kpi-card__label {
  font-size: 11px;
  font-weight: 600;
  letter-spacing: 0.01em;
  text-transform: uppercase;
  color: t.$text-secondary;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.db-analytics-kpi-card__value {
  font-size: 25px;
  font-weight: 700;
  letter-spacing: -0.01em;
  line-height: 1.15;
  color: t.$text-primary;
}

.db-analytics-table-wrap {
  overflow-x: auto;
  max-height: 260px;
  overflow-y: auto;
}

.db-analytics-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 12px;

  th {
    text-align: left;
    font-weight: 500;
    color: t.$text-secondary;
    padding: 6px 10px;
    border-bottom: 1px solid t.$border-strong;
    background: t.$surface-1;
    position: sticky;
    top: 0;
    white-space: nowrap;
  }

  td {
    padding: 6px 10px;
    border-bottom: 0.5px solid t.$border;
    color: t.$text-primary;
    white-space: nowrap;
  }

  tr:last-child td {
    border-bottom: none;
  }
}

.db-analytics-chart-card__no-data {
  padding: 24px 12px;
  text-align: center;
  font-size: 12px;
  color: t.$text-muted;
}

















// ResultsPanel.tsx

import React from 'react';
import {
  LineChart,
  Line,
  BarChart,
  Bar,
  AreaChart,
  Area,
  ScatterChart,
  Scatter,
  PieChart,
  Pie,
  Cell,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  ResponsiveContainer,
} from 'recharts';
import styles from './ResultsPanel.module.scss';
import { ChartResult, ChartType } from '../types';
import { Icon, IconComponent } from '../icons';
import ScrollableChart from './ScrollableChart';

interface Props {
  charts: ChartResult[];
  loadError?: string | null;
  isGenerating?: boolean;
}

const chartTypeIcon: Partial<Record<ChartType, IconComponent>> = {
  kpi: Icon.gauge,
  line: Icon.chartLine,
  bar: Icon.chartBar,
  pie: Icon.chartPie,
  area: Icon.chartArea,
  scatter: Icon.chartDots,
  table: Icon.table,
};

const FALLBACK_ICON: IconComponent = Icon.chartAreaLine;

const getChartIcon = (type: ChartResult['chart_type']): IconComponent =>
  chartTypeIcon[type] ?? FALLBACK_ICON;

// A richer, more vivid categorical palette than the previous muted set —
// each series/slice gets a distinct, saturated hue that still reads well
// against the light surface.
const SERIES_COLORS = [
  '#4f46e5', // indigo
  '#0aa36f', // green
  '#e08a00', // amber
  '#dc4646', // red
  '#0c8fce', // sky blue
  '#a238c9', // violet
  '#d1385c', // rose
];

const tooltipStyle = {
  background: '#ffffff',
  border: '1px solid rgba(11,11,11,0.08)',
  borderRadius: 10,
  fontSize: 12,
  boxShadow: '0 6px 20px rgba(11,11,11,0.10)',
  padding: '8px 12px',
};

const formatKpiValue = (value: number) => {
  if (Math.abs(value) >= 1_000_000) return `${(value / 1_000_000).toFixed(1)}M`;
  if (Math.abs(value) >= 1000) return value.toLocaleString();
  if (!Number.isInteger(value)) return value.toFixed(1);
  return String(value);
};

const renderKpi = (chart: ChartResult) => (
  <div className={styles['db-analytics-kpi-grid']}>
    {chart.data.map((row, i) => {
      const label = String(row[chart.x_axis ?? 'metric']);
      const value = Number(row[chart.y_axis ?? 'value']);
      return (
        <div key={i} className={styles['db-analytics-kpi-card']}>
          <span className={styles['db-analytics-kpi-card__label']}>{label}</span>
          <span className={styles['db-analytics-kpi-card__value']}>{formatKpiValue(value)}</span>
        </div>
      );
    })}
  </div>
);

// When the API doesn't provide an explicit `series` list (it can be null),
// fall back to every numeric column in the data besides the x-axis key.
const deriveSeriesKeys = (chart: ChartResult, xKey: string): string[] => {
  if (chart.series && chart.series.length > 0) return chart.series;
  if (!chart.data || chart.data.length === 0) return [];

  const sample = chart.data[0];
  return Object.keys(sample).filter((key) => key !== xKey && typeof sample[key] === 'number');
};

const axisTick = { fontSize: 11.5, fill: '#8b8a84', fontWeight: 500 };
const axisLine = { stroke: '#d8d7cf' };

// Charts stay readable at any card width, but cramming many categories into
// a fixed card causes label collisions and squeezed bars/points. Instead of
// shrinking further, give the plot a sensible minimum width per category and
// let the card scroll horizontally once the natural width exceeds it.
const PX_PER_CATEGORY = 56;
const MIN_PLOT_WIDTH = 280;

const getPlotWidth = (pointCount: number) => Math.max(MIN_PLOT_WIDTH, pointCount * PX_PER_CATEGORY);

const renderChart = (chart: ChartResult) => {
  const xKey = chart.x_axis ?? 'name';
  const seriesKeys = deriveSeriesKeys(chart, xKey);
  const gradientPrefix = `db-analytics-grad-${chart.id}`;

  if (!chart.data || chart.data.length === 0) {
    return <div className={styles['db-analytics-chart-card__no-data']}>No data returned for this chart.</div>;
  }

  if (chart.chart_type === 'kpi') {
    return renderKpi(chart);
  }

  if (chart.chart_type === 'line') {
    if (seriesKeys.length === 0) {
      return (
        <div className={styles['db-analytics-chart-card__no-data']}>
          No numeric series found to plot for this chart.
        </div>
      );
    }
    return (
      <ScrollableChart minWidth={getPlotWidth(chart.data.length)} height={220}>
        <ResponsiveContainer width="100%" height="100%">
          <LineChart data={chart.data} margin={{ top: 8, right: 12, left: -16, bottom: 0 }}>
            <CartesianGrid stroke="#ece9df" vertical={false} />
            <XAxis dataKey={xKey} tick={axisTick} axisLine={axisLine} tickLine={false} />
            <YAxis tick={axisTick} axisLine={false} tickLine={false} />
            <Tooltip contentStyle={tooltipStyle} cursor={{ stroke: '#d8d7cf', strokeWidth: 1 }} />
            {seriesKeys.map((key, i) => (
              <Line
                key={key}
                type="monotone"
                dataKey={key}
                stroke={SERIES_COLORS[i % SERIES_COLORS.length]}
                strokeWidth={1.5}
                dot={{ r: 2.5, strokeWidth: 1.5, stroke: '#fff', fill: SERIES_COLORS[i % SERIES_COLORS.length] }}
                activeDot={{ r: 5, strokeWidth: 1.5, stroke: '#fff' }}
              />
            ))}
          </LineChart>
        </ResponsiveContainer>
      </ScrollableChart>
    );
  }

  if (chart.chart_type === 'area') {
    if (seriesKeys.length === 0) {
      return (
        <div className={styles['db-analytics-chart-card__no-data']}>
          No numeric series found to plot for this chart.
        </div>
      );
    }
    return (
      <ScrollableChart minWidth={getPlotWidth(chart.data.length)} height={220}>
        <ResponsiveContainer width="100%" height="100%">
          <AreaChart data={chart.data} margin={{ top: 8, right: 12, left: -16, bottom: 0 }}>
            <defs>
              {seriesKeys.map((key, i) => (
                <linearGradient key={key} id={`${gradientPrefix}-${i}`} x1="0" y1="0" x2="0" y2="1">
                  <stop offset="0%" stopColor={SERIES_COLORS[i % SERIES_COLORS.length]} stopOpacity={0.35} />
                  <stop offset="100%" stopColor={SERIES_COLORS[i % SERIES_COLORS.length]} stopOpacity={0.02} />
                </linearGradient>
              ))}
            </defs>
            <CartesianGrid stroke="#ece9df" vertical={false} />
            <XAxis dataKey={xKey} tick={axisTick} axisLine={axisLine} tickLine={false} />
            <YAxis tick={axisTick} axisLine={false} tickLine={false} />
            <Tooltip contentStyle={tooltipStyle} cursor={{ stroke: '#d8d7cf', strokeWidth: 1 }} />
            {seriesKeys.map((key, i) => (
              <Area
                key={key}
                type="monotone"
                dataKey={key}
                stackId={chart.stacked ? 'stack' : undefined}
                stroke={SERIES_COLORS[i % SERIES_COLORS.length]}
                fill={`url(#${gradientPrefix}-${i})`}
                strokeWidth={1.5}
              />
            ))}
          </AreaChart>
        </ResponsiveContainer>
      </ScrollableChart>
    );
  }

  if (chart.chart_type === 'bar') {
    if (seriesKeys.length === 0) {
      return (
        <div className={styles['db-analytics-chart-card__no-data']}>
          No numeric series found to plot for this chart.
        </div>
      );
    }
    return (
      <ScrollableChart minWidth={getPlotWidth(chart.data.length)} height={220}>
        <ResponsiveContainer width="100%" height="100%">
          <BarChart data={chart.data} margin={{ top: 8, right: 12, left: -16, bottom: 0 }} barGap={6}>
            <defs>
              {seriesKeys.map((key, i) => (
                <linearGradient key={key} id={`${gradientPrefix}-bar-${i}`} x1="0" y1="0" x2="0" y2="1">
                  <stop offset="0%" stopColor={SERIES_COLORS[i % SERIES_COLORS.length]} stopOpacity={1} />
                  <stop offset="100%" stopColor={SERIES_COLORS[i % SERIES_COLORS.length]} stopOpacity={0.7} />
                </linearGradient>
              ))}
            </defs>
            <CartesianGrid stroke="#ece9df" vertical={false} />
            <XAxis dataKey={xKey} tick={axisTick} axisLine={axisLine} tickLine={false} />
            <YAxis tick={axisTick} axisLine={false} tickLine={false} />
            <Tooltip contentStyle={tooltipStyle} cursor={{ fill: 'rgba(79, 70, 229, 0.06)' }} />
            {seriesKeys.map((key, i) => (
              <Bar
                key={key}
                dataKey={key}
                stackId={chart.stacked ? 'stack' : undefined}
                fill={`url(#${gradientPrefix}-bar-${i})`}
                radius={[6, 6, 0, 0]}
                maxBarSize={32}
              />
            ))}
          </BarChart>
        </ResponsiveContainer>
      </ScrollableChart>
    );
  }

  if (chart.chart_type === 'scatter') {
    return (
      <ScrollableChart minWidth={getPlotWidth(chart.data.length)} height={220}>
        <ResponsiveContainer width="100%" height="100%">
          <ScatterChart margin={{ top: 8, right: 12, left: -16, bottom: 0 }}>
            <CartesianGrid stroke="#ece9df" />
            <XAxis dataKey={xKey} tick={axisTick} axisLine={axisLine} tickLine={false} />
            <YAxis dataKey={chart.y_axis ?? undefined} tick={axisTick} axisLine={false} tickLine={false} />
            <Tooltip contentStyle={tooltipStyle} cursor={{ strokeDasharray: '3 3' }} />
            <Scatter data={chart.data} fill={SERIES_COLORS[0]} fillOpacity={0.8} />
          </ScatterChart>
        </ResponsiveContainer>
      </ScrollableChart>
    );
  }

  if (chart.chart_type === 'table') {
    const columns = chart.data.length > 0 ? Object.keys(chart.data[0]) : [];
    return (
      <div className={styles['db-analytics-table-wrap']}>
        <table className={styles['db-analytics-table']}>
          <thead>
            <tr>
              {columns.map((col) => (
                <th key={col}>{col}</th>
              ))}
            </tr>
          </thead>
          <tbody>
            {chart.data.map((row, i) => (
              <tr key={i}>
                {columns.map((col) => (
                  <td key={col}>{String(row[col])}</td>
                ))}
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    );
  }

  // pie (default/fallback)
  const total = chart.data.reduce((sum, d) => sum + Number(d[chart.y_axis ?? 'value'] ?? 0), 0);
  const nameKey = chart.x_axis ?? 'name';
  const valueKey = chart.y_axis ?? 'value';

  return (
    <>
      <ResponsiveContainer width="100%" height={200}>
        <PieChart>
          <Pie
            data={chart.data}
            dataKey={valueKey}
            nameKey={nameKey}
            innerRadius={50}
            outerRadius={82}
            paddingAngle={2}
            strokeWidth={1.5}
            stroke="#fcfcfb"
            label={({ percent }) => (percent > 0.06 ? `${Math.round(percent * 100)}%` : '')}
            labelLine={false}
          >
            {chart.data.map((_, i) => (
              <Cell key={i} fill={SERIES_COLORS[i % SERIES_COLORS.length]} />
            ))}
          </Pie>
          <Tooltip contentStyle={tooltipStyle} />
        </PieChart>
      </ResponsiveContainer>
      <div className={styles['db-analytics-chart-legend']}>
        {chart.data.map((d, i) => (
          <React.Fragment key={i}>
            {i > 0 && <span className={styles['db-analytics-chart-legend__divider']} aria-hidden="true" />}
            <span className={styles['db-analytics-chart-legend__item']}>
              <span
                className={styles['db-analytics-chart-legend__swatch']}
                style={{ background: SERIES_COLORS[i % SERIES_COLORS.length] }}
              />
              <span className={styles['db-analytics-chart-legend__name']}>{String(d[nameKey])}</span>
              <span className={styles['db-analytics-chart-legend__pct']}>
                {total > 0 ? Math.round((Number(d[valueKey]) / total) * 100) : 0}%
              </span>
            </span>
          </React.Fragment>
        ))}
      </div>
    </>
  );
};

const ResultsPanel: React.FC<Props> = ({ charts, loadError, isGenerating }) => {
  const showEmptyState = charts.length === 0;

  return (
    <div className={styles['db-analytics-results-panel']}>
      <div className={styles['db-analytics-results-panel__header']}>
        <div className={styles['db-analytics-results-panel__header-text']}>
          <h2 className={styles['db-analytics-results-panel__title']}>Generated insights</h2>
          <p className={styles['db-analytics-results-panel__subtitle']}>
            Visualizations from your conversation
          </p>
        </div>
      </div>
      <div className={styles['db-analytics-results-panel__body']}>
        {loadError && <div className={styles['db-analytics-results-panel__error']}>{loadError}</div>}

        {showEmptyState ? (
          <div className={styles['db-analytics-empty-state']}>
            <Icon.chartAreaLine
              size={28}
              aria-hidden="true"
              className={isGenerating ? styles['db-analytics-empty-state__icon--generating'] : undefined}
            />
            <p>{isGenerating ? 'Drawing your chart…' : 'Charts generated from your chat will appear here.'}</p>
          </div>
        ) : (
          <div className={styles['db-analytics-chart-grid']}>
            {charts.map((chart, index) => (
              <div
                key={`${chart.id}-${index}`}
                className={`${styles['db-analytics-chart-card']} ${
                  chart.chart_type === 'kpi' ? styles['db-analytics-chart-card--span-2'] : ''
                }`}
              >
                <div className={styles['db-analytics-chart-card__header']}>
                  <div className={styles['db-analytics-chart-card__heading']}>
                    <span className={styles['db-analytics-chart-card__icon']}>
                      {(() => {
                        const ChartIcon = getChartIcon(chart.chart_type);
                        return <ChartIcon size={14} aria-hidden="true" />;
                      })()}
                    </span>
                    <h3 className={styles['db-analytics-chart-card__title']}>{chart.title}</h3>
                  </div>
                  <span className={styles['db-analytics-chart-card__badge']}>{chart.chart_type}</span>
                </div>
                {renderChart(chart)}
              </div>
            ))}
            {isGenerating && (
              <div className={styles['db-analytics-chart-card']}>
                <div className={styles['db-analytics-empty-state']}>
                  <Icon.chartAreaLine
                    size={28}
                    aria-hidden="true"
                    className={styles['db-analytics-empty-state__icon--generating']}
                  />
                  <p>Drawing your chart…</p>
                </div>
              </div>
            )}
          </div>
        )}
      </div>
    </div>
  );
};

export default ResultsPanel;









// ScrollableChart.module.scss
@use '../../../styles/tokens' as t;

.db-analytics-scrollable-chart {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.db-analytics-scrollable-chart__viewport {
  overflow-x: auto;
  overflow-y: hidden;

  // Hide the native scrollbar completely — we render our own always-visible
  // track/thumb below instead, so there's no reliance on OS/browser overlay
  // scrollbar behavior (which only shows on hover/scroll and fades out).
  scrollbar-width: none; // Firefox
  -ms-overflow-style: none; // legacy Edge/IE

  &::-webkit-scrollbar {
    display: none; // Chrome/Safari/Edge (Chromium)
  }
}

.db-analytics-scrollable-chart__track {
  position: relative;
  height: 5px;
  border-radius: 999px;
  background: t.$surface-0;
  border: 1px solid t.$border;
  flex-shrink: 0;
}

.db-analytics-scrollable-chart__thumb {
  position: absolute;
  top: -1px;
  bottom: -1px;
  border-radius: 999px;
  background: t.$border-stronger;
  transition: background 0.15s ease;
  min-width: 24px;
}

.db-analytics-scrollable-chart__viewport:hover + .db-analytics-scrollable-chart__track,
.db-analytics-scrollable-chart:hover .db-analytics-scrollable-chart__thumb {
  background: t.$text-muted;
}








// ScrollableChart.tsx

import React, { useEffect, useLayoutEffect, useRef, useState } from 'react';
import styles from './ScrollableChart.module.scss';

interface Props {
  minWidth: number;
  height: number;
  children: React.ReactNode;
}

// Wraps chart content in a horizontally scrollable box with our own
// always-rendered track + thumb, rather than relying on the browser's
// native scrollbar. Native/OS "overlay" scrollbars only paint while
// actively hovering or scrolling and fade out immediately after — which is
// exactly the flashing behavior we don't want here. Drawing the indicator
// ourselves guarantees it's visible any time the content overflows,
// regardless of OS scrollbar settings or browser vendor.
const ScrollableChart: React.FC<Props> = ({ minWidth, height, children }) => {
  const viewportRef = useRef<HTMLDivElement>(null);
  const [viewportWidth, setViewportWidth] = useState(0);
  const [scrollLeft, setScrollLeft] = useState(0);

  const contentWidth = Math.max(minWidth, viewportWidth);
  const isScrollable = contentWidth > viewportWidth + 1;

  useLayoutEffect(() => {
    const el = viewportRef.current;
    if (!el) return;

    const measure = () => setViewportWidth(el.clientWidth);
    measure();

    const observer = new ResizeObserver(measure);
    observer.observe(el);
    return () => observer.disconnect();
  }, []);

  useEffect(() => {
    const el = viewportRef.current;
    if (!el) return;
    const onScroll = () => setScrollLeft(el.scrollLeft);
    el.addEventListener('scroll', onScroll, { passive: true });
    return () => el.removeEventListener('scroll', onScroll);
  }, []);

  const maxScroll = Math.max(1, contentWidth - viewportWidth);
  const thumbWidthPct = Math.min(100, (viewportWidth / contentWidth) * 100);
  const thumbLeftPct = (scrollLeft / maxScroll) * (100 - thumbWidthPct);

  return (
    <div className={styles['db-analytics-scrollable-chart']}>
      <div ref={viewportRef} className={styles['db-analytics-scrollable-chart__viewport']}>
        <div style={{ minWidth, height }}>{children}</div>
      </div>
      {isScrollable && (
        <div className={styles['db-analytics-scrollable-chart__track']} aria-hidden="true">
          <div
            className={styles['db-analytics-scrollable-chart__thumb']}
            style={{ width: `${thumbWidthPct}%`, left: `${thumbLeftPct}%` }}
          />
        </div>
      )}
    </div>
  );
};

export default ScrollableChart;
