import { useEffect, useMemo, useState } from 'react';
import { Layers, Check, Play, GitCompare, Sparkles, FlaskConical } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchModels } from '../../store/slices/modelsSlice';
import { fetchEvaluations } from '../../store/slices/evaluationsSlice';
import { runComparison, resetComparison } from '../../store/slices/comparisonSlice';
import RadarChart from '../common/RadarChart';
import ScoreRing from '../common/ScoreRing';
import Dropdown from '../common/Dropdown';
import styles from './Comparison.module.scss';

const COLORS = ['#6366F1', '#F59E0B', '#10B981', '#EF4444', '#0EA5E9', '#A855F7'];

export default function Comparison() {
  const dispatch = useAppDispatch();
  const models = useAppSelector((s) => s.models.items);
  const evaluations = useAppSelector((s) => s.evaluations.list);
  const { result, status: compareStatus, error: compareError } = useAppSelector((s) => s.comparison);

  const [selBenchmark, setSelBenchmark] = useState<string | null>(null);
  const [selModelIds, setSelModelIds] = useState<string[]>([]);

  useEffect(() => {
    dispatch(fetchModels());
    dispatch(fetchEvaluations());
  }, [dispatch]);

  // Unique benchmark names (from evaluation history) to populate the dropdown.
  const benchmarkOptions = useMemo(() => {
    const seen = new Set<string>();
    return evaluations
      .filter((e) => e.benchmark && !seen.has(e.benchmark) && seen.add(e.benchmark))
      .map((e) => ({ value: e.benchmark, label: e.benchmark }));
  }, [evaluations]);

  // Evaluations sharing the selected benchmark -> resolve dataset_id + union of model_ids.
  const benchmarkEvals = useMemo(
    () => evaluations.filter((e) => e.benchmark === selBenchmark),
    [evaluations, selBenchmark]
  );

  const datasetId = benchmarkEvals[0]?.dataset_id ?? null;

  const availableModelIds = useMemo(() => {
    const ids = new Set<string>();
    benchmarkEvals.forEach((e) => e.model_ids.forEach((id) => ids.add(id)));
    return Array.from(ids);
  }, [benchmarkEvals]);

  const handleSelectBenchmark = (value: string) => {
    setSelBenchmark(value);
    setSelModelIds([]);
    dispatch(resetComparison());
  };

  const toggleModel = (id: string) => {
    setSelModelIds((prev) =>
      prev.includes(id) ? prev.filter((m) => m !== id) : [...prev, id]
    );
  };

  const canCompare = Boolean(datasetId) && selModelIds.length >= 2 && compareStatus !== 'loading';

  const handleCompare = () => {
    if (!datasetId) return;
    dispatch(runComparison({ datasetId, modelIds: selModelIds }));
  };

  const modelName = (id: string) => models.find((m) => m.id === id)?.name || id;

  // Flatten each model's `metrics` array into a lookup so the table/radar
  // can index by metric name instead of array position.
  const rows = useMemo(() => {
    if (!result) return [];
    return result.comparisons.map((c) => {
      const m: Record<string, number> = {};
      c.metrics.forEach((met) => { m[met.metric] = met.score; });
      return {
        modelId: c.model_id,
        name: modelName(c.model_id),
        provider: c.provider,
        status: c.status,
        score: m.score ?? 0,
        accuracy: m.accuracy ?? 0,
        benchmarkAccuracy: m.benchmark_accuracy ?? 0,
        passed: m.passed_tests ?? 0,
        total: m.total_tests ?? 0,
        values: [
          m.score ?? 0,
          m.accuracy ?? 0,
          m.benchmark_accuracy ?? 0,
          m.total_tests ? (m.passed_tests ?? 0) / m.total_tests : 0,
        ],
      };
    });
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [result, models]);

  return (
    <div className="page-enter pg-shell">
      <div className={styles['comparison__header']}>
        <div>
          <p className={styles['comparison__header-eyebrow']}>Analysis</p>
          <h1>Model Comparison</h1>
          <p className={styles['comparison__header-sub']}>Compare models head-to-head on a shared benchmark</p>
        </div>
        <div className={styles['comparison__header-meta']}>
          <Layers size={13} />
          {rows.length} model{rows.length === 1 ? '' : 's'} compared
        </div>
      </div>

      <div className="pg-body">
        <div className={styles['comparison__controls']}>
          <span className={styles['comparison__label']}>Benchmark:</span>
          <Dropdown
            value={selBenchmark ?? ''}
            onChange={handleSelectBenchmark}
            width={240}
            options={benchmarkOptions}
            placeholder="Select a benchmark…"
          />
        </div>

        {!selBenchmark && (
          <div className={styles['empty-state']}>
            <div className={styles['empty-state__icon']}>
              <GitCompare size={28} />
            </div>
            <h3>Pick a benchmark to get started</h3>
            <p>
              Choose a benchmark above and select two or more models that were evaluated
              against it to see a side-by-side breakdown of scores, accuracy, and pass rates.
            </p>
            <div className={styles['empty-state__stats']}>
              <div className={styles['empty-state__stat']}>
                <FlaskConical size={16} />
                <span><strong>{benchmarkOptions.length}</strong> benchmark{benchmarkOptions.length === 1 ? '' : 's'} available</span>
              </div>
              <div className={styles['empty-state__stat']}>
                <Sparkles size={16} />
                <span><strong>{models.length}</strong> model{models.length === 1 ? '' : 's'} in catalog</span>
              </div>
            </div>
          </div>
        )}

        {selBenchmark && (
          <div className="card">
            <div className={styles['comparison__panel-title']}>Select models</div>
            <div className={styles['comparison__panel-sub']}>
              Models evaluated against {selBenchmark}
            </div>
            <div className={styles['model-select-grid']}>
              {availableModelIds.map((id) => {
                const active = selModelIds.includes(id);
                const colorIdx = selModelIds.indexOf(id);
                return (
                  <button
                    key={id}
                    type="button"
                    className={`${styles['model-select-item']} ${active ? styles.active : ''}`}
                    onClick={() => toggleModel(id)}
                    style={active ? { borderColor: COLORS[colorIdx % COLORS.length] } : undefined}
                  >
                    <span className={styles['model-select-item__check']}>
                      {active && <Check size={12} />}
                    </span>
                    {modelName(id)}
                  </button>
                );
              })}
              {availableModelIds.length === 0 && (
                <div className={styles.empty}>No models found for this benchmark.</div>
              )}
            </div>
            <button
              type="button"
              className="btn btn-ind"
              disabled={!canCompare}
              onClick={handleCompare}
              style={{ marginTop: 16 }}
            >
              <Play size={14} /> Compare {selModelIds.length > 0 ? `(${selModelIds.length})` : ''}
            </button>
            {selModelIds.length === 1 && (
              <div className={styles['comparison__hint']}>Select at least 2 models to compare.</div>
            )}
          </div>
        )}

        {compareStatus === 'loading' && <ComparisonSkeleton />}

        {compareStatus === 'failed' && (
          <div className={`card ${styles.empty}`}>{compareError || 'Comparison failed.'}</div>
        )}

        {compareStatus === 'succeeded' && result && rows.length > 0 && (
          <>
            <div className={styles['comparison__controls']}>
              <span className={styles['comparison__label']}>Comparing:</span>
              {rows.map((r, i) => (
                <span
                  key={r.modelId}
                  className={styles['model-chip']}
                  style={{ borderColor: COLORS[i % COLORS.length], color: COLORS[i % COLORS.length], background: `${COLORS[i % COLORS.length]}14` }}
                >
                  <span className={styles['model-chip__dot']} style={{ background: COLORS[i % COLORS.length] }} /> {r.name}
                </span>
              ))}
            </div>

            <div className={styles['comparison__grid']}>
              <div className="card">
                <div className={styles['comparison__panel-title']}>Strength Profile</div>
                <div className={styles['comparison__panel-sub']}>Score · Accuracy · Benchmark accuracy · Pass rate</div>
                <div className="radar-wrap">
                  <RadarChart models={rows} size={280} colors={COLORS} />
                </div>
                <div className={styles['comparison__legend']}>
                  {rows.map((r, i) => (
                    <span key={r.modelId}><span className={styles['comparison__dot']} style={{ background: COLORS[i % COLORS.length] }} /> {r.name}</span>
                  ))}
                </div>
              </div>

              <div className="card" style={{ padding: 0 }}>
                <div className={styles['comparison__panel-title']} style={{ padding: '20px 24px', borderBottom: '1px solid var(--border-light)' }}>
                  {result.dataset_name} — Metric Breakdown
                </div>
                <table className="tbl">
                  <thead>
                    <tr>
                      <th>Model</th>
                      <th>Provider</th>
                      <th>Score</th>
                      <th>Accuracy</th>
                      <th>Benchmark Acc.</th>
                      <th>Passed</th>
                      <th>Failed</th>
                    </tr>
                  </thead>
                  <tbody>
                    {rows.map((r, i) => (
                      <tr key={r.modelId}>
                        <td style={{ fontWeight: 700, color: COLORS[i % COLORS.length] }}>{r.name}</td>
                        <td style={{ color: 'var(--text-secondary)' }}>{r.provider || '—'}</td>
                        <td style={{ fontFamily: "'JetBrains Mono',monospace", fontWeight: 700 }}>{(r.score * 100).toFixed(1)}%</td>
                        <td style={{ fontFamily: "'JetBrains Mono',monospace" }}>{(r.accuracy * 100).toFixed(1)}%</td>
                        <td style={{ fontFamily: "'JetBrains Mono',monospace" }}>{(r.benchmarkAccuracy * 100).toFixed(1)}%</td>
                        <td style={{ color: '#10B981', fontWeight: 700 }}>{r.passed}</td>
                        <td style={{ color: '#EF4444', fontWeight: 700 }}>{r.total - r.passed}</td>
                      </tr>
                    ))}
                  </tbody>
                </table>
              </div>
            </div>

            <div className="card">
              <div className={styles['comparison__panel-title']} style={{ marginBottom: 20 }}>Score Comparison</div>
              <div className={styles['comparison__scores']}>
                {rows.map((r, i) => (
                  <div key={r.modelId} className={styles['comparison__score-item']}>
                    <ScoreRing score={Math.round(r.score * 100)} size={100} stroke={7} color={COLORS[i % COLORS.length]} label="SCORE" />
                    <div style={{ fontWeight: 700, fontSize: 14, textAlign: 'center' }}>{r.name}</div>
                    <div style={{ fontSize: 12, color: 'var(--text-secondary)' }}>{r.passed}/{r.total} passed</div>
                  </div>
                ))}
              </div>
            </div>
          </>
        )}
      </div>
    </div>
  );
}

function ComparisonSkeleton() {
  return (
    <div className={styles['comparison__grid']}>
      <div className="card">
        <div className={styles.skeletonLine} style={{ width: '40%', height: 16, marginBottom: 20 }} />
        <div className={styles.skeletonCircle} />
      </div>
      <div className="card">
        <div className={styles.skeletonLine} style={{ width: '100%', height: 40, marginBottom: 12 }} />
        {[...Array(3)].map((_, i) => (
          <div key={i} className={styles.skeletonLine} style={{ width: '100%', height: 32, marginBottom: 8 }} />
        ))}
      </div>
    </div>
  );
}




















@use '../../styles/_variables' as *;

.comparison {
  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding: 24px 32px 18px;
    margin-bottom: 24px;
    border-bottom: 1px solid $border-light;

    h1 {
      font-family: $font-display;
      font-size: 1.5rem;
      font-weight: 800;
      letter-spacing: -0.02em;
      color: $text-primary;
      line-height: 1.2;
    }
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
    color: $indigo;
    margin-bottom: 6px;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $indigo;
    }
  }

  &__header-sub {
    margin-top: 4px;
    font-size: 0.875rem;
    color: $text-secondary;
  }

  &__header-meta {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-secondary;
    background: $surface-alt;
    border: 1px solid $border-light;
    border-radius: 999px;
    padding: 7px 13px;
    white-space: nowrap;
    margin-bottom: 3px;
  }

  &__controls {
    display: flex;
    align-items: center;
    gap: 10px;
    flex-wrap: wrap;
    margin-top: 20px;
    margin-bottom: 20px;
  }

  &__label {
    font-size: 12px;
    font-weight: 700;
    color: $text-secondary;
    text-transform: uppercase;
    letter-spacing: .04em;
  }

  &__grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 20px;
    margin-bottom: 20px;
  }

  &__panel-title {
    font-size: 14px;
    font-weight: 700;
  }

  &__panel-sub {
    font-size: 12px;
    color: $text-secondary;
    margin-top: 2px;
    margin-bottom: 16px;
  }

  &__legend {
    display: flex;
    gap: 14px;
    justify-content: center;
    margin-top: 12px;
    font-size: 12px;
    font-weight: 600;
    color: $text-secondary;
  }

  &__dot {
    display: inline-block;
    width: 8px;
    height: 8px;
    border-radius: 50%;
    margin-right: 5px;
  }

  &__scores {
    display: flex;
    gap: 32px;
    flex-wrap: wrap;
    justify-content: center;
  }

  &__hint {
    margin-top: 8px;
    font-size: 12px;
    color: $text-muted;
  }
}

.model-chip {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  font-size: 12px;
  font-weight: 600;
  border: 1px solid;
  border-radius: 999px;
  padding: 5px 10px;

  &__dot {
    width: 7px;
    height: 7px;
    border-radius: 50%;
    display: inline-block;
  }
}

.model-select-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(160px, 1fr));
  gap: 8px;
  margin-top: 14px;
}

.model-select-item {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 12px;
  border: 1px solid $border-light;
  border-radius: 10px;
  background: $surface-alt;
  font-size: 13px;
  font-weight: 600;
  color: $text-primary;
  cursor: pointer;
  text-align: left;
  transition: all .15s;

  &:hover { border-color: $indigo-light; }

  &.active {
    background: $surface;
    box-shadow: 0 1px 3px rgba(20, 40, 160, .12);
  }

  &__check {
    width: 16px;
    height: 16px;
    border-radius: 4px;
    border: 1.5px solid $border;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-shrink: 0;
    color: inherit;
  }
}

.empty {
  padding: 24px;
  text-align: center;
  color: $text-secondary;
  font-size: 13px;
}

// ---------------------------------------------------------------------------
// Empty state shown before a benchmark is selected — replaces the blank
// page with something inviting instead of a dropdown floating over nothing.
// ---------------------------------------------------------------------------
.empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  padding: 56px 32px;
  border: 1px dashed $border;
  border-radius: 16px;
  background: $surface-alt;

  &__icon {
    width: 56px;
    height: 56px;
    border-radius: 16px;
    background: $indigo-pale;
    color: $indigo;
    display: flex;
    align-items: center;
    justify-content: center;
    margin-bottom: 18px;
  }

  h3 {
    font-size: 16px;
    font-weight: 700;
    color: $text-primary;
    margin-bottom: 8px;
  }

  p {
    max-width: 420px;
    font-size: 13px;
    line-height: 1.6;
    color: $text-secondary;
    margin-bottom: 24px;
  }

  &__stats {
    display: flex;
    gap: 12px;
    flex-wrap: wrap;
    justify-content: center;
  }

  &__stat {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    font-size: 12.5px;
    color: $text-secondary;
    background: $surface;
    border: 1px solid $border-light;
    border-radius: 999px;
    padding: 8px 14px;

    svg { color: $indigo; flex-shrink: 0; }

    strong {
      color: $text-primary;
      font-weight: 700;
    }
  }
}

// ---------------------------------------------------------------------------
// Skeleton loader shown while POST /datasets/{id}/compare is in flight.
// ---------------------------------------------------------------------------
@keyframes skeleton-pulse {
  0%, 100% { opacity: .55; }
  50% { opacity: 1; }
}

.skeletonLine,
.skeletonCircle {
  background: $surface-alt;
  border-radius: 8px;
  animation: skeleton-pulse 1.3s ease-in-out infinite;
}

.skeletonCircle {
  width: 100px;
  height: 100px;
  border-radius: 50%;
  margin: 0 auto;
}

@media (max-width: 900px) {
  .comparison__grid {
    grid-template-columns: 1fr;
  }
}
