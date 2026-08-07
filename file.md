import { useEffect, useMemo, useState, type FC } from 'react';
import { Database, Search, X, LayoutGrid, Tag, RefreshCw, AlertCircle, ExternalLink, ListChecks } from 'lucide-react';
import { fetchBenchmarks } from './api';
import type { Benchmark } from './types';
import Spinner from '../../../components/Spinner/Spinner';
import './Datasets.scss';

const CAPABILITY_TINTS = ['blue', 'violet', 'amber', 'jade', 'rose'] as const;

function capabilityTint(capability: string) {
  let hash = 0;
  for (let i = 0; i < capability.length; i += 1) hash = (hash * 31 + capability.charCodeAt(i)) >>> 0;
  return CAPABILITY_TINTS[hash % CAPABILITY_TINTS.length];
}

const Datasets: FC = () => {
  const [benchmarks, setBenchmarks] = useState<Benchmark[]>([]);
  const [total, setTotal] = useState(0);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const [query, setQuery] = useState('');
  const [type, setType] = useState('All');
  const [tasksModalFor, setTasksModalFor] = useState<Benchmark | null>(null);

  const load = () => {
    setLoading(true);
    setError(null);
    fetchBenchmarks()
      .then((res) => {
        setBenchmarks(res.benchmarks);
        setTotal(res.total);
      })
      .catch((err) => {
        setError(err instanceof Error ? err.message : 'Failed to load benchmarks.');
      })
      .finally(() => setLoading(false));
  };

  useEffect(() => {
    load();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  const types = useMemo(() => {
    const set = new Set(benchmarks.map((b) => b.type));
    return ['All', ...Array.from(set).sort()];
  }, [benchmarks]);

  const filtered = useMemo(
    () =>
      benchmarks.filter((b) => {
        if (type !== 'All' && b.type !== type) return false;
        if (query && !b.name.toLowerCase().includes(query.toLowerCase()) && !b.description.toLowerCase().includes(query.toLowerCase())) {
          return false;
        }
        return true;
      }),
    [benchmarks, query, type]
  );

  return (
    <div className="datasets-page">
      <div className="datasets-page__header">
        <div className="datasets-page__header-left">
          <p className="datasets-page__header-eyebrow">Test suite library</p>
          <h1 className="datasets-page__title">Test Suites</h1>
          <p className="datasets-page__subtitle">Benchmark datasets and custom tests</p>
        </div>

        <div className="datasets-page__header-right">
          <div className="datasets-page__header-meta">
            <Database size={13} />
            {total} suites available
          </div>
          <button type="button" className="datasets-page__btn datasets-page__btn--outline" onClick={load} disabled={loading}>
            <RefreshCw size={14} strokeWidth={2.25} className={loading ? 'datasets-page__spin' : undefined} /> Refresh
          </button>
        </div>
      </div>

      <div className="datasets-page__filters">
        <div className="datasets-page__search">
          <Search size={15} />
          <input type="text" placeholder="Search test suites..." value={query} onChange={(e) => setQuery(e.target.value)} />
          {query && (
            <button type="button" className="datasets-page__search-clear" onClick={() => setQuery('')} aria-label="Clear search">
              <X size={13} />
            </button>
          )}
        </div>

        <div className="datasets-page__seg">
          {types.map((t) => (
            <button
              key={t}
              type="button"
              className={`datasets-page__seg-item${type === t ? ' datasets-page__seg-item--active' : ''}`}
              onClick={() => setType(t)}
            >
              {t === 'All' ? <LayoutGrid size={13} strokeWidth={2.25} /> : <Tag size={13} strokeWidth={2.25} />}
              {t}
            </button>
          ))}
        </div>
      </div>

      {loading && (
        <div className="datasets-page__loading">
          <Spinner label="Loading test suites…" />
        </div>
      )}

      {!loading && error && (
        <div className="datasets-page__empty datasets-page__empty--error">
          <AlertCircle size={22} />
          <p>{error}</p>
          <button type="button" className="datasets-page__btn datasets-page__btn--outline" onClick={load}>
            <RefreshCw size={14} strokeWidth={2.25} /> Try again
          </button>
        </div>
      )}

      {!loading && !error && filtered.length === 0 && (
        <div className="datasets-page__empty">
          <Database size={22} />
          <p>No test suites match your filters.</p>
        </div>
      )}

      {!loading && !error && filtered.length > 0 && (
        <div className="datasets-page__grid">
          {filtered.map((b) => {
            // Defensive: the real API can omit these arrays entirely for
            // some benchmarks instead of returning [], so never assume
            // they're present.
            const tasks = b.tasks ?? [];
            const requiredCapabilities = b.required_capabilities ?? [];
            return (
              <div className="datasets-page__card" key={b.name}>
                <div className="datasets-page__card-top">
                  <span className="datasets-page__card-icon">
                    <Database size={18} strokeWidth={1.5} />
                  </span>
                  <div className="datasets-page__card-heading">
                    <h4>{b.name}</h4>
                    <div className="datasets-page__card-meta">
                      <span className="datasets-page__tag">{b.type}</span>
                      <span>{b.task_count} tasks</span>
                    </div>
                  </div>
                  {tasks.length > 0 && (
                    <button type="button" className="datasets-page__card-tasks-toggle" onClick={() => setTasksModalFor(b)}>
                      <ListChecks size={13} strokeWidth={2.25} />
                      {tasks.length}
                    </button>
                  )}
                </div>

                <p className="datasets-page__card-desc">{b.description}</p>

                <div className="datasets-page__card-stats">
                  <div className="datasets-page__card-stats-row">
                    <div className="datasets-page__card-stat">
                      <span className="datasets-page__card-stat-label">Tasks</span>
                      <span className="datasets-page__card-stat-value n">{b.task_count}</span>
                    </div>
                    <div className="datasets-page__card-stat">
                      <span className="datasets-page__card-stat-label">Capabilities</span>
                      <span className="datasets-page__card-stat-value n">{requiredCapabilities.length}</span>
                    </div>
                    <div className="datasets-page__card-stat">
                      <span className="datasets-page__card-stat-label">Dataset</span>
                      <span className="datasets-page__card-stat-value datasets-page__card-stat-value--sm">{b.huggingface_dataset}</span>
                    </div>
                  </div>

                  {requiredCapabilities.length > 0 && (
                    <div className="datasets-page__caps">
                      {requiredCapabilities.map((c) => (
                        <span key={c} className={`datasets-page__cap-pill datasets-page__cap-pill--${capabilityTint(c)}`}>
                          {c}
                        </span>
                      ))}
                    </div>
                  )}
                </div>

                <div className="datasets-page__card-foot">
                  <span className="datasets-page__card-foot-source">
                    <ExternalLink size={12} /> {b.huggingface_dataset}
                  </span>
                </div>
              </div>
            );
          })}
        </div>
      )}

      {tasksModalFor && (
        <div className="datasets-page__overlay" onClick={() => setTasksModalFor(null)}>
          <div className="datasets-page__modal" onClick={(e) => e.stopPropagation()}>
            <div className="datasets-page__modal-head">
              <div>
                <span className="datasets-page__tag datasets-page__tag--blue">{tasksModalFor.type}</span>
                <h2 className="datasets-page__modal-title">{tasksModalFor.name}</h2>
                <p className="datasets-page__modal-sub">All {(tasksModalFor.tasks ?? []).length} tasks</p>
              </div>
              <button type="button" className="datasets-page__modal-close" onClick={() => setTasksModalFor(null)} aria-label="Close">
                <X size={16} />
              </button>
            </div>

            <div className="datasets-page__modal-body">
              {(tasksModalFor.tasks ?? []).map((t) => (
                <p className="datasets-page__card-task" key={t.name}>
                  <b>{t.name}:</b> <span>{t.value}</span>
                </p>
              ))}
            </div>
          </div>
        </div>
      )}
    </div>
  );
};

export default Datasets;
