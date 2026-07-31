import { useEffect, useMemo, useState, type FC } from 'react';
import { UploadCloud, Check, ListChecks, X } from 'lucide-react';
import { TEST_SUITES } from '../data';
import type { EvalTypeId } from '../types';

interface Props {
  evalType: EvalTypeId | null;
  selected: string | null;
  onSelect: (id: string) => void;
  subgroupSelections: string[];
  onSubgroupsChange: (ids: string[]) => void;
}

const CATEGORIES = ['All', 'Agents', 'Coding', 'General', 'RAG', 'Finance'];

const DatasetStep: FC<Props> = ({ evalType, selected, onSelect, subgroupSelections, onSubgroupsChange }) => {
  const [tab, setTab] = useState<'official' | 'private'>('official');
  const [category, setCategory] = useState('All');
  const [panelDatasetId, setPanelDatasetId] = useState<string | null>(null);
  const [panelSelection, setPanelSelection] = useState<string[]>([]);

  const filtered = useMemo(
    () => TEST_SUITES.filter((t) => category === 'All' || t.category === category),
    [category]
  );

  const panelDataset = useMemo(() => TEST_SUITES.find((t) => t.id === panelDatasetId) ?? null, [panelDatasetId]);

  const openPanel = (datasetId: string) => {
    const dataset = TEST_SUITES.find((t) => t.id === datasetId);
    // If this dataset is already selected, seed with its current subgroup selection.
    // Otherwise start fresh with every subgroup checked by default.
    const existing = selected === datasetId ? subgroupSelections : [];
    const seed = existing.length > 0 ? existing : (dataset?.subgroups.map((s) => s.id) ?? []);
    setPanelSelection(seed);
    setPanelDatasetId(datasetId);
  };

  const closePanel = () => {
    setPanelDatasetId(null);
  };

  const toggleSubgroup = (id: string) => {
    setPanelSelection((prev) => (prev.includes(id) ? prev.filter((x) => x !== id) : [...prev, id]));
  };

  const selectAll = () => {
    if (!panelDataset) return;
    setPanelSelection(panelDataset.subgroups.map((s) => s.id));
  };

  const selectNone = () => setPanelSelection([]);

  const applyPanel = () => {
    if (!panelDatasetId) return;
    onSelect(panelDatasetId);
    onSubgroupsChange(panelSelection);
    closePanel();
  };

  // close on escape
  useEffect(() => {
    if (!panelDatasetId) return;
    const handler = (e: KeyboardEvent) => {
      if (e.key === 'Escape') closePanel();
    };
    document.addEventListener('keydown', handler);
    return () => document.removeEventListener('keydown', handler);
  }, [panelDatasetId]);

  const selectedQuestionCount = (dataset: (typeof TEST_SUITES)[number], subgroupIds: string[]) =>
    dataset.subgroups.filter((s) => subgroupIds.includes(s.id)).reduce((sum, s) => sum + s.count, 0);

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
          <div className="run-eval__category-filters">
            {CATEGORIES.map((c) => (
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

          <div className="run-eval__dataset-grid">
            {filtered.map((d) => {
              const isSelected = selected === d.id;
              const recommended = evalType ? d.recommendedFor.includes(evalType) : false;
              const activeSubgroupCount = isSelected && subgroupSelections.length > 0 ? subgroupSelections.length : d.subgroups.length;
              const activeQuestionCount = isSelected && subgroupSelections.length > 0
                ? selectedQuestionCount(d, subgroupSelections)
                : d.questions;

              return (
                <div
                  key={d.id}
                  role="button"
                  tabIndex={0}
                  className={`run-eval__dataset-card${isSelected ? ' run-eval__dataset-card--selected' : ''}`}
                  onClick={() => onSelect(d.id)}
                  onKeyDown={(e) => {
                    if (e.key === 'Enter' || e.key === ' ') {
                      e.preventDefault();
                      onSelect(d.id);
                    }
                  }}
                >
                  <div className="run-eval__dataset-top">
                    <span className="run-eval__dataset-name">{d.name}</span>
                    <span className="run-eval__dataset-top-actions">
                      <button
                        type="button"
                        className="run-eval__dataset-subgroups-btn"
                        title="Choose subcategories"
                        onClick={(e) => {
                          e.stopPropagation();
                          openPanel(d.id);
                        }}
                      >
                        <ListChecks size={15} />
                      </button>
                      {isSelected && (
                        <span className="run-eval__type-check">
                          <Check size={12} strokeWidth={2.75} />
                        </span>
                      )}
                    </span>
                  </div>
                  <p className="run-eval__dataset-desc">{d.description}</p>
                  <div className="run-eval__dataset-meta n">
                    <span>{activeQuestionCount} questions</span>
                    <span>{activeSubgroupCount}/{d.subgroups.length} subcategories</span>
                    <span>{d.difficulty}</span>
                  </div>
                  {recommended && <span className="run-eval__badge run-eval__badge--soft">Recommended</span>}
                </div>
              );
            })}
          </div>
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

      {/* ---------- subgroup slide-over panel ---------- */}
      {panelDataset && (
        <>
          <div className="run-eval__panel-overlay" onClick={closePanel} />
          <aside className="run-eval__panel">
            <div className="run-eval__panel-head">
              <div>
                <p className="run-eval__panel-eyebrow">{panelDataset.category}</p>
                <h3 className="run-eval__panel-title">{panelDataset.name}</h3>
              </div>
              <button type="button" className="run-eval__panel-close" onClick={closePanel} aria-label="Close">
                <X size={16} />
              </button>
            </div>

            <div className="run-eval__panel-toolbar">
              <span>{panelSelection.length} of {panelDataset.subgroups.length} selected</span>
              <span className="run-eval__panel-toolbar-actions">
                <button type="button" className="run-eval__link" onClick={selectAll}>Select all</button>
                <button type="button" className="run-eval__link" onClick={selectNone}>Clear</button>
              </span>
            </div>

            <div className="run-eval__panel-list">
              {panelDataset.subgroups.map((s) => {
                const checked = panelSelection.includes(s.id);
                return (
                  <label key={s.id} className="run-eval__panel-item">
                    <span className="run-eval__panel-item-main">
                      <input type="checkbox" checked={checked} onChange={() => toggleSubgroup(s.id)} />
                      <span className="run-eval__panel-item-name">{s.name}</span>
                    </span>
                    <span className="run-eval__panel-item-count n">{s.count}</span>
                  </label>
                );
              })}
            </div>

            <div className="run-eval__panel-footer">
              <span className="run-eval__panel-footer-total n">
                {selectedQuestionCount(panelDataset, panelSelection)} questions
              </span>
              <button
                type="button"
                className="run-eval__panel-apply"
                disabled={panelSelection.length === 0}
                onClick={applyPanel}
              >
                Apply
              </button>
            </div>
          </aside>
        </>
      )}
    </div>
  );
};

export default DatasetStep;
