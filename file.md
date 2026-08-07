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
                return (
                  <div
                    key={ev.id}
                    className={`history__item${isActive ? ' history__item--active' : ''}`}
                    onClick={() => setSelectedId(ev.id)}
                  >
                    <div className="history__icon">
                      <HistoryIcon type={ev.eval_type} />
                    </div>

                    <div className="history__content">
                      <h4>{ev.name}</h4>
                      <div className="history__meta">
                        <span className="history__type">{typeLabel(ev.eval_type)}</span>
                        <span className={`history__status-badge history__status-badge--${statusTint(ev.status)}`}>
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
                    <span className={`history__status-badge history__status-badge--${statusTint(selected.status)}`}>
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
