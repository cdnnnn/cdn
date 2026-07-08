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
                <React.Fragment key={db.id}>
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














//FollowupChips.tsx

import React from 'react';
import styles from './FollowupChips.module.scss';
import { Icon } from '../icons';

interface Props {
  followups: string[];
  loading?: boolean;
  onSelect: (text: string) => void;
}

const FollowupChips: React.FC<Props> = ({ followups = [], loading, onSelect }) => {
  if (!loading && followups.length === 0) return null;

  return (
    <div className={styles['db-analytics-followups']}>
      <span className={styles['db-analytics-followups__label']}>
        <Icon.sparkles size={12} aria-hidden="true" />
        {loading ? 'Finding follow-up questions…' : 'Continue exploring'}
      </span>
      {!loading && (
        <div className={styles['db-analytics-followups__list']}>
          {followups.map((text, i) => (
            <button key={i} className={styles['db-analytics-followups__chip']} onClick={() => onSelect(text)}>
              <span className={styles['db-analytics-followups__chip-icon']}>
                <Icon.messageCircle size={12} aria-hidden="true" />
              </span>
              <span className={styles['db-analytics-followups__chip-text']}>{text}</span>
              <span className={styles['db-analytics-followups__chip-arrow']}>
                <Icon.arrowRight size={13} aria-hidden="true" />
              </span>
            </button>
          ))}
        </div>
      )}
    </div>
  );
};

export default FollowupChips;













// SuggestedPrompts.tsx

import React from 'react';
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
  if (loading) {
    return (
      <div className={styles['db-analytics-suggested-prompts']}>
        <div className={styles['db-analytics-suggested-prompts__intro']}>
          <Icon.loader size={20} aria-hidden="true" className={styles['db-analytics-suggested-prompts__spin']} />
          <p>Finding good questions to start with…</p>
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
        <p>Not sure where to start? Try one of these.</p>
      </div>

      <div className={styles['db-analytics-suggested-prompts__grid']}>
        {prompts.map((prompt, i) => (
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
            <p className={styles['db-analytics-suggested-prompts__text']}>{prompt.text}</p>
            <span className={styles['db-analytics-suggested-prompts__cta']}>
              Ask this
              <Icon.arrowRight size={13} aria-hidden="true" />
            </span>
          </button>
        ))}
      </div>
    </div>
  );
};

export default SuggestedPrompts;









// api.ts

const BASE_URL = 'http://107.108.32.188:8000';

// TODO: replace with your actual static token, or better, load it from an env variable
// (e.g. import.meta.env.VITE_API_TOKEN) instead of hardcoding it here.
const AUTH_TOKEN = 'YOUR_STATIC_TOKEN_HERE';

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

async function request<T>(path: string, options?: RequestInit): Promise<T> {
  const res = await fetch(`${BASE_URL}${path}`, {
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${AUTH_TOKEN}`,
    },
    ...options,
  });
  if (!res.ok) {
    let detail = '';
    try {
      const body = await res.json();
      detail = body?.message || body?.detail || '';
    } catch {
      // ignore parse errors
    }
    throw new Error(detail || `Request failed with status ${res.status}`);
  }
  const text = await res.text();
  return (text ? JSON.parse(text) : null) as T;
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

async function requestList<T>(path: string, options?: RequestInit): Promise<T[]> {
  const raw = await request<unknown>(path, options);
  return unwrapList<T>(raw);
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

async function* streamQuery(sessionId: string, query: string): AsyncGenerator<ApiQueryEvent> {
  const res = await fetch(`${BASE_URL}/sessions/${sessionId}/query`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${AUTH_TOKEN}`,
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

async function uploadCsv(payload: UploadCsvPayload): Promise<ApiDatabase> {
  const formData = new FormData();
  formData.append('file', payload.file);
  formData.append('name', payload.name);
  formData.append('table_name', payload.table_name);

  const res = await fetch(`${BASE_URL}/dbs/upload-csv`, {
    method: 'POST',
    headers: {
      // No Content-Type here on purpose — the browser sets
      // multipart/form-data with the correct boundary automatically when
      // the body is a FormData instance. Setting it manually breaks upload.
      Authorization: `Bearer ${AUTH_TOKEN}`,
    },
    body: formData,
  });

  if (!res.ok) {
    let detail = '';
    try {
      const body = await res.json();
      detail = body?.message || body?.detail || '';
    } catch {
      // ignore parse errors
    }
    throw new Error(detail || `Request failed with status ${res.status}`);
  }

  return res.json();
}

export const api = {
  listDatabases: () => requestList<ApiDatabase>('/dbs'),

  listSessions: () => requestList<ApiSession>('/sessions'),

  createDatabase: (payload: CreateDbPayload) =>
    request<ApiDatabase>('/dbs', {
      method: 'POST',
      body: JSON.stringify(payload),
    }),

  uploadCsv,

  connectDatabase: (dbId: string, password: string) =>
    request<{ message: string; db_id: string; cache_until: string }>(
      `/dbs/${dbId}/connect`,
      {
        method: 'POST',
        body: JSON.stringify({ password }),
      }
    ),

  disconnectDatabase: (dbId: string) =>
    request<{ message: string; db_id: string }>(`/dbs/${dbId}/disconnect`, {
      method: 'POST',
    }),

  getSchema: (dbId: string) => request<ApiSchema>(`/dbs/${dbId}/schema`),

  createSession: (payload: CreateSessionPayload) =>
    request<ApiSession>('/sessions', {
      method: 'POST',
      body: JSON.stringify(payload),
    }),

  getMessages: (sessionId: string) => requestList<ApiMessage>(`/sessions/${sessionId}/messages`),

  getSuggestedPrompts: async (sessionId: string): Promise<ApiSuggestedPromptsResponse> => {
    const raw = await request<Partial<ApiSuggestedPromptsResponse>>(
      `/sessions/${sessionId}/suggested-prompts`,
      { method: 'POST' }
    );
    return { prompts: Array.isArray(raw?.prompts) ? raw.prompts : [] };
  },

  streamQuery,

  getFollowups: async (sessionId: string): Promise<ApiFollowupsResponse> => {
    const raw = await request<Partial<ApiFollowupsResponse>>(`/sessions/${sessionId}/followups`, {
      method: 'POST',
    });
    return { followups: Array.isArray(raw?.followups) ? raw.followups : [] };
  },
};
