//Modelcomparator.tsx
import { useMemo, useState, type FC } from 'react';
import { Search, X, CheckCircle2, GitCompareArrows, ArrowLeft, Trophy } from 'lucide-react';
import { MODELS } from '../RunEvaluation/data';
import type { ModelInfo } from '../RunEvaluation/types';
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

const ModelComparator: FC = () => {
  const [query, setQuery] = useState('');
  const [selectedIds, setSelectedIds] = useState<string[]>([]);
  const [comparing, setComparing] = useState(false);

  const filtered = useMemo(
    () =>
      MODELS.filter(
        (m) =>
          !query ||
          m.name.toLowerCase().includes(query.toLowerCase()) ||
          m.provider.toLowerCase().includes(query.toLowerCase())
      ),
    [query]
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
                <p>No models match your search.</p>
              </div>
            )}
          </div>
        </>
      )}

      {comparing && selectedModels.length > 0 && (
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

























//Modelcomparator.scss
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
    bottom: 22px;
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


















//Sidebar.tsx
import { NavLink } from 'react-router-dom';
import type { FC, ReactNode } from 'react';
import type { LucideIcon } from 'lucide-react';
import {
  Play,
  LayoutGrid,
  Clock,
  Box,
  PlugZap,
  Database,
  FileBarChart,
  Settings,
  CircleHelp,
  ChevronsLeft,
  GitCompareArrows,
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import { toggleSidebar } from '../../store/slices/uiSlice';
import './Sidebar.scss';

interface NavItem {
  to: string;
  label: string;
  icon: LucideIcon;
  end?: boolean;
}

interface NavSection {
  label: string;
  items: NavItem[];
}

const NAV_SECTIONS: NavSection[] = [
  {
    label: 'Overview',
    items: [{ to: '/app', label: 'Dashboard', icon: LayoutGrid, end: true }],
  },
  {
    label: 'Workspace',
    items: [
      { to: '/app/run-evaluation', label: 'New Evaluation', icon: Play },
      { to: '/app/history', label: 'History', icon: Clock },
      { to: '/app/models', label: 'Models', icon: Box },
      { to: '/app/model-comparator', label: 'Model Comparator', icon: GitCompareArrows },
      { to: '/app/providers', label: 'Providers', icon: PlugZap },
      { to: '/app/datasets', label: 'Test Suites', icon: Database },
      { to: '/app/reports', label: 'Reports', icon: FileBarChart },
    ],
  },
];

interface RowProps {
  to?: string;
  label: string;
  icon: ReactNode;
  end?: boolean;
  collapsed: boolean;
  onClick?: () => void;
}

const Row: FC<RowProps> = ({ to, label, icon, end, collapsed, onClick }) => {
  const className = ({ isActive }: { isActive: boolean }) =>
    `app-sidebar__item${isActive ? ' app-sidebar__item--active' : ''}${collapsed ? ' app-sidebar__item--collapsed' : ''}`;

  if (to) {
    return (
      <NavLink to={to} end={end} title={collapsed ? label : undefined} className={className}>
        <span className="app-sidebar__icon">{icon}</span>
        {!collapsed && <span>{label}</span>}
      </NavLink>
    );
  }

  return (
    <button
      type="button"
      title={collapsed ? label : undefined}
      onClick={onClick}
      className={`app-sidebar__item app-sidebar__item--button${collapsed ? ' app-sidebar__item--collapsed' : ''}`}
    >
      <span className="app-sidebar__icon">{icon}</span>
      {!collapsed && <span>{label}</span>}
    </button>
  );
};

const Sidebar: FC = () => {
  const dispatch = useAppDispatch();
  const collapsed = useAppSelector((s) => s.ui.sidebarCollapsed);

  return (
    <aside className={`app-sidebar${collapsed ? ' app-sidebar--collapsed' : ''}`}>
      <button
        type="button"
        className="app-sidebar__collapse"
        onClick={() => dispatch(toggleSidebar())}
        title={collapsed ? 'Expand sidebar' : 'Collapse sidebar'}
        aria-label={collapsed ? 'Expand sidebar' : 'Collapse sidebar'}
      >
        {!collapsed && <span>Menu</span>}
        <ChevronsLeft
          size={15}
          strokeWidth={2}
          className={collapsed ? 'app-sidebar__collapse-ic app-sidebar__collapse-ic--flipped' : 'app-sidebar__collapse-ic'}
        />
      </button>

      <div className="app-sidebar__top">
        <nav className="app-sidebar__nav">
          {NAV_SECTIONS.map((section) => (
            <div className="app-sidebar__section" key={section.label}>
              {!collapsed && <p className="app-sidebar__section-label">{section.label}</p>}
              <ul>
                {section.items.map(({ to, label, icon: Icon, end }) => (
                  <li key={to}>
                    <Row to={to} label={label} end={end} collapsed={collapsed} icon={<Icon size={16.5} strokeWidth={2} />} />
                  </li>
                ))}
              </ul>
            </div>
          ))}
        </nav>
      </div>

      <div className="app-sidebar__footer">
        <Row to="/app/settings" label="Settings" collapsed={collapsed} icon={<Settings size={16.5} strokeWidth={2} />} />
        <Row label="Help" collapsed={collapsed} icon={<CircleHelp size={16.5} strokeWidth={2} />} />
      </div>
    </aside>
  );
};

export default Sidebar;


























//app.tsx
import { useEffect } from 'react';
import { Routes, Route } from 'react-router-dom';
import type { FC } from 'react';
import MainLayout from './layouts/MainLayout';
import WorkspaceLayout from './layouts/WorkspaceLayout';
import Landing from './pages/Landing/Landing';
import SsoLogin from './pages/SsoLogin/SsoLogin';
import Dashboard from './pages/Workspace/Dashboard/Dashboard';
import History from './pages/Workspace/History/History';
import Models from './pages/Workspace/Models/Models';
import ModelComparator from './pages/Workspace/ModelComparator/ModelComparator';
import Providers from './pages/Workspace/Providers/Providers';
import Datasets from './pages/Workspace/Datasets/Datasets';
import Reports from './pages/Workspace/Reports/Reports';
import Settings from './pages/Workspace/Settings/Settings';
import RunEvaluation from './pages/Workspace/RunEvaluation/RunEvaluation';
import AuthGuard from './components/AuthGuard/AuthGuard';
import { useAppSelector } from './store/hooks';

const THEME_STORAGE_KEY = 'semcoeval-theme';

const App: FC = () => {
  const theme = useAppSelector((s) => s.ui.theme);

  useEffect(() => {
    document.documentElement.setAttribute('data-theme', theme);
    window.localStorage.setItem(THEME_STORAGE_KEY, theme);
  }, [theme]);

  return (
    <Routes>
      {/* Public — SSO login/re-auth page, outside the auth gate */}
      <Route path="/sso-login" element={<SsoLogin />} />

      {/* Everything else requires an authenticated session */}
      <Route element={<AuthGuard />}>
        <Route element={<MainLayout />}>
          <Route path="/" element={<Landing />} />

          <Route path="/app" element={<WorkspaceLayout />}>
            <Route index element={<Dashboard />} />
            <Route path="run-evaluation" element={<RunEvaluation />} />
            <Route path="history" element={<History />} />
            <Route path="models" element={<Models />} />
            <Route path="model-comparator" element={<ModelComparator />} />
            <Route path="providers" element={<Providers />} />
            <Route path="datasets" element={<Datasets />} />
            <Route path="reports" element={<Reports />} />
            <Route path="settings" element={<Settings />} />
          </Route>
        </Route>
      </Route>
    </Routes>
  );
};

export default App;
