//data.ts
export interface Report {
  id: string;
  title: string;
  date: string;
  summary: string;
  topModel: string;
  verdict: string;
  metricsTested: string[];
  downloadSize: string;
}

export const REPORTS: Report[] = [
  {
    id: 'rep-1',
    title: 'Executive Recommendation: Autonomous Agent Standardization',
    date: 'July 26, 2026',
    summary:
      'After subjecting 6 candidate models to the Autonomous Tool & Workflow Suite, Model Alpha Agent and Model Delta Agent v2 emerged as top performers. Model Alpha offers a 76% cost reduction and 35% speed gain while matching proprietary tool execution accuracy (99.1%).',
    topModel: 'Model Alpha Agent',
    verdict: 'Adopt Model Alpha Agent for high-volume automated tools; preserve Model Delta Agent v2 for highly ambiguous policy edge-cases.',
    metricsTested: ['Tool Calling Success', 'Response Time', 'Task Accuracy', 'Cost per 1,000 Tasks'],
    downloadSize: '2.4 MB PDF',
  },
  {
    id: 'rep-2',
    title: 'Quarterly RAG Factual Accuracy Assessment',
    date: 'July 22, 2026',
    summary:
      'Evaluation of our customer-facing knowledge assistant across 350 ground-truth QA pairs. Model Theta Long-Context demonstrated the lowest incorrect answer rate (0.2%), while Groq-hosted Model Zeta Instruct provided the fastest user interaction speed (280 t/s).',
    topModel: 'Model Theta Long-Context',
    verdict: 'Implement Model Zeta Instruct as frontline responder with automated fall-back to Model Theta Long-Context for queries over 30,000 characters.',
    metricsTested: ['Incorrect Answer Rate', 'Response Time', 'Retrieval Precision'],
    downloadSize: '1.8 MB PDF',
  },
  {
    id: 'rep-3',
    title: 'Software Engineering Agent Benchmark Review',
    date: 'July 14, 2026',
    summary:
      'Ran 500 real-world GitHub issue resolutions through SWE-bench Verified. Model Delta Agent v2 independently diagnosed and patched 95.9% of issues with passing unit tests, ahead of every other candidate by a wide margin.',
    topModel: 'Model Delta Agent v2',
    verdict: 'Roll out Model Delta Agent v2 for the internal bug-triage pipeline; keep a human reviewer in the loop for security-sensitive patches.',
    metricsTested: ['Patch Success Rate', 'Test Pass Rate', 'Time to Resolution'],
    downloadSize: '3.1 MB PDF',
  },
];












//Reports.tsx
import type { FC } from 'react';
import { FileBarChart, Download, Share2, Lightbulb } from 'lucide-react';
import { REPORTS } from './data';
import './Reports.scss';

const Reports: FC = () => {
  return (
    <div className="reports-page">
      <div className="reports-page__header">
        <div className="reports-page__header-left">
          <p className="reports-page__header-eyebrow">Executive summaries</p>
          <h1 className="reports-page__title">Reports</h1>
          <p className="reports-page__subtitle">Generated analysis and recommendations</p>
        </div>

        <div className="reports-page__header-meta">
          <FileBarChart size={13} />
          {REPORTS.length} reports generated
        </div>
      </div>

      <div className="reports-page__body">
        <div className="reports-page__list">
          {REPORTS.map((r) => (
            <div className="reports-page__card" key={r.id}>
              <div className="reports-page__card-head">
                <span className="reports-page__date">{r.date}</span>
                <div className="reports-page__actions">
                  <button type="button" className="reports-page__btn">
                    <Download size={13} /> {r.downloadSize}
                  </button>
                  <button type="button" className="reports-page__icon-btn" aria-label="Share report" title="Share report">
                    <Share2 size={13} />
                  </button>
                </div>
              </div>

              <h3 className="reports-page__title-text">{r.title}</h3>
              <p className="reports-page__summary">{r.summary}</p>

              <div className="reports-page__verdict">
                <span className="reports-page__verdict-icon">
                  <Lightbulb size={15} strokeWidth={2} />
                </span>
                <div>
                  <strong>Recommendation</strong>
                  <p>{r.verdict}</p>
                </div>
              </div>

              <div className="reports-page__footer">
                <div className="reports-page__top-model">
                  <span className="reports-page__top-label">Top:</span>
                  <span className="reports-page__top-value">{r.topModel}</span>
                </div>
                <div className="reports-page__metrics">
                  {r.metricsTested.map((m) => (
                    <span key={m} className="reports-page__metric-tag">
                      {m}
                    </span>
                  ))}
                </div>
              </div>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
};

export default Reports;

















//Reports.scss
@use '../../../styles/variables' as *;

.reports-page {
  display: flex;
  flex-direction: column;
  height: calc(100vh - 166px);
  min-height: 0;
  gap: 16px;

  /* ---------- header ---------- */
  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding-bottom: 18px;
    margin-bottom: 2px;
    border-bottom: 1px solid $border-subtle;
  }

  &__header-left {
    display: flex;
    flex-direction: column;
  }

  &__header-eyebrow {
    display: flex;
    align-items: center;
    gap: 8px;
    font-family: $font-mono;
    font-size: 0.6875rem;
    font-weight: 700;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    color: $primary;
    margin-bottom: 6px;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $primary;
    }
  }

  &__header-meta {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-secondary;
    background: $bg-subtle;
    border: 1px solid $border-subtle;
    border-radius: 999px;
    padding: 7px 13px;
    white-space: nowrap;
    margin-bottom: 3px;
  }

  &__title {
    font-size: 21px;
    font-weight: 800;
    letter-spacing: -0.02em;
    color: $text-primary;
  }

  &__subtitle {
    margin-top: 3px;
    color: $text-secondary;
    font-size: 0.84375rem;
  }

  /* ---------- scrollable body ---------- */
  &__body {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding-right: 4px;
    margin-right: -4px;
  }

  &__list {
    display: flex;
    flex-direction: column;
    gap: 14px;
    max-width: 46rem;
  }

  /* ---------- report card ---------- */
  &__card {
    display: flex;
    flex-direction: column;
    gap: 12px;
    padding: 22px 24px;
    border: 1px solid $border-subtle;
    border-radius: 14px;
    background: $bg-main;
    box-shadow: $shadow-xs;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &:hover {
      border-color: $border-strong;
      box-shadow: $shadow-sm;
    }
  }

  &__card-head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
  }

  &__date {
    font-family: $font-mono;
    font-size: 0.71875rem;
    font-weight: 600;
    color: $text-tertiary;
  }

  &__actions {
    display: flex;
    align-items: center;
    gap: 6px;
  }

  &__btn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-family: $font-body;
    font-size: 0.75rem;
    font-weight: 600;
    padding: 6px 11px;
    border-radius: 7px;
    border: 1px solid $border-default;
    background: $bg-main;
    color: $text-primary;
    cursor: pointer;
    transition: border-color 0.14s ease;

    &:hover {
      border-color: $text-primary;
    }
  }

  &__icon-btn {
    display: grid;
    place-items: center;
    width: 28px;
    height: 28px;
    border-radius: 7px;
    border: 1px solid $border-default;
    background: $bg-main;
    color: $text-secondary;
    cursor: pointer;
    transition: border-color 0.14s ease, color 0.14s ease;

    &:hover {
      border-color: $text-primary;
      color: $text-primary;
    }
  }

  &__title-text {
    font-size: 1.0625rem;
    font-weight: 700;
    letter-spacing: -0.01em;
    line-height: 1.35;
    color: $text-primary;
  }

  &__summary {
    margin-top: -6px;
    font-size: 0.84375rem;
    line-height: 1.6;
    color: $text-secondary;
  }

  /* ---------- verdict box ---------- */
  &__verdict {
    display: flex;
    gap: 11px;
    padding: 13px 15px;
    background: $primary-light;
    border-radius: 10px;

    strong {
      display: block;
      font-size: 0.75rem;
      font-weight: 700;
      color: $primary;
      margin-bottom: 3px;
    }

    p {
      font-size: 0.8125rem;
      line-height: 1.5;
      color: $text-primary;
    }
  }

  &__verdict-icon {
    flex-shrink: 0;
    width: 30px;
    height: 30px;
    border-radius: 9px;
    background: $bg-main;
    color: $primary;
    display: grid;
    place-items: center;
  }

  /* ---------- footer ---------- */
  &__footer {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
    margin-top: 2px;
    padding-top: 13px;
    border-top: 1px solid $border-subtle;
  }

  &__top-model {
    display: flex;
    align-items: baseline;
    gap: 6px;
  }

  &__top-label {
    font-size: 0.71875rem;
    color: $text-tertiary;
  }

  &__top-value {
    font-size: 0.8125rem;
    font-weight: 700;
    color: $text-primary;
  }

  &__metrics {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
  }

  &__metric-tag {
    font-size: 0.6875rem;
    font-weight: 500;
    color: $text-secondary;
    background: $bg-subtle;
    border: 1px solid $border-subtle;
    border-radius: 6px;
    padding: 3px 8px;
  }
}
