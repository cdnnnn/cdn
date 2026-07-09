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
    if (creating) return;
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
      setName('');
      setSelectedIds([]);
      onCreated(session);
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
        .getMessages(sessionId)
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
