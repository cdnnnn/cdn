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
  --white: #fff;
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
    font-size: 0.75rem;
    font-weight: 600;
    color: var(--blue);
    background: var(--blue-wash);
    border: 0.0625rem solid $primary-subtle;
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
    font-size: 0.656rem;
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
    font-size: 0.875rem;
    font-weight: 600;
    padding: 0.625rem 1.125rem;
    border-radius: 0.5rem;
    border: 0.0625rem solid transparent;
    cursor: pointer;
    text-decoration: none;
    display: inline-flex;
    align-items: center;
    gap: 0.5rem;
    transition: background 0.15s ease, border-color 0.15s ease, color 0.15s ease;

    &--lg {
      padding: 0.875rem 1.5rem;
      font-size: 0.9375rem;
    }

    &--primary {
      background: var(--blue);
      color: var(--white);

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
    border: 0.0625rem solid var(--rule);
    border-radius: 0.875rem;
    box-shadow: 0 0.0625rem 0.125rem rgba(14, 21, 38, 0.04), 0 1.25rem 2.875rem -1.625rem rgba(14, 21, 38, 0.22);
    overflow: hidden;
  }

  &__panel-head {
    display: flex;
    align-items: center;
    flex-wrap: wrap;
    gap: 0.5rem 0.875rem;
    padding: 0.875rem 1.25rem;
    border-bottom: 0.0625rem solid var(--rule-2);
    background: var(--paper);
  }

  &__panel-title {
    font-size: 0.875rem;
    font-weight: 600;
    letter-spacing: -0.01em;
  }

  &__panel-meta {
    margin-left: auto;
    font-size: 0.78125rem;
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
    border-bottom: 0.0625rem dashed var(--rule);

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
    font-size: 0.84375rem;
    font-weight: 600;
  }

  &__rail-unit {
    font-size: 0.75rem;
    color: var(--ink-3);
  }

  &__rail-dir {
    margin-left: auto;
    font-size: 0.75rem;
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
    font-size: 0.65625rem;
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
    border: 0.1875rem solid var(--ink-3);
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
    font-size: 0.71875rem;
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
    border-top: 0.0625rem solid var(--rule-2);
    background: var(--paper);
    font-size: 0.8125rem;
    color: var(--ink-2);

    b {
      color: var(--ink);
      font-weight: 600;
    }
  }

  &__k {
    font-size: 0.625rem;
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
      font-size: 1.0625rem;
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
    font-size: 1.3125rem;
    font-weight: 700;
    letter-spacing: -0.03em;
  }

  &__stat-l {
    font-size: 0.8125rem;
    color: var(--ink-2);
  }

  &__strip-note {
    margin-left: auto;
    font-size: 0.78125rem;
    color: var(--ink-3);
  }

  /* ================= GRADED TRANSCRIPT ================= */
  &__ask {
    background: var(--white);
    border: 0.0625rem solid var(--rule);
    border-left: 0.1875rem solid var(--blue);
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
    font-size: 1.09375rem;
    line-height: 1.45;
  }

  &__ask-src {
    margin-top: 1rem;
    padding: 0.6875rem 0.8125rem;
    background: var(--paper);
    border-radius: 0.4375rem;
    font-size: 0.8125rem;
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
    border: 0.0625rem solid var(--rule);
    border-left: 0.1875rem solid var(--rule);
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
      font-size: 0.90625rem;
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
    font-size: 0.84375rem;
    font-weight: 600;
  }

  &__ans-mark {
    margin-left: auto;
    font-size: 0.625rem;
    font-weight: 700;
    letter-spacing: 0.11em;
    text-transform: uppercase;
    padding: 0.1875rem 0.5rem;
    border-radius: 0.3125rem;
  }

  &__ans-foot {
    margin-top: auto;
    padding-top: 0.8125rem;
    font-size: 0.78125rem;
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
    font-size: 0.78125rem;
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
    border: 0.0625rem solid var(--rule);
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
      font-size: 1.15625rem;
      margin: 0.75rem 0 0.5625rem;
    }

    p {
      font-size: 0.90625rem;
      color: var(--ink-2);
    }
  }

  &__step-k {
    font-size: 0.6875rem;
    font-weight: 700;
    letter-spacing: 0.12em;
    color: var(--blue);
  }

  &__step-hint {
    margin-top: 0.875rem;
    padding-top: 0.75rem;
    border-top: 0.0625rem dashed var(--rule);
    font-size: 0.75rem;
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
    border: 0.0625rem solid var(--rule);
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
      font-size: 1.1875rem;
      margin: 1rem 0 0.625rem;
    }

    p {
      font-size: 0.90625rem;
      color: var(--ink-2);
    }
  }

  &__mode-tag {
    align-self: flex-start;
    font-size: 0.625rem;
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
    border-top: 0.0625rem solid var(--rule-2);
    display: flex;
    flex-wrap: wrap;
    gap: 0.375rem;
  }

  &__chip {
    font-size: 0.71875rem;
    color: var(--ink-2);
    border: 0.0625rem solid var(--rule);
    border-radius: 0.3125rem;
    padding: 0.1875rem 0.5rem;
  }

  /* ================= LEDGER ================= */
  &__ledger {
    border-top: 0.0625rem solid var(--rule);
  }

  &__row {
    display: grid;
    grid-template-columns: 2.5rem 1.05fr 1.45fr;
    gap: 1.5rem;
    padding: 1.625rem 0;
    border-bottom: 0.0625rem solid var(--rule);
    align-items: start;

    h3 {
      font-size: 1.21875rem;
    }

    p {
      font-size: 0.9375rem;
      color: var(--ink-2);
    }
  }

  &__row-k {
    font-size: 0.75rem;
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
      font-size: 1.0625rem;
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
      font-size: 0.65625rem;
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

















//Landing.tsx
import { Link } from 'react-router-dom';
import type { CSSProperties, FC } from 'react';
import './Landing.scss';

// Pip position/animation are driven by CSS custom properties, which the
// React CSSProperties type doesn't know about — cast per usage below.
type PipStyle = CSSProperties & { '--x': string; '--d': string };

const Landing: FC = () => {
  return (
    <div className="landing">
      {/* ==================== HERO ==================== */}
      <header className="landing__hero" id="top">
        <div className="landing__shell landing__hero-in">
          <div className="landing__hero-copy">
            <span className="landing__hero-badge">
              <span className="landing__hero-badge-dot" />
              Model Evaluation Platform · v1.0.0
            </span>
            <p className="landing__eyebrow">Model evaluation for enterprise AI</p>
            <h1>
              Stop choosing models <em>on vibes.</em>
            </h1>
            <p className="landing__hero-sub">
              SemcoEval runs the same test suite across every model you're considering and puts
              accuracy, latency and cost on one page — so the model you ship is the one the
              evidence picked.
            </p>
            <div className="landing__hero-cta">
              <Link className="landing__btn landing__btn--primary landing__btn--lg" to="/app">
                Run an evaluation
              </Link>
              <a className="landing__btn landing__btn--ghost landing__btn--lg" href="#run">
                See how it works
              </a>
            </div>
          </div>

          {/* SIGNATURE: the score rails */}
          <section className="landing__panel" aria-label="Sample evaluation result">
            <div className="landing__panel-head">
              <span className="landing__panel-title">Run 4127 — Enterprise QA suite</span>
              <span className="landing__panel-meta">240 prompts · 4 models · complete</span>
            </div>

            <div className="landing__rails">
              <div className="landing__rail">
                <div className="landing__rail-top">
                  <span className="landing__rail-label">Accuracy</span>
                  <span className="landing__rail-unit">% graded correct</span>
                  <span className="landing__rail-dir">higher is better →</span>
                </div>
                <div className="landing__axis">
                  <span className="landing__pip" style={{ '--x': '48.8%', '--d': '.05s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      Model Alpha <b className="n">79.5</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                  <span className="landing__pip landing__pip--low" style={{ '--x': '60.3%', '--d': '.12s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      Model Beta <b className="n">84.1</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                  <span className="landing__pip landing__pip--brand" style={{ '--x': '71.8%', '--d': '.19s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      Model Gamma <b className="n">88.7</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                  <span
                    className="landing__pip landing__pip--best landing__pip--low"
                    style={{ '--x': '78.0%', '--d': '.26s' } as PipStyle}
                  >
                    <span className="landing__pip-flag">
                      Model Delta <b className="n">91.2</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                </div>
                <div className="landing__axis-scale n">
                  <span>60</span>
                  <span>70</span>
                  <span>80</span>
                  <span>90</span>
                  <span>100</span>
                </div>
              </div>

              <div className="landing__rail">
                <div className="landing__rail-top">
                  <span className="landing__rail-label">Response time</span>
                  <span className="landing__rail-unit">p95, seconds</span>
                  <span className="landing__rail-dir">← lower is better</span>
                </div>
                <div className="landing__axis">
                  <span className="landing__pip landing__pip--best" style={{ '--x': '12.5%', '--d': '.10s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      Model Alpha <b className="n">0.4</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                  <span className="landing__pip landing__pip--brand" style={{ '--x': '37.5%', '--d': '.17s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      Model Gamma <b className="n">1.2</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                  <span className="landing__pip" style={{ '--x': '65.6%', '--d': '.24s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      Model Delta <b className="n">2.1</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                  <span className="landing__pip landing__pip--low" style={{ '--x': '90.6%', '--d': '.31s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      Model Beta <b className="n">2.9</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                </div>
                <div className="landing__axis-scale n">
                  <span>0.0</span>
                  <span>0.8</span>
                  <span>1.6</span>
                  <span>2.4</span>
                  <span>3.2</span>
                </div>
              </div>

              <div className="landing__rail">
                <div className="landing__rail-top">
                  <span className="landing__rail-label">Cost</span>
                  <span className="landing__rail-unit">USD per 1,000 runs</span>
                  <span className="landing__rail-dir">← lower is better</span>
                </div>
                <div className="landing__axis">
                  <span className="landing__pip landing__pip--best" style={{ '--x': '4.8%', '--d': '.15s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      Model Alpha <b className="n">$2.40</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                  <span className="landing__pip landing__pip--low" style={{ '--x': '13.6%', '--d': '.22s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      Model Beta <b className="n">$6.80</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                  <span className="landing__pip landing__pip--brand" style={{ '--x': '49.0%', '--d': '.29s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      Model Gamma <b className="n">$24.50</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                  <span className="landing__pip" style={{ '--x': '82.4%', '--d': '.36s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      Model Delta <b className="n">$41.20</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                </div>
                <div className="landing__axis-scale n">
                  <span>$0</span>
                  <span>$12.50</span>
                  <span>$25</span>
                  <span>$37.50</span>
                  <span>$50</span>
                </div>
              </div>
            </div>

            <div className="landing__panel-foot">
              <span>
                <span className="landing__k">Most accurate</span>
                <b>Model Delta</b>
              </span>
              <span>
                <span className="landing__k">Fastest</span>
                <b>Model Alpha</b>
              </span>
              <span>
                <span className="landing__k">Cheapest</span>
                <b>Model Alpha</b>
              </span>
            </div>
          </section>
        </div>
      </header>

      {/* ==================== PROOF STRIP ==================== */}
      <section className="landing__strip">
        <div className="landing__shell landing__strip-in">
          <span className="landing__stat">
            <span className="landing__stat-n n">9</span>
            <span className="landing__stat-l">providers</span>
          </span>
          <span className="landing__stat">
            <span className="landing__stat-n n">100+</span>
            <span className="landing__stat-l">models</span>
          </span>
          <span className="landing__stat">
            <span className="landing__stat-n n">14</span>
            <span className="landing__stat-l">benchmark suites</span>
          </span>
          <span className="landing__stat">
            <span className="landing__stat-n n">3</span>
            <span className="landing__stat-l">evaluation types</span>
          </span>
          <span className="landing__strip-note">
            OpenAI · Anthropic · Google · Groq · Together · OpenRouter · Azure · Fireworks · Ollama
          </span>
        </div>
      </section>

      {/* ==================== GRADING ==================== */}
      <section className="landing__band" id="grading">
        <div className="landing__shell">
          <div className="landing__band-head">
            <p className="landing__eyebrow">Grading</p>
            <h2>A score you can't audit is just a rumour.</h2>
            <p>
              Open any cell in the leaderboard and you get the prompt, the full response, and the
              reason it was marked. Here's prompt 112 from the run above.
            </p>
          </div>

          <div className="landing__band-body">
            <div className="landing__ask">
              <p className="landing__eyebrow">Prompt 112 · Billing policy</p>
              <p className="landing__ask-q">
                A customer on the Growth plan downgrades halfway through their billing cycle. What
                happens to the credits they've already paid for?
              </p>
              <p className="landing__ask-src">
                <b>Grading source — Billing Policy v4.2 §4.2:</b> unused credits carry over as
                account balance, no refund is issued, and the plan change applies from the next
                billing date.
              </p>
            </div>

            <div className="landing__answers">
              <article className="landing__ans landing__ans--pass">
                <div className="landing__ans-top">
                  <span className="landing__ans-name">Model Delta</span>
                  <span className="landing__ans-mark">Pass</span>
                </div>
                <p>
                  Their unused credits stay on the account as a balance and roll into the next
                  cycle. No refund is issued, and the downgrade takes effect on their next billing
                  date.
                </p>
                <div className="landing__ans-foot n">
                  <span>1.9s</span>
                  <span>412 tokens</span>
                  <span>3/3 facts matched</span>
                </div>
              </article>

              <article className="landing__ans landing__ans--pass">
                <div className="landing__ans-top">
                  <span className="landing__ans-name">Model Gamma</span>
                  <span className="landing__ans-mark">Pass</span>
                </div>
                <p>
                  Credits already purchased remain available as account balance. The downgrade
                  applies from the start of the next billing period rather than immediately.
                </p>
                <div className="landing__ans-foot n">
                  <span>1.1s</span>
                  <span>288 tokens</span>
                  <span>3/3 facts matched</span>
                </div>
              </article>

              <article className="landing__ans landing__ans--pass">
                <div className="landing__ans-top">
                  <span className="landing__ans-name">Model Beta</span>
                  <span className="landing__ans-mark">Pass</span>
                </div>
                <p>
                  The remaining credits carry over to the account. The plan downgrade is scheduled
                  for the next billing date.
                </p>
                <div className="landing__ans-foot n">
                  <span>2.7s</span>
                  <span>196 tokens</span>
                  <span>2/3 facts matched</span>
                </div>
              </article>

              <article className="landing__ans landing__ans--fail">
                <div className="landing__ans-top">
                  <span className="landing__ans-name">Model Alpha</span>
                  <span className="landing__ans-mark">Fail</span>
                </div>
                <p>
                  The customer receives a <span className="landing__hl">prorated refund</span> for
                  the unused portion of their Growth plan, processed back to their original
                  payment method within 5–7 business days.
                </p>
                <div className="landing__ans-note">
                  <b>Contradicts source.</b> §4.2 states no refund is issued. The refund window and
                  payment-method detail appear nowhere in the grading source.
                </div>
                <div className="landing__ans-foot n">
                  <span>0.4s</span>
                  <span>241 tokens</span>
                  <span>0/3 facts matched</span>
                </div>
              </article>
            </div>
          </div>
        </div>
      </section>

      {/* ==================== HOW A RUN WORKS ==================== */}
      <section className="landing__band landing__band--paper" id="run">
        <div className="landing__shell">
          <div className="landing__band-head">
            <p className="landing__eyebrow">How a run works</p>
            <h2>Four steps, and the answer is a table — not an opinion.</h2>
            <p>
              Same prompts, same grader, same conditions for every model in the run. Otherwise
              you're comparing two different experiments.
            </p>
          </div>

          <div className="landing__band-body landing__pipeline">
            <div className="landing__step">
              <p className="landing__step-k">STEP 01</p>
              <h3>Connect a provider</h3>
              <p>
                Paste an API key. SemcoEval reads the provider's model list and fills in context
                length, vision and tool-calling support for you.
              </p>
              <p className="landing__step-hint">No per-model setup</p>
            </div>
            <div className="landing__step">
              <p className="landing__step-k">STEP 02</p>
              <h3>Pick the shortlist</h3>
              <p>
                Filter every connected provider at once by price, speed, context window, modality
                or benchmark score, then select the models to compare.
              </p>
              <p className="landing__step-hint">One catalogue, all providers</p>
            </div>
            <div className="landing__step">
              <p className="landing__step-k">STEP 03</p>
              <h3>Choose the tests</h3>
              <p>Start from a standard benchmark suite, or upload your own prompts and expected answers.</p>
              <p className="landing__step-hint">CSV · JSON · JSONL</p>
            </div>
            <div className="landing__step">
              <p className="landing__step-k">STEP 04</p>
              <h3>Read the results</h3>
              <p>
                Scores, response times and cost land in one leaderboard, with every response kept
                so you can check the grading yourself.
              </p>
              <p className="landing__step-hint">Export as PDF or CSV</p>
            </div>
          </div>
        </div>
      </section>

      {/* ==================== MODES ==================== */}
      <section className="landing__band" id="modes">
        <div className="landing__shell">
          <div className="landing__band-head">
            <p className="landing__eyebrow">What you can test</p>
            <h2>Three kinds of system, three different scorecards.</h2>
            <p>
              A chat model and a tool-using agent fail in completely different ways, so they
              aren't graded the same way.
            </p>
          </div>

          <div className="landing__band-body landing__modes">
            <article className="landing__mode">
              <span className="landing__mode-tag">Fast evaluation</span>
              <h3>Chat &amp; text models</h3>
              <p>Grade base knowledge, summarisation quality and conversational tone against standardised suites.</p>
              <div className="landing__chips">
                <span className="landing__chip">accuracy</span>
                <span className="landing__chip">coherence</span>
                <span className="landing__chip">tone match</span>
                <span className="landing__chip">p95 latency</span>
              </div>
            </article>
            <article className="landing__mode">
              <span className="landing__mode-tag">For automation</span>
              <h3>Autonomous agents</h3>
              <p>
                Test multi-step tool execution — whether the agent picked the right tool, with the
                right arguments, in the right order.
              </p>
              <div className="landing__chips">
                <span className="landing__chip">task completion</span>
                <span className="landing__chip">tool accuracy</span>
                <span className="landing__chip">step count</span>
                <span className="landing__chip">cost per task</span>
              </div>
            </article>
            <article className="landing__mode">
              <span className="landing__mode-tag">High precision</span>
              <h3>Document search &amp; RAG</h3>
              <p>
                Measure how much of an answer is actually supported by the retrieved documents,
                and catch the answers that aren't.
              </p>
              <div className="landing__chips">
                <span className="landing__chip">groundedness</span>
                <span className="landing__chip">retrieval recall</span>
                <span className="landing__chip">citation match</span>
                <span className="landing__chip">refusal rate</span>
              </div>
            </article>
          </div>
        </div>
      </section>

      {/* ==================== PLATFORM ==================== */}
      <section className="landing__band landing__band--paper" id="platform">
        <div className="landing__shell">
          <div className="landing__band-head">
            <p className="landing__eyebrow">Platform</p>
            <h2>Built for the part after the demo.</h2>
          </div>

          <div className="landing__band-body landing__ledger">
            <div className="landing__row">
              <p className="landing__row-k n">01</p>
              <h3>Every response is kept</h3>
              <p>
                Scores are only as good as the grading behind them. Open any cell to read the
                exact prompt, the model's full response, and why it was marked right or wrong.
              </p>
            </div>
            <div className="landing__row">
              <p className="landing__row-k n">02</p>
              <h3>Re-run against a saved baseline</h3>
              <p>
                Pin a run as your baseline. When a provider ships a new version, re-run the same
                suite and see exactly which tests moved, and in which direction.
              </p>
            </div>
            <div className="landing__row">
              <p className="landing__row-k n">03</p>
              <h3>Cost projected to your real volume</h3>
              <p>
                Enter your expected monthly request count and each model's score sits next to what
                it would actually cost you at that scale — not per million tokens.
              </p>
            </div>
            <div className="landing__row">
              <p className="landing__row-k n">04</p>
              <h3>Keys stay in your workspace</h3>
              <p>
                Provider keys are stored per workspace with scoped team access. On-prem models
                running through Ollama never leave your network.
              </p>
            </div>
            <div className="landing__row">
              <p className="landing__row-k n">05</p>
              <h3>Reports your stakeholders will read</h3>
              <p>
                Turn a run into a shareable report with the recommendation, the trade-offs and the
                raw table — the thing you take into the architecture review.
              </p>
            </div>
          </div>
        </div>
      </section>

      {/* ==================== CLOSE ==================== */}
      <section className="landing__close">
        <div className="landing__shell landing__close-in">
          <p className="landing__eyebrow">Get started</p>
          <h2>Your next model decision, with the receipts.</h2>
          <p>
            Connect one provider, pick a standard suite, and have a defensible answer before your
            next architecture review.
          </p>
          <div className="landing__close-cta">
            <Link className="landing__btn landing__btn--primary landing__btn--lg" to="/app">
              Run an evaluation
            </Link>
            <a className="landing__btn landing__btn--ghost landing__btn--lg" href="#grading">
              See a sample report
            </a>
          </div>
        </div>
      </section>
    </div>
  );
};

export default Landing;



















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

  /* ---------- page header ---------- */
  &__header {
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
    padding: 32px 36px 36px;
    overflow: hidden;

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

  /* ---------- progress tracker ---------- */
  &__tracker {
    position: relative;
    padding-bottom: 28px;
    margin-bottom: 4px;
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
    margin-top: 2rem;
    padding-top: 1.5rem;
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
          <DatasetStep evalType={draft.type} selected={draft.dataset} onSelect={(id) => setDraft((d) => ({ ...d, dataset: id }))} />
        )}
        {step === 6 && (
          <MetricsStep evalType={draft.type} selected={draft.metrics} onToggle={(id) => toggleInArray('metrics', id)} />
        )}
        {step === 7 && <ReviewStep draft={draft} />}

        {error && <p className="run-eval__error">{error}</p>}

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
