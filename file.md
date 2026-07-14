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
  min-width: 0;
  overflow-y: auto;
  overflow-x: hidden;
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
  min-width: 0;
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
  min-width: 0;
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
    display: block;
    width: max-content;
    max-width: 100%;
    overflow-x: auto;
    border-collapse: collapse;
    margin: 8px 0;
    font-size: 12px;

    // A visible, non-intrusive scrollbar so it's clear the table itself
    // scrolls when it's too wide, rather than the page silently clipping it.
    scrollbar-width: thin;
    scrollbar-color: t.$border-stronger transparent;

    &::-webkit-scrollbar {
      height: 6px;
    }

    &::-webkit-scrollbar-track {
      background: transparent;
    }

    &::-webkit-scrollbar-thumb {
      background: t.$border-stronger;
      border-radius: 999px;
    }
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

.db-analytics-chat-composer__footer-start {
  display: flex;
  align-items: center;
  gap: 8px;
  min-width: 0;
  flex: 1;
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
import ModelSelectMenu from './ModelSelectMenu';

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
  models: string[];
  selectedModel: string;
  onModelChange: (model: string) => void;
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
  models = [],
  selectedModel,
  onModelChange,
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
            <div className={styles['db-analytics-chat-composer__footer-start']}>
              <ModelSelectMenu
                models={models}
                value={selectedModel}
                onChange={onModelChange}
                disabled={composerDisabled}
              />
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
            </div>
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






















//ExpandChartSlider.module.scss
@use '../../../styles/tokens' as t;

.db-analytics-expand-overlay {
  position: fixed;
  inset: 0;
  z-index: 150;
  display: flex;
  justify-content: flex-end;
}

.db-analytics-expand-overlay__backdrop {
  position: absolute;
  inset: 0;
  background: rgba(11, 11, 11, 0.32);
  animation: db-analytics-expand-fade-in 0.15s ease;
}

.db-analytics-expand-panel {
  position: relative;
  max-width: 100vw;
  height: 100%;
  background: t.$surface-2;
  display: flex;
  flex-direction: column;
  box-shadow: -8px 0 24px rgba(11, 11, 11, 0.14);
  animation: db-analytics-expand-slide-in 0.22s cubic-bezier(0.16, 1, 0.3, 1);
}

.db-analytics-expand-panel--resizing {
  animation: none;
  transition: none;
}

.db-analytics-expand-panel__resize-handle {
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

.db-analytics-expand-panel__resize-handle--active::after {
  background: t.$accent;
}

@keyframes db-analytics-expand-slide-in {
  from {
    transform: translateX(100%);
  }
  to {
    transform: translateX(0);
  }
}

@keyframes db-analytics-expand-fade-in {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

.db-analytics-expand-panel__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 64px;
  padding: 0 22px;
  border-bottom: 1px solid t.$border-strong;
  background: t.$surface-1;
  flex-shrink: 0;
}

.db-analytics-expand-panel__heading {
  display: flex;
  align-items: center;
  gap: 12px;
  min-width: 0;
}

.db-analytics-expand-panel__icon {
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

.db-analytics-expand-panel__title {
  font-size: 15px;
  font-weight: 600;
  color: t.$text-primary;
  margin: 0 0 2px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 420px;
}

.db-analytics-expand-panel__badge {
  font-size: 10px;
  font-weight: 600;
  letter-spacing: 0.02em;
  text-transform: uppercase;
  color: t.$text-secondary;
  background: t.$surface-0;
  border: 0.5px solid t.$border;
  padding: 2px 7px;
  border-radius: 999px;
}

.db-analytics-expand-panel__close-btn {
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

.db-analytics-expand-panel__body {
  flex: 1;
  overflow-y: auto;
  padding: 28px;
}
















//ExpandChartSlider.tsx
import React, { useEffect, useState } from 'react';
import { useTranslation } from 'react-i18next';
import styles from './ExpandChartSlider.module.scss';
import { ChartResult, ChartType } from '../types';
import { Icon, IconComponent } from '../icons';
import { renderChart, getChartIcon } from './ResultsPanel';
import { useResizableWidth } from '../hooks/useResizableWidth';

interface Props {
  chart: ChartResult | null;
  displayType: ChartType;
  onClose: () => void;
}

const ExpandChartSlider: React.FC<Props> = ({ chart, displayType, onClose }) => {
  const { t } = useTranslation();
  const { width, isResizing, handlePointerDown } = useResizableWidth({
    defaultWidth: 900,
    minWidth: 560,
    maxWidth: 'viewport',
  });
  // Sticky for the rest of this panel's lifetime once dragging starts — see
  // the matching comment in DbSchemaSlider.tsx for why this can't just be
  // tied to the live isResizing flag (it would re-trigger the entrance
  // @keyframes on drag-end and produce a visible snap-back-then-forward).
  const [hasResizedOnce, setHasResizedOnce] = useState(false);
  useEffect(() => {
    if (isResizing) setHasResizedOnce(true);
  }, [isResizing]);

  if (!chart) return null;

  const ChartIcon: IconComponent = getChartIcon(displayType);

  return (
    <div className={styles['db-analytics-expand-overlay']} role="dialog" aria-modal="true">
      <div className={styles['db-analytics-expand-overlay__backdrop']} onClick={onClose} />
      <aside
        className={`${styles['db-analytics-expand-panel']} ${
          hasResizedOnce ? styles['db-analytics-expand-panel--resizing'] : ''
        }`}
        style={{ width: `${width}px` }}
      >
        <div
          className={`${styles['db-analytics-expand-panel__resize-handle']} ${
            isResizing ? styles['db-analytics-expand-panel__resize-handle--active'] : ''
          }`}
          onPointerDown={handlePointerDown}
          role="separator"
          aria-orientation="vertical"
          aria-label={t('db_analytics.expandChartSlider.resizeHandle')}
          title={t('db_analytics.expandChartSlider.resizeHandle')}
        />
        <div className={styles['db-analytics-expand-panel__header']}>
          <div className={styles['db-analytics-expand-panel__heading']}>
            <span className={styles['db-analytics-expand-panel__icon']}>
              <ChartIcon size={16} aria-hidden="true" />
            </span>
            <div>
              <h2 className={styles['db-analytics-expand-panel__title']}>{chart.title}</h2>
              <span className={styles['db-analytics-expand-panel__badge']}>
                {t(`db_analytics.chartTypeMenu.types.${displayType}`, { defaultValue: displayType })}
              </span>
            </div>
          </div>
          <button
            className={styles['db-analytics-expand-panel__close-btn']}
            aria-label={t('db_analytics.expandChartSlider.close')}
            onClick={onClose}
          >
            <Icon.close size={18} aria-hidden="true" />
          </button>
        </div>

        <div className={styles['db-analytics-expand-panel__body']}>
          {renderChart(chart, 440, displayType)}
        </div>
      </aside>
    </div>
  );
};

export default ExpandChartSlider;
















//ModelSelectMenu.module.scss
@use '../../../styles/tokens' as t;

.db-analytics-model-menu {
  position: relative;
  flex-shrink: 0;
}

.db-analytics-model-menu__trigger {
  display: flex;
  align-items: center;
  gap: 5px;
  font-size: 10.5px;
  font-weight: 600;
  color: t.$text-secondary;
  background: t.$surface-0;
  border: 0.5px solid t.$border-strong;
  padding: 4px 8px;
  border-radius: 999px;
  cursor: pointer;
  font-family: inherit;
  white-space: nowrap;
  max-width: 130px;
  transition: background 0.15s ease, border-color 0.15s ease;

  &:hover:not(:disabled) {
    background: t.$surface-2;
    border-color: t.$border-stronger;
  }

  &:disabled {
    opacity: 0.55;
    cursor: not-allowed;
  }
}

.db-analytics-model-menu__label {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  min-width: 0;
}

.db-analytics-model-menu__chevron {
  transform: rotate(-90deg);
  color: t.$text-muted;
  flex-shrink: 0;
}

.db-analytics-model-menu__list {
  position: absolute;
  bottom: calc(100% + 6px);
  left: 0;
  z-index: 20;
  list-style: none;
  margin: 0;
  padding: 5px;
  min-width: 160px;
  max-height: 220px;
  overflow-y: auto;
  background: t.$surface-2;
  border: 1px solid t.$border-strong;
  border-radius: t.$radius;
  box-shadow: 0 -8px 22px rgba(11, 11, 11, 0.14);
  animation: db-analytics-model-menu-pop 0.12s ease;
}

@keyframes db-analytics-model-menu-pop {
  from {
    opacity: 0;
    transform: translateY(2px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.db-analytics-model-menu__option {
  display: flex;
  align-items: center;
  gap: 8px;
  width: 100%;
  padding: 7px 8px;
  border: none;
  background: none;
  border-radius: 6px;
  font-size: 12px;
  font-weight: 500;
  color: t.$text-primary;
  cursor: pointer;
  font-family: inherit;
  text-align: left;

  span {
    flex: 1;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &:hover {
    background: t.$surface-0;
  }
}

.db-analytics-model-menu__option--selected {
  background: t.$accent-bg;
  color: #0c447c;

  &:hover {
    background: t.$accent-bg;
  }
}

















//ModelSelectMenu.tsx
import React, { useEffect, useRef, useState } from 'react';
import { useTranslation } from 'react-i18next';
import styles from './ModelSelectMenu.module.scss';
import { Icon } from '../icons';

interface Props {
  models: string[];
  value: string;
  onChange: (model: string) => void;
  disabled?: boolean;
}

const ModelSelectMenu: React.FC<Props> = ({ models, value, onChange, disabled }) => {
  const { t } = useTranslation();
  const [open, setOpen] = useState(false);
  const rootRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!open) return;
    const onClickOutside = (e: MouseEvent) => {
      if (rootRef.current && !rootRef.current.contains(e.target as Node)) {
        setOpen(false);
      }
    };
    document.addEventListener('mousedown', onClickOutside);
    return () => document.removeEventListener('mousedown', onClickOutside);
  }, [open]);

  if (models.length === 0) return null;

  return (
    <div className={styles['db-analytics-model-menu']} ref={rootRef}>
      <button
        type="button"
        className={styles['db-analytics-model-menu__trigger']}
        onClick={() => !disabled && setOpen((o) => !o)}
        disabled={disabled}
        aria-haspopup="listbox"
        aria-expanded={open}
        title={t('db_analytics.chatPanel.composer.selectModel')}
      >
        <Icon.sparkles size={11} aria-hidden="true" />
        <span className={styles['db-analytics-model-menu__label']}>{value}</span>
        <Icon.arrowRight size={10} aria-hidden="true" className={styles['db-analytics-model-menu__chevron']} />
      </button>

      {open && (
        <ul className={styles['db-analytics-model-menu__list']} role="listbox">
          {models.map((model) => {
            const isSelected = model === value;
            return (
              <li key={model}>
                <button
                  type="button"
                  className={`${styles['db-analytics-model-menu__option']} ${
                    isSelected ? styles['db-analytics-model-menu__option--selected'] : ''
                  }`}
                  role="option"
                  aria-selected={isSelected}
                  onClick={() => {
                    onChange(model);
                    setOpen(false);
                  }}
                >
                  <span>{model}</span>
                  {isSelected && <Icon.check size={12} aria-hidden="true" />}
                </button>
              </li>
            );
          })}
        </ul>
      )}
    </div>
  );
};

export default ModelSelectMenu;
















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

  // --- LLM model selection ---
  const [models, setModels] = useState<string[]>([]);
  const [selectedModel, setSelectedModel] = useState('');

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

  // Fetched independently of the main app load — if this fails, the
  // composer simply won't show a model picker rather than blocking
  // everything else from loading.
  useEffect(() => {
    api
      .getModels()
      .then((res) => {
        setModels(res.models);
        setSelectedModel(res.default_model || res.models[0] || '');
      })
      .catch((err) => {
        console.error('Failed to load models:', err);
      });
  }, []);

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
      for await (const event of api.streamQuery(sessionId, text, selectedModel || undefined)) {
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
            onDatabaseConnected={handleDatabaseConnected}
            onDatabaseDisconnected={handleDatabaseDisconnected}
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
            models={models}
            selectedModel={selectedModel}
            onModelChange={setSelectedModel}
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





















//types.ts
import {
  ApiDatabase,
  ApiSession,
  ApiMessage,
  ApiGraph,
  ApiChartType,
  ApiSuggestedPrompt,
  ApiQueryEvent,
  ApiQueryStepEvent,
  ApiQueryMessageEvent,
  ApiQueryDoneEvent,
  ApiQueryErrorEvent,
  ChartInsightResponse,
  ApiSchema,
  ApiTable,
  ApiColumn,
  ApiModelsResponse,
} from '../../services/api';

export type DatabaseItem = ApiDatabase;
export type SessionItem = ApiSession;
export type ChatMessage = ApiMessage;
export type ChartResult = ApiGraph;
export type ChartType = ApiChartType;
export type SuggestedPrompt = ApiSuggestedPrompt;
export type QueryEvent = ApiQueryEvent;
export type QueryStepEvent = ApiQueryStepEvent;
export type QueryMessageEvent = ApiQueryMessageEvent;
export type QueryDoneEvent = ApiQueryDoneEvent;
export type QueryErrorEvent = ApiQueryErrorEvent;
export type ChartInsight = ChartInsightResponse;
export type DbSchema = ApiSchema;
export type DbTable = ApiTable;
export type DbColumn = ApiColumn;
export type ModelsResponse = ApiModelsResponse;

// A single prompt/response pairing in column 3 — the user's question and
// the graphs the assistant's reply to it produced. Only assistant messages
// that actually returned graphs become a group.
export interface ChartGroup {
  id: string;
  prompt: string;
  graphs: ChartResult[];
}



















//en.json
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
      "refreshingDatabases": "Refreshing databases…",
      "connect": "Connect",
      "disconnect": "Disconnect",
      "disconnecting": "Disconnecting…"
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
        "preparingFollowups": "Preparing follow-up suggestions…",
        "selectModel": "Select model"
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
      "close": "Close",
      "resizeHandle": "Drag to resize"
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
      "moreColumns_other": "+{{count}} more columns",
      "zoomIn": "Zoom in",
      "zoomOut": "Zoom out",
      "resetZoom": "Reset zoom"
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















//ko.json
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
      "refreshingDatabases": "데이터베이스 새로고침 중…",
      "connect": "연결",
      "disconnect": "연결 해제",
      "disconnecting": "연결 해제 중…"
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
        "preparingFollowups": "후속 질문을 준비하는 중…",
        "selectModel": "모델 선택"
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
      "close": "닫기",
      "resizeHandle": "드래그하여 크기 조절"
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
      "moreColumns_other": "열 {{count}}개 더 보기",
      "zoomIn": "확대",
      "zoomOut": "축소",
      "resetZoom": "확대/축소 초기화"
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
  db_type: 'mysql' | 'postgres' | 'mssql';
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

export type CsvUploadMode = 'combined' | 'separate';

export interface UploadCsvPayload {
  files: File[];
  name: string;
  mode: CsvUploadMode;
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

export interface ApiModelsResponse {
  models: string[];
  default_model: string;
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
  const anyErr = err as {
    response?: { status?: number; data?: { message?: string; detail?: unknown; status?: string; result?: unknown } };
    message?: string;
  };
  const data = anyErr?.response?.data;
  // Some endpoints (e.g. /upload-csv) respond with { status: "Error", result: "..." }
  // instead of the more common { detail } / { message } shape.
  if (data?.status === 'Error' && typeof data?.result === 'string') return data.result;
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
async function* streamQuery(
  sessionId: string,
  query: string,
  model?: string
): AsyncGenerator<ApiQueryEvent> {
  const authHeader = http.defaults.headers.common?.['Authorization'];

  const res = await fetch(`${http.defaults.baseURL ?? ''}${API_PREFIX}/sessions/${sessionId}/query`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      ...(authHeader ? { Authorization: String(authHeader) } : {}),
    },
    body: JSON.stringify(model ? { query, model } : { query }),
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
  // Multiple files share the same field name — this is the standard way to
  // send a file list via multipart/form-data; most backends (including
  // FastAPI's `List[UploadFile]`) expect repeated `files` entries rather
  // than an indexed/bracketed key.
  payload.files.forEach((file) => formData.append('files', file));
  formData.append('name', payload.name);
  formData.append('mode', payload.mode);

  try {
    const res = await http.post(`${API_PREFIX}/upload-csv`, formData, {
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

  // Unlike every other endpoint here, this one is a top-level path (not
  // nested under API_PREFIX) — confirmed intentional, not an oversight.
  getModels: async (): Promise<ApiModelsResponse> => {
    try {
      const res = await http.get('/prompt/get_models');
      const body = unwrapObject<{ models?: unknown; default_model?: unknown }>(res.data) ?? {};
      const models = Array.isArray(body.models) ? (body.models as string[]) : [];
      const default_model = typeof body.default_model === 'string' ? body.default_model : models[0] ?? '';
      return { models, default_model };
    } catch (err) {
      throw new Error(extractErrorMessage(err, 'Failed to load the list of models.'));
    }
  },
};
