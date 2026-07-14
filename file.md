//DatabaseManagerSlider.tsx

import React, { useEffect, useRef, useState } from 'react';
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
  // Sticky for the rest of this panel's lifetime once dragging starts — see
  // the matching comment in DbSchemaSlider.tsx for why this can't just be
  // tied to the live isResizing flag (it would re-trigger the entrance
  // @keyframes on drag-end and produce a visible snap-back-then-forward).
  const [hasResizedOnce, setHasResizedOnce] = useState(false);
  useEffect(() => {
    if (isResizing) setHasResizedOnce(true);
  }, [isResizing]);

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
        className={`${styles['db-analytics-slider-panel']} ${
          hasResizedOnce ? styles['db-analytics-slider-panel--resizing'] : ''
        }`}
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
  // Sticky for the rest of this panel's lifetime once the person starts
  // dragging — not just tied to the live isResizing flag. If it were tied
  // to isResizing alone, the moment a drag ends the class list changes
  // again, which re-triggers the base slide-in @keyframes from its 0%
  // state (translateX(100%)) and produces a visible snap-back-then-forward
  // instead of just settling at the new width.
  const [hasResizedOnce, setHasResizedOnce] = useState(false);
  useEffect(() => {
    if (isResizing) setHasResizedOnce(true);
  }, [isResizing]);

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
        className={`${styles['db-analytics-schema-panel']} ${
          hasResizedOnce ? styles['db-analytics-schema-panel--resizing'] : ''
        }`}
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
