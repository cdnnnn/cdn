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
    color: $on-primary;
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
      box-shadow: 0 0 0 0.1875rem $primary-subtle;
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
    color: $on-primary;
    box-shadow: $shadow-sm;
    transition: transform 0.18s ease, background 0.14s ease;

    &--light {
      background: $warning;
    }

    &--dark {
      background: $primary;
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
  background: $bg-main;
  border-top: 1px solid $border-subtle;

  &__version,
  &__copyright {
    font-size: 0.71875rem;
    color: $text-tertiary;
    white-space: nowrap;
  }
}



















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
  // Theme-aware surface color for cards/panels/buttons on this page —
  // was hardcoded to #fff, which showed white cards floating on a dark page.
  --white: #{$bg-main};
  // Primary button text needs to stay constant white against the blue fill
  // in both themes, unlike --white above which now follows the theme.
  --on-brand: #{$on-primary};
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

  /* ================= LEADERBOARD CARD (hero signature) ================= */
  &__board {
    margin-top: clamp(2.625rem, 5.5vw, 4.25rem);
    background: var(--white);
    border: 1px solid var(--rule);
    border-radius: 1rem;
    // Was hardcoded to light-mode-only rgba(14,21,38,...) values; $shadow-lg
    // resolves correctly in both themes (and now carries the glossy inset
    // highlight in dark mode too).
    box-shadow: $shadow-lg;
    overflow: hidden;
  }

  &__board-head {
    display: flex;
    align-items: center;
    flex-wrap: wrap;
    gap: 0.5rem 0.875rem;
    padding: 1.125rem 1.375rem;
    border-bottom: 1px solid var(--rule-2);
  }

  &__board-title {
    font-size: 0.9375rem;
    font-weight: 700;
    letter-spacing: -0.01em;
  }

  &__board-meta {
    margin-left: auto;
    font-size: 0.8125rem;
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

  &__board-cols {
    display: grid;
    grid-template-columns: 2.5rem 1.5fr 1fr 1fr 1fr;
    padding: 0 1.375rem;
    font-size: 0.71875rem;
    font-weight: 600;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: var(--ink-3);
    gap: 0.75rem;

    span {
      padding: 0.75rem 0;
    }

    .num {
      text-align: right;
    }
  }

  &__board-rows {
    padding: 0 1.375rem 0.5rem;
  }

  &__board-row {
    display: grid;
    grid-template-columns: 2.5rem 1.5fr 1fr 1fr 1fr;
    align-items: center;
    gap: 0.75rem;
    padding: 0.875rem 0;
    border-top: 1px solid var(--rule-2);

    &--best &__board-rank,
    &--best .landing__board-rank {
      background: var(--jade-w);
      border-color: var(--jade);
      color: var(--jade);
    }
  }

  &__board-rank {
    width: 26px;
    height: 26px;
    border-radius: 50%;
    background: var(--paper);
    border: 1px solid var(--rule);
    display: grid;
    place-items: center;
    font-size: 0.75rem;
    font-weight: 700;
    color: var(--ink-2);
  }

  &__board-model {
    display: flex;
    align-items: center;
    gap: 0.625rem;
    min-width: 0;
  }

  &__board-dot {
    width: 8px;
    height: 8px;
    border-radius: 50%;
    flex-shrink: 0;

    &--jade {
      background: var(--jade);
    }

    &--blue {
      background: var(--blue);
    }

    &--muted {
      background: var(--ink-3);
    }
  }

  &__board-name {
    font-size: 0.9375rem;
    font-weight: 600;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;

    small {
      display: block;
      font-size: 0.75rem;
      font-weight: 400;
      color: var(--ink-3);
    }
  }

  &__board-metric {
    text-align: right;
  }

  &__board-bar {
    margin-top: 5px;
    height: 4px;
    border-radius: 2px;
    background: var(--rule-2);
    overflow: hidden;

    i {
      display: block;
      height: 100%;
      border-radius: 2px;
      background: var(--blue);

      &.landing__board-bar-fill--best {
        background: var(--jade);
      }
    }
  }

  &__val {
    font-size: 0.9375rem;
    font-weight: 700;

    &--best {
      color: var(--jade);
    }
  }

  &__board-foot {
    display: flex;
    flex-wrap: wrap;
    gap: 0.5rem 1.625rem;
    padding: 1rem 1.375rem;
    border-top: 1px solid var(--rule-2);
    background: var(--paper);
    font-size: 0.84375rem;
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

  /* ================= STATS ================= */
  &__stats {
    background: var(--paper);
    border-block: 0.0625rem solid var(--rule-2);
  }

  &__stats-in {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
  }

  &__stat-card {
    padding: 1.75rem var(--gut);
    text-align: left;
    border-left: 1px solid var(--rule-2);

    &:first-child {
      border-left: 0;
    }
  }

  &__stat-n {
    display: block;
    font-size: 2rem;
    font-weight: 700;
    letter-spacing: -0.03em;
    color: var(--ink);
  }

  &__stat-l {
    display: block;
    margin-top: 2px;
    font-size: 0.875rem;
    color: var(--ink-2);
  }

  &__stats-note {
    padding: 0.875rem var(--gut);
    border-top: 1px solid var(--rule-2);
    font-size: 0.8125rem;
    color: var(--ink-3);
    text-align: center;
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
    .landing__stats-in {
      grid-template-columns: 1fr 1fr;
    }
    .landing__stat-card:nth-child(3) {
      border-left: 0;
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
    .landing__board-cols,
    .landing__board-row {
      grid-template-columns: 2.25rem 1.4fr 1fr 1fr;
    }
    .landing__lat {
      display: none;
    }
  }

  @media (max-width: 620px) {
    .landing__pipeline {
      grid-template-columns: 1fr;
    }
    .landing__stats-in {
      grid-template-columns: 1fr 1fr;
    }
    .landing__stat-card {
      border-left: 0;
      border-top: 1px solid var(--rule-2);
      padding: 1.25rem var(--gut);

      &:nth-child(1),
      &:nth-child(2) {
        border-top: 0;
      }
    }
    .landing__board-cols,
    .landing__board-row {
      grid-template-columns: 2rem 1.3fr 1fr;
    }
    .landing__cost {
      display: none;
    }
  }
}
