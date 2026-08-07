//types.ts
export type EvaluationStatusValue = 'pending' | 'running' | 'completed' | 'failed' | 'canceled';

/* ---------- API: GET /evaluations (history list) ---------- */

export interface EvaluationDatasetConfig {
  dataset_id: string;
}

export interface EvaluationListItem {
  id: string;
  name: string;
  description: string;
  eval_type: string;
  dataset_id: string;
  datasets_config: EvaluationDatasetConfig[];
  benchmark: string;
  model_ids: string[];
  selected_metrics: string[];
  run_samples: number;
  selected_category: string[];
  status: EvaluationStatusValue;
  progress: number;
  total_questions: number;
  top_model: string | null;
  top_score: number | null;
  created_at: string;
  started_at: string | null;
  completed_at: string | null;
}

export interface EvaluationsListResponse {
  evaluations: EvaluationListItem[];
}

/* ---------- API: GET /evaluations/{id}/results ---------- */

export interface TestDetail {
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
  details: TestDetail[];
}

export interface EvaluationResultsResponse {
  evaluation_id: string;
  status: EvaluationStatusValue;
  top_model: string;
  top_score: number;
  results: ModelResult[];
}

// Returned with HTTP 400 when the evaluation hasn't finished running yet
export interface EvaluationResultsErrorResponse {
  detail: string;
}
















//api.ts
import api from '../../../services/api';
import type { EvaluationsListResponse, EvaluationResultsResponse } from './types';

/**
 * GET /evaluations
 * Full evaluation history — used to populate the History sidebar list.
 */
export async function fetchEvaluations(): Promise<EvaluationsListResponse> {
  const res = await api.get<EvaluationsListResponse>('/evaluations');
  return res.data;
}

/**
 * GET /evaluations/{evaluation_id}/results
 * Per-model results + per-test detail for a completed evaluation.
 * Returns HTTP 400 with { detail: "Execution not completed." } if the
 * evaluation hasn't finished running yet — callers should catch and
 * inspect err.response.data.detail for that case.
 */
export async function fetchEvaluationResults(evaluationId: string): Promise<EvaluationResultsResponse> {
  const res = await api.get<EvaluationResultsResponse>(`/evaluations/${evaluationId}/results`);
  return res.data;
}
















//History.tsx
import { useEffect, useMemo, useState, type FC, type MouseEvent } from 'react';
import { useNavigate } from 'react-router-dom';
import { Play, Search, Copy, Trash2, X, Database, FileBarChart, AlertCircle, RefreshCw } from 'lucide-react';
import { fetchEvaluations, fetchEvaluationResults } from './api';
import type { EvaluationListItem, EvaluationResultsResponse, EvaluationResultsErrorResponse } from './types';
import Spinner from '../../../components/Spinner/Spinner';
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

function daysAgo(dateStr: string): number {
  const then = new Date(dateStr).getTime();
  if (Number.isNaN(then)) return Infinity;
  return Math.floor((Date.now() - then) / (1000 * 60 * 60 * 24));
}

function formatDate(dateStr: string): string {
  const d = new Date(dateStr);
  if (Number.isNaN(d.getTime())) return '—';
  return d.toLocaleDateString(undefined, { year: 'numeric', month: 'short', day: 'numeric' });
}

interface ApiErrorLike {
  response?: {
    data?: EvaluationResultsErrorResponse;
  };
}

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

  useEffect(() => {
    loadList();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  const filtered = useMemo(() => {
    return items.filter((ev) => {
      if (query && !ev.name.toLowerCase().includes(query.toLowerCase())) return false;
      if (typeFilter !== 'all' && ev.eval_type !== typeFilter) return false;
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
  }, [selected]);

  const handleDuplicate = (e: MouseEvent, _id: string) => {
    e.stopPropagation();
    navigate('/app/run-evaluation');
  };

  const handleDelete = (e: MouseEvent, id: string) => {
    e.stopPropagation();
    // NOTE: no DELETE /evaluations/{id} endpoint was provided — this only
    // removes the row from local state. Wire in a real delete call here
    // once that endpoint exists.
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

        <button
          type="button"
          className="history__btn history__btn--primary history__btn--push"
          onClick={() => navigate('/app/run-evaluation')}
        >
          <Play size={14} strokeWidth={2.25} /> New Evaluation
        </button>
      </div>

      {listLoading && (
        <div className="history__empty">
          <Spinner label="Loading history…" />
        </div>
      )}

      {!listLoading && listError && (
        <div className="history__empty history__empty--error">
          <AlertCircle size={22} />
          <p>{listError}</p>
          <button type="button" className="history__btn history__btn--outline" onClick={loadList}>
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
        <div className="history__layout">
          <div className="history__sidebar">
            <div className="history__sidebar-list">
              {filtered.map((ev) => {
                const tint = typeTint(ev.eval_type);
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
                      <span className={`history__type-badge history__type-badge--${tint}`}>{typeLabel(ev.eval_type)}</span>
                    </div>
                    <div className="history__item-meta">
                      <span>{formatDate(ev.created_at)}</span>
                      <span className="history__item-score n">{ev.top_score != null ? ev.top_score.toFixed(3) : '—'}</span>
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
                  <span className={`history__type-badge history__type-badge--${typeTint(selected.eval_type)}`}>
                    {typeLabel(selected.eval_type)}
                  </span>
                  <h2 className="history__detail-name">{selected.name}</h2>
                  <span className="history__detail-date">
                    {statusLabel(selected.status)} &middot; {formatDate(selected.created_at)}
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
                  <span className="history__stat-card-value history__stat-card-value--accent n">
                    {selected.top_score != null ? selected.top_score.toFixed(3) : '—'}
                  </span>
                </div>
                <div className="history__stat-card">
                  <span className="history__stat-card-label">Status</span>
                  <span className="history__stat-card-value history__stat-card-value--sm">{statusLabel(selected.status)}</span>
                </div>
              </div>

              <p className="history__section-title">Full results</p>

              {selected.status !== 'completed' && (
                <div className="history__empty history__empty--inline">
                  <AlertCircle size={18} />
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
                <div className="history__empty history__empty--inline">
                  <Spinner label="Loading results…" />
                </div>
              )}

              {selected.status === 'completed' && !resultsLoading && resultsError && (
                <div className="history__empty history__empty--inline history__empty--error">
                  <AlertCircle size={18} />
                  <p>{resultsError}</p>
                </div>
              )}

              {selected.status === 'completed' && !resultsLoading && !resultsError && results && (
                <div className="history__table-wrap">
                  <table className="history__table">
                    <thead>
                      <tr>
                        <th style={{ width: 56 }}>Rank</th>
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
