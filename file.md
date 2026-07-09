//ChatPanel.tsx
import React, { useState, useEffect, useRef, KeyboardEvent } from 'react';
import ReactMarkdown from 'react-markdown';
import remarkGfm from 'remark-gfm';
import styles from './ChatPanel.module.scss';
import { ChatMessage, SuggestedPrompt, DatabaseItem } from '../types';
import { Icon } from '../icons';
import SuggestedPrompts from './SuggestedPrompts';
import QueryProgress from './QueryProgress';
import FollowupChips from './FollowupChips';

interface Props {
  sessionTitle: string;
  messages: ChatMessage[];
  loading?: boolean;
  hasActiveSession: boolean;
  suggestedPrompts: SuggestedPrompt[];
  suggestedPromptsLoading?: boolean;
  suggestedPromptsError?: string | null;
  isStreaming: boolean;
  streamSteps: string[];
  streamError?: string | null;
  followups: string[];
  followupsLoading?: boolean;
  disconnectedDatabases: DatabaseItem[];
  onManageDatabases: () => void;
  onSend: (text: string) => void;
}

const ChatPanel: React.FC<Props> = ({
  sessionTitle,
  messages = [],
  loading,
  hasActiveSession,
  suggestedPrompts = [],
  suggestedPromptsLoading,
  suggestedPromptsError,
  isStreaming,
  streamSteps = [],
  streamError,
  followups = [],
  followupsLoading,
  disconnectedDatabases = [],
  onManageDatabases,
  onSend,
}) => {
  const [draft, setDraft] = useState('');
  const threadEndRef = useRef<HTMLDivElement>(null);

  const isBlocked = disconnectedDatabases.length > 0;
  const composerDisabled = isStreaming || isBlocked;

  // Keep the thread pinned to the bottom while a response streams in (and
  // whenever new messages/steps/followups appear generally), so the person
  // doesn't have to manually scroll to follow along in real time.
  useEffect(() => {
    threadEndRef.current?.scrollIntoView({ behavior: 'smooth', block: 'end' });
  }, [messages, streamSteps, isStreaming, followups, followupsLoading]);

  const submit = () => {
    const trimmed = draft.trim();
    if (!trimmed || composerDisabled) return;
    onSend(trimmed);
    setDraft('');
  };

  const handleKeyDown = (e: KeyboardEvent<HTMLTextAreaElement>) => {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault();
      submit();
    }
  };

  const handlePromptSelect = (prompt: SuggestedPrompt) => {
    if (composerDisabled) return;
    onSend(prompt.text);
  };

  const handleFollowupSelect = (text: string) => {
    if (composerDisabled) return;
    onSend(text);
  };

  const showSuggestedPrompts = hasActiveSession && !loading && messages.length === 0 && !isStreaming;
  const showFollowups =
    !isStreaming && !loading && messages.length > 0 && (followupsLoading || followups.length > 0);

  const composerPlaceholder = isBlocked
    ? 'Connect the databases below to start asking questions…'
    : isStreaming
    ? 'Waiting for the current response to finish…'
    : 'Ask about revenue, trends, or run a custom query...';

  return (
    <div className={styles['db-analytics-chat-panel']}>
      <div className={styles['db-analytics-chat-panel__header']}>
        <div className={styles['db-analytics-chat-panel__header-text']}>
          <h2 className={styles['db-analytics-chat-panel__title']}>{sessionTitle || 'New session'}</h2>
          <p className={styles['db-analytics-chat-panel__subtitle']}>
            Ask questions about your connected databases
          </p>
        </div>
      </div>

      <div className={styles['db-analytics-chat-panel__thread']}>
        {loading ? (
          <div className={styles['db-analytics-chat-panel__empty']}>
            <Icon.loader size={28} aria-hidden="true" className={styles['db-analytics-chat-panel__spin']} />
            <p>Loading conversation…</p>
          </div>
        ) : showSuggestedPrompts ? (
          <div className={styles['db-analytics-chat-panel__empty']}>
            <SuggestedPrompts
              prompts={suggestedPrompts}
              loading={suggestedPromptsLoading}
              error={suggestedPromptsError}
              onSelect={handlePromptSelect}
            />
            {!suggestedPromptsLoading && suggestedPrompts.length === 0 && !suggestedPromptsError && (
              <>
                <Icon.messageCircle size={28} aria-hidden="true" />
                <p>Ask a question to get started with this session.</p>
              </>
            )}
          </div>
        ) : messages.length === 0 && !isStreaming ? (
          <div className={styles['db-analytics-chat-panel__empty']}>
            <Icon.messageCircle size={28} aria-hidden="true" />
            <p>Select a session to get started.</p>
          </div>
        ) : (
          <>
            {messages.map((msg) => (
              <div
                key={msg.id}
                className={`${styles['db-analytics-chat-bubble']} ${
                  msg.role === 'user'
                    ? styles['db-analytics-chat-bubble--user']
                    : styles['db-analytics-chat-bubble--assistant']
                }`}
              >
                {msg.role === 'assistant' && (
                  <div className={styles['db-analytics-chat-bubble__avatar']}>
                    <Icon.sparkles size={14} aria-hidden="true" />
                  </div>
                )}
                <div className={styles['db-analytics-chat-bubble__content']}>
                  {msg.role === 'assistant' ? (
                    <div className={styles['db-analytics-chat-bubble__markdown']}>
                      <ReactMarkdown remarkPlugins={[remarkGfm]}>{msg.content}</ReactMarkdown>
                    </div>
                  ) : (
                    <p className={styles['db-analytics-chat-bubble__text']}>{msg.content}</p>
                  )}
                  {msg.graphs.length > 0 && (
                    <div className={styles['db-analytics-chat-bubble__chart-ref']}>
                      <Icon.chartBar size={13} aria-hidden="true" />
                      {msg.graphs.length === 1
                        ? '1 chart generated in results panel'
                        : `${msg.graphs.length} charts generated in results panel`}
                    </div>
                  )}
                </div>
              </div>
            ))}

            {isStreaming && <QueryProgress steps={streamSteps} isStreaming={isStreaming} error={streamError} />}

            {showFollowups && (
              <FollowupChips followups={followups} loading={followupsLoading} onSelect={handleFollowupSelect} />
            )}
          </>
        )}
        <div ref={threadEndRef} />
      </div>

      {isBlocked && (
        <div className={styles['db-analytics-db-notice']} role="status">
          <span className={styles['db-analytics-db-notice__icon']}>
            <Icon.infoCircle size={16} aria-hidden="true" />
          </span>
          <div className={styles['db-analytics-db-notice__body']}>
            <p className={styles['db-analytics-db-notice__title']}>
              {disconnectedDatabases.length === 1
                ? 'A database in this session is disconnected'
                : `${disconnectedDatabases.length} databases in this session are disconnected`}
            </p>
            <p className={styles['db-analytics-db-notice__desc']}>
              Reconnect{' '}
              {disconnectedDatabases.map((db, i) => (
                <React.Fragment key={db.id ? `${db.id}-${i}` : `db-${i}`}>
                  <strong>{db.name}</strong>
                  {i < disconnectedDatabases.length - 2
                    ? ', '
                    : i === disconnectedDatabases.length - 2
                    ? ' and '
                    : ''}
                </React.Fragment>
              ))}{' '}
              to continue this conversation.
            </p>
          </div>
          <button className={styles['db-analytics-db-notice__action']} onClick={onManageDatabases}>
            Connect now
          </button>
        </div>
      )}

      <div className={styles['db-analytics-chat-composer']}>
        <textarea
          className={styles['db-analytics-chat-composer__input']}
          placeholder={composerPlaceholder}
          value={draft}
          onChange={(e) => setDraft(e.target.value)}
          onKeyDown={handleKeyDown}
          rows={3}
          disabled={composerDisabled}
        />
        <div className={styles['db-analytics-chat-composer__actions']}>
          <span className={styles['db-analytics-chat-composer__hint']}>
            Enter to send &middot; Shift+Enter for new line
          </span>
          <button
            className={styles['db-analytics-chat-composer__send-btn']}
            onClick={submit}
            aria-label="Send message"
            disabled={composerDisabled || !draft.trim()}
          >
            {isStreaming ? (
              <Icon.loader size={14} aria-hidden="true" className={styles['db-analytics-chat-panel__spin']} />
            ) : (
              <Icon.send size={14} aria-hidden="true" />
            )}
            Send
          </button>
        </div>
      </div>
    </div>
  );
};

export default ChatPanel;












//DatabaseList.tsx
import React, { useState } from 'react';
import styles from './DatabaseList.module.scss';
import { DatabaseItem } from '../types';
import { Icon } from '../icons';
import { getDbTypeMeta } from '../dbTypeMeta';
import SkeletonListItem from './SkeletonListItem';
import DbSchemaSlider from './DbSchemaSlider';

interface Props {
  databases: DatabaseItem[];
  onManage: () => void;
  disabled?: boolean;
  loading?: boolean;
}

const DatabaseList: React.FC<Props> = ({ databases, onManage, disabled, loading }) => {
  const disabledTitle = disabled ? 'Wait for the current response to finish' : undefined;
  const [schemaTarget, setSchemaTarget] = useState<DatabaseItem | null>(null);

  return (
    <div className={styles['db-analytics-db-panel']}>
      <div className={styles['db-analytics-db-panel__header']}>
        <h2 className={styles['db-analytics-db-panel__title']}>Databases</h2>
        <div className={styles['db-analytics-db-panel__header-actions']}>
          <button
            className={styles['db-analytics-db-panel__icon-btn']}
            aria-label="Manage databases"
            onClick={onManage}
            disabled={disabled}
            title={disabled ? disabledTitle : 'Manage databases'}
          >
            <Icon.settings size={15} aria-hidden="true" />
          </button>
          <button
            className={styles['db-analytics-db-panel__icon-btn']}
            aria-label="Add database connection"
            onClick={onManage}
            disabled={disabled}
            title={disabled ? disabledTitle : 'Add database'}
          >
            <Icon.plus size={16} aria-hidden="true" />
          </button>
        </div>
      </div>
      <div className={styles['db-analytics-db-panel__body']}>
        {loading ? (
          <SkeletonListItem variant="db" count={4} />
        ) : databases.length === 0 ? (
          <div className={styles['db-analytics-db-panel__empty']}>
            <span className={styles['db-analytics-db-panel__empty-icon']}>
              <Icon.database size={18} aria-hidden="true" />
            </span>
            <p>No databases connected</p>
            <button
              className={styles['db-analytics-db-panel__link']}
              onClick={onManage}
              disabled={disabled}
              title={disabledTitle}
            >
              Connect a database
            </button>
          </div>
        ) : (
          <ul className={styles['db-analytics-db-list']}>
            {databases.map((db, index) => {
              const typeMeta = getDbTypeMeta(db.db_type);
              const isCsv = db.db_type === 'csv';
              const schemaEnabled = isCsv || db.connected;
              return (
                <li key={db.id ? `${db.id}-${index}` : `db-${index}`} className={styles['db-analytics-db-item']}>
                  <div
                    className={`${styles['db-analytics-db-item__icon']} ${
                      styles[`db-analytics-db-item__icon--${typeMeta.modifier}`] ?? ''
                    }`}
                  >
                    {isCsv ? (
                      <Icon.csv size={14} aria-hidden="true" />
                    ) : (
                      <Icon.database size={14} aria-hidden="true" />
                    )}
                  </div>
                  <div className={styles['db-analytics-db-item__info']}>
                    <span className={styles['db-analytics-db-item__name']}>{db.name}</span>
                    <span className={styles['db-analytics-db-item__meta']}>
                      <span
                        className={`${styles['db-analytics-db-item__type-badge']} ${
                          styles[`db-analytics-db-item__type-badge--${typeMeta.modifier}`] ?? ''
                        }`}
                      >
                        {typeMeta.label}
                      </span>
                      <span className={styles['db-analytics-db-item__db-name']}>{db.db_name}</span>
                    </span>
                  </div>
                  <div className={styles['db-analytics-db-item__actions']}>
                    <button
                      type="button"
                      className={styles['db-analytics-db-item__schema-btn']}
                      onClick={() => schemaEnabled && setSchemaTarget(db)}
                      disabled={!schemaEnabled}
                      aria-label="View schema and ER diagram"
                      title={schemaEnabled ? 'View schema & ER diagram' : 'Connect this database to view its schema'}
                    >
                      <Icon.schema size={13} aria-hidden="true" />
                    </button>
                    <span
                      className={`${styles['db-analytics-db-status-dot']} ${
                        schemaEnabled
                          ? styles['db-analytics-db-status-dot--connected']
                          : styles['db-analytics-db-status-dot--disconnected']
                      }`}
                      title={isCsv ? 'Ready' : db.connected ? 'Connected' : 'Disconnected'}
                      aria-label={isCsv ? 'Ready' : db.connected ? 'Connected' : 'Disconnected'}
                    />
                  </div>
                </li>
              );
            })}
          </ul>
        )}
      </div>

      <DbSchemaSlider database={schemaTarget} onClose={() => setSchemaTarget(null)} />
    </div>
  );
};

export default DatabaseList;










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
                    }`}
                    onClick={() => fileInputRef.current?.click()}
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
                        {csvFile ? csvFile.name : 'Choose a .csv file'}
                      </span>
                      <span className={styles['db-analytics-file-drop__subtitle']}>
                        {csvFile
                          ? `${(csvFile.size / 1024).toFixed(1)} KB — click to change`
                          : 'Only .csv files are accepted'}
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















//NewSessionSlider.tsx
import React, { useState } from 'react';
import styles from './NewSessionSlider.module.scss';
import { DatabaseItem, SessionItem } from '../types';
import { api } from '../../../services/api';
import { Icon } from '../icons';

interface Props {
  open: boolean;
  databases: DatabaseItem[];
  onClose: () => void;
  onCreated: (session: SessionItem) => void;
  onManageDatabases: () => void;
}

const NewSessionSlider: React.FC<Props> = ({ open, databases, onClose, onCreated, onManageDatabases }) => {
  const [name, setName] = useState('');
  const [selectedIds, setSelectedIds] = useState<string[]>([]);
  const [creating, setCreating] = useState(false);
  const [error, setError] = useState<string | null>(null);

  if (!open) return null;

  const toggleDb = (id: string) => {
    setSelectedIds((prev) => (prev.includes(id) ? prev.filter((x) => x !== id) : [...prev, id]));
  };

  const handleCreate = async () => {
    setError(null);
    if (!name.trim()) {
      setError('Please enter a session name.');
      return;
    }
    if (selectedIds.length === 0) {
      setError('Select at least one database for this session.');
      return;
    }
    setCreating(true);
    try {
      const session = await api.createSession({ name: name.trim(), db_ids: selectedIds });
      onCreated(session);
      setName('');
      setSelectedIds([]);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to create session.');
    } finally {
      setCreating(false);
    }
  };

  return (
    <div className={styles['db-analytics-slider-overlay']} role="dialog" aria-modal="true">
      <div className={styles['db-analytics-slider-overlay__backdrop']} onClick={onClose} />
      <aside className={styles['db-analytics-slider-panel']}>
        <div className={styles['db-analytics-slider-panel__header']}>
          <h2 className={styles['db-analytics-slider-panel__title']}>New chat session</h2>
          <button className={styles['db-analytics-slider-panel__close-btn']} aria-label="Close" onClick={onClose}>
            <Icon.close size={16} aria-hidden="true" />
          </button>
        </div>

        <div className={styles['db-analytics-slider-panel__body']}>
          <div className={styles['db-analytics-form']}>
            {error && <div className={styles['db-analytics-form__error']}>{error}</div>}

            <label className={styles['db-analytics-form__field']}>
              <span className={styles['db-analytics-form__label']}>Session name</span>
              <input
                className={styles['db-analytics-form__input']}
                value={name}
                onChange={(e) => setName(e.target.value)}
                placeholder="e.g. Q3 marketing review"
                autoFocus
              />
            </label>

            <div className={styles['db-analytics-form__field']}>
              <span className={styles['db-analytics-form__label']}>Connect databases</span>
              <p className={styles['db-analytics-form__hint']}>
                Select one or more connected databases to make available in this session. Disconnected
                databases can't be added until they're connected.
              </p>

              <ul className={styles['db-analytics-db-select-list']}>
                {databases.length === 0 && (
                  <li className={styles['db-analytics-db-select-empty']}>
                    No databases available.{' '}
                    <button className={styles['db-analytics-link-btn']} onClick={onManageDatabases}>
                      Add one
                    </button>
                  </li>
                )}
                {databases.map((db, index) => {
                  const checked = selectedIds.includes(db.id);
                  const isDisabled = !db.connected;
                  return (
                    <li key={db.id ? `${db.id}-${index}` : `db-${index}`}>
                      <button
                        type="button"
                        className={`${styles['db-analytics-db-select-item']} ${
                          checked ? styles['db-analytics-db-select-item--checked'] : ''
                        } ${isDisabled ? styles['db-analytics-db-select-item--disabled'] : ''}`}
                        onClick={() => !isDisabled && toggleDb(db.id)}
                        disabled={isDisabled}
                        title={isDisabled ? 'Connect this database first to include it in a session' : undefined}
                      >
                        <div className={styles['db-analytics-db-select-item__top']}>
                          <span className={styles['db-analytics-db-select-item__icon']}>
                            <Icon.database size={14} aria-hidden="true" />
                          </span>
                          <span
                            className={`${styles['db-analytics-db-select-item__checkbox']} ${
                              checked ? styles['db-analytics-db-select-item__checkbox--checked'] : ''
                            }`}
                          >
                            {checked && <Icon.check size={12} aria-hidden="true" />}
                          </span>
                        </div>
                        <span className={styles['db-analytics-db-select-item__info']}>
                          <span className={styles['db-analytics-db-select-item__name']}>{db.name}</span>
                          <span className={styles['db-analytics-db-select-item__meta']}>
                            {db.db_type} &middot; {db.db_name}
                          </span>
                        </span>
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
                      </button>
                    </li>
                  );
                })}
              </ul>
            </div>

            <div className={styles['db-analytics-form__actions']}>
              <button
                className={`${styles['db-analytics-btn']} ${styles['db-analytics-btn--ghost']}`}
                onClick={onManageDatabases}
              >
                <Icon.plus size={14} aria-hidden="true" />
                Manage databases
              </button>
              <button
                className={`${styles['db-analytics-btn']} ${styles['db-analytics-btn--primary']}`}
                onClick={handleCreate}
                disabled={creating}
              >
                {creating ? 'Creating…' : 'Create session'}
              </button>
            </div>
          </div>
        </div>
      </aside>
    </div>
  );
};

export default NewSessionSlider;












//SessionHistory.tsx
import React from 'react';
import styles from './SessionHistory.module.scss';
import { SessionItem } from '../types';
import { Icon } from '../icons';
import SkeletonListItem from './SkeletonListItem';

interface Props {
  sessions: SessionItem[];
  activeSessionId: string | null;
  onSelect: (id: string) => void;
  onNewSession: () => void;
  disabled?: boolean;
  loading?: boolean;
}

const formatSessionDate = (iso: string) => {
  const date = new Date(iso);
  return date.toLocaleDateString(undefined, {
    day: 'numeric',
    month: 'short',
    year: 'numeric',
  });
};

const SessionHistory: React.FC<Props> = ({
  sessions,
  activeSessionId,
  onSelect,
  onNewSession,
  disabled,
  loading,
}) => {
  return (
    <div className={styles['db-analytics-session-panel']}>
      <div className={styles['db-analytics-session-panel__header']}>
        <h2 className={styles['db-analytics-session-panel__title']}>Chat sessions</h2>
        <button
          className={styles['db-analytics-session-panel__icon-btn']}
          aria-label="New chat session"
          onClick={onNewSession}
          disabled={disabled}
          title={disabled ? 'Wait for the current response to finish' : undefined}
        >
          <Icon.plus size={16} aria-hidden="true" />
        </button>
      </div>
      <div className={styles['db-analytics-session-panel__body']}>
        {loading ? (
          <SkeletonListItem variant="session" count={4} />
        ) : sessions.length === 0 ? (
          <div className={styles['db-analytics-session-panel__empty']}>
            <span className={styles['db-analytics-session-panel__empty-icon']}>
              <Icon.messageCircle size={18} aria-hidden="true" />
            </span>
            <p>No sessions yet</p>
            <span>Start a new conversation to begin.</span>
          </div>
        ) : (
          <ul className={styles['db-analytics-session-list']}>
            {sessions.map((session, index) => {
              const isActive = session.id === activeSessionId;
              return (
                <li key={session.id ? `${session.id}-${index}` : `session-${index}`}>
                  <button
                    className={`${styles['db-analytics-session-item']} ${
                      isActive ? styles['db-analytics-session-item--active'] : ''
                    }`}
                    onClick={() => onSelect(session.id)}
                    disabled={disabled}
                    title={disabled ? 'Wait for the current response to finish' : undefined}
                  >
                    <span className={styles['db-analytics-session-item__icon-wrap']}>
                      <Icon.messageCircle size={15} aria-hidden="true" />
                    </span>
                    <span className={styles['db-analytics-session-item__text']}>
                      <span className={styles['db-analytics-session-item__title']}>{session.name}</span>
                      <span className={styles['db-analytics-session-item__meta']}>
                        <span className={styles['db-analytics-session-item__db-pill']}>
                          <Icon.database size={10} aria-hidden="true" />
                          {(session.db_ids ?? []).length}
                        </span>
                        <span className={styles['db-analytics-session-item__time']}>
                          {formatSessionDate(session.created_at)}
                        </span>
                      </span>
                    </span>
                  </button>
                </li>
              );
            })}
          </ul>
        )}
      </div>
    </div>
  );
};

export default SessionHistory;
