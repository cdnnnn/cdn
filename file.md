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

const COLORS = ['#2B2BF5', '#E08600', '#0FA968', '#DC2626', '#0EA5E9', '#A855F7'];

export default function Comparison() {
  const dispatch = useAppDispatch();
  const models = useAppSelector((s) => s.models.items);
  const evaluations = useAppSelector((s) => s.evaluations.list);
  const { result, status: compareStatus, error: compareError } = useAppSelector((s) => s.comparison);

  const [selBenchmark, setSelBenchmark] = useState<string | null>(null);
  const [selModelIds, setSelModelIds] = useState<string[]>([]);
  const [compareErrorLocal, setCompareErrorLocal] = useState<string | null>(null);

  useEffect(() => {
    dispatch(fetchModels());
    dispatch(fetchEvaluations());
  }, [dispatch]);

  // Clear any stale comparison result/error from a previous visit to this
  // page — the redux slice otherwise persists across route changes even
  // though local selection state (selBenchmark, selModelIds) resets.
  useEffect(() => {
    return () => {
      dispatch(resetComparison());
    };
  }, [dispatch]);

  // Unique benchmark names (from evaluation history) to populate the dropdown.
  const benchmarkOptions = useMemo(() => {
    const seen = new Set<string>();
    return evaluations
      .filter((e) => e.benchmark && !seen.has(e.benchmark) && seen.add(e.benchmark))
      .map((e) => ({ value: e.benchmark, label: e.benchmark }));
  }, [evaluations]);

  // Evaluations sharing the selected benchmark -> resolve dataset_id + union of model_ids.
  // Matched case-insensitively / trimmed, since a stray whitespace or casing
  // difference between the dropdown value and the stored `benchmark` string
  // would otherwise silently leave `datasetId` null.
  const benchmarkEvals = useMemo(() => {
    if (!selBenchmark) return [];
    const target = selBenchmark.trim().toLowerCase();
    return evaluations.filter((e) => e.benchmark?.trim().toLowerCase() === target);
  }, [evaluations, selBenchmark]);

  const datasetId = benchmarkEvals.find((e) => e.dataset_id)?.dataset_id ?? null;

  const availableModelIds = useMemo(() => {
    const ids = new Set<string>();
    benchmarkEvals.forEach((e) => e.model_ids.forEach((id) => ids.add(id)));
    return Array.from(ids);
  }, [benchmarkEvals]);

  const handleSelectBenchmark = (value: string) => {
    setSelBenchmark(value);
    setSelModelIds([]);
    setCompareErrorLocal(null);
    dispatch(resetComparison());
  };

  const toggleModel = (id: string) => {
    setSelModelIds((prev) =>
      prev.includes(id) ? prev.filter((m) => m !== id) : [...prev, id]
    );
  };

  // Button enablement depends only on what the user has actually chosen —
  // model count — not on whether `datasetId` resolved cleanly. That way a
  // data-matching hiccup shows as a clear inline error on click instead of
  // an inexplicably-disabled button.
  const canCompare = selModelIds.length >= 2 && compareStatus !== 'loading';

  const handleCompare = () => {
    if (!datasetId) {
      setCompareErrorLocal('Could not find a dataset for this benchmark. Try reselecting it from the dropdown.');
      return;
    }
    setCompareErrorLocal(null);
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
          <span className={styles['comparison__label']}>Dataset:</span>
          <Dropdown
            value={selBenchmark ?? ''}
            onChange={handleSelectBenchmark}
            width={240}
            options={benchmarkOptions}
            placeholder="Select a Dataset"
          />
        </div>

        {!selBenchmark && (
          <div className={styles['empty-state']}>
            <div className={styles['empty-state__icon']}>
              <GitCompare size={28} />
            </div>
            <h3>Pick a dataset to get started</h3>
            <p>
              Choose a dataset above and select two or more models that were evaluated
              against it to see a side-by-side breakdown of scores, accuracy, and pass rates.
            </p>
            <div className={styles['empty-state__stats']}>
              <div className={styles['empty-state__stat']}>
                <FlaskConical size={16} />
                <span><strong>{benchmarkOptions.length}</strong> dataset{benchmarkOptions.length === 1 ? '' : 's'} available</span>
              </div>
              <div className={styles['empty-state__stat']}>
                <Sparkles size={16} />
                <span><strong>{models.length}</strong> model{models.length === 1 ? '' : 's'} in catalog</span>
              </div>
            </div>
          </div>
        )}

        {selBenchmark && (
          <div className={styles.panel}>
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
              className={styles['compare-btn']}
              disabled={!canCompare}
              onClick={handleCompare}
            >
              <Play size={14} /> Compare {selModelIds.length > 0 ? `(${selModelIds.length})` : ''}
            </button>
            {selModelIds.length === 1 && (
              <div className={styles['comparison__hint']}>Select at least 2 models to compare.</div>
            )}
            {compareErrorLocal && (
              <div className={`${styles['comparison__hint']} ${styles['comparison__hint--error']}`}>{compareErrorLocal}</div>
            )}
          </div>
        )}

        {compareStatus === 'loading' && <div className={styles['loading-wrap']}><ComparisonSkeleton /></div>}

        {compareStatus === 'failed' && (
          <div className={`${styles.panel} ${styles.empty} ${styles['loading-wrap']}`}>{compareError || 'Comparison failed.'}</div>
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
              <div className={styles.panel}>
                <div className={styles['comparison__panel-title']}>Strength Profile</div>
                <div className={styles['comparison__panel-sub']}>Score · Accuracy · Benchmark accuracy · Pass rate</div>
                <div className={styles['radar-wrap']}>
                  <RadarChart models={rows} size={280} colors={COLORS} />
                </div>
                <div className={styles['comparison__legend']}>
                  {rows.map((r, i) => (
                    <span key={r.modelId}><span className={styles['comparison__dot']} style={{ background: COLORS[i % COLORS.length] }} /> {r.name}</span>
                  ))}
                </div>
              </div>

              <div className={`${styles.panel} ${styles['panel--flush']}`}>
                <div className={styles['table-title']}>
                  {result.dataset_name} — Metric Breakdown
                </div>
                <table className={styles['results-table']}>
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
                        <td className={styles['cell-model']} style={{ color: COLORS[i % COLORS.length] }}>{r.name}</td>
                        <td className={styles['cell-provider']}>{r.provider || '—'}</td>
                        <td className={styles['cell-num']}>{(r.score * 100).toFixed(1)}%</td>
                        <td className={`${styles['cell-num']} ${styles['cell-num--muted']}`}>{(r.accuracy * 100).toFixed(1)}%</td>
                        <td className={`${styles['cell-num']} ${styles['cell-num--muted']}`}>{(r.benchmarkAccuracy * 100).toFixed(1)}%</td>
                        <td className={styles['cell-pass']}>{r.passed}</td>
                        <td className={styles['cell-fail']}>{r.total - r.passed}</td>
                      </tr>
                    ))}
                  </tbody>
                </table>
              </div>
            </div>

            <div className={styles.panel}>
              <div className={`${styles['comparison__panel-title']} ${styles['panel-title--spaced']}`}>Score Comparison</div>
              <div className={styles['comparison__scores']}>
                {rows.map((r, i) => (
                  <div key={r.modelId} className={styles['comparison__score-item']}>
                    <ScoreRing score={Math.round(r.score * 100)} size={100} stroke={7} color={COLORS[i % COLORS.length]} label="SCORE" />
                    <div className={styles['score-item__name']}>{r.name}</div>
                    <div className={styles['score-item__meta']}>{r.passed}/{r.total} passed</div>
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
      <div className={styles.panel}>
        <div className={`${styles.skeletonLine} ${styles['skeletonLine--title']}`} />
        <div className={styles.skeletonCircle} />
      </div>
      <div className={styles.panel}>
        <div className={`${styles.skeletonLine} ${styles['skeletonLine--row']}`} />
        {[...Array(3)].map((_, i) => (
          <div key={i} className={`${styles.skeletonLine} ${styles['skeletonLine--row']}`} />
        ))}
      </div>
    </div>
  );
}






























@use '../../styles/_variables' as *;

// ===========================================================================
// Comparison — mirrors History/Reports design system: ink/paper palette,
// ultramarine signal accent, mono instrument labels, hover-lift, mono
// numerals, self-contained panels (no dependency on global .card/.tbl/.btn).
// ===========================================================================

$ink:      #14161B;
$ink-2:    #565B66;
$ink-3:    #8A909B;
$paper:    #F5F6F8;
$card:     #FFFFFF;
$line:     #E6E8EC;
$line-2:   #EEF0F3;
$signal:   #2B2BF5;
$signal-2: #1C1CC7;
$wash:     #ECEDFF;
$ok:       #0FA968;
$ok-wash:  #E7F7EF;
$amber:    #E08600;
$amber-wash: #FDF3E3;
$danger:   #DC2626;
$danger-wash: #FDECEC;
$ink-wash: #EEF0F2;

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

$soft: 0 1px 2px rgba(20, 22, 27, 0.05);
$lift: 0 14px 30px -14px rgba(20, 22, 27, 0.22);

%micro {
  font-family: $mono;
  font-size: 0.6875rem;
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

.comparison {
  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding: 24px 32px 20px;
    border-bottom: 1px solid $line;
    background: $card;

    h1 {
      font-family: $display;
      font-size: 1.5rem;
      font-weight: 800;
      letter-spacing: -0.02em;
      color: $ink;
      line-height: 1.2;
    }
  }

  &__header-eyebrow {
    @extend %micro;
    display: flex;
    align-items: center;
    gap: 8px;
    color: $signal;
    margin-bottom: 6px;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $signal;
    }
  }

  &__header-sub {
    margin-top: 4px;
    font-size: 0.84375rem;
    color: $ink-2;
  }

  &__header-meta {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 7px;
    padding: 7px 13px;
    border-radius: 999px;
    border: 1px solid $line;
    background: $paper;
    font-family: $mono;
    font-size: 0.71875rem;
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ink-2;
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
    @extend %micro;
    font-size: 0.6875rem;
    color: $ink-3;
  }

  &__grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 16px;
    margin-bottom: 20px;
  }

  &__panel-title {
    font-family: $display;
    font-size: 0.875rem;
    font-weight: 700;
    color: $ink;
  }

  &__panel-sub {
    font-size: 0.75rem;
    color: $ink-3;
    margin-top: 2px;
    margin-bottom: 16px;
  }

  &__legend {
    display: flex;
    gap: 14px;
    justify-content: center;
    margin-top: 12px;
    font-family: $mono;
    font-size: 0.6875rem;
    font-weight: 700;
    color: $ink-2;
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

  &__score-item {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 8px;
  }

  &__hint {
    margin-top: 8px;
    font-family: $mono;
    font-size: 0.71875rem;
    color: $ink-3;

    &--error { color: $danger; }
  }
}

.score-item__name { font-family: $display; font-weight: 700; font-size: 0.875rem; color: $ink; text-align: center; }
.score-item__meta { font-family: $mono; font-size: 0.75rem; color: $ink-3; }

// ---- shared card/panel (replaces global .card) -----------------------------
.panel {
  padding: 20px 24px;
  background: $card;
  border: 1px solid $line;
  border-radius: 16px;
  box-shadow: $soft;

  &--flush { padding: 0; }
}

.panel-title--spaced { margin-bottom: 20px; }

.table-title {
  font-family: $display;
  font-size: 0.875rem;
  font-weight: 700;
  color: $ink;
  padding: 20px 24px;
  border-bottom: 1px solid $line;
}

.radar-wrap {
  display: flex;
  justify-content: center;
  padding: 8px 0;
}

.loading-wrap { margin-top: 20px; }

// ---- model chips (comparing summary) ---------------------------------------
.model-chip {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  font-family: $mono;
  font-size: 0.71875rem;
  font-weight: 700;
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

// ---- model select grid ------------------------------------------------------
.model-select-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(160px, 1fr));
  gap: 8px;
  margin-top: 14px;
  margin-bottom: 16px;
}

.model-select-item {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 12px;
  border: 1.5px solid $line;
  border-radius: 10px;
  background: $paper;
  font-size: 0.8125rem;
  font-weight: 650;
  color: $ink;
  cursor: pointer;
  text-align: left;
  transition: border-color 0.15s ease, background 0.15s ease, box-shadow 0.15s ease;

  &:hover { border-color: $ink-3; }

  &.active {
    background: $card;
    box-shadow: $soft;
  }

  &__check {
    width: 16px;
    height: 16px;
    border-radius: 4px;
    border: 1.5px solid $line;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-shrink: 0;
    color: inherit;
  }
}

.compare-btn {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 9px 16px;
  border-radius: 10px;
  border: 1px solid $signal;
  background: $signal;
  color: #fff;
  font-family: $mono;
  font-size: 0.75rem;
  font-weight: 700;
  letter-spacing: 0.03em;
  cursor: pointer;
  transition: background 0.15s ease, border-color 0.15s ease, opacity 0.15s ease;

  &:hover:not(:disabled) { background: $signal-2; border-color: $signal-2; }

  &:disabled {
    opacity: 0.4;
    cursor: not-allowed;
  }
}

.empty {
  padding: 24px;
  text-align: center;
  color: $ink-3;
  font-size: 0.8125rem;
}

// ---------------------------------------------------------------------------
// Empty state shown before a benchmark is selected.
// ---------------------------------------------------------------------------
.empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  padding: 56px 32px;
  border: 1px dashed $line;
  border-radius: 16px;
  background: $paper;

  &__icon {
    width: 56px;
    height: 56px;
    border-radius: 16px;
    background: $wash;
    color: $signal;
    display: flex;
    align-items: center;
    justify-content: center;
    margin-bottom: 18px;
  }

  h3 {
    font-family: $display;
    font-size: 1rem;
    font-weight: 700;
    color: $ink;
    margin-bottom: 8px;
  }

  p {
    max-width: 420px;
    font-size: 0.8125rem;
    line-height: 1.6;
    color: $ink-2;
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
    font-size: 0.78125rem;
    color: $ink-2;
    background: $card;
    border: 1px solid $line;
    border-radius: 999px;
    padding: 8px 14px;

    svg { color: $signal; flex-shrink: 0; }

    strong {
      color: $ink;
      font-weight: 700;
    }
  }
}

// ---- results table (shared visual grammar with History/Reports) -----------
.results-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.84375rem;

  thead th {
    text-align: left;
    background: $paper;
    border-bottom: 1px solid $line;
    @extend %micro;
    font-size: 0.5625rem;
    color: $ink-3;
    padding: 11px 14px;
    white-space: nowrap;
  }

  tbody tr {
    border-bottom: 1px solid $line-2;
    transition: background 0.13s ease;

    &:last-child { border-bottom: 0; }
    &:hover { background: $paper; }
  }

  tbody td {
    padding: 12px 14px;
    color: $ink;
  }
}

.cell-model { font-family: $display; font-weight: 700; }
.cell-provider { color: $ink-2; }
.cell-num { font-family: $mono; font-size: 0.8125rem; font-weight: 700; color: $ink; }
.cell-num--muted { font-weight: 500; color: $ink-2; }
.cell-pass { font-family: $mono; font-size: 0.8125rem; font-weight: 700; color: $ok; }
.cell-fail { font-family: $mono; font-size: 0.8125rem; font-weight: 700; color: $danger; }

// ---------------------------------------------------------------------------
// Skeleton loader
// ---------------------------------------------------------------------------
@keyframes comparison-skeleton-pulse {
  0%, 100% { opacity: 0.55; }
  50% { opacity: 1; }
}

.skeletonLine,
.skeletonCircle {
  background: $line-2;
  border-radius: 8px;
  animation: comparison-skeleton-pulse 1.3s ease-in-out infinite;
}

.skeletonLine--title { width: 40%; height: 16px; margin-bottom: 20px; }
.skeletonLine--row { width: 100%; height: 32px; margin-bottom: 8px; }

.skeletonCircle {
  width: 100px;
  height: 100px;
  border-radius: 50%;
  margin: 20px auto 0;
}

@media (max-width: 900px) {
  .comparison__grid {
    grid-template-columns: 1fr;
  }
}

@media (max-width: 640px) {
  .comparison__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
}
