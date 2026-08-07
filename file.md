import { useEffect, useMemo, useState, type FC, type MouseEvent } from 'react';
import { useNavigate } from 'react-router-dom';
import {
  Play,
  Search,
  Copy,
  Trash2,
  X,
  Bot,
  MessageSquare,
  Trophy,
  ListChecks,
  Info,
  FileBarChart,
  Loader2,
  AlertCircle,
  RefreshCw,
  Database,
} from 'lucide-react';
import { fetchEvaluations, fetchEvaluationResults } from './api';
import type { EvaluationListItem, EvaluationResultsResponse, EvaluationResultsErrorResponse } from './types';
import Select from './Select';
import './History.scss';

const TYPE_FILTERS = [
  { value: 'all', label: 'All Types' },
  { value: 'model', label: 'AI Model' },
  { value: 'agent', label: 'Agent' },
  { value: 'rag', label: 'RAG' },
];

const DATE_FILTERS = [
  { value: 30, label: 'Last 30 days' },
  { value: 7, label: 'Last 7 days' },
  { value: Infinity, label: 'All time' },
];

function matchesType(evType: string, filter: string) {
  if (filter === 'all') return true;
  return evType === filter;
}

// Mirrors the icon logic from renderHistory() in the original app.js:
// Agent -> bot, RAG -> search, everything else -> message-square
function HistoryIcon({ type }: { type: string }) {
  if (type === 'agent') return <Bot />;
  if (type === 'rag') return <Search />;
  return <MessageSquare />;
}

function typeTint(type: string): 'violet' | 'blue' | 'amber' {
  if (type === 'agent') return 'violet';
  if (type === 'rag') return 'blue';
  return 'amber';
}

function typeLabel(type: string): string {
  if (type === 'agent') return 'Agent';
  if (type === 'rag') return 'RAG';
  return 'AI Model';
}

function statusLabel(status: string): string {
  switch (status) {
    case 'completed':
      return 'Completed';
    case 'running':
      return 'Running';
    case 'pending':
      return 'Pending';
    case 'failed':
      return 'Failed';
    case 'canceled':
      return 'Canceled';
    default:
      return status;
  }
}

function statusTint(status: string): 'green' | 'blue' | 'amber' | 'danger' | 'violet' {
  switch (status) {
    case 'completed':
      return 'green';
    case 'running':
      return 'blue';
    case 'pending':
      return 'amber';
    case 'failed':
    case 'canceled':
      return 'danger';
    default:
      return 'violet';
  }
}

function daysAgo(dateStr: string): number {
  const then = new Date(dateStr).getTime();
  if (Number.isNaN(then)) return Infinity;
  return Math.floor((Date.now() - then) / (1000 * 60 * 60 * 24));
}

function formatDate(dateStr: string | null): string {
  if (!dateStr) return '—';
  const d = new Date(dateStr);
  if (Number.isNaN(d.getTime())) return '—';
  return d.toLocaleDateString(undefined, { year: 'numeric', month: 'short', day: 'numeric' });
}

interface ApiErrorLike {
  response?: {
    data?: EvaluationResultsErrorResponse;
  };
}

// Add this to History.scss if it's not already there (mirrors .datasets-page__spin):
//   &__spin { animation: history-spin 0.8s linear infinite; }
//   @keyframes history-spin { to { transform: rotate(360deg); } }

const History: FC = () => {
  const navigate = useNavigate();

  // ── Evaluations list (GET /evaluations) ──────────────────────────
  const [items, setItems] = useState<EvaluationListItem[]>([]);
  const [listLoading, setListLoading] = useState(true);
  const [listError, setListError] = useState<string | null>(null);

  const [query, setQuery] = useState('');
  const [typeFilter, setTypeFilter] = useState('all');
  const [dateFilter, setDateFilter] = useState(30);
  const [selectedId, setSelectedId] = useState<string | null>(null);

  // ── Selected evaluation's results (GET /evaluations/{id}/results) ─
  const [results, setResults] = useState<EvaluationResultsResponse | null>(null);
  const [resultsLoading, setResultsLoading] = useState(false);
  const [resultsError, setResultsError] = useState<string | null>(null);

  const loadList = () => {
    setListLoading(true);
    setListError(null);
    fetchEvaluations()
      .then((res) => {
        setItems(res.evaluations);
        setSelectedId((prev) => prev ?? res.evaluations[0]?.id ?? null);
      })
      .catch((err) => {
        setListError(err instanceof Error ? err.message : 'Failed to load evaluation history.');
      })
      .finally(() => setListLoading(false));
  };

  // Background refresh — no loading state, no error banner, no spinner.
  // Keeps the list (and therefore any status changes) current without
  // disturbing whatever the user is looking at.
  const refreshListSilently = () => {
    fetchEvaluations()
      .then((res) => {
        setItems(res.evaluations);
      })
      .catch(() => {
        // Silent by design — a background poll failing shouldn't interrupt
        // the user. loadList()'s error path still covers the initial load.
      });
  };

  useEffect(() => {
    loadList();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  useEffect(() => {
    const interval = setInterval(refreshListSilently, 10000);
    return () => clearInterval(interval);
  }, []);

  const filtered = useMemo(() => {
    return items.filter((ev) => {
      if (query && !ev.name.toLowerCase().includes(query.toLowerCase())) return false;
      if (!matchesType(ev.eval_type, typeFilter)) return false;
      if (daysAgo(ev.created_at) > dateFilter) return false;
      return true;
    });
  }, [items, query, typeFilter, dateFilter]);

  const selected = useMemo(
    () => filtered.find((ev) => ev.id === selectedId) ?? filtered[0] ?? null,
    [filtered, selectedId]
  );

  // Lazy-fetch results whenever the selected evaluation changes.
  // Only fires the network call for completed runs.
  useEffect(() => {
    if (!selected) {
      setResults(null);
      setResultsError(null);
      setResultsLoading(false);
      return;
    }

    let cancelled = false;
    setResults(null);
    setResultsError(null);

    if (selected.status !== 'completed') {
      setResultsLoading(false);
      return;
    }

    setResultsLoading(true);
    fetchEvaluationResults(selected.id)
      .then((res) => {
        if (cancelled) return;
        setResults(res);
      })
      .catch((err: ApiErrorLike | Error) => {
        if (cancelled) return;
        const detail = (err as ApiErrorLike)?.response?.data?.detail;
        setResultsError(detail ?? (err instanceof Error ? err.message : 'Failed to load results.'));
      })
      .finally(() => {
        if (!cancelled) setResultsLoading(false);
      });

    return () => {
      cancelled = true;
    };
    // Depend on the primitive id/status rather than the `selected` object
    // itself — `selected` is a new reference every time the silent 10s
    // poll refreshes `items`, even when nothing about this row changed.
    // Keying off id/status means results only refetch when the selection
    // changes or the evaluation's status actually transitions (e.g. to
    // "completed"), not on every background refresh.
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [selected?.id, selected?.status]);

  // Mirrors duplicateEval(id): send the user into a fresh Run Evaluation flow.
  const handleDuplicate = (e: MouseEvent, _id: string) => {
    e.stopPropagation();
    navigate('/app/run-evaluation');
  };

  // Mirrors deleteEval(id): confirm, then remove from the list.
  // NOTE: no DELETE /evaluations/{id} endpoint was provided — this only
  // removes the row from local state. Wire in a real delete call once
  // that endpoint exists.
  const handleDelete = (e: MouseEvent, id: string) => {
    e.stopPropagation();
    if (window.confirm('Delete this evaluation?')) {
      setItems((prev) => prev.filter((ev) => ev.id !== id));
      if (selectedId === id) setSelectedId(null);
    }
  };

  return (
    <div className="history">
      <div className="history__header">
        <div className="history__header-left">
          <p className="history__header-eyebrow">Evaluation records</p>
          <h1 className="history__title">History</h1>
          <p className="history__subtitle">Past evaluations</p>
        </div>

        <div className="history__header-right">
          <div className="history__header-meta">
            <Database size={13} />
            {items.length} evaluations logged
          </div>
          <button type="button" className="history__btn history__btn--primary" onClick={() => navigate('/app/run-evaluation')}>
            <Play size={14} strokeWidth={2.25} /> New Evaluation
          </button>
        </div>
      </div>

      <div className="history__filters">
        <div className="history__search">
          <Search size={15} />
          <input type="text" placeholder="Search..." value={query} onChange={(e) => setQuery(e.target.value)} />
          {query && (
            <button type="button" className="history__search-clear" onClick={() => setQuery('')} aria-label="Clear search">
              <X size={13} />
            </button>
          )}
        </div>
        <Select value={typeFilter} options={TYPE_FILTERS} onChange={setTypeFilter} width={140} />
        <Select value={dateFilter} options={DATE_FILTERS} onChange={setDateFilter} width={140} />
      </div>

      {listLoading && (
        <div className="history__empty">
          <Loader2 size={22} className="history__spin" />
          <p>Loading history…</p>
        </div>
      )}

      {!listLoading && listError && (
        <div className="history__empty">
          <AlertCircle size={22} />
          <p>{listError}</p>
          <button type="button" className="history__btn" onClick={loadList}>
            <RefreshCw size={14} strokeWidth={2.25} /> Try again
          </button>
        </div>
      )}

      {!listLoading && !listError && filtered.length === 0 && (
        <div className="history__empty">
          <Search size={22} />
          <p>No evaluations match your filters.</p>
        </div>
      )}

      {!listLoading && !listError && filtered.length > 0 && (
        <div className="history__body">
          <div className="history__list-panel">
            <div className="history__list">
              {filtered.map((ev) => {
                const isActive = selected?.id === ev.id;
                const isRunning = ev.status === 'running';
                return (
                  <div
                    key={ev.id}
                    className={`history__item${isActive ? ' history__item--active' : ''}${
                      isRunning ? ' history__item--running' : ''
                    }`}
                    onClick={() => setSelectedId(ev.id)}
                  >
                    <div className="history__icon">
                      <HistoryIcon type={ev.eval_type} />
                    </div>

                    <div className="history__content">
                      <h4>{ev.name}</h4>
                      <div className="history__meta">
                        <span className="history__type">{typeLabel(ev.eval_type)}</span>
                        <span className={`history__status-badge history__status-badge--${statusTint(ev.status)}${
                          isRunning ? ' history__status-badge--live' : ''
                        }`}>
                          {isRunning && <span className="history__status-dot" />}
                          {statusLabel(ev.status)}
                        </span>
                        <span>{formatDate(ev.created_at)}</span>
                      </div>
                    </div>

                    <div className="history__results">
                      <div className="history__stat">
                        <span className="history__stat-label">Winner</span>
                        <span className="history__stat-value">{ev.top_model ?? '—'}</span>
                      </div>
                      <div className="history__stat">
                        <span className="history__stat-label">Score</span>
                        <span className="history__stat-value history__stat-value--highlight n">
                          {ev.top_score != null ? ev.top_score.toFixed(3) : '—'}
                        </span>
                      </div>
                      <div className="history__stat">
                        <span className="history__stat-label">Models</span>
                        <span className="history__stat-value n">{ev.model_ids.length}</span>
                      </div>
                    </div>

                    <div className="history__actions">
                      <button
                        type="button"
                        className="history__btn history__btn--sm"
                        onClick={(e) => handleDuplicate(e, ev.id)}
                        aria-label="Duplicate evaluation"
                      >
                        <Copy size={16} />
                      </button>
                      <button
                        type="button"
                        className="history__btn history__btn--sm"
                        onClick={(e) => handleDelete(e, ev.id)}
                        aria-label="Delete evaluation"
                      >
                        <Trash2 size={16} />
                      </button>
                    </div>
                  </div>
                );
              })}
            </div>
          </div>

          {selected && (
            <div className="history__detail-panel">
              <div className="history__detail-head">
                <div className="history__detail-head-left">
                  <span className={`history__type-badge history__type-badge--${typeTint(selected.eval_type)}`}>
                    {typeLabel(selected.eval_type)}
                  </span>
                  <h2 className="history__detail-name">{selected.name}</h2>
                  <span className="history__detail-date">
                    <span
                      className={`history__status-badge history__status-badge--${statusTint(selected.status)}${
                        selected.status === 'running' ? ' history__status-badge--live' : ''
                      }`}
                    >
                      {selected.status === 'running' && <span className="history__status-dot" />}
                      {statusLabel(selected.status)}
                    </span>
                    &middot; {formatDate(selected.created_at)}
                  </span>
                </div>

                <div className="history__detail-actions">
                  <button type="button" className="history__btn" onClick={(e) => handleDuplicate(e, selected.id)}>
                    <Copy size={13} /> Duplicate
                  </button>
                  <button type="button" className="history__btn history__btn--danger" onClick={(e) => handleDelete(e, selected.id)}>
                    <Trash2 size={13} /> Delete
                  </button>
                  <button
                    type="button"
                    className="history__btn history__btn--primary"
                    onClick={() => navigate('/app/reports', { state: { evaluationId: selected.id } })}
                    disabled={selected.status !== 'completed'}
                  >
                    <FileBarChart size={13} /> View Report
                  </button>
                </div>
              </div>

              {/* Summary cards — winner card kept as-is; the other two are repurposed
                  to fields the real API actually returns (no avg response time / cost). */}
              <div className="history__summary-cards">
                <div className="history__summary-card history__summary-card--winner">
                  <div className="history__summary-icon">
                    <Trophy />
                  </div>
                  <div className="history__summary-content">
                    <div className="history__summary-label">Winner</div>
                    <div className="history__summary-value">{selected.top_model ?? '—'}</div>
                    <div className="history__summary-score">
                      {selected.top_score != null ? selected.top_score.toFixed(3) : '—'}
                    </div>
                  </div>
                </div>
                <div className="history__summary-card">
                  <div className="history__summary-icon">
                    <ListChecks />
                  </div>
                  <div className="history__summary-content">
                    <div className="history__summary-label">Total Questions</div>
                    <div className="history__summary-value">{selected.total_questions}</div>
                    <div className="history__summary-score">{selected.model_ids.length} models tested</div>
                  </div>
                </div>
                <div className="history__summary-card">
                  <div className="history__summary-icon">
                    <Info />
                  </div>
                  <div className="history__summary-content">
                    <div className="history__summary-label">Status</div>
                    <div className="history__summary-value">{statusLabel(selected.status)}</div>
                    <div className="history__summary-score">
                      {selected.completed_at ? `Completed ${formatDate(selected.completed_at)}` : formatDate(selected.started_at)}
                    </div>
                  </div>
                </div>
              </div>

              <p className="history__section-title">Full results</p>

              {selected.status !== 'completed' && (
                <div className="history__empty">
                  <AlertCircle size={22} />
                  <p>
                    {selected.status === 'running' || selected.status === 'pending'
                      ? 'This evaluation is still running. Results will appear once it completes.'
                      : selected.status === 'failed'
                      ? 'This evaluation failed, so no results are available.'
                      : 'This evaluation was canceled, so no results are available.'}
                  </p>
                </div>
              )}

              {selected.status === 'completed' && resultsLoading && (
                <div className="history__empty">
                  <Loader2 size={22} className="history__spin" />
                  <p>Loading results…</p>
                </div>
              )}

              {selected.status === 'completed' && !resultsLoading && resultsError && (
                <div className="history__empty">
                  <AlertCircle size={22} />
                  <p>{resultsError}</p>
                </div>
              )}

              {selected.status === 'completed' && !resultsLoading && !resultsError && results && (
                <div className="history__table-wrap">
                  <table className="history__table">
                    <thead>
                      <tr>
                        <th style={{ width: 48 }}>Rank</th>
                        <th>Model</th>
                        <th>Provider</th>
                        <th>Score</th>
                        <th>Accuracy</th>
                        <th>Passed</th>
                        <th>Failed</th>
                      </tr>
                    </thead>
                    <tbody>
                      {results.results
                        .slice()
                        .sort((a, b) => a.rank - b.rank)
                        .map((r) => (
                          <tr key={r.model_id}>
                            <td>
                              <span className={`history__rank-pill${r.rank === 1 ? ' history__rank-pill--1' : ''}`}>{r.rank}</span>
                            </td>
                            <td className="history__cell-strong">{r.model_id}</td>
                            <td>{r.provider}</td>
                            <td className={`n${r.rank === 1 ? ' history__score-cell' : ''}`}>{r.score.toFixed(3)}</td>
                            <td className="n">{r.accuracy.toFixed(3)}</td>
                            <td className="n">
                              {r.passed_tests}/{r.total_tests}
                            </td>
                            <td className="n">{r.failed_tests}</td>
                          </tr>
                        ))}
                    </tbody>
                  </table>
                </div>
              )}
            </div>
          )}
        </div>
      )}
    </div>
  );
};

export default History;






















@use '../../../styles/variables' as *;

.history {
  display: flex;
  flex-direction: column;
  gap: 18px;
  min-height: 0;
  // .main-layout__body / .workspace-layout / .workspace-layout__content form an
  // unbroken flex chain sized to the viewport (header/footer are fixed and offset
  // via margin), so height: 100% here resolves to the visible content area.
  // workspace-layout__content also has a 3rem bottom padding, which would
  // otherwise leave a large gap below .history — pull most of that back in,
  // keeping just a small 0.75rem breathing-room strip at the true bottom edge,
  // and giving the two scrollable panels below a real height to scroll within.
  height: calc(100% + 3rem - 0.75rem);
  margin-bottom: calc(-3rem + 0.75rem);

  /* ---------- header — matches run-eval__header's eyebrow/title/subtitle + meta-pill pattern ---------- */
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

  &__header-right {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 10px;
    margin-bottom: 3px;
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
  }

  &__title {
    font-size: 21px;
    font-weight: 800;
    letter-spacing: -0.03em;
    color: $text-primary;
    line-height: 1.15;
  }

  &__subtitle {
    margin-top: 3px;
    color: $text-secondary;
    font-size: 0.84375rem;
  }

  /* ---------- generic buttons ---------- */
  &__btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 7px;
    font-family: $font-body;
    font-size: 0.8125rem;
    font-weight: 600;
    padding: 9px 14px;
    border-radius: 8px;
    border: 1px solid $border-default;
    background: $bg-main;
    color: $text-secondary;
    cursor: pointer;
    white-space: nowrap;
    transition: background 0.12s ease, border-color 0.12s ease, color 0.12s ease, box-shadow 0.12s ease;

    &:hover {
      border-color: $primary;
      box-shadow: $shadow-sm;
    }

    &--primary {
      background: $primary;
      border-color: $primary;
      color: #fff;

      &:hover {
        background: $primary-hover;
        border-color: $primary-hover;
        color: #fff;
      }
    }

    &--danger:hover {
      border-color: $danger;
      color: $danger;
      background: $danger-subtle;
    }

    &--sm {
      padding: 6px;

      svg {
        width: 15px;
        height: 15px;
      }
    }
  }

  /* ---------- filters ---------- */
  &__filters {
    flex-shrink: 0;
    display: flex;
    flex-wrap: wrap;
    gap: 10px;
  }

  &__search {
    display: flex;
    align-items: center;
    gap: 9px;
    width: 280px;
    max-width: 100%;
    border: 1px solid $border-default;
    border-radius: 10px;
    padding: 9px 12px;
    background: $bg-main;
    color: $text-tertiary;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &:focus-within {
      border-color: $primary;
      box-shadow: 0 0 0 3px $primary-light;
    }

    input {
      flex: 1;
      border: none;
      outline: none;
      font-size: 0.8125rem;
      color: $text-primary;
      background: transparent;
      font-family: $font-body;
      min-width: 0;

      &::placeholder {
        color: $text-tertiary;
      }
    }
  }

  &__search-clear {
    flex-shrink: 0;
    width: 18px;
    height: 18px;
    border-radius: 50%;
    border: none;
    background: $bg-inset;
    color: $text-tertiary;
    display: grid;
    place-items: center;
    cursor: pointer;
    transition: background 0.14s ease, color 0.14s ease;

    &:hover {
      background: $border-default;
      color: $text-primary;
    }
  }

  /* ---------- custom dropdown (used by <Select />) ---------- */
  &-select {
    position: relative;
    flex-shrink: 0;

    &__trigger {
      width: 100%;
      display: flex;
      align-items: center;
      justify-content: space-between;
      gap: 8px;
      border: 1px solid $border-default;
      border-radius: 10px;
      padding: 9px 12px;
      background: $bg-main;
      font-size: 0.8125rem;
      font-weight: 500;
      font-family: $font-body;
      color: $text-primary;
      cursor: pointer;
      transition: border-color 0.14s ease, box-shadow 0.14s ease;

      &:hover {
        border-color: $border-strong;
      }

      &--open {
        border-color: $primary;
        box-shadow: 0 0 0 3px $primary-light;
      }
    }

    &__chevron {
      flex-shrink: 0;
      color: $text-tertiary;
      transition: transform 0.16s ease;
    }

    &__trigger--open &__chevron {
      transform: rotate(180deg);
    }

    &__menu {
      position: absolute;
      top: calc(100% + 6px);
      left: 0;
      right: 0;
      z-index: 20;
      background: $bg-main;
      border: 1px solid $border-subtle;
      border-radius: 10px;
      box-shadow: $shadow-lg;
      padding: 5px;
      display: flex;
      flex-direction: column;
      gap: 1px;
    }

    &__option {
      display: flex;
      align-items: center;
      justify-content: space-between;
      gap: 8px;
      width: 100%;
      text-align: left;
      padding: 8px 10px;
      border: none;
      border-radius: 7px;
      background: transparent;
      font-size: 0.8125rem;
      font-family: $font-body;
      color: $text-secondary;
      cursor: pointer;
      transition: background 0.12s ease, color 0.12s ease;

      &:hover {
        background: $bg-subtle;
        color: $text-primary;
      }

      &--active {
        color: $primary;
        font-weight: 600;

        svg {
          color: $primary;
        }
      }
    }
  }

  /* ---------- default two-column layout: 400px list + detail, each independently scrollable ---------- */
  &__body {
    flex: 1;
    display: flex;
    gap: 16px;
    min-height: 0;
  }

  &__list-panel {
    flex: 0 0 400px;
    max-width: 400px;
    min-height: 0;
    overflow-y: auto;
  }

  /* ---------- list ---------- */
  &__list {
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  &__item {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 10px;
    padding: 14px 16px;
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: $radius-lg;
    cursor: pointer;
    transition: border-color 0.12s ease, box-shadow 0.12s ease, background 0.12s ease;

    &:hover {
      border-color: $primary;
      box-shadow: $shadow-sm;
    }

    &--active {
      border-color: $primary;
      background: $primary-light;
      box-shadow: $shadow-sm;
    }

    // Running evaluations get a slow, subtle glow pulse around the card
    // border so it reads as "live" without being distracting in a list.
    &--running {
      border-color: rgba($primary, 0.35);
      animation: history-card-pulse 2.2s ease-in-out infinite;
    }

    &--running.history__item--active {
      animation: history-card-pulse-active 2.2s ease-in-out infinite;
    }
  }

  @keyframes history-card-pulse {
    0%,
    100% {
      box-shadow: 0 0 0 0 rgba($primary, 0.14);
    }
    50% {
      box-shadow: 0 0 0 5px rgba($primary, 0.08);
    }
  }

  @keyframes history-card-pulse-active {
    0%,
    100% {
      box-shadow: $shadow-sm, 0 0 0 0 rgba($primary, 0.18);
    }
    50% {
      box-shadow: $shadow-sm, 0 0 0 6px rgba($primary, 0.1);
    }
  }

  &__icon {
    width: 36px;
    height: 36px;
    background: $primary-light;
    border-radius: $radius-md;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-shrink: 0;

    svg {
      width: 18px;
      height: 18px;
      color: $primary;
      stroke-width: 1.5;
    }
  }

  &__item--active &__icon {
    background: $bg-main;
  }

  &__content {
    flex-basis: calc(100% - 46px);
    flex-grow: 1;
    min-width: 0;

    h4 {
      font-size: 14px;
      font-weight: 500;
      margin-bottom: 4px;
      color: $text-primary;
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
    }
  }

  &__meta {
    display: flex;
    align-items: center;
    gap: 10px;
    font-size: 12px;
    color: $text-tertiary;
    overflow: hidden;

    span:last-child {
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
    }
  }

  &__type {
    flex-shrink: 0;
    padding: 2px 8px;
    background: $primary-light;
    color: $primary;
    border-radius: 4px;
    font-weight: 500;
  }

  &__item--active &__type {
    background: $bg-main;
  }

  /* ---------- status badge (list row + detail header) ---------- */
  &__status-badge {
    flex-shrink: 0;
    font-size: 0.625rem;
    font-weight: 700;
    letter-spacing: 0.02em;
    border-radius: 999px;
    padding: 2px 8px;
    white-space: nowrap;

    &--green {
      color: $success;
      background: $success-subtle;
    }

    &--blue {
      color: $primary;
      background: $primary-light;
    }

    &--amber {
      color: $warning;
      background: $warning-subtle;
    }

    &--violet {
      color: $violet;
      background: $violet-light;
    }

    &--danger {
      color: $danger;
      background: $danger-subtle;
    }
  }

  &__item--active &__status-badge--green,
  &__item--active &__status-badge--blue,
  &__item--active &__status-badge--amber,
  &__item--active &__status-badge--violet,
  &__item--active &__status-badge--danger {
    background: $bg-main;
  }

  &__status-badge--live {
    display: inline-flex;
    align-items: center;
    gap: 4px;
  }

  &__status-dot {
    flex-shrink: 0;
    width: 6px;
    height: 6px;
    border-radius: 50%;
    background: currentColor;
    animation: history-dot-pulse 1.4s ease-in-out infinite;
  }

  @keyframes history-dot-pulse {
    0%,
    100% {
      opacity: 1;
      transform: scale(1);
    }
    50% {
      opacity: 0.35;
      transform: scale(0.7);
    }
  }

  &__detail-date &__status-badge {
    margin-right: 6px;
    vertical-align: 1px;
  }

  &__results {
    display: flex;
    gap: 16px;
    flex-shrink: 0;
  }

  &__stat {
    display: flex;
    flex-direction: column;
  }

  &__stat-label {
    font-size: 10px;
    text-transform: uppercase;
    letter-spacing: 0.4px;
    color: $text-tertiary;
    margin-bottom: 2px;
  }

  &__stat-value {
    font-size: 13px;
    font-weight: 500;
    color: $text-primary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    max-width: 84px;

    &--highlight {
      color: $primary;
    }
  }

  &__actions {
    display: flex;
    gap: 6px;
    flex-shrink: 0;
    margin-left: auto;
  }

  /* ---------- detail panel ---------- */
  &__detail-panel {
    flex: 1;
    min-width: 0;
    min-height: 0;
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 16px;
    box-shadow: $shadow-xs;
    padding: 26px 28px;
    overflow-y: auto;
  }

  &__detail-head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
    padding-bottom: 18px;
    margin-bottom: 22px;
    border-bottom: 1px solid $border-subtle;
  }

  &__detail-head-left {
    display: flex;
    flex-direction: column;
    gap: 8px;
    min-width: 0;
  }

  &__detail-name {
    font-size: 1.25rem;
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $text-primary;
  }

  &__detail-date {
    font-size: 0.8125rem;
    color: $text-tertiary;
  }

  &__detail-actions {
    display: flex;
    gap: 8px;
    flex-shrink: 0;
    flex-wrap: wrap;
  }

  /* ---------- type badge (sidebar + detail) ---------- */
  &__type-badge {
    width: fit-content;
    flex-shrink: 0;
    font-size: 0.625rem;
    font-weight: 700;
    border-radius: 999px;
    padding: 2px 8px;

    &--violet {
      color: $violet;
      background: $violet-light;
    }

    &--blue {
      color: $primary;
      background: $primary-light;
    }

    &--amber {
      color: $warning;
      background: $warning-subtle;
    }
  }

  /* ---------- summary cards — matches reference .results-summary-cards / .summary-card ---------- */
  &__summary-cards {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 16px;
    margin-bottom: 24px;
  }

  &__summary-card {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: $radius-lg;
    padding: 20px;
    display: flex;
    gap: 14px;
    min-width: 0;

    &--winner {
      // Reference uses a fixed warm amber gradient/border for the winner
      // card in both themes (it's a celebratory accent, not a surface
      // token), so this intentionally does not switch with dark mode.
      background: linear-gradient(135deg, #fffbeb 0%, #fef3c7 100%);
      border-color: #fde68a;
    }
  }

  &__summary-icon {
    width: 44px;
    height: 44px;
    background: $bg-subtle;
    border-radius: $radius-md;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-shrink: 0;

    svg {
      width: 22px;
      height: 22px;
      color: $text-secondary;
      stroke-width: 1.5;
    }
  }

  &__summary-card--winner &__summary-icon {
    background: rgba(180, 83, 9, 0.12);

    svg {
      color: #b45309;
    }
  }

  &__summary-content {
    min-width: 0;
  }

  &__summary-label {
    font-size: 11px;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    color: $text-tertiary;
    margin-bottom: 4px;
  }

  &__summary-value {
    font-size: 16px;
    font-weight: 600;
    color: $text-primary;
    margin-bottom: 2px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__summary-score {
    font-size: 13px;
    color: $text-secondary;
  }

  /* ---------- results table ---------- */
  &__section-title {
    font-size: 0.8125rem;
    font-weight: 700;
    color: $text-primary;
    margin-bottom: 12px;
  }

  &__table-wrap {
    overflow-x: auto;
  }

  &__table {
    width: 100%;
    border-collapse: collapse;

    thead th {
      text-align: left;
      font-size: 0.6875rem;
      font-weight: 700;
      letter-spacing: 0.05em;
      text-transform: uppercase;
      color: $text-tertiary;
      padding: 10px 14px;
      background: $bg-subtle;
      white-space: nowrap;

      &:first-child {
        border-radius: 8px 0 0 8px;
      }

      &:last-child {
        border-radius: 0 8px 8px 0;
      }
    }

    tbody td {
      padding: 12px 14px;
      font-size: 0.84375rem;
      color: $text-secondary;
      border-bottom: 1px solid $border-subtle;
      white-space: nowrap;
    }

    tbody tr:last-child td {
      border-bottom: none;
    }
  }

  &__cell-strong {
    font-weight: 600;
    color: $text-primary;
  }

  &__rank-pill {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 22px;
    height: 22px;
    border-radius: 6px;
    background: $bg-inset;
    color: $text-tertiary;
    font-size: 0.6875rem;
    font-weight: 700;

    &--1 {
      background: $primary-light;
      color: $primary;
    }
  }

  &__score-cell {
    color: $success;
    font-weight: 700;
  }

  /* ---------- loading spinner (used by History.tsx's Loader2 icons) ---------- */
  &__spin {
    animation: history-spin 0.8s linear infinite;
  }

  @keyframes history-spin {
    to {
      transform: rotate(360deg);
    }
  }

  /* ---------- empty state ---------- */
  &__empty {
    flex-shrink: 0;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 10px;
    padding: 52px 20px;
    border: 1px dashed $border-strong;
    border-radius: 14px;
    color: $text-tertiary;
    font-size: 0.84375rem;

    svg {
      color: $text-tertiary;
    }
  }

  /* ---------- responsive ---------- */
  @media (max-width: 900px) {
    height: auto;
    margin-bottom: 0;

    &__header {
      flex-direction: column;
      align-items: flex-start;
    }

    &__header-right {
      margin-bottom: 0;
    }

    &__body {
      flex-direction: column;
    }

    &__list-panel {
      flex: 0 0 auto;
      max-width: none;
      max-height: 16rem;
    }

    &__detail-panel {
      max-height: none;
    }

    &__summary-cards {
      grid-template-columns: 1fr;
    }
  }

  @media (max-width: 520px) {
    &__header-right {
      width: 100%;
      justify-content: space-between;
    }

    &__detail-panel {
      padding: 18px 16px;
    }

    &__detail-head {
      flex-direction: column;
    }
  }
}
