//ChartAskAiSlider.module.scss

@use '../../../styles/tokens' as t;

.db-analytics-chart-ask-overlay {
  position: fixed;
  inset: 0;
  z-index: 150;
  display: flex;
  justify-content: flex-end;
}

.db-analytics-chart-ask-overlay__backdrop {
  position: absolute;
  inset: 0;
  background: rgba(11, 11, 11, 0.32);
  animation: db-analytics-chart-ask-fade-in 0.15s ease;
}

@keyframes db-analytics-chart-ask-fade-in {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

.db-analytics-chart-ask-panel {
  position: relative;
  width: 78vw;
  min-width: 720px;
  max-width: 1180px;
  height: 100%;
  background: t.$surface-2;
  display: flex;
  flex-direction: column;
  box-shadow: -8px 0 24px rgba(11, 11, 11, 0.14);
  animation: db-analytics-chart-ask-slide-in 0.22s cubic-bezier(0.16, 1, 0.3, 1);
}

@keyframes db-analytics-chart-ask-slide-in {
  from {
    transform: translateX(100%);
  }
  to {
    transform: translateX(0);
  }
}

.db-analytics-chart-ask-panel__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 64px;
  padding: 0 22px;
  border-bottom: 1px solid t.$border-strong;
  background: t.$surface-1;
  flex-shrink: 0;
}

.db-analytics-chart-ask-panel__heading {
  display: flex;
  align-items: center;
  gap: 12px;
  min-width: 0;
}

.db-analytics-chart-ask-panel__icon {
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

.db-analytics-chart-ask-panel__title {
  font-size: 15px;
  font-weight: 600;
  color: t.$text-primary;
  margin: 0 0 2px;
}

.db-analytics-chart-ask-panel__subtitle {
  font-size: 11.5px;
  color: t.$text-muted;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 420px;
  display: block;
}

.db-analytics-chart-ask-panel__close-btn {
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

.db-analytics-chart-ask-panel__body {
  flex: 1;
  min-height: 0;
  display: grid;
  grid-template-columns: minmax(0, 1fr) minmax(0, 1fr);
}

.db-analytics-chart-ask-panel__chart-col {
  padding: 22px;
  border-right: 1px solid t.$border-strong;
  overflow-y: auto;
  background: t.$surface-1;
}

.db-analytics-chart-ask-panel__chart-card {
  border: 1px solid t.$border-strong;
  border-radius: t.$radius-lg;
  padding: 16px 18px 18px;
  background: t.$surface-2;
  box-shadow: 0 1px 3px rgba(11, 11, 11, 0.05);
}

.db-analytics-chart-ask-panel__chart-card-header {
  display: flex;
  align-items: center;
  gap: 9px;
  margin-bottom: 14px;
  padding-bottom: 11px;
  border-bottom: 0.5px solid t.$border;

  h3 {
    font-size: 13.5px;
    font-weight: 600;
    color: t.$text-primary;
    margin: 0;
  }
}

.db-analytics-chart-ask-panel__chart-card-icon {
  width: 26px;
  height: 26px;
  border-radius: 7px;
  background: t.$gradient-primary;
  color: #fff;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.db-analytics-chart-ask-panel__chat-col {
  display: flex;
  flex-direction: column;
  min-height: 0;
}

.db-analytics-chart-ask-panel__thread {
  flex: 1;
  min-height: 0;
  overflow-y: auto;
  padding: 20px 22px;
  display: flex;
  flex-direction: column;
  gap: 18px;
}

.db-analytics-chart-ask-panel__empty {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 10px;
  text-align: center;
  color: t.$text-muted;
  padding: 24px;

  p {
    font-size: 13px;
    margin: 0;
    max-width: 260px;
    line-height: 1.6;
  }
}

.db-analytics-chart-ask-panel__qa {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.db-analytics-chart-ask-panel__question {
  align-self: flex-end;
  max-width: 88%;
  background: t.$accent-bg;
  border: 1px solid rgba(79, 70, 229, 0.14);
  border-radius: t.$radius-lg;
  padding: 9px 13px;

  p {
    margin: 0;
    font-size: 13px;
    line-height: 1.55;
    color: t.$text-primary;
  }
}

.db-analytics-chart-ask-panel__answer {
  display: flex;
  gap: 8px;
  max-width: 92%;
}

.db-analytics-chart-ask-panel__answer-avatar {
  width: 24px;
  height: 24px;
  border-radius: 50%;
  background: #eeedfe;
  color: #3c3489;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  margin-top: 2px;
}

.db-analytics-chart-ask-panel__answer-content {
  background: t.$surface-0;
  border-radius: t.$radius-lg;
  padding: 10px 13px;
  flex: 1;
  min-width: 0;
}

.db-analytics-chart-ask-panel__thinking {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 12.5px;
  color: t.$text-muted;
}

.db-analytics-chart-ask-panel__error {
  font-size: 12.5px;
  color: #791f1f;
}

.db-analytics-chart-ask-panel__spin {
  animation: db-analytics-chart-ask-spin 0.8s linear infinite;
}

@keyframes db-analytics-chart-ask-spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

.db-analytics-chart-ask-panel__markdown {
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

  ul,
  ol {
    margin: 0 0 8px;
    padding-left: 20px;
  }

  li {
    margin: 2px 0;
  }

  strong {
    font-weight: 600;
  }

  code {
    font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
    font-size: 12px;
    background: t.$surface-1;
    border: 0.5px solid t.$border;
    border-radius: 4px;
    padding: 1px 5px;
  }
}

.db-analytics-chart-ask-panel__composer {
  border-top: 0.5px solid t.$border;
  padding: 14px 22px;
  flex-shrink: 0;
  display: flex;
  gap: 10px;
  align-items: flex-end;
}

.db-analytics-chart-ask-panel__input {
  flex: 1;
  resize: none;
  border: 0.5px solid t.$border-strong;
  border-radius: t.$radius;
  padding: 10px 12px;
  font-size: 13px;
  font-family: inherit;
  color: t.$text-primary;
  background: t.$surface-1;

  &:focus {
    outline: none;
    border-color: t.$accent;
    box-shadow: 0 0 0 2px rgba(42, 120, 214, 0.15);
  }

  &::placeholder {
    color: t.$text-muted;
  }

  &:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }
}

.db-analytics-chart-ask-panel__send-btn {
  display: flex;
  align-items: center;
  gap: 6px;
  background: t.$gradient-primary;
  color: #fff;
  border: none;
  border-radius: t.$radius;
  padding: 10px 16px;
  font-size: 13px;
  font-weight: 500;
  font-family: inherit;
  cursor: pointer;
  box-shadow: 0 1px 2px rgba(79, 70, 229, 0.25);
  flex-shrink: 0;
  transition: filter 0.15s ease, box-shadow 0.15s ease;

  &:hover:not(:disabled) {
    filter: brightness(1.08);
    box-shadow: 0 2px 6px rgba(79, 70, 229, 0.35);
  }

  &:disabled {
    opacity: 0.55;
    cursor: not-allowed;
  }
}

@media (max-width: 900px) {
  .db-analytics-chart-ask-panel {
    min-width: 100vw;
    max-width: 100vw;
  }

  .db-analytics-chart-ask-panel__body {
    grid-template-columns: minmax(0, 1fr);
    grid-auto-rows: min-content;
    overflow-y: auto;
  }

  .db-analytics-chart-ask-panel__chart-col {
    border-right: none;
    border-bottom: 1px solid t.$border-strong;
  }
}











//ChartAskAiSlider.tsx

import React, { useEffect, useRef, useState, KeyboardEvent } from 'react';
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
            ? { ...e, loading: false, error: err instanceof Error ? err.message : 'Something went wrong.' }
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
              <h2 className={styles['db-analytics-chart-ask-panel__title']}>Ask about this chart</h2>
              <span className={styles['db-analytics-chart-ask-panel__subtitle']}>{chart.title}</span>
            </div>
          </div>
          <button className={styles['db-analytics-chart-ask-panel__close-btn']} aria-label="Close" onClick={onClose}>
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
                  <p>Ask a question about this specific chart — its trend, an outlier, what's driving it.</p>
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
                            Thinking…
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
                placeholder="Ask something about this chart…"
                value={draft}
                onChange={(e) => setDraft(e.target.value)}
                onKeyDown={handleKeyDown}
                rows={2}
                disabled={isAsking}
              />
              <button
                className={styles['db-analytics-chart-ask-panel__send-btn']}
                onClick={submit}
                aria-label="Ask"
                disabled={isAsking || !draft.trim()}
              >
                {isAsking ? (
                  <Icon.loader size={14} aria-hidden="true" className={styles['db-analytics-chart-ask-panel__spin']} />
                ) : (
                  <Icon.send size={14} aria-hidden="true" />
                )}
                Ask
              </button>
            </div>
          </div>
        </div>
      </aside>
    </div>
  );
};

export default ChartAskAiSlider;












//ResultsPanel.module.scss
@use '../../../styles/tokens' as t;

.db-analytics-results-panel {
  background: t.$surface-2;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  flex: 1;
  min-height: 0;
}

.db-analytics-results-panel__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 53px;
  padding: 0 16px;
  border-bottom: 1px solid t.$border-strong;
  background: t.$surface-1;
  flex-shrink: 0;
}

.db-analytics-results-panel__header-text {
  display: flex;
  flex-direction: column;
  justify-content: center;
  gap: 1px;
  min-width: 0;
}

.db-analytics-results-panel__title {
  font-size: 14px;
  font-weight: 500;
  margin: 0;
  color: t.$text-primary;
  line-height: 1.3;
}

.db-analytics-results-panel__subtitle {
  font-size: 11px;
  color: t.$text-muted;
  margin: 0;
  line-height: 1.3;
}

.db-analytics-results-panel__body {
  flex: 1;
  min-height: 0;
  overflow-y: auto;

  // Force a classic, always-visible scrollbar rather than relying on the
  // OS/browser's overlay scrollbar, which only paints on hover/scroll and
  // fades out immediately after on many systems.
  scrollbar-width: thin;
  scrollbar-color: t.$border-stronger transparent;

  &::-webkit-scrollbar {
    width: 8px;
    -webkit-appearance: none;
  }

  &::-webkit-scrollbar-track {
    background: t.$surface-0;
  }

  &::-webkit-scrollbar-thumb {
    background: t.$border-stronger;
    border-radius: 999px;
    border: 2px solid t.$surface-0;
  }

  &::-webkit-scrollbar-thumb:hover {
    background: t.$text-muted;
  }
}

.db-analytics-results-panel__error {
  margin: 12px;
  font-size: 12px;
  color: #791f1f;
  background: #fbe9e9;
  padding: 8px 11px;
  border-radius: t.$radius;
}

.db-analytics-generating-banner {
  display: flex;
  align-items: center;
  gap: 8px;
  margin: 12px 12px 0;
  padding: 10px 14px;
  border-radius: t.$radius-lg;
  background: t.$accent-bg;
  border: 1px solid rgba(79, 70, 229, 0.18);
  color: #4530a8;
  font-size: 12.5px;
  font-weight: 500;
}

.db-analytics-chart-groups {
  display: flex;
  flex-direction: column;
}

.db-analytics-chart-groups__divider {
  height: 1px;
  margin: 4px 20px 8px;
  background: t.$border-strong;
}

.db-analytics-chart-group__prompt {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  margin: 4px 12px 8px;
}

.db-analytics-chart-group__prompt-icon {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: t.$surface-0;
  color: t.$text-muted;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  margin-top: 1px;
}

.db-analytics-chart-group__prompt-text {
  font-size: 12.5px;
  font-weight: 600;
  color: t.$text-primary;
  margin: 0;
  line-height: 1.5;
}

.db-analytics-chart-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 12px;
  padding: 12px;
}

@media (max-width: 1500px) {
  .db-analytics-chart-grid {
    grid-template-columns: minmax(0, 1fr);
  }

  .db-analytics-chart-card--span-2 {
    grid-column: auto;
  }
}

.db-analytics-chart-card {
  border: 1px solid t.$border-strong;
  border-radius: t.$radius-lg;
  padding: 14px 16px 16px;
  min-width: 0;
  background: t.$surface-2;
  box-shadow: 0 1px 3px rgba(11, 11, 11, 0.05);
}

.db-analytics-chart-card--span-2 {
  grid-column: 1 / -1;
}

.db-analytics-chart-card__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 14px;
  padding-bottom: 11px;
  border-bottom: 0.5px solid t.$border;
}

.db-analytics-chart-card__heading {
  display: flex;
  align-items: center;
  gap: 9px;
  min-width: 0;
}

.db-analytics-chart-card__icon {
  width: 28px;
  height: 28px;
  border-radius: 8px;
  background: t.$gradient-primary;
  color: #fff;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  box-shadow: 0 2px 6px rgba(79, 70, 229, 0.25);
}

.db-analytics-chart-card__title {
  font-size: 13.5px;
  font-weight: 600;
  color: t.$text-primary;
  margin: 0;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.db-analytics-chart-card__actions {
  display: flex;
  align-items: center;
  gap: 6px;
  flex-shrink: 0;
}

.db-analytics-chart-card__expand-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 24px;
  height: 24px;
  border-radius: 999px;
  border: 0.5px solid t.$border-strong;
  background: t.$surface-0;
  color: t.$text-secondary;
  cursor: pointer;
  flex-shrink: 0;
  transition: background 0.15s ease, border-color 0.15s ease, color 0.15s ease;

  &:hover {
    background: t.$surface-2;
    border-color: t.$border-stronger;
    color: t.$text-primary;
  }
}

.db-analytics-chart-legend {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  row-gap: 8px;
  margin-top: 12px;
  padding-top: 12px;
  border-top: 0.5px solid t.$border;
}

.db-analytics-chart-legend__divider {
  width: 1px;
  align-self: stretch;
  min-height: 16px;
  background: t.$border-strong;
  margin: 0 12px;
  flex-shrink: 0;
}

.db-analytics-chart-legend__item {
  display: flex;
  align-items: center;
  gap: 7px;
  font-size: 12.5px;
  color: t.$text-secondary;
  white-space: nowrap;
}

.db-analytics-chart-legend__swatch {
  width: 9px;
  height: 9px;
  border-radius: 3px;
  flex-shrink: 0;
}

.db-analytics-chart-legend__name {
  max-width: 140px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  color: t.$text-primary;
  font-weight: 500;
}

.db-analytics-chart-legend__pct {
  font-weight: 700;
  color: t.$text-primary;
  font-variant-numeric: tabular-nums;
  flex-shrink: 0;
}

.db-analytics-empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 8px;
  padding: 48px 16px;
  color: t.$text-muted;
  text-align: center;
  min-height: 100%;
  box-sizing: border-box;

  p {
    font-size: 13px;
    margin: 0;
    max-width: 220px;
  }
}

.db-analytics-empty-state__icon--generating {
  color: t.$accent;
  animation: db-analytics-empty-state-draw 1.6s ease-in-out infinite;
}

@keyframes db-analytics-empty-state-draw {
  0% {
    transform: scale(0.9) rotate(-4deg);
    opacity: 0.55;
  }
  50% {
    transform: scale(1.08) rotate(3deg);
    opacity: 1;
  }
  100% {
    transform: scale(0.9) rotate(-4deg);
    opacity: 0.55;
  }
}

.db-analytics-kpi-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(140px, 1fr));
  gap: 12px;
}

.db-analytics-kpi-card {
  position: relative;
  display: flex;
  flex-direction: column;
  gap: 6px;
  padding: 16px 16px 14px;
  border-radius: t.$radius-lg;
  background: t.$surface-0;
  border: 1px solid t.$border;
  overflow: hidden;
  transition: transform 0.15s ease, box-shadow 0.15s ease, border-color 0.15s ease;

  &:hover {
    transform: translateY(-2px);
    box-shadow: 0 6px 16px rgba(11, 11, 11, 0.07);
    border-color: t.$border-strong;
  }
}

.db-analytics-kpi-card__label {
  font-size: 11px;
  font-weight: 600;
  letter-spacing: 0.01em;
  text-transform: uppercase;
  color: t.$text-secondary;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.db-analytics-kpi-card__value {
  font-size: 25px;
  font-weight: 700;
  letter-spacing: -0.01em;
  line-height: 1.15;
  color: t.$text-primary;
}

.db-analytics-table-wrap {
  overflow-x: auto;
  max-height: 260px;
  overflow-y: auto;
}

.db-analytics-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 12px;

  th {
    text-align: left;
    font-weight: 500;
    color: t.$text-secondary;
    padding: 6px 10px;
    border-bottom: 1px solid t.$border-strong;
    background: t.$surface-1;
    position: sticky;
    top: 0;
    white-space: nowrap;
  }

  td {
    padding: 6px 10px;
    border-bottom: 0.5px solid t.$border;
    color: t.$text-primary;
    white-space: nowrap;
  }

  tr:last-child td {
    border-bottom: none;
  }
}

.db-analytics-chart-card__no-data {
  padding: 24px 12px;
  text-align: center;
  font-size: 12px;
  color: t.$text-muted;
}



















//ResultsPanel.tsx
import React, { useState } from 'react';
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
// wouldn't produce anything meaningful.
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

export // Resolves which fields to use for a pie slice's label and value. Prefers
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

export const renderChart = (chart: ChartResult, height = 220, overrideType?: ChartType) => {
  const xKey = chart.x_axis ?? 'name';
  const seriesKeys = deriveSeriesKeys(chart, xKey);
  const gradientPrefix = `db-analytics-grad-${chart.id}`;
  const effectiveType = overrideType ?? chart.chart_type;

  if (!chart.data || chart.data.length === 0) {
    return <div className={styles['db-analytics-chart-card__no-data']}>No data returned for this chart.</div>;
  }

  if (effectiveType === 'kpi') {
    return renderKpi(chart);
  }

  if (effectiveType === 'line') {
    if (seriesKeys.length === 0) {
      return (
        <div className={styles['db-analytics-chart-card__no-data']}>
          No numeric series found to plot for this chart.
        </div>
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
        <div className={styles['db-analytics-chart-card__no-data']}>
          No numeric series found to plot for this chart.
        </div>
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
        <div className={styles['db-analytics-chart-card__no-data']}>
          No numeric series found to plot for this chart.
        </div>
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
  const showEmptyState = chartGroups.length === 0;
  const [typeOverrides, setTypeOverrides] = useState<Record<string, ChartType>>({});
  const [expandedChartId, setExpandedChartId] = useState<string | null>(null);
  const [askAiChartId, setAskAiChartId] = useState<string | null>(null);

  const getDisplayType = (chart: ChartResult): ChartType => typeOverrides[chart.id] ?? chart.chart_type;

  const allCharts = chartGroups.flatMap((g) => g.graphs);
  const expandedChart = allCharts.find((c) => c.id === expandedChartId) ?? null;
  const expandedDisplayType = expandedChart ? getDisplayType(expandedChart) : 'bar';
  const askAiChart = allCharts.find((c) => c.id === askAiChartId) ?? null;
  const askAiDisplayType = askAiChart ? getDisplayType(askAiChart) : 'bar';

  return (
    <div className={styles['db-analytics-results-panel']}>
      <div className={styles['db-analytics-results-panel__header']}>
        <div className={styles['db-analytics-results-panel__header-text']}>
          <h2 className={styles['db-analytics-results-panel__title']}>Generated insights</h2>
          <p className={styles['db-analytics-results-panel__subtitle']}>
            Visualizations from your conversation
          </p>
        </div>
      </div>
      <div className={styles['db-analytics-results-panel__body']}>
        {loadError && <div className={styles['db-analytics-results-panel__error']}>{loadError}</div>}

        {isGenerating && (
          <div className={styles['db-analytics-generating-banner']}>
            <Icon.chartAreaLine
              size={18}
              aria-hidden="true"
              className={styles['db-analytics-empty-state__icon--generating']}
            />
            <span>Drawing your chart…</span>
          </div>
        )}

        {showEmptyState && !isGenerating ? (
          <div className={styles['db-analytics-empty-state']}>
            <Icon.chartAreaLine size={28} aria-hidden="true" />
            <p>Charts generated from your chat will appear here.</p>
          </div>
        ) : (
          <div className={styles['db-analytics-chart-groups']}>
            {chartGroups.map((group, groupIndex) => (
              <React.Fragment key={group.id}>
                {groupIndex > 0 && <div className={styles['db-analytics-chart-groups__divider']} aria-hidden="true" />}
                <section className={styles['db-analytics-chart-group']}>
                  {group.prompt && (
                    <div className={styles['db-analytics-chart-group__prompt']}>
                      <span className={styles['db-analytics-chart-group__prompt-icon']}>
                        <Icon.messageCircle size={12} aria-hidden="true" />
                      </span>
                      <p className={styles['db-analytics-chart-group__prompt-text']}>{group.prompt}</p>
                    </div>
                  )}
                  <div className={styles['db-analytics-chart-grid']}>
                    {group.graphs.map((chart, index) => {
                      const displayType = getDisplayType(chart);
                      const canSwitchType = chart.chart_type !== 'kpi' && chart.chart_type !== 'table';
                      return (
                        <div
                          key={`${chart.id}-${index}`}
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
                                    setTypeOverrides((prev) => ({ ...prev, [chart.id]: type }))
                                  }
                                />
                              )}
                              <button
                                type="button"
                                className={styles['db-analytics-chart-card__expand-btn']}
                                onClick={() => setAskAiChartId(chart.id)}
                                aria-label="Ask AI about this chart"
                                title="Ask AI"
                              >
                                <Icon.askAi size={14} aria-hidden="true" />
                              </button>
                              <button
                                type="button"
                                className={styles['db-analytics-chart-card__expand-btn']}
                                onClick={() => setExpandedChartId(chart.id)}
                                aria-label="Expand chart"
                                title="Expand"
                              >
                                <Icon.expand size={14} aria-hidden="true" />
                              </button>
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
        onClose={() => setExpandedChartId(null)}
      />

      <ChartAskAiSlider
        sessionId={sessionId}
        chart={askAiChart}
        displayType={askAiDisplayType}
        onClose={() => setAskAiChartId(null)}
      />
    </div>
  );
};

export default ResultsPanel;












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










//icons.tsx
// Central place for every SVG icon used across the DBAnalytics feature.
// Swaps the old Tabler icon-font (`<i className="ti ti-x" />`) for real
// inline SVG components from lucide-react.
import React from 'react';
import {
  Plus,
  X,
  Check,
  Database,
  DatabaseZap,
  Settings,
  Info,
  Loader2,
  MessageCircle,
  Sparkles,
  BarChart3,
  Send,
  AreaChart,
  Gauge,
  LineChart,
  PieChart,
  Table,
  ScatterChart,
  Lightbulb,
  ArrowRight,
  Upload,
  FileSpreadsheet,
  Maximize2,
  MessageCircleQuestion,
  type LucideProps,
} from 'lucide-react';

export type IconComponent = React.FC<LucideProps>;

export const Icon = {
  plus: Plus,
  close: X,
  check: Check,
  database: Database,
  databaseInsights: DatabaseZap,
  settings: Settings,
  infoCircle: Info,
  loader: Loader2,
  messageCircle: MessageCircle,
  sparkles: Sparkles,
  chartBar: BarChart3,
  send: Send,
  chartAreaLine: AreaChart,
  gauge: Gauge,
  chartLine: LineChart,
  chartPie: PieChart,
  chartArea: AreaChart,
  chartDots: ScatterChart,
  table: Table,
  lightbulb: Lightbulb,
  arrowRight: ArrowRight,
  upload: Upload,
  csv: FileSpreadsheet,
  expand: Maximize2,
  askAi: MessageCircleQuestion,
} as const;

export type IconName = keyof typeof Icon;











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

// A single prompt/response pairing in column 3 — the user's question and
// the graphs the assistant's reply to it produced. Only assistant messages
// that actually returned graphs become a group.
export interface ChartGroup {
  id: string;
  prompt: string;
  graphs: ChartResult[];
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
