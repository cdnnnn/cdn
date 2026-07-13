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
  overflow-y: auto;
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
    width: 100%;
    border-collapse: collapse;
    margin: 8px 0;
    font-size: 12px;
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











//ChatPanel.tsx
import React, { useState, useEffect, useRef, KeyboardEvent } from 'react';
import { useTranslation } from 'react-i18next';
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
  // Covers the whole "one unit of work" span: for a freshly selected
  // session that's history + starter prompts/follow-ups; for a sent
  // message that's the stream + the trailing follow-up fetch. The composer
  // stays locked for the entire span rather than unlocking the instant
  // streaming/history-loading finishes but before suggestions have landed.
  isAwaitingResponseUnit: boolean;
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
  isAwaitingResponseUnit,
  streamSteps = [],
  streamError,
  followups = [],
  followupsLoading,
  disconnectedDatabases = [],
  onManageDatabases,
  onSend,
}) => {
  const { t } = useTranslation();
  const [draft, setDraft] = useState('');
  const threadEndRef = useRef<HTMLDivElement>(null);

  const isBlocked = disconnectedDatabases.length > 0;
  // Locked for the entire response unit — streaming, history loading, and
  // the trailing suggested-prompts/follow-ups fetch all count. The person
  // can't type or send another message until the whole thing settles.
  const composerDisabled = !!loading || isAwaitingResponseUnit || isBlocked;

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

  const composerPlaceholder = loading
    ? t('db_analytics.chatPanel.composer.placeholderLoading')
    : isBlocked
    ? t('db_analytics.chatPanel.composer.placeholderBlocked')
    : isAwaitingResponseUnit
    ? t('db_analytics.chatPanel.composer.placeholderStreaming')
    : t('db_analytics.chatPanel.composer.placeholderDefault');

  return (
    <div className={styles['db-analytics-chat-panel']}>
      <div className={styles['db-analytics-chat-panel__header']}>
        <div className={styles['db-analytics-chat-panel__header-text']}>
          <h2 className={styles['db-analytics-chat-panel__title']}>{sessionTitle || t('db_analytics.chatPanel.newSession')}</h2>
          <p className={styles['db-analytics-chat-panel__subtitle']}>{t('db_analytics.chatPanel.subtitle')}</p>
        </div>
      </div>

      <div className={styles['db-analytics-chat-panel__thread']}>
        {loading ? (
          <div className={styles['db-analytics-chat-panel__empty']}>
            <Icon.loader size={28} aria-hidden="true" className={styles['db-analytics-chat-panel__spin']} />
            <p>{t('db_analytics.chatPanel.loadingConversation')}</p>
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
                <p>{t('db_analytics.chatPanel.askToGetStarted')}</p>
              </>
            )}
          </div>
        ) : messages.length === 0 && !isStreaming ? (
          <div className={styles['db-analytics-chat-panel__empty']}>
            <Icon.messageCircle size={28} aria-hidden="true" />
            <p>{t('db_analytics.chatPanel.selectSessionToStart')}</p>
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
                      {t('db_analytics.chatPanel.chartRef', { count: msg.graphs.length })}
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
                ? t('db_analytics.chatPanel.dbNotice.titleSingle')
                : t('db_analytics.chatPanel.dbNotice.titlePlural', { count: disconnectedDatabases.length })}
            </p>
            <p className={styles['db-analytics-db-notice__desc']}>
              {t('db_analytics.chatPanel.dbNotice.reconnectPrefix')}{' '}
              {disconnectedDatabases.map((db, i) => (
                <React.Fragment key={db.id ? `${db.id}-${i}` : `db-${i}`}>
                  <strong>{db.name}</strong>
                  {i < disconnectedDatabases.length - 2
                    ? ', '
                    : i === disconnectedDatabases.length - 2
                    ? ` ${t('db_analytics.chatPanel.dbNotice.and')} `
                    : ''}
                </React.Fragment>
              ))}{' '}
              {t('db_analytics.chatPanel.dbNotice.reconnectSuffix')}
            </p>
          </div>
          <button className={styles['db-analytics-db-notice__action']} onClick={onManageDatabases}>
            {t('db_analytics.chatPanel.dbNotice.connectNow')}
          </button>
        </div>
      )}

      <div className={styles['db-analytics-chat-composer']}>
        <div className={styles['db-analytics-chat-composer__box']}>
          <textarea
            className={styles['db-analytics-chat-composer__input']}
            placeholder={composerPlaceholder}
            value={draft}
            onChange={(e) => setDraft(e.target.value)}
            onKeyDown={handleKeyDown}
            rows={3}
            disabled={composerDisabled}
          />
          <div className={styles['db-analytics-chat-composer__box-footer']}>
            <span className={styles['db-analytics-chat-composer__hint']}>
              {!isStreaming && isAwaitingResponseUnit ? (
                <span className={styles['db-analytics-chat-composer__hint--pending']}>
                  <Icon.sparkles size={11} aria-hidden="true" className={styles['db-analytics-chat-panel__spin']} />
                  {t('db_analytics.chatPanel.composer.preparingFollowups')}
                </span>
              ) : (
                t('db_analytics.chatPanel.composer.hint')
              )}
            </span>
            <button
              className={styles['db-analytics-chat-composer__send-btn']}
              onClick={submit}
              aria-label={t('db_analytics.chatPanel.composer.sendLabel')}
              disabled={composerDisabled || !draft.trim()}
            >
              {isStreaming ? (
                <Icon.loader size={14} aria-hidden="true" className={styles['db-analytics-chat-panel__spin']} />
              ) : (
                <Icon.send size={14} aria-hidden="true" />
              )}
              {t('db_analytics.chatPanel.composer.send')}
            </button>
          </div>
        </div>
      </div>
    </div>
  );
};

export default ChatPanel;
















//DatabaseManagerSlider.tsx
import React, { useRef, useState } from 'react';
import { useTranslation } from 'react-i18next';
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

const emptyForm: CreateDbPayload = {
  name: '',
  db_name: '',
  db_type: 'mysql',
  host: '',
  port: DEFAULT_PORTS.mysql,
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
  const { t } = useTranslation();
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
    const file = fileList?.[0];
    if (!file) return;
    if (!file.name.toLowerCase().endsWith('.csv')) {
      setCsvError(t('db_analytics.databaseManagerSlider.csvUpload.errorOnlyCsv'));
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
      setCsvError(t('db_analytics.databaseManagerSlider.csvUpload.errorSelectFile'));
      return;
    }
    if (!csvForm.name.trim() || !csvForm.table_name.trim()) {
      setCsvError(t('db_analytics.databaseManagerSlider.csvUpload.errorRequiredFields'));
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
      setCsvError(err instanceof Error ? err.message : t('db_analytics.databaseManagerSlider.csvUpload.genericError'));
    } finally {
      setCsvUploading(false);
    }
  };

  return (
    <div className={styles['db-analytics-slider-overlay']} role="dialog" aria-modal="true">
      <div className={styles['db-analytics-slider-overlay__backdrop']} onClick={onClose} />
      <aside className={styles['db-analytics-slider-panel']}>
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
                          ? t('db_analytics.databaseManagerSlider.csvUpload.dropHere')
                          : csvFile
                          ? csvFile.name
                          : t('db_analytics.databaseManagerSlider.csvUpload.chooseFile')}
                      </span>
                      <span className={styles['db-analytics-file-drop__subtitle']}>
                        {isDraggingCsv
                          ? t('db_analytics.databaseManagerSlider.csvUpload.releaseToSelect')
                          : csvFile
                          ? `${(csvFile.size / 1024).toFixed(1)} KB — ${t(
                              'databaseManagerSlider.csvUpload.clickToChange'
                            )}`
                          : t('db_analytics.databaseManagerSlider.csvUpload.dragOrClick')}
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

                <label className={styles['db-analytics-form__field']}>
                  <span className={styles['db-analytics-form__label']}>
                    {t('db_analytics.databaseManagerSlider.csvUpload.tableNameLabel')}
                  </span>
                  <input
                    className={styles['db-analytics-form__input']}
                    value={csvForm.table_name}
                    onChange={(e) => setCsvForm((prev) => ({ ...prev, table_name: e.target.value }))}
                    placeholder={t('db_analytics.databaseManagerSlider.csvUpload.tableNamePlaceholder')}
                  />
                </label>

                <p className={styles['db-analytics-form__hint']}>
                  <Icon.infoCircle size={14} aria-hidden="true" /> {t('db_analytics.databaseManagerSlider.csvUpload.hint')}
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















//SuggestedPrompts.module.scss
@use '../../../styles/tokens' as t;

.db-analytics-suggested-prompts {
  width: 100%;
  max-width: 640px;
  margin: 0 auto;
  padding: 8px 4px 4px;
}

.db-analytics-suggested-prompts__intro {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  margin-bottom: 16px;

  p {
    font-size: 13px;
    color: t.$text-secondary;
    margin: 0;
  }
}

.db-analytics-suggested-prompts__badge {
  width: 24px;
  height: 24px;
  border-radius: 7px;
  background: t.$gradient-primary;
  color: #fff;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.db-analytics-suggested-prompts__spin {
  animation: db-analytics-suggested-prompts-spin 0.8s linear infinite;
  color: t.$text-muted;
}

@keyframes db-analytics-suggested-prompts-spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

.db-analytics-suggested-prompts__grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 10px;
}

.db-analytics-suggested-prompts__card {
  display: flex;
  flex-direction: column;
  align-items: stretch;
  gap: 8px;
  text-align: left;
  padding: 12px 14px;
  border: 1px solid t.$border-strong;
  border-radius: t.$radius-lg;
  background: t.$surface-2;
  cursor: pointer;
  font-family: inherit;
  transition: border-color 0.15s ease, box-shadow 0.15s ease, transform 0.15s ease;
  position: relative;
  overflow: hidden;

  &::before {
    content: '';
    position: absolute;
    inset: 0 auto 0 0;
    width: 3px;
    background: t.$gradient-primary;
    opacity: 0;
    transition: opacity 0.15s ease;
  }

  &:hover {
    border-color: t.$border-stronger;
    box-shadow: 0 3px 10px rgba(79, 70, 229, 0.1);
    transform: translateY(-1px);

    &::before {
      opacity: 1;
    }

    .db-analytics-suggested-prompts__cta {
      color: t.$accent;

      svg {
        transform: translateX(2px);
      }
    }
  }

  &:active {
    transform: translateY(0);
  }
}

.db-analytics-suggested-prompts__card-top {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
}

.db-analytics-suggested-prompts__label {
  font-size: 12.5px;
  font-weight: 600;
  color: t.$text-primary;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.db-analytics-suggested-prompts__db {
  display: flex;
  align-items: center;
  gap: 4px;
  font-size: 10px;
  font-weight: 500;
  color: t.$text-secondary;
  background: t.$surface-0;
  border: 0.5px solid t.$border;
  padding: 2px 7px;
  border-radius: 999px;
  flex-shrink: 0;
  white-space: nowrap;
}

.db-analytics-suggested-prompts__text-row {
  display: flex;
  align-items: flex-start;
  gap: 6px;
}

.db-analytics-suggested-prompts__text {
  flex: 1;
  min-width: 0;
  font-size: 12px;
  line-height: 1.5;
  color: t.$text-secondary;
  margin: 0;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.db-analytics-suggested-prompts__text--expanded {
  display: block;
  -webkit-line-clamp: unset;
  overflow: visible;
}

.db-analytics-suggested-prompts__expand-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 18px;
  height: 18px;
  border-radius: 5px;
  color: t.$text-muted;
  cursor: pointer;
  flex-shrink: 0;
  margin-top: 1px;
  transition: background 0.12s ease, color 0.12s ease;

  &:hover {
    background: t.$surface-0;
    color: t.$accent;
  }

  &:focus-visible {
    outline: 2px solid t.$accent;
    outline-offset: 1px;
  }
}

.db-analytics-suggested-prompts__expand-icon {
  transform: rotate(90deg);
  transition: transform 0.15s ease;
}

.db-analytics-suggested-prompts__expand-icon--open {
  transform: rotate(-90deg);
}

.db-analytics-suggested-prompts__cta {
  display: flex;
  align-items: center;
  gap: 4px;
  font-size: 11.5px;
  font-weight: 500;
  color: t.$text-muted;
  transition: color 0.15s ease;

  svg {
    transition: transform 0.15s ease;
  }
}
















//SuggestedPrompts.tsx
import React, { useEffect, useRef, useState } from 'react';
import { useTranslation } from 'react-i18next';
import styles from './SuggestedPrompts.module.scss';
import { SuggestedPrompt } from '../types';
import { Icon } from '../icons';

interface Props {
  prompts: SuggestedPrompt[];
  loading?: boolean;
  error?: string | null;
  onSelect: (prompt: SuggestedPrompt) => void;
}

const SuggestedPrompts: React.FC<Props> = ({ prompts = [], loading, error, onSelect }) => {
  const { t } = useTranslation();
  const [expandedIndex, setExpandedIndex] = useState<number | null>(null);
  const [overflowingIndices, setOverflowingIndices] = useState<Set<number>>(new Set());
  const textRefs = useRef<Record<number, HTMLParagraphElement | null>>({});

  // Only show the expand toggle on cards whose text is actually clamped —
  // a short prompt that already fits in two lines doesn't need one.
  useEffect(() => {
    const next = new Set<number>();
    prompts.forEach((_, i) => {
      const el = textRefs.current[i];
      if (el && el.scrollHeight > el.clientHeight + 1) {
        next.add(i);
      }
    });
    setOverflowingIndices(next);
  }, [prompts]);

  if (loading) {
    return (
      <div className={styles['db-analytics-suggested-prompts']}>
        <div className={styles['db-analytics-suggested-prompts__intro']}>
          <Icon.loader size={20} aria-hidden="true" className={styles['db-analytics-suggested-prompts__spin']} />
          <p>{t('db_analytics.suggestedPrompts.finding')}</p>
        </div>
      </div>
    );
  }

  if (error) {
    return null;
  }

  if (prompts.length === 0) {
    return null;
  }

  return (
    <div className={styles['db-analytics-suggested-prompts']}>
      <div className={styles['db-analytics-suggested-prompts__intro']}>
        <span className={styles['db-analytics-suggested-prompts__badge']}>
          <Icon.lightbulb size={14} aria-hidden="true" />
        </span>
        <p>{t('db_analytics.suggestedPrompts.intro')}</p>
      </div>

      <div className={styles['db-analytics-suggested-prompts__grid']}>
        {prompts.map((prompt, i) => {
          const isExpanded = expandedIndex === i;
          const canExpand = overflowingIndices.has(i);
          return (
            <button
              key={`${prompt.label}-${i}`}
              className={styles['db-analytics-suggested-prompts__card']}
              onClick={() => onSelect(prompt)}
            >
              <div className={styles['db-analytics-suggested-prompts__card-top']}>
                <span className={styles['db-analytics-suggested-prompts__label']}>{prompt.label}</span>
                <span className={styles['db-analytics-suggested-prompts__db']}>
                  <Icon.database size={11} aria-hidden="true" />
                  {prompt.db_name}
                </span>
              </div>
              <div className={styles['db-analytics-suggested-prompts__text-row']}>
                <p
                  ref={(el) => {
                    textRefs.current[i] = el;
                  }}
                  className={`${styles['db-analytics-suggested-prompts__text']} ${
                    isExpanded ? styles['db-analytics-suggested-prompts__text--expanded'] : ''
                  }`}
                >
                  {prompt.text}
                </p>
                {canExpand && (
                  <span
                    className={styles['db-analytics-suggested-prompts__expand-btn']}
                    role="button"
                    tabIndex={0}
                    aria-label={
                      isExpanded
                        ? t('db_analytics.suggestedPrompts.showLess')
                        : t('db_analytics.suggestedPrompts.showMore')
                    }
                    title={
                      isExpanded
                        ? t('db_analytics.suggestedPrompts.showLess')
                        : t('db_analytics.suggestedPrompts.showMore')
                    }
                    onClick={(e) => {
                      e.stopPropagation();
                      setExpandedIndex(isExpanded ? null : i);
                    }}
                    onKeyDown={(e) => {
                      if (e.key === 'Enter' || e.key === ' ') {
                        e.preventDefault();
                        e.stopPropagation();
                        setExpandedIndex(isExpanded ? null : i);
                      }
                    }}
                  >
                    <Icon.arrowRight
                      size={12}
                      aria-hidden="true"
                      className={`${styles['db-analytics-suggested-prompts__expand-icon']} ${
                        isExpanded ? styles['db-analytics-suggested-prompts__expand-icon--open'] : ''
                      }`}
                    />
                  </span>
                )}
              </div>
              <span className={styles['db-analytics-suggested-prompts__cta']}>
                {t('db_analytics.suggestedPrompts.askThis')}
                <Icon.arrowRight size={13} aria-hidden="true" />
              </span>
            </button>
          );
        })}
      </div>
    </div>
  );
};

export default SuggestedPrompts;














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
      }
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
      }
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
        "clickToChange": "click to change",
        "nameLabel": "Name",
        "namePlaceholder": "e.g. Sample",
        "tableNameLabel": "Table name",
        "tableNamePlaceholder": "e.g. Sample Table",
        "hint": "Uploaded CSVs are ready to query immediately — no password or connection step needed.",
        "reset": "Reset",
        "upload": "Upload CSV",
        "uploading": "Uploading…",
        "errorOnlyCsv": "Only .csv files are supported.",
        "errorSelectFile": "Please select a CSV file to upload.",
        "errorRequiredFields": "Please provide both a name and a table name.",
        "genericError": "Failed to upload CSV file."
      }
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
      }
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
      }
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
        "clickToChange": "클릭하여 변경",
        "nameLabel": "이름",
        "namePlaceholder": "예: Sample",
        "tableNameLabel": "테이블 이름",
        "tableNamePlaceholder": "예: Sample Table",
        "hint": "업로드된 CSV는 즉시 조회할 수 있습니다 — 비밀번호나 연결 절차가 필요하지 않습니다.",
        "reset": "초기화",
        "upload": "CSV 업로드",
        "uploading": "업로드하는 중…",
        "errorOnlyCsv": ".csv 파일만 지원됩니다.",
        "errorSelectFile": "업로드할 CSV 파일을 선택해 주세요.",
        "errorRequiredFields": "이름과 테이블 이름을 모두 입력해 주세요.",
        "genericError": "CSV 파일 업로드에 실패했습니다."
      }
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
