//Footer.scss
@use '../../styles/variables' as *;

.app-footer {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  z-index: $z-footer;
  height: $footer-height;
  display: flex;
  align-items: center;
  background: $bg-subtle;
  border-top: 0.0625rem solid $border-subtle;

  &__inner {
    width: 100%;
    max-width: 75rem;
    margin: 0 auto;
    padding: 0 clamp(1.25rem, 5vw, 3.75rem);
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 0.875rem;

    p {
      font-size: 0.78125rem;
      color: $text-tertiary;
    }
  }

  &__links {
    margin-left: auto;
    display: flex;
    gap: 1.25rem;

    a {
      font-size: 0.8125rem;
      color: $text-secondary;
      text-decoration: none;

      &:hover {
        color: $text-primary;
      }
    }
  }
}






















//Footer.tsx
import type { FC } from 'react';
import './Footer.scss';

const Footer: FC = () => {
  return (
    <footer className="app-footer">
      <div className="app-footer__inner">
        <p>SemcoEval — Semco Enterprise Labs</p>
        <div className="app-footer__links">
          <a href="/#run">Docs</a>
          <a href="/#platform">Security</a>
        </div>
      </div>
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
  display: flex;
  align-items: center;
  background: rgba(255, 255, 255, 0.88);
  backdrop-filter: blur(0.75rem);
  border-bottom: 0.0625rem solid $border-subtle;

  &__inner {
    width: 100%;
    max-width: 75rem; // 1200px
    margin: 0 auto;
    padding: 0 clamp(1.25rem, 5vw, 3.75rem);
    display: flex;
    align-items: center;
    gap: 1.875rem;
  }

  &__brand {
    display: flex;
    align-items: center;
    gap: 0.625rem;
    text-decoration: none;
    color: $text-primary;
  }

  &__brand-mark {
    width: 1.75rem;
    height: 1.75rem;
    border-radius: 0.4375rem;
    background: $text-primary;
    color: #fff;
    display: grid;
    place-items: center;
    font-weight: 700;
    font-size: 0.875rem;
  }

  &__brand-name {
    font-weight: 700;
    font-size: 1.03rem;
    letter-spacing: -0.02em;
  }

  &__nav {
    margin-left: auto;
    display: flex;
    align-items: center;
    gap: 1.625rem;

    a {
      font-size: 0.84375rem;
      font-weight: 500;
      color: $text-secondary;
      text-decoration: none;

      &:hover {
        color: $text-primary;
      }
    }
  }

  @media (max-width: 38.75rem) {
    &__nav {
      display: none;
    }
  }
}
























//Header.tsx
import { Link } from 'react-router-dom';
import type { FC } from 'react';
import './Header.scss';

interface NavLink {
  label: string;
  href: string;
}

const NAV_LINKS: NavLink[] = [
  { label: 'Grading', href: '/#grading' },
  { label: 'How a run works', href: '/#run' },
  { label: 'What you can test', href: '/#modes' },
  { label: 'Platform', href: '/#platform' },
];

const Header: FC = () => {
  return (
    <header className="app-header">
      <div className="app-header__inner">
        <Link className="app-header__brand" to="/">
          <span className="app-header__brand-mark">S</span>
          <span className="app-header__brand-name">SemcoEval</span>
        </Link>

        <nav className="app-header__nav">
          {NAV_LINKS.map((link) => (
            <a key={link.label} href={link.href}>
              {link.label}
            </a>
          ))}
        </nav>
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

.app-sidebar {
  width: $sidebar-width;
  flex-shrink: 0;
  height: 100%;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  background: $bg-main;
  border-right: 0.0625rem solid $border-subtle;
  padding: 0.875rem 0.75rem;

  &__top {
    flex: 1;
    display: flex;
    flex-direction: column;
  }

  &__brand {
    display: flex;
    align-items: center;
    gap: 0.625rem;
    padding: 0.5rem 0.625rem 1.25rem;
  }

  &__brand-logo {
    width: 1.75rem;
    height: 1.75rem;
    border-radius: 0.4375rem;
    background: $text-primary;
    color: #fff;
    display: grid;
    place-items: center;
    font-weight: 700;
    font-size: 0.875rem;
  }

  &__brand-title {
    font-family: $font-display;
    font-weight: 700;
    font-size: 1rem;
    color: $text-primary;
  }

  &__nav {
    list-style: none;
    display: flex;
    flex-direction: column;
    gap: 0.125rem;
  }

  &__item {
    display: flex;
    align-items: center;
    gap: 0.6875rem;
    padding: 0.5625rem 0.6875rem;
    border-radius: 0.5rem;
    font-size: 0.875rem;
    font-weight: 500;
    color: $text-secondary;
    text-decoration: none;
    border: none;
    background: transparent;
    width: 100%;
    cursor: pointer;
    transition: background 0.14s ease, color 0.14s ease;

    svg {
      opacity: 0.8;
      flex-shrink: 0;
    }

    &:hover {
      background: $bg-subtle;
      color: $text-primary;
    }

    &--active {
      background: $primary-light;
      color: $primary;
      font-weight: 600;
      position: relative;

      svg {
        opacity: 1;
      }

      &::before {
        content: '';
        position: absolute;
        left: -0.75rem;
        top: 50%;
        transform: translateY(-50%);
        width: 0.1875rem;
        height: 1.125rem;
        border-radius: 0 0.1875rem 0.1875rem 0;
        background: $primary;
      }
    }

    &--highlight {
      background: $primary;
      color: #fff;
      font-weight: 600;
      margin-bottom: 0.875rem;

      svg {
        opacity: 1;
      }

      &:hover {
        background: $primary-hover;
        color: #fff;
      }
    }

    &--button {
      text-align: left;
    }
  }

  &__footer {
    border-top: 0.0625rem solid $border-subtle;
    padding-top: 0.625rem;
    display: flex;
    flex-direction: column;
    gap: 0.125rem;
  }
}


















//Sidebar.tsx
import { NavLink } from 'react-router-dom';
import type { FC } from 'react';
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
} from 'lucide-react';
import './Sidebar.scss';

interface NavItem {
  to: string;
  label: string;
  icon: LucideIcon;
}

const PRIMARY_NAV: NavItem[] = [
  { to: '/app/history', label: 'History', icon: Clock },
  { to: '/app/models', label: 'Models', icon: Box },
  { to: '/app/providers', label: 'Providers', icon: PlugZap },
  { to: '/app/datasets', label: 'Test Suites', icon: Database },
  { to: '/app/reports', label: 'Reports', icon: FileBarChart },
];

const Sidebar: FC = () => {
  return (
    <aside className="app-sidebar">
      <div className="app-sidebar__top">
        <div className="app-sidebar__brand">
          <div className="app-sidebar__brand-logo">S</div>
          <span className="app-sidebar__brand-title">SemcoEval</span>
        </div>

        <ul className="app-sidebar__nav">
          <li>
            <NavLink to="/app/run-evaluation" className="app-sidebar__item app-sidebar__item--highlight">
              <Play size={17} />
              <span>New Evaluation</span>
            </NavLink>
          </li>
          <li>
            <NavLink
              to="/app"
              end
              className={({ isActive }) =>
                `app-sidebar__item${isActive ? ' app-sidebar__item--active' : ''}`
              }
            >
              <LayoutGrid size={17} />
              <span>Dashboard</span>
            </NavLink>
          </li>
          {PRIMARY_NAV.map(({ to, label, icon: Icon }) => (
            <li key={to}>
              <NavLink
                to={to}
                className={({ isActive }) =>
                  `app-sidebar__item${isActive ? ' app-sidebar__item--active' : ''}`
                }
              >
                <Icon size={17} />
                <span>{label}</span>
              </NavLink>
            </li>
          ))}
        </ul>
      </div>

      <div className="app-sidebar__footer">
        <NavLink
          to="/app/settings"
          className={({ isActive }) =>
            `app-sidebar__item${isActive ? ' app-sidebar__item--active' : ''}`
          }
        >
          <Settings size={17} />
          <span>Settings</span>
        </NavLink>
        <button type="button" className="app-sidebar__item app-sidebar__item--button">
          <CircleHelp size={17} />
          <span>Help</span>
        </button>
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
    max-width: 75rem;
    margin: 0 auto;
    padding: 0 var(--gut);
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
      width: 1.25rem;
      height: 0.0625rem;
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
    max-width: 45rem;
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
        height: 0.375rem;
        background: repeating-linear-gradient(to right, var(--blue) 0 0.09375rem, transparent 0.09375rem 0.4375rem);
        opacity: 0.45;
      }
    }
  }

  &__hero-sub {
    margin-top: 1.375rem;
    max-width: 35rem;
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

  &__hero-note {
    font-size: 0.78125rem;
    color: var(--ink-3);
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
      width: 0.375rem;
      height: 0.375rem;
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
    height: 0.125rem;
    background: var(--rule);
    border-radius: 0.125rem;

    &::after {
      content: '';
      position: absolute;
      left: 0;
      right: 0;
      top: 0;
      height: 0.4375rem;
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
    width: 0.8125rem;
    height: 0.8125rem;
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
    max-width: 41.25rem;

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
    border-top: 0.0625rem solid var(--rule);
  }

  &__step {
    padding: 1.5rem 1.5rem 1.625rem 0;
    border-right: 0.0625rem solid var(--rule);
    position: relative;

    &:last-child {
      border-right: 0;
    }

    &::before {
      content: '';
      position: absolute;
      top: -0.0625rem;
      left: 0;
      width: 2.5rem;
      height: 0.1875rem;
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
    max-width: 43.75rem;

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
  @media (max-width: 62.5rem) {
    .landing__pipeline {
      grid-template-columns: 1fr 1fr;
    }
    .landing__step:nth-child(2) {
      border-right: 0;
    }
    .landing__step:nth-child(-n + 2) {
      border-bottom: 0.0625rem solid var(--rule);
    }
    .landing__modes {
      grid-template-columns: 1fr;
    }
  }

  @media (max-width: 53.75rem) {
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

  @media (max-width: 38.75rem) {
    .landing__pipeline {
      grid-template-columns: 1fr;
    }
    .landing__step {
      border-right: 0;
      border-bottom: 0.0625rem solid var(--rule);
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
              <span className="landing__hero-note">Connect one API key to start</span>
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
                      Llama 3.3 70B <b className="n">79.5</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                  <span className="landing__pip landing__pip--low" style={{ '--x': '60.3%', '--d': '.12s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      Hermes 3 70B <b className="n">84.1</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                  <span className="landing__pip landing__pip--brand" style={{ '--x': '71.8%', '--d': '.19s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      GPT-4o <b className="n">88.7</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                  <span
                    className="landing__pip landing__pip--best landing__pip--low"
                    style={{ '--x': '78.0%', '--d': '.26s' } as PipStyle}
                  >
                    <span className="landing__pip-flag">
                      Claude 3.5 Sonnet <b className="n">91.2</b>
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
                      Llama 3.3 70B <b className="n">0.4</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                  <span className="landing__pip landing__pip--brand" style={{ '--x': '37.5%', '--d': '.17s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      GPT-4o <b className="n">1.2</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                  <span className="landing__pip" style={{ '--x': '65.6%', '--d': '.24s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      Claude 3.5 Sonnet <b className="n">2.1</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                  <span className="landing__pip landing__pip--low" style={{ '--x': '90.6%', '--d': '.31s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      Hermes 3 70B <b className="n">2.9</b>
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
                      Llama 3.3 70B <b className="n">$2.40</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                  <span className="landing__pip landing__pip--low" style={{ '--x': '13.6%', '--d': '.22s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      Hermes 3 70B <b className="n">$6.80</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                  <span className="landing__pip landing__pip--brand" style={{ '--x': '49.0%', '--d': '.29s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      GPT-4o <b className="n">$24.50</b>
                    </span>
                    <span className="landing__pip-dot"></span>
                  </span>
                  <span className="landing__pip" style={{ '--x': '82.4%', '--d': '.36s' } as PipStyle}>
                    <span className="landing__pip-flag">
                      Claude 3.5 Sonnet <b className="n">$41.20</b>
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
                <b>Claude 3.5 Sonnet</b>
              </span>
              <span>
                <span className="landing__k">Fastest</span>
                <b>Llama 3.3 70B</b>
              </span>
              <span>
                <span className="landing__k">Cheapest</span>
                <b>Llama 3.3 70B</b>
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
                  <span className="landing__ans-name">Claude 3.5 Sonnet</span>
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
                  <span className="landing__ans-name">GPT-4o</span>
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
                  <span className="landing__ans-name">Hermes 3 70B</span>
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
                  <span className="landing__ans-name">Llama 3.3 70B</span>
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

















//Dashboard.tsx
import type { FC } from 'react';
import PagePlaceholder from '../../../components/PagePlaceholder/PagePlaceholder';

const Dashboard: FC = () => <PagePlaceholder title="Dashboard" subtitle="Compare AI models, run standardized tests, and make data-driven decisions." />;

export default Dashboard;













//Datasets.tsx
import type { FC } from 'react';
import PagePlaceholder from '../../../components/PagePlaceholder/PagePlaceholder';

const Datasets: FC = () => (
  <PagePlaceholder title="Test Suites" subtitle="Standard benchmarks and custom datasets" />
);

export default Datasets;
















//History.tsx

import type { FC } from 'react';
import PagePlaceholder from '../../../components/PagePlaceholder/PagePlaceholder';

const History: FC = () => <PagePlaceholder title="History" subtitle="Past evaluation runs" />;

export default History;

















//Models.tsx
import type { FC } from 'react';
import PagePlaceholder from '../../../components/PagePlaceholder/PagePlaceholder';

const Models: FC = () => <PagePlaceholder title="Models" subtitle="Browse available models across providers" />;

export default Models;













//Providers.tsx
import type { FC } from 'react';
import PagePlaceholder from '../../../components/PagePlaceholder/PagePlaceholder';

const Providers: FC = () => <PagePlaceholder title="Providers" subtitle="Manage connected API providers" />;

export default Providers;













//Reports.tsx
import type { FC } from 'react';
import PagePlaceholder from '../../../components/PagePlaceholder/PagePlaceholder';

const Reports: FC = () => <PagePlaceholder title="Reports" subtitle="Generated analysis and recommendations" />;

export default Reports;













//RunEvaluation.tsx
import type { FC } from 'react';
import PagePlaceholder from '../../../components/PagePlaceholder/PagePlaceholder';

const RunEvaluation: FC = () => (
  <PagePlaceholder title="New Evaluation" subtitle="Run a model comparison" />
);

export default RunEvaluation;
















//Settings.tsx
import type { FC } from 'react';
import PagePlaceholder from '../../../components/PagePlaceholder/PagePlaceholder';

const Settings: FC = () => <PagePlaceholder title="Settings" subtitle="Workspace configuration" />;

export default Settings;


















//uiSlice.ts
import { createSlice, type PayloadAction } from '@reduxjs/toolkit';

export interface UiState {
  sidebarCollapsed: boolean;
  activeNavItem: string;
}

const initialState: UiState = {
  sidebarCollapsed: false,
  activeNavItem: 'dashboard',
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
  },
});

export const { toggleSidebar, setSidebarCollapsed, setActiveNavItem } = uiSlice.actions;
export default uiSlice.reducer;



















//hooks.ts
import { useDispatch, useSelector, type TypedUseSelectorHook } from 'react-redux';
import type { RootState, AppDispatch } from './index';

export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;














//index.ts
import { configureStore } from '@reduxjs/toolkit';
import uiReducer from './slices/uiSlice';

export const store = configureStore({
  reducer: {
    ui: uiReducer,
  },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;

export default store;











//_variables.scss
// ============================================================
// SemcoEval — design tokens (converted from theme.css / landing.css)
// Base: 1rem = 16px
// ============================================================

// ---------- Brand / primary ----------
$primary: #1428a0;
$primary-hover: #1d37c9;
$primary-light: #eef1fe;
$primary-subtle: #e2e7fc;

// ---------- Surfaces ----------
$bg-page: #f6f7f9;
$bg-subtle: #f3f5f8;
$bg-inset: #edf0f4;
$bg-main: #ffffff;

// ---------- Borders ----------
$border-default: #dce0e7;
$border-subtle: #e9ecf1;
$border-strong: #c7cdd8;

// ---------- Text ----------
$text-primary: #0e1526;
$text-secondary: #46506b;
$text-tertiary: #7a8399;

// ---------- Status ----------
$success: #0f7a5a;
$success-subtle: #e4f4ee;
$warning: #b7791f;
$warning-subtle: #fdf3e0;
$danger: #c0303b;
$danger-subtle: #fcebec;

// ---------- Shadows ----------
$shadow-xs: 0 0.0625rem 0.125rem rgba(14, 21, 38, 0.04);
$shadow-sm: 0 0.0625rem 0.125rem rgba(14, 21, 38, 0.05);
$shadow-md: 0 0.125rem 0.25rem rgba(14, 21, 38, 0.05), 0 0.5rem 1.25rem -0.75rem rgba(14, 21, 38, 0.16);
$shadow-lg: 0 0.25rem 0.5rem rgba(14, 21, 38, 0.05), 0 1.125rem 2.75rem -1.375rem rgba(14, 21, 38, 0.24);
$shadow-xl: 0 1.75rem 4.375rem -1.875rem rgba(14, 21, 38, 0.34);

// ---------- Radius ----------
$radius-sm: 0.375rem;
$radius-md: 0.5rem;
$radius-lg: 0.75rem;
$radius-xl: 1rem;

// ---------- Typography ----------
$font-display: 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
$font-body: 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
$font-mono: 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;

// ---------- Layout ----------
$header-height: 3.75rem; // 60px
$footer-height: 3.375rem; // 54px
$sidebar-width: 15rem; // 240px

// ---------- Z-index ----------
$z-header: 100;
$z-footer: 100;
$z-sidebar: 90;

















//global.scss
@use './variables' as *;

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
  font-size: 1rem;
  line-height: 1.55;
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















//App.tsx
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

const App: FC = () => {
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


















//main.tsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';
import { Provider } from 'react-redux';
import store from './store';
import App from './App.tsx';
import './styles/global.scss';

createRoot(document.getElementById('root') as HTMLElement).render(
  <StrictMode>
    <Provider store={store}>
      <BrowserRouter>
        <App />
      </BrowserRouter>
    </Provider>
  </StrictMode>
);












//package.json
{
  "name": "semcoeval-app",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "lint": "oxlint",
    "preview": "vite preview"
  },
  "dependencies": {
    "@reduxjs/toolkit": "^2.12.0",
    "lucide-react": "^1.27.0",
    "react": "^19.2.7",
    "react-dom": "^19.2.7",
    "react-redux": "^9.3.0",
    "react-router-dom": "^7.18.1",
    "sass": "^1.102.0"
  },
  "devDependencies": {
    "@types/node": "^26.1.2",
    "@types/react": "^19.2.17",
    "@types/react-dom": "^19.2.3",
    "@vitejs/plugin-react": "^6.0.3",
    "oxlint": "^1.71.0",
    "typescript": "^7.0.2",
    "vite": "^8.1.1"
  }
}
