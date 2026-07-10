//ChartConstructingAnimation.module.scss
@use '../../../styles/tokens' as t;

.db-analytics-chart-draw {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 14px;
  padding: 24px;
  height: 100%;
  min-height: 260px;
}

.db-analytics-chart-draw__svg {
  width: 100%;
  max-width: 300px;
  height: auto;
  overflow: visible;
}

.db-analytics-chart-draw__erase-group {
  animation: db-analytics-chart-draw-erase var(--cycle-duration, 3.5s) ease-in-out infinite;
  transform-origin: 115px 150px;
}

@keyframes db-analytics-chart-draw-erase {
  0% {
    opacity: 0;
  }
  4% {
    opacity: 1;
  }
  78% {
    opacity: 1;
    transform: scale(1);
  }
  92% {
    opacity: 0;
    transform: scale(0.94);
  }
  100% {
    opacity: 0;
    transform: scale(0.94);
  }
}

.db-analytics-chart-draw__axis {
  stroke: t.$border-strong;
  stroke-width: 1.5;
}

.db-analytics-chart-draw__bar {
  fill: t.$accent;
  fill-opacity: 0.16;
  stroke: t.$accent;
  stroke-width: 1.5;
  animation: db-analytics-chart-draw-bar var(--cycle-duration, 3.5s) cubic-bezier(0.22, 1, 0.36, 1) infinite;
}

.db-analytics-chart-draw__bar:nth-of-type(1) {
  animation-name: db-analytics-chart-draw-bar-1;
}
.db-analytics-chart-draw__bar:nth-of-type(2) {
  animation-name: db-analytics-chart-draw-bar-2;
}
.db-analytics-chart-draw__bar:nth-of-type(3) {
  animation-name: db-analytics-chart-draw-bar-3;
}
.db-analytics-chart-draw__bar:nth-of-type(4) {
  animation-name: db-analytics-chart-draw-bar-4;
}

@keyframes db-analytics-chart-draw-bar-1 {
  0%,
  2% {
    transform: scaleY(0);
  }
  16%,
  100% {
    transform: scaleY(1);
  }
}
@keyframes db-analytics-chart-draw-bar-2 {
  0%,
  6% {
    transform: scaleY(0);
  }
  20%,
  100% {
    transform: scaleY(1);
  }
}
@keyframes db-analytics-chart-draw-bar-3 {
  0%,
  10% {
    transform: scaleY(0);
  }
  24%,
  100% {
    transform: scaleY(1);
  }
}
@keyframes db-analytics-chart-draw-bar-4 {
  0%,
  14% {
    transform: scaleY(0);
  }
  28%,
  100% {
    transform: scaleY(1);
  }
}

.db-analytics-chart-draw__line {
  fill: none;
  stroke: #4f46e5;
  stroke-width: 2;
  stroke-linecap: round;
  stroke-linejoin: round;
  stroke-dasharray: 260;
  animation: db-analytics-chart-draw-line var(--cycle-duration, 3.5s) ease-out infinite;
}

@keyframes db-analytics-chart-draw-line {
  0%,
  32% {
    stroke-dashoffset: 260;
    opacity: 0;
  }
  34% {
    opacity: 1;
  }
  56%,
  100% {
    stroke-dashoffset: 0;
    opacity: 1;
  }
}

.db-analytics-chart-draw__dot {
  fill: #4f46e5;
  transform-box: fill-box;
  transform-origin: center;
  animation: db-analytics-chart-draw-dot var(--cycle-duration, 3.5s) cubic-bezier(0.34, 1.56, 0.64, 1) infinite;
}

.db-analytics-chart-draw__dot:nth-of-type(1) {
  animation-name: db-analytics-chart-draw-dot-1;
}
.db-analytics-chart-draw__dot:nth-of-type(2) {
  animation-name: db-analytics-chart-draw-dot-2;
}
.db-analytics-chart-draw__dot:nth-of-type(3) {
  animation-name: db-analytics-chart-draw-dot-3;
}
.db-analytics-chart-draw__dot:nth-of-type(4) {
  animation-name: db-analytics-chart-draw-dot-4;
}

@keyframes db-analytics-chart-draw-dot-1 {
  0%,
  33% {
    opacity: 0;
    transform: scale(0);
  }
  38%,
  100% {
    opacity: 1;
    transform: scale(1);
  }
}
@keyframes db-analytics-chart-draw-dot-2 {
  0%,
  40% {
    opacity: 0;
    transform: scale(0);
  }
  45%,
  100% {
    opacity: 1;
    transform: scale(1);
  }
}
@keyframes db-analytics-chart-draw-dot-3 {
  0%,
  47% {
    opacity: 0;
    transform: scale(0);
  }
  52%,
  100% {
    opacity: 1;
    transform: scale(1);
  }
}
@keyframes db-analytics-chart-draw-dot-4 {
  0%,
  54% {
    opacity: 0;
    transform: scale(0);
  }
  59%,
  100% {
    opacity: 1;
    transform: scale(1);
  }
}

.db-analytics-chart-draw__caption {
  margin: 0;
  font-size: 12.5px;
  font-weight: 500;
  color: t.$text-muted;
  animation: db-analytics-chart-draw-caption-pulse 1.6s ease-in-out infinite;
}

@keyframes db-analytics-chart-draw-caption-pulse {
  0%,
  100% {
    opacity: 0.55;
  }
  50% {
    opacity: 1;
  }
}













//ChartConstructingAnimation.tsx
import React from 'react';
import styles from './ChartConstructingAnimation.module.scss';

// Four bars plus a line series, drawn on the same plot, on a continuous
// loop: bars grow from the baseline (staggered), then a line draws itself
// across their tops with a point "popping in" at each vertex, everything
// holds briefly, fades/shrinks out, and the cycle restarts. All pieces
// share one CSS custom property (--cycle-duration) with matching
// percentage-based keyframes (see the .module.scss) and
// `animation-iteration-count: infinite`, so every repeat stays in sync.
const BAR_X = [40, 90, 140, 190];
const BAR_HEIGHTS = [70, 110, 55, 90];
const BASELINE_Y = 150;
const LINE_POINTS = [
  { x: 40, y: 60 },
  { x: 90, y: 40 },
  { x: 140, y: 85 },
  { x: 190, y: 30 },
];
const BAR_WIDTH = 22;
const CYCLE_DURATION_SECONDS = 3.6;

const ChartConstructingAnimation: React.FC = () => {
  const linePath = LINE_POINTS.map((p, i) => `${i === 0 ? 'M' : 'L'} ${p.x} ${p.y}`).join(' ');
  const cycleDurationVar = { ['--cycle-duration' as string]: `${CYCLE_DURATION_SECONDS}s` };

  return (
    <div className={styles['db-analytics-chart-draw']}>
      <svg viewBox="0 0 230 170" className={styles['db-analytics-chart-draw__svg']}>
        <g className={styles['db-analytics-chart-draw__erase-group']} style={cycleDurationVar}>
          <line
            x1={20}
            y1={BASELINE_Y}
            x2={210}
            y2={BASELINE_Y}
            className={styles['db-analytics-chart-draw__axis']}
          />

          {BAR_X.map((x, i) => {
            const height = BAR_HEIGHTS[i];
            return (
              <rect
                key={i}
                x={x - BAR_WIDTH / 2}
                y={BASELINE_Y - height}
                width={BAR_WIDTH}
                height={height}
                rx={3}
                className={styles['db-analytics-chart-draw__bar']}
                style={{ transformOrigin: `${x}px ${BASELINE_Y}px`, ...cycleDurationVar }}
              />
            );
          })}

          <path d={linePath} className={styles['db-analytics-chart-draw__line']} style={cycleDurationVar} />
          {LINE_POINTS.map((p, i) => (
            <circle
              key={i}
              cx={p.x}
              cy={p.y}
              r={3.5}
              className={styles['db-analytics-chart-draw__dot']}
              style={cycleDurationVar}
            />
          ))}
        </g>
      </svg>
      <p className={styles['db-analytics-chart-draw__caption']}>Drawing your chart…</p>
    </div>
  );
};

export default ChartConstructingAnimation;
















//DatabaseManagerSlider.module.scss
@use '../../../styles/tokens' as t;

.db-analytics-slider-overlay {
  position: fixed;
  inset: 0;
  z-index: 100;
  display: flex;
  justify-content: flex-end;
}

.db-analytics-slider-overlay__backdrop {
  position: absolute;
  inset: 0;
  background: rgba(11, 11, 11, 0.32);
  animation: db-analytics-fade-in 0.15s ease;
}

.db-analytics-slider-panel {
  position: relative;
  width: 600px;
  max-width: 100vw;
  height: 100%;
  background: t.$surface-2;
  display: flex;
  flex-direction: column;
  box-shadow: -8px 0 24px rgba(11, 11, 11, 0.12);
  animation: db-analytics-slide-in-right 0.22s cubic-bezier(0.16, 1, 0.3, 1);
}

.db-analytics-slider-panel__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 60px;
  padding: 0 20px;
  border-bottom: 1px solid t.$border-strong;
  background: t.$surface-1;
  flex-shrink: 0;
}

.db-analytics-slider-panel__title {
  font-size: 16px;
  font-weight: 500;
  margin: 0;
}

.db-analytics-slider-panel__close-btn {
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

  &:hover {
    background: t.$surface-0;
  }
}

.db-analytics-slider-panel__body {
  flex: 1;
  overflow-y: auto;
  padding: 20px;
}

@keyframes db-analytics-slide-in-right {
  from {
    transform: translateX(100%);
  }
  to {
    transform: translateX(0);
  }
}

@keyframes db-analytics-fade-in {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

.db-analytics-slider-tabs {
  display: flex;
  gap: 4px;
  padding: 12px 20px 0;
  border-bottom: 1px solid t.$border-strong;
  background: t.$surface-1;
  flex-shrink: 0;
}

.db-analytics-slider-tabs__item {
  display: flex;
  align-items: center;
  gap: 6px;
  background: none;
  border: none;
  font-family: inherit;
  font-size: 13px;
  font-weight: 500;
  color: t.$text-muted;
  padding: 10px 14px;
  cursor: pointer;
  border-bottom: 2px solid transparent;
  margin-bottom: -1px;

  &:hover {
    color: t.$text-primary;
  }
}

.db-analytics-slider-tabs__item--active {
  color: t.$text-primary;
  border-bottom-color: t.$text-primary;
}

.db-analytics-db-manage-list {
  list-style: none;
  margin: 0;
  padding: 0;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.db-analytics-db-manage-item {
  padding: 16px;
  border: 1px solid t.$border-strong;
  border-radius: t.$radius-lg;
  background: t.$surface-2;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;

  &:hover {
    border-color: t.$border-stronger;
    box-shadow: 0 2px 8px rgba(11, 11, 11, 0.05);
  }
}

.db-analytics-db-manage-item__top {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 12px;
}

.db-analytics-db-manage-item__identity {
  display: flex;
  align-items: center;
  gap: 12px;
  min-width: 0;
}

.db-analytics-db-manage-item__icon {
  width: 38px;
  height: 38px;
  border-radius: t.$radius;
  background: t.$indigo-bg;
  color: t.$indigo-fg;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.db-analytics-db-manage-item__icon--mysql {
  background: t.$blue-bg;
  color: t.$blue-fg;
}

.db-analytics-db-manage-item__icon--postgres {
  background: t.$violet-bg;
  color: t.$violet-fg;
}

.db-analytics-db-manage-item__icon--csv {
  background: t.$green-bg;
  color: t.$green-fg;
}

.db-analytics-db-manage-item__name-block {
  display: flex;
  align-items: center;
  gap: 8px;
  min-width: 0;
}

.db-analytics-db-manage-item__name {
  font-size: 14.5px;
  font-weight: 600;
  color: t.$text-primary;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.db-analytics-db-manage-item__type-badge {
  font-size: 10px;
  font-weight: 600;
  letter-spacing: 0.02em;
  text-transform: uppercase;
  color: t.$text-secondary;
  background: t.$surface-0;
  border: 0.5px solid t.$border;
  padding: 2px 7px;
  border-radius: 999px;
  flex-shrink: 0;
}

.db-analytics-db-manage-item__type-badge--mysql {
  color: t.$blue-fg;
  background: t.$blue-bg;
  border-color: transparent;
}

.db-analytics-db-manage-item__type-badge--postgres {
  color: t.$violet-fg;
  background: t.$violet-bg;
  border-color: transparent;
}

.db-analytics-db-manage-item__type-badge--csv {
  color: t.$green-fg;
  background: t.$green-bg;
  border-color: transparent;
}

.db-analytics-db-manage-item__meta-grid {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 12px;
  margin: 14px 0 0;
  padding: 12px 14px;
  background: t.$surface-0;
  border-radius: t.$radius;
}

.db-analytics-db-manage-item__meta-cell {
  min-width: 0;

  dt {
    font-size: 10.5px;
    font-weight: 600;
    letter-spacing: 0.02em;
    text-transform: uppercase;
    color: t.$text-muted;
    margin: 0 0 3px;
  }

  dd {
    font-size: 12.5px;
    color: t.$text-primary;
    margin: 0;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
  }
}

.db-analytics-db-manage-item__footer {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  margin-top: 14px;
  padding-top: 12px;
  border-top: 0.5px solid t.$border;
}

.db-analytics-db-manage-empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  padding: 44px 20px;
  border: 1px dashed t.$border-strong;
  border-radius: t.$radius-lg;
  background: t.$surface-0;
}

.db-analytics-db-manage-empty__icon {
  width: 44px;
  height: 44px;
  border-radius: 50%;
  background: t.$accent-bg;
  color: #0c447c;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 12px;
}

.db-analytics-db-manage-empty__title {
  font-size: 13.5px;
  font-weight: 600;
  color: t.$text-primary;
  margin: 0 0 4px;
}

.db-analytics-db-manage-empty__desc {
  font-size: 12px;
  color: t.$text-muted;
  margin: 0;
  max-width: 280px;
}

.db-analytics-db-status {
  font-size: 11px;
  font-weight: 500;
  display: flex;
  align-items: center;
  gap: 5px;
  white-space: nowrap;
  flex-shrink: 0;
}

.db-analytics-db-status__dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
}

.db-analytics-db-status--connected {
  color: t.$status-connected-fg;

  .db-analytics-db-status__dot {
    background: t.$success;
  }
}

.db-analytics-db-status--disconnected {
  color: t.$status-disconnected-fg;

  .db-analytics-db-status__dot {
    background: t.$danger;
  }
}

.db-analytics-form-card {
  border: 1px solid t.$border-strong;
  border-radius: t.$radius-lg;
  background: t.$surface-2;
  padding: 18px;
}

.db-analytics-form-card__header {
  display: flex;
  align-items: flex-start;
  gap: 12px;
  padding-bottom: 16px;
  margin-bottom: 18px;
  border-bottom: 1px solid t.$border;
}

.db-analytics-form-card__icon {
  width: 34px;
  height: 34px;
  border-radius: t.$radius;
  background: t.$accent-bg;
  color: #0c447c;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.db-analytics-form-card__icon--csv {
  background: #e3f5ea;
  color: #0a7a41;
}

.db-analytics-form-card__title {
  font-size: 14px;
  font-weight: 600;
  color: t.$text-primary;
  margin: 0 0 3px;
}

.db-analytics-form-card__subtitle {
  font-size: 12px;
  color: t.$text-muted;
  margin: 0;
  line-height: 1.5;
}

.db-analytics-file-drop {
  display: flex;
  align-items: center;
  gap: 12px;
  width: 100%;
  padding: 14px;
  border: 1.5px dashed t.$border-strong;
  border-radius: t.$radius;
  background: t.$surface-0;
  cursor: pointer;
  font-family: inherit;
  text-align: left;
  transition: border-color 0.15s ease, background 0.15s ease, box-shadow 0.15s ease, transform 0.15s ease;

  &:hover {
    border-color: t.$accent;
    background: t.$accent-bg;
  }
}

.db-analytics-file-drop--filled {
  border-style: solid;
  border-color: rgba(10, 122, 65, 0.35);
  background: #e3f5ea;

  &:hover {
    border-color: rgba(10, 122, 65, 0.5);
    background: #d7f0e1;
  }
}

.db-analytics-file-drop--dragging {
  border-style: solid;
  border-color: t.$accent;
  background: t.$accent-bg;
  box-shadow: 0 0 0 3px rgba(42, 120, 214, 0.12);
  transform: scale(1.01);

  .db-analytics-file-drop__icon {
    background: t.$accent;
    color: #fff;
  }
}

.db-analytics-file-drop__icon {
  width: 34px;
  height: 34px;
  border-radius: t.$radius;
  background: t.$surface-2;
  color: t.$text-muted;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.db-analytics-file-drop--filled .db-analytics-file-drop__icon {
  background: #fff;
  color: #0a7a41;
}

.db-analytics-file-drop__text {
  display: flex;
  flex-direction: column;
  gap: 2px;
  min-width: 0;
}

.db-analytics-file-drop__title {
  font-size: 13px;
  font-weight: 600;
  color: t.$text-primary;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.db-analytics-file-drop__subtitle {
  font-size: 11.5px;
  color: t.$text-muted;
}

.db-analytics-file-drop__input {
  // Visually hidden but still accessible/functional — the styled button
  // above triggers it via ref, so the native input itself stays off-screen.
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}

.db-analytics-form {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.db-analytics-form__row {
  display: flex;
  gap: 12px;
}

.db-analytics-form__field {
  display: flex;
  flex-direction: column;
  gap: 6px;
  flex: 1;
}

.db-analytics-form__field--narrow {
  flex: 0 0 120px;
}

.db-analytics-form__label {
  font-size: 12px;
  font-weight: 500;
  color: t.$text-secondary;
}

.db-analytics-form__input {
  font-family: inherit;
  font-size: 13px;
  padding: 9px 11px;
  border: 1px solid t.$border-strong;
  border-radius: t.$radius;
  background: t.$surface-1;
  color: t.$text-primary;

  &:focus {
    outline: none;
    border-color: t.$accent;
    box-shadow: 0 0 0 2px rgba(42, 120, 214, 0.15);
  }
}

.db-analytics-form__hint {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 12px;
  color: t.$text-muted;
  margin: 0;
}

.db-analytics-form__error {
  font-size: 12px;
  color: #791f1f;
  background: #fbe9e9;
  padding: 8px 11px;
  border-radius: t.$radius;
}

.db-analytics-form__actions {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-top: 8px;
  border-top: 1px solid t.$border;
}

.db-analytics-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  font-size: 13px;
  font-weight: 500;
  font-family: inherit;
  border-radius: t.$radius;
  padding: 8px 16px;
  cursor: pointer;
  border: 1px solid transparent;
  transition: filter 0.15s ease, background 0.15s ease, border-color 0.15s ease;

  &:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }
}

.db-analytics-btn--primary {
  background: t.$gradient-primary;
  color: #fff;
  box-shadow: 0 1px 2px rgba(79, 70, 229, 0.25);

  &:hover:not(:disabled) {
    filter: brightness(1.08);
    box-shadow: 0 2px 6px rgba(79, 70, 229, 0.35);
  }

  &:active:not(:disabled) {
    filter: brightness(0.96);
  }
}

.db-analytics-btn--ghost {
  background: transparent;
  border-color: t.$border-strong;
  color: t.$text-primary;

  &:hover:not(:disabled) {
    background: t.$surface-0;
  }
}

.db-analytics-btn--sm {
  padding: 5px 11px;
  font-size: 12px;
}

.db-analytics-btn--icon {
  width: 30px;
  height: 30px;
  padding: 0;
  background: t.$surface-0;
  border-color: t.$border-strong;
  color: t.$text-secondary;

  &:hover:not(:disabled) {
    background: t.$surface-2;
    border-color: t.$border-stronger;
    color: t.$text-primary;
  }
}












//DatabaseManagerSlider.tsx
import React, { useRef, useState } from 'react';
import styles from './DatabaseManagerSlider.module.scss';
import { DatabaseItem } from '../types';
import { api, CreateDbPayload } from '../../../services/api';
import ConnectPasswordModal from './ConnectPasswordModal';
import DbSchemaSlider from './DbSchemaSlider';
import { Icon } from '../icons';
import { getDbTypeMeta } from '../dbTypeMeta';

interface Props {
  open: boolean;
  databases: DatabaseItem[];
  onClose: () => void;
  onDatabaseCreated: (db: DatabaseItem) => void;
  onDatabaseConnected: (dbId: string, cacheUntil: string) => void;
  onDatabaseDisconnected: (dbId: string) => void;
}

type Tab = 'existing' | 'new' | 'csv';

const emptyForm: CreateDbPayload = {
  name: '',
  db_name: '',
  db_type: 'mysql',
  host: '',
  port: 3306,
  username: '',
};

const emptyCsvForm = {
  name: '',
  table_name: '',
};

const DatabaseManagerSlider: React.FC<Props> = ({
  open,
  databases,
  onClose,
  onDatabaseCreated,
  onDatabaseConnected,
  onDatabaseDisconnected,
}) => {
  const [tab, setTab] = useState<Tab>('existing');
  const [form, setForm] = useState<CreateDbPayload>(emptyForm);
  const [creating, setCreating] = useState(false);
  const [createError, setCreateError] = useState<string | null>(null);
  const [busyDbId, setBusyDbId] = useState<string | null>(null);
  const [connectTarget, setConnectTarget] = useState<DatabaseItem | null>(null);
  const [schemaTarget, setSchemaTarget] = useState<DatabaseItem | null>(null);

  const [csvForm, setCsvForm] = useState(emptyCsvForm);
  const [csvFile, setCsvFile] = useState<File | null>(null);
  const [csvUploading, setCsvUploading] = useState(false);
  const [csvError, setCsvError] = useState<string | null>(null);
  const [isDraggingCsv, setIsDraggingCsv] = useState(false);
  const fileInputRef = useRef<HTMLInputElement>(null);

  if (!open) return null;

  const updateField = <K extends keyof CreateDbPayload>(key: K, value: CreateDbPayload[K]) => {
    setForm((prev) => ({ ...prev, [key]: value }));
  };

  const handleCreate = async () => {
    setCreateError(null);
    if (!form.name.trim() || !form.db_name.trim() || !form.host.trim() || !form.username.trim()) {
      setCreateError('Please fill in all required fields.');
      return;
    }
    setCreating(true);
    try {
      const created = await api.createDatabase(form);
      onDatabaseCreated(created);
      setForm(emptyForm);
      setTab('existing');
    } catch (err) {
      setCreateError(err instanceof Error ? err.message : 'Failed to create database.');
    } finally {
      setCreating(false);
    }
  };

  const handleDisconnect = async (db: DatabaseItem) => {
    setBusyDbId(db.id);
    try {
      await api.disconnectDatabase(db.id);
      onDatabaseDisconnected(db.id);
    } catch (err) {
      console.error(err);
    } finally {
      setBusyDbId(null);
    }
  };

  const handleFilePick = (fileList: FileList | null) => {
    const file = fileList?.[0];
    if (!file) return;
    if (!file.name.toLowerCase().endsWith('.csv')) {
      setCsvError('Only .csv files are supported.');
      return;
    }
    setCsvError(null);
    setCsvFile(file);
  };

  const handleDragOver = (e: React.DragEvent<HTMLButtonElement>) => {
    e.preventDefault();
    e.stopPropagation();
    if (!isDraggingCsv) setIsDraggingCsv(true);
  };

  const handleDragLeave = (e: React.DragEvent<HTMLButtonElement>) => {
    e.preventDefault();
    e.stopPropagation();
    // Ignore dragleave events fired when moving between child elements of
    // the drop zone — only actually clear the state once the pointer has
    // left the drop zone itself.
    if (e.currentTarget.contains(e.relatedTarget as Node)) return;
    setIsDraggingCsv(false);
  };

  const handleDrop = (e: React.DragEvent<HTMLButtonElement>) => {
    e.preventDefault();
    e.stopPropagation();
    setIsDraggingCsv(false);
    handleFilePick(e.dataTransfer.files);
  };

  const handleCsvUpload = async () => {
    setCsvError(null);
    if (!csvFile) {
      setCsvError('Please select a CSV file to upload.');
      return;
    }
    if (!csvForm.name.trim() || !csvForm.table_name.trim()) {
      setCsvError('Please provide both a name and a table name.');
      return;
    }
    setCsvUploading(true);
    try {
      const created = await api.uploadCsv({
        file: csvFile,
        name: csvForm.name.trim(),
        table_name: csvForm.table_name.trim(),
      });
      onDatabaseCreated(created);
      setCsvForm(emptyCsvForm);
      setCsvFile(null);
      if (fileInputRef.current) fileInputRef.current.value = '';
      setTab('existing');
    } catch (err) {
      setCsvError(err instanceof Error ? err.message : 'Failed to upload CSV file.');
    } finally {
      setCsvUploading(false);
    }
  };

  return (
    <div className={styles['db-analytics-slider-overlay']} role="dialog" aria-modal="true">
      <div className={styles['db-analytics-slider-overlay__backdrop']} onClick={onClose} />
      <aside className={styles['db-analytics-slider-panel']}>
        <div className={styles['db-analytics-slider-panel__header']}>
          <h2 className={styles['db-analytics-slider-panel__title']}>Manage databases</h2>
          <button className={styles['db-analytics-slider-panel__close-btn']} aria-label="Close" onClick={onClose}>
            <Icon.close size={16} aria-hidden="true" />
          </button>
        </div>

        <div className={styles['db-analytics-slider-tabs']}>
          <button
            className={`${styles['db-analytics-slider-tabs__item']} ${
              tab === 'existing' ? styles['db-analytics-slider-tabs__item--active'] : ''
            }`}
            onClick={() => setTab('existing')}
          >
            Existing databases
          </button>
          <button
            className={`${styles['db-analytics-slider-tabs__item']} ${
              tab === 'new' ? styles['db-analytics-slider-tabs__item--active'] : ''
            }`}
            onClick={() => setTab('new')}
          >
            <Icon.plus size={14} aria-hidden="true" />
            New connection
          </button>
          <button
            className={`${styles['db-analytics-slider-tabs__item']} ${
              tab === 'csv' ? styles['db-analytics-slider-tabs__item--active'] : ''
            }`}
            onClick={() => setTab('csv')}
          >
            <Icon.upload size={14} aria-hidden="true" />
            Upload CSV
          </button>
        </div>

        <div className={styles['db-analytics-slider-panel__body']}>
          {tab === 'existing' ? (
            <ul className={styles['db-analytics-db-manage-list']}>
              {databases.length === 0 && (
                <li className={styles['db-analytics-db-manage-empty']}>
                  <span className={styles['db-analytics-db-manage-empty__icon']}>
                    <Icon.database size={22} aria-hidden="true" />
                  </span>
                  <p className={styles['db-analytics-db-manage-empty__title']}>No databases yet</p>
                  <p className={styles['db-analytics-db-manage-empty__desc']}>
                    Create a connection or upload a CSV file to start querying data.
                  </p>
                </li>
              )}
              {databases.map((db, index) => {
                const typeMeta = getDbTypeMeta(db.db_type);
                const isCsv = db.db_type === 'csv';
                return (
                  <li key={db.id ? `${db.id}-${index}` : `db-${index}`} className={styles['db-analytics-db-manage-item']}>
                    <div className={styles['db-analytics-db-manage-item__top']}>
                      <div className={styles['db-analytics-db-manage-item__identity']}>
                        <div
                          className={`${styles['db-analytics-db-manage-item__icon']} ${
                            styles[`db-analytics-db-manage-item__icon--${typeMeta.modifier}`] ?? ''
                          }`}
                        >
                          {isCsv ? (
                            <Icon.csv size={17} aria-hidden="true" />
                          ) : (
                            <Icon.database size={17} aria-hidden="true" />
                          )}
                        </div>
                        <div className={styles['db-analytics-db-manage-item__name-block']}>
                          <span className={styles['db-analytics-db-manage-item__name']}>{db.name}</span>
                          <span
                            className={`${styles['db-analytics-db-manage-item__type-badge']} ${
                              styles[`db-analytics-db-manage-item__type-badge--${typeMeta.modifier}`] ?? ''
                            }`}
                          >
                            {typeMeta.label}
                          </span>
                        </div>
                      </div>
                      {isCsv ? (
                        <span
                          className={`${styles['db-analytics-db-status']} ${styles['db-analytics-db-status--connected']}`}
                        >
                          <span className={styles['db-analytics-db-status__dot']} aria-hidden="true" />
                          Ready
                        </span>
                      ) : (
                        <span
                          className={`${styles['db-analytics-db-status']} ${
                            db.connected
                              ? styles['db-analytics-db-status--connected']
                              : styles['db-analytics-db-status--disconnected']
                          }`}
                        >
                          <span className={styles['db-analytics-db-status__dot']} aria-hidden="true" />
                          {db.connected ? 'Connected' : 'Disconnected'}
                        </span>
                      )}
                    </div>

                    <dl className={styles['db-analytics-db-manage-item__meta-grid']}>
                      <div className={styles['db-analytics-db-manage-item__meta-cell']}>
                        <dt>{isCsv ? 'Table' : 'Database'}</dt>
                        <dd>{db.db_name}</dd>
                      </div>
                      {!isCsv && (
                        <div className={styles['db-analytics-db-manage-item__meta-cell']}>
                          <dt>Port</dt>
                          <dd>{db.port}</dd>
                        </div>
                      )}
                      <div className={styles['db-analytics-db-manage-item__meta-cell']}>
                        <dt>{isCsv ? 'Source' : 'User'}</dt>
                        <dd>{isCsv ? 'CSV upload' : db.username}</dd>
                      </div>
                    </dl>

                    <div className={styles['db-analytics-db-manage-item__footer']}>
                      <button
                        type="button"
                        className={`${styles['db-analytics-btn']} ${styles['db-analytics-btn--icon']}`}
                        onClick={() => setSchemaTarget(db)}
                        disabled={!isCsv && !db.connected}
                        title={
                          isCsv || db.connected
                            ? 'View schema & ER diagram'
                            : 'Connect this database to view its schema'
                        }
                        aria-label="View schema and ER diagram"
                      >
                        <Icon.schema size={14} aria-hidden="true" />
                      </button>
                      {!isCsv &&
                        (db.connected ? (
                          <button
                            className={`${styles['db-analytics-btn']} ${styles['db-analytics-btn--ghost']} ${styles['db-analytics-btn--sm']}`}
                            onClick={() => handleDisconnect(db)}
                            disabled={busyDbId === db.id}
                          >
                            {busyDbId === db.id ? 'Disconnecting…' : 'Disconnect'}
                          </button>
                        ) : (
                          <button
                            className={`${styles['db-analytics-btn']} ${styles['db-analytics-btn--primary']} ${styles['db-analytics-btn--sm']}`}
                            onClick={() => setConnectTarget(db)}
                          >
                            Connect
                          </button>
                        ))}
                    </div>
                  </li>
                );
              })}
            </ul>
          ) : tab === 'new' ? (
            <div className={styles['db-analytics-form-card']}>
              <div className={styles['db-analytics-form-card__header']}>
                <span className={styles['db-analytics-form-card__icon']}>
                  <Icon.database size={16} aria-hidden="true" />
                </span>
                <div>
                  <h3 className={styles['db-analytics-form-card__title']}>New database connection</h3>
                  <p className={styles['db-analytics-form-card__subtitle']}>
                    Connect a MySQL or PostgreSQL database to query from this workspace.
                  </p>
                </div>
              </div>

              <div className={styles['db-analytics-form']}>
                {createError && <div className={styles['db-analytics-form__error']}>{createError}</div>}

                <label className={styles['db-analytics-form__field']}>
                  <span className={styles['db-analytics-form__label']}>Connection name</span>
                  <input
                    className={styles['db-analytics-form__input']}
                    value={form.name}
                    onChange={(e) => updateField('name', e.target.value)}
                    placeholder="e.g. Production Warehouse"
                  />
                </label>

                <label className={styles['db-analytics-form__field']}>
                  <span className={styles['db-analytics-form__label']}>Database type</span>
                  <select
                    className={styles['db-analytics-form__input']}
                    value={form.db_type}
                    onChange={(e) => updateField('db_type', e.target.value as CreateDbPayload['db_type'])}
                  >
                    <option value="mysql">MySQL</option>
                    <option value="postgres">PostgreSQL</option>
                  </select>
                </label>

                <div className={styles['db-analytics-form__row']}>
                  <label className={styles['db-analytics-form__field']}>
                    <span className={styles['db-analytics-form__label']}>Host</span>
                    <input
                      className={styles['db-analytics-form__input']}
                      value={form.host}
                      onChange={(e) => updateField('host', e.target.value)}
                      placeholder="e.g. 10.0.0.4"
                    />
                  </label>
                  <label
                    className={`${styles['db-analytics-form__field']} ${styles['db-analytics-form__field--narrow']}`}
                  >
                    <span className={styles['db-analytics-form__label']}>Port</span>
                    <input
                      className={styles['db-analytics-form__input']}
                      type="number"
                      value={form.port}
                      onChange={(e) => updateField('port', Number(e.target.value))}
                    />
                  </label>
                </div>

                <label className={styles['db-analytics-form__field']}>
                  <span className={styles['db-analytics-form__label']}>Database name</span>
                  <input
                    className={styles['db-analytics-form__input']}
                    value={form.db_name}
                    onChange={(e) => updateField('db_name', e.target.value)}
                    placeholder="e.g. analytics_prod"
                  />
                </label>

                <label className={styles['db-analytics-form__field']}>
                  <span className={styles['db-analytics-form__label']}>Username</span>
                  <input
                    className={styles['db-analytics-form__input']}
                    value={form.username}
                    onChange={(e) => updateField('username', e.target.value)}
                    placeholder="e.g. readonly_user"
                  />
                </label>

                <p className={styles['db-analytics-form__hint']}>
                  <Icon.infoCircle size={14} aria-hidden="true" /> You'll be asked for the password
                  separately when connecting.
                </p>

                <div className={styles['db-analytics-form__actions']}>
                  <button
                    className={`${styles['db-analytics-btn']} ${styles['db-analytics-btn--ghost']}`}
                    onClick={() => setForm(emptyForm)}
                    disabled={creating}
                  >
                    Reset
                  </button>
                  <button
                    className={`${styles['db-analytics-btn']} ${styles['db-analytics-btn--primary']}`}
                    onClick={handleCreate}
                    disabled={creating}
                  >
                    {creating ? 'Creating…' : 'Create connection'}
                  </button>
                </div>
              </div>
            </div>
          ) : (
            <div className={styles['db-analytics-form-card']}>
              <div className={styles['db-analytics-form-card__header']}>
                <span
                  className={`${styles['db-analytics-form-card__icon']} ${styles['db-analytics-form-card__icon--csv']}`}
                >
                  <Icon.csv size={16} aria-hidden="true" />
                </span>
                <div>
                  <h3 className={styles['db-analytics-form-card__title']}>Upload a CSV file</h3>
                  <p className={styles['db-analytics-form-card__subtitle']}>
                    Turn a spreadsheet into a queryable table — no database connection required.
                  </p>
                </div>
              </div>

              <div className={styles['db-analytics-form']}>
                {csvError && <div className={styles['db-analytics-form__error']}>{csvError}</div>}

                <label className={styles['db-analytics-form__field']}>
                  <span className={styles['db-analytics-form__label']}>CSV file</span>
                  <button
                    type="button"
                    className={`${styles['db-analytics-file-drop']} ${
                      csvFile ? styles['db-analytics-file-drop--filled'] : ''
                    } ${isDraggingCsv ? styles['db-analytics-file-drop--dragging'] : ''}`}
                    onClick={() => fileInputRef.current?.click()}
                    onDragOver={handleDragOver}
                    onDragEnter={handleDragOver}
                    onDragLeave={handleDragLeave}
                    onDrop={handleDrop}
                  >
                    <span className={styles['db-analytics-file-drop__icon']}>
                      {csvFile ? (
                        <Icon.csv size={18} aria-hidden="true" />
                      ) : (
                        <Icon.upload size={18} aria-hidden="true" />
                      )}
                    </span>
                    <span className={styles['db-analytics-file-drop__text']}>
                      <span className={styles['db-analytics-file-drop__title']}>
                        {isDraggingCsv
                          ? 'Drop your CSV file here'
                          : csvFile
                          ? csvFile.name
                          : 'Choose a .csv file'}
                      </span>
                      <span className={styles['db-analytics-file-drop__subtitle']}>
                        {isDraggingCsv
                          ? 'Release to select this file'
                          : csvFile
                          ? `${(csvFile.size / 1024).toFixed(1)} KB — click to change`
                          : 'Drag & drop or click to browse — only .csv files are accepted'}
                      </span>
                    </span>
                  </button>
                  <input
                    ref={fileInputRef}
                    type="file"
                    accept=".csv,text/csv"
                    className={styles['db-analytics-file-drop__input']}
                    onChange={(e) => handleFilePick(e.target.files)}
                  />
                </label>

                <label className={styles['db-analytics-form__field']}>
                  <span className={styles['db-analytics-form__label']}>Name</span>
                  <input
                    className={styles['db-analytics-form__input']}
                    value={csvForm.name}
                    onChange={(e) => setCsvForm((prev) => ({ ...prev, name: e.target.value }))}
                    placeholder="e.g. Sample"
                  />
                </label>

                <label className={styles['db-analytics-form__field']}>
                  <span className={styles['db-analytics-form__label']}>Table name</span>
                  <input
                    className={styles['db-analytics-form__input']}
                    value={csvForm.table_name}
                    onChange={(e) => setCsvForm((prev) => ({ ...prev, table_name: e.target.value }))}
                    placeholder="e.g. Sample Table"
                  />
                </label>

                <p className={styles['db-analytics-form__hint']}>
                  <Icon.infoCircle size={14} aria-hidden="true" /> Uploaded CSVs are ready to query
                  immediately — no password or connection step needed.
                </p>

                <div className={styles['db-analytics-form__actions']}>
                  <button
                    className={`${styles['db-analytics-btn']} ${styles['db-analytics-btn--ghost']}`}
                    onClick={() => {
                      setCsvForm(emptyCsvForm);
                      setCsvFile(null);
                      setCsvError(null);
                      if (fileInputRef.current) fileInputRef.current.value = '';
                    }}
                    disabled={csvUploading}
                  >
                    Reset
                  </button>
                  <button
                    className={`${styles['db-analytics-btn']} ${styles['db-analytics-btn--primary']}`}
                    onClick={handleCsvUpload}
                    disabled={csvUploading}
                  >
                    {csvUploading ? 'Uploading…' : 'Upload CSV'}
                  </button>
                </div>
              </div>
            </div>
          )}
        </div>
      </aside>

      {connectTarget && (
        <ConnectPasswordModal
          database={connectTarget}
          onClose={() => setConnectTarget(null)}
          onConnected={(cacheUntil) => {
            onDatabaseConnected(connectTarget.id, cacheUntil);
            setConnectTarget(null);
          }}
        />
      )}

      <DbSchemaSlider database={schemaTarget} onClose={() => setSchemaTarget(null)} />
    </div>
  );
};

export default DatabaseManagerSlider;
















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

  // Force a classic, always-visible scrollbar rather than relying on the
  // OS/browser's overlay scrollbar, which only paints on hover/scroll and
  // fades out immediately after on many systems.
  scrollbar-width: thin;
  scrollbar-color: t.$border-stronger transparent;

  &::-webkit-scrollbar {
    width: 8px;
    -webkit-appearance: none;
  }

  &::-webkit-scrollbar-track {
    background: t.$surface-0;
  }

  &::-webkit-scrollbar-thumb {
    background: t.$border-stronger;
    border-radius: 999px;
    border: 2px solid t.$surface-0;
  }

  &::-webkit-scrollbar-thumb:hover {
    background: t.$text-muted;
  }
}

.db-analytics-results-panel__error {
  margin: 12px;
  font-size: 12px;
  color: #791f1f;
  background: #fbe9e9;
  padding: 8px 11px;
  border-radius: t.$radius;
}

.db-analytics-chart-groups {
  display: flex;
  flex-direction: column;
}

.db-analytics-chart-groups__divider {
  height: 1px;
  margin: 4px 20px 8px;
  background: t.$border-strong;
}

.db-analytics-chart-group__prompt {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  margin: 4px 12px 8px;
}

.db-analytics-chart-group__prompt-icon {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: t.$surface-0;
  color: t.$text-muted;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  margin-top: 1px;
}

.db-analytics-chart-group__prompt-text {
  font-size: 12.5px;
  font-weight: 600;
  color: t.$text-primary;
  margin: 0;
  line-height: 1.5;
}

.db-analytics-chart-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 12px;
  padding: 12px;
}

@media (max-width: 1500px) {
  .db-analytics-chart-grid {
    grid-template-columns: minmax(0, 1fr);
  }

  .db-analytics-chart-card--span-2 {
    grid-column: auto;
  }
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


















//ResultsPanel.tsx
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

export // Resolves which fields to use for a pie slice's label and value. Prefers
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

  const nameKey =
    chart.x_axis && keys.includes(chart.x_axis)
      ? chart.x_axis
      : keys.find((k) => typeof sample[k] === 'string') ?? keys[0] ?? 'name';

  const valueKey =
    chart.y_axis && keys.includes(chart.y_axis)
      ? chart.y_axis
      : keys.find((k) => k !== nameKey && typeof sample[k] === 'number') ?? 'value';

  return { nameKey, valueKey };
};

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
  const showEmptyState = chartGroups.length === 0;
  const [typeOverrides, setTypeOverrides] = useState<Record<string, ChartType>>({});
  const [expandedChartId, setExpandedChartId] = useState<string | null>(null);
  const [askAiChartId, setAskAiChartId] = useState<string | null>(null);

  const getDisplayType = (chart: ChartResult): ChartType => typeOverrides[chart.id] ?? chart.chart_type;

  const allCharts = chartGroups.flatMap((g) => g.graphs);
  const expandedChart = allCharts.find((c) => c.id === expandedChartId) ?? null;
  const expandedDisplayType = expandedChart ? getDisplayType(expandedChart) : 'bar';
  const askAiChart = allCharts.find((c) => c.id === askAiChartId) ?? null;
  const askAiDisplayType = askAiChart ? getDisplayType(askAiChart) : 'bar';

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
          <ChartConstructingAnimation />
        ) : showEmptyState ? (
          <div className={styles['db-analytics-empty-state']}>
            <Icon.chartAreaLine size={28} aria-hidden="true" />
            <p>Charts generated from your chat will appear here.</p>
          </div>
        ) : (
          <div className={styles['db-analytics-chart-groups']}>
            {chartGroups.map((group, groupIndex) => (
              <React.Fragment key={group.id}>
                {groupIndex > 0 && <div className={styles['db-analytics-chart-groups__divider']} aria-hidden="true" />}
                <section className={styles['db-analytics-chart-group']}>
                  {group.prompt && (
                    <div className={styles['db-analytics-chart-group__prompt']}>
                      <span className={styles['db-analytics-chart-group__prompt-icon']}>
                        <Icon.messageCircle size={12} aria-hidden="true" />
                      </span>
                      <p className={styles['db-analytics-chart-group__prompt-text']}>{group.prompt}</p>
                    </div>
                  )}
                  <div className={styles['db-analytics-chart-grid']}>
                    {group.graphs.map((chart, index) => {
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
                              {chart.chart_type !== 'kpi' && (
                                <>
                                  <button
                                    type="button"
                                    className={styles['db-analytics-chart-card__expand-btn']}
                                    onClick={() => setAskAiChartId(chart.id)}
                                    aria-label="Ask AI about this chart"
                                    title="Ask AI"
                                  >
                                    <Icon.askAi size={14} aria-hidden="true" />
                                  </button>
                                  <button
                                    type="button"
                                    className={styles['db-analytics-chart-card__expand-btn']}
                                    onClick={() => setExpandedChartId(chart.id)}
                                    aria-label="Expand chart"
                                    title="Expand"
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
        onClose={() => setExpandedChartId(null)}
      />

      <ChartAskAiSlider
        sessionId={sessionId}
        chart={askAiChart}
        displayType={askAiDisplayType}
        onClose={() => setAskAiChartId(null)}
      />
    </div>
  );
};

export default ResultsPanel;
