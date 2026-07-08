//DatabaseList.tsx

import React from 'react';
import styles from './DatabaseList.module.scss';
import { DatabaseItem } from '../types';
import { Icon } from '../icons';
import { getDbTypeMeta } from '../dbTypeMeta';
import SkeletonListItem from './SkeletonListItem';

interface Props {
  databases: DatabaseItem[];
  onManage: () => void;
  disabled?: boolean;
  loading?: boolean;
}

const DatabaseList: React.FC<Props> = ({ databases, onManage, disabled, loading }) => {
  const disabledTitle = disabled ? 'Wait for the current response to finish' : undefined;

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
            {databases.map((db) => {
              const typeMeta = getDbTypeMeta(db.db_type);
              const isCsv = db.db_type === 'csv';
              return (
                <li key={db.id} className={styles['db-analytics-db-item']}>
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
                  <span
                    className={`${styles['db-analytics-db-status-dot']} ${
                      isCsv || db.connected
                        ? styles['db-analytics-db-status-dot--connected']
                        : styles['db-analytics-db-status-dot--disconnected']
                    }`}
                    title={isCsv ? 'Ready' : db.connected ? 'Connected' : 'Disconnected'}
                    aria-label={isCsv ? 'Ready' : db.connected ? 'Connected' : 'Disconnected'}
                  />
                </li>
              );
            })}
          </ul>
        )}
      </div>
    </div>
  );
};

export default DatabaseList;












// SessionHistory.tsx


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
            {sessions.map((session) => {
              const isActive = session.id === activeSessionId;
              return (
                <li key={session.id}>
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












// SkeletonListItem.module.scss

@use '../../../styles/tokens' as t;

.db-analytics-skeleton-list {
  list-style: none;
  margin: 0;
  padding: 0;
  display: flex;
  flex-direction: column;
  gap: 3px;
}

.db-analytics-skeleton-item {
  display: flex;
  align-items: flex-start;
  gap: 10px;
  padding: 8px 10px;
}

.db-analytics-skeleton-item__icon {
  width: 28px;
  height: 28px;
  border-radius: 7px;
  flex-shrink: 0;
}

.db-analytics-skeleton-item__lines {
  display: flex;
  flex-direction: column;
  gap: 7px;
  flex: 1;
  min-width: 0;
  padding-top: 2px;
}

.db-analytics-skeleton-item__line {
  height: 9px;
  border-radius: 4px;
}

.db-analytics-skeleton-item__line--title {
  height: 10px;
}

.db-analytics-skeleton-item__line--meta {
  height: 8px;
}

.db-analytics-skeleton-item__meta-row {
  display: flex;
  align-items: center;
  gap: 8px;
}

.db-analytics-skeleton-item__pill {
  width: 28px;
  height: 14px;
  border-radius: 999px;
  flex-shrink: 0;
}

.db-analytics-skeleton-shimmer {
  background: linear-gradient(
    90deg,
    t.$surface-0 25%,
    rgba(11, 11, 11, 0.06) 50%,
    t.$surface-0 75%
  );
  background-size: 200% 100%;
  animation: db-analytics-skeleton-shimmer-move 1.5s ease-in-out infinite;
}

@keyframes db-analytics-skeleton-shimmer-move {
  0% {
    background-position: 200% 0;
  }
  100% {
    background-position: -200% 0;
  }
}












// SkeletonListItem.tsx


import React from 'react';
import styles from './SkeletonListItem.module.scss';

interface Props {
  /** Number of skeleton rows to render */
  count?: number;
  /** Row layout: 'session' shows a title + two meta pills, 'db' shows a title + one meta line */
  variant?: 'session' | 'db';
}

const SkeletonListItem: React.FC<Props> = ({ count = 4, variant = 'session' }) => {
  return (
    <ul className={styles['db-analytics-skeleton-list']} aria-hidden="true">
      {Array.from({ length: count }).map((_, i) => (
        <li key={i} className={styles['db-analytics-skeleton-item']}>
          <span
            className={`${styles['db-analytics-skeleton-item__icon']} ${styles['db-analytics-skeleton-shimmer']}`}
          />
          <span className={styles['db-analytics-skeleton-item__lines']}>
            <span
              className={`${styles['db-analytics-skeleton-item__line']} ${styles['db-analytics-skeleton-item__line--title']} ${styles['db-analytics-skeleton-shimmer']}`}
              style={{ width: `${62 + ((i * 13) % 28)}%` }}
            />
            {variant === 'session' ? (
              <span className={styles['db-analytics-skeleton-item__meta-row']}>
                <span
                  className={`${styles['db-analytics-skeleton-item__pill']} ${styles['db-analytics-skeleton-shimmer']}`}
                />
                <span
                  className={`${styles['db-analytics-skeleton-item__line']} ${styles['db-analytics-skeleton-item__line--meta']} ${styles['db-analytics-skeleton-shimmer']}`}
                  style={{ width: '38%' }}
                />
              </span>
            ) : (
              <span
                className={`${styles['db-analytics-skeleton-item__line']} ${styles['db-analytics-skeleton-item__line--meta']} ${styles['db-analytics-skeleton-shimmer']}`}
                style={{ width: `${40 + ((i * 9) % 20)}%` }}
              />
            )}
          </span>
        </li>
      ))}
    </ul>
  );
};

export default SkeletonListItem;










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
          ((activeSession.db_ids ?? []).includes(db.id) || queryMissingDbIds.includes(db.id)) &&
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
            loading={loading}
          />
          <DatabaseList
            databases={databases}
            onManage={() => setDbSliderOpen(true)}
            disabled={isStreaming}
            loading={loading}
          />
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
