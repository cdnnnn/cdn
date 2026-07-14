//ResultsPanel.tsx
import React, { useState } from 'react';
import { useTranslation } from 'react-i18next';
import i18n from '../../../i18n';
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
import { ChartResult, ChartType, ChartGroup } from '../types';
import { Icon, IconComponent } from '../icons';
import ChartTypeMenu from './ChartTypeMenu';
import ExpandChartSlider from './ExpandChartSlider';
import ChartAskAiSlider from './ChartAskAiSlider';
import ChartConstructingAnimation from './ChartConstructingAnimation';

interface Props {
  chartGroups: ChartGroup[];
  loadError?: string | null;
  isGenerating?: boolean;
  sessionId: string | null;
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

export const getChartIcon = (type: ChartResult['chart_type']): IconComponent =>
  chartTypeIcon[type] ?? FALLBACK_ICON;

// Graph types a person can switch a card to from its header dropdown.
// KPI and table are left out — they're structurally different (not a
// category/series plot) and switching into them from a plotted chart
// wouldn't produce anything meaningful. Labels are resolved through i18n at
// render time in ChartTypeMenu (see chartTypeMenu.types.* keys) — this array
// only carries the stable `value`s plus an English fallback label.
export const SWITCHABLE_CHART_TYPES: { value: ChartType; label: string }[] = [
  { value: 'bar', label: 'Bar' },
  { value: 'line', label: 'Line' },
  { value: 'area', label: 'Area' },
  { value: 'pie', label: 'Pie' },
  { value: 'scatter', label: 'Scatter' },
];

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

// Uses the same CSS custom properties the rest of the app's theme runs on
// (see global.scss / _tokens.scss) rather than hardcoded colors — Recharts
// renders tooltip content via inline styles on a plain DOM node outside the
// app's own component tree, so it never picked up dark-mode text/background
// colors before; every value here was effectively frozen to a light theme,
// which is why tooltip text became unreadable (dark text implied by
// Recharts' own defaults, sitting on top of what was still a hardcoded
// white box) once the surrounding UI switched to dark mode.
const tooltipStyle = {
  background: 'var(--db-analytics-surface-2)',
  color: 'var(--db-analytics-text-primary)',
  border: '1px solid var(--db-analytics-border-strong)',
  borderRadius: 10,
  fontSize: 12,
  boxShadow: '0 6px 20px rgba(11,11,11,0.10)',
  padding: '8px 12px',
};

const tooltipLabelStyle = {
  color: 'var(--db-analytics-text-primary)',
  fontWeight: 600,
  marginBottom: 4,
};

const tooltipItemStyle = {
  color: 'var(--db-analytics-text-secondary)',
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
  const numericKeys = Object.keys(sample).filter((key) => typeof sample[key] === 'number');
  const excludingXKey = numericKeys.filter((key) => key !== xKey);

  // A chart originally shaped for pie (or any type where x_axis names the
  // value/category field rather than a literal plot axis) can end up with
  // an x_axis that happens to match the data's only numeric column — e.g.
  // x_axis: "value" on a {region, value} pie dataset. Excluding that column
  // would leave zero series to plot when switching to bar/line/area, even
  // though the data is perfectly plottable. If excluding xKey empties the
  // set entirely, trust the numeric columns as-is instead of the axis hint.
  return excludingXKey.length > 0 ? excludingXKey : numericKeys;
};

// Resolves which field should act as the category/label axis for any chart
// type — prefers the API-given x_axis, but only if that key actually
// exists on the data AND isn't itself a numeric value column (a chart
// originally shaped for pie can have x_axis literally name the value
// field, e.g. x_axis: "value" on a {region, value} dataset — trusting that
// blindly would try to plot categories against a number). Falls back to
// the first string-valued column, since that's the most reliable signal
// for "this is a label, not a metric" regardless of what the axis hint says.
const resolveCategoryKey = (chart: ChartResult): string => {
  const sample = chart.data[0] ?? {};
  const keys = Object.keys(sample);

  if (chart.x_axis && keys.includes(chart.x_axis) && typeof sample[chart.x_axis] !== 'number') {
    return chart.x_axis;
  }
  return keys.find((k) => typeof sample[k] === 'string') ?? chart.x_axis ?? keys[0] ?? 'name';
};

const axisTick = { fontSize: 11.5, fill: 'var(--db-analytics-text-muted)', fontWeight: 500 };
const axisLine = { stroke: 'var(--db-analytics-border-strong)' };

// Resolves which fields to use for a pie slice's label and value. Prefers
// the API-given x_axis/y_axis, but only if that key actually exists on the
// data — a null/missing/incorrect axis field (common on pie responses,
// since they don't always set x_axis/y_axis the way category charts do)
// used to silently default to 'name'/'value', which is wrong whenever the
// real fields are named something else (e.g. 'region', 'total'). Falls back
// to the first string-valued key for the name and the first numeric-valued
// key for the value.
const resolvePieKeys = (chart: ChartResult): { nameKey: string; valueKey: string } => {
  const sample = chart.data[0] ?? {};
  const keys = Object.keys(sample);
  const nameKey = resolveCategoryKey(chart);

  const valueKey =
    chart.y_axis && keys.includes(chart.y_axis)
      ? chart.y_axis
      : keys.find((k) => k !== nameKey && typeof sample[k] === 'number') ?? 'value';

  return { nameKey, valueKey };
};

// renderChart is a plain function (not a component), called both from this
// file's JSX map and from ExpandChartSlider/ChartAskAiSlider, so it can't
// use the useTranslation() hook. It reads directly off the shared i18next
// instance instead, which works outside of React's render/hook lifecycle
// and still re-resolves to the current language on every call.
export const renderChart = (chart: ChartResult, height = 220, overrideType?: ChartType) => {
  const xKey = chart.data && chart.data.length > 0 ? resolveCategoryKey(chart) : chart.x_axis ?? 'name';
  const seriesKeys = deriveSeriesKeys(chart, xKey);
  const gradientPrefix = `db-analytics-grad-${chart.id}`;
  const effectiveType = overrideType ?? chart.chart_type;

  if (!chart.data || chart.data.length === 0) {
    return (
      <div className={styles['db-analytics-chart-card__no-data']}>{i18n.t('db_analytics.resultsPanel.noDataForChart')}</div>
    );
  }

  if (effectiveType === 'kpi') {
    return renderKpi(chart);
  }

  if (effectiveType === 'line') {
    if (seriesKeys.length === 0) {
      return (
        <div className={styles['db-analytics-chart-card__no-data']}>{i18n.t('db_analytics.resultsPanel.noSeriesFound')}</div>
      );
    }
    return (
      <ResponsiveContainer width="100%" height={height}>
        <LineChart data={chart.data} margin={{ top: 8, right: 12, left: -16, bottom: 0 }}>
          <CartesianGrid stroke="var(--db-analytics-border)" vertical={false} />
          <XAxis dataKey={xKey} tick={axisTick} axisLine={axisLine} tickLine={false} />
          <YAxis tick={axisTick} axisLine={false} tickLine={false} />
          <Tooltip
            contentStyle={tooltipStyle}
            labelStyle={tooltipLabelStyle}
            itemStyle={tooltipItemStyle}
            cursor={{ stroke: 'var(--db-analytics-border-stronger)', strokeWidth: 1 }}
          />
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
    );
  }

  if (effectiveType === 'area') {
    if (seriesKeys.length === 0) {
      return (
        <div className={styles['db-analytics-chart-card__no-data']}>{i18n.t('db_analytics.resultsPanel.noSeriesFound')}</div>
      );
    }
    return (
      <ResponsiveContainer width="100%" height={height}>
        <AreaChart data={chart.data} margin={{ top: 8, right: 12, left: -16, bottom: 0 }}>
          <defs>
            {seriesKeys.map((key, i) => (
              <linearGradient key={key} id={`${gradientPrefix}-${i}`} x1="0" y1="0" x2="0" y2="1">
                <stop offset="0%" stopColor={SERIES_COLORS[i % SERIES_COLORS.length]} stopOpacity={0.35} />
                <stop offset="100%" stopColor={SERIES_COLORS[i % SERIES_COLORS.length]} stopOpacity={0.02} />
              </linearGradient>
            ))}
          </defs>
          <CartesianGrid stroke="var(--db-analytics-border)" vertical={false} />
          <XAxis dataKey={xKey} tick={axisTick} axisLine={axisLine} tickLine={false} />
          <YAxis tick={axisTick} axisLine={false} tickLine={false} />
          <Tooltip
            contentStyle={tooltipStyle}
            labelStyle={tooltipLabelStyle}
            itemStyle={tooltipItemStyle}
            cursor={{ stroke: 'var(--db-analytics-border-stronger)', strokeWidth: 1 }}
          />
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
    );
  }

  if (effectiveType === 'bar') {
    if (seriesKeys.length === 0) {
      return (
        <div className={styles['db-analytics-chart-card__no-data']}>{i18n.t('db_analytics.resultsPanel.noSeriesFound')}</div>
      );
    }
    return (
      <>
        <ResponsiveContainer width="100%" height={height}>
          <BarChart data={chart.data} margin={{ top: 8, right: 12, left: -16, bottom: 0 }} barGap={6}>
            <defs>
              {seriesKeys.map((key, i) => (
                <linearGradient key={key} id={`${gradientPrefix}-bar-${i}`} x1="0" y1="0" x2="0" y2="1">
                  <stop offset="0%" stopColor={SERIES_COLORS[i % SERIES_COLORS.length]} stopOpacity={1} />
                  <stop offset="100%" stopColor={SERIES_COLORS[i % SERIES_COLORS.length]} stopOpacity={0.7} />
                </linearGradient>
              ))}
            </defs>
            <CartesianGrid stroke="var(--db-analytics-border)" vertical={false} />
            <XAxis dataKey={xKey} tick={axisTick} axisLine={axisLine} tickLine={false} />
            <YAxis tick={axisTick} axisLine={false} tickLine={false} />
            <Tooltip
              contentStyle={tooltipStyle}
              labelStyle={tooltipLabelStyle}
              itemStyle={tooltipItemStyle}
              cursor={{ fill: 'var(--db-analytics-accent-bg)' }}
            />
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
        {seriesKeys.length > 1 && (
          <div className={styles['db-analytics-chart-legend']}>
            {seriesKeys.map((key, i) => (
              <React.Fragment key={key}>
                {i > 0 && <span className={styles['db-analytics-chart-legend__divider']} aria-hidden="true" />}
                <span className={styles['db-analytics-chart-legend__item']}>
                  <span
                    className={styles['db-analytics-chart-legend__swatch']}
                    style={{ background: SERIES_COLORS[i % SERIES_COLORS.length] }}
                  />
                  <span className={styles['db-analytics-chart-legend__name']}>{key}</span>
                </span>
              </React.Fragment>
            ))}
          </div>
        )}
      </>
    );
  }

  if (effectiveType === 'scatter') {
    return (
      <ResponsiveContainer width="100%" height={height}>
        <ScatterChart margin={{ top: 8, right: 12, left: -16, bottom: 0 }}>
          <CartesianGrid stroke="var(--db-analytics-border)" />
          <XAxis dataKey={xKey} tick={axisTick} axisLine={axisLine} tickLine={false} />
          <YAxis dataKey={chart.y_axis ?? undefined} tick={axisTick} axisLine={false} tickLine={false} />
          <Tooltip
            contentStyle={tooltipStyle}
            labelStyle={tooltipLabelStyle}
            itemStyle={tooltipItemStyle}
            cursor={{ strokeDasharray: '3 3', stroke: 'var(--db-analytics-border-stronger)' }}
          />
          <Scatter data={chart.data} fill={SERIES_COLORS[0]} fillOpacity={0.8} />
        </ScatterChart>
      </ResponsiveContainer>
    );
  }

  if (effectiveType === 'table') {
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
  const { nameKey, valueKey } = resolvePieKeys(chart);
  const total = chart.data.reduce((sum, d) => sum + Number(d[valueKey] ?? 0), 0);

  return (
    <>
      <ResponsiveContainer width="100%" height={height - 20}>
        <PieChart>
          <Pie
            data={chart.data}
            dataKey={valueKey}
            nameKey={nameKey}
            innerRadius={50}
            outerRadius={82}
            paddingAngle={2}
            strokeWidth={1.5}
            stroke="var(--db-analytics-surface-2)"
            label={({ percent }) => (percent > 0.06 ? `${Math.round(percent * 100)}%` : '')}
            labelLine={false}
          >
            {chart.data.map((_, i) => (
              <Cell key={i} fill={SERIES_COLORS[i % SERIES_COLORS.length]} />
            ))}
          </Pie>
          <Tooltip contentStyle={tooltipStyle} labelStyle={tooltipLabelStyle} itemStyle={tooltipItemStyle} />
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

const ResultsPanel: React.FC<Props> = ({ chartGroups, loadError, isGenerating, sessionId }) => {
  const { t } = useTranslation();
  const showEmptyState = chartGroups.length === 0;
  const [typeOverrides, setTypeOverrides] = useState<Record<string, ChartType>>({});
  const [expandedChartKey, setExpandedChartKey] = useState<string | null>(null);
  const [askAiChartKey, setAskAiChartKey] = useState<string | null>(null);

  // Chart ids aren't guaranteed unique across the whole session — the
  // backend can reuse the same id (e.g. a generic "revenue_kpis") for
  // charts belonging to different prompts/groups. Using chart.id alone as
  // the key for type overrides / expand / ask-AI state meant switching one
  // card's type silently changed every other card that happened to share
  // that id. This composite key scopes state to one specific card instance.
  const getChartKey = (groupId: string, chart: ChartResult, index: number) => `${groupId}::${chart.id}::${index}`;

  const getDisplayType = (key: string, chart: ChartResult): ChartType => typeOverrides[key] ?? chart.chart_type;

  const chartsByKey = new Map<string, ChartResult>();
  chartGroups.forEach((group) => {
    group.graphs.forEach((chart, index) => {
      chartsByKey.set(getChartKey(group.id, chart, index), chart);
    });
  });

  const expandedChart = expandedChartKey ? chartsByKey.get(expandedChartKey) ?? null : null;
  const expandedDisplayType = expandedChart && expandedChartKey ? getDisplayType(expandedChartKey, expandedChart) : 'bar';
  const askAiChart = askAiChartKey ? chartsByKey.get(askAiChartKey) ?? null : null;
  const askAiDisplayType = askAiChart && askAiChartKey ? getDisplayType(askAiChartKey, askAiChart) : 'bar';

  return (
    <div className={styles['db-analytics-results-panel']}>
      <div className={styles['db-analytics-results-panel__header']}>
        <div className={styles['db-analytics-results-panel__header-text']}>
          <h2 className={styles['db-analytics-results-panel__title']}>{t('db_analytics.resultsPanel.title')}</h2>
          <p className={styles['db-analytics-results-panel__subtitle']}>{t('db_analytics.resultsPanel.subtitle')}</p>
        </div>
      </div>
      <div className={styles['db-analytics-results-panel__body']}>
        {loadError && <div className={styles['db-analytics-results-panel__error']}>{loadError}</div>}

        {isGenerating ? (
          <ChartConstructingAnimation />
        ) : showEmptyState ? (
          <div className={styles['db-analytics-empty-state']}>
            <Icon.chartAreaLine size={28} aria-hidden="true" />
            <p>{t('db_analytics.resultsPanel.emptyState')}</p>
          </div>
        ) : (
          <div className={styles['db-analytics-chart-groups']}>
            {chartGroups.map((group, groupIndex) => (
              <React.Fragment key={group.id}>
                {groupIndex > 0 && <div className={styles['db-analytics-chart-groups__divider']} aria-hidden="true" />}
                <section className={styles['db-analytics-chart-group']}>
                  {group.prompt && (
                    <div className={styles['db-analytics-chart-group__prompt']}>
                      <span className={styles['db-analytics-chart-group__prompt-label']}>
                        {t('db_analytics.resultsPanel.prompt')}
                      </span>
                      <p className={styles['db-analytics-chart-group__prompt-text']}>{group.prompt}</p>
                    </div>
                  )}
                  <div className={styles['db-analytics-chart-grid']}>
                    {group.graphs.map((chart, index) => {
                      const chartKey = getChartKey(group.id, chart, index);
                      const displayType = getDisplayType(chartKey, chart);
                      const canSwitchType = chart.chart_type !== 'kpi' && chart.chart_type !== 'table';
                      return (
                        <div
                          key={chartKey}
                          className={`${styles['db-analytics-chart-card']} ${
                            chart.chart_type === 'kpi' ? styles['db-analytics-chart-card--span-2'] : ''
                          }`}
                        >
                          <div className={styles['db-analytics-chart-card__header']}>
                            <div className={styles['db-analytics-chart-card__heading']}>
                              <span className={styles['db-analytics-chart-card__icon']}>
                                {(() => {
                                  const ChartIcon = getChartIcon(displayType);
                                  return <ChartIcon size={14} aria-hidden="true" />;
                                })()}
                              </span>
                              <h3 className={styles['db-analytics-chart-card__title']}>{chart.title}</h3>
                            </div>
                            <div className={styles['db-analytics-chart-card__actions']}>
                              {canSwitchType && (
                                <ChartTypeMenu
                                  value={displayType}
                                  onChange={(type) =>
                                    setTypeOverrides((prev) => ({ ...prev, [chartKey]: type }))
                                  }
                                />
                              )}
                              {chart.chart_type !== 'kpi' && (
                                <>
                                  <button
                                    type="button"
                                    className={styles['db-analytics-chart-card__expand-btn']}
                                    onClick={() => setAskAiChartKey(chartKey)}
                                    aria-label={t('db_analytics.resultsPanel.askAiAboutChart')}
                                    title={t('db_analytics.resultsPanel.askAi')}
                                  >
                                    <Icon.askAi size={14} aria-hidden="true" />
                                  </button>
                                  <button
                                    type="button"
                                    className={styles['db-analytics-chart-card__expand-btn']}
                                    onClick={() => setExpandedChartKey(chartKey)}
                                    aria-label={t('db_analytics.resultsPanel.expandChart')}
                                    title={t('db_analytics.resultsPanel.expand')}
                                  >
                                    <Icon.expand size={14} aria-hidden="true" />
                                  </button>
                                </>
                              )}
                            </div>
                          </div>
                          {renderChart(chart, 220, displayType)}
                        </div>
                      );
                    })}
                  </div>
                </section>
              </React.Fragment>
            ))}
          </div>
        )}
      </div>

      <ExpandChartSlider
        chart={expandedChart}
        displayType={expandedDisplayType}
        onClose={() => setExpandedChartKey(null)}
      />

      <ChartAskAiSlider
        sessionId={sessionId}
        chart={askAiChart}
        displayType={askAiDisplayType}
        onClose={() => setAskAiChartKey(null)}
      />
    </div>
  );
};

export default ResultsPanel;
