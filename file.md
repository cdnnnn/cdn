//ChatPanel.module.scss
@use '../../../styles/tokens' as t;

.db-analytics-chat-panel {
  background: t.$surface-2;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  flex: 1;
  min-height: 0;
}

.db-analytics-chat-panel__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 53px;
  padding: 0 16px;
  border-bottom: 1px solid t.$border-strong;
  background: t.$surface-1;
  flex-shrink: 0;
}

.db-analytics-chat-panel__header-text {
  display: flex;
  flex-direction: column;
  justify-content: center;
  gap: 1px;
  min-width: 0;
}

.db-analytics-chat-panel__title {
  font-size: 14px;
  font-weight: 500;
  margin: 0;
  color: t.$text-primary;
  line-height: 1.3;
}

.db-analytics-chat-panel__subtitle {
  font-size: 11px;
  color: t.$text-muted;
  margin: 0;
  line-height: 1.3;
}

.db-analytics-chat-panel__thread {
  display: flex;
  flex-direction: column;
  gap: 12px;
  padding: 12px;
  flex: 1;
  min-height: 0;
  min-width: 0;
  overflow-y: auto;
  overflow-x: hidden;
}

.db-analytics-chat-panel__empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 8px;
  padding: 48px 16px;
  color: t.$text-muted;
  text-align: center;
  flex: 1;

  p {
    font-size: 13px;
    margin: 0;
    max-width: 220px;
  }
}

.db-analytics-chat-bubble {
  display: flex;
  gap: 8px;
  max-width: 92%;
  min-width: 0;
}

.db-analytics-chat-bubble--user {
  align-self: flex-end;
  flex-direction: row-reverse;

  .db-analytics-chat-bubble__content {
    background: t.$accent-bg;
    border: 1px solid rgba(79, 70, 229, 0.14);
  }

  .db-analytics-chat-bubble__text {
    color: t.$text-primary;
  }
}

.db-analytics-chat-bubble--assistant {
  align-self: flex-start;
}

.db-analytics-chat-bubble__avatar {
  width: 26px;
  height: 26px;
  border-radius: 50%;
  background: #eeedfe;
  color: #3c3489;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 13px;
  flex-shrink: 0;
  margin-top: 2px;
}

.db-analytics-chat-bubble__content {
  background: t.$surface-0;
  border-radius: t.$radius-lg;
  padding: 10px 12px;
  min-width: 0;
}

.db-analytics-chat-bubble__text {
  font-size: 13px;
  line-height: 1.6;
  margin: 0;
  color: t.$text-primary;
}

.db-analytics-chat-bubble__markdown {
  font-size: 13px;
  line-height: 1.6;
  color: t.$text-primary;

  > *:first-child {
    margin-top: 0;
  }

  > *:last-child {
    margin-bottom: 0;
  }

  p {
    margin: 0 0 8px;
  }

  h1,
  h2,
  h3,
  h4 {
    font-weight: 600;
    line-height: 1.35;
    margin: 14px 0 6px;
  }

  h1 {
    font-size: 17px;
  }

  h2 {
    font-size: 15px;
  }

  h3,
  h4 {
    font-size: 13.5px;
  }

  ul,
  ol {
    margin: 0 0 8px;
    padding-left: 20px;
  }

  li {
    margin: 2px 0;
  }

  li > p {
    margin: 0;
  }

  strong {
    font-weight: 600;
  }

  em {
    font-style: italic;
  }

  a {
    color: t.$accent;
    text-decoration: underline;

    &:hover {
      color: #185fa5;
    }
  }

  blockquote {
    margin: 8px 0;
    padding: 4px 12px;
    border-left: 3px solid t.$border-strong;
    color: t.$text-secondary;
  }

  code {
    font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
    font-size: 12px;
    background: t.$surface-1;
    border: 0.5px solid t.$border;
    border-radius: 4px;
    padding: 1px 5px;
  }

  pre {
    margin: 8px 0;
    padding: 10px 12px;
    background: t.$surface-1;
    border: 0.5px solid t.$border-strong;
    border-radius: t.$radius;
    overflow-x: auto;

    code {
      background: none;
      border: none;
      padding: 0;
      font-size: 12px;
    }
  }

  hr {
    border: none;
    border-top: 0.5px solid t.$border;
    margin: 12px 0;
  }

  table {
    display: block;
    width: max-content;
    max-width: 100%;
    overflow-x: auto;
    border-collapse: collapse;
    margin: 8px 0;
    font-size: 12px;

    // A visible, non-intrusive scrollbar so it's clear the table itself
    // scrolls when it's too wide, rather than the page silently clipping it.
    scrollbar-width: thin;
    scrollbar-color: t.$border-stronger transparent;

    &::-webkit-scrollbar {
      height: 6px;
    }

    &::-webkit-scrollbar-track {
      background: transparent;
    }

    &::-webkit-scrollbar-thumb {
      background: t.$border-stronger;
      border-radius: 999px;
    }
  }

  th,
  td {
    text-align: left;
    padding: 6px 8px;
    border: 0.5px solid t.$border-strong;
  }

  th {
    background: t.$surface-1;
    font-weight: 600;
  }
}

.db-analytics-chat-bubble__chart-ref {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 11px;
  color: t.$text-secondary;
  margin-top: 8px;
  padding-top: 8px;
  border-top: 0.5px solid t.$border;
}

.db-analytics-db-notice {
  display: flex;
  align-items: flex-start;
  gap: 10px;
  margin: 0 12px;
  padding: 12px 14px;
  border: 1px solid rgba(237, 161, 0, 0.35);
  border-radius: t.$radius-lg;
  background: #fdf6e8;
  flex-shrink: 0;
}

.db-analytics-db-notice__icon {
  width: 26px;
  height: 26px;
  border-radius: 50%;
  background: #f7e6bd;
  color: #8a5a00;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  margin-top: 1px;
}

.db-analytics-db-notice__body {
  flex: 1;
  min-width: 0;
}

.db-analytics-db-notice__title {
  font-size: 12.5px;
  font-weight: 600;
  color: #6b4400;
  margin: 0 0 2px;
}

.db-analytics-db-notice__desc {
  font-size: 12px;
  color: #8a5a00;
  margin: 0;
  line-height: 1.5;

  strong {
    font-weight: 600;
  }
}

.db-analytics-db-notice__action {
  align-self: center;
  flex-shrink: 0;
  font-size: 12px;
  font-weight: 600;
  color: #6b4400;
  background: #fff;
  border: 1px solid rgba(237, 161, 0, 0.4);
  padding: 7px 13px;
  border-radius: t.$radius;
  cursor: pointer;
  font-family: inherit;
  white-space: nowrap;
  transition: background 0.15s ease, border-color 0.15s ease;

  &:hover {
    background: #fef8ea;
    border-color: rgba(237, 161, 0, 0.6);
  }
}

.db-analytics-chat-composer {
  border-top: 0.5px solid t.$border;
  padding: 12px;
  flex-shrink: 0;
}

.db-analytics-chat-composer__box {
  border: 0.5px solid t.$border-strong;
  border-radius: t.$radius-lg;
  background: t.$surface-1;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;

  &:focus-within {
    border-color: t.$accent;
    box-shadow: 0 0 0 2px rgba(42, 120, 214, 0.15);
  }

  &:has(.db-analytics-chat-composer__input:disabled) {
    opacity: 0.6;
    background: t.$surface-0;
  }
}

.db-analytics-chat-composer__input {
  display: block;
  width: 100%;
  box-sizing: border-box;
  resize: none;
  border: none;
  border-radius: t.$radius-lg t.$radius-lg 0 0;
  padding: 10px 12px 4px;
  font-size: 13px;
  font-family: inherit;
  color: t.$text-primary;
  background: transparent;

  &:focus {
    outline: none;
  }

  &::placeholder {
    color: t.$text-muted;
  }

  &:disabled {
    cursor: not-allowed;
  }
}

.db-analytics-chat-composer__box-footer {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  padding: 4px 8px 8px 12px;
}

.db-analytics-chat-composer__hint {
  font-size: 11px;
  color: t.$text-muted;
  min-width: 0;
}

.db-analytics-chat-composer__hint--pending {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  color: t.$accent;
  font-weight: 500;
}

.db-analytics-chat-composer__send-btn {
  display: flex;
  align-items: center;
  gap: 6px;
  background: t.$gradient-primary;
  color: #fff;
  border: none;
  border-radius: t.$radius;
  padding: 7px 14px;
  font-size: 13px;
  font-weight: 500;
  font-family: inherit;
  cursor: pointer;
  box-shadow: 0 1px 2px rgba(79, 70, 229, 0.25);
  flex-shrink: 0;

  &:hover:not(:disabled) {
    filter: brightness(1.08);
    box-shadow: 0 2px 6px rgba(79, 70, 229, 0.35);
  }

  &:active:not(:disabled) {
    filter: brightness(0.96);
  }

  &:disabled {
    cursor: not-allowed;
    opacity: 0.55;
    box-shadow: none;
  }
}

.db-analytics-chat-panel__spin {
  animation: db-analytics-chat-panel-spin 0.8s linear infinite;
}

@keyframes db-analytics-chat-panel-spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}











//DatabaseList.module.scss
@use '../../../styles/tokens' as t;

.db-analytics-db-panel {
  background: t.$surface-2;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  flex: 1;
  min-height: 0;
}

.db-analytics-db-panel__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 53px;
  padding: 0 16px;
  border-bottom: 1px solid t.$border-strong;
  background: t.$surface-1;
  flex-shrink: 0;
}

.db-analytics-db-panel__title {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 14px;
  font-weight: 500;
  margin: 0;
  color: t.$text-primary;
}

.db-analytics-db-panel__refresh-spin {
  color: t.$text-muted;
  animation: db-analytics-db-panel-refresh-spin 0.8s linear infinite;
}

@keyframes db-analytics-db-panel-refresh-spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

.db-analytics-db-panel__header-actions {
  display: flex;
  align-items: center;
  gap: 6px;
}

.db-analytics-db-panel__icon-btn {
  width: 28px;
  height: 28px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border-radius: t.$radius;
  border: 0.5px solid t.$border-strong;
  background: transparent;
  color: t.$text-secondary;
  cursor: pointer;
  font-size: 15px;
  flex-shrink: 0;

  &:hover:not(:disabled) {
    background: t.$surface-0;
    border-color: t.$border-strong;
  }

  &:disabled {
    opacity: 0.45;
    cursor: not-allowed;
  }
}

.db-analytics-db-panel__body {
  padding: 8px;
  flex: 1;
  min-height: 0;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
}

.db-analytics-db-panel__empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  padding: 32px 16px;
  gap: 3px;

  p {
    font-size: 13px;
    font-weight: 600;
    color: t.$text-primary;
    margin: 4px 0 2px;
  }
}

.db-analytics-db-panel__empty-icon {
  width: 34px;
  height: 34px;
  border-radius: 50%;
  background: t.$surface-0;
  color: t.$text-muted;
  display: flex;
  align-items: center;
  justify-content: center;
}

.db-analytics-db-panel__link {
  background: none;
  border: none;
  padding: 0;
  color: t.$accent;
  font-size: 12px;
  font-weight: 600;
  font-family: inherit;
  cursor: pointer;

  &:hover:not(:disabled) {
    color: #185fa5;
    text-decoration: underline;
  }

  &:disabled {
    opacity: 0.55;
    cursor: not-allowed;
    text-decoration: none;
  }
}

.db-analytics-db-list {
  list-style: none;
  margin: 0;
  padding: 0;
  display: flex;
  flex-direction: column;
  gap: 3px;
}

.db-analytics-db-item {
  display: flex;
  flex-direction: column;
  gap: 4px;
  padding: 7px 8px;
  border-radius: t.$radius;
  transition: background 0.15s ease;

  &:hover {
    background: t.$surface-0;
  }
}

.db-analytics-db-item__row {
  display: flex;
  align-items: center;
  gap: 10px;
}

.db-analytics-db-item__connect-row {
  display: flex;
  justify-content: flex-end;
  padding-left: 40px;
}

.db-analytics-db-item__connect-btn {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-size: 10.5px;
  font-weight: 600;
  padding: 3px 9px;
  border-radius: 999px;
  border: 0.5px solid t.$border-strong;
  background: t.$surface-2;
  color: t.$text-secondary;
  cursor: pointer;
  font-family: inherit;
  transition: background 0.12s ease, border-color 0.12s ease, color 0.12s ease;

  &:hover:not(:disabled) {
    background: t.$surface-2;
    border-color: t.$border-stronger;
    color: t.$text-primary;
    box-shadow: 0 1px 2px rgba(11, 11, 11, 0.06);
  }

  &:disabled {
    opacity: 0.55;
    cursor: not-allowed;
  }
}

.db-analytics-db-item__connect-btn--primary {
  background: t.$gradient-primary;
  color: #fff;
  border-color: transparent;

  &:hover:not(:disabled) {
    filter: brightness(1.08);
    color: #fff;
    border-color: transparent;
  }
}

.db-analytics-db-item__spin {
  animation: db-analytics-db-item-spin 0.8s linear infinite;
}

@keyframes db-analytics-db-item-spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

.db-analytics-db-item__icon {
  width: 30px;
  height: 30px;
  border-radius: 7px;
  background: t.$indigo-bg;
  color: t.$indigo-fg;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.db-analytics-db-item__icon--mysql {
  background: t.$blue-bg;
  color: t.$blue-fg;
}

.db-analytics-db-item__icon--postgres {
  background: t.$violet-bg;
  color: t.$violet-fg;
}

.db-analytics-db-item__icon--mssql {
  background: t.$amber-bg;
  color: t.$amber-fg;
}

.db-analytics-db-item__icon--csv {
  background: t.$green-bg;
  color: t.$green-fg;
}

.db-analytics-db-item__info {
  display: flex;
  flex-direction: column;
  gap: 3px;
  min-width: 0;
  flex: 1;
}

.db-analytics-db-item__name {
  font-size: 12.5px;
  font-weight: 600;
  color: t.$text-primary;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.db-analytics-db-item__meta {
  display: flex;
  align-items: center;
  gap: 6px;
  min-width: 0;
}

.db-analytics-db-item__type-badge {
  font-size: 9px;
  font-weight: 700;
  letter-spacing: 0.02em;
  text-transform: uppercase;
  padding: 1px 5px;
  border-radius: 4px;
  flex-shrink: 0;
  color: t.$text-secondary;
  background: t.$surface-0;
  border: 0.5px solid t.$border;
}

.db-analytics-db-item__type-badge--mysql {
  color: t.$blue-fg;
  background: t.$blue-bg;
  border-color: transparent;
}

.db-analytics-db-item__type-badge--postgres {
  color: t.$violet-fg;
  background: t.$violet-bg;
  border-color: transparent;
}

.db-analytics-db-item__type-badge--mssql {
  color: t.$amber-fg;
  background: t.$amber-bg;
  border-color: transparent;
}

.db-analytics-db-item__type-badge--csv {
  color: t.$green-fg;
  background: t.$green-bg;
  border-color: transparent;
}

.db-analytics-db-item__db-name {
  font-size: 10.5px;
  color: t.$text-muted;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.db-analytics-db-status-dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  flex-shrink: 0;
  box-shadow: 0 0 0 2px t.$surface-2;
}

.db-analytics-db-status-dot--connected {
  background: t.$success;
}

.db-analytics-db-status-dot--disconnected {
  background: t.$danger;
}

.db-analytics-db-item__actions {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-shrink: 0;
}

.db-analytics-db-item__schema-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 22px;
  height: 22px;
  border-radius: 6px;
  border: 0.5px solid t.$border-strong;
  background: transparent;
  color: t.$text-muted;
  cursor: pointer;
  flex-shrink: 0;
  transition: background 0.15s ease, border-color 0.15s ease, color 0.15s ease;

  &:hover:not(:disabled) {
    background: t.$surface-2;
    border-color: t.$border-stronger;
    color: t.$accent;
  }

  &:disabled {
    opacity: 0.35;
    cursor: not-allowed;
  }
}


















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

const axisTick = { fontSize: 11.5, fill: '#8b8a84', fontWeight: 500 };
const axisLine = { stroke: '#d8d7cf' };

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
        <div className={styles['db-analytics-chart-card__no-data']}>{i18n.t('db_analytics.resultsPanel.noSeriesFound')}</div>
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
