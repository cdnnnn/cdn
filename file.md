import { useMemo, useState, type FC } from 'react';
import { Search, ListChecks, X, FileBarChart } from 'lucide-react';
import { TEST_SUITES } from '../RunEvaluation/data';

const CATEGORIES = ['All', 'Agents', 'Coding', 'General', 'RAG', 'Finance'];

const Datasets: FC = () => {
  const [query, setQuery] = useState('');
  const [category, setCategory] = useState('All');
  const [panelId, setPanelId] = useState<string | null>(null);

  const filtered = useMemo(() => {
    return TEST_SUITES.filter((d) => {
      if (category !== 'All' && d.category !== category) return false;
      if (query && !d.name.toLowerCase().includes(query.toLowerCase()) && !d.task.toLowerCase().includes(query.toLowerCase())) {
        return false;
      }
      return true;
    });
  }, [query, category]);

  const panelDataset = useMemo(() => TEST_SUITES.find((d) => d.id === panelId) ?? null, [panelId]);

  return (
    <div className="datasets">
      <div className="datasets__header">
        <div>
          <p className="datasets__eyebrow">Workspace</p>
          <h1 className="datasets__title">Test Suites</h1>
          <p className="datasets__subtitle">Browse available benchmarks and inspect their subcategories.</p>
        </div>
      </div>

      <div className="datasets__toolbar">
        <div className="datasets__search">
          <Search size={14} />
          <input
            type="text"
            placeholder="Search test suites…"
            value={query}
            onChange={(e) => setQuery(e.target.value)}
          />
        </div>
        <div className="datasets__categories">
          {CATEGORIES.map((c) => (
            <button
              key={c}
              type="button"
              className={`datasets__chip${category === c ? ' datasets__chip--active' : ''}`}
              onClick={() => setCategory(c)}
            >
              {c}
            </button>
          ))}
        </div>
      </div>

      <div className="datasets__grid">
        {filtered.map((d) => (
          <div key={d.id} className="datasets__card">
            <div className="datasets__card-top">
              <span className="datasets__card-category">{d.category}</span>
              <button
                type="button"
                className="datasets__card-icon-btn"
                title="View subcategories"
                onClick={() => setPanelId(d.id)}
              >
                <ListChecks size={15} />
              </button>
            </div>
            <h3 className="datasets__card-name">{d.name}</h3>
            <p className="datasets__card-desc">{d.description}</p>
            <div className="datasets__card-meta n">
              <span>{d.questions} questions</span>
              <span>{d.subgroups.length} subcategories</span>
              <span>{d.difficulty}</span>
            </div>
            <div className="datasets__card-footer">
              <span>{d.maintainer}</span>
              <span>{d.version}</span>
            </div>
            {d.featured && <span className="datasets__badge">Featured</span>}
          </div>
        ))}

        {filtered.length === 0 && (
          <div className="datasets__empty">
            <FileBarChart size={22} />
            <p>No test suites match your search.</p>
          </div>
        )}
      </div>

      {/* ---------- subgroup detail panel ---------- */}
      {panelDataset && (
        <>
          <div className="datasets__panel-overlay" onClick={() => setPanelId(null)} />
          <aside className="datasets__panel">
            <div className="datasets__panel-head">
              <div>
                <p className="datasets__panel-eyebrow">{panelDataset.category}</p>
                <h3 className="datasets__panel-title">{panelDataset.name}</h3>
              </div>
              <button type="button" className="datasets__panel-close" onClick={() => setPanelId(null)} aria-label="Close">
                <X size={16} />
              </button>
            </div>

            <div className="datasets__panel-toolbar">
              <span>{panelDataset.subgroups.length} subcategories</span>
              <span className="n">{panelDataset.questions} questions total</span>
            </div>

            <div className="datasets__panel-list">
              {panelDataset.subgroups.map((s) => (
                <div key={s.id} className="datasets__panel-item">
                  <span className="datasets__panel-item-name">{s.name}</span>
                  <span className="datasets__panel-item-count n">{s.count}</span>
                </div>
              ))}
            </div>

            <div className="datasets__panel-footer">
              <span>Maintained by {panelDataset.maintainer}</span>
              <span>{panelDataset.version}</span>
            </div>
          </aside>
        </>
      )}
    </div>
  );
};

export default Datasets;
























@import '../../../styles/variables';

.datasets {
  display: flex;
  flex-direction: column;
  gap: 20px;

  &__header {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
  }

  &__eyebrow {
    margin: 0 0 2px;
    font-size: 12px;
    font-weight: 600;
    color: $text-tertiary;
    text-transform: uppercase;
    letter-spacing: 0.04em;
  }

  &__title {
    margin: 0 0 4px;
    font-size: 20px;
    font-weight: 700;
    color: $text-primary;
  }

  &__subtitle {
    margin: 0;
    font-size: 13.5px;
    color: $text-secondary;
  }

  /* ---------- toolbar ---------- */
  &__toolbar {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 16px;
    flex-wrap: wrap;
  }

  &__search {
    display: flex;
    align-items: center;
    gap: 8px;
    border: 1px solid $border-default;
    background: $bg-main;
    border-radius: $radius-sm;
    padding: 9px 13px;
    color: $text-tertiary;
    min-width: 240px;

    input {
      flex: 1;
      border: none;
      background: transparent;
      font-size: 13px;
      color: $text-primary;
      outline: none;

      &::placeholder {
        color: $text-tertiary;
      }
    }
  }

  &__categories {
    display: flex;
    gap: 8px;
    flex-wrap: wrap;
  }

  &__chip {
    border: 1px solid $border-default;
    background: $bg-main;
    border-radius: 999px;
    padding: 7px 14px;
    font-size: 12.5px;
    font-weight: 600;
    color: $text-secondary;
    cursor: pointer;

    &:hover {
      border-color: $border-strong;
    }

    &--active {
      background: $primary-light;
      border-color: $primary-subtle;
      color: $primary;
    }
  }

  /* ---------- grid ---------- */
  &__grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
    gap: 14px;
  }

  &__card {
    position: relative;
    background: $bg-main;
    border: 1px solid $border-default;
    border-radius: $radius-lg;
    padding: 18px 20px;
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  &__card-top {
    display: flex;
    align-items: center;
    justify-content: space-between;
  }

  &__card-category {
    font-size: 10.5px;
    font-weight: 700;
    color: $primary;
    background: $primary-light;
    padding: 3px 9px;
    border-radius: 999px;
    text-transform: uppercase;
    letter-spacing: 0.03em;
  }

  &__card-icon-btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 28px;
    height: 28px;
    border: 1px solid transparent;
    border-radius: $radius-sm;
    background: $bg-subtle;
    color: $text-tertiary;
    cursor: pointer;
    transition: background 0.15s, color 0.15s, border-color 0.15s;

    &:hover {
      background: $primary-light;
      border-color: $primary-subtle;
      color: $primary;
    }
  }

  &__card-name {
    margin: 0;
    font-size: 14.5px;
    font-weight: 700;
    color: $text-primary;
    line-height: 1.35;
  }

  &__card-desc {
    margin: 0;
    font-size: 12px;
    color: $text-secondary;
    line-height: 1.5;
  }

  &__card-meta {
    display: flex;
    gap: 12px;
    font-size: 11.5px;
    color: $text-tertiary;
    margin-top: 2px;
  }

  &__card-footer {
    display: flex;
    justify-content: space-between;
    font-size: 11px;
    color: $text-tertiary;
    padding-top: 8px;
    border-top: 1px solid $border-subtle;
    margin-top: 4px;
  }

  &__badge {
    position: absolute;
    top: -8px;
    right: 16px;
    background: $primary;
    color: #fff;
    font-size: 10px;
    font-weight: 700;
    padding: 3px 10px;
    border-radius: 999px;
    text-transform: uppercase;
    letter-spacing: 0.03em;
  }

  &__empty {
    grid-column: 1 / -1;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 8px;
    padding: 48px 24px;
    color: $text-tertiary;
    font-size: 13px;
  }

  /* ---------- subgroup panel ---------- */
  &__panel-overlay {
    position: fixed;
    inset: 0;
    background: rgba(14, 21, 38, 0.4);
    z-index: 200;
  }

  &__panel {
    position: fixed;
    top: 0;
    right: 0;
    bottom: 0;
    width: 400px;
    max-width: 100vw;
    background: $bg-main;
    border-left: 1px solid $border-default;
    box-shadow: $shadow-xl;
    z-index: 201;
    display: flex;
    flex-direction: column;
  }

  &__panel-head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
    padding: 22px 24px 18px;
    border-bottom: 1px solid $border-subtle;
    background: $bg-subtle;
  }

  &__panel-eyebrow {
    margin: 0 0 4px;
    font-size: 11px;
    font-weight: 700;
    color: $primary;
    text-transform: uppercase;
    letter-spacing: 0.05em;
  }

  &__panel-title {
    margin: 0;
    font-size: 16px;
    font-weight: 700;
    color: $text-primary;
    line-height: 1.35;
  }

  &__panel-close {
    flex-shrink: 0;
    width: 30px;
    height: 30px;
    display: flex;
    align-items: center;
    justify-content: center;
    border: 1px solid $border-default;
    background: $bg-main;
    border-radius: $radius-sm;
    color: $text-secondary;
    cursor: pointer;

    &:hover {
      background: $bg-inset;
      border-color: $border-strong;
    }
  }

  &__panel-toolbar {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 14px 24px;
    font-size: 12.5px;
    font-weight: 600;
    color: $text-secondary;
    border-bottom: 1px solid $border-subtle;
  }

  &__panel-list {
    flex: 1;
    overflow-y: auto;
    padding: 10px 14px;
  }

  &__panel-item {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
    padding: 13px 14px;
    margin-bottom: 4px;
    border: 1px solid transparent;
    border-radius: $radius-md;

    &:hover {
      background: $bg-subtle;
      border-color: $border-subtle;
    }
  }

  &__panel-item-name {
    font-size: 13.5px;
    font-weight: 500;
    color: $text-primary;
  }

  &__panel-item-count {
    font-size: 12px;
    font-weight: 600;
    color: $text-tertiary;
    background: $bg-inset;
    padding: 2px 9px;
    border-radius: 999px;
  }

  &__panel-footer {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 14px 24px;
    font-size: 12px;
    color: $text-tertiary;
    border-top: 1px solid $border-subtle;
    background: $bg-subtle;
  }
}
