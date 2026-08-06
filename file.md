//History.tsx
import { useEffect, useMemo, useState, type FC, type MouseEvent } from 'react';
import { useNavigate } from 'react-router-dom';
import { Play, Search, Copy, Trash2, X, Database, FileBarChart } from 'lucide-react';
import { fetchEvaluations, fetchEvaluationResults, EvaluationNotCompletedError } from './historyApi';
import type { EvaluationApi, ModelResult } from './types';
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

function typeTint(type: string): 'violet' | 'blue' | 'amber' {
  if (type.includes('Agent')) return 'violet';
  if (type.includes('RAG')) return 'blue';
  return 'amber';
}

function matchesType(evType: string, filter: string) {
  if (filter === 'all') return true;
  if (filter === 'Agent') return evType.includes('Agent');
  if (filter === 'RAG') return evType.includes('RAG');
  return evType.includes('AI Model');
}

function daysAgo(dateStr: string | null): number {
  if (!dateStr) return Infinity;
  return Math.floor((Date.now() - new Date(dateStr).getTime()) / (1000 * 60 * 60 * 24));
}

function formatDate(dateStr: string | null): string {
  if (!dateStr) return '—';
  return new Date(dateStr).toLocaleDateString();
}

// Assumes fractional scores (0–1), matching the 0.95 / 0.345 examples in the API docs.
function formatScore(score: number | null | undefined): string {
  if (score === null || score === undefined || Number.isNaN(score)) return '—';
  return `${(score * 100).toFixed(1)}%`;
}

const History: FC = () => {
  const navigate = useNavigate();
  const [items, setItems] = useState<EvaluationApi[]>([]);
  const [query, setQuery] = useState('');
  const [typeFilter, setTypeFilter] = useState('all');
  const [dateFilter, setDateFilter] = useState(30);
  const [selectedId, setSelectedId] = useState<string | null>(null);

  // Per-model results table for whichever evaluation is currently selected.
  const [results, setResults] = useState<ModelResult[]>([]);

  useEffect(() => {
    fetchEvaluations().then((res) => {
      setItems(res.evaluations);
      setSelectedId(res.evaluations[0]?.id ?? null);
    });
  }, []);

  const filtered = useMemo(() => {
    return items.filter((ev) => {
      const type = TYPE_LABELS[ev.eval_type] ?? ev.eval_type;
      if (query && !ev.name.toLowerCase().includes(query.toLowerCase())) return false;
      if (!matchesType(type, typeFilter)) return false;
      if (daysAgo(ev.created_at) > dateFilter) return false;
      return true;
    });
  }, [items, query, typeFilter, dateFilter]);

  const selected = useMemo(
    () => filtered.find((ev) => ev.id === selectedId) ?? filtered[0] ?? null,
    [filtered, selectedId]
  );

  useEffect(() => {
    if (!selected || selected.status !== 'completed') {
      setResults([]);
      return;
    }
    fetchEvaluationResults(selected.id)
      .then((res) => setResults(res.results))
      .catch((err) => {
        if (!(err instanceof EvaluationNotCompletedError)) console.error(err);
        setResults([]);
      });
  }, [selected]);

  const handleDuplicate = (e: MouseEvent, _id: string) => {
    e.stopPropagation();
    navigate('/app/run-evaluation');
  };

  const handleDelete = (e: MouseEvent, id: string) => {
    e.stopPropagation();
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

      {filtered.length === 0 ? (
        <div className="history__empty">
          <Search size={22} />
          <p>No evaluations match your filters.</p>
        </div>
      ) : (
        <div className="history__layout">
          <div className="history__sidebar">
            <div className="history__sidebar-list">
              {filtered.map((ev) => {
                const type = TYPE_LABELS[ev.eval_type] ?? ev.eval_type;
                const tint = typeTint(type);
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
                      <span className={`history__type-badge history__type-badge--${tint}`}>
                        {type.split('(')[0].trim()}
                      </span>
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
                    {(TYPE_LABELS[selected.eval_type] ?? selected.eval_type).split('(')[0].trim()}
                  </span>
                  <h2 className="history__detail-name">{selected.name}</h2>
                  <span className="history__detail-date">
                    {selected.status === 'running' ? 'Running' : 'Completed'} &middot; {formatDate(selected.created_at)}
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
                  <span className="history__stat-card-value history__stat-card-value--sm">{selected.status}</span>
                </div>
              </div>

              <p className="history__section-title">Full results</p>
              <div className="history__table-wrap">
                <table className="history__table">
                  <thead>
                    <tr>
                      <th style={{ width: 56 }}>Rank</th>
                      <th>Model</th>
                      <th>Provider</th>
                      <th>Score</th>
                      <th>Accuracy</th>
                      <th>Speed</th>
                      <th>Cost</th>
                    </tr>
                  </thead>
                  <tbody>
                    {results.map((r) => (
                      <tr key={r.model_id}>
                        <td>
                          <span className={`history__rank-pill${r.rank === 1 ? ' history__rank-pill--1' : ''}`}>{r.rank}</span>
                        </td>
                        {/* API only returns model_id, not a display name — swap in a
                            models lookup (fetchModels) here if you want the friendly name. */}
                        <td className="history__cell-strong">{r.model_id}</td>
                        <td>{r.provider}</td>
                        <td className={`n${r.rank === 1 ? ' history__score-cell' : ''}`}>{formatScore(r.score)}</td>
                        <td className="n">{formatScore(r.accuracy)}</td>
                        {/* Not returned by GET /evaluations/{id}/results */}
                        <td className="n">—</td>
                        <td className="n">—</td>
                      </tr>
                    ))}
                  </tbody>
                </table>
              </div>
            </div>
          )}
        </div>
      )}
    </div>
  );
};

export default History;
