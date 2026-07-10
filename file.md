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
  const isDisabled = !!disabled || !!loading;
  const disabledTitle = loading ? 'Loading databases…' : disabled ? 'Wait for the current response to finish' : undefined;
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
            disabled={isDisabled}
            title={isDisabled ? disabledTitle : 'Manage databases'}
          >
            <Icon.settings size={15} aria-hidden="true" />
          </button>
          <button
            className={styles['db-analytics-db-panel__icon-btn']}
            aria-label="Add database connection"
            onClick={onManage}
            disabled={isDisabled}
            title={isDisabled ? disabledTitle : 'Add database'}
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
              disabled={isDisabled}
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









//SessionHistory.tsx
import React, { useEffect, useRef, useState } from 'react';
import styles from './SessionHistory.module.scss';
import { SessionItem, DatabaseItem } from '../types';
import { Icon } from '../icons';
import SkeletonListItem from './SkeletonListItem';

interface Props {
  sessions: SessionItem[];
  databases: DatabaseItem[];
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
  databases,
  activeSessionId,
  onSelect,
  onNewSession,
  disabled,
  loading,
}) => {
  const [openDbListFor, setOpenDbListFor] = useState<string | null>(null);
  const popoverRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!openDbListFor) return;
    const onClickOutside = (e: MouseEvent) => {
      if (popoverRef.current && !popoverRef.current.contains(e.target as Node)) {
        setOpenDbListFor(null);
      }
    };
    document.addEventListener('mousedown', onClickOutside);
    return () => document.removeEventListener('mousedown', onClickOutside);
  }, [openDbListFor]);

  const getSessionDatabases = (session: SessionItem): DatabaseItem[] => {
    const ids = session.db_ids ?? [];
    return databases.filter((db) => ids.includes(db.id));
  };

  return (
    <div className={styles['db-analytics-session-panel']}>
      <div className={styles['db-analytics-session-panel__header']}>
        <h2 className={styles['db-analytics-session-panel__title']}>Chat sessions</h2>
        <button
          className={styles['db-analytics-session-panel__icon-btn']}
          aria-label="New chat session"
          onClick={onNewSession}
          disabled={disabled || loading}
          title={loading ? 'Loading sessions…' : disabled ? 'Wait for the current response to finish' : undefined}
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
              const sessionDbs = getSessionDatabases(session);
              const isPopoverOpen = openDbListFor === session.id;
              return (
                <li
                  key={session.id ? `${session.id}-${index}` : `session-${index}`}
                  className={styles['db-analytics-session-item-wrap']}
                >
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
                        <span
                          className={styles['db-analytics-session-item__db-pill']}
                          role="button"
                          tabIndex={0}
                          onClick={(e) => {
                            e.stopPropagation();
                            setOpenDbListFor(isPopoverOpen ? null : session.id);
                          }}
                          onKeyDown={(e) => {
                            if (e.key === 'Enter' || e.key === ' ') {
                              e.preventDefault();
                              e.stopPropagation();
                              setOpenDbListFor(isPopoverOpen ? null : session.id);
                            }
                          }}
                          title="View connected databases"
                        >
                          <Icon.database size={10} aria-hidden="true" />
                          {(session.db_ids ?? []).length}
                        </span>
                        <span className={styles['db-analytics-session-item__time']}>
                          {formatSessionDate(session.created_at)}
                        </span>
                      </span>
                    </span>
                  </button>

                  {isPopoverOpen && (
                    <div ref={popoverRef} className={styles['db-analytics-session-db-popover']}>
                      <span className={styles['db-analytics-session-db-popover__title']}>
                        Connected databases
                      </span>
                      {sessionDbs.length === 0 ? (
                        <span className={styles['db-analytics-session-db-popover__empty']}>
                          No matching databases found.
                        </span>
                      ) : (
                        <ul className={styles['db-analytics-session-db-popover__list']}>
                          {sessionDbs.map((db) => (
                            <li key={db.id} className={styles['db-analytics-session-db-popover__item']}>
                              <Icon.database size={11} aria-hidden="true" />
                              <span className={styles['db-analytics-session-db-popover__name']}>{db.name}</span>
                              <span
                                className={`${styles['db-analytics-session-db-popover__status']} ${
                                  db.connected ? styles['db-analytics-session-db-popover__status--on'] : ''
                                }`}
                              />
                            </li>
                          ))}
                        </ul>
                      )}
                    </div>
                  )}
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












//DBAnalytics.tsx

import React, { useEffect, useState, useCallback, useRef } from 'react';
import styles from './DBAnalytics.module.scss';
import SessionHistory from './components/SessionHistory';
import DatabaseList from './components/DatabaseList';
import ChatPanel from './components/ChatPanel';
import ResultsPanel from './components/ResultsPanel';
import DatabaseManagerSlider from './components/DatabaseManagerSlider';
import NewSessionSlider from './components/NewSessionSlider';
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







//api.ts
// TODO: point this at your project's actual configured axios instance
// (the one with the auth-token interceptor already set up).
import { http } from '../lib/http';

const API_PREFIX = '/db-analytics';

export interface ApiDatabase {
  id: string;
  name: string;
  db_type: string;
  host?: string;
  port: number;
  db_name: string;
  username: string;
  created_at: string;
  connected: boolean;
  cache_until: string | null;
}

export interface ApiSession {
  id: string;
  name: string;
  db_ids: string[];
  created_at: string;
}

export interface ApiColumn {
  name: string;
  type: string;
  nullable: boolean;
  primary_key: boolean;
  default: string | null;
}

export interface ApiTable {
  name: string;
  columns: ApiColumn[];
  row_count: number;
}

export interface ApiSchema {
  db_id: string;
  db_name: string;
  tables: ApiTable[];
}

export interface CreateDbPayload {
  db_name: string;
  db_type: 'mysql' | 'postgres';
  host: string;
  name: string;
  port: number;
  username: string;
}

export interface CreateSessionPayload {
  name: string;
  db_ids: string[];
}

// Thrown when POST /sessions/:id/query responds 424 because one or more of
// the session's databases isn't currently connected. Carries the specific
// db ids so the UI can point at exactly which connections need attention.
export class MissingDbConnectionError extends Error {
  missingDbIds: string[];

  constructor(message: string, missingDbIds: string[]) {
    super(message);
    this.name = 'MissingDbConnectionError';
    this.missingDbIds = missingDbIds;
  }
}

export interface UploadCsvPayload {
  file: File;
  name: string;
  table_name: string;
}

export interface ApiGraphQuery {
  db_id: string;
  db_name: string;
  sql: string;
  row_count: number;
}

export type ApiChartType = 'kpi' | 'line' | 'bar' | 'pie' | 'area' | 'scatter' | 'table';

export interface ApiGraph {
  id: string;
  chart_type: ApiChartType;
  title: string;
  x_axis: string | null;
  y_axis: string | null;
  series: string[] | null;
  stacked: boolean;
  data: Record<string, string | number>[];
  queries: ApiGraphQuery[];
}

export interface ApiMessage {
  id: string;
  role: 'user' | 'assistant';
  content: string;
  graphs: ApiGraph[];
  created_at: string;
}

export interface ApiSuggestedPrompt {
  text: string;
  label: string;
  db_name: string;
}

export interface ApiSuggestedPromptsResponse {
  prompts: ApiSuggestedPrompt[];
}

export interface ApiQueryStepEvent {
  type: 'step';
  step: string;
}

export interface ApiQueryMessageEvent {
  type: 'message';
  step: string;
}

export interface ApiQueryDoneEvent {
  type: 'done';
}

export interface ApiQueryErrorEvent {
  type: 'error';
  message: string;
}

export type ApiQueryEvent =
  | ApiQueryStepEvent
  | ApiQueryMessageEvent
  | ApiQueryDoneEvent
  | ApiQueryErrorEvent;

export interface ApiFollowupsResponse {
  followups: string[];
}

export interface ChartInsightPayload {
  chart: ApiGraph;
  question: string;
}

export interface ChartInsightResponse {
  insight: string;
}

// Some list endpoints inconsistently respond either as a bare array
// (`[...]`) or wrapped in an envelope (`{ status: "Success", result: [...] }`).
// This normalizes either shape into a plain array so callers never have to
// guess which one they got — and never crash calling array methods on an
// object if the envelope shape shows up unexpectedly.
function unwrapList<T>(payload: unknown): T[] {
  if (Array.isArray(payload)) return payload;
  if (payload && typeof payload === 'object' && Array.isArray((payload as { result?: unknown }).result)) {
    return (payload as { result: T[] }).result;
  }
  return [];
}

async function getList<T>(path: string, signal?: AbortSignal): Promise<T[]> {
  const res = await http.get(`${API_PREFIX}${path}`, { signal });
  return unwrapList<T>(res.data);
}

// Guarantees every field the UI relies on is present, and unwraps a
// possible `{ status, result: {...} }` envelope first — createSession was
// previously skipping that step, which meant a wrapped response left every
// field (including id) undefined and silently broke session selection,
// message loading, and everything downstream of it.
function normalizeSession(raw: unknown): ApiSession {
  const obj = unwrapObject<Partial<ApiSession>>(raw) ?? {};
  return {
    id: obj.id ?? '',
    name: obj.name ?? '',
    db_ids: Array.isArray(obj.db_ids) ? obj.db_ids : [],
    created_at: obj.created_at ?? new Date().toISOString(),
  };
}

// Extracts a human-readable message from an axios error response, whatever
// shape the backend used (`detail` as a string, `message`, or nothing at all).
function extractErrorMessage(err: unknown, fallback: string): string {
  const anyErr = err as { response?: { data?: { message?: string; detail?: unknown } }; message?: string };
  const data = anyErr?.response?.data;
  if (typeof data?.detail === 'string') return data.detail;
  if (typeof data?.message === 'string') return data.message;
  if (typeof anyErr?.message === 'string') return anyErr.message;
  return fallback;
}

// Parses a streaming response body into individual JSON payloads as they
// arrive. Two framing styles are supported, detected automatically per line:
//  1) Newline-delimited JSON — one `{...}` object per line (no `data:` prefix,
//     no blank-line separators). This is what a lot of simple SSE-flavored
//     backends actually send even when the content-type says event-stream.
//  2) Standard SSE framing — `data: {...}` lines, events separated by a
//     blank line, potentially multi-line data blocks.
// Critically, this parses and yields as soon as a complete line/frame is
// available rather than waiting for the whole response to buffer, so the
// consumer can render each event the moment it arrives instead of getting
// everything dumped at once when the stream closes.
async function* parseEventStream(res: Response): AsyncGenerator<ApiQueryEvent> {
  if (!res.body) return;
  const reader = res.body.getReader();
  const decoder = new TextDecoder();
  let buffer = '';
  let sseDataBuffer: string[] = [];

  const tryParse = (raw: string): ApiQueryEvent | null => {
    const trimmed = raw.trim();
    if (!trimmed) return null;
    try {
      return JSON.parse(trimmed) as ApiQueryEvent;
    } catch {
      return null;
    }
  };

  const flushSseBuffer = function* (): Generator<ApiQueryEvent> {
    if (sseDataBuffer.length === 0) return;
    const joined = sseDataBuffer.join('\n');
    sseDataBuffer = [];
    const parsed = tryParse(joined);
    if (parsed) yield parsed;
  };

  const processLine = function* (line: string): Generator<ApiQueryEvent> {
    if (line.startsWith('data:')) {
      // Standard SSE frame: accumulate until a blank line flushes it.
      sseDataBuffer.push(line.slice(5).trim());
      return;
    }
    if (line.trim() === '') {
      // Blank line = SSE event boundary.
      yield* flushSseBuffer();
      return;
    }
    // Not SSE-prefixed — try treating the line as a standalone JSON event
    // (newline-delimited JSON framing).
    const parsed = tryParse(line);
    if (parsed) {
      yield parsed;
    }
  };

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    buffer += decoder.decode(value, { stream: true });

    const lines = buffer.split('\n');
    // Keep the last (possibly incomplete) line in the buffer for next read.
    buffer = lines.pop() ?? '';

    for (const line of lines) {
      yield* processLine(line);
    }
  }

  // Flush anything left after the stream closes.
  if (buffer.trim()) {
    yield* processLine(buffer);
  }
  yield* flushSseBuffer();
}

// NOTE: this one deliberately stays on raw `fetch` instead of the axios
// instance. Axios buffers the full response body before resolving (or needs
// extra adapter config to expose a readable stream), which defeats the
// purpose here — we need to read and yield each SSE/NDJSON chunk as it
// arrives so the UI can render step-by-step progress in real time. `fetch`'s
// `res.body` ReadableStream gives us that directly. The auth header is
// applied manually below since this bypasses the axios interceptor.
async function* streamQuery(sessionId: string, query: string): AsyncGenerator<ApiQueryEvent> {
  const authHeader = http.defaults.headers.common?.['Authorization'];

  const res = await fetch(`${http.defaults.baseURL ?? ''}${API_PREFIX}/sessions/${sessionId}/query`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      ...(authHeader ? { Authorization: String(authHeader) } : {}),
    },
    body: JSON.stringify({ query }),
  });

  if (!res.ok) {
    let detail = '';
    try {
      const body = await res.json();
      // 424 shape: { detail: { message: string, missing_db_ids: string[] } }
      if (res.status === 424 && body?.detail && typeof body.detail === 'object') {
        const message = body.detail.message || 'One or more databases are not connected.';
        const missingDbIds = Array.isArray(body.detail.missing_db_ids) ? body.detail.missing_db_ids : [];
        throw new MissingDbConnectionError(message, missingDbIds);
      }
      detail = typeof body?.detail === 'string' ? body.detail : body?.message || '';
    } catch (err) {
      if (err instanceof MissingDbConnectionError) throw err;
      // ignore parse errors from a non-JSON error body
    }
    throw new Error(detail || `Request failed with status ${res.status}`);
  }

  yield* parseEventStream(res);
}

// Mirrors unwrapList, but for single-object responses. Some create/action
// endpoints have been observed to wrap their result the same way list
// endpoints do (`{ status: "Success", result: {...} }`) instead of
// returning the object directly — this normalizes either shape.
function unwrapObject<T extends object>(payload: unknown): T {
  if (payload && typeof payload === 'object' && !Array.isArray(payload)) {
    const withResult = payload as { result?: unknown; status?: unknown };
    if (withResult.result && typeof withResult.result === 'object' && !Array.isArray(withResult.result)) {
      return withResult.result as T;
    }
  }
  return payload as T;
}

// Guarantees every field the UI relies on is present with a sane default,
// even if the create/upload response is missing some of them.
function normalizeDatabase(raw: unknown): ApiDatabase {
  const obj = unwrapObject<Partial<ApiDatabase>>(raw) ?? {};
  return {
    id: obj.id ?? '',
    name: obj.name ?? '',
    db_type: obj.db_type ?? '',
    host: obj.host,
    port: obj.port ?? 0,
    db_name: obj.db_name ?? '',
    username: obj.username ?? '',
    created_at: obj.created_at ?? new Date().toISOString(),
    connected: obj.connected ?? false,
    cache_until: obj.cache_until ?? null,
  };
}

async function uploadCsv(payload: UploadCsvPayload): Promise<ApiDatabase> {
  const formData = new FormData();
  formData.append('file', payload.file);
  formData.append('name', payload.name);
  formData.append('table_name', payload.table_name);

  try {
    const res = await http.post(`${API_PREFIX}/dbs/upload-csv`, formData, {
      headers: {
        // Let axios/the browser set the multipart boundary automatically —
        // don't hardcode 'multipart/form-data' here or the boundary gets lost.
        'Content-Type': 'multipart/form-data',
      },
    });
    return normalizeDatabase(res.data);
  } catch (err) {
    throw new Error(extractErrorMessage(err, 'Failed to upload CSV file.'));
  }
}

export const api = {
  listDatabases: () => getList<ApiDatabase>('/dbs'),

  listSessions: async () => {
    const raw = await getList<Partial<ApiSession> & { id: string }>('/sessions');
    return raw.map(normalizeSession);
  },

  createDatabase: async (payload: CreateDbPayload): Promise<ApiDatabase> => {
    try {
      const res = await http.post(`${API_PREFIX}/dbs`, payload);
      return normalizeDatabase(res.data);
    } catch (err) {
      throw new Error(extractErrorMessage(err, 'Failed to create database.'));
    }
  },

  uploadCsv,

  connectDatabase: async (
    dbId: string,
    password: string
  ): Promise<{ message: string; db_id: string; cache_until: string }> => {
    try {
      const res = await http.post(`${API_PREFIX}/dbs/${dbId}/connect`, { password });
      return res.data;
    } catch (err) {
      throw new Error(extractErrorMessage(err, 'Failed to connect.'));
    }
  },

  disconnectDatabase: async (dbId: string): Promise<{ message: string; db_id: string }> => {
    try {
      const res = await http.post(`${API_PREFIX}/dbs/${dbId}/disconnect`);
      return res.data;
    } catch (err) {
      throw new Error(extractErrorMessage(err, 'Failed to disconnect.'));
    }
  },

  getSchema: async (dbId: string): Promise<ApiSchema> => {
    const res = await http.get(`${API_PREFIX}/dbs/${dbId}/schema`);
    const body = unwrapObject<Partial<ApiSchema>>(res.data) ?? {};
    return {
      db_id: body.db_id ?? dbId,
      db_name: body.db_name ?? '',
      tables: Array.isArray(body.tables) ? body.tables : [],
    };
  },

  createSession: async (payload: CreateSessionPayload): Promise<ApiSession> => {
    try {
      const res = await http.post(`${API_PREFIX}/sessions`, payload);
      const normalized = normalizeSession(res.data);

      if (!normalized.id) {
        // A session with no id can never be selected/matched correctly
        // downstream — better to surface this clearly than silently hand
        // back something broken.
        throw new Error('The server did not return an id for the new session.');
      }

      // The create response has been observed to come back missing fields
      // right after creation (the server hasn't fully propagated everything
      // yet) — db_ids empty, name blank — even though what was submitted is
      // properly saved moments later. Rather than show a blank name or "0
      // databases" until the next full refresh, trust what we just sent as
      // a fallback whenever the response doesn't echo it back.
      const db_ids = normalized.db_ids.length > 0 ? normalized.db_ids : payload.db_ids;
      const name = normalized.name.trim().length > 0 ? normalized.name : payload.name;
      return { ...normalized, name, db_ids };
    } catch (err) {
      throw new Error(extractErrorMessage(err, 'Failed to create session.'));
    }
  },

  getMessages: (sessionId: string, signal?: AbortSignal) =>
    getList<ApiMessage>(`/sessions/${sessionId}/messages`, signal),

  getSuggestedPrompts: async (sessionId: string): Promise<ApiSuggestedPromptsResponse> => {
    const res = await http.post(`${API_PREFIX}/sessions/${sessionId}/suggested-prompts`);
    const body = unwrapObject<{ prompts?: unknown }>(res.data) ?? {};
    return { prompts: Array.isArray(body.prompts) ? (body.prompts as ApiSuggestedPrompt[]) : [] };
  },

  streamQuery,

  getFollowups: async (sessionId: string): Promise<ApiFollowupsResponse> => {
    const res = await http.post(`${API_PREFIX}/sessions/${sessionId}/followups`);
    const body = unwrapObject<{ followups?: unknown }>(res.data) ?? {};
    return { followups: Array.isArray(body.followups) ? (body.followups as string[]) : [] };
  },

  getChartInsight: async (sessionId: string, payload: ChartInsightPayload): Promise<ChartInsightResponse> => {
    try {
      const res = await http.post(`${API_PREFIX}/sessions/${sessionId}/chart_insight`, payload);
      const body = unwrapObject<{ insight?: unknown }>(res.data) ?? {};
      return { insight: typeof body.insight === 'string' ? body.insight : '' };
    } catch (err) {
      throw new Error(extractErrorMessage(err, 'Failed to get an insight for this question.'));
    }
  },
};
