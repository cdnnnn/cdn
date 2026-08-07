import { useMemo, useState, type FC } from 'react';
import { UploadCloud, Check, Loader2, AlertTriangle, Layers } from 'lucide-react';
import type { Benchmark, EvalTypeId } from '../types';

interface Props {
  evalType: EvalTypeId | null;
  benchmarks: Benchmark[];
  loading: boolean;
  error: string | null;
  selected: string | null;
  onSelect: (id: string) => void;
  subgroup: string[] | string | undefined;
  onToggleSubgroup: (value: string) => void;
}

const DatasetStep: FC<Props> = ({
  evalType,
  benchmarks,
  loading,
  error,
  selected,
  onSelect,
  subgroup: subgroupProp,
  onToggleSubgroup,
}) => {
  const [tab, setTab] = useState<'official' | 'private'>('official');
  const [category, setCategory] = useState('All');

  // Defensive: some callers pass a single string or leave this undefined
  // instead of a string[]. Normalize once here so nothing downstream has
  // to guard against it.
  const subgroup: string[] = Array.isArray(subgroupProp)
    ? subgroupProp
    : subgroupProp
    ? [subgroupProp as unknown as string]
    : [];

  const categories = useMemo(() => {
    const set = new Set<string>();
    benchmarks.forEach((b) => set.add(b.type));
    return ['All', ...Array.from(set).sort()];
  }, [benchmarks]);

  const filtered = useMemo(
    () => benchmarks.filter((b) => category === 'All' || b.type === category),
    [benchmarks, category]
  );

  const selectedBenchmark = useMemo(
    () => benchmarks.find((b) => b.name === selected) ?? null,
    [benchmarks, selected]
  );

  const handleTaskToggle = (taskValue: string) => {
    onToggleSubgroup(taskValue);
  };

  return (
    <div className="run-eval__card">
      <h2 className="run-eval__step-title">Pick a test suite</h2>
      <p className="run-eval__step-desc">Test suites contain questions that measure AI capabilities.</p>

      <div className="run-eval__tabs">
        {(['official', 'private'] as const).map((t) => (
          <button
            key={t}
            type="button"
            className={`run-eval__tab${tab === t ? ' run-eval__tab--active' : ''}`}
            onClick={() => setTab(t)}
          >
            {t === 'official' ? 'Benchmarks' : 'Upload'}
          </button>
        ))}
      </div>

      {tab === 'official' && (
        <>
          {loading && (
            <div className="run-eval__loading-state">
              <Loader2 size={18} className="run-eval__spin" />
              Loading test suites…
            </div>
          )}

          {!loading && error && (
            <div className="run-eval__inline-error">
              <AlertTriangle size={15} />
              {error}
            </div>
          )}

          {!loading && !error && (
            <>
              <div className="run-eval__category-filters">
                {categories.map((c) => (
                  <button
                    key={c}
                    type="button"
                    className={`run-eval__chip${category === c ? ' run-eval__chip--active' : ''}`}
                    onClick={() => setCategory(c)}
                  >
                    {c}
                  </button>
                ))}
              </div>

              {/* Dataset grid + persistent subgroup column, each with its own scroll. */}
              <div className="run-eval__dataset-layout">
                <div className="run-eval__dataset-grid-scroll">
                  <div className="run-eval__dataset-grid">
                    {filtered.map((b) => {
                      const isSelected = selected === b.name;
                      const recommended = evalType ? b.type === evalType : false;
                      const hasSubgroupSelected = isSelected && subgroup.length > 0;
                      const tasks = b.tasks ?? [];
                      const requiredCapabilities = b.required_capabilities ?? [];
                      return (
                        <button
                          key={b.name}
                          type="button"
                          className={`run-eval__dataset-card${isSelected ? ' run-eval__dataset-card--selected' : ''}`}
                          onClick={() => onSelect(b.name)}
                        >
                          <div className="run-eval__dataset-top">
                            <span className="run-eval__dataset-name">{b.name}</span>
                            <span className="run-eval__dataset-top-actions">
                              {tasks.length > 0 && (
                                <span className="run-eval__subgroup-btn" title="Has subgroups">
                                  <Layers size={12} />
                                  {tasks.length}
                                </span>
                              )}
                              {isSelected && (
                                <span className="run-eval__type-check">
                                  <Check size={12} strokeWidth={2.75} />
                                </span>
                              )}
                            </span>
                          </div>
                          <p className="run-eval__dataset-desc">{b.description}</p>
                          <div className="run-eval__dataset-meta n">
                            <span>{b.task_count} tasks</span>
                            <span>{b.type}</span>
                          </div>
                          {requiredCapabilities.length > 0 && (
                            <div className="run-eval__dataset-caps">
                              {requiredCapabilities.slice(0, 4).map((c) => (
                                <span key={c} className="run-eval__chip run-eval__chip--static">
                                  {c}
                                </span>
                              ))}
                            </div>
                          )}
                          {hasSubgroupSelected && (
                            <span className="run-eval__badge run-eval__badge--soft">
                              {subgroup.length} subgroup{subgroup.length === 1 ? '' : 's'} selected
                            </span>
                          )}
                          {!hasSubgroupSelected && recommended && (
                            <span className="run-eval__badge run-eval__badge--soft">Recommended</span>
                          )}
                        </button>
                      );
                    })}
                    {filtered.length === 0 && <p className="run-eval__empty">No test suites match this category.</p>}
                  </div>
                </div>

                {/* Subgroup column — only shown once a dataset with subgroups is selected. */}
                {selectedBenchmark && (selectedBenchmark.tasks ?? []).length > 0 && (
                  <aside className="run-eval__subgroup-panel">
                    <div className="run-eval__subgroup-panel-head">
                      <p className="run-eval__subgroup-panel-eyebrow">Subgroups</p>
                      <h3 className="run-eval__subgroup-panel-title">{selectedBenchmark.name}</h3>
                      <p className="run-eval__subgroup-panel-sub">
                        {subgroup.length > 0
                          ? `${subgroup.length} of ${(selectedBenchmark.tasks ?? []).length} selected`
                          : `${(selectedBenchmark.tasks ?? []).length} available`}
                      </p>
                    </div>

                    <div className="run-eval__subgroup-panel-scroll">
                      {(selectedBenchmark.tasks ?? []).map((task) => {
                        const taskSelected = subgroup.includes(task.value);
                        return (
                          <button
                            key={task.value}
                            type="button"
                            className={`run-eval__drawer-task${taskSelected ? ' run-eval__drawer-task--selected' : ''}`}
                            onClick={() => handleTaskToggle(task.value)}
                            role="checkbox"
                            aria-checked={taskSelected}
                          >
                            <span className={`run-eval__checkbox${taskSelected ? ' run-eval__checkbox--checked' : ''}`}>
                              {taskSelected && <Check size={12} strokeWidth={3} />}
                            </span>
                            <span className="run-eval__drawer-task-name">{task.name}</span>
                          </button>
                        );
                      })}
                    </div>
                  </aside>
                )}
              </div>
            </>
          )}
        </>
      )}

      {tab === 'private' && (
        <div className="run-eval__upload-zone">
          <UploadCloud size={26} />
          <h3>Upload Test Data</h3>
          <p>Drag &amp; drop or click to browse</p>
          <div className="run-eval__format-chips">
            {['CSV', 'JSON', 'JSONL', 'HuggingFace'].map((f) => (
              <span key={f} className="run-eval__chip run-eval__chip--static">
                {f}
              </span>
            ))}
          </div>
        </div>
      )}
    </div>
  );
};

export default DatasetStep;
