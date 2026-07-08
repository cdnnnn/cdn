//SessionHistory.tsx
import React from 'react';
import styles from './SessionHistory.module.scss';
import { SessionItem } from '../types';
import { Icon } from '../icons';

interface Props {
  sessions: SessionItem[];
  activeSessionId: string | null;
  onSelect: (id: string) => void;
  onNewSession: () => void;
  disabled?: boolean;
}

const formatSessionDate = (iso: string) => {
  const date = new Date(iso);
  return date.toLocaleDateString(undefined, {
    day: 'numeric',
    month: 'short',
    year: 'numeric',
  });
};

const SessionHistory: React.FC<Props> = ({ sessions, activeSessionId, onSelect, onNewSession, disabled }) => {
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
        {sessions.length === 0 ? (
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






//DBAnalytics.tsx

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
          />
          <DatabaseList databases={databases} onManage={() => setDbSliderOpen(true)} disabled={isStreaming} />
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










// api.ts

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

async function getList<T>(path: string): Promise<T[]> {
  const res = await http.get(`${API_PREFIX}${path}`);
  return unwrapList<T>(res.data);
}

// Guarantees db_ids is always a real array. The backend has been observed to
// omit it or send null for sessions with no databases attached yet (e.g.
// right after creation), and `.includes()`/`.filter()` on undefined throws.
function normalizeSession(raw: Partial<ApiSession> & { id: string }): ApiSession {
  return {
    id: raw.id,
    name: raw.name ?? '',
    db_ids: Array.isArray(raw.db_ids) ? raw.db_ids : [],
    created_at: raw.created_at ?? new Date().toISOString(),
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
    return res.data;
  },

  createSession: async (payload: CreateSessionPayload): Promise<ApiSession> => {
    try {
      const res = await http.post(`${API_PREFIX}/sessions`, payload);
      return normalizeSession(res.data);
    } catch (err) {
      throw new Error(extractErrorMessage(err, 'Failed to create session.'));
    }
  },

  getMessages: (sessionId: string) => getList<ApiMessage>(`/sessions/${sessionId}/messages`),

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
};
