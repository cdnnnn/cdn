//Footer.scss
@use '../../styles/variables' as *;

.app-footer {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  z-index: $z-footer;
  height: $footer-height;
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.75rem;
  padding: 0 1.125rem;
  background: #fff;
  border-top: 0.0625rem solid $border-subtle;

  &__version,
  &__copyright {
    font-size: 0.65625rem;
    color: $text-tertiary;
    white-space: nowrap;
  }
}













//Footer.tsx
import type { FC } from 'react';
import './Footer.scss';

const APP_VERSION = 'v1.0.0';

const Footer: FC = () => {
  const year = new Date().getFullYear();

  return (
    <footer className="app-footer">
      <span className="app-footer__version">{APP_VERSION}</span>
      <span className="app-footer__copyright">&copy; {year} SemcoEval</span>
    </footer>
  );
};

export default Footer;

















//Header.scss
@use '../../styles/variables' as *;

.app-header {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  z-index: $z-header;
  height: $header-height;
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.75rem;
  padding: 0 1.125rem;
  background: rgba(255, 255, 255, 0.88);
  backdrop-filter: blur(0.75rem);
  border-bottom: 0.0625rem solid $border-subtle;

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
    box-shadow: 0 0.0625rem 0.125rem rgba(14, 21, 38, 0.06), inset 0 0 0 0.0625rem rgba(255, 255, 255, 0.14);
  }

  &__brand-name {
    font-family: $font-display;
    font-weight: 700;
    font-size: 1.03rem;
    letter-spacing: -0.02em;
    white-space: nowrap;
  }

  &__user {
    position: relative;
    flex-shrink: 0;
  }

  &__avatar {
    width: 34px;
    height: 34px;
    border-radius: 50%;
    border: 0.0625rem solid $border-default;
    background: $primary-light;
    color: $primary;
    font-family: $font-display;
    font-size: 0.75rem;
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
    border: 0.0625rem solid $border-subtle;
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
    font-size: 0.78125rem;
    font-weight: 700;
    display: grid;
    place-items: center;
  }

  &__drop-info {
    min-width: 0;
  }

  &__drop-name {
    font-size: 0.84375rem;
    font-weight: 600;
    color: $text-primary;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  &__drop-role {
    font-size: 0.71875rem;
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
    font-size: 0.8125rem;
    font-weight: 500;
    cursor: pointer;
    transition: background 0.14s ease;

    &:hover {
      background: $danger-subtle;
    }
  }

  @media (max-width: 620px) {
    &__brand-name {
      font-size: 0.95rem;
    }
  }
}



















//Header.tsx
import { Link } from 'react-router-dom';
import { useEffect, useRef, useState, type FC } from 'react';
import { Gauge, LogOut } from 'lucide-react';
import { useAppSelector } from '../../store/hooks';
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
  const user = useAppSelector((s) => s.user.user);
  const [menuOpen, setMenuOpen] = useState(false);
  const menuRef = useRef<HTMLDivElement>(null);
  const initials = getInitials(user.name);

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
    </header>
  );
};

export default Header;















//PagePlaceholder.scss
@use '../../styles/variables' as *;

.page-placeholder {
  &__title {
    font-size: 1.6875rem;
    font-weight: 700;
    color: $text-primary;
  }

  &__subtitle {
    margin-top: 0.375rem;
    color: $text-secondary;
    font-size: 0.90625rem;
  }

  &__box {
    margin-top: 1.5rem;
    border: 0.0938rem dashed $border-strong;
    border-radius: 0.75rem;
    padding: 3rem 1.5rem;
    text-align: center;
    color: $text-tertiary;
    font-size: 0.875rem;
    background: $bg-main;
  }
}













//PagePlaceholder.tsx
import type { FC } from 'react';
import './PagePlaceholder.scss';

interface PagePlaceholderProps {
  title: string;
  subtitle?: string;
}

const PagePlaceholder: FC<PagePlaceholderProps> = ({ title, subtitle }) => {
  return (
    <div className="page-placeholder">
      <h1 className="page-placeholder__title">{title}</h1>
      {subtitle && <p className="page-placeholder__subtitle">{subtitle}</p>}
      <div className="page-placeholder__box">Content coming soon</div>
    </div>
  );
};

export default PagePlaceholder;















//Sidebar.scss
@use '../../styles/variables' as *;

$sidebar-width-collapsed: 68px;

.app-sidebar {
  width: $sidebar-width;
  flex-shrink: 0;
  height: 100%;
  overflow-y: auto;
  overflow-x: hidden;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  background: $bg-main;
  border-right: 0.0625rem solid $border-subtle;
  padding: 0.75rem 0.875rem 1.125rem;
  transition: width 0.18s ease;

  &--collapsed {
    width: $sidebar-width-collapsed;
    padding-left: 0.625rem;
    padding-right: 0.625rem;
  }

  &__top {
    flex: 1;
    display: flex;
    flex-direction: column;
    min-height: 0;
  }

  /* ---------- collapse toggle ---------- */
  &__collapse {
    display: flex;
    align-items: center;
    justify-content: space-between;
    width: 100%;
    padding: 0.375rem 0.5rem;
    margin-bottom: 0.75rem;
    border: none;
    border-radius: 0.5rem;
    background: transparent;
    color: $text-tertiary;
    font-size: 0.75rem;
    font-weight: 600;
    cursor: pointer;
    transition: background 0.14s ease, color 0.14s ease;

    &:hover {
      background: $bg-subtle;
      color: $text-secondary;
    }
  }

  &--collapsed &__collapse {
    justify-content: center;
  }

  &__collapse-ic {
    transition: transform 0.18s ease;
    flex-shrink: 0;

    &--flipped {
      transform: rotate(180deg);
    }
  }

  /* ---------- nav sections ---------- */
  &__nav {
    display: flex;
    flex-direction: column;
    gap: 1.375rem;
    margin-top: 0.5rem;
  }

  &__section {
    ul {
      list-style: none;
      display: flex;
      flex-direction: column;
      gap: 0.125rem;
    }
  }

  &__section-label {
    font-family: $font-mono;
    font-size: 0.6625rem;
    font-weight: 600;
    letter-spacing: 0.09em;
    text-transform: uppercase;
    color: $text-tertiary;
    padding: 0 0.625rem;
    margin-bottom: 0.375rem;
  }

  /* ---------- nav items ---------- */
  &__icon {
    display: grid;
    place-items: center;
    flex-shrink: 0;
    color: inherit;
  }

  &__item {
    display: flex;
    align-items: center;
    gap: 0.6875rem;
    padding: 0.5rem 0.625rem;
    border-radius: 0.5rem;
    font-size: 0.84375rem;
    font-weight: 500;
    color: $text-secondary;
    text-decoration: none;
    border: none;
    background: transparent;
    width: 100%;
    cursor: pointer;
    transition: background 0.14s ease, color 0.14s ease;

    svg {
      opacity: 0.75;
      color: $text-tertiary;
      transition: color 0.14s ease, opacity 0.14s ease;
    }

    &:hover {
      background: $bg-subtle;
      color: $text-primary;

      svg {
        opacity: 1;
        color: $text-secondary;
      }
    }

    &--active {
      background: $primary-light;
      color: $primary;
      font-weight: 600;

      svg {
        opacity: 1;
        color: $primary;
      }

      &:hover {
        background: $primary-light;
        color: $primary;
      }
    }

    &--button {
      text-align: left;
    }

    &--collapsed {
      justify-content: center;
      padding: 0.5rem;
    }
  }

  /* ---------- footer ---------- */
  &__footer {
    border-top: 0.0625rem solid $border-subtle;
    margin-top: 0.875rem;
    padding-top: 0.625rem;
    display: flex;
    flex-direction: column;
    gap: 0.125rem;
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


















//MainLayout.scss
@use '../styles/variables' as *;

.main-layout {
  height: 100vh;
  display: flex;
  flex-direction: column;

  &__body {
    flex: 1;
    margin-top: $header-height;
    margin-bottom: $footer-height;
    overflow-y: auto;
    min-height: 0; // allow flex child to scroll
  }
}














//MainLayout.tsx
import { Outlet } from 'react-router-dom';
import type { FC } from 'react';
import Header from '../components/Header/Header';
import Footer from '../components/Footer/Footer';
import './MainLayout.scss';

const MainLayout: FC = () => {
  return (
    <div className="main-layout">
      <Header />
      <div className="main-layout__body">
        <Outlet />
      </div>
      <Footer />
    </div>
  );
};

export default MainLayout;















//WorkspaceLayout.scss
@use '../styles/variables' as *;

.workspace-layout {
  height: 100%;
  display: flex;
  overflow: hidden; // sidebar stays put; only content area scrolls

  &__content {
    flex: 1;
    height: 100%;
    overflow-y: auto;
    background: $bg-page;
    padding: 1.75rem 2rem 3rem;
  }
}














//WorkspaceLayout.tsx
import { Outlet } from 'react-router-dom';
import type { FC } from 'react';
import Sidebar from '../components/Sidebar/Sidebar';
import './WorkspaceLayout.scss';

const WorkspaceLayout: FC = () => {
  return (
    <div className="workspace-layout">
      <Sidebar />
      <main className="workspace-layout__content">
        <Outlet />
      </main>
    </div>
  );
};

export default WorkspaceLayout;












//DatasetStep.tsx
import { useMemo, useState, type FC } from 'react';
import { UploadCloud, Check } from 'lucide-react';
import { TEST_SUITES } from '../data';
import type { EvalTypeId } from '../types';

interface Props {
  evalType: EvalTypeId | null;
  selected: string | null;
  onSelect: (id: string) => void;
}

const CATEGORIES = ['All', 'Agents', 'Coding', 'General', 'RAG', 'Finance', 'Healthcare'];

const DatasetStep: FC<Props> = ({ evalType, selected, onSelect }) => {
  const [tab, setTab] = useState<'official' | 'private'>('official');
  const [category, setCategory] = useState('All');

  const filtered = useMemo(
    () => TEST_SUITES.filter((t) => category === 'All' || t.category === category),
    [category]
  );

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
              return (
                <button
                  key={d.id}
                  type="button"
                  className={`run-eval__dataset-card${isSelected ? ' run-eval__dataset-card--selected' : ''}`}
                  onClick={() => onSelect(d.id)}
                >
                  <div className="run-eval__dataset-top">
                    <span className="run-eval__dataset-name">{d.name}</span>
                    {isSelected && (
                      <span className="run-eval__type-check">
                        <Check size={12} strokeWidth={2.75} />
                      </span>
                    )}
                  </div>
                  <p className="run-eval__dataset-desc">{d.description}</p>
                  <div className="run-eval__dataset-meta n">
                    <span>{d.questions} questions</span>
                    <span>{d.difficulty}</span>
                    <span>{d.language}</span>
                  </div>
                  {recommended && <span className="run-eval__badge run-eval__badge--soft">Recommended</span>}
                </button>
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
    </div>
  );
};

export default DatasetStep;

















//MetricsStep.tsx
import { useMemo, type FC } from 'react';
import { Check } from 'lucide-react';
import { METRICS } from '../data';
import type { EvalTypeId } from '../types';

interface Props {
  evalType: EvalTypeId | null;
  selected: string[];
  onToggle: (id: string) => void;
}

const MetricsStep: FC<Props> = ({ evalType, selected, onToggle }) => {
  const typeMetrics = useMemo(() => {
    if (evalType === 'agent') return METRICS.agent;
    if (evalType === 'rag') return METRICS.rag;
    return METRICS.model;
  }, [evalType]);

  const groups = [
    { label: 'Universal', items: METRICS.universal },
    { label: 'Specific to this evaluation type', items: typeMetrics },
  ];

  return (
    <div className="run-eval__card run-eval__card--wide">
      <div className="run-eval__step-header-row">
        <div>
          <h2 className="run-eval__step-title">What to measure?</h2>
          <p className="run-eval__step-desc">
            Select the metrics that matter for your use case. Metrics are tailored to your evaluation type.
          </p>
        </div>
        <div className="run-eval__metrics-count">
          <span>{selected.length}</span> selected
        </div>
      </div>

      {groups.map((group) => (
        <div className="run-eval__metric-group" key={group.label}>
          <p className="run-eval__filter-title">{group.label}</p>
          <div className="run-eval__metrics-grid">
            {group.items.map((m) => {
              const isSelected = selected.includes(m.id);
              return (
                <button
                  key={m.id}
                  type="button"
                  className={`run-eval__metric-card${isSelected ? ' run-eval__metric-card--selected' : ''}`}
                  onClick={() => onToggle(m.id)}
                  title={m.tooltip}
                >
                  <span className="run-eval__metric-name">{m.name}</span>
                  <span className="run-eval__metric-tooltip">{m.tooltip}</span>
                  {isSelected && (
                    <span className="run-eval__type-check">
                      <Check size={12} strokeWidth={2.75} />
                    </span>
                  )}
                </button>
              );
            })}
          </div>
        </div>
      ))}
    </div>
  );
};

export default MetricsStep;















//ModelsStep.tsx
import { useMemo, useState, type FC } from 'react';
import { Search, SlidersHorizontal, Check, X } from 'lucide-react';
import { MODELS } from '../data';
import type { ModelInfo } from '../types';

interface Props {
  providers: string[];
  selected: string[];
  onToggle: (id: string) => void;
  onClear: () => void;
}

function priceTier(pricing: string): 'free' | 'low' | 'mid' | 'high' {
  const n = parseFloat(pricing.replace(/[^0-9.]/g, ''));
  if (n === 0) return 'free';
  if (n < 1) return 'low';
  if (n <= 5) return 'mid';
  return 'high';
}

const PRICE_LABELS: Record<string, string> = { free: 'FREE', low: '< $1', mid: '$1 – $5', high: '$5+' };

const ModelsStep: FC<Props> = ({ providers, selected, onToggle, onClear }) => {
  const [query, setQuery] = useState('');
  const [capFilters, setCapFilters] = useState<string[]>([]);
  const [priceFilters, setPriceFilters] = useState<string[]>([]);
  const [showFilters, setShowFilters] = useState(true);

  const pool = useMemo(
    () => (providers.length ? MODELS.filter((m) => providers.includes(m.providerId)) : MODELS),
    [providers]
  );

  const allCapabilities = useMemo(() => {
    const set = new Set<string>();
    pool.forEach((m) => m.capabilities.forEach((c) => set.add(c)));
    return Array.from(set).sort();
  }, [pool]);

  const filtered = useMemo(() => {
    return pool.filter((m: ModelInfo) => {
      if (query && !m.name.toLowerCase().includes(query.toLowerCase()) && !m.provider.toLowerCase().includes(query.toLowerCase())) {
        return false;
      }
      if (capFilters.length && !capFilters.every((c) => m.capabilities.includes(c))) return false;
      if (priceFilters.length && !priceFilters.includes(priceTier(m.pricing))) return false;
      return true;
    });
  }, [pool, query, capFilters, priceFilters]);

  const toggleCap = (cap: string) =>
    setCapFilters((prev) => (prev.includes(cap) ? prev.filter((c) => c !== cap) : [...prev, cap]));
  const togglePrice = (tier: string) =>
    setPriceFilters((prev) => (prev.includes(tier) ? prev.filter((c) => c !== tier) : [...prev, tier]));
  const resetFilters = () => {
    setCapFilters([]);
    setPriceFilters([]);
    setQuery('');
  };

  return (
    <div className="run-eval__card run-eval__card--wide">
      <h2 className="run-eval__step-title">Choose models</h2>
      <p className="run-eval__step-desc">Select the models you want to compare. Use filters to narrow the list.</p>

      <div className="run-eval__models-layout">
        {showFilters && (
          <aside className="run-eval__filters">
            <div className="run-eval__filters-head">
              <span>Filters</span>
              <button type="button" className="run-eval__link" onClick={resetFilters}>
                Reset all
              </button>
            </div>

            <div className="run-eval__filter-section">
              <p className="run-eval__filter-title">Capabilities</p>
              <div className="run-eval__filter-options">
                {allCapabilities.map((cap) => (
                  <label key={cap} className="run-eval__filter-chip">
                    <input type="checkbox" checked={capFilters.includes(cap)} onChange={() => toggleCap(cap)} />
                    {cap}
                  </label>
                ))}
              </div>
            </div>

            <div className="run-eval__filter-section">
              <p className="run-eval__filter-title">Pricing (per 1M tokens)</p>
              <div className="run-eval__filter-options">
                {Object.entries(PRICE_LABELS).map(([tier, label]) => (
                  <label key={tier} className="run-eval__filter-chip">
                    <input type="checkbox" checked={priceFilters.includes(tier)} onChange={() => togglePrice(tier)} />
                    {label}
                  </label>
                ))}
              </div>
            </div>
          </aside>
        )}

        <div className="run-eval__models-main">
          <div className="run-eval__search-bar">
            <Search size={15} />
            <input
              type="text"
              placeholder="Search models..."
              value={query}
              onChange={(e) => setQuery(e.target.value)}
            />
            <button type="button" className="run-eval__btn run-eval__btn--sm" onClick={() => setShowFilters((v) => !v)}>
              <SlidersHorizontal size={14} /> Filters
            </button>
          </div>

          {(capFilters.length > 0 || priceFilters.length > 0) && (
            <div className="run-eval__active-filters">
              {capFilters.map((c) => (
                <span key={c} className="run-eval__tag">
                  {c}
                  <button type="button" onClick={() => toggleCap(c)}>
                    <X size={11} />
                  </button>
                </span>
              ))}
              {priceFilters.map((p) => (
                <span key={p} className="run-eval__tag">
                  {PRICE_LABELS[p]}
                  <button type="button" onClick={() => togglePrice(p)}>
                    <X size={11} />
                  </button>
                </span>
              ))}
            </div>
          )}

          <div className="run-eval__models-grid">
            {filtered.map((m) => {
              const isSelected = selected.includes(m.id);
              return (
                <button
                  key={m.id}
                  type="button"
                  className={`run-eval__model-card${isSelected ? ' run-eval__model-card--selected' : ''}`}
                  onClick={() => onToggle(m.id)}
                >
                  <div className="run-eval__model-top">
                    <span className="run-eval__model-name">{m.name}</span>
                    {isSelected && (
                      <span className="run-eval__type-check">
                        <Check size={12} strokeWidth={2.75} />
                      </span>
                    )}
                  </div>
                  <span className="run-eval__model-provider">{m.provider}</span>
                  <div className="run-eval__model-caps">
                    {m.capabilities.slice(0, 3).map((c) => (
                      <span key={c} className="run-eval__chip run-eval__chip--static">
                        {c}
                      </span>
                    ))}
                  </div>
                  <div className="run-eval__model-meta n">
                    <span>{m.contextWindow}</span>
                    <span>{m.speedRating}</span>
                    <span>{m.pricing}</span>
                  </div>
                </button>
              );
            })}
            {filtered.length === 0 && <p className="run-eval__empty">No models match these filters.</p>}
          </div>

          {selected.length > 0 && (
            <div className="run-eval__selected-bar">
              <span>
                <strong>{selected.length}</strong> models selected
              </span>
              <button type="button" className="run-eval__btn run-eval__btn--sm" onClick={onClear}>
                Clear
              </button>
            </div>
          )}
        </div>
      </div>
    </div>
  );
};

export default ModelsStep;














//NameStep.tsx
import type { FC } from 'react';
import { SUGGESTED_NAMES } from '../data';

interface Props {
  name: string;
  onChange: (name: string) => void;
}

const NameStep: FC<Props> = ({ name, onChange }) => {
  return (
    <div className="run-eval__card">
      <h2 className="run-eval__step-title">Name your evaluation</h2>
      <p className="run-eval__step-desc">Give it a memorable name so you can find it later.</p>

      <div className="run-eval__field">
        <label className="run-eval__label" htmlFor="eval-name">
          Evaluation Name
        </label>
        <input
          id="eval-name"
          type="text"
          className="run-eval__input run-eval__input--lg"
          placeholder="e.g., Q3 Customer Support Bot Test"
          value={name}
          onChange={(e) => onChange(e.target.value)}
        />
      </div>

      <div className="run-eval__suggestions">
        <span className="run-eval__suggestions-label">Try:</span>
        {SUGGESTED_NAMES.map((s) => (
          <button key={s} type="button" className="run-eval__chip" onClick={() => onChange(s)}>
            {s}
          </button>
        ))}
      </div>
    </div>
  );
};

export default NameStep;























//ProvidersStep.tsx
import type { FC } from 'react';
import { CheckCircle2, PlusCircle, Info } from 'lucide-react';
import { PROVIDERS } from '../data';

interface Props {
  selected: string[];
  onToggle: (id: string) => void;
  onGoToProviders: () => void;
}

const ProvidersStep: FC<Props> = ({ selected, onToggle, onGoToProviders }) => {
  return (
    <div className="run-eval__card">
      <h2 className="run-eval__step-title">Select providers</h2>
      <p className="run-eval__step-desc">
        Choose which AI providers to include. Only connected providers are available.
      </p>

      <div className="run-eval__provider-grid">
        {PROVIDERS.map((p) => {
          const connected = p.status === 'connected';
          const isSelected = selected.includes(p.id);
          return (
            <button
              key={p.id}
              type="button"
              disabled={!connected}
              className={`run-eval__provider-card${isSelected ? ' run-eval__provider-card--selected' : ''}${
                !connected ? ' run-eval__provider-card--disabled' : ''
              }`}
              onClick={() => connected && onToggle(p.id)}
            >
              <span className="run-eval__provider-logo">{p.logo}</span>

              <span className="run-eval__provider-info">
                <span className="run-eval__provider-name-row">
                  <span className="run-eval__provider-name">{p.name}</span>
                  <span className={`run-eval__status-badge${connected ? ' run-eval__status-badge--on' : ''}`}>
                    {connected ? <CheckCircle2 size={11} /> : <PlusCircle size={11} />}
                    {connected ? 'Connected' : 'Not connected'}
                  </span>
                </span>
                <span className="run-eval__provider-desc">
                  {connected ? `${p.modelCount} models available` : `${p.modelCount}+ models`}
                </span>
              </span>

              {isSelected && (
                <span className="run-eval__type-check">
                  <CheckCircle2 size={13} strokeWidth={2.75} />
                </span>
              )}
            </button>
          );
        })}
      </div>

      <div className="run-eval__hint">
        <Info size={14} />
        <span>
          Need another provider?{' '}
          <button type="button" className="run-eval__link" onClick={onGoToProviders}>
            Add it in Settings
          </button>
        </span>
      </div>
    </div>
  );
};

export default ProvidersStep;


















//ReviewStep.tsx
import { useMemo, type FC } from 'react';
import { Info } from 'lucide-react';
import { EVAL_TYPES, MODELS, TEST_SUITES } from '../data';
import type { EvaluationDraft } from '../types';

interface Props {
  draft: EvaluationDraft;
}

const ReviewStep: FC<Props> = ({ draft }) => {
  const typeInfo = EVAL_TYPES.find((t) => t.id === draft.type);
  const modelNames = draft.models.map((id) => MODELS.find((m) => m.id === id)?.name).filter(Boolean);
  const dataset = TEST_SUITES.find((d) => d.id === draft.dataset);

  const { cost, minutes } = useMemo(() => {
    const questions = dataset?.questions ?? 0;
    const modelCount = draft.models.length || 1;
    const estCost = questions * modelCount * 0.0009;
    const estMinutes = Math.max(1, Math.round((questions * modelCount) / 180));
    return { cost: estCost, minutes: estMinutes };
  }, [dataset, draft.models.length]);

  return (
    <div className="run-eval__card">
      <h2 className="run-eval__step-title">Review &amp; Run</h2>
      <p className="run-eval__step-desc">Confirm your settings before starting.</p>

      <div className="run-eval__review">
        <div className="run-eval__review-row">
          <span>Name</span>
          <span>{draft.name || '—'}</span>
        </div>
        <div className="run-eval__review-row">
          <span>Type</span>
          <span>{typeInfo?.title ?? '—'}</span>
        </div>
        <div className="run-eval__review-row">
          <span>Models</span>
          <span>{modelNames.length ? modelNames.join(', ') : '—'}</span>
        </div>
        <div className="run-eval__review-row">
          <span>Test Suite</span>
          <span>{dataset?.name ?? '—'}</span>
        </div>
        <div className="run-eval__review-row">
          <span>Questions</span>
          <span>{dataset ? dataset.questions : '—'}</span>
        </div>
        <div className="run-eval__review-divider" />
        <div className="run-eval__review-row run-eval__review-row--highlight">
          <span>Est. Cost</span>
          <span>~${cost.toFixed(2)}</span>
        </div>
        <div className="run-eval__review-row run-eval__review-row--highlight">
          <span>Est. Time</span>
          <span>~{minutes} min</span>
        </div>
      </div>

      <div className="run-eval__hint">
        <Info size={14} />
        <span>Costs are estimates. Actual costs depend on provider pricing.</span>
      </div>
    </div>
  );
};

export default ReviewStep;
















//TypeStep.tsx
import type { FC } from 'react';
import { MessageSquare, Bot, Search, Check } from 'lucide-react';
import { EVAL_TYPES } from '../data';
import type { EvalTypeId } from '../types';

interface Props {
  value: EvalTypeId | null;
  onChange: (id: EvalTypeId) => void;
}

const ICONS: Record<EvalTypeId, FC<{ size?: number }>> = {
  model: MessageSquare,
  agent: Bot,
  rag: Search,
};

const TypeStep: FC<Props> = ({ value, onChange }) => {
  return (
    <div className="run-eval__card">
      <h2 className="run-eval__step-title">What are you testing?</h2>
      <p className="run-eval__step-desc">Different AI types need different evaluation methods.</p>

      <div className="run-eval__type-grid">
        {EVAL_TYPES.map((t) => {
          const Icon = ICONS[t.id];
          const selected = value === t.id;
          return (
            <button
              key={t.id}
              type="button"
              className={`run-eval__type-card${selected ? ' run-eval__type-card--selected' : ''}`}
              onClick={() => onChange(t.id)}
            >
              <span className="run-eval__type-icon">
                <Icon size={20} />
              </span>
              <span className="run-eval__type-content">
                <span className="run-eval__type-title">{t.title}</span>
                <span className="run-eval__type-desc">{t.desc}</span>
              </span>
              <span className="run-eval__badge">{t.badge}</span>
              {selected && (
                <span className="run-eval__type-check">
                  <Check size={13} strokeWidth={2.75} />
                </span>
              )}
            </button>
          );
        })}
      </div>
    </div>
  );
};

export default TypeStep;














//data.ts
import type { EvalType, Metric, ModelInfo, Provider, TestSuite } from './types';

export const EVAL_TYPES: EvalType[] = [
  {
    id: 'model',
    title: 'General Chat & Text (AI Model)',
    desc: 'Evaluate base model knowledge, summarization quality, and conversation tone across standardized test suites.',
    badge: 'Fast Evaluation',
  },
  {
    id: 'agent',
    title: 'Autonomous Workflow (Agent Evaluation)',
    desc: 'Test autonomous agents on multi-step tool execution, function calling, and programmatic workflow accuracy.',
    badge: 'Recommended for Automation',
  },
  {
    id: 'rag',
    title: 'Document Search & Answering (Knowledge / RAG)',
    desc: 'Measure how accurately AI models retrieve information from documents without generating incorrect answers.',
    badge: 'High Precision',
  },
];

export const PROVIDERS: Provider[] = [
  { id: 'openai', name: 'OpenAI', status: 'connected', modelCount: 8, logo: 'O', desc: 'Industry benchmark provider offering flagship chat and reasoning models.' },
  { id: 'anthropic', name: 'Anthropic', status: 'connected', modelCount: 5, logo: 'A', desc: 'Safety-first reasoning models optimized for long context and tool use.' },
  { id: 'together', name: 'Together AI', status: 'connected', modelCount: 14, logo: 'T', desc: 'High-performance infrastructure hosting open-weight agent models.' },
  { id: 'groq', name: 'Groq Cloud', status: 'connected', modelCount: 6, logo: 'G', desc: 'Ultra-low response time LPU inference engine for instant generation.' },
  { id: 'gemini', name: 'Google Gemini', status: 'connected', modelCount: 6, logo: 'G', desc: 'Massive context window models with native multi-modal reasoning.' },
  { id: 'openrouter', name: 'OpenRouter', status: 'not_connected', modelCount: 45, logo: 'R', desc: 'Unified routing engine with failover access to open and proprietary models.' },
  { id: 'azure', name: 'Azure OpenAI Service', status: 'not_connected', modelCount: 8, logo: 'Az', desc: 'Enterprise tenant hosting with zero data-retention SLAs.' },
  { id: 'ollama', name: 'Ollama Local (On-Prem)', status: 'not_connected', modelCount: 12, logo: 'OL', desc: 'Locally running quantized models inside your own network.' },
];

export const MODELS: ModelInfo[] = [
  { id: 'm-1', name: 'Model Alpha Agent', provider: 'Together AI', providerId: 'together', capabilities: ['Tool Calling', 'Autonomous Agent', 'JSON Mode'], contextWindow: '128k tokens', pricing: '$0.70 / 1M tokens', speedRating: 'Ultra Fast (85 t/s)', accuracyScore: 94.8, agentScore: 97.5, category: 'agent' },
  { id: 'm-2', name: 'Model Delta Agent v2', provider: 'Anthropic', providerId: 'anthropic', capabilities: ['Tool Calling', 'Deep Reasoning', 'Vision'], contextWindow: '200k tokens', pricing: '$3.00 / 1M tokens', speedRating: 'Fast (55 t/s)', accuracyScore: 95.4, agentScore: 96.2, category: 'agent' },
  { id: 'm-3', name: 'Model Gamma Agent', provider: 'OpenAI', providerId: 'openai', capabilities: ['Tool Calling', 'Multimodal Vision', 'Reasoning'], contextWindow: '128k tokens', pricing: '$2.50 / 1M tokens', speedRating: 'Fast (65 t/s)', accuracyScore: 94.2, agentScore: 95.0, category: 'agent' },
  { id: 'm-4', name: 'Model Epsilon Reasoning', provider: 'Together AI', providerId: 'together', capabilities: ['Deep Math', 'Advanced Logic', 'Code Generation'], contextWindow: '64k tokens', pricing: '$0.55 / 1M tokens', speedRating: 'Medium (42 t/s)', accuracyScore: 96.1, agentScore: 91.8, category: 'model' },
  { id: 'm-5', name: 'Model Zeta Instruct', provider: 'Groq Cloud', providerId: 'groq', capabilities: ['Instant Response', 'General Chat', 'Tool Calling'], contextWindow: '128k tokens', pricing: '$0.59 / 1M tokens', speedRating: 'Instant (280 t/s)', accuracyScore: 91.5, agentScore: 89.4, category: 'model' },
  { id: 'm-6', name: 'Model Theta Long-Context', provider: 'Google Gemini', providerId: 'gemini', capabilities: ['2M Tokens', 'Multi-modal Video', 'Document Analysis'], contextWindow: '2,000,000 tokens', pricing: '$2.50 / 1M tokens', speedRating: 'Fast (50 t/s)', accuracyScore: 93.7, agentScore: 91.0, category: 'rag' },
  { id: 'm-7', name: 'Model Theta Flash', provider: 'Google Gemini', providerId: 'gemini', capabilities: ['Vision', 'Tool Calling', 'Streaming'], contextWindow: '1,000,000 tokens', pricing: '$0.10 / 1M tokens', speedRating: 'Ultra Fast (120 t/s)', accuracyScore: 92.4, agentScore: 94.3, category: 'agent' },
  { id: 'm-8', name: 'Model Delta Opus', provider: 'Anthropic', providerId: 'anthropic', capabilities: ['Deep Reasoning', 'Long Context', 'Vision'], contextWindow: '200k tokens', pricing: '$15.00 / 1M tokens', speedRating: 'Medium (25 t/s)', accuracyScore: 97.2, agentScore: 94.8, category: 'model' },
  { id: 'm-9', name: 'Model Eta Instruct', provider: 'Together AI', providerId: 'together', capabilities: ['Coding', 'Math', 'Multilingual'], contextWindow: '128k tokens', pricing: '$0.90 / 1M tokens', speedRating: 'Fast (65 t/s)', accuracyScore: 93.8, agentScore: 92.1, category: 'model' },
  { id: 'm-10', name: 'Model Gamma Mini', provider: 'OpenAI', providerId: 'openai', capabilities: ['Vision', 'Tool Calling', 'Streaming'], contextWindow: '128k tokens', pricing: '$0.15 / 1M tokens', speedRating: 'Ultra Fast (90 t/s)', accuracyScore: 88.5, agentScore: 87.2, category: 'model' },
];

export const TEST_SUITES: TestSuite[] = [
  { id: 'ts-agent', name: 'Autonomous Tool & Workflow Suite', category: 'Agents', questions: 420, language: 'English / JSON', task: 'Multi-step API Execution & Self-Correction', difficulty: 'Expert', version: 'v3.2', maintainer: 'SemcoEval Labs', description: 'Evaluates multi-step tool calling, nested JSON schema generation, error recovery, and workflow execution.', recommendedFor: ['agent'], featured: true },
  { id: 'ts-swe', name: 'SWE-bench Verified Software Engineer Suite', category: 'Coding', questions: 500, language: 'Python / JS / TS', task: 'Autonomous GitHub Issue Resolution', difficulty: 'Expert', version: '2026.2', maintainer: 'Open Source AI Labs', description: 'Measures ability to independently diagnose bug reports, run local unit tests, and commit working patches.', recommendedFor: ['agent', 'model'], featured: true },
  { id: 'ts-mmlu', name: 'MMLU-Pro General Knowledge & Reasoning Suite', category: 'General', questions: 1400, language: 'Multilingual (24 Languages)', task: 'Academic & Professional Problem Solving', difficulty: 'High', version: 'Pro-v2', maintainer: 'Stanford CRFM', description: 'Comprehensive multi-domain benchmark covering 57 disciplines including law, physics, and ethics.', recommendedFor: ['model'], featured: false },
  { id: 'ts-ragas', name: 'Ragas Document Factual Recall & Faithfulness', category: 'RAG', questions: 350, language: 'English', task: 'Context Precision & Hallucination Defense', difficulty: 'Medium', version: '1.4', maintainer: 'Ragas Ecosystem', description: 'Evaluates retrieval accuracy and ensures answers derive strictly from verified documents.', recommendedFor: ['rag'], featured: true },
  { id: 'ts-finance', name: 'Corporate Finance & Audit Math Suite', category: 'Finance', questions: 280, language: 'English / Numeric', task: 'Exact Mathematical Deduction & Regulation', difficulty: 'Advanced', version: '2026-Q1', maintainer: 'SemcoEval Labs', description: 'Validates exact calculation precision, formula interpretation, and compliance audit reasoning.', recommendedFor: ['model', 'agent'], featured: false },
  { id: 'ts-health', name: 'Clinical Diagnostic Safety & Care Benchmark', category: 'Healthcare', questions: 310, language: 'English / Medical', task: 'Diagnostic Logic & Patient Guardrails', difficulty: 'Expert', version: 'Med-2.1', maintainer: 'HealthAI Foundation', description: 'Assesses diagnostic recommendation accuracy and safety compliance in health triage.', recommendedFor: ['model'], featured: false },
];

export const METRICS: { universal: Metric[]; model: Metric[]; agent: Metric[]; rag: Metric[] } = {
  universal: [
    { id: 'accuracy', name: 'Accuracy', tooltip: 'Percentage of correct answers or successful task completions.', defaultChecked: true },
    { id: 'latency', name: 'Response Time', tooltip: 'Average time to generate a complete response (seconds).', defaultChecked: true },
    { id: 'cost', name: 'Cost Efficiency', tooltip: 'Cost per 1,000 API calls at provider pricing.', defaultChecked: true },
    { id: 'safety', name: 'Safety Score', tooltip: 'Resistance to jailbreaks, prompt injection, and harmful outputs.', defaultChecked: false },
  ],
  model: [
    { id: 'fluency', name: 'Fluency & Coherence', tooltip: 'How natural and well-structured the generated text is.', defaultChecked: true },
    { id: 'instruction_following', name: 'Instruction Following', tooltip: 'How well the model adheres to specific instructions and constraints.', defaultChecked: true },
    { id: 'reasoning', name: 'Reasoning Quality', tooltip: 'Logical consistency, chain-of-thought clarity, and problem decomposition.', defaultChecked: true },
    { id: 'factuality', name: 'Factual Accuracy', tooltip: 'Correctness of factual claims based on world knowledge.', defaultChecked: false },
    { id: 'helpfulness', name: 'Helpfulness', tooltip: "How useful and relevant the response is to the user's query.", defaultChecked: false },
  ],
  agent: [
    { id: 'tool_success', name: 'Tool Calling Success', tooltip: 'Percentage of function/API calls with correct syntax and parameters.', defaultChecked: true },
    { id: 'task_completion', name: 'Task Completion Rate', tooltip: 'Percentage of multi-step tasks completed successfully end-to-end.', defaultChecked: true },
    { id: 'action_accuracy', name: 'Action Sequencing', tooltip: 'Correct ordering and logic of multi-step action plans.', defaultChecked: true },
    { id: 'error_recovery', name: 'Error Recovery', tooltip: 'Ability to detect failures and self-correct without human intervention.', defaultChecked: true },
    { id: 'planning', name: 'Planning Quality', tooltip: 'Quality of task decomposition and strategic planning.', defaultChecked: false },
  ],
  rag: [
    { id: 'faithfulness', name: 'Faithfulness', tooltip: 'Does the answer accurately reflect the retrieved context without adding unsupported claims?', defaultChecked: true },
    { id: 'answer_relevance', name: 'Answer Relevance', tooltip: 'How well the generated answer addresses the original question.', defaultChecked: true },
    { id: 'context_precision', name: 'Context Precision', tooltip: 'Are the retrieved documents relevant to the question?', defaultChecked: true },
    { id: 'groundedness', name: 'Groundedness', tooltip: 'Is every claim in the answer supported by the source documents?', defaultChecked: true },
    { id: 'hallucination', name: 'Hallucination Rate', tooltip: 'Percentage of fabricated facts not present in retrieved context.', defaultChecked: true },
  ],
};

export const SUGGESTED_NAMES = ['Agent Tool Calling Test', 'Support Bot Comparison', 'Code Generation Test'];

















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
    align-items: flex-start;
    justify-content: space-between;
    gap: 1rem;
    margin-bottom: 1.5rem;
  }

  &__title {
    font-size: 30px;
    font-weight: 800;
    letter-spacing: -0.03em;
    color: $text-primary;
  }

  &__subtitle {
    margin-top: 0.375rem;
    color: $text-secondary;
    font-size: 0.90625rem;
  }

  /* ---------- buttons ---------- */
  &__btn {
    display: inline-flex;
    align-items: center;
    gap: 0.5rem;
    font-size: 0.84375rem;
    font-weight: 600;
    padding: 0.5625rem 0.9375rem;
    border-radius: 0.5rem;
    border: 0.0625rem solid transparent;
    cursor: pointer;
    transition: background 0.14s ease, border-color 0.14s ease, color 0.14s ease;
    font-family: $font-body;

    &--primary {
      background: $primary;
      color: #fff;
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
      font-size: 0.84375rem;
    }

    &--sm {
      padding: 0.375rem 0.6875rem;
      font-size: 0.78125rem;
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
    top: 15px;
    left: 15px;
    right: 15px;
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
    width: 30px;
    height: 30px;
    border-radius: 50%;
    display: grid;
    place-items: center;
    font-size: 0.75rem;
    font-weight: 700;
    background: $bg-main;
    border: 2px solid $border-default;
    color: $text-tertiary;
    transition: background 0.16s ease, border-color 0.16s ease, color 0.16s ease, transform 0.16s ease;
  }

  &__node-label {
    font-size: 0.6875rem;
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
      color: #fff;
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
      color: #fff;
    }
  }

  /* ---------- step kicker + heading ---------- */
  &__step-kicker {
    display: flex;
    align-items: center;
    gap: 8px;
    font-family: $font-mono;
    font-size: 0.6875rem;
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
    font-size: 24px;
    font-weight: 800;
    letter-spacing: -0.02em;
    line-height: 1.2;
    color: $text-primary;
  }

  &__step-desc {
    margin-top: 6px;
    font-size: 0.875rem;
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
    font-size: 0.78125rem;
    font-weight: 600;
    color: $text-secondary;
    margin-bottom: 0.4375rem;
  }

  &__input {
    width: 100%;
    border: 0.0625rem solid $border-default;
    border-radius: 0.5rem;
    padding: 0.625rem 0.75rem;
    font-size: 0.875rem;
    font-family: $font-body;
    color: $text-primary;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &::placeholder {
      color: #a8b1bb;
    }

    &:focus {
      outline: none;
      border-color: $primary;
      box-shadow: 0 0 0 0.1875rem $primary-light;
    }

    &--lg {
      padding: 0.75rem 0.875rem;
      font-size: 0.9375rem;
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
    font-size: 0.78125rem;
    color: $text-tertiary;
    margin-right: 0.125rem;
  }

  &__chip {
    font-size: 0.75rem;
    font-weight: 500;
    color: $text-secondary;
    background: $bg-subtle;
    border: 0.0625rem solid $border-default;
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
      color: #fff;
    }

    &--static {
      cursor: default;
      font-size: 0.6875rem;
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
    border: 0.0625rem solid $border-default;
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
    font-size: 0.9375rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__type-desc {
    font-size: 0.8125rem;
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
    color: #fff;
    display: grid;
    place-items: center;
  }

  &__badge {
    align-self: flex-start;
    flex-shrink: 0;
    font-size: 0.65625rem;
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
    border: 0.0625rem solid $border-default;
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
    color: #fff;
    font-weight: 700;
    font-size: 0.8125rem;
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
    font-size: 0.875rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__provider-desc {
    font-size: 0.75rem;
    color: $text-tertiary;
  }

  &__status-badge {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 0.3125rem;
    font-size: 0.65625rem;
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
    font-size: 0.8125rem;
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
    border: 0.0625rem solid $border-subtle;
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
    font-size: 0.8125rem;
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
    font-size: 0.65625rem;
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
    font-size: 0.8125rem;
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
    border: 0.0625rem solid $border-default;
    border-radius: 0.5rem;
    padding: 0.5625rem 0.75rem;
    color: $text-tertiary;

    input {
      flex: 1;
      border: none;
      outline: none;
      font-size: 0.84375rem;
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
    font-size: 0.71875rem;
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
    border: 0.0625rem solid $border-default;
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
    font-size: 0.875rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__model-provider {
    font-size: 0.78125rem;
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
    font-size: 0.71875rem;
    color: $text-tertiary;
    margin-top: 0.125rem;
  }

  &__empty {
    grid-column: 1 / -1;
    padding: 2rem;
    text-align: center;
    color: $text-tertiary;
    font-size: 0.84375rem;
  }

  &__selected-bar {
    margin-top: 1rem;
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0.75rem 1rem;
    background: $primary-light;
    border-radius: 0.625rem;
    font-size: 0.84375rem;
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
    border-bottom: 0.0625rem solid $border-subtle;
  }

  &__tab {
    padding: 0.5625rem 0.25rem;
    margin-right: 1.25rem;
    border: none;
    background: transparent;
    font-size: 0.84375rem;
    font-weight: 600;
    color: $text-tertiary;
    cursor: pointer;
    border-bottom: 0.125rem solid transparent;
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
    border: 0.0625rem solid $border-default;
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
    font-size: 0.875rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__dataset-desc {
    font-size: 0.8125rem;
    color: $text-secondary;
    line-height: 1.5;
  }

  &__dataset-meta {
    display: flex;
    flex-wrap: wrap;
    gap: 0.625rem;
    font-size: 0.71875rem;
    color: $text-tertiary;
  }

  &__empty-state,
  &__upload-zone {
    margin-top: 1.5rem;
    border: 0.0938rem dashed $border-strong;
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
      font-size: 0.9375rem;
      color: $text-primary;
    }

    p {
      font-size: 0.8125rem;
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
    font-size: 0.8125rem;
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
    border: 0.0625rem solid $border-default;
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
    font-size: 0.8125rem;
    font-weight: 600;
    color: $text-primary;
  }

  &__metric-tooltip {
    display: block;
    margin-top: 0.25rem;
    font-size: 0.71875rem;
    color: $text-tertiary;
    line-height: 1.4;
  }

  /* ---------- review step ---------- */
  &__review {
    margin-top: 1.5rem;
    border: 0.0625rem solid $border-subtle;
    border-radius: 0.75rem;
    overflow: hidden;
  }

  &__review-row {
    display: flex;
    align-items: baseline;
    justify-content: space-between;
    gap: 1rem;
    padding: 0.75rem 1rem;
    font-size: 0.84375rem;
    border-bottom: 0.0625rem solid $border-subtle;

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
    font-size: 0.8125rem;
    color: $danger;
    background: $danger-subtle;
    border-radius: 0.5rem;
    padding: 0.625rem 0.875rem;
  }

  &__nav {
    flex-shrink: 0;
    margin-top: 1.25rem;
    padding-top: 1.25rem;
    border-top: 0.0625rem solid $border-subtle;
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















//RunEvaluation.tsx
import { useState, type FC } from 'react';
import { useNavigate } from 'react-router-dom';
import { ArrowLeft, ArrowRight, Play, Check } from 'lucide-react';
import NameStep from './steps/NameStep';
import TypeStep from './steps/TypeStep';
import ProvidersStep from './steps/ProvidersStep';
import ModelsStep from './steps/ModelsStep';
import DatasetStep from './steps/DatasetStep';
import MetricsStep from './steps/MetricsStep';
import ReviewStep from './steps/ReviewStep';
import { METRICS } from './data';
import { WIZARD_STEPS, type EvaluationDraft } from './types';
import './RunEvaluation.scss';

const EMPTY_DRAFT: EvaluationDraft = {
  name: '',
  type: null,
  providers: [],
  models: [],
  dataset: null,
  metrics: [],
};

const RunEvaluation: FC = () => {
  const navigate = useNavigate();
  const [step, setStep] = useState(1);
  const [draft, setDraft] = useState<EvaluationDraft>(EMPTY_DRAFT);
  const [error, setError] = useState<string | null>(null);
  const totalSteps = WIZARD_STEPS.length;

  const toggleInArray = (key: 'providers' | 'models' | 'metrics', id: string) => {
    setDraft((d) => {
      const arr = d[key];
      const next = arr.includes(id) ? arr.filter((x) => x !== id) : [...arr, id];
      return { ...d, [key]: next };
    });
  };

  const setType = (id: EvaluationDraft['type']) => {
    setDraft((d) => {
      if (d.type === id) return d;
      const defaults = [
        ...METRICS.universal.filter((m) => m.defaultChecked).map((m) => m.id),
        ...(id === 'agent' ? METRICS.agent : id === 'rag' ? METRICS.rag : METRICS.model)
          .filter((m) => m.defaultChecked)
          .map((m) => m.id),
      ];
      return { ...d, type: id, metrics: defaults };
    });
  };

  const validate = (): boolean => {
    setError(null);
    if (step === 1 && !draft.name.trim()) {
      setError('Enter an evaluation name to continue.');
      return false;
    }
    if (step === 2 && !draft.type) {
      setError('Select an evaluation type to continue.');
      return false;
    }
    if (step === 3 && draft.providers.length === 0) {
      setError('Select at least one provider to continue.');
      return false;
    }
    if (step === 4 && draft.models.length === 0) {
      setError('Select at least one model to continue.');
      return false;
    }
    if (step === 5 && !draft.dataset) {
      setError('Select a test suite to continue.');
      return false;
    }
    if (step === 6 && draft.metrics.length === 0) {
      setError('Select at least one metric to continue.');
      return false;
    }
    return true;
  };

  const goNext = () => {
    if (!validate()) return;
    setStep((s) => Math.min(totalSteps, s + 1));
  };
  const goBack = () => setStep((s) => Math.max(1, s - 1));
  const goToStep = (target: number) => {
    if (target < step) setStep(target);
  };

  const startEvaluation = () => {
    if (!validate()) return;
    navigate('/app/history');
  };

  return (
    <div className="run-eval">
      <div className="run-eval__header">
        <div>
          <h1 className="run-eval__title">New Evaluation</h1>
          <p className="run-eval__subtitle">Compare AI models with standardized tests</p>
        </div>
      </div>

      <div className="run-eval__wizard">
        <div className="run-eval__tracker">
          <div className="run-eval__tracker-bar">
            <div
              className="run-eval__tracker-fill"
              style={{ width: `${((step - 1) / (totalSteps - 1)) * 100}%` }}
            />
          </div>
          <div className="run-eval__tracker-nodes">
            {WIZARD_STEPS.map((label, i) => {
              const num = i + 1;
              const state = num === step ? 'active' : num < step ? 'complete' : 'upcoming';
              return (
                <button
                  key={label}
                  type="button"
                  className={`run-eval__node run-eval__node--${state}`}
                  onClick={() => goToStep(num)}
                  disabled={num > step}
                >
                  <span className="run-eval__node-dot">
                    {state === 'complete' ? <Check size={12} strokeWidth={3} /> : num}
                  </span>
                  <span className="run-eval__node-label">{label}</span>
                </button>
              );
            })}
          </div>
        </div>

        <p className="run-eval__step-kicker">
          Step {step} of {totalSteps}
        </p>

        <div className="run-eval__body">
          {step === 1 && <NameStep name={draft.name} onChange={(name) => setDraft((d) => ({ ...d, name }))} />}
          {step === 2 && <TypeStep value={draft.type} onChange={setType} />}
          {step === 3 && (
            <ProvidersStep
              selected={draft.providers}
              onToggle={(id) => toggleInArray('providers', id)}
              onGoToProviders={() => navigate('/app/providers')}
            />
          )}
          {step === 4 && (
            <ModelsStep
              providers={draft.providers}
              selected={draft.models}
              onToggle={(id) => toggleInArray('models', id)}
              onClear={() => setDraft((d) => ({ ...d, models: [] }))}
            />
          )}
          {step === 5 && (
            <DatasetStep
              evalType={draft.type}
              selected={draft.dataset}
              onSelect={(id) => setDraft((d) => ({ ...d, dataset: id }))}
            />
          )}
          {step === 6 && (
            <MetricsStep evalType={draft.type} selected={draft.metrics} onToggle={(id) => toggleInArray('metrics', id)} />
          )}
          {step === 7 && <ReviewStep draft={draft} />}

          {error && <p className="run-eval__error">{error}</p>}
        </div>

        <div className="run-eval__nav">
          {step > 1 ? (
            <button type="button" className="run-eval__btn run-eval__btn--secondary run-eval__btn--lg" onClick={goBack}>
              <ArrowLeft size={16} /> Back
            </button>
          ) : (
            <span />
          )}

          {step < totalSteps ? (
            <button type="button" className="run-eval__btn run-eval__btn--primary run-eval__btn--lg" onClick={goNext}>
              Continue <ArrowRight size={16} />
            </button>
          ) : (
            <button type="button" className="run-eval__btn run-eval__btn--primary run-eval__btn--lg" onClick={startEvaluation}>
              <Play size={16} /> Start Evaluation
            </button>
          )}
        </div>
      </div>
    </div>
  );
};

export default RunEvaluation;














//types.ts
export type EvalTypeId = 'model' | 'agent' | 'rag';

export interface EvalType {
  id: EvalTypeId;
  title: string;
  desc: string;
  badge: string;
}

export interface Provider {
  id: string;
  name: string;
  status: 'connected' | 'not_connected';
  modelCount: number;
  logo: string;
  desc: string;
}

export interface ModelInfo {
  id: string;
  name: string;
  provider: string;
  providerId: string;
  capabilities: string[];
  contextWindow: string;
  pricing: string;
  speedRating: string;
  accuracyScore: number;
  agentScore: number;
  category: EvalTypeId;
}

export interface TestSuite {
  id: string;
  name: string;
  category: string;
  questions: number;
  language: string;
  task: string;
  difficulty: string;
  version: string;
  maintainer: string;
  description: string;
  recommendedFor: EvalTypeId[];
  featured: boolean;
}

export interface Metric {
  id: string;
  name: string;
  tooltip: string;
  defaultChecked: boolean;
}

export interface EvaluationDraft {
  name: string;
  type: EvalTypeId | null;
  providers: string[];
  models: string[];
  dataset: string | null;
  metrics: string[];
}

export const WIZARD_STEPS = ['Name', 'Type', 'Providers', 'Models', 'Test Suite', 'Metrics', 'Review'] as const;
