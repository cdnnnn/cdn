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
    items: [
      { to: '/app', label: 'Dashboard', icon: LayoutGrid, end: true },
      { to: '/app/run-evaluation', label: 'New Evaluation', icon: Play },
    ],
  },
  {
    label: 'Workspace',
    items: [
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















//Sidebar.scss
@use '../../styles/variables' as *;

$sidebar-width-collapsed: 4.25rem;

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
