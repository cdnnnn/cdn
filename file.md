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
  max-width: 100vw;
  height: 100%;
  background: t.$surface-2;
  display: flex;
  flex-direction: column;
  box-shadow: -8px 0 24px rgba(11, 11, 11, 0.12);
  animation: db-analytics-slide-in-right 0.22s cubic-bezier(0.16, 1, 0.3, 1);
}

.db-analytics-slider-panel--resizing {
  animation: none;
  transition: none;
}

.db-analytics-slider-panel__resize-handle {
  position: absolute;
  top: 0;
  left: -3px;
  width: 6px;
  height: 100%;
  cursor: col-resize;
  z-index: 5;
  touch-action: none;
  background: transparent;

  &::after {
    content: '';
    position: absolute;
    top: 0;
    left: 50%;
    width: 2px;
    height: 100%;
    transform: translateX(-50%);
    background: transparent;
    transition: background 0.12s ease;
  }

  &:hover::after {
    background: t.$accent;
  }
}

.db-analytics-slider-panel__resize-handle--active::after {
  background: t.$accent;
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
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 10px;
}

.db-analytics-db-manage-item {
  min-width: 0;
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

.db-analytics-db-manage-item__icon--mssql {
  background: t.$amber-bg;
  color: t.$amber-fg;
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

.db-analytics-db-manage-item__type-badge--mssql {
  color: t.$amber-fg;
  background: t.$amber-bg;
  border-color: transparent;
}

.db-analytics-db-manage-item__type-badge--csv {
  color: t.$green-fg;
  background: t.$green-bg;
  border-color: transparent;
}

.db-analytics-db-manage-item__meta-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 10px 12px;
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

.db-analytics-csv-file-list {
  list-style: none;
  margin: -6px 0 0;
  padding: 0;
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.db-analytics-csv-file-list__item {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 6px 8px 6px 10px;
  border: 0.5px solid t.$border;
  border-radius: t.$radius;
  background: t.$surface-1;
  color: t.$text-muted;
  font-size: 12px;
}

.db-analytics-csv-file-list__name {
  flex: 1;
  min-width: 0;
  color: t.$text-primary;
  font-weight: 500;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.db-analytics-csv-file-list__size {
  flex-shrink: 0;
  font-variant-numeric: tabular-nums;
}

.db-analytics-csv-file-list__remove {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 18px;
  height: 18px;
  border-radius: 5px;
  border: none;
  background: transparent;
  color: t.$text-muted;
  cursor: pointer;
  flex-shrink: 0;

  &:hover:not(:disabled) {
    background: t.$surface-2;
    color: t.$danger;
  }

  &:disabled {
    cursor: not-allowed;
    opacity: 0.5;
  }
}

.db-analytics-form__field-hint {
  font-size: 11.5px;
  color: t.$text-muted;
  margin: 6px 0 0;
  line-height: 1.5;
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

.db-analytics-type-switch {
  display: inline-flex;
  align-items: center;
  gap: 2px;
  padding: 3px;
  border: 1px solid t.$border-strong;
  border-radius: t.$radius;
  background: t.$surface-1;
  width: fit-content;
}

.db-analytics-type-switch__option {
  appearance: none;
  border: none;
  background: transparent;
  color: t.$text-secondary;
  font-family: inherit;
  font-size: 12.5px;
  font-weight: 500;
  padding: 7px 16px;
  border-radius: calc(t.$radius - 2px);
  cursor: pointer;
  transition: background 0.14s ease, color 0.14s ease, box-shadow 0.14s ease;

  &:hover {
    color: t.$text-primary;
  }
}

.db-analytics-type-switch__option--active {
  background: t.$gradient-primary;
  color: #fff;
  box-shadow: 0 2px 6px rgba(79, 70, 229, 0.3);

  &:hover {
    color: #fff;
  }
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
import { useTranslation } from 'react-i18next';
import styles from './DatabaseManagerSlider.module.scss';
import { DatabaseItem } from '../types';
import { api, CreateDbPayload, CsvUploadMode } from '../../../services/api';
import ConnectPasswordModal from './ConnectPasswordModal';
import DbSchemaSlider from './DbSchemaSlider';
import { Icon } from '../icons';
import { getDbTypeMeta } from '../dbTypeMeta';
import { useResizableWidth } from '../hooks/useResizableWidth';

interface Props {
  open: boolean;
  databases: DatabaseItem[];
  onClose: () => void;
  onDatabaseCreated: (db: DatabaseItem) => void;
  onDatabaseConnected: (dbId: string, cacheUntil: string) => void;
  onDatabaseDisconnected: (dbId: string) => void;
}

type Tab = 'existing' | 'new' | 'csv';

// Options for the database-type switch on the "New connection" form.
// Adding another engine later is just one more entry here. Labels ("MySQL",
// "PostgreSQL", "MSSQL") are proper product names and intentionally not
// translated — see the note in dbTypeMeta.ts.
const DB_TYPE_OPTIONS: { value: CreateDbPayload['db_type']; label: string }[] = [
  { value: 'mysql', label: 'MySQL' },
  { value: 'postgres', label: 'PostgreSQL' },
  { value: 'mssql', label: 'MSSQL' },
];

// Standard default ports per engine — used to pre-fill the port field, and
// to detect whether the user has left it untouched (so switching db type
// doesn't clobber a value they typed in on purpose).
const DEFAULT_PORTS: Record<CreateDbPayload['db_type'], number> = {
  mysql: 3306,
  postgres: 5432,
  mssql: 1433,
};

// Options for the CSV upload mode switch. "combined" merges every selected
// file into one table under a single connection; "separate" keeps each
// file as its own table within that same connection.
const CSV_MODE_OPTIONS: { value: CsvUploadMode; labelKey: string }[] = [
  { value: 'combined', labelKey: 'db_analytics.databaseManagerSlider.csvUpload.modeCombined' },
  { value: 'separate', labelKey: 'db_analytics.databaseManagerSlider.csvUpload.modeSeparate' },
];

// Server-enforced CSV upload limits, mirrored here so the person gets
// immediate feedback client-side instead of waiting on a round trip that
// will fail anyway.
const CSV_MAX_FILE_SIZE_BYTES = 50 * 1024 * 1024; // 50MB per file
const CSV_MAX_FILE_COUNT = 20;

const emptyForm: CreateDbPayload = {
  name: '',
  db_name: '',
  db_type: 'mysql',
  host: '',
  port: DEFAULT_PORTS.mysql,
  username: '',
};

const emptyCsvForm: { name: string; mode: CsvUploadMode } = {
  name: '',
  mode: 'combined',
};

const DatabaseManagerSlider: React.FC<Props> = ({
  open,
  databases,
  onClose,
  onDatabaseCreated,
  onDatabaseConnected,
  onDatabaseDisconnected,
}) => {
  const { t } = useTranslation();
  const [tab, setTab] = useState<Tab>('existing');
  const [form, setForm] = useState<CreateDbPayload>(emptyForm);
  const [creating, setCreating] = useState(false);
  const [createError, setCreateError] = useState<string | null>(null);
  const [busyDbId, setBusyDbId] = useState<string | null>(null);
  const [connectTarget, setConnectTarget] = useState<DatabaseItem | null>(null);
  const [schemaTarget, setSchemaTarget] = useState<DatabaseItem | null>(null);

  const [csvForm, setCsvForm] = useState(emptyCsvForm);
  const [csvFiles, setCsvFiles] = useState<File[]>([]);
  const [csvUploading, setCsvUploading] = useState(false);
  const [csvError, setCsvError] = useState<string | null>(null);
  const [isDraggingCsv, setIsDraggingCsv] = useState(false);
  const fileInputRef = useRef<HTMLInputElement>(null);

  const { width, isResizing, handlePointerDown } = useResizableWidth({
    defaultWidth: 600,
    minWidth: 420,
    maxWidth: 1100,
  });

  if (!open) return null;

  const updateField = <K extends keyof CreateDbPayload>(key: K, value: CreateDbPayload[K]) => {
    setForm((prev) => ({ ...prev, [key]: value }));
  };

  const handleDbTypeChange = (nextType: CreateDbPayload['db_type']) => {
    setForm((prev) => {
      const wasOnDefaultPort = prev.port === DEFAULT_PORTS[prev.db_type];
      return {
        ...prev,
        db_type: nextType,
        port: wasOnDefaultPort ? DEFAULT_PORTS[nextType] : prev.port,
      };
    });
  };

  const handleCreate = async () => {
    setCreateError(null);
    if (!form.name.trim() || !form.db_name.trim() || !form.host.trim() || !form.username.trim()) {
      setCreateError(t('db_analytics.databaseManagerSlider.newConnection.errorRequiredFields'));
      return;
    }
    setCreating(true);
    try {
      const created = await api.createDatabase(form);
      onDatabaseCreated(created);
      setForm(emptyForm);
      setTab('existing');
    } catch (err) {
      setCreateError(err instanceof Error ? err.message : t('db_analytics.databaseManagerSlider.newConnection.genericError'));
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
    if (!fileList || fileList.length === 0) return;
    const incoming = Array.from(fileList);

    const nonCsv = incoming.filter((f) => !f.name.toLowerCase().endsWith('.csv'));
    if (nonCsv.length > 0) {
      setCsvError(t('db_analytics.databaseManagerSlider.csvUpload.errorOnlyCsv'));
      return;
    }

    const tooLarge = incoming.filter((f) => f.size > CSV_MAX_FILE_SIZE_BYTES);
    if (tooLarge.length > 0) {
      setCsvError(
        t('db_analytics.databaseManagerSlider.csvUpload.errorFileTooLarge', {
          names: tooLarge.map((f) => f.name).join(', '),
          maxSize: 50,
        })
      );
      return;
    }

    setCsvFiles((prev) => {
      // Avoid adding an exact duplicate (same name + size) if the person
      // drags/selects overlapping files across multiple picks.
      const existingKeys = new Set(prev.map((f) => `${f.name}:${f.size}`));
      const deduped = incoming.filter((f) => !existingKeys.has(`${f.name}:${f.size}`));
      const combined = [...prev, ...deduped];

      if (combined.length > CSV_MAX_FILE_COUNT) {
        setCsvError(
          t('db_analytics.databaseManagerSlider.csvUpload.errorTooManyFiles', { max: CSV_MAX_FILE_COUNT })
        );
        // Keep only up to the limit rather than silently dropping the
        // person's selection entirely — they can remove a few and retry.
        return combined.slice(0, CSV_MAX_FILE_COUNT);
      }

      setCsvError(null);
      return combined;
    });
  };

  const handleRemoveFile = (index: number) => {
    setCsvFiles((prev) => prev.filter((_, i) => i !== index));
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
    if (csvFiles.length === 0) {
      setCsvError(t('db_analytics.databaseManagerSlider.csvUpload.errorSelectFile'));
      return;
    }
    if (!csvForm.name.trim()) {
      setCsvError(t('db_analytics.databaseManagerSlider.csvUpload.errorRequiredFields'));
      return;
    }
    setCsvUploading(true);
    try {
      const created = await api.uploadCsv({
        files: csvFiles,
        name: csvForm.name.trim(),
        mode: csvForm.mode,
      });
      onDatabaseCreated(created);
      setCsvForm(emptyCsvForm);
      setCsvFiles([]);
      if (fileInputRef.current) fileInputRef.current.value = '';
      setTab('existing');
    } catch (err) {
      setCsvError(err instanceof Error ? err.message : t('db_analytics.databaseManagerSlider.csvUpload.genericError'));
    } finally {
      setCsvUploading(false);
    }
  };

  return (
    <div className={styles['db-analytics-slider-overlay']} role="dialog" aria-modal="true">
      <div className={styles['db-analytics-slider-overlay__backdrop']} onClick={onClose} />
      <aside
        className={`${styles['db-analytics-slider-panel']} ${isResizing ? styles['db-analytics-slider-panel--resizing'] : ''}`}
        style={{ width: `${width}px` }}
      >
        <div
          className={`${styles['db-analytics-slider-panel__resize-handle']} ${
            isResizing ? styles['db-analytics-slider-panel__resize-handle--active'] : ''
          }`}
          onPointerDown={handlePointerDown}
          role="separator"
          aria-orientation="vertical"
          aria-label={t('db_analytics.databaseManagerSlider.resizeHandle')}
          title={t('db_analytics.databaseManagerSlider.resizeHandle')}
        />
        <div className={styles['db-analytics-slider-panel__header']}>
          <h2 className={styles['db-analytics-slider-panel__title']}>{t('db_analytics.databaseManagerSlider.title')}</h2>
          <button
            className={styles['db-analytics-slider-panel__close-btn']}
            aria-label={t('db_analytics.databaseManagerSlider.close')}
            onClick={onClose}
          >
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
            {t('db_analytics.databaseManagerSlider.tabs.existing')}
          </button>
          <button
            className={`${styles['db-analytics-slider-tabs__item']} ${
              tab === 'new' ? styles['db-analytics-slider-tabs__item--active'] : ''
            }`}
            onClick={() => setTab('new')}
          >
            <Icon.plus size={14} aria-hidden="true" />
            {t('db_analytics.databaseManagerSlider.tabs.new')}
          </button>
          <button
            className={`${styles['db-analytics-slider-tabs__item']} ${
              tab === 'csv' ? styles['db-analytics-slider-tabs__item--active'] : ''
            }`}
            onClick={() => setTab('csv')}
          >
            <Icon.upload size={14} aria-hidden="true" />
            {t('db_analytics.databaseManagerSlider.tabs.csv')}
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
                  <p className={styles['db-analytics-db-manage-empty__title']}>
                    {t('db_analytics.databaseManagerSlider.emptyTitle')}
                  </p>
                  <p className={styles['db-analytics-db-manage-empty__desc']}>
                    {t('db_analytics.databaseManagerSlider.emptyDesc')}
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
                          {t('db_analytics.databaseList.status.ready')}
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
                          {db.connected ? t('db_analytics.databaseList.status.connected') : t('db_analytics.databaseList.status.disconnected')}
                        </span>
                      )}
                    </div>

                    <dl className={styles['db-analytics-db-manage-item__meta-grid']}>
                      <div className={styles['db-analytics-db-manage-item__meta-cell']}>
                        <dt>{isCsv ? t('db_analytics.databaseManagerSlider.table') : t('db_analytics.databaseManagerSlider.database')}</dt>
                        <dd>{db.db_name}</dd>
                      </div>
                      {!isCsv && (
                        <div className={styles['db-analytics-db-manage-item__meta-cell']}>
                          <dt>{t('db_analytics.databaseManagerSlider.port')}</dt>
                          <dd>{db.port}</dd>
                        </div>
                      )}
                      <div className={styles['db-analytics-db-manage-item__meta-cell']}>
                        <dt>{isCsv ? t('db_analytics.databaseManagerSlider.source') : t('db_analytics.databaseManagerSlider.user')}</dt>
                        <dd>{isCsv ? t('db_analytics.databaseManagerSlider.csvUploadSource') : db.username}</dd>
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
                            ? t('db_analytics.databaseList.viewSchema')
                            : t('db_analytics.databaseList.connectToViewSchema')
                        }
                        aria-label={t('db_analytics.databaseList.viewSchemaAndErDiagram')}
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
                            {busyDbId === db.id
                              ? t('db_analytics.databaseManagerSlider.disconnecting')
                              : t('db_analytics.databaseManagerSlider.disconnect')}
                          </button>
                        ) : (
                          <button
                            className={`${styles['db-analytics-btn']} ${styles['db-analytics-btn--primary']} ${styles['db-analytics-btn--sm']}`}
                            onClick={() => setConnectTarget(db)}
                          >
                            {t('db_analytics.databaseManagerSlider.connect')}
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
                  <h3 className={styles['db-analytics-form-card__title']}>
                    {t('db_analytics.databaseManagerSlider.newConnection.title')}
                  </h3>
                  <p className={styles['db-analytics-form-card__subtitle']}>
                    {t('db_analytics.databaseManagerSlider.newConnection.subtitle')}
                  </p>
                </div>
              </div>

              <div className={styles['db-analytics-form']}>
                {createError && <div className={styles['db-analytics-form__error']}>{createError}</div>}

                <label className={styles['db-analytics-form__field']}>
                  <span className={styles['db-analytics-form__label']}>
                    {t('db_analytics.databaseManagerSlider.newConnection.connectionName')}
                  </span>
                  <input
                    className={styles['db-analytics-form__input']}
                    value={form.name}
                    onChange={(e) => updateField('name', e.target.value)}
                    placeholder={t('db_analytics.databaseManagerSlider.newConnection.connectionNamePlaceholder')}
                  />
                </label>

                <div className={styles['db-analytics-form__field']}>
                  <span className={styles['db-analytics-form__label']}>
                    {t('db_analytics.databaseManagerSlider.newConnection.databaseType')}
                  </span>
                  <div
                    className={styles['db-analytics-type-switch']}
                    role="radiogroup"
                    aria-label={t('db_analytics.databaseManagerSlider.newConnection.databaseType')}
                  >
                    {DB_TYPE_OPTIONS.map((option) => (
                      <button
                        key={option.value}
                        type="button"
                        role="radio"
                        aria-checked={form.db_type === option.value}
                        className={`${styles['db-analytics-type-switch__option']} ${
                          form.db_type === option.value ? styles['db-analytics-type-switch__option--active'] : ''
                        }`}
                        onClick={() => handleDbTypeChange(option.value)}
                      >
                        {option.label}
                      </button>
                    ))}
                  </div>
                </div>

                <div className={styles['db-analytics-form__row']}>
                  <label className={styles['db-analytics-form__field']}>
                    <span className={styles['db-analytics-form__label']}>
                      {t('db_analytics.databaseManagerSlider.newConnection.host')}
                    </span>
                    <input
                      className={styles['db-analytics-form__input']}
                      value={form.host}
                      onChange={(e) => updateField('host', e.target.value)}
                      placeholder={t('db_analytics.databaseManagerSlider.newConnection.hostPlaceholder')}
                    />
                  </label>
                  <label
                    className={`${styles['db-analytics-form__field']} ${styles['db-analytics-form__field--narrow']}`}
                  >
                    <span className={styles['db-analytics-form__label']}>
                      {t('db_analytics.databaseManagerSlider.newConnection.port')}
                    </span>
                    <input
                      className={styles['db-analytics-form__input']}
                      type="number"
                      value={form.port === 0 ? '' : form.port}
                      onChange={(e) => {
                        const raw = e.target.value;
                        // Allow the field to be fully cleared while typing —
                        // coercing an empty string to 0 immediately (via
                        // Number('')) was forcing a visible "0" back into
                        // the input on every keystroke, including at the
                        // very start of typing a new value.
                        updateField('port', raw === '' ? 0 : Number(raw));
                      }}
                    />
                  </label>
                </div>

                <label className={styles['db-analytics-form__field']}>
                  <span className={styles['db-analytics-form__label']}>
                    {t('db_analytics.databaseManagerSlider.newConnection.databaseName')}
                  </span>
                  <input
                    className={styles['db-analytics-form__input']}
                    value={form.db_name}
                    onChange={(e) => updateField('db_name', e.target.value)}
                    placeholder={t('db_analytics.databaseManagerSlider.newConnection.databaseNamePlaceholder')}
                  />
                </label>

                <label className={styles['db-analytics-form__field']}>
                  <span className={styles['db-analytics-form__label']}>
                    {t('db_analytics.databaseManagerSlider.newConnection.username')}
                  </span>
                  <input
                    className={styles['db-analytics-form__input']}
                    value={form.username}
                    onChange={(e) => updateField('username', e.target.value)}
                    placeholder={t('db_analytics.databaseManagerSlider.newConnection.usernamePlaceholder')}
                  />
                </label>

                <p className={styles['db-analytics-form__hint']}>
                  <Icon.infoCircle size={14} aria-hidden="true" />{' '}
                  {t('db_analytics.databaseManagerSlider.newConnection.passwordHint')}
                </p>

                <div className={styles['db-analytics-form__actions']}>
                  <button
                    className={`${styles['db-analytics-btn']} ${styles['db-analytics-btn--ghost']}`}
                    onClick={() => setForm(emptyForm)}
                    disabled={creating}
                  >
                    {t('db_analytics.databaseManagerSlider.newConnection.reset')}
                  </button>
                  <button
                    className={`${styles['db-analytics-btn']} ${styles['db-analytics-btn--primary']}`}
                    onClick={handleCreate}
                    disabled={creating}
                  >
                    {creating
                      ? t('db_analytics.databaseManagerSlider.newConnection.creating')
                      : t('db_analytics.databaseManagerSlider.newConnection.createConnection')}
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
                  <h3 className={styles['db-analytics-form-card__title']}>
                    {t('db_analytics.databaseManagerSlider.csvUpload.title')}
                  </h3>
                  <p className={styles['db-analytics-form-card__subtitle']}>
                    {t('db_analytics.databaseManagerSlider.csvUpload.subtitle')}
                  </p>
                </div>
              </div>

              <div className={styles['db-analytics-form']}>
                {csvError && <div className={styles['db-analytics-form__error']}>{csvError}</div>}

                <label className={styles['db-analytics-form__field']}>
                  <span className={styles['db-analytics-form__label']}>
                    {t('db_analytics.databaseManagerSlider.csvUpload.fileLabel')}
                  </span>
                  <span className={styles['db-analytics-form__field-hint']}>
                    <Icon.infoCircle size={12} aria-hidden="true" />
                    {t('db_analytics.databaseManagerSlider.csvUpload.limitsHint', {
                      maxSize: 50,
                      maxCount: CSV_MAX_FILE_COUNT,
                    })}
                  </span>
                  <button
                    type="button"
                    className={`${styles['db-analytics-file-drop']} ${
                      csvFiles.length > 0 ? styles['db-analytics-file-drop--filled'] : ''
                    } ${isDraggingCsv ? styles['db-analytics-file-drop--dragging'] : ''}`}
                    onClick={() => fileInputRef.current?.click()}
                    onDragOver={handleDragOver}
                    onDragEnter={handleDragOver}
                    onDragLeave={handleDragLeave}
                    onDrop={handleDrop}
                  >
                    <span className={styles['db-analytics-file-drop__icon']}>
                      {csvFiles.length > 0 ? (
                        <Icon.csv size={18} aria-hidden="true" />
                      ) : (
                        <Icon.upload size={18} aria-hidden="true" />
                      )}
                    </span>
                    <span className={styles['db-analytics-file-drop__text']}>
                      <span className={styles['db-analytics-file-drop__title']}>
                        {isDraggingCsv
                          ? t('db_analytics.databaseManagerSlider.csvUpload.dropHere')
                          : csvFiles.length > 0
                          ? t('db_analytics.databaseManagerSlider.csvUpload.filesSelectedCount', {
                              count: csvFiles.length,
                              max: CSV_MAX_FILE_COUNT,
                            })
                          : t('db_analytics.databaseManagerSlider.csvUpload.chooseFile')}
                      </span>
                      <span className={styles['db-analytics-file-drop__subtitle']}>
                        {isDraggingCsv
                          ? t('db_analytics.databaseManagerSlider.csvUpload.releaseToSelect')
                          : csvFiles.length > 0
                          ? t('db_analytics.databaseManagerSlider.csvUpload.addMoreFiles')
                          : t('db_analytics.databaseManagerSlider.csvUpload.dragOrClick')}
                      </span>
                    </span>
                  </button>
                  <input
                    ref={fileInputRef}
                    type="file"
                    accept=".csv,text/csv"
                    multiple
                    className={styles['db-analytics-file-drop__input']}
                    onChange={(e) => {
                      handleFilePick(e.target.files);
                      // Reset so picking the exact same file(s) again after
                      // removing them from the list still fires onChange.
                      e.target.value = '';
                    }}
                  />
                </label>

                {csvFiles.length > 0 && (
                  <ul className={styles['db-analytics-csv-file-list']}>
                    {csvFiles.map((file, i) => (
                      <li key={`${file.name}-${file.size}-${i}`} className={styles['db-analytics-csv-file-list__item']}>
                        <Icon.csv size={13} aria-hidden="true" />
                        <span className={styles['db-analytics-csv-file-list__name']}>{file.name}</span>
                        <span className={styles['db-analytics-csv-file-list__size']}>
                          {(file.size / 1024).toFixed(1)} KB
                        </span>
                        <button
                          type="button"
                          className={styles['db-analytics-csv-file-list__remove']}
                          onClick={() => handleRemoveFile(i)}
                          aria-label={t('db_analytics.databaseManagerSlider.csvUpload.removeFile', { name: file.name })}
                          disabled={csvUploading}
                        >
                          <Icon.close size={12} aria-hidden="true" />
                        </button>
                      </li>
                    ))}
                  </ul>
                )}

                <label className={styles['db-analytics-form__field']}>
                  <span className={styles['db-analytics-form__label']}>
                    {t('db_analytics.databaseManagerSlider.csvUpload.nameLabel')}
                  </span>
                  <input
                    className={styles['db-analytics-form__input']}
                    value={csvForm.name}
                    onChange={(e) => setCsvForm((prev) => ({ ...prev, name: e.target.value }))}
                    placeholder={t('db_analytics.databaseManagerSlider.csvUpload.namePlaceholder')}
                  />
                </label>

                <div className={styles['db-analytics-form__field']}>
                  <span className={styles['db-analytics-form__label']}>
                    {t('db_analytics.databaseManagerSlider.csvUpload.modeLabel')}
                  </span>
                  <div className={styles['db-analytics-type-switch']} role="radiogroup" aria-label={t('db_analytics.databaseManagerSlider.csvUpload.modeLabel')}>
                    {CSV_MODE_OPTIONS.map((option) => (
                      <button
                        key={option.value}
                        type="button"
                        role="radio"
                        aria-checked={csvForm.mode === option.value}
                        className={`${styles['db-analytics-type-switch__option']} ${
                          csvForm.mode === option.value ? styles['db-analytics-type-switch__option--active'] : ''
                        }`}
                        onClick={() => setCsvForm((prev) => ({ ...prev, mode: option.value }))}
                      >
                        {t(option.labelKey)}
                      </button>
                    ))}
                  </div>
                  <p className={styles['db-analytics-form__field-hint']}>
                    {csvForm.mode === 'combined'
                      ? t('db_analytics.databaseManagerSlider.csvUpload.modeCombinedHint')
                      : t('db_analytics.databaseManagerSlider.csvUpload.modeSeparateHint')}
                  </p>
                </div>

                <p className={styles['db-analytics-form__hint']}>
                  <Icon.infoCircle size={14} aria-hidden="true" /> {t('db_analytics.databaseManagerSlider.csvUpload.hint')}
                </p>

                <div className={styles['db-analytics-form__actions']}>
                  <button
                    className={`${styles['db-analytics-btn']} ${styles['db-analytics-btn--ghost']}`}
                    onClick={() => {
                      setCsvForm(emptyCsvForm);
                      setCsvFiles([]);
                      setCsvError(null);
                      if (fileInputRef.current) fileInputRef.current.value = '';
                    }}
                    disabled={csvUploading}
                  >
                    {t('db_analytics.databaseManagerSlider.csvUpload.reset')}
                  </button>
                  <button
                    className={`${styles['db-analytics-btn']} ${styles['db-analytics-btn--primary']}`}
                    onClick={handleCsvUpload}
                    disabled={csvUploading}
                  >
                    {csvUploading
                      ? t('db_analytics.databaseManagerSlider.csvUpload.uploading')
                      : t('db_analytics.databaseManagerSlider.csvUpload.upload')}
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












//DbSchemaSlider.module.scss
@use '../../../styles/tokens' as t;

.db-analytics-schema-overlay {
  position: fixed;
  inset: 0;
  z-index: 150;
  display: flex;
  justify-content: flex-end;
}

.db-analytics-schema-overlay__backdrop {
  position: absolute;
  inset: 0;
  background: rgba(11, 11, 11, 0.32);
  animation: db-analytics-schema-fade-in 0.15s ease;
}

@keyframes db-analytics-schema-fade-in {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

.db-analytics-schema-panel {
  position: relative;
  max-width: 100vw;
  height: 100%;
  background: t.$surface-2;
  display: flex;
  flex-direction: column;
  box-shadow: -8px 0 24px rgba(11, 11, 11, 0.14);
  animation: db-analytics-schema-slide-in 0.22s cubic-bezier(0.16, 1, 0.3, 1);
}

.db-analytics-schema-panel--resizing {
  animation: none;
  transition: none;
}

.db-analytics-schema-panel__resize-handle {
  position: absolute;
  top: 0;
  left: -3px;
  width: 6px;
  height: 100%;
  cursor: col-resize;
  z-index: 5;
  touch-action: none;
  background: transparent;

  &::after {
    content: '';
    position: absolute;
    top: 0;
    left: 50%;
    width: 2px;
    height: 100%;
    transform: translateX(-50%);
    background: transparent;
    transition: background 0.12s ease;
  }

  &:hover::after {
    background: t.$accent;
  }
}

.db-analytics-schema-panel__resize-handle--active::after {
  background: t.$accent;
}

@keyframes db-analytics-schema-slide-in {
  from {
    transform: translateX(100%);
  }
  to {
    transform: translateX(0);
  }
}

.db-analytics-schema-panel__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 64px;
  padding: 0 20px;
  border-bottom: 1px solid t.$border-strong;
  background: t.$surface-1;
  flex-shrink: 0;
}

.db-analytics-schema-panel__heading {
  display: flex;
  align-items: center;
  gap: 12px;
  min-width: 0;
}

.db-analytics-schema-panel__icon {
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

.db-analytics-schema-panel__title {
  font-size: 15px;
  font-weight: 600;
  color: t.$text-primary;
  margin: 0 0 2px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 320px;
}

.db-analytics-schema-panel__subtitle {
  font-size: 11.5px;
  color: t.$text-muted;
}

.db-analytics-schema-panel__close-btn {
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

.db-analytics-schema-tabs {
  display: flex;
  gap: 4px;
  padding: 12px 20px 0;
  border-bottom: 1px solid t.$border-strong;
  background: t.$surface-1;
  flex-shrink: 0;
}

.db-analytics-schema-tabs__item {
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

.db-analytics-schema-tabs__item--active {
  color: t.$text-primary;
  border-bottom-color: t.$text-primary;
}

.db-analytics-schema-tabs__count {
  font-size: 10.5px;
  font-weight: 600;
  color: t.$text-secondary;
  background: t.$surface-0;
  border: 0.5px solid t.$border;
  padding: 1px 6px;
  border-radius: 999px;
}

.db-analytics-schema-panel__body {
  flex: 1;
  overflow-y: auto;
  padding: 18px 20px;
}

.db-analytics-schema-empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 10px;
  padding: 60px 20px;
  color: t.$text-muted;
  text-align: center;

  p {
    font-size: 13px;
    margin: 0;
    max-width: 260px;
  }
}

.db-analytics-schema-spin {
  animation: db-analytics-schema-spin 0.8s linear infinite;
}

@keyframes db-analytics-schema-spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

.db-analytics-schema-summary {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 12.5px;
  color: t.$text-secondary;
  margin-bottom: 14px;

  strong {
    color: t.$text-primary;
    font-weight: 700;
  }
}

.db-analytics-schema-summary__dot {
  color: t.$text-muted;
}

.db-analytics-schema-table-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.db-analytics-schema-table {
  border: 1px solid t.$border-strong;
  border-radius: t.$radius-lg;
  overflow: hidden;
}

.db-analytics-schema-table__header {
  display: flex;
  align-items: center;
  gap: 10px;
  width: 100%;
  padding: 11px 14px;
  background: t.$surface-1;
  border: none;
  cursor: pointer;
  font-family: inherit;
  text-align: left;

  &:hover {
    background: t.$surface-0;
  }
}

.db-analytics-schema-table__icon {
  width: 24px;
  height: 24px;
  border-radius: 6px;
  background: t.$accent-bg;
  color: #0c447c;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.db-analytics-schema-table__name {
  font-size: 13px;
  font-weight: 600;
  color: t.$text-primary;
  font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
}

.db-analytics-schema-table__meta {
  flex: 1;
  text-align: right;
  font-size: 11px;
  color: t.$text-muted;
}

.db-analytics-schema-table__chevron {
  color: t.$text-muted;
  transform: rotate(90deg);
  transition: transform 0.15s ease;
  flex-shrink: 0;
}

.db-analytics-schema-table__chevron--open {
  transform: rotate(-90deg);
}

.db-analytics-schema-table__body {
  padding: 4px 0;
  border-top: 1px solid t.$border;
}

.db-analytics-schema-columns {
  width: 100%;
  border-collapse: collapse;
  font-size: 12px;

  th {
    text-align: left;
    font-weight: 600;
    font-size: 10.5px;
    text-transform: uppercase;
    letter-spacing: 0.02em;
    color: t.$text-muted;
    padding: 8px 14px 6px;
  }

  td {
    padding: 6px 14px;
    color: t.$text-primary;
    border-top: 0.5px solid t.$border;
  }

  tr:first-child td {
    border-top: none;
  }
}

.db-analytics-schema-columns__name {
  display: flex;
  align-items: center;
  gap: 6px;
  font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
  font-weight: 500;
}

.db-analytics-schema-columns__pk {
  font-size: 9px;
  font-weight: 700;
  color: t.$accent;
  background: t.$accent-bg;
  padding: 1px 5px;
  border-radius: 4px;
  flex-shrink: 0;
}

.db-analytics-schema-columns__type {
  font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
  color: t.$text-secondary;
  font-size: 11px;
}














//DbSchemaSlider.tsx
import React, { useEffect, useMemo, useState } from 'react';
import { useTranslation } from 'react-i18next';
import styles from './DbSchemaSlider.module.scss';
import { DatabaseItem, DbSchema } from '../types';
import { Icon } from '../icons';
import { api } from '../../../services/api';
import SchemaErDiagram from './SchemaErDiagram';
import { useResizableWidth } from '../hooks/useResizableWidth';

interface Props {
  database: DatabaseItem | null;
  onClose: () => void;
}

type Tab = 'tables' | 'diagram';

const DbSchemaSlider: React.FC<Props> = ({ database, onClose }) => {
  const { t } = useTranslation();
  const [tab, setTab] = useState<Tab>('tables');
  const [schema, setSchema] = useState<DbSchema | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [expandedTable, setExpandedTable] = useState<string | null>(null);
  const { width, isResizing, handlePointerDown } = useResizableWidth({
    defaultWidth: 800,
    minWidth: 480,
    maxWidth: 1400,
  });

  useEffect(() => {
    if (!database) return;
    let cancelled = false;
    setSchema(null);
    setError(null);
    setTab('tables');
    setExpandedTable(null);
    setLoading(true);

    api
      .getSchema(database.id)
      .then((res) => {
        if (!cancelled) {
          setSchema(res);
          setExpandedTable(res.tables[0]?.name ?? null);
        }
      })
      .catch((err) => {
        if (!cancelled) setError(err instanceof Error ? err.message : t('db_analytics.dbSchemaSlider.genericError'));
      })
      .finally(() => {
        if (!cancelled) setLoading(false);
      });

    return () => {
      cancelled = true;
    };
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [database?.id]);

  const totalRows = useMemo(
    () => schema?.tables.reduce((sum, t) => sum + (t.row_count ?? 0), 0) ?? 0,
    [schema]
  );

  if (!database) return null;

  return (
    <div className={styles['db-analytics-schema-overlay']} role="dialog" aria-modal="true">
      <div className={styles['db-analytics-schema-overlay__backdrop']} onClick={onClose} />
      <aside
        className={`${styles['db-analytics-schema-panel']} ${isResizing ? styles['db-analytics-schema-panel--resizing'] : ''}`}
        style={{ width: `${width}px` }}
      >
        <div
          className={`${styles['db-analytics-schema-panel__resize-handle']} ${
            isResizing ? styles['db-analytics-schema-panel__resize-handle--active'] : ''
          }`}
          onPointerDown={handlePointerDown}
          role="separator"
          aria-orientation="vertical"
          aria-label={t('db_analytics.dbSchemaSlider.resizeHandle')}
          title={t('db_analytics.dbSchemaSlider.resizeHandle')}
        />
        <div className={styles['db-analytics-schema-panel__header']}>
          <div className={styles['db-analytics-schema-panel__heading']}>
            <span className={styles['db-analytics-schema-panel__icon']}>
              <Icon.schema size={16} aria-hidden="true" />
            </span>
            <div>
              <h2 className={styles['db-analytics-schema-panel__title']}>{database.name}</h2>
              <span className={styles['db-analytics-schema-panel__subtitle']}>
                {t('db_analytics.dbSchemaSlider.subtitle')}
              </span>
            </div>
          </div>
          <button
            className={styles['db-analytics-schema-panel__close-btn']}
            aria-label={t('db_analytics.dbSchemaSlider.close')}
            onClick={onClose}
          >
            <Icon.close size={18} aria-hidden="true" />
          </button>
        </div>

        <div className={styles['db-analytics-schema-tabs']}>
          <button
            className={`${styles['db-analytics-schema-tabs__item']} ${
              tab === 'tables' ? styles['db-analytics-schema-tabs__item--active'] : ''
            }`}
            onClick={() => setTab('tables')}
          >
            <Icon.table size={14} aria-hidden="true" />
            {t('db_analytics.dbSchemaSlider.tabs.tables')}
            {schema && <span className={styles['db-analytics-schema-tabs__count']}>{schema.tables.length}</span>}
          </button>
          <button
            className={`${styles['db-analytics-schema-tabs__item']} ${
              tab === 'diagram' ? styles['db-analytics-schema-tabs__item--active'] : ''
            }`}
            onClick={() => setTab('diagram')}
          >
            <Icon.schema size={14} aria-hidden="true" />
            {t('db_analytics.dbSchemaSlider.tabs.erDiagram')}
          </button>
        </div>

        <div className={styles['db-analytics-schema-panel__body']}>
          {loading && (
            <div className={styles['db-analytics-schema-empty']}>
              <Icon.loader size={22} aria-hidden="true" className={styles['db-analytics-schema-spin']} />
              <p>{t('db_analytics.dbSchemaSlider.loadingSchema')}</p>
            </div>
          )}

          {!loading && error && (
            <div className={styles['db-analytics-schema-empty']}>
              <Icon.infoCircle size={22} aria-hidden="true" />
              <p>{error}</p>
            </div>
          )}

          {!loading && !error && schema && schema.tables.length === 0 && (
            <div className={styles['db-analytics-schema-empty']}>
              <Icon.table size={22} aria-hidden="true" />
              <p>{t('db_analytics.dbSchemaSlider.noTablesFound')}</p>
            </div>
          )}

          {!loading && !error && schema && schema.tables.length > 0 && tab === 'tables' && (
            <>
              <div className={styles['db-analytics-schema-summary']}>
                <span>
                  <strong>{schema.tables.length}</strong>{' '}
                  {schema.tables.length === 1 ? t('db_analytics.dbSchemaSlider.tableLabel') : t('db_analytics.dbSchemaSlider.tableLabelPlural')}
                </span>
                <span className={styles['db-analytics-schema-summary__dot']}>&middot;</span>
                <span>{t('db_analytics.dbSchemaSlider.totalRows', { count: totalRows })}</span>
              </div>

              <div className={styles['db-analytics-schema-table-list']}>
                {schema.tables.map((table) => {
                  const isOpen = expandedTable === table.name;
                  return (
                    <div key={table.name} className={styles['db-analytics-schema-table']}>
                      <button
                        className={styles['db-analytics-schema-table__header']}
                        onClick={() => setExpandedTable(isOpen ? null : table.name)}
                        aria-expanded={isOpen}
                      >
                        <span className={styles['db-analytics-schema-table__icon']}>
                          <Icon.table size={13} aria-hidden="true" />
                        </span>
                        <span className={styles['db-analytics-schema-table__name']}>{table.name}</span>
                        <span className={styles['db-analytics-schema-table__meta']}>
                          {t('db_analytics.dbSchemaSlider.columnCount', { count: table.columns.length })} &middot;{' '}
                          {t('db_analytics.dbSchemaSlider.rowCount', { count: table.row_count })}
                        </span>
                        <Icon.arrowRight
                          size={13}
                          aria-hidden="true"
                          className={`${styles['db-analytics-schema-table__chevron']} ${
                            isOpen ? styles['db-analytics-schema-table__chevron--open'] : ''
                          }`}
                        />
                      </button>

                      {isOpen && (
                        <div className={styles['db-analytics-schema-table__body']}>
                          <table className={styles['db-analytics-schema-columns']}>
                            <thead>
                              <tr>
                                <th>{t('db_analytics.dbSchemaSlider.table.columnHeader')}</th>
                                <th>{t('db_analytics.dbSchemaSlider.table.typeHeader')}</th>
                                <th>{t('db_analytics.dbSchemaSlider.table.nullableHeader')}</th>
                                <th>{t('db_analytics.dbSchemaSlider.table.defaultHeader')}</th>
                              </tr>
                            </thead>
                            <tbody>
                              {table.columns.map((col) => (
                                <tr key={col.name}>
                                  <td>
                                    <span className={styles['db-analytics-schema-columns__name']}>
                                      {col.primary_key && (
                                        <span
                                          className={styles['db-analytics-schema-columns__pk']}
                                          title={t('db_analytics.dbSchemaSlider.table.primaryKey')}
                                        >
                                          {t('db_analytics.dbSchemaSlider.table.pkAbbr')}
                                        </span>
                                      )}
                                      {col.name}
                                    </span>
                                  </td>
                                  <td>
                                    <span className={styles['db-analytics-schema-columns__type']}>{col.type}</span>
                                  </td>
                                  <td>
                                    {col.nullable
                                      ? t('db_analytics.dbSchemaSlider.table.nullableYes')
                                      : t('db_analytics.dbSchemaSlider.table.nullableNo')}
                                  </td>
                                  <td>{col.default ?? '—'}</td>
                                </tr>
                              ))}
                            </tbody>
                          </table>
                        </div>
                      )}
                    </div>
                  );
                })}
              </div>
            </>
          )}

          {!loading && !error && schema && schema.tables.length > 0 && tab === 'diagram' && (
            <SchemaErDiagram tables={schema.tables} />
          )}
        </div>
      </aside>
    </div>
  );
};

export default DbSchemaSlider;












//useResizableWidth.ts
import { useCallback, useEffect, useRef, useState } from 'react';

interface Options {
  defaultWidth: number;
  minWidth: number;
  maxWidth: number;
  // The panel slides in from the right and the handle sits on its left
  // edge — dragging left (negative delta) should grow it, dragging right
  // should shrink it. Pass `direction: 'right'` for a panel anchored on the
  // left instead, where the relationship is the opposite.
  direction?: 'left' | 'right';
}

interface ResizableWidth {
  width: number;
  isResizing: boolean;
  handlePointerDown: (e: React.PointerEvent) => void;
}

// Drag-to-resize for a fixed-width side panel (a slide-over drawer, not a
// grid column). Tracks width in state, clamps to [minWidth, maxWidth], and
// attaches window-level pointer listeners only while actively dragging.
export function useResizableWidth({ defaultWidth, minWidth, maxWidth, direction = 'left' }: Options): ResizableWidth {
  const [width, setWidth] = useState(defaultWidth);
  const [isResizing, setIsResizing] = useState(false);
  const dragStartXRef = useRef(0);
  const dragStartWidthRef = useRef(defaultWidth);

  const handlePointerDown = useCallback(
    (e: React.PointerEvent) => {
      e.preventDefault();
      dragStartXRef.current = e.clientX;
      dragStartWidthRef.current = width;
      setIsResizing(true);
    },
    [width]
  );

  useEffect(() => {
    if (!isResizing) return;

    const handlePointerMove = (e: PointerEvent) => {
      const rawDelta = e.clientX - dragStartXRef.current;
      const delta = direction === 'left' ? -rawDelta : rawDelta;
      const next = Math.min(maxWidth, Math.max(minWidth, dragStartWidthRef.current + delta));
      setWidth(next);
    };

    const handlePointerUp = () => setIsResizing(false);

    window.addEventListener('pointermove', handlePointerMove);
    window.addEventListener('pointerup', handlePointerUp);
    return () => {
      window.removeEventListener('pointermove', handlePointerMove);
      window.removeEventListener('pointerup', handlePointerUp);
    };
  }, [isResizing, minWidth, maxWidth, direction]);

  return { width, isResizing, handlePointerDown };
}

















{
  "db_analytics": {
    "app": {
      "title": "DB Analytics",
      "subtitle": "Ask questions about your databases in natural language and get instant charts and insights.",
      "databasesConnected_one": "{{count}} database connected",
      "databasesConnected_other": "{{count}} databases connected"
    },
    "sessionHistory": {
      "title": "Chat sessions",
      "newSession": "New chat session",
      "loadingSessions": "Loading sessions…",
      "waitForResponse": "Wait for the current response to finish",
      "emptyTitle": "No sessions yet",
      "emptyDesc": "Start a new conversation to begin.",
      "viewConnectedDatabases": "View connected databases",
      "connectedDatabasesPopoverTitle": "Connected databases",
      "connectedDatabasesEmpty": "No matching databases found."
    },
    "databaseList": {
      "title": "Databases",
      "manageDatabases": "Manage databases",
      "addDatabase": "Add database",
      "addDatabaseConnection": "Add database connection",
      "loadingDatabases": "Loading databases…",
      "waitForResponse": "Wait for the current response to finish",
      "emptyTitle": "No databases connected",
      "connectADatabase": "Connect a database",
      "viewSchema": "View schema & ER diagram",
      "connectToViewSchema": "Connect this database to view its schema",
      "viewSchemaAndErDiagram": "View schema and ER diagram",
      "status": {
        "ready": "Ready",
        "connected": "Connected",
        "disconnected": "Disconnected"
      },
      "refreshingDatabases": "Refreshing databases…"
    },
    "chatPanel": {
      "newSession": "New session",
      "subtitle": "Ask questions about your connected databases",
      "loadingConversation": "Loading conversation…",
      "askToGetStarted": "Ask a question to get started with this session.",
      "selectSessionToStart": "Select a session to get started.",
      "chartRef_one": "1 chart generated in results panel",
      "chartRef_other": "{{count}} charts generated in results panel",
      "dbNotice": {
        "titleSingle": "A database in this session is disconnected",
        "titlePlural_one": "{{count}} database in this session is disconnected",
        "titlePlural_other": "{{count}} databases in this session are disconnected",
        "reconnectPrefix": "Reconnect",
        "reconnectSuffix": "to continue this conversation.",
        "and": "and",
        "connectNow": "Connect now"
      },
      "composer": {
        "placeholderLoading": "Loading…",
        "placeholderBlocked": "Connect the databases below to start asking questions…",
        "placeholderStreaming": "Waiting for the current response to finish…",
        "placeholderDefault": "Ask about revenue, trends, or run a custom query...",
        "hint": "Enter to send · Shift+Enter for new line",
        "sendLabel": "Send message",
        "send": "Send",
        "preparingFollowups": "Preparing follow-up suggestions…"
      }
    },
    "suggestedPrompts": {
      "finding": "Finding good questions to start with…",
      "intro": "Not sure where to start? Try one of these.",
      "askThis": "Ask this",
      "showMore": "Show full prompt",
      "showLess": "Show less"
    },
    "followupChips": {
      "finding": "Finding follow-up questions…",
      "continueExploring": "Continue exploring"
    },
    "queryProgress": {
      "genericError": "Something went wrong."
    },
    "resultsPanel": {
      "title": "Generated insights",
      "subtitle": "Visualizations from your conversation",
      "emptyState": "Charts generated from your chat will appear here.",
      "drawingChart": "Drawing your chart…",
      "noDataForChart": "No data returned for this chart.",
      "noSeriesFound": "No numeric series found to plot for this chart.",
      "prompt": "Prompt",
      "askAiAboutChart": "Ask AI about this chart",
      "askAi": "Ask AI",
      "expandChart": "Expand chart",
      "expand": "Expand"
    },
    "chartTypeMenu": {
      "changeType": "Change chart type",
      "types": {
        "bar": "Bar",
        "line": "Line",
        "area": "Area",
        "pie": "Pie",
        "scatter": "Scatter"
      }
    },
    "expandChartSlider": {
      "close": "Close"
    },
    "chartAskAiSlider": {
      "title": "Ask about this chart",
      "close": "Close",
      "emptyPrompt": "Ask a question about this specific chart — its trend, an outlier, what's driving it.",
      "thinking": "Thinking…",
      "placeholder": "Ask something about this chart…",
      "ask": "Ask",
      "genericError": "Something went wrong."
    },
    "dbSchemaSlider": {
      "close": "Close",
      "subtitle": "Schema & relationships",
      "genericError": "Failed to load schema.",
      "tabs": {
        "tables": "Tables",
        "erDiagram": "ER Diagram"
      },
      "loadingSchema": "Loading schema…",
      "noTablesFound": "No tables found for this database.",
      "tableLabel": "table",
      "tableLabelPlural": "tables",
      "tableCount_one": "{{count}} table",
      "tableCount_other": "{{count}} tables",
      "totalRows": "{{count}} total rows",
      "columnCount_one": "{{count}} col",
      "columnCount_other": "{{count}} cols",
      "rowCount_one": "{{count}} row",
      "rowCount_other": "{{count}} rows",
      "table": {
        "columnHeader": "Column",
        "typeHeader": "Type",
        "nullableHeader": "Nullable",
        "defaultHeader": "Default",
        "nullableYes": "Yes",
        "nullableNo": "No",
        "primaryKey": "Primary key",
        "pkAbbr": "PK"
      },
      "resizeHandle": "Drag to resize"
    },
    "schemaErDiagram": {
      "noRelationships": "No relationships could be inferred from column names — tables are shown without connecting lines.",
      "moreColumns_one": "+{{count}} more column",
      "moreColumns_other": "+{{count}} more columns"
    },
    "connectPasswordModal": {
      "title": "Connect to {{name}}",
      "close": "Close",
      "description": "Enter the password for <strong>{{username}}</strong> on <strong>{{dbName}}</strong> to establish this connection.",
      "passwordLabel": "Password",
      "passwordPlaceholder": "••••••••",
      "cancel": "Cancel",
      "connect": "Connect",
      "connecting": "Connecting…",
      "errorRequired": "Password is required.",
      "genericError": "Failed to connect."
    },
    "databaseManagerSlider": {
      "title": "Manage databases",
      "close": "Close",
      "tabs": {
        "existing": "Existing databases",
        "new": "New connection",
        "csv": "Upload CSV"
      },
      "emptyTitle": "No databases yet",
      "emptyDesc": "Create a connection or upload a CSV file to start querying data.",
      "table": "Table",
      "database": "Database",
      "port": "Port",
      "source": "Source",
      "user": "User",
      "csvUploadSource": "CSV upload",
      "disconnect": "Disconnect",
      "disconnecting": "Disconnecting…",
      "connect": "Connect",
      "newConnection": {
        "title": "New database connection",
        "subtitle": "Connect a MySQL, PostgreSQL, or MSSQL database to query from this workspace.",
        "connectionName": "Connection name",
        "connectionNamePlaceholder": "e.g. Production Warehouse",
        "databaseType": "Database type",
        "host": "Host",
        "hostPlaceholder": "e.g. 10.0.0.4",
        "port": "Port",
        "databaseName": "Database name",
        "databaseNamePlaceholder": "e.g. analytics_prod",
        "username": "Username",
        "usernamePlaceholder": "e.g. readonly_user",
        "passwordHint": "You'll be asked for the password separately when connecting.",
        "reset": "Reset",
        "createConnection": "Create connection",
        "creating": "Creating…",
        "errorRequiredFields": "Please fill in all required fields.",
        "genericError": "Failed to create database."
      },
      "csvUpload": {
        "title": "Upload a CSV file",
        "subtitle": "Turn a spreadsheet into a queryable table — no database connection required.",
        "fileLabel": "CSV file",
        "chooseFile": "Choose a .csv file",
        "dropHere": "Drop your CSV file here",
        "releaseToSelect": "Release to select this file",
        "dragOrClick": "Drag & drop or click to browse — only .csv files are accepted",
        "nameLabel": "Name",
        "namePlaceholder": "e.g. Sample",
        "hint": "Uploaded CSVs are ready to query immediately — no password or connection step needed.",
        "reset": "Reset",
        "upload": "Upload CSV",
        "uploading": "Uploading…",
        "errorOnlyCsv": "Only .csv files are supported.",
        "errorSelectFile": "Please select at least one CSV file to upload.",
        "errorRequiredFields": "Please provide a name for this connection.",
        "genericError": "Failed to upload CSV file.",
        "addMoreFiles": "Click or drop to add more files",
        "removeFile": "Remove {{name}}",
        "modeLabel": "Upload mode",
        "modeCombined": "Combined",
        "modeSeparate": "Separate",
        "modeCombinedHint": "All selected files are merged into a single table.",
        "modeSeparateHint": "Each file becomes its own table within this connection.",
        "filesSelectedCount": "{{count}} / {{max}} files selected",
        "limitsHint": "Max {{maxSize}}MB per file, up to {{maxCount}} files.",
        "errorFileTooLarge": "These files exceed the {{maxSize}}MB limit: {{names}}",
        "errorTooManyFiles": "You can upload up to {{max}} files at a time. Extra files were not added."
      },
      "resizeHandle": "Drag to resize"
    },
    "newSessionSlider": {
      "title": "New chat session",
      "close": "Close",
      "sessionName": "Session name",
      "sessionNamePlaceholder": "e.g. Q3 marketing review",
      "connectDatabases": "Connect databases",
      "connectDatabasesHint": "Select one or more connected databases to make available in this session. Disconnected databases can't be added until they're connected.",
      "noDatabasesAvailable": "No databases available.",
      "addOne": "Add one",
      "connectFirstToInclude": "Connect this database first to include it in a session",
      "manageDatabases": "Manage databases",
      "createSession": "Create session",
      "creating": "Creating…",
      "errorNameRequired": "Please enter a session name.",
      "errorSelectDatabase": "Select at least one database for this session.",
      "genericError": "Failed to create session.",
      "status": {
        "connected": "Connected",
        "disconnected": "Disconnected"
      }
    },
    "dbTypes": {
      "mysql": "MySQL",
      "postgres": "PostgreSQL",
      "mssql": "MSSQL",
      "csv": "CSV"
    }
  }
}


















{
  "db_analytics": {
    "app": {
      "title": "DB 분석",
      "subtitle": "자연어로 데이터베이스에 질문하고 즉시 차트와 인사이트를 확인하세요.",
      "databasesConnected_other": "데이터베이스 {{count}}개 연결됨"
    },
    "sessionHistory": {
      "title": "채팅 세션",
      "newSession": "새 채팅 세션",
      "loadingSessions": "세션 불러오는 중…",
      "waitForResponse": "현재 응답이 끝날 때까지 기다려 주세요",
      "emptyTitle": "아직 세션이 없습니다",
      "emptyDesc": "새 대화를 시작해 보세요.",
      "viewConnectedDatabases": "연결된 데이터베이스 보기",
      "connectedDatabasesPopoverTitle": "연결된 데이터베이스",
      "connectedDatabasesEmpty": "일치하는 데이터베이스가 없습니다."
    },
    "databaseList": {
      "title": "데이터베이스",
      "manageDatabases": "데이터베이스 관리",
      "addDatabase": "데이터베이스 추가",
      "addDatabaseConnection": "데이터베이스 연결 추가",
      "loadingDatabases": "데이터베이스 불러오는 중…",
      "waitForResponse": "현재 응답이 끝날 때까지 기다려 주세요",
      "emptyTitle": "연결된 데이터베이스가 없습니다",
      "connectADatabase": "데이터베이스 연결",
      "viewSchema": "스키마 및 ER 다이어그램 보기",
      "connectToViewSchema": "스키마를 보려면 먼저 이 데이터베이스를 연결하세요",
      "viewSchemaAndErDiagram": "스키마 및 ER 다이어그램 보기",
      "status": {
        "ready": "준비됨",
        "connected": "연결됨",
        "disconnected": "연결 끊김"
      },
      "refreshingDatabases": "데이터베이스 새로고침 중…"
    },
    "chatPanel": {
      "newSession": "새 세션",
      "subtitle": "연결된 데이터베이스에 대해 질문해 보세요",
      "loadingConversation": "대화 불러오는 중…",
      "askToGetStarted": "이 세션을 시작하려면 질문을 입력하세요.",
      "selectSessionToStart": "시작하려면 세션을 선택하세요.",
      "chartRef_other": "결과 패널에 차트 {{count}}개가 생성되었습니다",
      "dbNotice": {
        "titleSingle": "이 세션의 데이터베이스 연결이 끊어졌습니다",
        "titlePlural_other": "이 세션의 데이터베이스 {{count}}개가 연결 끊김 상태입니다",
        "reconnectPrefix": "대화를 계속하려면",
        "reconnectSuffix": "을(를) 다시 연결하세요.",
        "and": "및",
        "connectNow": "지금 연결하기"
      },
      "composer": {
        "placeholderLoading": "불러오는 중…",
        "placeholderBlocked": "질문을 시작하려면 아래 데이터베이스를 연결하세요…",
        "placeholderStreaming": "현재 응답이 끝나기를 기다리는 중…",
        "placeholderDefault": "매출, 트렌드에 대해 질문하거나 원하는 쿼리를 실행해 보세요...",
        "hint": "Enter로 전송 · Shift+Enter로 줄바꿈",
        "sendLabel": "메시지 보내기",
        "send": "보내기",
        "preparingFollowups": "후속 질문을 준비하는 중…"
      }
    },
    "suggestedPrompts": {
      "finding": "시작하기 좋은 질문을 찾는 중…",
      "intro": "무엇을 물어봐야 할지 모르시겠나요? 아래 중 하나를 선택해 보세요.",
      "askThis": "이 질문하기",
      "showMore": "전체 프롬프트 보기",
      "showLess": "간략히 보기"
    },
    "followupChips": {
      "finding": "후속 질문을 찾는 중…",
      "continueExploring": "계속 살펴보기"
    },
    "queryProgress": {
      "genericError": "문제가 발생했습니다."
    },
    "resultsPanel": {
      "title": "생성된 인사이트",
      "subtitle": "대화에서 생성된 시각화 자료",
      "emptyState": "대화에서 생성된 차트가 여기에 표시됩니다.",
      "drawingChart": "차트를 그리는 중…",
      "noDataForChart": "이 차트에 대한 데이터가 없습니다.",
      "noSeriesFound": "이 차트에 표시할 숫자 데이터를 찾을 수 없습니다.",
      "prompt": "프롬프트",
      "askAiAboutChart": "이 차트에 대해 AI에게 질문하기",
      "askAi": "AI에게 질문",
      "expandChart": "차트 확대",
      "expand": "확대"
    },
    "chartTypeMenu": {
      "changeType": "차트 유형 변경",
      "types": {
        "bar": "막대형",
        "line": "선형",
        "area": "영역형",
        "pie": "원형",
        "scatter": "분산형"
      }
    },
    "expandChartSlider": {
      "close": "닫기"
    },
    "chartAskAiSlider": {
      "title": "이 차트에 대해 질문하기",
      "close": "닫기",
      "emptyPrompt": "이 차트의 추세, 특이점, 원인 등에 대해 질문해 보세요.",
      "thinking": "생각하는 중…",
      "placeholder": "이 차트에 대해 궁금한 점을 입력하세요…",
      "ask": "질문하기",
      "genericError": "문제가 발생했습니다."
    },
    "dbSchemaSlider": {
      "close": "닫기",
      "subtitle": "스키마 및 관계",
      "genericError": "스키마를 불러오지 못했습니다.",
      "tabs": {
        "tables": "테이블",
        "erDiagram": "ER 다이어그램"
      },
      "loadingSchema": "스키마 불러오는 중…",
      "noTablesFound": "이 데이터베이스에서 테이블을 찾을 수 없습니다.",
      "tableLabel": "개 테이블",
      "tableLabelPlural": "개 테이블",
      "tableCount_other": "테이블 {{count}}개",
      "totalRows": "총 {{count}}개 행",
      "columnCount_other": "열 {{count}}개",
      "rowCount_other": "행 {{count}}개",
      "table": {
        "columnHeader": "열",
        "typeHeader": "유형",
        "nullableHeader": "NULL 허용",
        "defaultHeader": "기본값",
        "nullableYes": "예",
        "nullableNo": "아니요",
        "primaryKey": "기본 키",
        "pkAbbr": "PK"
      },
      "resizeHandle": "드래그하여 크기 조절"
    },
    "schemaErDiagram": {
      "noRelationships": "열 이름으로부터 관계를 추론할 수 없습니다 — 테이블이 연결선 없이 표시됩니다.",
      "moreColumns_other": "열 {{count}}개 더 보기"
    },
    "connectPasswordModal": {
      "title": "{{name}}에 연결",
      "close": "닫기",
      "description": "<strong>{{dbName}}</strong>의 <strong>{{username}}</strong> 비밀번호를 입력하여 연결하세요.",
      "passwordLabel": "비밀번호",
      "passwordPlaceholder": "••••••••",
      "cancel": "취소",
      "connect": "연결",
      "connecting": "연결하는 중…",
      "errorRequired": "비밀번호를 입력해 주세요.",
      "genericError": "연결에 실패했습니다."
    },
    "databaseManagerSlider": {
      "title": "데이터베이스 관리",
      "close": "닫기",
      "tabs": {
        "existing": "기존 데이터베이스",
        "new": "새 연결",
        "csv": "CSV 업로드"
      },
      "emptyTitle": "아직 데이터베이스가 없습니다",
      "emptyDesc": "연결을 생성하거나 CSV 파일을 업로드하여 데이터 조회를 시작하세요.",
      "table": "테이블",
      "database": "데이터베이스",
      "port": "포트",
      "source": "소스",
      "user": "사용자",
      "csvUploadSource": "CSV 업로드",
      "disconnect": "연결 해제",
      "disconnecting": "연결 해제하는 중…",
      "connect": "연결",
      "newConnection": {
        "title": "새 데이터베이스 연결",
        "subtitle": "이 작업 공간에서 조회할 MySQL, PostgreSQL 또는 MSSQL 데이터베이스를 연결하세요.",
        "connectionName": "연결 이름",
        "connectionNamePlaceholder": "예: Production Warehouse",
        "databaseType": "데이터베이스 유형",
        "host": "호스트",
        "hostPlaceholder": "예: 10.0.0.4",
        "port": "포트",
        "databaseName": "데이터베이스 이름",
        "databaseNamePlaceholder": "예: analytics_prod",
        "username": "사용자 이름",
        "usernamePlaceholder": "예: readonly_user",
        "passwordHint": "비밀번호는 연결할 때 별도로 입력하게 됩니다.",
        "reset": "초기화",
        "createConnection": "연결 생성",
        "creating": "생성하는 중…",
        "errorRequiredFields": "모든 필수 항목을 입력해 주세요.",
        "genericError": "데이터베이스 생성에 실패했습니다."
      },
      "csvUpload": {
        "title": "CSV 파일 업로드",
        "subtitle": "스프레드시트를 조회 가능한 테이블로 변환하세요 — 데이터베이스 연결이 필요하지 않습니다.",
        "fileLabel": "CSV 파일",
        "chooseFile": ".csv 파일 선택",
        "dropHere": "여기에 CSV 파일을 놓으세요",
        "releaseToSelect": "놓으면 이 파일이 선택됩니다",
        "dragOrClick": "드래그 앤 드롭하거나 클릭하여 찾아보기 — .csv 파일만 지원됩니다",
        "nameLabel": "이름",
        "namePlaceholder": "예: Sample",
        "hint": "업로드된 CSV는 즉시 조회할 수 있습니다 — 비밀번호나 연결 절차가 필요하지 않습니다.",
        "reset": "초기화",
        "upload": "CSV 업로드",
        "uploading": "업로드하는 중…",
        "errorOnlyCsv": ".csv 파일만 지원됩니다.",
        "errorSelectFile": "업로드할 CSV 파일을 하나 이상 선택해 주세요.",
        "errorRequiredFields": "연결 이름을 입력해 주세요.",
        "genericError": "CSV 파일 업로드에 실패했습니다.",
        "addMoreFiles": "클릭하거나 파일을 놓아 더 추가하세요",
        "removeFile": "{{name}} 제거",
        "modeLabel": "업로드 방식",
        "modeCombined": "통합",
        "modeSeparate": "개별",
        "modeCombinedHint": "선택한 모든 파일이 하나의 테이블로 합쳐집니다.",
        "modeSeparateHint": "각 파일이 이 연결 내에서 개별 테이블이 됩니다.",
        "filesSelectedCount": "{{count}} / {{max}}개 파일 선택됨",
        "limitsHint": "파일당 최대 {{maxSize}}MB, 최대 {{maxCount}}개까지 업로드할 수 있습니다.",
        "errorFileTooLarge": "다음 파일이 {{maxSize}}MB 제한을 초과했습니다: {{names}}",
        "errorTooManyFiles": "한 번에 최대 {{max}}개의 파일을 업로드할 수 있습니다. 초과된 파일은 추가되지 않았습니다."
      },
      "resizeHandle": "드래그하여 크기 조절"
    },
    "newSessionSlider": {
      "title": "새 채팅 세션",
      "close": "닫기",
      "sessionName": "세션 이름",
      "sessionNamePlaceholder": "예: 3분기 마케팅 리뷰",
      "connectDatabases": "데이터베이스 연결",
      "connectDatabasesHint": "이 세션에서 사용할 연결된 데이터베이스를 하나 이상 선택하세요. 연결이 끊긴 데이터베이스는 연결하기 전까지 추가할 수 없습니다.",
      "noDatabasesAvailable": "사용 가능한 데이터베이스가 없습니다.",
      "addOne": "추가하기",
      "connectFirstToInclude": "세션에 포함하려면 먼저 이 데이터베이스를 연결하세요",
      "manageDatabases": "데이터베이스 관리",
      "createSession": "세션 생성",
      "creating": "생성하는 중…",
      "errorNameRequired": "세션 이름을 입력해 주세요.",
      "errorSelectDatabase": "이 세션에 사용할 데이터베이스를 하나 이상 선택하세요.",
      "genericError": "세션 생성에 실패했습니다.",
      "status": {
        "connected": "연결됨",
        "disconnected": "연결 끊김"
      }
    },
    "dbTypes": {
      "mysql": "MySQL",
      "postgres": "PostgreSQL",
      "mssql": "MSSQL",
      "csv": "CSV"
    }
  }
}
