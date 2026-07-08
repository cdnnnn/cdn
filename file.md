//ChartTypeMenu.module.scss

@use '../../../styles/tokens' as t;

.db-analytics-chart-type-menu {
  position: relative;
}

.db-analytics-chart-type-menu__trigger {
  display: flex;
  align-items: center;
  gap: 5px;
  font-size: 10.5px;
  font-weight: 600;
  color: t.$text-secondary;
  background: t.$surface-0;
  border: 0.5px solid t.$border-strong;
  padding: 4px 8px;
  border-radius: 999px;
  cursor: pointer;
  font-family: inherit;
  white-space: nowrap;
  transition: background 0.15s ease, border-color 0.15s ease;

  &:hover {
    background: t.$surface-2;
    border-color: t.$border-stronger;
  }
}

.db-analytics-chart-type-menu__chevron {
  transform: rotate(90deg);
  color: t.$text-muted;
  flex-shrink: 0;
}

.db-analytics-chart-type-menu__list {
  position: absolute;
  top: calc(100% + 6px);
  right: 0;
  z-index: 20;
  list-style: none;
  margin: 0;
  padding: 5px;
  min-width: 140px;
  background: t.$surface-2;
  border: 1px solid t.$border-strong;
  border-radius: t.$radius;
  box-shadow: 0 8px 22px rgba(11, 11, 11, 0.14);
  animation: db-analytics-chart-type-menu-pop 0.12s ease;
}

@keyframes db-analytics-chart-type-menu-pop {
  from {
    opacity: 0;
    transform: translateY(-2px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.db-analytics-chart-type-menu__option {
  display: flex;
  align-items: center;
  gap: 8px;
  width: 100%;
  padding: 7px 8px;
  border: none;
  background: none;
  border-radius: 6px;
  font-size: 12px;
  font-weight: 500;
  color: t.$text-primary;
  cursor: pointer;
  font-family: inherit;
  text-align: left;

  span {
    flex: 1;
  }

  &:hover {
    background: t.$surface-0;
  }
}

.db-analytics-chart-type-menu__option--selected {
  background: t.$accent-bg;
  color: #0c447c;

  &:hover {
    background: t.$accent-bg;
  }
}













// ChartTypeMenu.tsx

import React, { useEffect, useRef, useState } from 'react';
import styles from './ChartTypeMenu.module.scss';
import { ChartType } from '../types';
import { Icon, IconComponent } from '../icons';
import { getChartIcon, SWITCHABLE_CHART_TYPES } from './ResultsPanel';

interface Props {
  value: ChartType;
  onChange: (type: ChartType) => void;
}

const ChartTypeMenu: React.FC<Props> = ({ value, onChange }) => {
  const [open, setOpen] = useState(false);
  const rootRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!open) return;
    const onClickOutside = (e: MouseEvent) => {
      if (rootRef.current && !rootRef.current.contains(e.target as Node)) {
        setOpen(false);
      }
    };
    document.addEventListener('mousedown', onClickOutside);
    return () => document.removeEventListener('mousedown', onClickOutside);
  }, [open]);

  const CurrentIcon: IconComponent = getChartIcon(value);
  const currentLabel = SWITCHABLE_CHART_TYPES.find((t) => t.value === value)?.label ?? value;

  return (
    <div className={styles['db-analytics-chart-type-menu']} ref={rootRef}>
      <button
        type="button"
        className={styles['db-analytics-chart-type-menu__trigger']}
        onClick={() => setOpen((o) => !o)}
        aria-haspopup="listbox"
        aria-expanded={open}
        title="Change chart type"
      >
        <CurrentIcon size={12} aria-hidden="true" />
        <span>{currentLabel}</span>
        <Icon.arrowRight
          size={11}
          aria-hidden="true"
          className={styles['db-analytics-chart-type-menu__chevron']}
        />
      </button>

      {open && (
        <ul className={styles['db-analytics-chart-type-menu__list']} role="listbox">
          {SWITCHABLE_CHART_TYPES.map((option) => {
            const OptionIcon = getChartIcon(option.value);
            const isSelected = option.value === value;
            return (
              <li key={option.value}>
                <button
                  type="button"
                  className={`${styles['db-analytics-chart-type-menu__option']} ${
                    isSelected ? styles['db-analytics-chart-type-menu__option--selected'] : ''
                  }`}
                  role="option"
                  aria-selected={isSelected}
                  onClick={() => {
                    onChange(option.value);
                    setOpen(false);
                  }}
                >
                  <OptionIcon size={13} aria-hidden="true" />
                  <span>{option.label}</span>
                  {isSelected && <Icon.check size={12} aria-hidden="true" />}
                </button>
              </li>
            );
          })}
        </ul>
      )}
    </div>
  );
};

export default ChartTypeMenu;





// ExpandChartSlider.module.scss

@use '../../../styles/tokens' as t;

.db-analytics-expand-overlay {
  position: fixed;
  inset: 0;
  z-index: 150;
  display: flex;
  justify-content: flex-end;
}

.db-analytics-expand-overlay__backdrop {
  position: absolute;
  inset: 0;
  background: rgba(11, 11, 11, 0.32);
  animation: db-analytics-expand-fade-in 0.15s ease;
}

.db-analytics-expand-panel {
  position: relative;
  width: 60vw;
  min-width: 560px;
  max-width: 900px;
  height: 100%;
  background: t.$surface-2;
  display: flex;
  flex-direction: column;
  box-shadow: -8px 0 24px rgba(11, 11, 11, 0.14);
  animation: db-analytics-expand-slide-in 0.22s cubic-bezier(0.16, 1, 0.3, 1);
}

@keyframes db-analytics-expand-slide-in {
  from {
    transform: translateX(100%);
  }
  to {
    transform: translateX(0);
  }
}

@keyframes db-analytics-expand-fade-in {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

.db-analytics-expand-panel__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 64px;
  padding: 0 22px;
  border-bottom: 1px solid t.$border-strong;
  background: t.$surface-1;
  flex-shrink: 0;
}

.db-analytics-expand-panel__heading {
  display: flex;
  align-items: center;
  gap: 12px;
  min-width: 0;
}

.db-analytics-expand-panel__icon {
  width: 34px;
  height: 34px;
  border-radius: 9px;
  background: t.$gradient-primary;
  color: #fff;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  box-shadow: 0 2px 6px rgba(79, 70, 229, 0.25);
}

.db-analytics-expand-panel__title {
  font-size: 15px;
  font-weight: 600;
  color: t.$text-primary;
  margin: 0 0 2px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 420px;
}

.db-analytics-expand-panel__badge {
  font-size: 10px;
  font-weight: 600;
  letter-spacing: 0.02em;
  text-transform: uppercase;
  color: t.$text-secondary;
  background: t.$surface-0;
  border: 0.5px solid t.$border;
  padding: 2px 7px;
  border-radius: 999px;
}

.db-analytics-expand-panel__close-btn {
  width: 32px;
  height: 32px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border-radius: t.$radius;
  border: 0.5px solid t.$border-strong;
  background: transparent;
  color: t.$text-secondary;
  cursor: pointer;
  flex-shrink: 0;

  &:hover {
    background: t.$surface-0;
  }
}

.db-analytics-expand-panel__body {
  flex: 1;
  overflow-y: auto;
  padding: 28px;
}




// ExpandChartSlider.tsx

import React from 'react';
import styles from './ExpandChartSlider.module.scss';
import { ChartResult, ChartType } from '../types';
import { Icon, IconComponent } from '../icons';
import { renderChart, getChartIcon } from './ResultsPanel';

interface Props {
  chart: ChartResult | null;
  displayType: ChartType;
  onClose: () => void;
}

const ExpandChartSlider: React.FC<Props> = ({ chart, displayType, onClose }) => {
  if (!chart) return null;

  const ChartIcon: IconComponent = getChartIcon(displayType);

  return (
    <div className={styles['db-analytics-expand-overlay']} role="dialog" aria-modal="true">
      <div className={styles['db-analytics-expand-overlay__backdrop']} onClick={onClose} />
      <aside className={styles['db-analytics-expand-panel']}>
        <div className={styles['db-analytics-expand-panel__header']}>
          <div className={styles['db-analytics-expand-panel__heading']}>
            <span className={styles['db-analytics-expand-panel__icon']}>
              <ChartIcon size={16} aria-hidden="true" />
            </span>
            <div>
              <h2 className={styles['db-analytics-expand-panel__title']}>{chart.title}</h2>
              <span className={styles['db-analytics-expand-panel__badge']}>{displayType}</span>
            </div>
          </div>
          <button className={styles['db-analytics-expand-panel__close-btn']} aria-label="Close" onClick={onClose}>
            <Icon.close size={18} aria-hidden="true" />
          </button>
        </div>

        <div className={styles['db-analytics-expand-panel__body']}>
          {renderChart(chart, 440, displayType)}
        </div>
      </aside>
    </div>
  );
};

export default ExpandChartSlider;






// ResultsPanel.module.scss

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

.db-analytics-chart-card__actions {
  display: flex;
  align-items: center;
  gap: 6px;
  flex-shrink: 0;
}

.db-analytics-chart-card__expand-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 24px;
  height: 24px;
  border-radius: 999px;
  border: 0.5px solid t.$border-strong;
  background: t.$surface-0;
  color: t.$text-secondary;
  cursor: pointer;
  flex-shrink: 0;
  transition: background 0.15s ease, border-color 0.15s ease, color 0.15s ease;

  &:hover {
    background: t.$surface-2;
    border-color: t.$border-stronger;
    color: t.$text-primary;
  }
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

import React, { useState } from 'react';
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
import ChartTypeMenu from './ChartTypeMenu';
import ExpandChartSlider from './ExpandChartSlider';

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

export const getChartIcon = (type: ChartResult['chart_type']): IconComponent =>
  chartTypeIcon[type] ?? FALLBACK_ICON;

// Graph types a person can switch a card to from its header dropdown.
// KPI and table are left out — they're structurally different (not a
// category/series plot) and switching into them from a plotted chart
// wouldn't produce anything meaningful.
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

export const renderChart = (chart: ChartResult, height = 220, overrideType?: ChartType) => {
  const xKey = chart.x_axis ?? 'name';
  const seriesKeys = deriveSeriesKeys(chart, xKey);
  const gradientPrefix = `db-analytics-grad-${chart.id}`;
  const effectiveType = overrideType ?? chart.chart_type;

  if (!chart.data || chart.data.length === 0) {
    return <div className={styles['db-analytics-chart-card__no-data']}>No data returned for this chart.</div>;
  }

  if (effectiveType === 'kpi') {
    return renderKpi(chart);
  }

  if (effectiveType === 'line') {
    if (seriesKeys.length === 0) {
      return (
        <div className={styles['db-analytics-chart-card__no-data']}>
          No numeric series found to plot for this chart.
        </div>
      );
    }
    return (
      <ResponsiveContainer width="100%" height={height}>
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
    );
  }

  if (effectiveType === 'area') {
    if (seriesKeys.length === 0) {
      return (
        <div className={styles['db-analytics-chart-card__no-data']}>
          No numeric series found to plot for this chart.
        </div>
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
    );
  }

  if (effectiveType === 'bar') {
    if (seriesKeys.length === 0) {
      return (
        <div className={styles['db-analytics-chart-card__no-data']}>
          No numeric series found to plot for this chart.
        </div>
      );
    }
    return (
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
    );
  }

  if (effectiveType === 'scatter') {
    return (
      <ResponsiveContainer width="100%" height={height}>
        <ScatterChart margin={{ top: 8, right: 12, left: -16, bottom: 0 }}>
          <CartesianGrid stroke="#ece9df" />
          <XAxis dataKey={xKey} tick={axisTick} axisLine={axisLine} tickLine={false} />
          <YAxis dataKey={chart.y_axis ?? undefined} tick={axisTick} axisLine={false} tickLine={false} />
          <Tooltip contentStyle={tooltipStyle} cursor={{ strokeDasharray: '3 3' }} />
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
  const total = chart.data.reduce((sum, d) => sum + Number(d[chart.y_axis ?? 'value'] ?? 0), 0);
  const nameKey = chart.x_axis ?? 'name';
  const valueKey = chart.y_axis ?? 'value';

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
  const [typeOverrides, setTypeOverrides] = useState<Record<string, ChartType>>({});
  const [expandedChartId, setExpandedChartId] = useState<string | null>(null);

  const getDisplayType = (chart: ChartResult): ChartType => typeOverrides[chart.id] ?? chart.chart_type;

  const expandedChart = charts.find((c) => c.id === expandedChartId) ?? null;
  const expandedDisplayType = expandedChart ? getDisplayType(expandedChart) : 'bar';

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

        {isGenerating ? (
          <div className={styles['db-analytics-empty-state']}>
            <Icon.chartAreaLine
              size={28}
              aria-hidden="true"
              className={styles['db-analytics-empty-state__icon--generating']}
            />
            <p>Drawing your chart…</p>
          </div>
        ) : showEmptyState ? (
          <div className={styles['db-analytics-empty-state']}>
            <Icon.chartAreaLine size={28} aria-hidden="true" />
            <p>Charts generated from your chat will appear here.</p>
          </div>
        ) : (
          <div className={styles['db-analytics-chart-grid']}>
            {charts.map((chart, index) => {
              const displayType = getDisplayType(chart);
              const canSwitchType = chart.chart_type !== 'kpi' && chart.chart_type !== 'table';
              return (
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
                            setTypeOverrides((prev) => ({ ...prev, [chart.id]: type }))
                          }
                        />
                      )}
                      <button
                        type="button"
                        className={styles['db-analytics-chart-card__expand-btn']}
                        onClick={() => setExpandedChartId(chart.id)}
                        aria-label="Expand chart"
                        title="Expand"
                      >
                        <Icon.expand size={14} aria-hidden="true" />
                      </button>
                    </div>
                  </div>
                  {renderChart(chart, 220, displayType)}
                </div>
              );
            })}
          </div>
        )}
      </div>

      <ExpandChartSlider
        chart={expandedChart}
        displayType={expandedDisplayType}
        onClose={() => setExpandedChartId(null)}
      />
    </div>
  );
};

export default ResultsPanel;








// DBAnalytics.tsx

import React, { useEffect, useState, useCallback, useRef } from 'react';
import styles from './DBAnalytics.module.scss';
import SessionHistory from './components/SessionHistory';
import DatabaseList from './components/DatabaseList';
import ChatPanel from './components/ChatPanel';
import ResultsPanel from './components/ResultsPanel';
import DatabaseManagerSlider from './components/DatabaseManagerSlider';
import NewSessionSlider from './components/NewSessionSlider';
import { ChatMessage, ChartResult, DatabaseItem, SessionItem, SuggestedPrompt } from './types';
import { api, MissingDbConnectionError } from '../../services/api';
import { Icon } from './icons';

const STREAM_MESSAGE_ID = 'stream-in-progress';

const DBAnalytics: React.FC = () => {
  const [sessions, setSessions] = useState<SessionItem[]>([]);
  const [databases, setDatabases] = useState<DatabaseItem[]>([]);
  const [activeSessionId, setActiveSessionId] = useState<string | null>(null);
  const [messages, setMessages] = useState<ChatMessage[]>([]);

  const [loading, setLoading] = useState(true);
  const [loadError, setLoadError] = useState<string | null>(null);

  const [messagesLoading, setMessagesLoading] = useState(false);
  const [messagesError, setMessagesError] = useState<string | null>(null);

  const [suggestedPrompts, setSuggestedPrompts] = useState<SuggestedPrompt[]>([]);
  const [suggestedPromptsLoading, setSuggestedPromptsLoading] = useState(false);
  const [suggestedPromptsError, setSuggestedPromptsError] = useState<string | null>(null);

  // --- Live query streaming state ---
  const [isStreaming, setIsStreaming] = useState(false);
  const [streamSteps, setStreamSteps] = useState<string[]>([]);
  const [streamError, setStreamError] = useState<string | null>(null);

  // Populated when the query endpoint responds 424 because one or more of
  // the session's databases isn't connected. Cleared whenever the session
  // changes or the missing databases get reconnected.
  const [queryMissingDbIds, setQueryMissingDbIds] = useState<string[]>([]);

  // --- Follow-up questions state (shown after the latest assistant reply,
  // and also on initial load for sessions that already have a conversation) ---
  const [followups, setFollowups] = useState<string[]>([]);
  const [followupsLoading, setFollowupsLoading] = useState(false);

  const [dbSliderOpen, setDbSliderOpen] = useState(false);
  const [sessionSliderOpen, setSessionSliderOpen] = useState(false);

  // Tracks the session any in-flight async work (stream, follow-ups, message
  // refresh) belongs to. Every async callback checks this before touching
  // state, so results for a session the user has since navigated away from
  // never leak into the currently active session's view.
  const activeSessionRef = useRef<string | null>(null);

  const loadData = useCallback(async () => {
    setLoading(true);
    setLoadError(null);
    try {
      const [dbList, sessionList] = await Promise.all([api.listDatabases(), api.listSessions()]);
      setDatabases(dbList);
      setSessions(sessionList);
      setActiveSessionId((prev) => prev ?? (sessionList.length > 0 ? sessionList[0].id : null));
    } catch (err) {
      setLoadError(err instanceof Error ? err.message : 'Failed to load data.');
    } finally {
      setLoading(false);
    }
  }, []);

  useEffect(() => {
    loadData();
  }, [loadData]);

  // Load chat history whenever the active session changes, and — depending
  // on whether it already has a conversation — either show starter prompts
  // (empty session) or fetch follow-up questions right away (existing session).
  useEffect(() => {
    activeSessionRef.current = activeSessionId;

    // Clear everything belonging to the previous session immediately and
    // synchronously — this is what stops old graphs/messages from a prior
    // session showing up for a beat before the new session's data arrives.
    setMessages([]);
    setSuggestedPrompts([]);
    setSuggestedPromptsError(null);
    setFollowups([]);
    setFollowupsLoading(false);
    setStreamSteps([]);
    setStreamError(null);
    setIsStreaming(false);
    setQueryMissingDbIds([]);

    if (!activeSessionId) {
      setMessagesLoading(false);
      return;
    }

    const sessionId = activeSessionId;
    const isStale = () => activeSessionRef.current !== sessionId;

    setMessagesLoading(true);
    setMessagesError(null);

    api
      .getMessages(sessionId)
      .then((msgs) => {
        if (isStale()) return;
        setMessages(msgs);

        if (msgs.length === 0) {
          // No conversation yet — show starter prompts.
          setSuggestedPromptsLoading(true);
          api
            .getSuggestedPrompts(sessionId)
            .then((res) => {
              if (!isStale()) setSuggestedPrompts(res.prompts);
            })
            .catch((err) => {
              if (!isStale()) {
                setSuggestedPromptsError(
                  err instanceof Error ? err.message : 'Failed to load suggested prompts.'
                );
              }
            })
            .finally(() => {
              if (!isStale()) setSuggestedPromptsLoading(false);
            });
        } else {
          // Existing conversation — show follow-up questions by default.
          setFollowupsLoading(true);
          api
            .getFollowups(sessionId)
            .then((res) => {
              if (!isStale()) setFollowups(res.followups);
            })
            .catch(() => {
              // Follow-ups are a nice-to-have; fail silently.
            })
            .finally(() => {
              if (!isStale()) setFollowupsLoading(false);
            });
        }
      })
      .catch((err) => {
        if (!isStale()) {
          setMessagesError(err instanceof Error ? err.message : 'Failed to load messages.');
          setMessages([]);
        }
      })
      .finally(() => {
        if (!isStale()) setMessagesLoading(false);
      });
  }, [activeSessionId]);

  const activeSession = sessions.find((s) => s.id === activeSessionId) ?? null;

  // Databases this session depends on that are currently disconnected —
  // checked proactively from the session's db_ids on load, and reactively
  // whenever the query endpoint reports missing connections via a 424.
  const disconnectedSessionDbs: DatabaseItem[] = activeSession
    ? databases.filter(
        (db) =>
          (activeSession.db_ids.includes(db.id) || queryMissingDbIds.includes(db.id)) &&
          !db.connected &&
          db.db_type !== 'csv'
      )
    : [];
  const isBlockedByDisconnectedDb = disconnectedSessionDbs.length > 0;

  // Column 3 shows every graph produced across the session's messages, in order.
  // Column 3 shows only the graphs from the most recent assistant response —
  // not every graph accumulated across the whole session's history. This
  // applies the same way whether the session is brand new or has a long
  // existing conversation: whichever assistant message is last wins.
  const latestAssistantMessage = [...messages].reverse().find((msg) => msg.role === 'assistant');
  const charts: ChartResult[] = latestAssistantMessage?.graphs ?? [];

  const handleSelectSession = (id: string) => {
    if (isStreaming) return;
    setActiveSessionId(id);
  };

  const handleSendMessage = async (text: string) => {
    const sessionId = activeSessionId;
    if (!sessionId || isStreaming || isBlockedByDisconnectedDb) return;

    const isStale = () => activeSessionRef.current !== sessionId;

    const userMsg: ChatMessage = {
      id: `local-${Date.now()}`,
      role: 'user',
      content: text,
      graphs: [],
      created_at: new Date().toISOString(),
    };
    setMessages((prev) => [...prev, userMsg]);
    // Once a message is sent, starter prompts and any previous followups no longer apply.
    setSuggestedPrompts([]);
    setFollowups([]);

    setIsStreaming(true);
    setStreamSteps([]);
    setStreamError(null);

    let sawMessageEvent = false;
    let streamFailed = false;

    // Upserts the single in-progress assistant bubble with the latest text
    // seen so far, so the response prints incrementally as events arrive
    // instead of only appearing once the whole stream finishes.
    const upsertStreamingMessage = (text: string) => {
      setMessages((prev) => {
        const withoutProvisional = prev.filter((m) => m.id !== STREAM_MESSAGE_ID);
        return [
          ...withoutProvisional,
          {
            id: STREAM_MESSAGE_ID,
            role: 'assistant',
            content: text,
            graphs: [],
            created_at: new Date().toISOString(),
          },
        ];
      });
    };

    try {
      for await (const event of api.streamQuery(sessionId, text)) {
        if (isStale()) return;

        if (event.type === 'step') {
          setStreamSteps((prev) => [...prev, event.step]);
        } else if (event.type === 'message') {
          sawMessageEvent = true;
          upsertStreamingMessage(event.step);
        } else if (event.type === 'error') {
          streamFailed = true;
          setStreamError(event.message);
        } else if (event.type === 'done') {
          break;
        }
      }
    } catch (err) {
      if (!isStale()) {
        streamFailed = true;
        if (err instanceof MissingDbConnectionError) {
          setQueryMissingDbIds(err.missingDbIds);
          setStreamError(null);
          // The query never actually ran — remove the optimistic user
          // message so the thread doesn't imply it was processed.
          setMessages((prev) => prev.filter((m) => m.id !== userMsg.id));
        } else {
          setStreamError(err instanceof Error ? err.message : 'Something went wrong while running your query.');
        }
      }
    }

    if (isStale()) return;

    setIsStreaming(false);
    setStreamSteps([]);

    if (!sawMessageEvent || streamFailed) {
      // Nothing usable streamed back — drop the placeholder bubble if present.
      setMessages((prev) => prev.filter((m) => m.id !== STREAM_MESSAGE_ID));
    } else {
      // Reconcile with the server's message list to swap the provisional
      // bubble for the real message (real id + any graphs attached to it,
      // since the stream itself doesn't carry graphs).
      try {
        const freshMessages = await api.getMessages(sessionId);
        if (!isStale()) setMessages(freshMessages);
      } catch {
        // Keep the provisional message on screen if the refresh fails — the
        // user still sees the answer, just without graphs until they retry.
      }
    }

    if (isStale() || streamFailed || !sawMessageEvent) return;

    // Once the stream completes, fetch fresh follow-up question suggestions.
    setFollowupsLoading(true);
    try {
      const res = await api.getFollowups(sessionId);
      if (!isStale()) setFollowups(res.followups);
    } catch {
      // Follow-ups are a nice-to-have; fail silently rather than blocking the chat.
    } finally {
      if (!isStale()) setFollowupsLoading(false);
    }
  };

  const handleDatabaseCreated = (db: DatabaseItem) => {
    setDatabases((prev) => [...prev, db]);
  };

  const handleDatabaseConnected = (dbId: string, cacheUntil: string) => {
    setDatabases((prev) =>
      prev.map((db) => (db.id === dbId ? { ...db, connected: true, cache_until: cacheUntil } : db))
    );
  };

  const handleDatabaseDisconnected = (dbId: string) => {
    setDatabases((prev) =>
      prev.map((db) => (db.id === dbId ? { ...db, connected: false, cache_until: null } : db))
    );
  };

  const handleSessionCreated = (session: SessionItem) => {
    setSessions((prev) => [session, ...prev]);
    setActiveSessionId(session.id);
    setSessionSliderOpen(false);
  };

  return (
    <div className={styles['db-analytics']}>
      <header className={styles['db-analytics__feature-header']}>
        <div className={styles['db-analytics__feature-header-main']}>
          <span className={styles['db-analytics__feature-header-icon']}>
            <Icon.databaseInsights size={18} aria-hidden="true" />
          </span>
          <div className={styles['db-analytics__feature-header-text']}>
            <h1 className={styles['db-analytics__feature-header-title']}>DB Analytics</h1>
            <p className={styles['db-analytics__feature-header-subtitle']}>
              Ask questions about your databases in natural language and get instant charts and insights.
            </p>
          </div>
        </div>
        <span className={styles['db-analytics__feature-header-badge']}>
          <span className={styles['db-analytics__feature-header-badge-dot']} aria-hidden="true" />
          <span className={styles['db-analytics__feature-header-badge-text']}>
            {databases.filter((db) => db.connected).length} database
            {databases.filter((db) => db.connected).length !== 1 ? 's' : ''} connected
          </span>
        </span>
      </header>


      <main className={styles['db-analytics__body']}>
        <section className={`${styles['db-analytics__col']} ${styles['db-analytics__col--nav']}`}>
          <SessionHistory
            sessions={sessions}
            activeSessionId={activeSessionId}
            onSelect={handleSelectSession}
            onNewSession={() => setSessionSliderOpen(true)}
            disabled={isStreaming}
          />
          <DatabaseList databases={databases} onManage={() => setDbSliderOpen(true)} disabled={isStreaming} />
        </section>

        <section className={`${styles['db-analytics__col']} ${styles['db-analytics__col--chat']}`}>
          <ChatPanel
            sessionTitle={activeSession?.name ?? (loading ? 'Loading…' : 'No session selected')}
            messages={messages}
            loading={messagesLoading}
            hasActiveSession={!!activeSessionId}
            suggestedPrompts={suggestedPrompts}
            suggestedPromptsLoading={suggestedPromptsLoading}
            suggestedPromptsError={suggestedPromptsError}
            isStreaming={isStreaming}
            streamSteps={streamSteps}
            streamError={streamError}
            followups={followups}
            followupsLoading={followupsLoading}
            disconnectedDatabases={disconnectedSessionDbs}
            onManageDatabases={() => setDbSliderOpen(true)}
            onSend={handleSendMessage}
          />
        </section>

        <section className={`${styles['db-analytics__col']} ${styles['db-analytics__col--results']}`}>
          <ResultsPanel charts={charts} loadError={loadError ?? messagesError} isGenerating={isStreaming} />
        </section>
      </main>

      <DatabaseManagerSlider
        open={dbSliderOpen}
        databases={databases}
        onClose={() => setDbSliderOpen(false)}
        onDatabaseCreated={handleDatabaseCreated}
        onDatabaseConnected={handleDatabaseConnected}
        onDatabaseDisconnected={handleDatabaseDisconnected}
      />

      <NewSessionSlider
        open={sessionSliderOpen}
        databases={databases}
        onClose={() => setSessionSliderOpen(false)}
        onCreated={handleSessionCreated}
        onManageDatabases={() => {
          setSessionSliderOpen(false);
          setDbSliderOpen(true);
        }}
      />
    </div>
  );
};

export default DBAnalytics;







// icons.tsx


// Central place for every SVG icon used across the DBAnalytics feature.
// Swaps the old Tabler icon-font (`<i className="ti ti-x" />`) for real
// inline SVG components from lucide-react.
import React from 'react';
import {
  Plus,
  X,
  Check,
  Database,
  DatabaseZap,
  Settings,
  Info,
  Loader2,
  MessageCircle,
  Sparkles,
  BarChart3,
  Send,
  AreaChart,
  Gauge,
  LineChart,
  PieChart,
  Table,
  ScatterChart,
  Lightbulb,
  ArrowRight,
  Upload,
  FileSpreadsheet,
  Maximize2,
  type LucideProps,
} from 'lucide-react';

export type IconComponent = React.FC<LucideProps>;

export const Icon = {
  plus: Plus,
  close: X,
  check: Check,
  database: Database,
  databaseInsights: DatabaseZap,
  settings: Settings,
  infoCircle: Info,
  loader: Loader2,
  messageCircle: MessageCircle,
  sparkles: Sparkles,
  chartBar: BarChart3,
  send: Send,
  chartAreaLine: AreaChart,
  gauge: Gauge,
  chartLine: LineChart,
  chartPie: PieChart,
  chartArea: AreaChart,
  chartDots: ScatterChart,
  table: Table,
  lightbulb: Lightbulb,
  arrowRight: ArrowRight,
  upload: Upload,
  csv: FileSpreadsheet,
  expand: Maximize2,
} as const;

export type IconName = keyof typeof Icon;
