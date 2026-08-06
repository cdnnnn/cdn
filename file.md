//types.ts
/* Add these to your shared types.ts (or a local History-scoped types file) */

export type EvaluationStatus = 'pending' | 'running' | 'completed' | 'failed' | 'canceled';

export interface DatasetConfig {
  dataset_id: string;
}

export interface EvaluationApi {
  id: string;
  name: string;
  description: string;
  eval_type: string; // 'model' | 'agent' | 'rag' — confirm exact values with backend
  dataset_id: string;
  datasets_config: DatasetConfig[];
  benchmark: string;
  model_ids: string[];
  selected_metrics: string[];
  run_samples: number;
  selected_category: string[];
  status: EvaluationStatus;
  progress: number;
  total_questions: number;
  top_model: string | null;
  top_score: number | null;
  created_at: string;
  started_at: string | null;
  completed_at: string | null;
}

export interface EvaluationsListResponse {
  evaluations: EvaluationApi[];
}

/* ---------- API: GET /evaluations/{id}/results ---------- */

export interface ResultDetail {
  test_id: string;
  input: string;
  output: string;
  expected: string;
  latency_seconds: number;
  passed: boolean;
  score: number;
  metric_scores: Record<string, number>;
}

export interface ModelResult {
  model_id: string;
  provider: string;
  rank: number;
  score: number;
  accuracy: number;
  passed_tests: number;
  failed_tests: number;
  total_tests: number;
  metric_scores: Record<string, number>;
  details: ResultDetail[];
}

export interface EvaluationResultsResponse {
  evaluation_id: string;
  status: EvaluationStatus;
  top_model: string;
  top_score: number;
  results: ModelResult[];
}

// Shape of the 400 { detail: "Execution not completed." } error body
export interface ApiErrorBody {
  detail: string;
}
















//api.ts
import api from '../../../services/api';
import type { ApiErrorBody, EvaluationResultsResponse, EvaluationsListResponse } from './types';

export async function fetchEvaluations(): Promise<EvaluationsListResponse> {
  const res = await api.get<EvaluationsListResponse>('/evaluations');
  return res.data;
}

// Thrown when the run hasn't finished yet (backend returns 400 with a `detail` message
// rather than a results payload). Callers can check `notCompleted` to show a friendlier
// message instead of a generic error.
export class EvaluationNotCompletedError extends Error {
  notCompleted = true;
}

export async function fetchEvaluationResults(evaluationId: string): Promise<EvaluationResultsResponse> {
  try {
    const res = await api.get<EvaluationResultsResponse>(`/evaluations/${evaluationId}/results`);
    return res.data;
  } catch (err: unknown) {
    // Adjust this check to however your `api` client (axios?) surfaces status + body.
    const status = (err as { response?: { status?: number } })?.response?.status;
    const body = (err as { response?: { data?: ApiErrorBody } })?.response?.data;
    if (status === 400) {
      throw new EvaluationNotCompletedError(body?.detail ?? 'Execution not completed.');
    }
    throw err;
  }
}















//History.tsx
import { useEffect, useMemo, useState, type FC, type MouseEvent } from 'react';
import { useNavigate } from 'react-router-dom';
import { Play, Search, Copy, Trash2, X, Database, FileBarChart, AlertCircle, RefreshCw, Clock3 } from 'lucide-react';
import { fetchEvaluations, fetchEvaluationResults, EvaluationNotCompletedError } from './historyApi';
import type { EvaluationApi, ModelResult } from './types';
import Spinner from '../../../components/Spinner/Spinner';
import Select from './Select';
import './History.scss';

const TYPE_FILTERS = [
  { value: 'all', label: 'All Types' },
  { value: 'AI Model', label: 'AI Model' },
  { value: 'Agent', label: 'Agent' },
  { value: 'RAG', label: 'RAG' },
];

const DATE_FILTERS = [
  { value: 30, label: 'Last 30 days' },
  { value: 7, label: 'Last 7 days' },
  { value: Infinity, label: 'All time' },
];

// Maps the backend's `eval_type` values to the display labels used by the filters/badges.
// Confirm these keys match what the API actually sends.
const TYPE_LABELS: Record<string, string> = {
  model: 'AI Model',
  agent: 'Agent',
  rag: 'RAG',
};

const STATUS_LABELS: Record<EvaluationApi['status'], string> = {
  pending: 'Pending',
  running: 'Running',
  completed: 'Completed',
  failed: 'Failed',
  canceled: 'Canceled',
};

function typeTint(typeLabel: string): 'violet' | 'blue' | 'amber' {
  if (typeLabel.includes('Agent')) return 'violet';
  if (typeLabel.includes('RAG')) return 'blue';
  return 'amber';
}

function matchesType(typeLabel: string, filter: string) {
  if (filter === 'all') return true;
  return typeLabel === filter;
}

function daysAgo(dateStr: string | null): number {
  if (!dateStr) return Infinity;
  const diffMs = Date.now() - new Date(dateStr).getTime();
  return Math.floor(diffMs / (1000 * 60 * 60 * 24));
}

function formatDate(dateStr: string | null): string {
  if (!dateStr) return '—';
  return new Date(dateStr).toLocaleDateString(undefined, { year: 'numeric', month: 'short', day: 'numeric' });
}

// Assumes fractional scores (0–1), matching the 0.95 / 0.345 examples in the API docs.
// Flip this to a plain toFixed if the backend ever sends 0–100 instead.
function formatScore(score: number | null | undefined): string {
  if (score === null || score === undefined || Number.isNaN(score)) return '—';
  return `${(score * 100).toFixed(1)}%`;
}

const History: FC = () => {
  const navigate = useNavigate();
  const [items, setItems] = useState<EvaluationApi[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const [query, setQuery] = useState('');
  const [typeFilter, setTypeFilter] = useState('all');
  const [dateFilter, setDateFilter] = useState(30);
  const [selectedId, setSelectedId] = useState<string | null>(null);

  // Results for whichever evaluation is currently selected — fetched lazily per-selection
  // rather than for every row up front.
  const [results, setResults] = useState<ModelResult[] | null>(null);
  const [resultsLoading, setResultsLoading] = useState(false);
  const [resultsError, setResultsError] = useState<string | null>(null);
  const [resultsNotCompleted, setResultsNotCompleted] = useState(false);

  const load = () => {
    setLoading(true);
    setError(null);
    fetchEvaluations()
      .then((res) => {
        setItems(res.evaluations);
        setSelectedId((prev) => prev ?? res.evaluations[0]?.id ?? null);
      })
      .catch((err) => {
        setError(err instanceof Error ? err.message : 'Failed to load evaluation history.');
      })
      .finally(() => setLoading(false));
  };

  useEffect(() => {
    load();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  const filtered = useMemo(() => {
    return items.filter((ev) => {
      const typeLabel = TYPE_LABELS[ev.eval_type] ?? ev.eval_type;
      if (query && !ev.name.toLowerCase().includes(query.toLowerCase())) return false;
      if (!matchesType(typeLabel, typeFilter)) return false;
      if (daysAgo(ev.created_at) > dateFilter) return false;
      return true;
    });
  }, [items, query, typeFilter, dateFilter]);

  const selected = useMemo(
    () => filtered.find((ev) => ev.id === selectedId) ?? filtered[0] ?? null,
    [filtered, selectedId]
  );

  // Fetch the results table whenever the selected evaluation changes.
  useEffect(() => {
    if (!selected) {
      setResults(null);
      return;
    }

    // No point calling the results endpoint for a run that hasn't finished —
    // the API will just 400 anyway.
    if (selected.status !== 'completed') {
      setResults(null);
      setResultsError(null);
      setResultsNotCompleted(selected.status !== 'failed' && selected.status !== 'canceled');
      return;
    }

    let cancelled = false;
    setResultsLoading(true);
    setResultsError(null);
    setResultsNotCompleted(false);

    fetchEvaluationResults(selected.id)
      .then((res) => {
        if (cancelled) return;
        setResults(res.results);
      })
      .catch((err) => {
        if (cancelled) return;
        if (err instanceof EvaluationNotCompletedError) {
          setResultsNotCompleted(true);
          setResults(null);
        } else {
          setResultsError(err instanceof Error ? err.message : 'Failed to load results.');
          setResults(null);
        }
      })
      .finally(() => {
        if (!cancelled) setResultsLoading(false);
      });

    return () => {
      cancelled = true;
    };
  }, [selected]);

  const handleDuplicate = (e: MouseEvent, _id: string) => {
    e.stopPropagation();
    navigate('/app/run-evaluation');
  };

  const handleDelete = (e: MouseEvent, id: string) => {
    e.stopPropagation();
    // NOTE: this only removes the row client-side. Wire up a DELETE /evaluations/{id}
    // call here once that endpoint exists, so the removal persists on refresh.
    setItems((prev) => prev.filter((ev) => ev.id !== id));
    if (selectedId === id) setSelectedId(null);
  };

  return (
    <div className="history">
      <div className="history__header">
        <div className="history__header-left">
          <p className="history__header-eyebrow">Evaluation records</p>
          <h1 className="history__title">History</h1>
          <p className="history__subtitle">Past evaluations</p>
        </div>

        <div className="history__header-meta">
          <Database size={13} />
          {items.length} evaluations logged
        </div>
      </div>

      <div className="history__filters">
        <div className="history__search">
          <Search size={15} />
          <input type="text" placeholder="Search evaluations..." value={query} onChange={(e) => setQuery(e.target.value)} />
          {query && (
            <button type="button" className="history__search-clear" onClick={() => setQuery('')} aria-label="Clear search">
              <X size={13} />
            </button>
          )}
        </div>
        <Select value={typeFilter} options={TYPE_FILTERS} onChange={setTypeFilter} width={150} />
        <Select value={dateFilter} options={DATE_FILTERS} onChange={setDateFilter} width={160} />

        <button type="button" className="history__btn history__btn--primary history__btn--push" onClick={() => navigate('/app/run-evaluation')}>
          <Play size={14} strokeWidth={2.25} /> New Evaluation
        </button>
      </div>

      {loading && (
        <div className="history__empty">
          <Spinner label="Loading evaluations…" />
        </div>
      )}

      {!loading && error && (
        <div className="history__empty history__empty--error">
          <AlertCircle size={22} />
          <p>{error}</p>
          <button type="button" className="history__btn history__btn--outline" onClick={load}>
            <RefreshCw size={14} strokeWidth={2.25} /> Try again
          </button>
        </div>
      )}

      {!loading && !error && filtered.length === 0 && (
        <div className="history__empty">
          <Search size={22} />
          <p>No evaluations match your filters.</p>
        </div>
      )}

      {!loading && !error && filtered.length > 0 && (
        <div className="history__layout">
          <div className="history__sidebar">
            <div className="history__sidebar-list">
              {filtered.map((ev) => {
                const typeLabel = TYPE_LABELS[ev.eval_type] ?? ev.eval_type;
                const tint = typeTint(typeLabel);
                const isActive = selected?.id === ev.id;
                return (
                  <button
                    type="button"
                    key={ev.id}
                    className={`history__item${isActive ? ' history__item--active' : ''}`}
                    onClick={() => setSelectedId(ev.id)}
                  >
                    <div className="history__item-top">
                      <span className="history__item-name">{ev.name}</span>
                      <span className={`history__type-badge history__type-badge--${tint}`}>{typeLabel}</span>
                    </div>
                    <div className="history__item-meta">
                      <span>{formatDate(ev.created_at)}</span>
                      <span className="history__item-score n">{formatScore(ev.top_score)}</span>
                    </div>
                  </button>
                );
              })}
            </div>
          </div>

          {selected && (
            <div className="history__detail">
              <div className="history__detail-head">
                <div className="history__detail-head-left">
                  <span className={`history__type-badge history__type-badge--${typeTint(TYPE_LABELS[selected.eval_type] ?? selected.eval_type)}`}>
                    {TYPE_LABELS[selected.eval_type] ?? selected.eval_type}
                  </span>
                  <h2 className="history__detail-name">{selected.name}</h2>
                  <span className="history__detail-date">
                    {STATUS_LABELS[selected.status] ?? selected.status} &middot; {formatDate(selected.created_at)}
                  </span>
                </div>

                <div className="history__detail-actions">
                  <button type="button" className="history__btn" onClick={(e) => handleDuplicate(e, selected.id)}>
                    <Copy size={13} /> Duplicate
                  </button>
                  <button
                    type="button"
                    className="history__btn history__btn--danger"
                    onClick={(e) => handleDelete(e, selected.id)}
                  >
                    <Trash2 size={13} /> Delete
                  </button>
                  <button type="button" className="history__btn history__btn--primary" onClick={() => navigate('/app/reports')}>
                    <FileBarChart size={13} /> View Report
                  </button>
                </div>
              </div>

              <div className="history__stat-row">
                <div className="history__stat-card">
                  <span className="history__stat-card-label">Models Tested</span>
                  <span className="history__stat-card-value n">{selected.model_ids.length}</span>
                </div>
                <div className="history__stat-card">
                  <span className="history__stat-card-label">Top Model</span>
                  <span className="history__stat-card-value history__stat-card-value--sm">{selected.top_model ?? '—'}</span>
                </div>
                <div className="history__stat-card">
                  <span className="history__stat-card-label">Top Score</span>
                  <span className="history__stat-card-value history__stat-card-value--accent n">{formatScore(selected.top_score)}</span>
                </div>
                <div className="history__stat-card">
                  <span className="history__stat-card-label">Status</span>
                  <span className="history__stat-card-value history__stat-card-value--sm">
                    {STATUS_LABELS[selected.status] ?? selected.status}
                  </span>
                </div>
              </div>

              <p className="history__section-title">Full results</p>

              {resultsLoading && (
                <div className="history__empty">
                  <Spinner label="Loading results…" />
                </div>
              )}

              {!resultsLoading && resultsNotCompleted && (
                <div className="history__empty">
                  <Clock3 size={20} />
                  <p>This evaluation hasn't finished running yet — results will appear here once it completes.</p>
                </div>
              )}

              {!resultsLoading && resultsError && (
                <div className="history__empty history__empty--error">
                  <AlertCircle size={20} />
                  <p>{resultsError}</p>
                </div>
              )}

              {!resultsLoading && !resultsError && !resultsNotCompleted && results && results.length === 0 && (
                <div className="history__empty">
                  <FileBarChart size={20} />
                  <p>No per-model results were returned for this run.</p>
                </div>
              )}

              {!resultsLoading && !resultsError && !resultsNotCompleted && results && results.length > 0 && (
                <div className="history__table-wrap">
                  <table className="history__table">
                    <thead>
                      <tr>
                        <th style={{ width: 56 }}>Rank</th>
                        <th>Model</th>
                        <th>Provider</th>
                        <th>Score</th>
                        <th>Accuracy</th>
                        <th>Tests Passed</th>
                      </tr>
                    </thead>
                    <tbody>
                      {results
                        .slice()
                        .sort((a, b) => a.rank - b.rank)
                        .map((r) => (
                          <tr key={r.model_id}>
                            <td>
                              <span className={`history__rank-pill${r.rank === 1 ? ' history__rank-pill--1' : ''}`}>{r.rank}</span>
                            </td>
                            {/* Only model_id is available here, not a display name — swap in
                                a models lookup (e.g. fetchModels) if you want the friendly name. */}
                            <td className="history__cell-strong">{r.model_id}</td>
                            <td>{r.provider}</td>
                            <td className={`n${r.rank === 1 ? ' history__score-cell' : ''}`}>{formatScore(r.score)}</td>
                            <td className="n">{formatScore(r.accuracy)}</td>
                            <td className="n">
                              {r.passed_tests}/{r.total_tests}
                            </td>
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
