//variable.scss
// ============================================================
// SemcoEval — design tokens
// Base: 1rem = 16px
//
// Color / shadow tokens are aliases for CSS custom properties.
// The actual light + dark values live in global.scss (:root and
// [data-theme='dark']), so every component that already uses
// $primary, $bg-main, $shadow-md, etc. gets dark mode automatically —
// nothing here needs to change per-component.
// ============================================================

// ---------- Brand / primary ----------
$primary: var(--primary);
$primary-hover: var(--primary-hover);
$primary-light: var(--primary-light);
$primary-subtle: var(--primary-subtle);

// ---------- Secondary accent (used by violet badges/pills across pages) ----------
$violet: var(--violet);
$violet-light: var(--violet-light);

// ---------- Surfaces ----------
$bg-page: var(--bg-page);
$bg-subtle: var(--bg-subtle);
$bg-inset: var(--bg-inset);
$bg-main: var(--bg-main);
$bg-header-glass: var(--bg-header-glass);

// ---------- Borders ----------
$border-default: var(--border-default);
$border-subtle: var(--border-subtle);
$border-strong: var(--border-strong);

// ---------- Text ----------
$text-primary: var(--text-primary);
$text-secondary: var(--text-secondary);
$text-tertiary: var(--text-tertiary);

// ---------- Status ----------
$success: var(--success);
$success-subtle: var(--success-subtle);
$warning: var(--warning);
$warning-subtle: var(--warning-subtle);
$danger: var(--danger);
$danger-subtle: var(--danger-subtle);

// ---------- On-fill text (mode-independent) ----------
// Text sitting on a saturated fill like $primary or $success stays white in
// both themes since those fills are always bright/saturated regardless of
// mode — this documents that intent instead of leaving raw #fff scattered
// through component files.
$on-primary: #fff;

// ---------- Shadows ----------
$shadow-xs: var(--shadow-xs);
$shadow-sm: var(--shadow-sm);
$shadow-md: var(--shadow-md);
$shadow-lg: var(--shadow-lg);
$shadow-xl: var(--shadow-xl);

// ---------- Radius (mode-independent) ----------
$radius-sm: 0.375rem;
$radius-md: 0.5rem;
$radius-lg: 0.75rem;
$radius-xl: 1rem;

// ---------- Typography (mode-independent) ----------
$font-display: 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
$font-body: 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
$font-mono: 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;

// ---------- Layout (mode-independent) ----------
$header-height: 60px;
$footer-height: 30px;
$sidebar-width: 240px;

// ---------- Z-index (mode-independent) ----------
$z-header: 100;
$z-footer: 100;
$z-sidebar: 90;













//App.tsx
import { useEffect } from 'react';
import { Routes, Route } from 'react-router-dom';
import type { FC } from 'react';
import MainLayout from './layouts/MainLayout';
import WorkspaceLayout from './layouts/WorkspaceLayout';
import Landing from './pages/Landing/Landing';
import Dashboard from './pages/Workspace/Dashboard/Dashboard';
import History from './pages/Workspace/History/History';
import Models from './pages/Workspace/Models/Models';
import Providers from './pages/Workspace/Providers/Providers';
import Datasets from './pages/Workspace/Datasets/Datasets';
import Reports from './pages/Workspace/Reports/Reports';
import Settings from './pages/Workspace/Settings/Settings';
import RunEvaluation from './pages/Workspace/RunEvaluation/RunEvaluation';
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
      <Route element={<MainLayout />}>
        <Route path="/" element={<Landing />} />

        <Route path="/app" element={<WorkspaceLayout />}>
          <Route index element={<Dashboard />} />
          <Route path="run-evaluation" element={<RunEvaluation />} />
          <Route path="history" element={<History />} />
          <Route path="models" element={<Models />} />
          <Route path="providers" element={<Providers />} />
          <Route path="datasets" element={<Datasets />} />
          <Route path="reports" element={<Reports />} />
          <Route path="settings" element={<Settings />} />
        </Route>
      </Route>
    </Routes>
  );
};

export default App;

















//Datasets.scss
@use '../../../styles/variables' as *;

.datasets-page {
  display: flex;
  flex-direction: column;
  gap: 16px;
  // Caps the page at a comfortable working width and centers it, so on very
  // wide viewports (1800px+) the sidebar/detail columns don't stretch into an
  // unusably wide layout — the extra space becomes gutters instead.
  max-width: 1680px;
  margin-left: auto;
  margin-right: auto;
  // Same flex chain as History/Models/Providers: main-layout__body / workspace-layout /
  // workspace-layout__content already resolve to the exact viewport height,
  // so height: 100% fills it precisely. workspace-layout__content also carries
  // a 3rem bottom padding, which would leave a gap below this page — pull most
  // of that back in, keeping a small 0.75rem breathing-room strip at the bottom.
  height: calc(100% + 3rem - 0.75rem);
  margin-bottom: calc(-3rem + 0.75rem);
  min-height: 0;

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

  &__header-right {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 10px;
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

  &__header-meta {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-secondary;
    background: $bg-subtle;
    border: 1px solid $border-subtle;
    border-radius: 999px;
    padding: 7px 13px;
    white-space: nowrap;
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

  &__btn {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    font-family: $font-body;
    font-size: 0.8125rem;
    font-weight: 600;
    padding: 9px 14px;
    border-radius: 8px;
    border: 1px solid transparent;
    cursor: pointer;
    white-space: nowrap;
    transition: background 0.14s ease, border-color 0.14s ease;

    &--primary {
      background: $primary;
      border-color: $primary;
      color: #fff;

      &:hover {
        background: $primary-hover;
        border-color: $primary-hover;
      }
    }
  }

  /* ---------- filters ---------- */
  &__filters {
    flex-shrink: 0;
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 12px;
  }

  &__search {
    display: flex;
    align-items: center;
    gap: 9px;
    width: 280px;
    max-width: 100%;
    border: 1px solid $border-default;
    border-radius: 10px;
    padding: 9px 12px;
    background: $bg-main;
    color: $text-tertiary;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &:focus-within {
      border-color: $primary;
      box-shadow: 0 0 0 3px $primary-light;
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
    transition: background 0.14s ease, color 0.14s ease;

    &:hover {
      background: $border-default;
      color: $text-primary;
    }
  }

  /* ---------- segmented category bar ---------- */
  &__seg {
    flex-shrink: 0;
    display: inline-flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 2px;
    padding: 3px;
    border: 1px solid $border-subtle;
    border-radius: 11px;
    background: $bg-subtle;
  }

  &__seg-item {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-family: $font-body;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-tertiary;
    background: transparent;
    border: none;
    border-radius: 8px;
    padding: 7px 12px;
    cursor: pointer;
    transition: background 0.14s ease, color 0.14s ease, box-shadow 0.14s ease;

    svg {
      opacity: 0.8;
    }

    &:hover {
      color: $text-primary;
    }

    &--active {
      background: $bg-main;
      color: $primary;
      box-shadow: $shadow-xs;

      svg {
        opacity: 1;
      }
    }
  }

  /* ---------- master-detail layout ---------- */
  &__layout {
    flex: 1;
    display: grid;
    grid-template-columns: 340px 1fr;
    gap: 16px;
    min-height: 0;
    overflow: hidden;
  }

  /* ---------- sidebar list ---------- */
  &__sidebar {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 16px;
    box-shadow: $shadow-xs;
    height: 100%;
    min-height: 0;
    display: flex;
    flex-direction: column;
    overflow: hidden;
  }

  &__sidebar-list {
    flex: 1;
    display: flex;
    flex-direction: column;
    min-height: 0;
    overflow-y: auto;
  }

  &__item {
    display: flex;
    flex-direction: column;
    gap: 6px;
    padding: 14px 16px;
    text-align: left;
    border: none;
    border-bottom: 1px solid $border-subtle;
    border-left: 3px solid transparent;
    background: $bg-main;
    width: 100%;
    cursor: pointer;
    transition: background 0.12s ease, border-color 0.12s ease;

    &:last-child {
      border-bottom: none;
    }

    &:hover {
      background: $bg-subtle;
    }

    &--active {
      background: $primary-light;
      border-left-color: $primary;

      .datasets-page__item-name {
        color: $primary;
      }
    }
  }

  &__item-top {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__item-name {
    font-size: 0.84375rem;
    font-weight: 600;
    color: $text-primary;
    line-height: 1.35;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__item-featured {
    flex-shrink: 0;
    display: inline-flex;
    color: $warning;
  }

  &__item-meta {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
    font-size: 0.75rem;
    color: $text-tertiary;
  }

  &__item-questions {
    flex-shrink: 0;
  }

  /* ---------- tags (shared by sidebar + detail) ---------- */
  &__tag {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 4px;
    font-size: 0.625rem;
    font-weight: 600;
    border-radius: 999px;
    padding: 2px 8px;

    &--blue {
      color: $primary;
      background: $primary-light;
    }

    &--violet {
      color: $violet;
      background: $violet-light;
    }

    &--amber {
      color: $warning;
      background: $warning-subtle;
    }

    &--jade {
      color: $success;
      background: $success-subtle;
    }

    &--rose {
      color: $danger;
      background: $danger-subtle;
    }

    &--featured {
      color: $warning;
      background: $warning-subtle;
    }
  }

  /* ---------- detail panel ---------- */
  &__detail {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 16px;
    box-shadow: $shadow-xs;
    padding: 26px 28px;
    height: 100%;
    min-height: 0;
    overflow-y: auto;
  }

  &__detail-head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
    padding-bottom: 18px;
    margin-bottom: 16px;
    border-bottom: 1px solid $border-subtle;
    position: sticky;
    top: -26px;
    padding-top: 26px;
    margin-top: -26px;
    background: $bg-main;
    z-index: 1;
  }

  &__detail-head-left {
    display: flex;
    flex-direction: column;
    gap: 8px;
    min-width: 0;
  }

  &__detail-tags {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;

    .datasets-page__tag {
      font-size: 0.6875rem;
      padding: 3px 10px;
    }
  }

  &__detail-name {
    font-size: 1.25rem;
    font-weight: 800;
    letter-spacing: -0.01em;
    line-height: 1.3;
    color: $text-primary;
  }

  &__detail-desc {
    font-size: 0.84375rem;
    line-height: 1.6;
    color: $text-secondary;
    margin-bottom: 24px;
  }

  /* ---------- stat cards ---------- */
  &__section-title {
    font-size: 0.8125rem;
    font-weight: 700;
    color: $text-primary;
    margin-bottom: 12px;
  }

  &__stat-row {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 12px;
    margin-bottom: 24px;
  }

  &__stat-card {
    background: $bg-subtle;
    border-radius: 12px;
    padding: 14px 16px;
    display: flex;
    flex-direction: column;
    gap: 4px;
    min-width: 0;
  }

  &__stat-card-label {
    font-size: 0.6875rem;
    font-weight: 600;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $text-tertiary;
  }

  &__stat-card-value {
    font-size: 1.25rem;
    font-weight: 800;
    color: $text-primary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;

    &--sm {
      font-size: 1rem;
      font-weight: 700;
    }
  }

  /* ---------- meta list ---------- */
  &__meta-list {
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  &__meta-item {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 0.8125rem;
    color: $text-secondary;

    svg {
      flex-shrink: 0;
      color: $text-tertiary;
    }
  }

  /* ---------- empty state ---------- */
  &__empty {
    flex-shrink: 0;
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

  /* ---------- responsive ---------- */
  @media (max-width: 900px) {
    &__layout {
      grid-template-columns: 1fr;
      grid-template-rows: 16rem 1fr;
      overflow: visible;
    }

    &__sidebar,
    &__detail {
      height: auto;
      max-height: 22rem;
    }

    &__stat-row {
      grid-template-columns: repeat(2, 1fr);
    }
  }

  @media (max-width: 520px) {
    &__detail {
      padding: 18px 16px;
    }

    &__detail-head {
      flex-direction: column;
    }

    &__stat-row {
      grid-template-columns: 1fr;
    }
  }
}
















//Datasets.tsx
import { useMemo, useState, type FC } from 'react';
import { useNavigate } from 'react-router-dom';
import {
  Upload,
  Database,
  Star,
  LayoutGrid,
  Bot,
  Code2,
  BookOpen,
  Search,
  DollarSign,
  HeartPulse,
  Globe,
  User,
  X,
} from 'lucide-react';
import { TEST_SUITES } from '../RunEvaluation/data';
import './Datasets.scss';

const CATEGORIES = [
  { value: 'All', icon: LayoutGrid },
  { value: 'Agents', icon: Bot },
  { value: 'Coding', icon: Code2 },
  { value: 'General', icon: BookOpen },
  { value: 'RAG', icon: Search },
  { value: 'Finance', icon: DollarSign },
  { value: 'Healthcare', icon: HeartPulse },
] as const;

const CATEGORY_TINTS: Record<string, 'blue' | 'violet' | 'amber' | 'jade' | 'rose'> = {
  Agents: 'violet',
  Coding: 'blue',
  General: 'jade',
  RAG: 'blue',
  Finance: 'amber',
  Healthcare: 'rose',
};

const DIFFICULTY_TINTS: Record<string, 'blue' | 'violet' | 'amber' | 'jade' | 'rose'> = {
  Medium: 'jade',
  High: 'blue',
  Advanced: 'violet',
  Expert: 'rose',
};

const Datasets: FC = () => {
  const navigate = useNavigate();
  const [query, setQuery] = useState('');
  const [category, setCategory] = useState('All');
  const [selectedId, setSelectedId] = useState<string | null>(TEST_SUITES[0]?.id ?? null);

  const filtered = useMemo(
    () =>
      TEST_SUITES.filter((d) => {
        if (category !== 'All' && d.category !== category) return false;
        if (query && !d.name.toLowerCase().includes(query.toLowerCase()) && !d.description.toLowerCase().includes(query.toLowerCase())) {
          return false;
        }
        return true;
      }),
    [query, category]
  );

  const selected = useMemo(
    () => filtered.find((d) => d.id === selectedId) ?? filtered[0] ?? null,
    [filtered, selectedId]
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
            {TEST_SUITES.length} suites available
          </div>
          <button type="button" className="datasets-page__btn datasets-page__btn--primary" onClick={() => navigate('/app/run-evaluation')}>
            <Upload size={14} strokeWidth={2.25} /> Upload
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
          {CATEGORIES.map((c) => (
            <button
              key={c.value}
              type="button"
              className={`datasets-page__seg-item${category === c.value ? ' datasets-page__seg-item--active' : ''}`}
              onClick={() => setCategory(c.value)}
            >
              <c.icon size={13} strokeWidth={2.25} />
              {c.value}
            </button>
          ))}
        </div>
      </div>

      {filtered.length === 0 ? (
        <div className="datasets-page__empty">
          <Database size={22} />
          <p>No test suites in this category.</p>
        </div>
      ) : (
        <div className="datasets-page__layout">
          <div className="datasets-page__sidebar">
            <div className="datasets-page__sidebar-list">
              {filtered.map((d) => {
                const catTint = CATEGORY_TINTS[d.category] ?? 'blue';
                const isActive = selected?.id === d.id;
                return (
                  <button
                    type="button"
                    key={d.id}
                    className={`datasets-page__item${isActive ? ' datasets-page__item--active' : ''}`}
                    onClick={() => setSelectedId(d.id)}
                  >
                    <div className="datasets-page__item-top">
                      <span className="datasets-page__item-name">{d.name}</span>
                      {d.featured && (
                        <span className="datasets-page__item-featured">
                          <Star size={11} strokeWidth={2.5} />
                        </span>
                      )}
                    </div>
                    <div className="datasets-page__item-meta">
                      <span className={`datasets-page__tag datasets-page__tag--${catTint}`}>{d.category}</span>
                      <span className="datasets-page__item-questions n">{d.questions.toLocaleString()} Qs</span>
                    </div>
                  </button>
                );
              })}
            </div>
          </div>

          {selected && (
            <div className="datasets-page__detail">
              {(() => {
                const catIcon = CATEGORIES.find((c) => c.value === selected.category)?.icon ?? Database;
                const catTint = CATEGORY_TINTS[selected.category] ?? 'blue';
                const diffTint = DIFFICULTY_TINTS[selected.difficulty] ?? 'blue';
                const CatIcon = catIcon;

                return (
                  <>
                    <div className="datasets-page__detail-head">
                      <div className="datasets-page__detail-head-left">
                        <div className="datasets-page__detail-tags">
                          <span className={`datasets-page__tag datasets-page__tag--${catTint}`}>
                            <CatIcon size={11} strokeWidth={2.5} />
                            {selected.category}
                          </span>
                          <span className={`datasets-page__tag datasets-page__tag--${diffTint}`}>{selected.difficulty}</span>
                          {selected.featured && (
                            <span className="datasets-page__tag datasets-page__tag--featured">
                              <Star size={11} strokeWidth={2.5} />
                              Featured
                            </span>
                          )}
                        </div>
                        <h2 className="datasets-page__detail-name">{selected.name}</h2>
                      </div>

                      <button type="button" className="datasets-page__btn datasets-page__btn--primary" onClick={() => navigate('/app/run-evaluation')}>
                        Use in Evaluation
                      </button>
                    </div>

                    <p className="datasets-page__detail-desc">{selected.description}</p>

                    <div className="datasets-page__stat-row">
                      <div className="datasets-page__stat-card">
                        <span className="datasets-page__stat-card-label">Questions</span>
                        <span className="datasets-page__stat-card-value n">{selected.questions.toLocaleString()}</span>
                      </div>
                      <div className="datasets-page__stat-card">
                        <span className="datasets-page__stat-card-label">Version</span>
                        <span className="datasets-page__stat-card-value datasets-page__stat-card-value--sm n">{selected.version}</span>
                      </div>
                      <div className="datasets-page__stat-card">
                        <span className="datasets-page__stat-card-label">Task</span>
                        <span className="datasets-page__stat-card-value datasets-page__stat-card-value--sm">{selected.task}</span>
                      </div>
                    </div>

                    <p className="datasets-page__section-title">Details</p>
                    <div className="datasets-page__meta-list">
                      <span className="datasets-page__meta-item">
                        <Globe size={13} /> {selected.language}
                      </span>
                      <span className="datasets-page__meta-item">
                        <User size={13} /> Maintained by {selected.maintainer}
                      </span>
                    </div>
                  </>
                );
              })()}
            </div>
          )}
        </div>
      )}
    </div>
  );
};

export default Datasets;
















//global.scss
@use './variables' as *;

:root {
  color-scheme: light;

  --primary: #1428a0;
  --primary-hover: #1d37c9;
  --primary-light: #eef1fe;
  --primary-subtle: #e2e7fc;

  --violet: #7c3aed;
  --violet-light: #f3e8ff;

  --bg-page: #f6f7f9;
  --bg-subtle: #f3f5f8;
  --bg-inset: #edf0f4;
  --bg-main: #ffffff;
  --bg-header-glass: rgba(255, 255, 255, 0.88);

  --border-default: #dce0e7;
  --border-subtle: #e9ecf1;
  --border-strong: #c7cdd8;

  --text-primary: #0e1526;
  --text-secondary: #46506b;
  --text-tertiary: #7a8399;

  --success: #0f7a5a;
  --success-subtle: #e4f4ee;
  --warning: #b7791f;
  --warning-subtle: #fdf3e0;
  --danger: #c0303b;
  --danger-subtle: #fcebec;

  --shadow-xs: 0 0.0625rem 0.125rem rgba(14, 21, 38, 0.04);
  --shadow-sm: 0 0.0625rem 0.125rem rgba(14, 21, 38, 0.05);
  --shadow-md: 0 0.125rem 0.25rem rgba(14, 21, 38, 0.05), 0 0.5rem 1.25rem -0.75rem rgba(14, 21, 38, 0.16);
  --shadow-lg: 0 0.25rem 0.5rem rgba(14, 21, 38, 0.05), 0 1.125rem 2.75rem -1.375rem rgba(14, 21, 38, 0.24);
  --shadow-xl: 0 1.75rem 4.375rem -1.875rem rgba(14, 21, 38, 0.34);
}

[data-theme='dark'] {
  color-scheme: dark;

  --primary: #6c8cff;
  --primary-hover: #85a3ff;
  --primary-light: #141c38;
  --primary-subtle: #1d2748;

  --violet: #c4a6ff;
  --violet-light: #1c1733;

  // True-black page with near-black cards sitting just barely above it —
  // bg-subtle/bg-inset step up in small, even increments so table headers,
  // input wells, and hover states stay readable without the elevation
  // jumps feeling abrupt against a pure-black base.
  --bg-page: #000000;
  --bg-main: #0d0d0d;
  --bg-subtle: #131313;
  --bg-inset: #1a1a1a;
  --bg-header-glass: rgba(0, 0, 0, 0.75);

  // Borders carry more of the depth signal here than shadows do, since a
  // black shadow is invisible against a black page — so these are a touch
  // brighter/more frequent than a typical dark theme would need.
  --border-default: #292929;
  --border-subtle: #1c1c1c;
  --border-strong: #3d3d3d;

  --text-primary: #f5f5f5;
  --text-secondary: #a8a8a8;
  --text-tertiary: #767676;

  --success: #34d399;
  --success-subtle: #0c1f18;
  --warning: #fbbf4a;
  --warning-subtle: #241c0c;
  --danger: #fb7185;
  --danger-subtle: #2a1014;

  // Drop shadows barely register on true black, so depth here leans on the
  // inset top highlight (a hairline of light catching each card's top edge)
  // more than usual — bumped up from the previous dark palette so cards and
  // dropdowns still read as "lifted" rather than flat against the page.
  --shadow-xs: 0 1px 2px rgba(0, 0, 0, 0.5), inset 0 1px 0 rgba(255, 255, 255, 0.04);
  --shadow-sm: 0 1px 3px rgba(0, 0, 0, 0.55), inset 0 1px 0 rgba(255, 255, 255, 0.05);
  --shadow-md: 0 0.5rem 1.75rem -0.75rem rgba(0, 0, 0, 0.7), inset 0 1px 0 rgba(255, 255, 255, 0.06);
  --shadow-lg: 0 1.125rem 3rem -1.25rem rgba(0, 0, 0, 0.75), inset 0 1px 0 rgba(255, 255, 255, 0.07);
  --shadow-xl: 0 1.75rem 5rem -1.5rem rgba(0, 0, 0, 0.85), inset 0 1px 0 rgba(255, 255, 255, 0.08);
}

* ,
*::before,
*::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

html {
  -webkit-font-smoothing: antialiased;
  scroll-behavior: smooth;
  font-size: 100%; // 1rem = 16px, respects user browser settings
}

body {
  font-family: $font-body;
  background: $bg-main;
  color: $text-primary;
  font-size: 1.0625rem;
  line-height: 1.55;
  transition: background-color 0.16s ease, color 0.16s ease;
}

a {
  color: inherit;
}

button,
input,
select,
textarea {
  font-family: inherit;
}

h1,
h2,
h3 {
  font-family: $font-display;
  letter-spacing: -0.025em;
  line-height: 1.12;
  font-weight: 700;
}

/* numbers hold their columns without a monospaced face */
.n {
  font-variant-numeric: tabular-nums;
  font-feature-settings: 'tnum' 1, 'lnum' 1;
}

:where(a, button, input, select, textarea, [tabindex]):focus-visible {
  outline: 0.125rem solid $primary;
  outline-offset: 0.125rem;
  border-radius: 0.25rem;
}

::selection {
  background: $primary-subtle;
  color: $text-primary;
}

@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}

























//Header.scss
@use '../../styles/variables' as *;

.app-header {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  z-index: $z-header;
  height: 60px;
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.75rem;
  padding: 0 1.125rem;
  background: $bg-header-glass;
  backdrop-filter: blur(0.75rem);
  border-bottom: 1px solid $border-subtle;

  &__brand {
    display: flex;
    align-items: center;
    gap: 0.625rem;
    text-decoration: none;
    color: $text-primary;
    flex-shrink: 0;
  }

  &__brand-mark {
    width: 32px;
    height: 32px;
    border-radius: 0.5rem;
    background: linear-gradient(155deg, $primary 0%, $primary-hover 100%);
    color: #fff;
    display: grid;
    place-items: center;
    box-shadow: $shadow-xs, inset 0 0 0 0.0625rem rgba(255, 255, 255, 0.14);
  }

  &__brand-name {
    font-family: $font-display;
    font-weight: 700;
    font-size: 1.0925rem;
    letter-spacing: -0.02em;
    white-space: nowrap;
  }

  &__right {
    display: flex;
    align-items: center;
    gap: 0.625rem;
    flex-shrink: 0;
  }

  /* ---------- theme toggle ---------- */
  &__theme-toggle {
    position: relative;
    flex-shrink: 0;
    width: 52px;
    height: 28px;
    border-radius: 999px;
    border: 1px solid $border-default;
    background: $bg-subtle;
    box-shadow: inset 0 0.0625rem 0.125rem rgba(0, 0, 0, 0.12), inset 0 -0.0625rem 0 rgba(255, 255, 255, 0.06);
    cursor: pointer;
    display: block;
    padding: 0;
    transition: border-color 0.14s ease;

    &:hover {
      border-color: $border-strong;
    }

    &:focus-visible {
      outline: none;
      border-color: $primary;
      box-shadow: inset 0 0.0625rem 0.125rem rgba(0, 0, 0, 0.12), inset 0 -0.0625rem 0 rgba(255, 255, 255, 0.06),
        0 0 0 0.1875rem $primary-subtle;
    }
  }

  &__theme-static {
    position: absolute;
    top: 50%;
    transform: translateY(-50%);
    color: $text-tertiary;
    opacity: 0.7;
    pointer-events: none;

    &--sun {
      left: 7px;
    }

    &--moon {
      right: 7px;
    }
  }

  &__theme-knob {
    position: absolute;
    top: 3px;
    left: 3px;
    width: 20px;
    height: 20px;
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    color: #fff;
    transition: transform 0.18s ease, background 0.18s ease, box-shadow 0.18s ease;

    &::before {
      content: '';
      position: absolute;
      inset: 0.0625rem 0.0625rem auto 0.0625rem;
      height: 45%;
      border-radius: 999px 999px 40% 40%;
      background: linear-gradient(to bottom, rgba(255, 255, 255, 0.55), rgba(255, 255, 255, 0));
      pointer-events: none;
    }

    &--light {
      background: linear-gradient(155deg, #ffcf6b 0%, $warning 100%);
      box-shadow: 0 0.125rem 0.375rem rgba(183, 121, 31, 0.45), 0 0 0 0.0625rem rgba(255, 255, 255, 0.25) inset;
    }

    &--dark {
      background: linear-gradient(155deg, $primary-hover 0%, $primary 100%);
      box-shadow: 0 0.125rem 0.5rem rgba(87, 108, 250, 0.55), 0 0 0 0.0625rem rgba(255, 255, 255, 0.18) inset;
      transform: translateX(24px);
    }
  }

  &__user {
    position: relative;
    flex-shrink: 0;
  }

  &__avatar {
    width: 34px;
    height: 34px;
    border-radius: 50%;
    border: 1px solid $border-default;
    background: $primary-light;
    color: $primary;
    font-family: $font-display;
    font-size: 0.8125rem;
    font-weight: 700;
    letter-spacing: 0.01em;
    display: grid;
    place-items: center;
    cursor: pointer;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &:hover {
      border-color: $primary;
    }

    &--open,
    &:focus-visible {
      outline: none;
      border-color: $primary;
      box-shadow: 0 0 0 0.1875rem $primary-subtle;
    }
  }

  &__dropdown {
    position: absolute;
    top: calc(100% + 0.625rem);
    right: 0;
    width: 232px;
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 0.75rem;
    box-shadow: $shadow-lg;
    padding: 0.625rem;
    z-index: $z-header;
  }

  &__drop-user {
    display: flex;
    align-items: center;
    gap: 0.625rem;
    padding: 0.375rem 0.375rem 0.625rem;
  }

  &__drop-avatar {
    width: 36px;
    height: 36px;
    flex-shrink: 0;
    border-radius: 50%;
    background: $primary-light;
    color: $primary;
    font-family: $font-display;
    font-size: 0.84375rem;
    font-weight: 700;
    display: grid;
    place-items: center;
  }

  &__drop-info {
    min-width: 0;
  }

  &__drop-name {
    font-size: 0.90625rem;
    font-weight: 600;
    color: $text-primary;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  &__drop-role {
    font-size: 0.78125rem;
    color: $text-tertiary;
    margin-top: 0.125rem;
  }

  &__drop-divider {
    height: 1px;
    background: $border-subtle;
    margin: 0.125rem 0.375rem 0.5rem;
  }

  &__drop-item {
    display: flex;
    align-items: center;
    gap: 0.5625rem;
    width: 100%;
    padding: 0.5rem 0.625rem;
    border: none;
    border-radius: 0.5rem;
    background: transparent;
    color: $danger;
    font-size: 0.875rem;
    font-weight: 500;
    cursor: pointer;
    transition: background 0.14s ease;

    &:hover {
      background: $danger-subtle;
    }
  }

  @media (max-width: 620px) {
    &__brand-name {
      font-size: 1.0125rem;
    }
  }
}


















//Header.tsx
import { Link } from 'react-router-dom';
import { useEffect, useRef, useState, type FC } from 'react';
import { Gauge, LogOut, Sun, Moon } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import { toggleTheme } from '../../store/slices/uiSlice';
import './Header.scss';

function getInitials(name: string): string {
  return name
    .split(' ')
    .filter(Boolean)
    .slice(0, 2)
    .map((w) => w[0]?.toUpperCase())
    .join('');
}

const Header: FC = () => {
  const dispatch = useAppDispatch();
  const user = useAppSelector((s) => s.user.user);
  const theme = useAppSelector((s) => s.ui.theme);
  const [menuOpen, setMenuOpen] = useState(false);
  const menuRef = useRef<HTMLDivElement>(null);
  const initials = getInitials(user.name);
  const isDark = theme === 'dark';

  useEffect(() => {
    const handler = (e: MouseEvent) => {
      if (menuRef.current && !menuRef.current.contains(e.target as Node)) {
        setMenuOpen(false);
      }
    };
    document.addEventListener('mousedown', handler);
    return () => document.removeEventListener('mousedown', handler);
  }, []);

  return (
    <header className="app-header">
      <Link className="app-header__brand" to="/">
        <span className="app-header__brand-mark">
          <Gauge size={18} strokeWidth={2.25} />
        </span>
        <span className="app-header__brand-name">SemcoEval</span>
      </Link>

      <div className="app-header__right">
        <button
          type="button"
          className="app-header__theme-toggle"
          onClick={() => dispatch(toggleTheme())}
          aria-label={isDark ? 'Switch to light mode' : 'Switch to dark mode'}
          title={isDark ? 'Switch to light mode' : 'Switch to dark mode'}
        >
          <Sun size={12} strokeWidth={2.25} className="app-header__theme-static app-header__theme-static--sun" />
          <Moon size={12} strokeWidth={2.25} className="app-header__theme-static app-header__theme-static--moon" />

          <span className={`app-header__theme-knob${isDark ? ' app-header__theme-knob--dark' : ' app-header__theme-knob--light'}`}>
            {isDark ? <Moon size={13} strokeWidth={2.5} /> : <Sun size={13} strokeWidth={2.5} />}
          </span>
        </button>

        <div className="app-header__user" ref={menuRef}>
          <button
            type="button"
            className={`app-header__avatar${menuOpen ? ' app-header__avatar--open' : ''}`}
            onClick={() => setMenuOpen((v) => !v)}
            aria-label="User menu"
            aria-expanded={menuOpen}
            title={user.name}
          >
            {initials}
          </button>

          {menuOpen && (
            <div className="app-header__dropdown">
              <div className="app-header__drop-user">
                <div className="app-header__drop-avatar">{initials}</div>
                <div className="app-header__drop-info">
                  <div className="app-header__drop-name">{user.name}</div>
                  <div className="app-header__drop-role">{user.role}</div>
                </div>
              </div>
              <div className="app-header__drop-divider" />
              <button type="button" className="app-header__drop-item">
                <LogOut size={15} strokeWidth={2} />
                Log out
              </button>
            </div>
          )}
        </div>
      </div>
    </header>
  );
};

export default Header;
























//History.scss
@use '../../../styles/variables' as *;

.history {
  display: flex;
  flex-direction: column;
  gap: 18px;
  // Caps the page at a comfortable reading/working width and centers it, so
  // on very wide viewports (1800px+) the sidebar/detail columns don't stretch
  // into an unusably wide layout — instead the extra space becomes gutters.
  max-width: 1680px;
  margin-left: auto;
  margin-right: auto;
  // .main-layout__body / .workspace-layout / .workspace-layout__content form an
  // unbroken flex chain sized to the viewport (header/footer are fixed and offset
  // via margin), so height: 100% here resolves to the visible content area.
  // workspace-layout__content also has a 3rem bottom padding, which would
  // otherwise leave a large gap below .history — pull most of that back in,
  // keeping just a small 0.75rem breathing-room strip at the true bottom edge.
  height: calc(100% + 3rem - 0.75rem);
  margin-bottom: calc(-3rem + 0.75rem);
  min-height: 0;

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

  &__header-meta {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-secondary;
    background: $bg-subtle;
    border: 1px solid $border-subtle;
    border-radius: 999px;
    padding: 7px 13px;
    white-space: nowrap;
    margin-bottom: 3px;
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

  /* ---------- generic buttons ---------- */
  &__btn {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    font-family: $font-body;
    font-size: 0.8125rem;
    font-weight: 600;
    padding: 9px 14px;
    border-radius: 8px;
    border: 1px solid $border-default;
    background: $bg-main;
    color: $text-secondary;
    cursor: pointer;
    white-space: nowrap;
    transition: background 0.14s ease, border-color 0.14s ease, color 0.14s ease;

    &:hover {
      border-color: $text-primary;
      color: $text-primary;
    }

    &--primary {
      background: $primary;
      border-color: $primary;
      color: #fff;

      &:hover {
        background: $primary-hover;
        border-color: $primary-hover;
        color: #fff;
      }
    }

    &--danger:hover {
      border-color: $danger;
      color: $danger;
      background: $danger-subtle;
    }

    &--push {
      margin-left: auto;
    }
  }

  /* ---------- filters ---------- */
  &__filters {
    flex-shrink: 0;
    display: flex;
    flex-wrap: wrap;
    gap: 10px;
  }

  &__search {
    display: flex;
    align-items: center;
    gap: 9px;
    width: 280px;
    max-width: 100%;
    border: 1px solid $border-default;
    border-radius: 10px;
    padding: 9px 12px;
    background: $bg-main;
    color: $text-tertiary;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &:focus-within {
      border-color: $primary;
      box-shadow: 0 0 0 3px $primary-light;
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
    transition: background 0.14s ease, color 0.14s ease;

    &:hover {
      background: $border-default;
      color: $text-primary;
    }
  }

  /* ---------- custom dropdown (used by <Select />) ---------- */
  &-select {
    position: relative;
    flex-shrink: 0;

    &__trigger {
      width: 100%;
      display: flex;
      align-items: center;
      justify-content: space-between;
      gap: 8px;
      border: 1px solid $border-default;
      border-radius: 10px;
      padding: 9px 12px;
      background: $bg-main;
      font-size: 0.8125rem;
      font-weight: 500;
      font-family: $font-body;
      color: $text-primary;
      cursor: pointer;
      transition: border-color 0.14s ease, box-shadow 0.14s ease;

      &:hover {
        border-color: $border-strong;
      }

      &--open {
        border-color: $primary;
        box-shadow: 0 0 0 3px $primary-light;
      }
    }

    &__chevron {
      flex-shrink: 0;
      color: $text-tertiary;
      transition: transform 0.16s ease;
    }

    &__trigger--open &__chevron {
      transform: rotate(180deg);
    }

    &__menu {
      position: absolute;
      top: calc(100% + 6px);
      left: 0;
      right: 0;
      z-index: 20;
      background: $bg-main;
      border: 1px solid $border-subtle;
      border-radius: 10px;
      box-shadow: $shadow-lg;
      padding: 5px;
      display: flex;
      flex-direction: column;
      gap: 1px;
    }

    &__option {
      display: flex;
      align-items: center;
      justify-content: space-between;
      gap: 8px;
      width: 100%;
      text-align: left;
      padding: 8px 10px;
      border: none;
      border-radius: 7px;
      background: transparent;
      font-size: 0.8125rem;
      font-family: $font-body;
      color: $text-secondary;
      cursor: pointer;
      transition: background 0.12s ease, color 0.12s ease;

      &:hover {
        background: $bg-subtle;
        color: $text-primary;
      }

      &--active {
        color: $primary;
        font-weight: 600;

        svg {
          color: $primary;
        }
      }
    }
  }

  /* ---------- master-detail layout ---------- */
  &__layout {
    flex: 1;
    display: grid;
    grid-template-columns: 340px 1fr;
    gap: 16px;
    min-height: 0;
    overflow: hidden;
  }

  /* ---------- sidebar list ---------- */
  &__sidebar {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 16px;
    box-shadow: $shadow-xs;
    height: 100%;
    min-height: 0;
    display: flex;
    flex-direction: column;
    overflow: hidden;
  }

  &__sidebar-list {
    flex: 1;
    display: flex;
    flex-direction: column;
    min-height: 0;
    overflow-y: auto;
  }

  &__item {
    display: flex;
    flex-direction: column;
    gap: 6px;
    padding: 14px 16px;
    text-align: left;
    border: none;
    border-bottom: 1px solid $border-subtle;
    border-left: 3px solid transparent;
    background: $bg-main;
    width: 100%;
    cursor: pointer;
    transition: background 0.12s ease, border-color 0.12s ease;

    &:last-child {
      border-bottom: none;
    }

    &:hover {
      background: $bg-subtle;
    }

    &--active {
      background: $primary-light;
      border-left-color: $primary;

      .history__item-name {
        color: $primary;
      }
    }
  }

  &__item-top {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__item-name {
    font-size: 0.84375rem;
    font-weight: 600;
    color: $text-primary;
    line-height: 1.35;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__item-meta {
    display: flex;
    align-items: center;
    justify-content: space-between;
    font-size: 0.75rem;
    color: $text-tertiary;
  }

  &__item-score {
    font-weight: 700;
    color: $success;
  }

  /* ---------- type badge (shared by sidebar + detail) ---------- */
  &__type-badge {
    flex-shrink: 0;
    font-size: 0.625rem;
    font-weight: 700;
    border-radius: 999px;
    padding: 2px 8px;

    &--violet {
      color: $violet;
      background: $violet-light;
    }

    &--blue {
      color: $primary;
      background: $primary-light;
    }

    &--amber {
      color: $warning;
      background: $warning-subtle;
    }
  }

  /* ---------- detail panel ---------- */
  &__detail {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 16px;
    box-shadow: $shadow-xs;
    padding: 26px 28px;
    height: 100%;
    min-height: 0;
    overflow-y: auto;
  }

  &__detail-head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
    padding-bottom: 18px;
    margin-bottom: 20px;
    border-bottom: 1px solid $border-subtle;
    position: sticky;
    top: -26px;
    padding-top: 26px;
    margin-top: -26px;
    background: $bg-main;
    z-index: 1;
  }

  &__detail-head-left {
    display: flex;
    flex-direction: column;
    gap: 8px;

    .history__type-badge {
      width: fit-content;
    }
  }

  &__detail-name {
    font-size: 1.25rem;
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $text-primary;
  }

  &__detail-date {
    font-size: 0.8125rem;
    color: $text-tertiary;
  }

  &__detail-actions {
    display: flex;
    gap: 8px;
    flex-shrink: 0;
    flex-wrap: wrap;
  }

  /* ---------- stat cards ---------- */
  &__stat-row {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 12px;
    margin-bottom: 24px;
  }

  &__stat-card {
    background: $bg-subtle;
    border-radius: 12px;
    padding: 14px 16px;
    display: flex;
    flex-direction: column;
    gap: 4px;
    min-width: 0;
  }

  &__stat-card-label {
    font-size: 0.6875rem;
    font-weight: 600;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $text-tertiary;
  }

  &__stat-card-value {
    font-size: 1.25rem;
    font-weight: 800;
    color: $text-primary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;

    &--sm {
      font-size: 1rem;
      font-weight: 700;
    }

    &--accent {
      color: $success;
    }
  }

  /* ---------- results table ---------- */
  &__section-title {
    font-size: 0.8125rem;
    font-weight: 700;
    color: $text-primary;
    margin-bottom: 12px;
  }

  &__table-wrap {
    overflow-x: auto;
  }

  &__table {
    width: 100%;
    border-collapse: collapse;

    thead th {
      text-align: left;
      font-size: 0.6875rem;
      font-weight: 700;
      letter-spacing: 0.05em;
      text-transform: uppercase;
      color: $text-tertiary;
      padding: 10px 14px;
      background: $bg-subtle;
      white-space: nowrap;

      &:first-child {
        border-radius: 8px 0 0 8px;
      }

      &:last-child {
        border-radius: 0 8px 8px 0;
      }
    }

    tbody td {
      padding: 12px 14px;
      font-size: 0.84375rem;
      color: $text-secondary;
      border-bottom: 1px solid $border-subtle;
      white-space: nowrap;
    }

    tbody tr:last-child td {
      border-bottom: none;
    }
  }

  &__cell-strong {
    font-weight: 600;
    color: $text-primary;
  }

  &__rank-pill {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 22px;
    height: 22px;
    border-radius: 6px;
    background: $bg-inset;
    color: $text-tertiary;
    font-size: 0.6875rem;
    font-weight: 700;

    &--1 {
      background: $primary-light;
      color: $primary;
    }
  }

  &__score-cell {
    color: $success;
    font-weight: 700;
  }

  /* ---------- empty state ---------- */
  &__empty {
    flex-shrink: 0;
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

  /* ---------- responsive ---------- */
  @media (max-width: 900px) {
    &__layout {
      grid-template-columns: 1fr;
      grid-template-rows: 16rem 1fr;
      overflow: visible;
    }

    &__sidebar,
    &__detail {
      height: auto;
      max-height: 22rem;
    }

    &__stat-row {
      grid-template-columns: repeat(2, 1fr);
    }
  }

  @media (max-width: 520px) {
    &__detail {
      padding: 18px 16px;
    }

    &__detail-head {
      flex-direction: column;
    }

    &__stat-row {
      grid-template-columns: 1fr;
    }
  }
}

















//History.tsx
import { useMemo, useState, type FC, type MouseEvent } from 'react';
import { useNavigate } from 'react-router-dom';
import { Play, Search, Copy, Trash2, X, Database, FileBarChart } from 'lucide-react';
import { RECENT_EVALUATIONS, type RecentEvaluation } from '../shared/evaluations';
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

const History: FC = () => {
  const navigate = useNavigate();
  const [items, setItems] = useState<RecentEvaluation[]>(RECENT_EVALUATIONS);
  const [query, setQuery] = useState('');
  const [typeFilter, setTypeFilter] = useState('all');
  const [dateFilter, setDateFilter] = useState(30);
  const [selectedId, setSelectedId] = useState<string | null>(RECENT_EVALUATIONS[0]?.id ?? null);

  const filtered = useMemo(() => {
    return items.filter((ev) => {
      if (query && !ev.name.toLowerCase().includes(query.toLowerCase())) return false;
      if (!matchesType(ev.type, typeFilter)) return false;
      if (ev.daysAgo > dateFilter) return false;
      return true;
    });
  }, [items, query, typeFilter, dateFilter]);

  const selected = useMemo(
    () => filtered.find((ev) => ev.id === selectedId) ?? filtered[0] ?? null,
    [filtered, selectedId]
  );

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
                const tint = typeTint(ev.type);
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
                        {ev.type.split('(')[0].trim()}
                      </span>
                    </div>
                    <div className="history__item-meta">
                      <span>{ev.date}</span>
                      <span className="history__item-score n">{ev.topScore}</span>
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
                  <span className={`history__type-badge history__type-badge--${typeTint(selected.type)}`}>
                    {selected.type.split('(')[0].trim()}
                  </span>
                  <h2 className="history__detail-name">{selected.name}</h2>
                  <span className="history__detail-date">
                    {selected.status === 'Running' ? 'Running' : 'Completed'} &middot; {selected.date}
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
                  <span className="history__stat-card-value n">{selected.modelsTested}</span>
                </div>
                <div className="history__stat-card">
                  <span className="history__stat-card-label">Top Model</span>
                  <span className="history__stat-card-value history__stat-card-value--sm">{selected.topModel}</span>
                </div>
                <div className="history__stat-card">
                  <span className="history__stat-card-label">Top Score</span>
                  <span className="history__stat-card-value history__stat-card-value--accent n">{selected.topScore}</span>
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
                    {selected.results.map((r) => (
                      <tr key={r.rank}>
                        <td>
                          <span className={`history__rank-pill${r.rank === 1 ? ' history__rank-pill--1' : ''}`}>{r.rank}</span>
                        </td>
                        <td className="history__cell-strong">{r.model}</td>
                        <td>{r.provider}</td>
                        <td className={`n${r.rank === 1 ? ' history__score-cell' : ''}`}>{r.score}</td>
                        <td className="n">{r.accuracy}</td>
                        <td className="n">{r.time}</td>
                        <td className="n">{r.cost}</td>
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





















//Landing.scss
@use '../../styles/variables' as *;

.landing {
  --blue: #{$primary};
  --blue-2: #{$primary-hover};
  --blue-wash: #{$primary-light};
  --jade: #{$success};
  --jade-w: #{$success-subtle};
  --red: #{$danger};
  --red-w: #{$danger-subtle};
  --amber-w: #{$warning-subtle};
  --ink: #{$text-primary};
  --ink-2: #{$text-secondary};
  --ink-3: #{$text-tertiary};
  --white: #{$bg-main};
  --on-brand: #fff;
  --paper: #{$bg-subtle};
  --rule: #{$border-default};
  --rule-2: #{$border-subtle};
  --gut: clamp(1.25rem, 5vw, 3.75rem);
  --band: clamp(4.5rem, 8.5vw, 7.75rem);

  background: var(--white);
  color: var(--ink);

  &__shell {
    max-width: 1200px;
    margin: 0 auto;
    padding: 0 var(--gut);
  }

  &__hero-badge {
    display: inline-flex;
    align-items: center;
    gap: 0.5rem;
    font-size: 0.8125rem;
    font-weight: 600;
    color: var(--blue);
    background: var(--blue-wash);
    border: 1px solid $primary-subtle;
    border-radius: 999px;
    padding: 0.375rem 0.75rem 0.375rem 0.625rem;
    margin-bottom: 1.125rem;
  }

  &__hero-badge-dot {
    width: 6px;
    height: 6px;
    border-radius: 50%;
    background: var(--jade);
    box-shadow: 0 0 0 0.1875rem var(--jade-w);
  }

  &__eyebrow {
    font-size: 0.7185rem;
    font-weight: 600;
    letter-spacing: 0.15em;
    text-transform: uppercase;
    color: var(--ink-3);
    display: flex;
    align-items: center;
    gap: 0.625rem;

    &::before {
      content: '';
      width: 20px;
      height: 1px;
      background: var(--blue);
    }
  }

  h1,
  h2,
  h3 {
    letter-spacing: -0.032em;
    line-height: 1.06;
    font-weight: 700;
  }

  /* ================= BUTTONS ================= */
  &__btn {
    font-family: $font-body;
    font-size: 0.9375rem;
    font-weight: 600;
    padding: 0.625rem 1.125rem;
    border-radius: 0.5rem;
    border: 1px solid transparent;
    cursor: pointer;
    text-decoration: none;
    display: inline-flex;
    align-items: center;
    gap: 0.5rem;
    transition: background 0.15s ease, border-color 0.15s ease, color 0.15s ease;

    &--lg {
      padding: 0.875rem 1.5rem;
      font-size: 1rem;
    }

    &--primary {
      background: var(--blue);
      color: var(--on-brand);

      &:hover {
        background: var(--blue-2);
      }
    }

    &--ghost {
      background: var(--white);
      color: var(--ink);
      border-color: var(--rule);

      &:hover {
        border-color: var(--ink);
      }
    }

    &:focus-visible {
      outline: 0.125rem solid var(--blue);
      outline-offset: 0.1875rem;
    }
  }

  /* ================= HERO ================= */
  &__hero {
    position: relative;
    overflow: hidden;
    padding: clamp(3.125rem, 6.5vw, 5.5rem) 0 var(--band);

    &::before {
      content: '';
      position: absolute;
      inset: 0;
      background-image: linear-gradient(to right, var(--rule-2) 0.0625rem, transparent 0.0625rem),
        linear-gradient(to bottom, var(--rule-2) 0.0625rem, transparent 0.0625rem);
      background-size: 3.375rem 3.375rem;
      mask-image: radial-gradient(115% 80% at 45% 0%, #000 22%, transparent 72%);
      -webkit-mask-image: radial-gradient(115% 80% at 45% 0%, #000 22%, transparent 72%);
      pointer-events: none;
    }
  }

  &__hero-in {
    position: relative;
  }

  &__hero-copy {
    max-width: 720px;
  }

  &__hero h1 {
    font-size: clamp(2.5rem, 6.2vw, 4.625rem);
    margin: 1.25rem 0 0;

    em {
      font-style: normal;
      color: var(--blue);
      position: relative;
      white-space: nowrap;

      &::after {
        content: '';
        position: absolute;
        left: 0;
        right: 0;
        bottom: -0.13em;
        height: 6px;
        background: repeating-linear-gradient(to right, var(--blue) 0 0.09375rem, transparent 0.09375rem 0.4375rem);
        opacity: 0.45;
      }
    }
  }

  &__hero-sub {
    margin-top: 1.375rem;
    max-width: 560px;
    font-size: clamp(1rem, 1.5vw, 1.156rem);
    color: var(--ink-2);
  }

  &__hero-cta {
    margin-top: 2rem;
    display: flex;
    flex-wrap: wrap;
    gap: 0.75rem;
    align-items: center;
  }

  /* ================= SCORE PANEL ================= */
  &__panel {
    margin-top: clamp(2.625rem, 5.5vw, 4.25rem);
    background: var(--white);
    border: 1px solid var(--rule);
    border-radius: 0.875rem;
    box-shadow: $shadow-lg;
    overflow: hidden;
  }

  &__panel-head {
    display: flex;
    align-items: center;
    flex-wrap: wrap;
    gap: 0.5rem 0.875rem;
    padding: 0.875rem 1.25rem;
    border-bottom: 1px solid var(--rule-2);
    background: var(--paper);
  }

  &__panel-title {
    font-size: 0.9375rem;
    font-weight: 600;
    letter-spacing: -0.01em;
  }

  &__panel-meta {
    margin-left: auto;
    font-size: 0.84375rem;
    color: var(--ink-3);
    display: inline-flex;
    align-items: center;
    gap: 0.4375rem;

    &::before {
      content: '';
      width: 6px;
      height: 6px;
      border-radius: 50%;
      background: var(--jade);
      box-shadow: 0 0 0 0.1875rem var(--jade-w);
    }
  }

  &__rails {
    padding: 0.375rem 2.5rem 1.25rem;
  }

  &__rail {
    padding: 1.25rem 0;
    border-bottom: 1px dashed var(--rule);

    &:last-child {
      border-bottom: 0;
    }
  }

  &__rail-top {
    display: flex;
    align-items: baseline;
    flex-wrap: wrap;
    gap: 0.25rem 0.75rem;
    margin-bottom: 1.75rem;
  }

  &__rail-label {
    font-size: 0.90625rem;
    font-weight: 600;
  }

  &__rail-unit {
    font-size: 0.8125rem;
    color: var(--ink-3);
  }

  &__rail-dir {
    margin-left: auto;
    font-size: 0.8125rem;
    color: var(--ink-3);
  }

  &__axis {
    position: relative;
    height: 2px;
    background: var(--rule);
    border-radius: 0.125rem;

    &::after {
      content: '';
      position: absolute;
      left: 0;
      right: 0;
      top: 0;
      height: 7px;
      background: repeating-linear-gradient(to right, var(--rule) 0 0.0625rem, transparent 0.0625rem 10%);
    }
  }

  &__axis-scale {
    display: flex;
    justify-content: space-between;
    font-size: 0.71875rem;
    color: var(--ink-3);
    margin-top: 0.75rem;
  }

  &__pip {
    position: absolute;
    top: 50%;
    left: var(--x);
    transform: translate(-50%, -50%);
    animation: landing-settle 0.85s cubic-bezier(0.2, 0.8, 0.2, 1) backwards;
    animation-delay: var(--d, 0s);
  }

  &__pip-dot {
    display: block;
    width: 13px;
    height: 13px;
    border-radius: 50%;
    background: var(--white);
    border: 3px solid var(--ink-3);
    transition: transform 0.15s ease;
  }

  &__pip:hover &__pip-dot {
    transform: scale(1.22);
  }

  &__pip-flag {
    position: absolute;
    bottom: 1.125rem;
    left: 50%;
    transform: translateX(-50%);
    white-space: nowrap;
    font-size: 0.78125rem;
    color: var(--ink-2);
    background: var(--white);
    padding: 0.0625rem 0.3125rem;
    border-radius: 0.25rem;

    b {
      color: var(--ink);
      font-weight: 700;
    }
  }

  &__pip--low &__pip-flag {
    bottom: auto;
    top: 1.125rem;
  }

  &__pip--best &__pip-dot {
    border-color: var(--jade);
  }

  &__pip--best &__pip-flag,
  &__pip--best &__pip-flag b {
    color: var(--jade);
  }

  &__pip--brand &__pip-dot {
    border-color: var(--blue);
  }

  &__panel-foot {
    display: flex;
    flex-wrap: wrap;
    gap: 0.5rem 1.625rem;
    padding: 0.875rem 1.25rem;
    border-top: 1px solid var(--rule-2);
    background: var(--paper);
    font-size: 0.875rem;
    color: var(--ink-2);

    b {
      color: var(--ink);
      font-weight: 600;
    }
  }

  &__k {
    font-size: 0.6875rem;
    font-weight: 600;
    letter-spacing: 0.13em;
    text-transform: uppercase;
    color: var(--ink-3);
    margin-right: 0.4375rem;
  }

  /* ================= BANDS ================= */
  &__band {
    padding: var(--band) 0;

    &--paper {
      background: var(--paper);
      border-block: 0.0625rem solid var(--rule-2);
    }
  }

  &__band-head {
    max-width: 660px;

    .landing__eyebrow {
      color: var(--blue);
    }

    h2 {
      font-size: clamp(1.75rem, 3.7vw, 2.75rem);
      margin: 1rem 0 0;
    }

    p {
      margin-top: 1rem;
      color: var(--ink-2);
      font-size: 1.125rem;
    }
  }

  &__band-body {
    margin-top: clamp(2.125rem, 4.5vw, 3.375rem);
  }

  /* ================= PROOF STRIP ================= */
  &__strip {
    background: var(--paper);
    border-block: 0.0625rem solid var(--rule-2);
  }

  &__strip-in {
    display: flex;
    flex-wrap: wrap;
    gap: 0.625rem 2.875rem;
    padding: 1.375rem 0;
    align-items: baseline;
  }

  &__stat {
    display: flex;
    align-items: baseline;
    gap: 0.5625rem;
  }

  &__stat-n {
    font-size: 1.375rem;
    font-weight: 700;
    letter-spacing: -0.03em;
  }

  &__stat-l {
    font-size: 0.875rem;
    color: var(--ink-2);
  }

  &__strip-note {
    margin-left: auto;
    font-size: 0.84375rem;
    color: var(--ink-3);
  }

  /* ================= GRADED TRANSCRIPT ================= */
  &__ask {
    background: var(--white);
    border: 1px solid var(--rule);
    border-left: 3px solid var(--blue);
    border-radius: 0.625rem;
    padding: 1.25rem 1.375rem;

    .landing__eyebrow {
      color: var(--ink-3);

      &::before {
        background: var(--ink-3);
      }
    }
  }

  &__ask-q {
    margin-top: 0.75rem;
    font-size: 1.15625rem;
    line-height: 1.45;
  }

  &__ask-src {
    margin-top: 1rem;
    padding: 0.6875rem 0.8125rem;
    background: var(--paper);
    border-radius: 0.4375rem;
    font-size: 0.875rem;
    color: var(--ink-2);

    b {
      color: var(--ink);
      font-weight: 600;
    }
  }

  &__answers {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 0.875rem;
    margin-top: 0.875rem;
  }

  &__ans {
    background: var(--white);
    border: 1px solid var(--rule);
    border-left: 3px solid var(--rule);
    border-radius: 0.625rem;
    padding: 1rem 1.125rem;
    display: flex;
    flex-direction: column;

    &--pass {
      border-left-color: var(--jade);

      .landing__ans-mark {
        background: var(--jade-w);
        color: var(--jade);
      }
    }

    &--fail {
      border-left-color: var(--red);

      .landing__ans-mark {
        background: var(--red-w);
        color: var(--red);
      }
    }

    p {
      font-size: 0.96875rem;
      color: var(--ink-2);
    }
  }

  &__ans-top {
    display: flex;
    align-items: center;
    gap: 0.625rem;
    margin-bottom: 0.6875rem;
  }

  &__ans-name {
    font-size: 0.90625rem;
    font-weight: 600;
  }

  &__ans-mark {
    margin-left: auto;
    font-size: 0.6875rem;
    font-weight: 700;
    letter-spacing: 0.11em;
    text-transform: uppercase;
    padding: 0.1875rem 0.5rem;
    border-radius: 0.3125rem;
  }

  &__ans-foot {
    margin-top: auto;
    padding-top: 0.8125rem;
    font-size: 0.84375rem;
    display: flex;
    gap: 1rem;
    color: var(--ink-3);
  }

  &__ans-note {
    margin-top: 0.75rem;
    background: var(--red-w);
    color: var(--red);
    border-radius: 0.4375rem;
    padding: 0.5625rem 0.6875rem;
    font-size: 0.84375rem;
    line-height: 1.45;
  }

  &__hl {
    background: var(--amber-w);
    padding: 0 0.1875rem;
    border-radius: 0.1875rem;
    color: var(--ink);
  }

  /* ================= PIPELINE ================= */
  &__pipeline {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 1rem;
  }

  &__step {
    padding: 1.5rem 1.375rem 1.625rem;
    border: 1px solid var(--rule);
    border-radius: 0.75rem;
    background: var(--white);
    position: relative;

    &::before {
      content: '';
      position: absolute;
      top: -0.0625rem;
      left: -0.0625rem;
      width: 40px;
      height: 3px;
      border-radius: 0.75rem 0 0 0;
      background: var(--blue);
    }

    h3 {
      font-size: 1.21875rem;
      margin: 0.75rem 0 0.5625rem;
    }

    p {
      font-size: 0.96875rem;
      color: var(--ink-2);
    }
  }

  &__step-k {
    font-size: 0.75rem;
    font-weight: 700;
    letter-spacing: 0.12em;
    color: var(--blue);
  }

  &__step-hint {
    margin-top: 0.875rem;
    padding-top: 0.75rem;
    border-top: 1px dashed var(--rule);
    font-size: 0.8125rem;
    color: var(--ink-3);
  }

  /* ================= MODES ================= */
  &__modes {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 1rem;
  }

  &__mode {
    background: var(--white);
    border: 1px solid var(--rule);
    border-radius: 0.75rem;
    padding: 1.5rem 1.375rem;
    display: flex;
    flex-direction: column;
    transition: border-color 0.15s ease, transform 0.15s ease;

    &:hover {
      border-color: var(--ink);
      transform: translateY(-0.125rem);
    }

    h3 {
      font-size: 1.25rem;
      margin: 1rem 0 0.625rem;
    }

    p {
      font-size: 0.96875rem;
      color: var(--ink-2);
    }
  }

  &__mode-tag {
    align-self: flex-start;
    font-size: 0.6875rem;
    font-weight: 600;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    color: var(--blue);
    background: var(--blue-wash);
    padding: 0.3125rem 0.5625rem;
    border-radius: 0.3125rem;
  }

  &__chips {
    margin-top: 1.25rem;
    padding-top: 0.875rem;
    border-top: 1px solid var(--rule-2);
    display: flex;
    flex-wrap: wrap;
    gap: 0.375rem;
  }

  &__chip {
    font-size: 0.78125rem;
    color: var(--ink-2);
    border: 1px solid var(--rule);
    border-radius: 0.3125rem;
    padding: 0.1875rem 0.5rem;
  }

  /* ================= LEDGER ================= */
  &__ledger {
    border-top: 1px solid var(--rule);
  }

  &__row {
    display: grid;
    grid-template-columns: 2.5rem 1.05fr 1.45fr;
    gap: 1.5rem;
    padding: 1.625rem 0;
    border-bottom: 1px solid var(--rule);
    align-items: start;

    h3 {
      font-size: 1.28125rem;
    }

    p {
      font-size: 1rem;
      color: var(--ink-2);
    }
  }

  &__row-k {
    font-size: 0.8125rem;
    color: var(--ink-3);
    padding-top: 0.3125rem;
  }

  /* ================= CLOSE ================= */
  &__close {
    padding: var(--band) 0;
  }

  &__close-in {
    max-width: 700px;

    .landing__eyebrow {
      color: var(--blue);
    }

    h2 {
      font-size: clamp(1.875rem, 4.4vw, 3.125rem);
      margin: 1rem 0 0;
    }

    p {
      margin-top: 1.125rem;
      color: var(--ink-2);
      font-size: 1.125rem;
    }
  }

  &__close-cta {
    margin-top: 1.875rem;
    display: flex;
    flex-wrap: wrap;
    gap: 0.75rem;
  }

  /* ================= RESPONSIVE ================= */
  @media (max-width: 1000px) {
    .landing__pipeline {
      grid-template-columns: 1fr 1fr;
    }
    .landing__modes {
      grid-template-columns: 1fr;
    }
  }

  @media (max-width: 860px) {
    .landing__answers {
      grid-template-columns: 1fr;
    }
    .landing__row {
      grid-template-columns: 1.875rem 1fr;
    }
    .landing__row p {
      grid-column: 2;
    }
  }

  @media (max-width: 620px) {
    .landing__pipeline {
      grid-template-columns: 1fr;
    }
    .landing__rails {
      padding-inline: 1.75rem;
    }
    .landing__rail-dir {
      display: none;
    }
    .landing__pip-flag {
      font-size: 0.71875rem;
    }
  }

  @media (prefers-reduced-motion: reduce) {
    .landing__pip {
      animation: none;
    }
    .landing__mode {
      transition: none;
    }
  }
}

@keyframes landing-settle {
  from {
    left: 0%;
    opacity: 0;
  }
  to {
    left: var(--x);
    opacity: 1;
  }
}





















//Models.scss
@use '../../../styles/variables' as *;

.models-page {
  display: flex;
  flex-direction: column;
  gap: 18px;
  // Caps the page at a comfortable working width and centers it, so on very
  // wide viewports (1800px+) the sidebar/detail columns don't stretch into an
  // unusably wide layout — the extra space becomes gutters instead.
  max-width: 1680px;
  margin-left: auto;
  margin-right: auto;
  // Same flex chain as History: main-layout__body / workspace-layout /
  // workspace-layout__content already resolve to the exact viewport height,
  // so height: 100% fills it precisely. workspace-layout__content also carries
  // a 3rem bottom padding, which would leave a gap below this page — pull most
  // of that back in, keeping a small 0.75rem breathing-room strip at the bottom.
  height: calc(100% + 3rem - 0.75rem);
  margin-bottom: calc(-3rem + 0.75rem);
  min-height: 0;

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

  &__header-meta {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-secondary;
    background: $bg-subtle;
    border: 1px solid $border-subtle;
    border-radius: 999px;
    padding: 7px 13px;
    white-space: nowrap;
    margin-bottom: 3px;
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

  /* ---------- filters ---------- */
  &__filters {
    flex-shrink: 0;
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 12px;
  }

  &__search {
    display: flex;
    align-items: center;
    gap: 9px;
    width: 280px;
    max-width: 100%;
    border: 1px solid $border-default;
    border-radius: 10px;
    padding: 9px 12px;
    background: $bg-main;
    color: $text-tertiary;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &:focus-within {
      border-color: $primary;
      box-shadow: 0 0 0 3px $primary-light;
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
    transition: background 0.14s ease, color 0.14s ease;

    &:hover {
      background: $border-default;
      color: $text-primary;
    }
  }

  &__seg {
    display: inline-flex;
    align-items: center;
    gap: 2px;
    padding: 3px;
    border: 1px solid $border-subtle;
    border-radius: 11px;
    background: $bg-subtle;
  }

  &__seg-item {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-family: $font-body;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-tertiary;
    background: transparent;
    border: none;
    border-radius: 8px;
    padding: 7px 12px;
    cursor: pointer;
    transition: background 0.14s ease, color 0.14s ease, box-shadow 0.14s ease;

    svg {
      opacity: 0.8;
    }

    &:hover {
      color: $text-primary;
    }

    &--active {
      background: $bg-main;
      color: $primary;
      box-shadow: $shadow-xs;

      svg {
        opacity: 1;
      }
    }
  }

  /* ---------- master-detail layout ---------- */
  &__layout {
    flex: 1;
    display: grid;
    grid-template-columns: 340px 1fr;
    gap: 16px;
    min-height: 0;
    overflow: hidden;
  }

  /* ---------- sidebar list ---------- */
  &__sidebar {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 16px;
    box-shadow: $shadow-xs;
    height: 100%;
    min-height: 0;
    display: flex;
    flex-direction: column;
    overflow: hidden;
  }

  &__sidebar-list {
    flex: 1;
    display: flex;
    flex-direction: column;
    min-height: 0;
    overflow-y: auto;
  }

  &__item {
    display: flex;
    flex-direction: column;
    gap: 6px;
    padding: 14px 16px;
    text-align: left;
    border: none;
    border-bottom: 1px solid $border-subtle;
    border-left: 3px solid transparent;
    background: $bg-main;
    width: 100%;
    cursor: pointer;
    transition: background 0.12s ease, border-color 0.12s ease;

    &:last-child {
      border-bottom: none;
    }

    &:hover {
      background: $bg-subtle;
    }

    &--active {
      background: $primary-light;
      border-left-color: $primary;

      .models-page__item-name {
        color: $primary;
      }
    }
  }

  &__item-top {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__item-name {
    font-size: 0.84375rem;
    font-weight: 600;
    color: $text-primary;
    line-height: 1.35;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__item-provider {
    flex-shrink: 0;
    font-size: 0.65625rem;
    font-weight: 600;
    color: $text-tertiary;
  }

  &__item-meta {
    display: flex;
    align-items: center;
    justify-content: space-between;
    font-size: 0.75rem;
    color: $text-tertiary;
  }

  &__item-score {
    font-weight: 700;
    color: $success;
  }

  /* ---------- detail panel ---------- */
  &__detail {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 16px;
    box-shadow: $shadow-xs;
    padding: 26px 28px;
    height: 100%;
    min-height: 0;
    overflow-y: auto;
  }

  &__detail-head {
    padding-bottom: 18px;
    margin-bottom: 16px;
    border-bottom: 1px solid $border-subtle;
    position: sticky;
    top: -26px;
    padding-top: 26px;
    margin-top: -26px;
    background: $bg-main;
    z-index: 1;
  }

  &__detail-head-left {
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  &__detail-name {
    font-size: 1.25rem;
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $text-primary;
  }

  &__detail-version {
    font-family: $font-mono;
    font-size: 0.75rem;
    color: $text-tertiary;
  }

  &__detail-desc {
    font-size: 0.84375rem;
    line-height: 1.6;
    color: $text-secondary;
    margin-bottom: 18px;
  }

  &__provider-badge {
    width: fit-content;
    font-size: 0.6875rem;
    font-weight: 600;
    color: $primary;
    background: $primary-light;
    border-radius: 999px;
    padding: 3px 10px;
  }

  /* ---------- capability pills ---------- */
  &__caps {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    margin-bottom: 24px;
  }

  &__cap-pill {
    font-size: 0.71875rem;
    font-weight: 600;
    border-radius: 999px;
    padding: 3px 10px;

    &--blue {
      color: $primary;
      background: $primary-light;
    }

    &--violet {
      color: $violet;
      background: $violet-light;
    }

    &--amber {
      color: $warning;
      background: $warning-subtle;
    }

    &--jade {
      color: $success;
      background: $success-subtle;
    }

    &--rose {
      color: $danger;
      background: $danger-subtle;
    }
  }

  /* ---------- stat cards ---------- */
  &__section-title {
    font-size: 0.8125rem;
    font-weight: 700;
    color: $text-primary;
    margin-bottom: 12px;
  }

  &__stat-row {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 12px;
    margin-bottom: 24px;
  }

  &__stat-card {
    background: $bg-subtle;
    border-radius: 12px;
    padding: 14px 16px;
    display: flex;
    flex-direction: column;
    gap: 4px;
    min-width: 0;
  }

  &__stat-card-label {
    font-size: 0.6875rem;
    font-weight: 600;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $text-tertiary;
  }

  &__stat-card-value {
    font-size: 1.25rem;
    font-weight: 800;
    color: $text-primary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;

    &--sm {
      font-size: 1rem;
      font-weight: 700;
    }

    &--accent {
      color: $success;
    }
  }

  /* ---------- agent score bar ---------- */
  &__agent-score {
    display: flex;
    align-items: center;
    gap: 12px;
  }

  &__agent-score-bar-track {
    flex: 1;
    height: 8px;
    border-radius: 999px;
    background: $bg-inset;
    overflow: hidden;
  }

  &__agent-score-bar-fill {
    height: 100%;
    border-radius: 999px;
    background: $primary;
  }

  &__agent-score-value {
    font-size: 0.9375rem;
    font-weight: 700;
    color: $success;
    flex-shrink: 0;
    width: 48px;
    text-align: right;
  }

  /* ---------- empty state ---------- */
  &__empty {
    flex-shrink: 0;
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

  /* ---------- responsive ---------- */
  @media (max-width: 900px) {
    &__layout {
      grid-template-columns: 1fr;
      grid-template-rows: 16rem 1fr;
      overflow: visible;
    }

    &__sidebar,
    &__detail {
      height: auto;
      max-height: 22rem;
    }

    &__stat-row {
      grid-template-columns: repeat(2, 1fr);
    }
  }

  @media (max-width: 520px) {
    &__detail {
      padding: 18px 16px;
    }

    &__stat-row {
      grid-template-columns: 1fr;
    }
  }
}

















//Models.tsx
import { useMemo, useState, type FC } from 'react';
import { Search, X, Boxes, LayoutGrid, Wrench, Eye, BrainCircuit, Code2 } from 'lucide-react';
import { MODELS } from '../RunEvaluation/data';
import './Models.scss';

const CAPABILITY_FILTERS = [
  { value: 'All', icon: LayoutGrid },
  { value: 'Tool Calling', icon: Wrench },
  { value: 'Vision', icon: Eye },
  { value: 'Reasoning', icon: BrainCircuit },
  { value: 'Coding', icon: Code2 },
];

const PILL_TINTS = ['blue', 'violet', 'amber', 'jade', 'rose'] as const;

function pillTint(capability: string) {
  let hash = 0;
  for (let i = 0; i < capability.length; i += 1) hash = (hash * 31 + capability.charCodeAt(i)) >>> 0;
  return PILL_TINTS[hash % PILL_TINTS.length];
}

const Models: FC = () => {
  const [query, setQuery] = useState('');
  const [capability, setCapability] = useState('All');
  const [selectedId, setSelectedId] = useState<string | null>(MODELS[0]?.id ?? null);

  const filtered = useMemo(() => {
    return MODELS.filter((m) => {
      if (query && !m.name.toLowerCase().includes(query.toLowerCase()) && !m.provider.toLowerCase().includes(query.toLowerCase())) {
        return false;
      }
      if (capability !== 'All') {
        const matches = m.capabilities.some((c) => c.toLowerCase().includes(capability.toLowerCase()));
        if (!matches) return false;
      }
      return true;
    });
  }, [query, capability]);

  const selected = useMemo(
    () => filtered.find((m) => m.id === selectedId) ?? filtered[0] ?? null,
    [filtered, selectedId]
  );

  return (
    <div className="models-page">
      <div className="models-page__header">
        <div className="models-page__header-left">
          <p className="models-page__header-eyebrow">Model catalog</p>
          <h1 className="models-page__title">Models</h1>
          <p className="models-page__subtitle">Browse available AI models across every connected provider</p>
        </div>

        <div className="models-page__header-meta">
          <Boxes size={13} />
          {MODELS.length} models available
        </div>
      </div>

      <div className="models-page__filters">
        <div className="models-page__search">
          <Search size={15} />
          <input type="text" placeholder="Search models..." value={query} onChange={(e) => setQuery(e.target.value)} />
          {query && (
            <button type="button" className="models-page__search-clear" onClick={() => setQuery('')} aria-label="Clear search">
              <X size={13} />
            </button>
          )}
        </div>

        <div className="models-page__seg">
          {CAPABILITY_FILTERS.map((c) => (
            <button
              key={c.value}
              type="button"
              className={`models-page__seg-item${capability === c.value ? ' models-page__seg-item--active' : ''}`}
              onClick={() => setCapability(c.value)}
            >
              <c.icon size={13} strokeWidth={2.25} />
              {c.value}
            </button>
          ))}
        </div>
      </div>

      {filtered.length === 0 ? (
        <div className="models-page__empty">
          <Search size={22} />
          <p>No models match your filters.</p>
        </div>
      ) : (
        <div className="models-page__layout">
          <div className="models-page__sidebar">
            <div className="models-page__sidebar-list">
              {filtered.map((m) => {
                const isActive = selected?.id === m.id;
                return (
                  <button
                    type="button"
                    key={m.id}
                    className={`models-page__item${isActive ? ' models-page__item--active' : ''}`}
                    onClick={() => setSelectedId(m.id)}
                  >
                    <div className="models-page__item-top">
                      <span className="models-page__item-name">{m.name}</span>
                      <span className="models-page__item-provider">{m.provider}</span>
                    </div>
                    <div className="models-page__item-meta">
                      <span>{m.pricing}</span>
                      <span className="models-page__item-score n">{m.accuracyScore}%</span>
                    </div>
                  </button>
                );
              })}
            </div>
          </div>

          {selected && (
            <div className="models-page__detail">
              <div className="models-page__detail-head">
                <div className="models-page__detail-head-left">
                  <span className="models-page__provider-badge">{selected.provider}</span>
                  <h2 className="models-page__detail-name">{selected.name}</h2>
                  <span className="models-page__detail-version">{selected.version}</span>
                </div>
              </div>

              <p className="models-page__detail-desc">{selected.description}</p>

              <div className="models-page__caps">
                {selected.capabilities.map((c) => (
                  <span key={c} className={`models-page__cap-pill models-page__cap-pill--${pillTint(c)}`}>
                    {c}
                  </span>
                ))}
              </div>

              <p className="models-page__section-title">Specifications</p>
              <div className="models-page__stat-row">
                <div className="models-page__stat-card">
                  <span className="models-page__stat-card-label">Context Window</span>
                  <span className="models-page__stat-card-value models-page__stat-card-value--sm">{selected.contextWindow}</span>
                </div>
                <div className="models-page__stat-card">
                  <span className="models-page__stat-card-label">Pricing</span>
                  <span className="models-page__stat-card-value models-page__stat-card-value--sm">{selected.pricing}</span>
                </div>
                <div className="models-page__stat-card">
                  <span className="models-page__stat-card-label">Speed</span>
                  <span className="models-page__stat-card-value models-page__stat-card-value--sm">{selected.speedRating}</span>
                </div>
                <div className="models-page__stat-card">
                  <span className="models-page__stat-card-label">Accuracy</span>
                  <span className="models-page__stat-card-value models-page__stat-card-value--accent n">{selected.accuracyScore}%</span>
                </div>
              </div>

              <p className="models-page__section-title">Agent performance</p>
              <div className="models-page__agent-score">
                <div className="models-page__agent-score-bar-track">
                  <div className="models-page__agent-score-bar-fill" style={{ width: `${selected.agentScore}%` }} />
                </div>
                <span className="models-page__agent-score-value n">{selected.agentScore}%</span>
              </div>
            </div>
          )}
        </div>
      )}
    </div>
  );
};

export default Models;


























//Providers.scss
@use '../../../styles/variables' as *;

.providers-page {
  display: flex;
  flex-direction: column;
  gap: 16px;
  // Caps the page at a comfortable working width and centers it, so on very
  // wide viewports (1800px+) the sidebar/detail columns don't stretch into an
  // unusably wide layout — the extra space becomes gutters instead.
  max-width: 1680px;
  margin-left: auto;
  margin-right: auto;
  // Same flex chain as History/Models: main-layout__body / workspace-layout /
  // workspace-layout__content already resolve to the exact viewport height,
  // so height: 100% fills it precisely. workspace-layout__content also carries
  // a 3rem bottom padding, which would leave a gap below this page — pull most
  // of that back in, keeping a small 0.75rem breathing-room strip at the bottom.
  height: calc(100% + 3rem - 0.75rem);
  margin-bottom: calc(-3rem + 0.75rem);
  min-height: 0;

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

  &__header-meta {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-secondary;
    background: $bg-subtle;
    border: 1px solid $border-subtle;
    border-radius: 999px;
    padding: 7px 13px;
    white-space: nowrap;
    margin-bottom: 3px;
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
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &:focus-within {
      border-color: $primary;
      box-shadow: 0 0 0 3px $primary-light;
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
    transition: background 0.14s ease, color 0.14s ease;

    &:hover {
      background: $border-default;
      color: $text-primary;
    }
  }

  /* ---------- master-detail layout ---------- */
  &__layout {
    flex: 1;
    display: grid;
    grid-template-columns: 340px 1fr;
    gap: 16px;
    min-height: 0;
    overflow: hidden;
  }

  /* ---------- sidebar list ---------- */
  &__sidebar {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 16px;
    box-shadow: $shadow-xs;
    height: 100%;
    min-height: 0;
    display: flex;
    flex-direction: column;
    overflow: hidden;
  }

  &__sidebar-list {
    flex: 1;
    display: flex;
    flex-direction: column;
    min-height: 0;
    overflow-y: auto;
  }

  &__item {
    display: flex;
    align-items: flex-start;
    gap: 12px;
    padding: 14px 16px;
    text-align: left;
    border: none;
    border-bottom: 1px solid $border-subtle;
    border-left: 3px solid transparent;
    background: $bg-main;
    width: 100%;
    cursor: pointer;
    transition: background 0.12s ease, border-color 0.12s ease;

    &:last-child {
      border-bottom: none;
    }

    &:hover {
      background: $bg-subtle;
    }

    &--active {
      background: $primary-light;
      border-left-color: $primary;

      .providers-page__item-name {
        color: $primary;
      }
    }
  }

  &__item-body {
    flex: 1;
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 6px;
  }

  /* ---------- avatar ---------- */
  &__avatar {
    flex-shrink: 0;
    width: 34px;
    height: 34px;
    border-radius: 10px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    font-size: 0.75rem;
    font-weight: 700;
    letter-spacing: -0.01em;

    &--lg {
      width: 46px;
      height: 46px;
      border-radius: 12px;
      font-size: 1rem;
    }

    &--blue {
      color: $primary;
      background: $primary-light;
    }

    &--violet {
      color: $violet;
      background: $violet-light;
    }

    &--amber {
      color: $warning;
      background: $warning-subtle;
    }

    &--jade {
      color: $success;
      background: $success-subtle;
    }

    &--rose {
      color: $danger;
      background: $danger-subtle;
    }
  }

  &__item-top {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__item-name {
    font-size: 0.84375rem;
    font-weight: 600;
    color: $text-primary;
    line-height: 1.35;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__item-meta {
    display: flex;
    align-items: center;
    justify-content: space-between;
    font-size: 0.75rem;
    color: $text-tertiary;
  }

  /* ---------- status tag (shared by sidebar + detail) ---------- */
  &__tag {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 4px;
    font-size: 0.6875rem;
    font-weight: 600;
    border-radius: 999px;
    padding: 3px 10px;

    &--jade {
      color: $success;
      background: $success-subtle;
    }

    &--gray {
      color: $text-tertiary;
      background: $bg-inset;
    }
  }

  /* ---------- detail panel ---------- */
  &__detail {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 16px;
    box-shadow: $shadow-xs;
    padding: 26px 28px;
    height: 100%;
    min-height: 0;
    overflow-y: auto;
  }

  &__detail-head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
    padding-bottom: 18px;
    margin-bottom: 16px;
    border-bottom: 1px solid $border-subtle;
    position: sticky;
    top: -26px;
    padding-top: 26px;
    margin-top: -26px;
    background: $bg-main;
    z-index: 1;
  }

  &__detail-head-left {
    display: flex;
    align-items: center;
    gap: 14px;
  }

  &__detail-head-text {
    display: flex;
    flex-direction: column;
    gap: 8px;

    .providers-page__tag {
      width: fit-content;
    }
  }

  &__detail-name {
    font-size: 1.25rem;
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $text-primary;
  }

  &__detail-actions {
    display: flex;
    gap: 8px;
    flex-shrink: 0;
    flex-wrap: wrap;
  }

  &__detail-desc {
    font-size: 0.84375rem;
    line-height: 1.6;
    color: $text-secondary;
    margin-bottom: 24px;
  }

  /* ---------- stat cards ---------- */
  &__section-title {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 0.8125rem;
    font-weight: 700;
    color: $text-primary;
    margin-bottom: 12px;

    &:not(:first-of-type) {
      margin-top: 24px;
    }

    svg {
      opacity: 0.8;
    }
  }

  /* ---------- capability pills ---------- */
  &__caps {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
  }

  &__cap-pill {
    font-size: 0.71875rem;
    font-weight: 600;
    border-radius: 999px;
    padding: 3px 10px;

    &--blue {
      color: $primary;
      background: $primary-light;
    }

    &--violet {
      color: $violet;
      background: $violet-light;
    }

    &--amber {
      color: $warning;
      background: $warning-subtle;
    }

    &--jade {
      color: $success;
      background: $success-subtle;
    }

    &--rose {
      color: $danger;
      background: $danger-subtle;
    }
  }

  /* ---------- model list ---------- */
  &__model-list {
    display: flex;
    flex-direction: column;
    gap: 1px;
    border: 1px solid $border-subtle;
    border-radius: 12px;
    overflow: hidden;
  }

  &__model-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
    padding: 12px 14px;
    background: $bg-main;
    border-bottom: 1px solid $border-subtle;

    &:last-child {
      border-bottom: none;
    }

    &:hover {
      background: $bg-subtle;
    }
  }

  &__model-row-main {
    display: flex;
    align-items: baseline;
    gap: 8px;
    min-width: 0;
  }

  &__model-name {
    font-size: 0.8125rem;
    font-weight: 600;
    color: $text-primary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__model-version {
    flex-shrink: 0;
    font-family: $font-mono;
    font-size: 0.6875rem;
    color: $text-tertiary;
  }

  &__model-row-stats {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 16px;
  }

  &__model-pricing {
    font-size: 0.75rem;
    color: $text-secondary;
  }

  &__model-accuracy {
    font-size: 0.8125rem;
    font-weight: 700;
    color: $success;
    width: 42px;
    text-align: right;
  }

  &__stat-row {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 12px;
    margin-bottom: 8px;
  }

  &__stat-card {
    background: $bg-subtle;
    border-radius: 12px;
    padding: 14px 16px;
    display: flex;
    flex-direction: column;
    gap: 4px;
    min-width: 0;
  }

  &__stat-card-label {
    display: flex;
    align-items: center;
    gap: 5px;
    font-size: 0.6875rem;
    font-weight: 600;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $text-tertiary;

    svg {
      flex-shrink: 0;
      opacity: 0.8;
    }
  }

  &__stat-card-value {
    font-size: 1.25rem;
    font-weight: 800;
    color: $text-primary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;

    &--sm {
      font-size: 1rem;
      font-weight: 700;
    }

    &--accent {
      color: $success;
    }
  }

  /* ---------- inline connect form ---------- */
  &__connect-form {
    margin-top: 8px;
    display: flex;
    flex-direction: column;
    gap: 8px;
    max-width: 22rem;
  }

  &__field-label {
    font-size: 0.71875rem;
    font-weight: 600;
    color: $text-secondary;
  }

  &__input-wrap {
    display: flex;
    align-items: center;
    gap: 8px;
    border: 1px solid $border-default;
    border-radius: 8px;
    padding: 0 11px;
    background: $bg-main;
    color: $text-tertiary;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &:focus-within {
      border-color: $primary;
      box-shadow: 0 0 0 3px $primary-light;
    }
  }

  &__input {
    flex: 1;
    width: 100%;
    border: none;
    outline: none;
    padding: 8px 0;
    font-size: 0.8125rem;
    font-family: $font-body;
    color: $text-primary;
    background: transparent;

    &::placeholder {
      color: #a8b1bb;
    }
  }

  &__form-actions {
    display: flex;
    justify-content: flex-end;
    gap: 8px;
    margin-top: 2px;
  }

  /* ---------- buttons ---------- */
  &__btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 6px;
    font-family: $font-body;
    font-size: 0.8125rem;
    font-weight: 600;
    padding: 8px 14px;
    border-radius: 8px;
    border: 1px solid transparent;
    cursor: pointer;
    transition: background 0.14s ease, border-color 0.14s ease, color 0.14s ease;

    &--outline {
      background: $bg-main;
      border-color: $border-default;
      color: $text-primary;

      &:hover {
        border-color: $text-primary;
      }
    }

    &--danger-outline {
      background: $bg-main;
      border-color: $border-default;
      color: $text-tertiary;

      &:hover {
        border-color: $danger;
        color: $danger;
        background: $danger-subtle;
      }
    }

    &--primary {
      background: $primary;
      border-color: $primary;
      color: #fff;

      &:hover {
        background: $primary-hover;
        border-color: $primary-hover;
      }
    }
  }

  /* ---------- empty state ---------- */
  &__empty {
    flex-shrink: 0;
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

  /* ---------- responsive ---------- */
  @media (max-width: 900px) {
    &__layout {
      grid-template-columns: 1fr;
      grid-template-rows: 16rem 1fr;
      overflow: visible;
    }

    &__sidebar,
    &__detail {
      height: auto;
      max-height: 22rem;
    }

    &__stat-row {
      grid-template-columns: repeat(2, 1fr);
    }
  }

  @media (max-width: 520px) {
    &__detail {
      padding: 18px 16px;
    }

    &__detail-head {
      flex-direction: column;
    }

    &__stat-row {
      grid-template-columns: 1fr;
    }
  }
}






















//Providers.tsx
import { useEffect, useMemo, useState, type FC, type FormEvent } from 'react';
import { PlugZap, CheckCircle2, Boxes, Settings2, Unplug, Search, X, Key, Layers, Gauge } from 'lucide-react';
import { PROVIDERS, MODELS } from '../RunEvaluation/data';
import type { Provider } from '../RunEvaluation/types';
import './Providers.scss';

const AVATAR_TINTS = ['blue', 'violet', 'amber', 'jade', 'rose'] as const;

function avatarTint(id: string) {
  let hash = 0;
  for (let i = 0; i < id.length; i += 1) hash = (hash * 31 + id.charCodeAt(i)) >>> 0;
  return AVATAR_TINTS[hash % AVATAR_TINTS.length];
}

function pillTint(capability: string) {
  let hash = 0;
  for (let i = 0; i < capability.length; i += 1) hash = (hash * 31 + capability.charCodeAt(i)) >>> 0;
  return AVATAR_TINTS[hash % AVATAR_TINTS.length];
}

const Providers: FC = () => {
  const [providers, setProviders] = useState<Provider[]>(PROVIDERS);
  const [query, setQuery] = useState('');
  const [selectedId, setSelectedId] = useState<string | null>(PROVIDERS[0]?.id ?? null);
  const [formOpen, setFormOpen] = useState(false);
  const [keyInput, setKeyInput] = useState('');

  const connectedCount = providers.filter((p) => p.status === 'connected').length;

  const filtered = useMemo(
    () => providers.filter((p) => !query || p.name.toLowerCase().includes(query.toLowerCase())),
    [providers, query]
  );

  const selected = useMemo(
    () => filtered.find((p) => p.id === selectedId) ?? filtered[0] ?? null,
    [filtered, selectedId]
  );

  const providerModels = useMemo(
    () => (selected ? MODELS.filter((m) => m.providerId === selected.id) : []),
    [selected]
  );

  const topCapabilities = useMemo(() => {
    const counts = new Map<string, number>();
    providerModels.forEach((m) => m.capabilities.forEach((c) => counts.set(c, (counts.get(c) ?? 0) + 1)));
    return Array.from(counts.entries())
      .sort((a, b) => b[1] - a[1])
      .slice(0, 6)
      .map(([name]) => name);
  }, [providerModels]);

  const avgAccuracy = useMemo(() => {
    if (providerModels.length === 0) return null;
    const sum = providerModels.reduce((acc, m) => acc + m.accuracyScore, 0);
    return (sum / providerModels.length).toFixed(1);
  }, [providerModels]);

  useEffect(() => {
    setFormOpen(false);
    setKeyInput('');
  }, [selected?.id]);

  const openForm = () => {
    setKeyInput(selected?.apiKey ?? '');
    setFormOpen(true);
  };

  const handleSubmit = (e: FormEvent) => {
    e.preventDefault();
    if (!selected || !keyInput.trim()) return;
    setProviders((prev) =>
      prev.map((p) => (p.id === selected.id ? { ...p, status: 'connected', apiKey: `sk-****-${keyInput.slice(-4)}` } : p))
    );
    setFormOpen(false);
  };

  const handleDisconnect = () => {
    if (!selected) return;
    setProviders((prev) => prev.map((p) => (p.id === selected.id ? { ...p, status: 'not_connected', apiKey: undefined } : p)));
    setFormOpen(false);
  };

  return (
    <div className="providers-page">
      <div className="providers-page__header">
        <div className="providers-page__header-left">
          <p className="providers-page__header-eyebrow">Provider connections</p>
          <h1 className="providers-page__title">Providers</h1>
          <p className="providers-page__subtitle">Manage AI service connections and API keys</p>
        </div>

        <div className="providers-page__header-meta">
          <PlugZap size={13} />
          {connectedCount} of {providers.length} connected
        </div>
      </div>

      <div className="providers-page__search">
        <Search size={15} />
        <input type="text" placeholder="Search providers..." value={query} onChange={(e) => setQuery(e.target.value)} />
        {query && (
          <button type="button" className="providers-page__search-clear" onClick={() => setQuery('')} aria-label="Clear search">
            <X size={13} />
          </button>
        )}
      </div>

      {filtered.length === 0 ? (
        <div className="providers-page__empty">
          <Search size={22} />
          <p>No providers match your search.</p>
        </div>
      ) : (
        <div className="providers-page__layout">
          <div className="providers-page__sidebar">
            <div className="providers-page__sidebar-list">
              {filtered.map((p) => {
                const connected = p.status === 'connected';
                const isActive = selected?.id === p.id;
                const tint = avatarTint(p.id);
                return (
                  <button
                    type="button"
                    key={p.id}
                    className={`providers-page__item${isActive ? ' providers-page__item--active' : ''}`}
                    onClick={() => setSelectedId(p.id)}
                  >
                    <span className={`providers-page__avatar providers-page__avatar--${tint}`}>{p.logo}</span>
                    <div className="providers-page__item-body">
                      <div className="providers-page__item-top">
                        <span className="providers-page__item-name">{p.name}</span>
                        <span className={`providers-page__tag${connected ? ' providers-page__tag--jade' : ' providers-page__tag--gray'}`}>
                          {connected && <CheckCircle2 size={11} strokeWidth={2.5} />}
                          {connected ? 'Connected' : 'Not connected'}
                        </span>
                      </div>
                      <div className="providers-page__item-meta">
                        <span>{p.modelCount} models</span>
                      </div>
                    </div>
                  </button>
                );
              })}
            </div>
          </div>

          {selected && (
            <div className="providers-page__detail">
              {(() => {
                const connected = selected.status === 'connected';
                const tint = avatarTint(selected.id);
                return (
                  <>
                    <div className="providers-page__detail-head">
                      <div className="providers-page__detail-head-left">
                        <span className={`providers-page__avatar providers-page__avatar--${tint} providers-page__avatar--lg`}>
                          {selected.logo}
                        </span>
                        <div className="providers-page__detail-head-text">
                          <span className={`providers-page__tag${connected ? ' providers-page__tag--jade' : ' providers-page__tag--gray'}`}>
                            {connected && <CheckCircle2 size={11} strokeWidth={2.5} />}
                            {connected ? 'Connected' : 'Not connected'}
                          </span>
                          <h2 className="providers-page__detail-name">{selected.name}</h2>
                        </div>
                      </div>

                      {!formOpen && (
                        <div className="providers-page__detail-actions">
                          {connected ? (
                            <>
                              <button type="button" className="providers-page__btn providers-page__btn--outline" onClick={openForm}>
                                <Settings2 size={13} /> Configure
                              </button>
                              <button
                                type="button"
                                className="providers-page__btn providers-page__btn--danger-outline"
                                onClick={handleDisconnect}
                              >
                                <Unplug size={13} /> Disconnect
                              </button>
                            </>
                          ) : (
                            <button type="button" className="providers-page__btn providers-page__btn--primary" onClick={openForm}>
                              <PlugZap size={13} /> Connect
                            </button>
                          )}
                        </div>
                      )}
                    </div>

                    <p className="providers-page__detail-desc">{selected.desc}</p>

                    <div className="providers-page__stat-row">
                      <div className="providers-page__stat-card">
                        <span className="providers-page__stat-card-label">
                          <Boxes size={11} /> Models Available
                        </span>
                        <span className="providers-page__stat-card-value n">{selected.modelCount}</span>
                      </div>
                      <div className="providers-page__stat-card">
                        <span className="providers-page__stat-card-label">
                          <Gauge size={11} /> Avg. Accuracy
                        </span>
                        <span className="providers-page__stat-card-value providers-page__stat-card-value--accent n">
                          {avgAccuracy ? `${avgAccuracy}%` : '—'}
                        </span>
                      </div>
                      <div className="providers-page__stat-card">
                        <span className="providers-page__stat-card-label">
                          <Key size={11} /> API Key
                        </span>
                        <span className="providers-page__stat-card-value providers-page__stat-card-value--sm">
                          {connected ? selected.apiKey : '—'}
                        </span>
                      </div>
                    </div>

                    {formOpen && (
                      <>
                        <p className="providers-page__section-title">{connected ? 'Update API key' : 'Connect provider'}</p>
                        <form className="providers-page__connect-form" onSubmit={handleSubmit}>
                          <label className="providers-page__field-label" htmlFor={`key-${selected.id}`}>
                            API Key
                          </label>
                          <div className="providers-page__input-wrap">
                            <Key size={14} />
                            <input
                              id={`key-${selected.id}`}
                              type="password"
                              className="providers-page__input"
                              placeholder="Enter API key"
                              value={keyInput}
                              onChange={(e) => setKeyInput(e.target.value)}
                              autoFocus
                            />
                          </div>
                          <div className="providers-page__form-actions">
                            <button type="button" className="providers-page__btn providers-page__btn--outline" onClick={() => setFormOpen(false)}>
                              Cancel
                            </button>
                            <button type="submit" className="providers-page__btn providers-page__btn--primary">
                              Save
                            </button>
                          </div>
                        </form>
                      </>
                    )}

                    {topCapabilities.length > 0 && (
                      <>
                        <p className="providers-page__section-title">
                          <Layers size={13} /> Capability coverage
                        </p>
                        <div className="providers-page__caps">
                          {topCapabilities.map((c) => (
                            <span key={c} className={`providers-page__cap-pill providers-page__cap-pill--${pillTint(c)}`}>
                              {c}
                            </span>
                          ))}
                        </div>
                      </>
                    )}

                    {providerModels.length > 0 && (
                      <>
                        <p className="providers-page__section-title">Models from {selected.name}</p>
                        <div className="providers-page__model-list">
                          {providerModels.map((m) => (
                            <div className="providers-page__model-row" key={m.id}>
                              <div className="providers-page__model-row-main">
                                <span className="providers-page__model-name">{m.name}</span>
                                <span className="providers-page__model-version">{m.version}</span>
                              </div>
                              <div className="providers-page__model-row-stats">
                                <span className="providers-page__model-pricing n">{m.pricing}</span>
                                <span className="providers-page__model-accuracy n">{m.accuracyScore}%</span>
                              </div>
                            </div>
                          ))}
                        </div>
                      </>
                    )}
                  </>
                );
              })()}
            </div>
          )}
        </div>
      )}
    </div>
  );
};

export default Providers;






















//RunEvaluation.scss
@use '../../../styles/variables' as *;

.run-eval {
  max-width: 1024px;
  margin: 0 auto;
  height: calc(100vh - 166px);
  display: flex;
  flex-direction: column;
  min-height: 0;

  @media (min-width: 1800px) {
    max-width: 1300px;
  }

  /* ---------- page header ---------- */
  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding-bottom: 18px;
    margin-bottom: 20px;
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

  &__header-meta {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-secondary;
    background: $bg-subtle;
    border: 1px solid $border-subtle;
    border-radius: 999px;
    padding: 7px 13px;
    white-space: nowrap;
    margin-bottom: 3px;
  }

  &__title {
    font-size: 21px;
    font-weight: 800;
    letter-spacing: -0.03em;
    color: $text-primary;
    line-height: 1.15;
  }

  &__subtitle {
    margin-top: 3px;
    color: $text-secondary;
    font-size: 0.84375rem;
  }

  /* ---------- buttons ---------- */
  &__btn {
    display: inline-flex;
    align-items: center;
    gap: 0.5rem;
    font-size: 0.90625rem;
    font-weight: 600;
    padding: 0.5625rem 0.9375rem;
    border-radius: 0.5rem;
    border: 1px solid transparent;
    cursor: pointer;
    transition: background 0.14s ease, border-color 0.14s ease, color 0.14s ease;
    font-family: $font-body;

    &--primary {
      background: $primary;
      color: $on-primary;
      border-color: $primary;

      &:hover {
        background: $primary-hover;
        border-color: $primary-hover;
      }
    }

    &--secondary {
      background: $bg-main;
      color: $text-primary;
      border-color: $border-default;

      &:hover {
        border-color: $text-primary;
      }
    }

    &--lg {
      padding: 0.625rem 1.125rem;
      font-size: 0.90625rem;
    }

    &--sm {
      padding: 0.375rem 0.6875rem;
      font-size: 0.84375rem;
      background: $bg-main;
      color: $text-secondary;
      border-color: $border-default;

      &:hover {
        border-color: $text-primary;
        color: $text-primary;
      }
    }

    &:disabled {
      opacity: 0.5;
      cursor: not-allowed;
    }
  }

  /* ---------- wizard shell ---------- */
  &__wizard {
    position: relative;
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 20px;
    box-shadow: $shadow-md;
    padding: 32px 36px 24px;
    overflow: hidden;
    flex: 1;
    min-height: 0;
    display: flex;
    flex-direction: column;

    &::before {
      content: '';
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      height: 3px;
      background: linear-gradient(90deg, $primary, $primary-hover 60%, $success);
    }
  }

  &__body {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding-right: 4px;
    margin-right: -4px;
  }

  /* ---------- progress tracker ---------- */
  &__tracker {
    position: relative;
    padding-bottom: 28px;
    margin-bottom: 4px;
    flex-shrink: 0;
  }

  &__tracker-bar {
    position: absolute;
    top: 18px;
    left: 18px;
    right: 18px;
    height: 2px;
    background: $border-default;
    border-radius: 2px;
  }

  &__tracker-fill {
    height: 100%;
    background: linear-gradient(90deg, $primary, $primary-hover);
    border-radius: 2px;
    transition: width 0.25s ease;
  }

  &__tracker-nodes {
    position: relative;
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
  }

  &__node {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 8px;
    border: none;
    background: transparent;
    cursor: pointer;
    padding: 0;
    flex: 1;

    &:disabled {
      cursor: default;
    }
  }

  &__node-dot {
    width: 36px;
    height: 36px;
    border-radius: 50%;
    display: grid;
    place-items: center;
    font-size: 0.8125rem;
    font-weight: 700;
    background: $bg-main;
    border: 2px solid $border-default;
    color: $text-tertiary;
    transition: background 0.16s ease, border-color 0.16s ease, color 0.16s ease, transform 0.16s ease;
  }

  &__node-label {
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-tertiary;
    text-align: center;
    max-width: 88px;
    line-height: 1.25;
    transition: color 0.16s ease;
  }

  &__node--active {
    .run-eval__node-dot {
      background: $primary;
      border-color: $primary;
      color: $on-primary;
      transform: scale(1.12);
      box-shadow: 0 0 0 4px $primary-light;
    }

    .run-eval__node-label {
      color: $primary;
    }
  }

  &__node--complete {
    .run-eval__node-dot {
      background: $success-subtle;
      border-color: $success;
      color: $success;
    }

    .run-eval__node-label {
      color: $text-secondary;
    }

    &:hover .run-eval__node-dot {
      border-color: $success;
      background: $success;
      color: $on-primary;
    }
  }

  /* ---------- step kicker + heading ---------- */
  &__step-kicker {
    display: flex;
    align-items: center;
    gap: 8px;
    font-family: $font-mono;
    font-size: 0.75rem;
    font-weight: 700;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    color: $primary;
    margin-bottom: 8px;
    flex-shrink: 0;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $primary;
    }
  }

  /* ---------- step cards ---------- */
  &__card {
    &--wide {
      max-width: none;
    }
  }

  &__step-title {
    font-size: 19px;
    font-weight: 800;
    letter-spacing: -0.02em;
    line-height: 1.2;
    color: $text-primary;
  }

  &__step-desc {
    margin-top: 6px;
    font-size: 0.9375rem;
    color: $text-secondary;
    max-width: 608px;
  }

  &__step-header-row {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 1rem;
  }

  &__field {
    max-width: 480px;
    margin-top: 1.75rem;
  }

  &__label {
    display: block;
    font-size: 0.84375rem;
    font-weight: 600;
    color: $text-secondary;
    margin-bottom: 0.4375rem;
  }

  &__input {
    width: 100%;
    border: 1px solid $border-default;
    border-radius: 0.5rem;
    padding: 0.625rem 0.75rem;
    font-size: 0.9375rem;
    font-family: $font-body;
    color: $text-primary;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &::placeholder {
      color: $text-tertiary;
    }

    &:focus {
      outline: none;
      border-color: $primary;
      box-shadow: 0 0 0 0.1875rem $primary-light;
    }

    &--lg {
      padding: 0.75rem 0.875rem;
      font-size: 1rem;
    }
  }

  /* ---------- suggestion / static chips ---------- */
  &__suggestions {
    margin-top: 1.125rem;
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 0.5rem;
  }

  &__suggestions-label {
    font-size: 0.84375rem;
    color: $text-tertiary;
    margin-right: 0.125rem;
  }

  &__chip {
    font-size: 0.8125rem;
    font-weight: 500;
    color: $text-secondary;
    background: $bg-subtle;
    border: 1px solid $border-default;
    border-radius: 999px;
    padding: 0.3125rem 0.75rem;
    cursor: pointer;
    transition: border-color 0.14s ease, color 0.14s ease, background 0.14s ease;

    &:hover {
      border-color: $primary;
      color: $primary;
    }

    &--active {
      background: $primary;
      border-color: $primary;
      color: $on-primary;
    }

    &--static {
      cursor: default;
      font-size: 0.75rem;
      padding: 0.1875rem 0.5rem;

      &:hover {
        border-color: $border-default;
        color: $text-secondary;
      }
    }
  }

  /* ---------- eval type cards ---------- */
  &__type-grid {
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
    margin-top: 1.5rem;
  }

  &__type-card {
    position: relative;
    display: flex;
    align-items: flex-start;
    gap: 0.875rem;
    text-align: left;
    width: 100%;
    padding: 1.125rem 3rem 1.125rem 1.125rem;
    border: 1px solid $border-default;
    border-radius: 0.75rem;
    background: $bg-main;
    cursor: pointer;
    transition: border-color 0.14s ease, background 0.14s ease;

    &:hover {
      border-color: $primary;
    }

    &--selected {
      border-color: $primary;
      background: $primary-light;
    }
  }

  &__type-icon {
    width: 38px;
    height: 38px;
    flex-shrink: 0;
    border-radius: 0.5rem;
    background: $bg-subtle;
    color: $primary;
    display: grid;
    place-items: center;
  }

  &__type-content {
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
    flex: 1;
  }

  &__type-title {
    font-size: 1rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__type-desc {
    font-size: 0.875rem;
    color: $text-secondary;
    line-height: 1.5;
  }

  &__type-check {
    position: absolute;
    top: 1.125rem;
    right: 1.125rem;
    width: 20px;
    height: 20px;
    border-radius: 50%;
    background: $primary;
    color: $on-primary;
    display: grid;
    place-items: center;
  }

  &__badge {
    align-self: flex-start;
    flex-shrink: 0;
    font-size: 0.71875rem;
    font-weight: 600;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $primary;
    background: $primary-light;
    border-radius: 0.375rem;
    padding: 0.25rem 0.5rem;

    &--soft {
      margin-top: 0.625rem;
    }
  }

  /* ---------- providers ---------- */
  &__provider-grid {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 0.75rem;
    margin-top: 1.5rem;
  }

  &__provider-card {
    position: relative;
    display: flex;
    align-items: flex-start;
    gap: 0.75rem;
    text-align: left;
    padding: 0.875rem 2.5rem 0.875rem 0.875rem;
    border: 1px solid $border-default;
    border-radius: 0.75rem;
    background: $bg-main;
    cursor: pointer;
    transition: border-color 0.14s ease, background 0.14s ease;

    &:hover:not(&--disabled) {
      border-color: $primary;
    }

    &--selected {
      border-color: $primary;
      background: $primary-light;
    }

    &--disabled {
      cursor: not-allowed;
      opacity: 0.7;
    }
  }

  &__provider-logo {
    width: 34px;
    height: 34px;
    flex-shrink: 0;
    border-radius: 0.5rem;
    background: $text-primary;
    color: $bg-main;
    font-weight: 700;
    font-size: 0.875rem;
    display: grid;
    place-items: center;
    margin-top: 0.0625rem;
  }

  &__provider-info {
    display: flex;
    flex-direction: column;
    gap: 0.3125rem;
    min-width: 0;
    flex: 1;
  }

  &__provider-name-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 0.5rem;
  }

  &__provider-name {
    font-size: 0.9375rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__provider-desc {
    font-size: 0.8125rem;
    color: $text-tertiary;
  }

  &__status-badge {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 0.3125rem;
    font-size: 0.71875rem;
    font-weight: 600;
    letter-spacing: 0.01em;
    color: $text-tertiary;
    background: $bg-inset;
    border-radius: 999px;
    padding: 0.1875rem 0.5rem;

    &--on {
      color: $success;
      background: $success-subtle;
    }
  }

  &__hint {
    margin-top: 1.25rem;
    display: flex;
    align-items: center;
    gap: 0.5rem;
    font-size: 0.875rem;
    color: $text-tertiary;

    svg {
      flex-shrink: 0;
    }
  }

  &__link {
    background: none;
    border: none;
    padding: 0;
    color: $primary;
    font-weight: 600;
    font-size: inherit;
    cursor: pointer;

    &:hover {
      text-decoration: underline;
    }
  }

  /* ---------- models step ---------- */
  &__models-layout {
    display: grid;
    grid-template-columns: 15rem 1fr;
    gap: 1.5rem;
    margin-top: 1.5rem;
    align-items: start;
  }

  &__filters {
    border: 1px solid $border-subtle;
    border-radius: 0.75rem;
    padding: 1rem;
    display: flex;
    flex-direction: column;
    gap: 1.125rem;
  }

  &__filters-head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    font-size: 0.875rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__filter-section {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
  }

  &__filter-title {
    font-family: $font-mono;
    font-size: 0.71875rem;
    font-weight: 600;
    letter-spacing: 0.08em;
    text-transform: uppercase;
    color: $text-tertiary;
  }

  &__filter-options {
    display: flex;
    flex-direction: column;
    gap: 0.4375rem;
  }

  &__filter-chip {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    font-size: 0.875rem;
    color: $text-secondary;
    cursor: pointer;

    input {
      accent-color: $primary;
    }
  }

  &__models-main {
    min-width: 0;
  }

  &__search-bar {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    border: 1px solid $border-default;
    border-radius: 0.5rem;
    padding: 0.5625rem 0.75rem;
    color: $text-tertiary;

    input {
      flex: 1;
      border: none;
      outline: none;
      font-size: 0.90625rem;
      color: $text-primary;
      background: transparent;
      font-family: $font-body;

      &::placeholder {
        color: $text-tertiary;
      }
    }
  }

  &__active-filters {
    display: flex;
    flex-wrap: wrap;
    gap: 0.375rem;
    margin-top: 0.75rem;
  }

  &__tag {
    display: inline-flex;
    align-items: center;
    gap: 0.375rem;
    font-size: 0.78125rem;
    color: $primary;
    background: $primary-light;
    border-radius: 0.375rem;
    padding: 0.25rem 0.25rem 0.25rem 0.5rem;

    button {
      display: grid;
      place-items: center;
      border: none;
      background: transparent;
      color: inherit;
      cursor: pointer;
    }
  }

  &__models-grid {
    margin-top: 1rem;
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 0.75rem;
  }

  &__model-card {
    position: relative;
    text-align: left;
    padding: 0.875rem 1rem;
    border: 1px solid $border-default;
    border-radius: 0.75rem;
    background: $bg-main;
    cursor: pointer;
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    transition: border-color 0.14s ease, background 0.14s ease;

    &:hover {
      border-color: $primary;
    }

    &--selected {
      border-color: $primary;
      background: $primary-light;
    }
  }

  &__model-top {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 0.5rem;
  }

  &__model-name {
    font-size: 0.9375rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__model-provider {
    font-size: 0.84375rem;
    color: $text-tertiary;
    margin-top: -0.25rem;
  }

  &__model-caps {
    display: flex;
    flex-wrap: wrap;
    gap: 0.3125rem;
  }

  &__model-meta {
    display: flex;
    flex-wrap: wrap;
    gap: 0.625rem;
    font-size: 0.78125rem;
    color: $text-tertiary;
    margin-top: 0.125rem;
  }

  &__empty {
    grid-column: 1 / -1;
    padding: 2rem;
    text-align: center;
    color: $text-tertiary;
    font-size: 0.90625rem;
  }

  &__selected-bar {
    margin-top: 1rem;
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0.75rem 1rem;
    background: $primary-light;
    border-radius: 0.625rem;
    font-size: 0.90625rem;
    color: $text-primary;

    strong {
      color: $primary;
    }
  }

  /* ---------- dataset step ---------- */
  &__tabs {
    display: flex;
    gap: 0.375rem;
    margin-top: 1.5rem;
    border-bottom: 1px solid $border-subtle;
  }

  &__tab {
    padding: 0.5625rem 0.25rem;
    margin-right: 1.25rem;
    border: none;
    background: transparent;
    font-size: 0.90625rem;
    font-weight: 600;
    color: $text-tertiary;
    cursor: pointer;
    border-bottom: 2px solid transparent;
    transition: color 0.14s ease, border-color 0.14s ease;

    &--active {
      color: $primary;
      border-bottom-color: $primary;
    }
  }

  &__category-filters {
    display: flex;
    flex-wrap: wrap;
    gap: 0.5rem;
    margin-top: 1.125rem;
  }

  &__dataset-grid {
    margin-top: 1.125rem;
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 0.75rem;
  }

  &__dataset-card {
    position: relative;
    text-align: left;
    padding: 1rem 1.125rem;
    border: 1px solid $border-default;
    border-radius: 0.75rem;
    background: $bg-main;
    cursor: pointer;
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    transition: border-color 0.14s ease, background 0.14s ease;

    &:hover {
      border-color: $primary;
    }

    &--selected {
      border-color: $primary;
      background: $primary-light;
    }
  }

  &__dataset-top {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 0.5rem;
  }

  &__dataset-name {
    font-size: 0.9375rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__dataset-desc {
    font-size: 0.875rem;
    color: $text-secondary;
    line-height: 1.5;
  }

  &__dataset-meta {
    display: flex;
    flex-wrap: wrap;
    gap: 0.625rem;
    font-size: 0.78125rem;
    color: $text-tertiary;
  }

  &__empty-state,
  &__upload-zone {
    margin-top: 1.5rem;
    border: 1.5008px dashed $border-strong;
    border-radius: 0.75rem;
    padding: 2.75rem 1.5rem;
    text-align: center;
    color: $text-tertiary;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 0.5rem;

    svg {
      color: $text-tertiary;
      margin-bottom: 0.25rem;
    }

    h3 {
      font-size: 1rem;
      color: $text-primary;
    }

    p {
      font-size: 0.875rem;
    }
  }

  &__upload-zone {
    cursor: pointer;

    &:hover {
      border-color: $primary;
    }
  }

  &__format-chips {
    display: flex;
    gap: 0.375rem;
    margin-top: 0.375rem;
  }

  /* ---------- metrics step ---------- */
  &__metrics-count {
    flex-shrink: 0;
    font-size: 0.875rem;
    color: $text-secondary;

    span {
      font-weight: 700;
      color: $primary;
    }
  }

  &__metric-group {
    margin-top: 1.5rem;
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
  }

  &__metrics-grid {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 0.625rem;
  }

  &__metric-card {
    position: relative;
    text-align: left;
    padding: 0.75rem 2rem 0.75rem 0.875rem;
    border: 1px solid $border-default;
    border-radius: 0.625rem;
    background: $bg-main;
    cursor: pointer;
    transition: border-color 0.14s ease, background 0.14s ease;

    &:hover {
      border-color: $primary;
    }

    &--selected {
      border-color: $primary;
      background: $primary-light;
    }
  }

  &__metric-name {
    display: block;
    font-size: 0.875rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__metric-tooltip {
    display: block;
    margin-top: 0.25rem;
    font-size: 0.78125rem;
    color: $text-tertiary;
    line-height: 1.4;
  }

  /* ---------- review step ---------- */
  &__review {
    margin-top: 1.5rem;
    border: 1px solid $border-subtle;
    border-radius: 0.75rem;
    overflow: hidden;
  }

  &__review-row {
    display: flex;
    align-items: baseline;
    justify-content: space-between;
    gap: 1rem;
    padding: 0.75rem 1rem;
    font-size: 0.90625rem;
    border-bottom: 1px solid $border-subtle;

    &:last-child {
      border-bottom: 0;
    }

    span:first-child {
      color: $text-tertiary;
      flex-shrink: 0;
    }

    span:last-child {
      color: $text-primary;
      font-weight: 500;
      text-align: right;
    }

    &--highlight span:last-child {
      color: $primary;
      font-weight: 700;
    }
  }

  &__review-divider {
    height: 1px;
    background: $border-subtle;
  }

  /* ---------- shared feedback ---------- */
  &__error {
    margin-top: 1.25rem;
    font-size: 0.875rem;
    color: $danger;
    background: $danger-subtle;
    border-radius: 0.5rem;
    padding: 0.625rem 0.875rem;
  }

  &__nav {
    flex-shrink: 0;
    margin-top: 1.25rem;
    padding-top: 1.25rem;
    border-top: 1px solid $border-subtle;
    display: flex;
    align-items: center;
    justify-content: space-between;
  }

  /* ---------- responsive ---------- */
  @media (max-width: 896px) {
    &__provider-grid,
    &__models-grid,
    &__dataset-grid,
    &__metrics-grid {
      grid-template-columns: 1fr;
    }

    &__models-layout {
      grid-template-columns: 1fr;
    }

    &__node-label {
      display: none;
    }
  }
}




















//uiSlice.ts
import { createSlice, type PayloadAction } from '@reduxjs/toolkit';

export type Theme = 'light' | 'dark';

export interface UiState {
  sidebarCollapsed: boolean;
  activeNavItem: string;
  theme: Theme;
}

const THEME_STORAGE_KEY = 'semcoeval-theme';

function getInitialTheme(): Theme {
  if (typeof window === 'undefined') return 'light';

  const stored = window.localStorage.getItem(THEME_STORAGE_KEY);
  if (stored === 'light' || stored === 'dark') return stored;

  return window.matchMedia?.('(prefers-color-scheme: dark)').matches ? 'dark' : 'light';
}

const initialState: UiState = {
  sidebarCollapsed: false,
  activeNavItem: 'dashboard',
  theme: getInitialTheme(),
};

const uiSlice = createSlice({
  name: 'ui',
  initialState,
  reducers: {
    toggleSidebar: (state) => {
      state.sidebarCollapsed = !state.sidebarCollapsed;
    },
    setSidebarCollapsed: (state, action: PayloadAction<boolean>) => {
      state.sidebarCollapsed = action.payload;
    },
    setActiveNavItem: (state, action: PayloadAction<string>) => {
      state.activeNavItem = action.payload;
    },
    toggleTheme: (state) => {
      state.theme = state.theme === 'dark' ? 'light' : 'dark';
    },
    setTheme: (state, action: PayloadAction<Theme>) => {
      state.theme = action.payload;
    },
  },
});

export const { toggleSidebar, setSidebarCollapsed, setActiveNavItem, toggleTheme, setTheme } = uiSlice.actions;
export default uiSlice.reducer;
