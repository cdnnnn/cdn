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
  const [compareErrorLocal, setCompareErrorLocal] = useState<string | null>(null);

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
            {compareErrorLocal && (
              <div className={styles['comparison__hint']} style={{ color: '#EF4444' }}>{compareErrorLocal}</div>
            )}
          </div>
        )}

        {compareStatus === 'loading' && <div style={{ marginTop: 20 }}><ComparisonSkeleton /></div>}

        {compareStatus === 'failed' && (
          <div className={`card ${styles.empty}`} style={{ marginTop: 20 }}>{compareError || 'Comparison failed.'}</div>
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
