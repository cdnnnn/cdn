//ModelComparator.tsx
import { useEffect, useMemo, useState, type FC } from 'react';
import { Search, X, CheckCircle2, GitCompareArrows, ArrowLeft, Trophy, Database, AlertCircle } from 'lucide-react';
import { MODELS } from '../RunEvaluation/data';
import type { ModelInfo } from '../RunEvaluation/types';
import { fetchBenchmarks } from '../Datasets/api';
import type { Benchmark } from '../Datasets/types';
import Spinner from '../../../components/Spinner/Spinner';
import './ModelComparator.scss';

const MAX_SELECTION = 4;

// ---------- helpers to pull comparable numbers out of display strings ----------
function parsePrice(pricing: string): number | null {
  const match = pricing.match(/([\d.]+)/);
  return match ? parseFloat(match[1]) : null;
}

function parseSpeed(speedRating: string): number | null {
  const match = speedRating.match(/\(([\d.]+)/);
  return match ? parseFloat(match[1]) : null;
}

function parseContextWindow(contextWindow: string): number {
  const cleaned = contextWindow.replace(/,/g, '');
  const kMatch = cleaned.match(/([\d.]+)\s*k/i);
  if (kMatch) return parseFloat(kMatch[1]) * 1_000;
  const numMatch = cleaned.match(/([\d.]+)/);
  return numMatch ? parseFloat(numMatch[1]) : 0;
}

// There's no real "which models were evaluated on this dataset" data yet, so
// this connects the two heuristically: a model counts as evaluated on a
// dataset if it shares at least one capability keyword with the dataset's
// required_capabilities (normalized + substring-matched, since the two data
// sources use different naming conventions — e.g. "tool_calling" vs "Tool Calling").
function normalizeCapability(value: string): string {
  return value.toLowerCase().replace(/[^a-z0-9]/g, '');
}

function isModelEvaluatedOn(model: ModelInfo, benchmark: Benchmark): boolean {
  return benchmark.required_capabilities.some((bc) => {
    const a = normalizeCapability(bc);
    return model.capabilities.some((mc) => {
      const b = normalizeCapability(mc);
      return a.includes(b) || b.includes(a);
    });
  });
}

type MetricRow = {
  key: string;
  label: string;
  getValue: (m: ModelInfo) => string;
  getSortValue?: (m: ModelInfo) => number | null;
  betterIsHigher?: boolean;
};

const METRIC_ROWS: MetricRow[] = [
  { key: 'provider', label: 'Provider', getValue: (m) => m.provider },
  { key: 'version', label: 'Version', getValue: (m) => m.version },
  { key: 'context', label: 'Context Window', getValue: (m) => m.contextWindow },
  {
    key: 'accuracy',
    label: 'Accuracy',
    getValue: (m) => `${m.accuracyScore}%`,
    getSortValue: (m) => m.accuracyScore,
    betterIsHigher: true,
  },
  {
    key: 'agent',
    label: 'Agent Score',
    getValue: (m) => `${m.agentScore}%`,
    getSortValue: (m) => m.agentScore,
    betterIsHigher: true,
  },
  {
    key: 'speed',
    label: 'Speed',
    getValue: (m) => m.speedRating,
    getSortValue: (m) => parseSpeed(m.speedRating),
    betterIsHigher: true,
  },
  {
    key: 'price',
    label: 'Pricing',
    getValue: (m) => m.pricing,
    getSortValue: (m) => parsePrice(m.pricing),
    betterIsHigher: false,
  },
];

const MODEL_COLORS = ['var(--primary)', 'var(--violet)', 'var(--warning)', 'var(--success)'];

type RadarAxis = {
  key: string;
  label: string;
  get: (m: ModelInfo) => number;
  higherIsBetter: boolean;
};

const RADAR_AXES: RadarAxis[] = [
  { key: 'accuracy', label: 'Accuracy', get: (m) => m.accuracyScore, higherIsBetter: true },
  { key: 'agent', label: 'Agent Score', get: (m) => m.agentScore, higherIsBetter: true },
  { key: 'speed', label: 'Speed', get: (m) => parseSpeed(m.speedRating) ?? 0, higherIsBetter: true },
  { key: 'price', label: 'Affordability', get: (m) => parsePrice(m.pricing) ?? 0, higherIsBetter: false },
  { key: 'context', label: 'Context', get: (m) => parseContextWindow(m.contextWindow), higherIsBetter: true },
];

const RadarChart: FC<{ models: ModelInfo[] }> = ({ models }) => {
  const size = 280;
  const center = size / 2;
  const maxRadius = 96;
  const angleStep = (2 * Math.PI) / RADAR_AXES.length;
  const angleFor = (i: number) => -Math.PI / 2 + i * angleStep;

  // Normalize each axis 0–100 across just the selected models, so the chart
  // always uses the full space regardless of the underlying units.
  const scoreFor = (axis: RadarAxis, m: ModelInfo) => {
    const values = models.map((mm) => axis.get(mm));
    const max = Math.max(...values);
    const min = Math.min(...values);
    const v = axis.get(m);
    if (axis.higherIsBetter) {
      return max === 0 ? 0 : (v / max) * 100;
    }
    // lower-is-better (price): cheapest gets 100, scale the rest down
    if (v === 0) return 100;
    const cheapest = min === 0 ? v : min;
    return (cheapest / v) * 100;
  };

  const pointFor = (i: number, valuePct: number) => {
    const angle = angleFor(i);
    const r = (valuePct / 100) * maxRadius;
    return [center + r * Math.cos(angle), center + r * Math.sin(angle)] as const;
  };

  const ringLevels = [0.25, 0.5, 0.75, 1];

  return (
    <svg viewBox={`0 0 ${size} ${size}`} className="model-comparator__radar-svg">
      {/* grid rings */}
      {ringLevels.map((level) => {
        const pts = RADAR_AXES.map((_, i) => pointFor(i, level * 100).join(',')).join(' ');
        return <polygon key={level} points={pts} className="model-comparator__radar-ring" />;
      })}

      {/* axis lines + labels */}
      {RADAR_AXES.map((axis, i) => {
        const [x, y] = pointFor(i, 100);
        const [lx, ly] = pointFor(i, 118);
        const anchor = Math.abs(Math.cos(angleFor(i))) < 0.15 ? 'middle' : Math.cos(angleFor(i)) > 0 ? 'start' : 'end';
        return (
          <g key={axis.key}>
            <line x1={center} y1={center} x2={x} y2={y} className="model-comparator__radar-axis" />
            <text x={lx} y={ly} textAnchor={anchor} dominantBaseline="middle" className="model-comparator__radar-label">
              {axis.label}
            </text>
          </g>
        );
      })}

      {/* one polygon per selected model */}
      {models.map((m, mi) => {
        const pts = RADAR_AXES.map((axis, i) => pointFor(i, scoreFor(axis, m)).join(',')).join(' ');
        const color = MODEL_COLORS[mi % MODEL_COLORS.length];
        return (
          <polygon key={m.id} points={pts} fill={color} fillOpacity={0.14} stroke={color} strokeWidth={2} strokeLinejoin="round" />
        );
      })}
    </svg>
  );
};

const ModelComparator: FC = () => {
  const [query, setQuery] = useState('');
  const [selectedIds, setSelectedIds] = useState<string[]>([]);
  const [comparing, setComparing] = useState(false);

  // ---------- dataset filter (optional) ----------
  const [benchmarks, setBenchmarks] = useState<Benchmark[]>([]);
  const [benchmarksLoading, setBenchmarksLoading] = useState(true);
  const [benchmarksError, setBenchmarksError] = useState<string | null>(null);
  const [selectedDatasetNames, setSelectedDatasetNames] = useState<string[]>([]);

  useEffect(() => {
    fetchBenchmarks()
      .then((res) => setBenchmarks(res.benchmarks))
      .catch((err) => setBenchmarksError(err instanceof Error ? err.message : 'Failed to load test suites.'))
      .finally(() => setBenchmarksLoading(false));
  }, []);

  const toggleDataset = (name: string) => {
    setSelectedDatasetNames((prev) => (prev.includes(name) ? prev.filter((n) => n !== name) : [...prev, name]));
  };

  // Models eligible for comparison: if no dataset is selected, every model is
  // fair game. Once one or more datasets are picked, only models sharing a
  // capability with at least one of them are shown.
  const eligibleModels = useMemo(() => {
    if (selectedDatasetNames.length === 0) return MODELS;
    const selectedBenchmarks = benchmarks.filter((b) => selectedDatasetNames.includes(b.name));
    return MODELS.filter((m) => selectedBenchmarks.some((b) => isModelEvaluatedOn(m, b)));
  }, [selectedDatasetNames, benchmarks]);

  // If narrowing the dataset filter drops a model that was already selected
  // for comparison, drop it from the selection too rather than leaving a
  // "selected but no longer eligible" model silently in the comparison.
  useEffect(() => {
    setSelectedIds((prev) => prev.filter((id) => eligibleModels.some((m) => m.id === id)));
  }, [eligibleModels]);

  const filtered = useMemo(
    () =>
      eligibleModels.filter(
        (m) =>
          !query ||
          m.name.toLowerCase().includes(query.toLowerCase()) ||
          m.provider.toLowerCase().includes(query.toLowerCase())
      ),
    [eligibleModels, query]
  );

  const selectedModels = useMemo(() => MODELS.filter((m) => selectedIds.includes(m.id)), [selectedIds]);

  const toggle = (id: string) => {
    setSelectedIds((prev) => {
      if (prev.includes(id)) return prev.filter((x) => x !== id);
      if (prev.length >= MAX_SELECTION) return prev;
      return [...prev, id];
    });
  };

  const clearSelection = () => {
    setSelectedIds([]);
    setComparing(false);
  };

  // For each metric row, figure out which of the selected models has the
  // "best" value, so the comparison table can highlight it.
  const bestByMetric = useMemo(() => {
    const map = new Map<string, string>(); // metricKey -> modelId
    METRIC_ROWS.forEach((row) => {
      if (!row.getSortValue) return;
      let bestId: string | null = null;
      let bestValue: number | null = null;
      selectedModels.forEach((m) => {
        const v = row.getSortValue!(m);
        if (v === null) return;
        const isBetter = bestValue === null || (row.betterIsHigher ? v > bestValue : v < bestValue);
        if (isBetter) {
          bestValue = v;
          bestId = m.id;
        }
      });
      if (bestId) map.set(row.key, bestId);
    });
    return map;
  }, [selectedModels]);

  return (
    <div className="model-comparator">
      <div className="model-comparator__header">
        <div className="model-comparator__header-left">
          <p className="model-comparator__header-eyebrow">Side-by-side evaluation</p>
          <h1 className="model-comparator__title">Model Comparator</h1>
          <p className="model-comparator__subtitle">
            {comparing ? 'Comparing selected models' : `Select up to ${MAX_SELECTION} models to compare`}
          </p>
        </div>

        {comparing && (
          <button type="button" className="model-comparator__btn model-comparator__btn--outline" onClick={() => setComparing(false)}>
            <ArrowLeft size={14} strokeWidth={2.25} /> Edit selection
          </button>
        )}
      </div>

      {!comparing && (
        <>
          <div className="model-comparator__datasets-panel">
            <div className="model-comparator__panel-row">
              <p className="model-comparator__panel-title">
                <Database size={12} strokeWidth={2.25} /> Filter by test suite
              </p>
              {selectedDatasetNames.length > 0 && (
                <button type="button" className="model-comparator__link-btn" onClick={() => setSelectedDatasetNames([])}>
                  Clear filter
                </button>
              )}
            </div>

            {benchmarksLoading && (
              <div className="model-comparator__datasets-loading">
                <Spinner size={16} />
                Loading test suites…
              </div>
            )}

            {!benchmarksLoading && benchmarksError && (
              <div className="model-comparator__datasets-error">
                <AlertCircle size={14} />
                {benchmarksError}
              </div>
            )}

            {!benchmarksLoading && !benchmarksError && (
              <div className="model-comparator__dataset-chips">
                {benchmarks.map((b) => {
                  const active = selectedDatasetNames.includes(b.name);
                  return (
                    <button
                      type="button"
                      key={b.name}
                      className={`model-comparator__dataset-chip${active ? ' model-comparator__dataset-chip--active' : ''}`}
                      onClick={() => toggleDataset(b.name)}
                    >
                      {b.name} <span className="n">· {b.task_count}</span>
                    </button>
                  );
                })}
              </div>
            )}

            {selectedDatasetNames.length > 0 && (
              <p className="model-comparator__filter-hint">
                Showing <b>{eligibleModels.length}</b> model{eligibleModels.length === 1 ? '' : 's'} evaluated on{' '}
                {selectedDatasetNames.length === 1 ? 'this test suite' : `${selectedDatasetNames.length} selected test suites`}.
              </p>
            )}
          </div>

          <div className="model-comparator__search">
            <Search size={15} />
            <input type="text" placeholder="Search models..." value={query} onChange={(e) => setQuery(e.target.value)} />
            {query && (
              <button type="button" className="model-comparator__search-clear" onClick={() => setQuery('')} aria-label="Clear search">
                <X size={13} />
              </button>
            )}
          </div>

          <div className="model-comparator__grid">
            {filtered.map((m) => {
              const selected = selectedIds.includes(m.id);
              const disabled = !selected && selectedIds.length >= MAX_SELECTION;
              return (
                <button
                  type="button"
                  key={m.id}
                  className={`model-comparator__card${selected ? ' model-comparator__card--selected' : ''}${
                    disabled ? ' model-comparator__card--disabled' : ''
                  }`}
                  onClick={() => !disabled && toggle(m.id)}
                  disabled={disabled}
                >
                  <span className={`model-comparator__checkbox${selected ? ' model-comparator__checkbox--checked' : ''}`}>
                    {selected && <CheckCircle2 size={14} strokeWidth={2.5} />}
                  </span>

                  <span className="model-comparator__card-body">
                    <span className="model-comparator__card-top">
                      <span className="model-comparator__card-name">{m.name}</span>
                      <span className="model-comparator__tag">{m.provider}</span>
                    </span>
                    <span className="model-comparator__card-desc">{m.description}</span>
                    <span className="model-comparator__card-meta">
                      <span>
                        Accuracy <b className="n">{m.accuracyScore}%</b>
                      </span>
                      <span>
                        Speed <b>{m.speedRating}</b>
                      </span>
                      <span>
                        Price <b>{m.pricing}</b>
                      </span>
                    </span>
                  </span>
                </button>
              );
            })}

            {filtered.length === 0 && (
              <div className="model-comparator__empty">
                <Search size={22} />
                <p>No models match your search or test suite filter.</p>
              </div>
            )}
          </div>
        </>
      )}

      {comparing && selectedModels.length > 0 && (
        <div className="model-comparator__compare-grid">
          <div className="model-comparator__chart-panel">
            <p className="model-comparator__panel-title">Overall shape</p>
            <RadarChart models={selectedModels} />
            <div className="model-comparator__legend">
              {selectedModels.map((m, i) => (
                <div className="model-comparator__legend-item" key={m.id}>
                  <span className="model-comparator__legend-dot" style={{ background: MODEL_COLORS[i % MODEL_COLORS.length] }} />
                  <span className="model-comparator__legend-name">{m.name}</span>
                </div>
              ))}
            </div>
            <p className="model-comparator__panel-hint">
              Each axis is normalized 0–100 across just the models you selected, so the shape shows relative strengths, not
              absolute scores.
            </p>
          </div>

          <div className="model-comparator__table-wrap">
          <table className="model-comparator__table">
            <thead>
              <tr>
                <th className="model-comparator__row-label-col">Metric</th>
                {selectedModels.map((m) => (
                  <th key={m.id}>
                    <div className="model-comparator__col-head">
                      <span className="model-comparator__col-name">{m.name}</span>
                      <button
                        type="button"
                        className="model-comparator__col-remove"
                        onClick={() => setSelectedIds((prev) => prev.filter((id) => id !== m.id))}
                        aria-label={`Remove ${m.name} from comparison`}
                      >
                        <X size={12} strokeWidth={2.5} />
                      </button>
                    </div>
                  </th>
                ))}
              </tr>
            </thead>
            <tbody>
              {METRIC_ROWS.map((row) => (
                <tr key={row.key}>
                  <td className="model-comparator__row-label-col model-comparator__row-label">{row.label}</td>
                  {selectedModels.map((m) => {
                    const isBest = bestByMetric.get(row.key) === m.id;
                    return (
                      <td key={m.id} className={isBest ? 'model-comparator__cell--best' : undefined}>
                        <span className="model-comparator__cell-value n">{row.getValue(m)}</span>
                        {isBest && (
                          <span className="model-comparator__cell-best-badge">
                            <Trophy size={11} strokeWidth={2.5} /> Best
                          </span>
                        )}
                      </td>
                    );
                  })}
                </tr>
              ))}
              <tr>
                <td className="model-comparator__row-label-col model-comparator__row-label">Capabilities</td>
                {selectedModels.map((m) => (
                  <td key={m.id}>
                    <div className="model-comparator__cap-list">
                      {m.capabilities.map((c) => (
                        <span key={c} className="model-comparator__cap-pill">
                          {c}
                        </span>
                      ))}
                    </div>
                  </td>
                ))}
              </tr>
            </tbody>
          </table>
          </div>
        </div>
      )}

      {selectedIds.length > 0 && (
        <div className="model-comparator__bar">
          <div className="model-comparator__bar-left">
            <GitCompareArrows size={15} strokeWidth={2.25} />
            <span>
              <b className="n">{selectedIds.length}</b> of {MAX_SELECTION} models selected
            </span>
          </div>
          <div className="model-comparator__bar-actions">
            <button type="button" className="model-comparator__btn model-comparator__btn--outline" onClick={clearSelection}>
              Clear
            </button>
            <button
              type="button"
              className="model-comparator__btn model-comparator__btn--primary"
              onClick={() => setComparing(true)}
              disabled={selectedIds.length < 2}
            >
              <GitCompareArrows size={14} strokeWidth={2.25} /> Compare Selected
            </button>
          </div>
        </div>
      )}
    </div>
  );
};

export default ModelComparator;


















//ModelComparator.scss
@use '../../../styles/variables' as *;

.model-comparator {
  display: flex;
  flex-direction: column;
  gap: 16px;
  padding-bottom: 76px; // room for the sticky bar so it never covers the last row

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

  /* ---------- search ---------- */
  &__search {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 9px;
    width: 300px;
    max-width: 100%;
    border: 1px solid $border-default;
    border-radius: 10px;
    padding: 9px 12px;
    background: $bg-main;
    color: $text-tertiary;
    transition: border-color 0.14s ease;

    &:focus-within {
      border-color: $primary;
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

    &:hover {
      background: $border-default;
      color: $text-primary;
    }
  }

  /* ---------- dataset filter panel ---------- */
  &__datasets-panel {
    flex-shrink: 0;
    background: $bg-subtle;
    border: 1px solid $border-subtle;
    border-radius: 12px;
    padding: 14px 16px;
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  &__panel-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
  }

  &__datasets-loading,
  &__datasets-error {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 0.78125rem;
    color: $text-tertiary;
  }

  &__datasets-error {
    color: $danger;
  }

  &__dataset-chips {
    display: flex;
    flex-wrap: wrap;
    gap: 7px;
  }

  &__dataset-chip {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-family: $font-body;
    font-size: 0.78125rem;
    font-weight: 600;
    color: $text-secondary;
    background: $bg-main;
    border: 1px solid $border-default;
    border-radius: 999px;
    padding: 6px 12px 6px 10px;
    cursor: pointer;
    transition: background 0.14s ease, border-color 0.14s ease, color 0.14s ease;

    .n {
      color: $text-tertiary;
      font-weight: 500;
    }

    &:hover {
      border-color: $border-strong;
    }

    &--active {
      background: $primary;
      border-color: $primary;
      color: $on-primary;

      .n {
        color: rgba(255, 255, 255, 0.75);
      }
    }
  }

  &__filter-hint {
    font-size: 0.75rem;
    color: $text-tertiary;

    b {
      color: $text-primary;
    }
  }

  &__link-btn {
    font-family: $font-body;
    font-size: 0.75rem;
    font-weight: 700;
    color: $primary;
    background: transparent;
    border: none;
    padding: 0;
    cursor: pointer;

    &:hover {
      text-decoration: underline;
    }
  }

  /* ---------- selection grid ---------- */
  &__grid {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 12px;
  }

  &__card {
    position: relative;
    display: flex;
    align-items: flex-start;
    gap: 12px;
    text-align: left;
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 12px;
    padding: 14px 16px;
    cursor: pointer;
    font-family: inherit;
    transition: border-color 0.14s ease, box-shadow 0.14s ease, background 0.14s ease;

    &:hover {
      border-color: $border-strong;
      box-shadow: $shadow-xs;
    }

    &--selected {
      border-color: $primary;
      background: $primary-light;
      box-shadow: 0 0 0 1px $primary;
    }

    &--disabled {
      opacity: 0.45;
      cursor: not-allowed;

      &:hover {
        border-color: $border-subtle;
        box-shadow: none;
      }
    }
  }

  &__checkbox {
    flex-shrink: 0;
    width: 20px;
    height: 20px;
    border-radius: 6px;
    border: 1.5px solid $border-strong;
    background: $bg-main;
    display: grid;
    place-items: center;
    color: transparent;
    margin-top: 1px;
    transition: background 0.14s ease, border-color 0.14s ease, color 0.14s ease;

    &--checked {
      background: $primary;
      border-color: $primary;
      color: $on-primary;
    }
  }

  &__card-body {
    display: flex;
    flex-direction: column;
    gap: 5px;
    min-width: 0;
  }

  &__card-top {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__card-name {
    font-size: 0.875rem;
    font-weight: 700;
    color: $text-primary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__tag {
    flex-shrink: 0;
    font-size: 0.625rem;
    font-weight: 700;
    color: $text-tertiary;
    background: $bg-inset;
    border-radius: 999px;
    padding: 2px 8px;
  }

  &__card--selected &__tag {
    background: $bg-main;
    color: $primary;
  }

  &__card-desc {
    font-size: 0.75rem;
    color: $text-secondary;
    line-height: 1.45;
    display: -webkit-box;
    -webkit-line-clamp: 2;
    -webkit-box-orient: vertical;
    overflow: hidden;
  }

  &__card-meta {
    display: flex;
    flex-wrap: wrap;
    gap: 10px;
    margin-top: 2px;
    font-size: 0.6875rem;
    color: $text-tertiary;

    b {
      color: $text-secondary;
      font-weight: 700;
      margin-left: 3px;
    }
  }

  &__empty {
    grid-column: 1 / -1;
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

  /* ---------- sticky selection/compare bar ---------- */
  &__bar {
    position: fixed;
    left: 50%;
    bottom: 34px;
    transform: translateX(-50%);
    z-index: 40;
    display: flex;
    align-items: center;
    gap: 18px;
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 999px;
    box-shadow: $shadow-lg;
    padding: 10px 12px 10px 18px;
  }

  &__bar-left {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 0.8125rem;
    color: $text-secondary;
    white-space: nowrap;

    svg {
      color: $primary;
    }

    b {
      color: $text-primary;
    }
  }

  &__bar-actions {
    display: flex;
    gap: 8px;
  }

  /* ---------- buttons ---------- */
  &__btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 6px;
    font-family: $font-body;
    font-size: 0.8125rem;
    font-weight: 700;
    padding: 8px 14px;
    border-radius: 999px;
    border: 1px solid transparent;
    cursor: pointer;
    white-space: nowrap;
    transition: background 0.14s ease, border-color 0.14s ease, color 0.14s ease, opacity 0.14s ease;

    &:disabled {
      opacity: 0.5;
      cursor: not-allowed;
    }

    &--outline {
      background: $bg-subtle;
      border-color: transparent;
      color: $text-secondary;

      &:hover:not(:disabled) {
        background: $bg-inset;
        color: $text-primary;
      }
    }

    &--primary {
      background: $primary;
      border-color: $primary;
      color: $on-primary;

      &:hover:not(:disabled) {
        background: $primary-hover;
        border-color: $primary-hover;
      }
    }
  }

  /* ---------- comparison table ---------- */
  /* ---------- comparison view: chart + table side by side ---------- */
  &__compare-grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 16px;
    align-items: start;
  }

  &__chart-panel {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 14px;
    padding: 20px;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 14px;
  }

  &__panel-title {
    align-self: flex-start;
    font-size: 0.75rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.04em;
    color: $text-tertiary;
  }

  &__radar-svg {
    width: 100%;
    max-width: 340px;
    height: auto;
    overflow: visible;
  }

  &__radar-ring {
    fill: none;
    stroke: $border-subtle;
    stroke-width: 1;
  }

  &__radar-axis {
    stroke: $border-default;
    stroke-width: 1;
  }

  &__radar-label {
    font-size: 8.5px;
    font-weight: 700;
    fill: $text-tertiary;
    text-transform: uppercase;
    letter-spacing: 0.02em;
  }

  &__legend {
    align-self: stretch;
    display: flex;
    flex-direction: column;
    gap: 8px;
    padding-top: 6px;
    border-top: 1px solid $border-subtle;
  }

  &__legend-item {
    display: flex;
    align-items: center;
    gap: 8px;
  }

  &__legend-dot {
    flex-shrink: 0;
    width: 9px;
    height: 9px;
    border-radius: 50%;
  }

  &__legend-name {
    font-size: 0.8125rem;
    font-weight: 600;
    color: $text-primary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__panel-hint {
    align-self: stretch;
    font-size: 0.6875rem;
    line-height: 1.5;
    color: $text-tertiary;
  }

  &__table-wrap {
    overflow-x: auto;
    border: 1px solid $border-subtle;
    border-radius: 14px;
  }

  &__table {
    width: 100%;
    border-collapse: collapse;
    min-width: 640px;

    thead th {
      background: $bg-subtle;
      padding: 14px 16px;
      border-bottom: 1px solid $border-subtle;
      text-align: left;
      vertical-align: top;
    }

    tbody td {
      padding: 13px 16px;
      border-bottom: 1px solid $border-subtle;
      font-size: 0.8125rem;
      color: $text-secondary;
      vertical-align: top;
    }

    tbody tr:last-child td {
      border-bottom: none;
    }
  }

  &__row-label-col {
    width: 160px;
    min-width: 160px;
    position: sticky;
    left: 0;
    background: $bg-main;
    z-index: 1;
  }

  thead &__row-label-col {
    background: $bg-subtle;
  }

  &__row-label {
    font-size: 0.75rem;
    font-weight: 700;
    color: $text-tertiary;
    text-transform: uppercase;
    letter-spacing: 0.03em;
  }

  &__col-head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 8px;
  }

  &__col-name {
    font-size: 0.875rem;
    font-weight: 800;
    color: $text-primary;
  }

  &__col-remove {
    flex-shrink: 0;
    width: 20px;
    height: 20px;
    border-radius: 50%;
    border: 1px solid $border-default;
    background: $bg-main;
    color: $text-tertiary;
    display: grid;
    place-items: center;
    cursor: pointer;
    transition: border-color 0.14s ease, color 0.14s ease;

    &:hover {
      border-color: $danger;
      color: $danger;
    }
  }

  &__cell-value {
    font-weight: 600;
    color: $text-primary;
  }

  &__cell--best {
    background: $success-subtle;
  }

  &__cell--best &__cell-value {
    color: $success;
    font-weight: 800;
  }

  &__cell-best-badge {
    display: inline-flex;
    align-items: center;
    gap: 3px;
    margin-left: 8px;
    font-size: 0.625rem;
    font-weight: 700;
    color: $success;
    text-transform: uppercase;
    letter-spacing: 0.03em;
  }

  &__cap-list {
    display: flex;
    flex-wrap: wrap;
    gap: 5px;
  }

  &__cap-pill {
    font-size: 0.6875rem;
    font-weight: 600;
    color: $primary;
    background: $primary-light;
    border-radius: 999px;
    padding: 2px 8px;
  }

  /* ---------- responsive ---------- */
  @media (max-width: 1400px) {
    &__grid {
      grid-template-columns: repeat(2, 1fr);
    }
  }

  @media (max-width: 1100px) {
    &__compare-grid {
      grid-template-columns: 1fr;
    }

    &__chart-panel {
      align-items: center;
    }
  }

  @media (max-width: 800px) {
    &__grid {
      grid-template-columns: 1fr;
    }

    &__bar {
      left: 16px;
      right: 16px;
      bottom: 16px;
      transform: none;
      flex-direction: column;
      align-items: stretch;
      border-radius: 16px;
    }

    &__bar-actions {
      justify-content: stretch;

      .model-comparator__btn {
        flex: 1;
      }
    }
  }

  /* ---------- ultra-wide ---------- */
  @media (min-width: 1800px) {
    &__title {
      font-size: 23px;
    }

    &__card-name {
      font-size: 0.9375rem;
    }

    &__card-desc {
      font-size: 0.8125rem;
    }
  }
}
