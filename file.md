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
  align-items: center;
  gap: 10px;
  padding: 7px 8px;
  border-radius: t.$radius;
  transition: background 0.15s ease;

  &:hover {
    background: t.$surface-0;
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












//DatabaseList.tsx
import React, { useState } from 'react';
import { useTranslation } from 'react-i18next';
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
  refreshing?: boolean;
}

const DatabaseList: React.FC<Props> = ({ databases, onManage, disabled, loading, refreshing }) => {
  const { t } = useTranslation();
  const isDisabled = !!disabled || !!loading || !!refreshing;
  const disabledTitle = loading
    ? t('db_analytics.databaseList.loadingDatabases')
    : refreshing
    ? t('db_analytics.databaseList.refreshingDatabases')
    : disabled
    ? t('db_analytics.databaseList.waitForResponse')
    : undefined;
  const [schemaTarget, setSchemaTarget] = useState<DatabaseItem | null>(null);

  return (
    <div className={styles['db-analytics-db-panel']}>
      <div className={styles['db-analytics-db-panel__header']}>
        <h2 className={styles['db-analytics-db-panel__title']}>
          {t('db_analytics.databaseList.title')}
          {refreshing && (
            <Icon.loader
              size={12}
              aria-hidden="true"
              className={styles['db-analytics-db-panel__refresh-spin']}
            />
          )}
        </h2>
        <div className={styles['db-analytics-db-panel__header-actions']}>
          <button
            className={styles['db-analytics-db-panel__icon-btn']}
            aria-label={t('db_analytics.databaseList.manageDatabases')}
            onClick={onManage}
            disabled={isDisabled}
            title={isDisabled ? disabledTitle : t('db_analytics.databaseList.manageDatabases')}
          >
            <Icon.settings size={15} aria-hidden="true" />
          </button>
          <button
            className={styles['db-analytics-db-panel__icon-btn']}
            aria-label={t('db_analytics.databaseList.addDatabaseConnection')}
            onClick={onManage}
            disabled={isDisabled}
            title={isDisabled ? disabledTitle : t('db_analytics.databaseList.addDatabase')}
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
            <p>{t('db_analytics.databaseList.emptyTitle')}</p>
            <button
              className={styles['db-analytics-db-panel__link']}
              onClick={onManage}
              disabled={isDisabled}
              title={disabledTitle}
            >
              {t('db_analytics.databaseList.connectADatabase')}
            </button>
          </div>
        ) : (
          <ul className={styles['db-analytics-db-list']}>
            {databases.map((db, index) => {
              const typeMeta = getDbTypeMeta(db.db_type);
              const isCsv = db.db_type === 'csv';
              const schemaEnabled = isCsv || db.connected;
              const statusLabel = isCsv
                ? t('db_analytics.databaseList.status.ready')
                : db.connected
                ? t('db_analytics.databaseList.status.connected')
                : t('db_analytics.databaseList.status.disconnected');
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
                      aria-label={t('db_analytics.databaseList.viewSchemaAndErDiagram')}
                      title={schemaEnabled ? t('db_analytics.databaseList.viewSchema') : t('db_analytics.databaseList.connectToViewSchema')}
                    >
                      <Icon.schema size={13} aria-hidden="true" />
                    </button>
                    <span
                      className={`${styles['db-analytics-db-status-dot']} ${
                        schemaEnabled
                          ? styles['db-analytics-db-status-dot--connected']
                          : styles['db-analytics-db-status-dot--disconnected']
                      }`}
                      title={statusLabel}
                      aria-label={statusLabel}
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

















//DBAnalytics.tsx
import React, { useEffect, useState, useCallback, useRef } from 'react';
import { useTranslation } from 'react-i18next';
import styles from './DBAnalytics.module.scss';
import SessionHistory from './components/SessionHistory';
import DatabaseList from './components/DatabaseList';
import ChatPanel from './components/ChatPanel';
import ResultsPanel from './components/ResultsPanel';
import DatabaseManagerSlider from './components/DatabaseManagerSlider';
import NewSessionSlider from './components/NewSessionSlider';
import LanguageSwitcher from './components/LanguageSwitcher';
import { ChatMessage, ChartGroup, DatabaseItem, SessionItem, SuggestedPrompt } from './types';
import { api, MissingDbConnectionError } from '../../services/api';
import { Icon } from './icons';

const STREAM_MESSAGE_ID = 'stream-in-progress';
const CHAT_COL_MIN_WIDTH = 380;
const CHAT_COL_DEFAULT_WIDTH = 380;
// Leaves the nav column (300px) and a sensible minimum for the results
// column so dragging the handle all the way right can't squeeze column 3
// down to nothing.
const RESULTS_COL_MIN_WIDTH = 320;
const NAV_COL_WIDTH = 300;

const DBAnalytics: React.FC = () => {
  const { t } = useTranslation();
  const [sessions, setSessions] = useState<SessionItem[]>([]);
  const [databases, setDatabases] = useState<DatabaseItem[]>([]);
  const [activeSessionId, setActiveSessionId] = useState<string | null>(null);
  const [messages, setMessages] = useState<ChatMessage[]>([]);

  const [loading, setLoading] = useState(true);
  const [loadError, setLoadError] = useState<string | null>(null);
  // Distinct from `loading` (the initial full-app load) — tracks a
  // subsequent, database-list-only refresh triggered after creating a
  // connection or uploading a CSV, so column 1 can show a brief loading
  // state without re-triggering the whole app's first-load skeleton.
  const [databasesRefreshing, setDatabasesRefreshing] = useState(false);

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

  // True for the whole span of "select a session and wait for its history
  // plus starter prompts / follow-ups to arrive" or "send a message and
  // wait for the streamed answer plus its follow-up suggestions" — treated
  // as one unit of work so the composer stays locked the entire time,
  // rather than unlocking the moment streaming/history-loading finishes but
  // before the trailing suggestion fetch has actually landed.
  const isAwaitingResponseUnit = messagesLoading || isStreaming || suggestedPromptsLoading || followupsLoading;

  const [dbSliderOpen, setDbSliderOpen] = useState(false);
  const [sessionSliderOpen, setSessionSliderOpen] = useState(false);

  // --- Resizable chat column (column 2) ---
  const [chatColWidth, setChatColWidth] = useState(CHAT_COL_DEFAULT_WIDTH);
  const [isResizing, setIsResizing] = useState(false);
  const bodyRef = useRef<HTMLDivElement>(null);
  const dragStartXRef = useRef(0);
  const dragStartWidthRef = useRef(CHAT_COL_DEFAULT_WIDTH);

  // Tracks the session any in-flight async work (stream, follow-ups, message
  // refresh) belongs to. Every async callback checks this before touching
  // state, so results for a session the user has since navigated away from
  // never leak into the currently active session's view.
  const activeSessionRef = useRef<string | null>(null);

  // Cancels the actual in-flight `getMessages` network request (not just
  // discarding its result) whenever the user selects a different session
  // before the previous one's history has finished loading — e.g. rapidly
  // clicking through multiple session cards.
  const messagesAbortRef = useRef<AbortController | null>(null);

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

  // Re-fetches just the database list from the server. Used after creating
  // a connection or uploading a CSV — rather than trusting the single
  // object the create/upload response hands back (which has repeatedly
  // come back with missing/stale fields right after creation on this
  // backend) and splicing it into local state, this pulls the authoritative
  // list so column 1 always reflects what the server actually has.
  const refreshDatabases = useCallback(async () => {
    setDatabasesRefreshing(true);
    try {
      const dbList = await api.listDatabases();
      setDatabases(dbList);
    } catch (err) {
      setLoadError(err instanceof Error ? err.message : 'Failed to refresh databases.');
    } finally {
      setDatabasesRefreshing(false);
    }
  }, []);

  const fetchSuggestedPrompts = useCallback((sessionId: string, isStale: () => boolean) => {
    setSuggestedPromptsLoading(true);
    setSuggestedPromptsError(null);
    api
      .getSuggestedPrompts(sessionId)
      .then((res) => {
        if (!isStale()) setSuggestedPrompts(res.prompts);
      })
      .catch((err) => {
        if (!isStale()) {
          setSuggestedPromptsError(err instanceof Error ? err.message : 'Failed to load suggested prompts.');
        }
      })
      .finally(() => {
        if (!isStale()) setSuggestedPromptsLoading(false);
      });
  }, []);

  const fetchFollowups = useCallback((sessionId: string, isStale: () => boolean) => {
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
  }, []);

  const loadSessionData = useCallback(
    (sessionId: string) => {
      activeSessionRef.current = sessionId;
      const isStale = () => activeSessionRef.current !== sessionId;

      // Cancel whatever getMessages request is still in flight from a
      // previously selected session — this is a real network-level abort,
      // not just a "discard the result" guard, so switching between session
      // cards quickly doesn't leave multiple stacked requests running.
      messagesAbortRef.current?.abort();
      const controller = new AbortController();
      messagesAbortRef.current = controller;

      setMessages([]);
      setSuggestedPrompts([]);
      setSuggestedPromptsError(null);
      setFollowups([]);
      setFollowupsLoading(false);
      setStreamSteps([]);
      setStreamError(null);
      setIsStreaming(false);
      setQueryMissingDbIds([]);
      setMessagesLoading(true);
      setMessagesError(null);

      api
        .getMessages(sessionId, controller.signal)
        .then((msgs) => {
          if (isStale()) return;
          setMessages(msgs);

          if (msgs.length === 0) {
            fetchSuggestedPrompts(sessionId, isStale);
          } else {
            fetchFollowups(sessionId, isStale);
          }
        })
        .catch((err) => {
          // A cancelled request (because the user picked another session
          // before this one resolved) isn't a real error — nothing to show,
          // the newer request's own handler already owns the UI state.
          if (controller.signal.aborted) return;
          if (isStale()) return;
          setMessagesError(err instanceof Error ? err.message : 'Failed to load messages.');
          setMessages([]);
          // A brand-new session's messages endpoint can 404 briefly right
          // after creation (same server propagation lag seen elsewhere) even
          // though the session itself exists and has no conversation yet.
          // Don't let that failure also hide starter prompts — treat it the
          // same as an empty conversation and try to fetch them anyway.
          fetchSuggestedPrompts(sessionId, isStale);
        })
        .finally(() => {
          if (controller.signal.aborted) return;
          if (!isStale()) setMessagesLoading(false);
        });
    },
    [fetchSuggestedPrompts, fetchFollowups]
  );

  // Load chat history whenever the active session changes, and — depending
  // on whether it already has a conversation — either show starter prompts
  // (empty session) or fetch follow-up questions right away (existing session).
  useEffect(() => {
    if (!activeSessionId) {
      activeSessionRef.current = null;
      setMessages([]);
      setSuggestedPrompts([]);
      setSuggestedPromptsError(null);
      setFollowups([]);
      setFollowupsLoading(false);
      setStreamSteps([]);
      setStreamError(null);
      setIsStreaming(false);
      setQueryMissingDbIds([]);
      setMessagesLoading(false);
      return;
    }
    loadSessionData(activeSessionId);
  }, [activeSessionId, loadSessionData]);

  const activeSession = sessions.find((s) => s.id === activeSessionId) ?? null;

  // Databases this session depends on that are currently disconnected —
  // checked proactively from the session's db_ids on load, and reactively
  // whenever the query endpoint reports missing connections via a 424.
  const disconnectedSessionDbs: DatabaseItem[] = activeSession
    ? databases.filter(
        (db) =>
          ((activeSession.db_ids ?? []).includes(db.id) || queryMissingDbIds.includes(db.id)) &&
          !db.connected &&
          db.db_type !== 'csv'
      )
    : [];
  const isBlockedByDisconnectedDb = disconnectedSessionDbs.length > 0;

  // Column 3 groups graphs by the prompt that produced them — every
  // assistant response that came back with at least one graph gets its own
  // section, labeled with the user's question, newest first. Responses that
  // didn't produce any graphs are skipped entirely (nothing to show).
  const chartGroups: ChartGroup[] = (() => {
    const groups: ChartGroup[] = [];
    let pendingPrompt = '';

    for (const msg of messages) {
      if (msg.role === 'user') {
        pendingPrompt = msg.content;
        continue;
      }
      if (msg.role === 'assistant' && msg.graphs && msg.graphs.length > 0) {
        groups.push({ id: msg.id, prompt: pendingPrompt, graphs: msg.graphs });
      }
    }

    return groups.reverse();
  })();

  const handleSelectSession = (id: string) => {
    if (isStreaming) return;
    setActiveSessionId(id);
  };

  const handleSendMessage = async (text: string) => {
    const sessionId = activeSessionId;
    if (!sessionId || isAwaitingResponseUnit || isBlockedByDisconnectedDb) return;

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
    // Splice the returned object in immediately so the UI feels responsive,
    // then refresh from the server right after — the create/upload response
    // has repeatedly come back with incomplete fields on this backend, and
    // a CSV upload in particular can produce a database whose shape isn't
    // fully known client-side (e.g. "separate" mode). The refresh is the
    // authoritative source of truth; this optimistic append just avoids a
    // blank-feeling gap before it resolves.
    setDatabases((prev) => [...prev, db]);
    refreshDatabases();
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
    // De-duplicate defensively: if a rapid double-submit (or any other
    // race) ever produces two session entries with the same id, keep only
    // the newest one rather than showing duplicate rows in the list.
    setSessions((prev) => [session, ...prev.filter((s) => s.id !== session.id)]);
    setSessionSliderOpen(false);

    if (session.id === activeSessionId) {
      // Selecting "the same" session id wouldn't normally re-trigger the
      // data-loading effect (React bails out on an unchanged state value),
      // which would leave a stale view showing no messages/prompts for a
      // session that's actually brand new. Force the effect to run again
      // for this session explicitly.
      loadSessionData(session.id);
    } else {
      setActiveSessionId(session.id);
    }
  };

  const handleResizeStart = (e: React.PointerEvent<HTMLDivElement>) => {
    e.preventDefault();
    dragStartXRef.current = e.clientX;
    dragStartWidthRef.current = chatColWidth;
    setIsResizing(true);
  };

  useEffect(() => {
    if (!isResizing) return;

    const handlePointerMove = (e: PointerEvent) => {
      const delta = e.clientX - dragStartXRef.current;
      const bodyWidth = bodyRef.current?.clientWidth ?? Infinity;
      const maxWidth = Math.max(CHAT_COL_MIN_WIDTH, bodyWidth - NAV_COL_WIDTH - RESULTS_COL_MIN_WIDTH);
      const nextWidth = Math.min(maxWidth, Math.max(CHAT_COL_MIN_WIDTH, dragStartWidthRef.current + delta));
      setChatColWidth(nextWidth);
    };

    const handlePointerUp = () => setIsResizing(false);

    window.addEventListener('pointermove', handlePointerMove);
    window.addEventListener('pointerup', handlePointerUp);
    return () => {
      window.removeEventListener('pointermove', handlePointerMove);
      window.removeEventListener('pointerup', handlePointerUp);
    };
  }, [isResizing]);

  return (
    <div className={styles['db-analytics']}>
      <header className={styles['db-analytics__feature-header']}>
        <div className={styles['db-analytics__feature-header-main']}>
          <span className={styles['db-analytics__feature-header-icon']}>
            <Icon.databaseInsights size={18} aria-hidden="true" />
          </span>
          <div className={styles['db-analytics__feature-header-text']}>
            <h1 className={styles['db-analytics__feature-header-title']}>{t('db_analytics.app.title')}</h1>
            <p className={styles['db-analytics__feature-header-subtitle']}>{t('db_analytics.app.subtitle')}</p>
          </div>
        </div>
        <div className={styles['db-analytics__feature-header-actions']}>
          <LanguageSwitcher />
          <span className={styles['db-analytics__feature-header-badge']}>
            <span className={styles['db-analytics__feature-header-badge-dot']} aria-hidden="true" />
            <span className={styles['db-analytics__feature-header-badge-text']}>
              {t('db_analytics.app.databasesConnected', { count: databases.filter((db) => db.connected).length })}
            </span>
          </span>
        </div>
      </header>


      <main
        ref={bodyRef}
        className={`${styles['db-analytics__body']} ${isResizing ? styles['db-analytics--resizing'] : ''}`}
        style={{ ['--db-analytics-chat-col-width' as string]: `${chatColWidth}px` }}
      >
        <section className={`${styles['db-analytics__col']} ${styles['db-analytics__col--nav']}`}>
          <SessionHistory
            sessions={sessions}
            databases={databases}
            activeSessionId={activeSessionId}
            onSelect={handleSelectSession}
            onNewSession={() => setSessionSliderOpen(true)}
            disabled={isStreaming}
            loading={loading}
          />
          <DatabaseList
            databases={databases}
            onManage={() => setDbSliderOpen(true)}
            disabled={isStreaming}
            loading={loading}
            refreshing={databasesRefreshing}
          />
        </section>

        <section className={`${styles['db-analytics__col']} ${styles['db-analytics__col--chat']}`}>
          <div
            className={`${styles['db-analytics__resize-handle']} ${
              isResizing ? styles['db-analytics__resize-handle--active'] : ''
            }`}
            onPointerDown={handleResizeStart}
            role="separator"
            aria-orientation="vertical"
            aria-label="Resize chat panel"
            title="Drag to resize"
          />
          <ChatPanel
            sessionTitle={activeSession?.name ?? (loading ? 'Loading…' : 'No session selected')}
            messages={messages}
            loading={loading || messagesLoading}
            hasActiveSession={!!activeSessionId}
            suggestedPrompts={suggestedPrompts}
            suggestedPromptsLoading={suggestedPromptsLoading}
            suggestedPromptsError={suggestedPromptsError}
            isStreaming={isStreaming}
            isAwaitingResponseUnit={isAwaitingResponseUnit}
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
          <ResultsPanel
            chartGroups={chartGroups}
            loadError={loadError ?? messagesError}
            isGenerating={isStreaming}
            sessionId={activeSessionId}
          />
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
