//ChartAskAiSlider.tsx
import React, { useEffect, useRef, useState, KeyboardEvent } from 'react';
import { useTranslation } from 'react-i18next';
import ReactMarkdown from 'react-markdown';
import remarkGfm from 'remark-gfm';
import styles from './ChartAskAiSlider.module.scss';
import { ChartResult, ChartType } from '../types';
import { Icon, IconComponent } from '../icons';
import { renderChart, getChartIcon } from './ResultsPanel';
import { api } from '../../../services/api';

interface QaEntry {
  id: string;
  question: string;
  answer?: string;
  loading?: boolean;
  error?: string | null;
}

interface Props {
  sessionId: string | null;
  chart: ChartResult | null;
  displayType: ChartType;
  onClose: () => void;
}

const ChartAskAiSlider: React.FC<Props> = ({ sessionId, chart, displayType, onClose }) => {
  const { t } = useTranslation();
  const [draft, setDraft] = useState('');
  const [entries, setEntries] = useState<QaEntry[]>([]);
  const [isAsking, setIsAsking] = useState(false);
  const threadEndRef = useRef<HTMLDivElement>(null);

  // Reset the conversation whenever a different chart is opened, so
  // previous questions don't linger against the wrong chart.
  useEffect(() => {
    setEntries([]);
    setDraft('');
    setIsAsking(false);
  }, [chart?.id]);

  useEffect(() => {
    threadEndRef.current?.scrollIntoView({ behavior: 'smooth', block: 'end' });
  }, [entries]);

  if (!chart) return null;

  const ChartIcon: IconComponent = getChartIcon(displayType);

  const submit = async () => {
    const question = draft.trim();
    if (!question || isAsking || !sessionId) return;

    const entryId = `qa-${Date.now()}`;
    setEntries((prev) => [...prev, { id: entryId, question, loading: true }]);
    setDraft('');
    setIsAsking(true);

    try {
      const res = await api.getChartInsight(sessionId, { chart, question });
      setEntries((prev) =>
        prev.map((e) => (e.id === entryId ? { ...e, loading: false, answer: res.insight } : e))
      );
    } catch (err) {
      setEntries((prev) =>
        prev.map((e) =>
          e.id === entryId
            ? {
                ...e,
                loading: false,
                error: err instanceof Error ? err.message : t('db_analytics.chartAskAiSlider.genericError'),
              }
            : e
        )
      );
    } finally {
      setIsAsking(false);
    }
  };

  const handleKeyDown = (e: KeyboardEvent<HTMLTextAreaElement>) => {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault();
      submit();
    }
  };

  return (
    <div className={styles['db-analytics-chart-ask-overlay']} role="dialog" aria-modal="true">
      <div className={styles['db-analytics-chart-ask-overlay__backdrop']} onClick={onClose} />
      <aside className={styles['db-analytics-chart-ask-panel']}>
        <div className={styles['db-analytics-chart-ask-panel__header']}>
          <div className={styles['db-analytics-chart-ask-panel__heading']}>
            <span className={styles['db-analytics-chart-ask-panel__icon']}>
              <Icon.askAi size={16} aria-hidden="true" />
            </span>
            <div>
              <h2 className={styles['db-analytics-chart-ask-panel__title']}>{t('db_analytics.chartAskAiSlider.title')}</h2>
              <span className={styles['db-analytics-chart-ask-panel__subtitle']}>{chart.title}</span>
            </div>
          </div>
          <button
            className={styles['db-analytics-chart-ask-panel__close-btn']}
            aria-label={t('db_analytics.chartAskAiSlider.close')}
            onClick={onClose}
          >
            <Icon.close size={18} aria-hidden="true" />
          </button>
        </div>

        <div className={styles['db-analytics-chart-ask-panel__body']}>
          <div className={styles['db-analytics-chart-ask-panel__chart-col']}>
            <div className={styles['db-analytics-chart-ask-panel__chart-card']}>
              <div className={styles['db-analytics-chart-ask-panel__chart-card-header']}>
                <span className={styles['db-analytics-chart-ask-panel__chart-card-icon']}>
                  <ChartIcon size={13} aria-hidden="true" />
                </span>
                <h3>{chart.title}</h3>
              </div>
              {renderChart(chart, 280, displayType)}
            </div>
          </div>

          <div className={styles['db-analytics-chart-ask-panel__chat-col']}>
            <div className={styles['db-analytics-chart-ask-panel__thread']}>
              {entries.length === 0 ? (
                <div className={styles['db-analytics-chart-ask-panel__empty']}>
                  <Icon.askAi size={26} aria-hidden="true" />
                  <p>{t('db_analytics.chartAskAiSlider.emptyPrompt')}</p>
                </div>
              ) : (
                entries.map((entry) => (
                  <div key={entry.id} className={styles['db-analytics-chart-ask-panel__qa']}>
                    <div className={styles['db-analytics-chart-ask-panel__question']}>
                      <p>{entry.question}</p>
                    </div>
                    <div className={styles['db-analytics-chart-ask-panel__answer']}>
                      <span className={styles['db-analytics-chart-ask-panel__answer-avatar']}>
                        <Icon.sparkles size={13} aria-hidden="true" />
                      </span>
                      <div className={styles['db-analytics-chart-ask-panel__answer-content']}>
                        {entry.loading ? (
                          <span className={styles['db-analytics-chart-ask-panel__thinking']}>
                            <Icon.loader
                              size={13}
                              aria-hidden="true"
                              className={styles['db-analytics-chart-ask-panel__spin']}
                            />
                            {t('db_analytics.chartAskAiSlider.thinking')}
                          </span>
                        ) : entry.error ? (
                          <span className={styles['db-analytics-chart-ask-panel__error']}>{entry.error}</span>
                        ) : (
                          <div className={styles['db-analytics-chart-ask-panel__markdown']}>
                            <ReactMarkdown remarkPlugins={[remarkGfm]}>{entry.answer ?? ''}</ReactMarkdown>
                          </div>
                        )}
                      </div>
                    </div>
                  </div>
                ))
              )}
              <div ref={threadEndRef} />
            </div>

            <div className={styles['db-analytics-chart-ask-panel__composer']}>
              <textarea
                className={styles['db-analytics-chart-ask-panel__input']}
                placeholder={t('db_analytics.chartAskAiSlider.placeholder')}
                value={draft}
                onChange={(e) => setDraft(e.target.value)}
                onKeyDown={handleKeyDown}
                rows={2}
                disabled={isAsking}
              />
              <button
                className={styles['db-analytics-chart-ask-panel__send-btn']}
                onClick={submit}
                aria-label={t('db_analytics.chartAskAiSlider.ask')}
                disabled={isAsking || !draft.trim()}
              >
                {isAsking ? (
                  <Icon.loader size={14} aria-hidden="true" className={styles['db-analytics-chart-ask-panel__spin']} />
                ) : (
                  <Icon.send size={14} aria-hidden="true" />
                )}
                {t('db_analytics.chartAskAiSlider.ask')}
              </button>
            </div>
          </div>
        </div>
      </aside>
    </div>
  );
};

export default ChartAskAiSlider;

















//ChartConstructingAnimation.tsx
import React from 'react';
import { useTranslation } from 'react-i18next';
import styles from './ChartConstructingAnimation.module.scss';

// Four bars plus a line series, drawn on the same plot, on a continuous
// loop: bars grow from the baseline (staggered), then a line draws itself
// across their tops with a point "popping in" at each vertex, everything
// holds briefly, fades/shrinks out, and the cycle restarts. All pieces
// share one CSS custom property (--cycle-duration) with matching
// percentage-based keyframes (see the .module.scss) and
// `animation-iteration-count: infinite`, so every repeat stays in sync.
const BAR_X = [40, 90, 140, 190];
const BAR_HEIGHTS = [70, 110, 55, 90];
const BASELINE_Y = 150;
const LINE_POINTS = [
  { x: 40, y: 60 },
  { x: 90, y: 40 },
  { x: 140, y: 85 },
  { x: 190, y: 30 },
];
const BAR_WIDTH = 22;
const CYCLE_DURATION_SECONDS = 3.6;

const ChartConstructingAnimation: React.FC = () => {
  const { t } = useTranslation();
  const linePath = LINE_POINTS.map((p, i) => `${i === 0 ? 'M' : 'L'} ${p.x} ${p.y}`).join(' ');
  const cycleDurationVar = { ['--cycle-duration' as string]: `${CYCLE_DURATION_SECONDS}s` };

  return (
    <div className={styles['db-analytics-chart-draw']}>
      <svg viewBox="0 0 230 170" className={styles['db-analytics-chart-draw__svg']}>
        <g className={styles['db-analytics-chart-draw__erase-group']} style={cycleDurationVar}>
          <line
            x1={20}
            y1={BASELINE_Y}
            x2={210}
            y2={BASELINE_Y}
            className={styles['db-analytics-chart-draw__axis']}
          />

          {BAR_X.map((x, i) => {
            const height = BAR_HEIGHTS[i];
            return (
              <rect
                key={i}
                x={x - BAR_WIDTH / 2}
                y={BASELINE_Y - height}
                width={BAR_WIDTH}
                height={height}
                rx={3}
                className={styles['db-analytics-chart-draw__bar']}
                style={{ transformOrigin: `${x}px ${BASELINE_Y}px`, ...cycleDurationVar }}
              />
            );
          })}

          <path d={linePath} className={styles['db-analytics-chart-draw__line']} style={cycleDurationVar} />
          {LINE_POINTS.map((p, i) => (
            <circle
              key={i}
              cx={p.x}
              cy={p.y}
              r={3.5}
              className={styles['db-analytics-chart-draw__dot']}
              style={cycleDurationVar}
            />
          ))}
        </g>
      </svg>
      <p className={styles['db-analytics-chart-draw__caption']}>{t('db_analytics.resultsPanel.drawingChart')}</p>
    </div>
  );
};

export default ChartConstructingAnimation;












//ChartTypeMenu.tsx
import React, { useEffect, useRef, useState } from 'react';
import { useTranslation } from 'react-i18next';
import styles from './ChartTypeMenu.module.scss';
import { ChartType } from '../types';
import { Icon, IconComponent } from '../icons';
import { getChartIcon, SWITCHABLE_CHART_TYPES } from './ResultsPanel';

interface Props {
  value: ChartType;
  onChange: (type: ChartType) => void;
}

const ChartTypeMenu: React.FC<Props> = ({ value, onChange }) => {
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

  const getTypeLabel = (type: ChartType) => t(`db_analytics.chartTypeMenu.types.${type}`, { defaultValue: type });

  const CurrentIcon: IconComponent = getChartIcon(value);

  return (
    <div className={styles['db-analytics-chart-type-menu']} ref={rootRef}>
      <button
        type="button"
        className={styles['db-analytics-chart-type-menu__trigger']}
        onClick={() => setOpen((o) => !o)}
        aria-haspopup="listbox"
        aria-expanded={open}
        title={t('db_analytics.chartTypeMenu.changeType')}
      >
        <CurrentIcon size={12} aria-hidden="true" />
        <span>{getTypeLabel(value)}</span>
        <Icon.arrowRight
          size={11}
          aria-hidden="true"
          className={styles['db-analytics-chart-type-menu__chevron']}
        />
      </button>

      {open && (
        <ul className={styles['db-analytics-chart-type-menu__list']} role="listbox">
          {SWITCHABLE_CHART_TYPES.map((option) => {
            const OptionIcon = getChartIcon(option.value);
            const isSelected = option.value === value;
            return (
              <li key={option.value}>
                <button
                  type="button"
                  className={`${styles['db-analytics-chart-type-menu__option']} ${
                    isSelected ? styles['db-analytics-chart-type-menu__option--selected'] : ''
                  }`}
                  role="option"
                  aria-selected={isSelected}
                  onClick={() => {
                    onChange(option.value);
                    setOpen(false);
                  }}
                >
                  <OptionIcon size={13} aria-hidden="true" />
                  <span>{getTypeLabel(option.value)}</span>
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

export default ChartTypeMenu;

















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
  );
};

export default ChatPanel;
















//ConnectPasswordModal.tsx
import React, { useState } from 'react';
import { useTranslation, Trans } from 'react-i18next';
import styles from './ConnectPasswordModal.module.scss';
import { DatabaseItem } from '../types';
import { api } from '../../../services/api';
import { Icon } from '../icons';

interface Props {
  database: DatabaseItem;
  onClose: () => void;
  onConnected: (cacheUntil: string) => void;
}

const ConnectPasswordModal: React.FC<Props> = ({ database, onClose, onConnected }) => {
  const { t } = useTranslation();
  const [password, setPassword] = useState('');
  const [connecting, setConnecting] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleConnect = async () => {
    if (!password) {
      setError(t('db_analytics.connectPasswordModal.errorRequired'));
      return;
    }
    setConnecting(true);
    setError(null);
    try {
      const res = await api.connectDatabase(database.id, password);
      onConnected(res.cache_until);
    } catch (err) {
      setError(err instanceof Error ? err.message : t('db_analytics.connectPasswordModal.genericError'));
    } finally {
      setConnecting(false);
    }
  };

  return (
    <div className={styles['db-analytics-modal-overlay']} role="dialog" aria-modal="true">
      <div className={styles['db-analytics-modal-overlay__backdrop']} onClick={onClose} />
      <div className={styles['db-analytics-modal-card']}>
        <div className={styles['db-analytics-modal-card__header']}>
          <h3 className={styles['db-analytics-modal-card__title']}>
            {t('db_analytics.connectPasswordModal.title', { name: database.name })}
          </h3>
          <button
            className={styles['db-analytics-modal-card__close-btn']}
            aria-label={t('db_analytics.connectPasswordModal.close')}
            onClick={onClose}
          >
            <Icon.close size={16} aria-hidden="true" />
          </button>
        </div>
        <div className={styles['db-analytics-modal-card__body']}>
          <p className={styles['db-analytics-modal-card__desc']}>
            <Trans
              i18nKey="connectPasswordModal.description"
              values={{ username: database.username, dbName: database.db_name }}
              components={{ strong: <strong /> }}
            />
          </p>
          {error && <div className={styles['db-analytics-modal-card__error']}>{error}</div>}
          <label className={styles['db-analytics-modal-card__field']}>
            <span className={styles['db-analytics-modal-card__label']}>
              {t('db_analytics.connectPasswordModal.passwordLabel')}
            </span>
            <input
              className={styles['db-analytics-modal-card__input']}
              type="password"
              value={password}
              autoFocus
              onChange={(e) => setPassword(e.target.value)}
              onKeyDown={(e) => e.key === 'Enter' && handleConnect()}
              placeholder={t('db_analytics.connectPasswordModal.passwordPlaceholder')}
            />
          </label>
        </div>
        <div className={styles['db-analytics-modal-card__actions']}>
          <button
            className={`${styles['db-analytics-modal-btn']} ${styles['db-analytics-modal-btn--ghost']}`}
            onClick={onClose}
            disabled={connecting}
          >
            {t('db_analytics.connectPasswordModal.cancel')}
          </button>
          <button
            className={`${styles['db-analytics-modal-btn']} ${styles['db-analytics-modal-btn--primary']}`}
            onClick={handleConnect}
            disabled={connecting}
          >
            {connecting ? t('db_analytics.connectPasswordModal.connecting') : t('db_analytics.connectPasswordModal.connect')}
          </button>
        </div>
      </div>
    </div>
  );
};

export default ConnectPasswordModal;
















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
}

const DatabaseList: React.FC<Props> = ({ databases, onManage, disabled, loading }) => {
  const { t } = useTranslation();
  const isDisabled = !!disabled || !!loading;
  const disabledTitle = loading
    ? t('db_analytics.databaseList.loadingDatabases')
    : disabled
    ? t('db_analytics.databaseList.waitForResponse')
    : undefined;
  const [schemaTarget, setSchemaTarget] = useState<DatabaseItem | null>(null);

  return (
    <div className={styles['db-analytics-db-panel']}>
      <div className={styles['db-analytics-db-panel__header']}>
        <h2 className={styles['db-analytics-db-panel__title']}>{t('db_analytics.databaseList.title')}</h2>
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
                      value={form.port}
                      onChange={(e) => updateField('port', Number(e.target.value))}
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











//DbSchemaSlider.tsx
import React, { useEffect, useMemo, useState } from 'react';
import { useTranslation } from 'react-i18next';
import styles from './DbSchemaSlider.module.scss';
import { DatabaseItem, DbSchema } from '../types';
import { Icon } from '../icons';
import { api } from '../../../services/api';
import SchemaErDiagram from './SchemaErDiagram';

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
      <aside className={styles['db-analytics-schema-panel']}>
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
















//ExpandChartSlider.tsx
import React from 'react';
import { useTranslation } from 'react-i18next';
import styles from './ExpandChartSlider.module.scss';
import { ChartResult, ChartType } from '../types';
import { Icon, IconComponent } from '../icons';
import { renderChart, getChartIcon } from './ResultsPanel';

interface Props {
  chart: ChartResult | null;
  displayType: ChartType;
  onClose: () => void;
}

const ExpandChartSlider: React.FC<Props> = ({ chart, displayType, onClose }) => {
  const { t } = useTranslation();
  if (!chart) return null;

  const ChartIcon: IconComponent = getChartIcon(displayType);

  return (
    <div className={styles['db-analytics-expand-overlay']} role="dialog" aria-modal="true">
      <div className={styles['db-analytics-expand-overlay__backdrop']} onClick={onClose} />
      <aside className={styles['db-analytics-expand-panel']}>
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


















//FollowupChips.tsx
import React from 'react';
import { useTranslation } from 'react-i18next';
import styles from './FollowupChips.module.scss';
import { Icon } from '../icons';

interface Props {
  followups: string[];
  loading?: boolean;
  onSelect: (text: string) => void;
}

const FollowupChips: React.FC<Props> = ({ followups = [], loading, onSelect }) => {
  const { t } = useTranslation();
  if (!loading && followups.length === 0) return null;

  return (
    <div className={styles['db-analytics-followups']}>
      <span className={styles['db-analytics-followups__label']}>
        <Icon.sparkles size={12} aria-hidden="true" />
        {loading ? t('db_analytics.followupChips.finding') : t('db_analytics.followupChips.continueExploring')}
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


















//ResultsPanel.tsx
import React, { useState } from 'react';
import { useTranslation } from 'react-i18next';
import i18n from '../../../i18n';
import {
  LineChart,
  Line,
  BarChart,
  Bar,
  AreaChart,
  Area,
  ScatterChart,
  Scatter,
  PieChart,
  Pie,
  Cell,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  ResponsiveContainer,
} from 'recharts';
import styles from './ResultsPanel.module.scss';
import { ChartResult, ChartType, ChartGroup } from '../types';
import { Icon, IconComponent } from '../icons';
import ChartTypeMenu from './ChartTypeMenu';
import ExpandChartSlider from './ExpandChartSlider';
import ChartAskAiSlider from './ChartAskAiSlider';
import ChartConstructingAnimation from './ChartConstructingAnimation';

interface Props {
  chartGroups: ChartGroup[];
  loadError?: string | null;
  isGenerating?: boolean;
  sessionId: string | null;
}

const chartTypeIcon: Partial<Record<ChartType, IconComponent>> = {
  kpi: Icon.gauge,
  line: Icon.chartLine,
  bar: Icon.chartBar,
  pie: Icon.chartPie,
  area: Icon.chartArea,
  scatter: Icon.chartDots,
  table: Icon.table,
};

const FALLBACK_ICON: IconComponent = Icon.chartAreaLine;

export const getChartIcon = (type: ChartResult['chart_type']): IconComponent =>
  chartTypeIcon[type] ?? FALLBACK_ICON;

// Graph types a person can switch a card to from its header dropdown.
// KPI and table are left out — they're structurally different (not a
// category/series plot) and switching into them from a plotted chart
// wouldn't produce anything meaningful. Labels are resolved through i18n at
// render time in ChartTypeMenu (see chartTypeMenu.types.* keys) — this array
// only carries the stable `value`s plus an English fallback label.
export const SWITCHABLE_CHART_TYPES: { value: ChartType; label: string }[] = [
  { value: 'bar', label: 'Bar' },
  { value: 'line', label: 'Line' },
  { value: 'area', label: 'Area' },
  { value: 'pie', label: 'Pie' },
  { value: 'scatter', label: 'Scatter' },
];

// A richer, more vivid categorical palette than the previous muted set —
// each series/slice gets a distinct, saturated hue that still reads well
// against the light surface.
const SERIES_COLORS = [
  '#4f46e5', // indigo
  '#0aa36f', // green
  '#e08a00', // amber
  '#dc4646', // red
  '#0c8fce', // sky blue
  '#a238c9', // violet
  '#d1385c', // rose
];

const tooltipStyle = {
  background: '#ffffff',
  border: '1px solid rgba(11,11,11,0.08)',
  borderRadius: 10,
  fontSize: 12,
  boxShadow: '0 6px 20px rgba(11,11,11,0.10)',
  padding: '8px 12px',
};

const formatKpiValue = (value: number) => {
  if (Math.abs(value) >= 1_000_000) return `${(value / 1_000_000).toFixed(1)}M`;
  if (Math.abs(value) >= 1000) return value.toLocaleString();
  if (!Number.isInteger(value)) return value.toFixed(1);
  return String(value);
};

const renderKpi = (chart: ChartResult) => (
  <div className={styles['db-analytics-kpi-grid']}>
    {chart.data.map((row, i) => {
      const label = String(row[chart.x_axis ?? 'metric']);
      const value = Number(row[chart.y_axis ?? 'value']);
      return (
        <div key={i} className={styles['db-analytics-kpi-card']}>
          <span className={styles['db-analytics-kpi-card__label']}>{label}</span>
          <span className={styles['db-analytics-kpi-card__value']}>{formatKpiValue(value)}</span>
        </div>
      );
    })}
  </div>
);

// When the API doesn't provide an explicit `series` list (it can be null),
// fall back to every numeric column in the data besides the x-axis key.
const deriveSeriesKeys = (chart: ChartResult, xKey: string): string[] => {
  if (chart.series && chart.series.length > 0) return chart.series;
  if (!chart.data || chart.data.length === 0) return [];

  const sample = chart.data[0];
  return Object.keys(sample).filter((key) => key !== xKey && typeof sample[key] === 'number');
};

const axisTick = { fontSize: 11.5, fill: '#8b8a84', fontWeight: 500 };
const axisLine = { stroke: '#d8d7cf' };

// Resolves which fields to use for a pie slice's label and value. Prefers
// the API-given x_axis/y_axis, but only if that key actually exists on the
// data — a null/missing/incorrect axis field (common on pie responses,
// since they don't always set x_axis/y_axis the way category charts do)
// used to silently default to 'name'/'value', which is wrong whenever the
// real fields are named something else (e.g. 'region', 'total'). Falls back
// to the first string-valued key for the name and the first numeric-valued
// key for the value.
const resolvePieKeys = (chart: ChartResult): { nameKey: string; valueKey: string } => {
  const sample = chart.data[0] ?? {};
  const keys = Object.keys(sample);

  const nameKey =
    chart.x_axis && keys.includes(chart.x_axis)
      ? chart.x_axis
      : keys.find((k) => typeof sample[k] === 'string') ?? keys[0] ?? 'name';

  const valueKey =
    chart.y_axis && keys.includes(chart.y_axis)
      ? chart.y_axis
      : keys.find((k) => k !== nameKey && typeof sample[k] === 'number') ?? 'value';

  return { nameKey, valueKey };
};

// renderChart is a plain function (not a component), called both from this
// file's JSX map and from ExpandChartSlider/ChartAskAiSlider, so it can't
// use the useTranslation() hook. It reads directly off the shared i18next
// instance instead, which works outside of React's render/hook lifecycle
// and still re-resolves to the current language on every call.
export const renderChart = (chart: ChartResult, height = 220, overrideType?: ChartType) => {
  const xKey = chart.x_axis ?? 'name';
  const seriesKeys = deriveSeriesKeys(chart, xKey);
  const gradientPrefix = `db-analytics-grad-${chart.id}`;
  const effectiveType = overrideType ?? chart.chart_type;

  if (!chart.data || chart.data.length === 0) {
    return (
      <div className={styles['db-analytics-chart-card__no-data']}>{i18n.t('db_analytics.resultsPanel.noDataForChart')}</div>
    );
  }

  if (effectiveType === 'kpi') {
    return renderKpi(chart);
  }

  if (effectiveType === 'line') {
    if (seriesKeys.length === 0) {
      return (
        <div className={styles['db-analytics-chart-card__no-data']}>{i18n.t('db_analytics.resultsPanel.noSeriesFound')}</div>
      );
    }
    return (
      <ResponsiveContainer width="100%" height={height}>
        <LineChart data={chart.data} margin={{ top: 8, right: 12, left: -16, bottom: 0 }}>
          <CartesianGrid stroke="#ece9df" vertical={false} />
          <XAxis dataKey={xKey} tick={axisTick} axisLine={axisLine} tickLine={false} />
          <YAxis tick={axisTick} axisLine={false} tickLine={false} />
          <Tooltip contentStyle={tooltipStyle} cursor={{ stroke: '#d8d7cf', strokeWidth: 1 }} />
          {seriesKeys.map((key, i) => (
            <Line
              key={key}
              type="monotone"
              dataKey={key}
              stroke={SERIES_COLORS[i % SERIES_COLORS.length]}
              strokeWidth={1.5}
              dot={{ r: 2.5, strokeWidth: 1.5, stroke: '#fff', fill: SERIES_COLORS[i % SERIES_COLORS.length] }}
              activeDot={{ r: 5, strokeWidth: 1.5, stroke: '#fff' }}
            />
          ))}
        </LineChart>
      </ResponsiveContainer>
    );
  }

  if (effectiveType === 'area') {
    if (seriesKeys.length === 0) {
      return (
        <div className={styles['db-analytics-chart-card__no-data']}>{i18n.t('db_analytics.resultsPanel.noSeriesFound')}</div>
      );
    }
    return (
      <ResponsiveContainer width="100%" height={height}>
        <AreaChart data={chart.data} margin={{ top: 8, right: 12, left: -16, bottom: 0 }}>
          <defs>
            {seriesKeys.map((key, i) => (
              <linearGradient key={key} id={`${gradientPrefix}-${i}`} x1="0" y1="0" x2="0" y2="1">
                <stop offset="0%" stopColor={SERIES_COLORS[i % SERIES_COLORS.length]} stopOpacity={0.35} />
                <stop offset="100%" stopColor={SERIES_COLORS[i % SERIES_COLORS.length]} stopOpacity={0.02} />
              </linearGradient>
            ))}
          </defs>
          <CartesianGrid stroke="#ece9df" vertical={false} />
          <XAxis dataKey={xKey} tick={axisTick} axisLine={axisLine} tickLine={false} />
          <YAxis tick={axisTick} axisLine={false} tickLine={false} />
          <Tooltip contentStyle={tooltipStyle} cursor={{ stroke: '#d8d7cf', strokeWidth: 1 }} />
          {seriesKeys.map((key, i) => (
            <Area
              key={key}
              type="monotone"
              dataKey={key}
              stackId={chart.stacked ? 'stack' : undefined}
              stroke={SERIES_COLORS[i % SERIES_COLORS.length]}
              fill={`url(#${gradientPrefix}-${i})`}
              strokeWidth={1.5}
            />
          ))}
        </AreaChart>
      </ResponsiveContainer>
    );
  }

  if (effectiveType === 'bar') {
    if (seriesKeys.length === 0) {
      return (
        <div className={styles['db-analytics-chart-card__no-data']}>{i18n.t('db_analytics.resultsPanel.noSeriesFound')}</div>
      );
    }
    return (
      <ResponsiveContainer width="100%" height={height}>
        <BarChart data={chart.data} margin={{ top: 8, right: 12, left: -16, bottom: 0 }} barGap={6}>
          <defs>
            {seriesKeys.map((key, i) => (
              <linearGradient key={key} id={`${gradientPrefix}-bar-${i}`} x1="0" y1="0" x2="0" y2="1">
                <stop offset="0%" stopColor={SERIES_COLORS[i % SERIES_COLORS.length]} stopOpacity={1} />
                <stop offset="100%" stopColor={SERIES_COLORS[i % SERIES_COLORS.length]} stopOpacity={0.7} />
              </linearGradient>
            ))}
          </defs>
          <CartesianGrid stroke="#ece9df" vertical={false} />
          <XAxis dataKey={xKey} tick={axisTick} axisLine={axisLine} tickLine={false} />
          <YAxis tick={axisTick} axisLine={false} tickLine={false} />
          <Tooltip contentStyle={tooltipStyle} cursor={{ fill: 'rgba(79, 70, 229, 0.06)' }} />
          {seriesKeys.map((key, i) => (
            <Bar
              key={key}
              dataKey={key}
              stackId={chart.stacked ? 'stack' : undefined}
              fill={`url(#${gradientPrefix}-bar-${i})`}
              radius={[6, 6, 0, 0]}
              maxBarSize={32}
            />
          ))}
        </BarChart>
      </ResponsiveContainer>
    );
  }

  if (effectiveType === 'scatter') {
    return (
      <ResponsiveContainer width="100%" height={height}>
        <ScatterChart margin={{ top: 8, right: 12, left: -16, bottom: 0 }}>
          <CartesianGrid stroke="#ece9df" />
          <XAxis dataKey={xKey} tick={axisTick} axisLine={axisLine} tickLine={false} />
          <YAxis dataKey={chart.y_axis ?? undefined} tick={axisTick} axisLine={false} tickLine={false} />
          <Tooltip contentStyle={tooltipStyle} cursor={{ strokeDasharray: '3 3' }} />
          <Scatter data={chart.data} fill={SERIES_COLORS[0]} fillOpacity={0.8} />
        </ScatterChart>
      </ResponsiveContainer>
    );
  }

  if (effectiveType === 'table') {
    const columns = chart.data.length > 0 ? Object.keys(chart.data[0]) : [];
    return (
      <div className={styles['db-analytics-table-wrap']}>
        <table className={styles['db-analytics-table']}>
          <thead>
            <tr>
              {columns.map((col) => (
                <th key={col}>{col}</th>
              ))}
            </tr>
          </thead>
          <tbody>
            {chart.data.map((row, i) => (
              <tr key={i}>
                {columns.map((col) => (
                  <td key={col}>{String(row[col])}</td>
                ))}
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    );
  }

  // pie (default/fallback)
  const { nameKey, valueKey } = resolvePieKeys(chart);
  const total = chart.data.reduce((sum, d) => sum + Number(d[valueKey] ?? 0), 0);

  return (
    <>
      <ResponsiveContainer width="100%" height={height - 20}>
        <PieChart>
          <Pie
            data={chart.data}
            dataKey={valueKey}
            nameKey={nameKey}
            innerRadius={50}
            outerRadius={82}
            paddingAngle={2}
            strokeWidth={1.5}
            stroke="#fcfcfb"
            label={({ percent }) => (percent > 0.06 ? `${Math.round(percent * 100)}%` : '')}
            labelLine={false}
          >
            {chart.data.map((_, i) => (
              <Cell key={i} fill={SERIES_COLORS[i % SERIES_COLORS.length]} />
            ))}
          </Pie>
          <Tooltip contentStyle={tooltipStyle} />
        </PieChart>
      </ResponsiveContainer>
      <div className={styles['db-analytics-chart-legend']}>
        {chart.data.map((d, i) => (
          <React.Fragment key={i}>
            {i > 0 && <span className={styles['db-analytics-chart-legend__divider']} aria-hidden="true" />}
            <span className={styles['db-analytics-chart-legend__item']}>
              <span
                className={styles['db-analytics-chart-legend__swatch']}
                style={{ background: SERIES_COLORS[i % SERIES_COLORS.length] }}
              />
              <span className={styles['db-analytics-chart-legend__name']}>{String(d[nameKey])}</span>
              <span className={styles['db-analytics-chart-legend__pct']}>
                {total > 0 ? Math.round((Number(d[valueKey]) / total) * 100) : 0}%
              </span>
            </span>
          </React.Fragment>
        ))}
      </div>
    </>
  );
};

const ResultsPanel: React.FC<Props> = ({ chartGroups, loadError, isGenerating, sessionId }) => {
  const { t } = useTranslation();
  const showEmptyState = chartGroups.length === 0;
  const [typeOverrides, setTypeOverrides] = useState<Record<string, ChartType>>({});
  const [expandedChartKey, setExpandedChartKey] = useState<string | null>(null);
  const [askAiChartKey, setAskAiChartKey] = useState<string | null>(null);

  // Chart ids aren't guaranteed unique across the whole session — the
  // backend can reuse the same id (e.g. a generic "revenue_kpis") for
  // charts belonging to different prompts/groups. Using chart.id alone as
  // the key for type overrides / expand / ask-AI state meant switching one
  // card's type silently changed every other card that happened to share
  // that id. This composite key scopes state to one specific card instance.
  const getChartKey = (groupId: string, chart: ChartResult, index: number) => `${groupId}::${chart.id}::${index}`;

  const getDisplayType = (key: string, chart: ChartResult): ChartType => typeOverrides[key] ?? chart.chart_type;

  const chartsByKey = new Map<string, ChartResult>();
  chartGroups.forEach((group) => {
    group.graphs.forEach((chart, index) => {
      chartsByKey.set(getChartKey(group.id, chart, index), chart);
    });
  });

  const expandedChart = expandedChartKey ? chartsByKey.get(expandedChartKey) ?? null : null;
  const expandedDisplayType = expandedChart && expandedChartKey ? getDisplayType(expandedChartKey, expandedChart) : 'bar';
  const askAiChart = askAiChartKey ? chartsByKey.get(askAiChartKey) ?? null : null;
  const askAiDisplayType = askAiChart && askAiChartKey ? getDisplayType(askAiChartKey, askAiChart) : 'bar';

  return (
    <div className={styles['db-analytics-results-panel']}>
      <div className={styles['db-analytics-results-panel__header']}>
        <div className={styles['db-analytics-results-panel__header-text']}>
          <h2 className={styles['db-analytics-results-panel__title']}>{t('db_analytics.resultsPanel.title')}</h2>
          <p className={styles['db-analytics-results-panel__subtitle']}>{t('db_analytics.resultsPanel.subtitle')}</p>
        </div>
      </div>
      <div className={styles['db-analytics-results-panel__body']}>
        {loadError && <div className={styles['db-analytics-results-panel__error']}>{loadError}</div>}

        {isGenerating ? (
          <ChartConstructingAnimation />
        ) : showEmptyState ? (
          <div className={styles['db-analytics-empty-state']}>
            <Icon.chartAreaLine size={28} aria-hidden="true" />
            <p>{t('db_analytics.resultsPanel.emptyState')}</p>
          </div>
        ) : (
          <div className={styles['db-analytics-chart-groups']}>
            {chartGroups.map((group, groupIndex) => (
              <React.Fragment key={group.id}>
                {groupIndex > 0 && <div className={styles['db-analytics-chart-groups__divider']} aria-hidden="true" />}
                <section className={styles['db-analytics-chart-group']}>
                  {group.prompt && (
                    <div className={styles['db-analytics-chart-group__prompt']}>
                      <span className={styles['db-analytics-chart-group__prompt-label']}>
                        {t('db_analytics.resultsPanel.prompt')}
                      </span>
                      <p className={styles['db-analytics-chart-group__prompt-text']}>{group.prompt}</p>
                    </div>
                  )}
                  <div className={styles['db-analytics-chart-grid']}>
                    {group.graphs.map((chart, index) => {
                      const chartKey = getChartKey(group.id, chart, index);
                      const displayType = getDisplayType(chartKey, chart);
                      const canSwitchType = chart.chart_type !== 'kpi' && chart.chart_type !== 'table';
                      return (
                        <div
                          key={chartKey}
                          className={`${styles['db-analytics-chart-card']} ${
                            chart.chart_type === 'kpi' ? styles['db-analytics-chart-card--span-2'] : ''
                          }`}
                        >
                          <div className={styles['db-analytics-chart-card__header']}>
                            <div className={styles['db-analytics-chart-card__heading']}>
                              <span className={styles['db-analytics-chart-card__icon']}>
                                {(() => {
                                  const ChartIcon = getChartIcon(displayType);
                                  return <ChartIcon size={14} aria-hidden="true" />;
                                })()}
                              </span>
                              <h3 className={styles['db-analytics-chart-card__title']}>{chart.title}</h3>
                            </div>
                            <div className={styles['db-analytics-chart-card__actions']}>
                              {canSwitchType && (
                                <ChartTypeMenu
                                  value={displayType}
                                  onChange={(type) =>
                                    setTypeOverrides((prev) => ({ ...prev, [chartKey]: type }))
                                  }
                                />
                              )}
                              {chart.chart_type !== 'kpi' && (
                                <>
                                  <button
                                    type="button"
                                    className={styles['db-analytics-chart-card__expand-btn']}
                                    onClick={() => setAskAiChartKey(chartKey)}
                                    aria-label={t('db_analytics.resultsPanel.askAiAboutChart')}
                                    title={t('db_analytics.resultsPanel.askAi')}
                                  >
                                    <Icon.askAi size={14} aria-hidden="true" />
                                  </button>
                                  <button
                                    type="button"
                                    className={styles['db-analytics-chart-card__expand-btn']}
                                    onClick={() => setExpandedChartKey(chartKey)}
                                    aria-label={t('db_analytics.resultsPanel.expandChart')}
                                    title={t('db_analytics.resultsPanel.expand')}
                                  >
                                    <Icon.expand size={14} aria-hidden="true" />
                                  </button>
                                </>
                              )}
                            </div>
                          </div>
                          {renderChart(chart, 220, displayType)}
                        </div>
                      );
                    })}
                  </div>
                </section>
              </React.Fragment>
            ))}
          </div>
        )}
      </div>

      <ExpandChartSlider
        chart={expandedChart}
        displayType={expandedDisplayType}
        onClose={() => setExpandedChartKey(null)}
      />

      <ChartAskAiSlider
        sessionId={sessionId}
        chart={askAiChart}
        displayType={askAiDisplayType}
        onClose={() => setAskAiChartKey(null)}
      />
    </div>
  );
};

export default ResultsPanel;















//SchemaErDiagram.tsx
import React, { useMemo } from 'react';
import { useTranslation } from 'react-i18next';
import styles from './SchemaErDiagram.module.scss';
import { DbTable } from '../types';
import { Icon } from '../icons';

interface Props {
  tables: DbTable[];
}

interface Relationship {
  fromTable: string;
  fromColumn: string;
  toTable: string;
  toColumn: string;
}

// Infers foreign-key-style relationships purely from naming conventions,
// since the schema endpoint doesn't return explicit FK metadata. A column
// like `customer_id` on `orders` is treated as pointing at the `id` (or
// `customer_id`) primary key column on a table named `customer`/`customers`.
function inferRelationships(tables: DbTable[]): Relationship[] {
  const tableByName = new Map(tables.map((t) => [t.name.toLowerCase(), t]));
  const relationships: Relationship[] = [];

  for (const table of tables) {
    for (const column of table.columns) {
      if (column.primary_key) continue;
      const match = column.name.match(/^(.+?)_id$/i);
      if (!match) continue;

      const base = match[1].toLowerCase();
      const candidates = [base, `${base}s`, base.replace(/s$/, '')];
      const targetName = candidates.find((c) => tableByName.has(c) && c !== table.name.toLowerCase());
      if (!targetName) continue;

      const targetTable = tableByName.get(targetName);
      if (!targetTable) continue;

      const targetPk = targetTable.columns.find((c) => c.primary_key) ?? targetTable.columns[0];
      if (!targetPk) continue;

      relationships.push({
        fromTable: table.name,
        fromColumn: column.name,
        toTable: targetTable.name,
        toColumn: targetPk.name,
      });
    }
  }

  return relationships;
}

const CARD_WIDTH = 220;
const CARD_HEADER_HEIGHT = 34;
const ROW_HEIGHT = 22;
const COL_GAP_X = 80;
const ROW_GAP_Y = 40;
const MAX_VISIBLE_COLUMNS = 6;

const SchemaErDiagram: React.FC<Props> = ({ tables }) => {
  const { t } = useTranslation();
  const relationships = useMemo(() => inferRelationships(tables), [tables]);

  // Simple grid layout: wrap tables into rows of up to 3, sized to fit each
  // table's own column count so nothing overlaps.
  const layout = useMemo(() => {
    const perRow = tables.length <= 4 ? 2 : 3;
    const positions = new Map<string, { x: number; y: number; height: number }>();
    const rowHeights: number[] = [];
    let currentRowMax = 0;

    tables.forEach((table, i) => {
      const visibleCols = Math.min(table.columns.length, MAX_VISIBLE_COLUMNS);
      const extra = table.columns.length > MAX_VISIBLE_COLUMNS ? ROW_HEIGHT : 0;
      const height = CARD_HEADER_HEIGHT + visibleCols * ROW_HEIGHT + extra + 10;
      currentRowMax = Math.max(currentRowMax, height);

      if ((i + 1) % perRow === 0 || i === tables.length - 1) {
        rowHeights.push(currentRowMax);
        currentRowMax = 0;
      }
    });

    let rowIndex = 0;
    let colIndex = 0;
    let yOffset = 20;

    tables.forEach((table, i) => {
      const visibleCols = Math.min(table.columns.length, MAX_VISIBLE_COLUMNS);
      const extra = table.columns.length > MAX_VISIBLE_COLUMNS ? ROW_HEIGHT : 0;
      const height = CARD_HEADER_HEIGHT + visibleCols * ROW_HEIGHT + extra + 10;

      const x = 24 + colIndex * (CARD_WIDTH + COL_GAP_X);
      const y = yOffset;
      positions.set(table.name, { x, y, height });

      colIndex++;
      if ((i + 1) % perRow === 0 || i === tables.length - 1) {
        yOffset += rowHeights[rowIndex] + ROW_GAP_Y;
        rowIndex++;
        colIndex = 0;
      }
    });

    const totalWidth = 24 * 2 + perRow * CARD_WIDTH + (perRow - 1) * COL_GAP_X;
    const totalHeight = yOffset + 20;

    return { positions, totalWidth, totalHeight };
  }, [tables]);

  const getColumnY = (tableName: string, columnName: string): number | null => {
    const pos = layout.positions.get(tableName);
    const table = tables.find((t) => t.name === tableName);
    if (!pos || !table) return null;
    const idx = table.columns.findIndex((c) => c.name === columnName);
    if (idx === -1 || idx >= MAX_VISIBLE_COLUMNS) return pos.y + CARD_HEADER_HEIGHT / 2;
    return pos.y + CARD_HEADER_HEIGHT + idx * ROW_HEIGHT + ROW_HEIGHT / 2;
  };

  if (tables.length === 0) return null;

  return (
    <div className={styles['db-analytics-er-diagram']}>
      <div className={styles['db-analytics-er-diagram__scroll']}>
        <svg
          width={layout.totalWidth}
          height={layout.totalHeight}
          viewBox={`0 0 ${layout.totalWidth} ${layout.totalHeight}`}
          className={styles['db-analytics-er-diagram__svg']}
        >
          <defs>
            <marker id="er-arrow" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
              <path d="M0,0 L6,3 L0,6 Z" className={styles['db-analytics-er-diagram__arrow']} />
            </marker>
          </defs>

          {relationships.map((rel, i) => {
            const fromPos = layout.positions.get(rel.fromTable);
            const toPos = layout.positions.get(rel.toTable);
            const fromY = getColumnY(rel.fromTable, rel.fromColumn);
            const toY = getColumnY(rel.toTable, rel.toColumn);
            if (!fromPos || !toPos || fromY === null || toY === null) return null;

            const fromOnLeft = fromPos.x < toPos.x;
            const startX = fromOnLeft ? fromPos.x + CARD_WIDTH : fromPos.x;
            const endX = fromOnLeft ? toPos.x : toPos.x + CARD_WIDTH;
            const midX = (startX + endX) / 2;

            const path = `M ${startX} ${fromY} C ${midX} ${fromY}, ${midX} ${toY}, ${endX} ${toY}`;

            return (
              <path
                key={i}
                d={path}
                className={styles['db-analytics-er-diagram__link']}
                markerEnd="url(#er-arrow)"
              />
            );
          })}

          {tables.map((table) => {
            const pos = layout.positions.get(table.name);
            if (!pos) return null;
            const visibleColumns = table.columns.slice(0, MAX_VISIBLE_COLUMNS);
            const hiddenCount = table.columns.length - visibleColumns.length;

            return (
              <g key={table.name} transform={`translate(${pos.x}, ${pos.y})`}>
                <rect
                  width={CARD_WIDTH}
                  height={pos.height}
                  rx={10}
                  className={styles['db-analytics-er-diagram__card']}
                />
                <rect width={CARD_WIDTH} height={CARD_HEADER_HEIGHT} rx={10} className={styles['db-analytics-er-diagram__card-header']} />
                <rect y={CARD_HEADER_HEIGHT - 10} width={CARD_WIDTH} height={10} className={styles['db-analytics-er-diagram__card-header']} />
                <text x={12} y={CARD_HEADER_HEIGHT / 2 + 4} className={styles['db-analytics-er-diagram__card-title']}>
                  {table.name}
                </text>
                <text
                  x={CARD_WIDTH - 12}
                  y={CARD_HEADER_HEIGHT / 2 + 4}
                  textAnchor="end"
                  className={styles['db-analytics-er-diagram__card-count']}
                >
                  {table.row_count.toLocaleString()}
                </text>

                {visibleColumns.map((col, i) => (
                  <g key={col.name} transform={`translate(0, ${CARD_HEADER_HEIGHT + i * ROW_HEIGHT})`}>
                    {i % 2 === 1 && (
                      <rect width={CARD_WIDTH} height={ROW_HEIGHT} className={styles['db-analytics-er-diagram__row-alt']} />
                    )}
                    {col.primary_key && (
                      <text x={12} y={ROW_HEIGHT / 2 + 4} className={styles['db-analytics-er-diagram__pk']}>
                        {t('db_analytics.dbSchemaSlider.table.pkAbbr')}
                      </text>
                    )}
                    <text
                      x={col.primary_key ? 34 : 12}
                      y={ROW_HEIGHT / 2 + 4}
                      className={styles['db-analytics-er-diagram__col-name']}
                    >
                      {col.name}
                    </text>
                    <text
                      x={CARD_WIDTH - 12}
                      y={ROW_HEIGHT / 2 + 4}
                      textAnchor="end"
                      className={styles['db-analytics-er-diagram__col-type']}
                    >
                      {col.type}
                    </text>
                  </g>
                ))}

                {hiddenCount > 0 && (
                  <text
                    x={12}
                    y={CARD_HEADER_HEIGHT + visibleColumns.length * ROW_HEIGHT + ROW_HEIGHT / 2 + 4}
                    className={styles['db-analytics-er-diagram__more']}
                  >
                    {t('db_analytics.schemaErDiagram.moreColumns', { count: hiddenCount })}
                  </text>
                )}
              </g>
            );
          })}
        </svg>
      </div>

      {relationships.length === 0 && (
        <div className={styles['db-analytics-er-diagram__note']}>
          <Icon.infoCircle size={13} aria-hidden="true" />
          {t('db_analytics.schemaErDiagram.noRelationships')}
        </div>
      )}
    </div>
  );
};

export default SchemaErDiagram;
















//SessionHistory.tsx
import React, { useEffect, useRef, useState } from 'react';
import { useTranslation } from 'react-i18next';
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

const SessionHistory: React.FC<Props> = ({
  sessions,
  databases,
  activeSessionId,
  onSelect,
  onNewSession,
  disabled,
  loading,
}) => {
  const { t, i18n } = useTranslation();
  const [openDbListFor, setOpenDbListFor] = useState<string | null>(null);
  const popoverRef = useRef<HTMLDivElement>(null);

  const formatSessionDate = (iso: string) => {
    const date = new Date(iso);
    return date.toLocaleDateString(i18n.language, {
      day: 'numeric',
      month: 'short',
      year: 'numeric',
    });
  };

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
        <h2 className={styles['db-analytics-session-panel__title']}>{t('db_analytics.sessionHistory.title')}</h2>
        <button
          className={styles['db-analytics-session-panel__icon-btn']}
          aria-label={t('db_analytics.sessionHistory.newSession')}
          onClick={onNewSession}
          disabled={disabled || loading}
          title={
            loading
              ? t('db_analytics.sessionHistory.loadingSessions')
              : disabled
              ? t('db_analytics.sessionHistory.waitForResponse')
              : undefined
          }
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
            <p>{t('db_analytics.sessionHistory.emptyTitle')}</p>
            <span>{t('db_analytics.sessionHistory.emptyDesc')}</span>
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
                    title={disabled ? t('db_analytics.sessionHistory.waitForResponse') : undefined}
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
                          title={t('db_analytics.sessionHistory.viewConnectedDatabases')}
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
                        {t('db_analytics.sessionHistory.connectedDatabasesPopoverTitle')}
                      </span>
                      {sessionDbs.length === 0 ? (
                        <span className={styles['db-analytics-session-db-popover__empty']}>
                          {t('db_analytics.sessionHistory.connectedDatabasesEmpty')}
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












//SuggestedPrompts.tsx
import React from 'react';
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
              {t('db_analytics.suggestedPrompts.askThis')}
              <Icon.arrowRight size={13} aria-hidden="true" />
            </span>
          </button>
        ))}
      </div>
    </div>
  );
};

export default SuggestedPrompts;















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
      "askThis": "Ask this"
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
      "askThis": "이 질문하기"
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
