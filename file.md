// ═══════════════════════════════════════════════
// styles/_variables.scss — Design tokens
// SemcoEval · restyled to match Content Analytics'
// dark navy / violet "glass panel" system
// ═══════════════════════════════════════════════

:root {
  // ── Surfaces (stepped navy) ──
  --bg0: #050c22; // outermost frame / gap
  --bg-page: #050c22; // scrollable page background
  --bg-main: #091130; // panel surface (header, footer, sidebar, cards)
  --bg-subtle: #112057; // hover state / input bg
  --bg-active: #162770; // active / pressed bg

  // ── Borders ──
  --border-subtle: rgba(139, 92, 246, 0.14);
  --border-strong: rgba(139, 92, 246, 0.28);

  // ── Text ──
  --text-primary: #e6eeff;
  --text-secondary: #b8c8f2;
  --text-tertiary: #7a90cc;

  // ── Brand / accent (violet) ──
  --primary: #8b5cf6;
  --primary-2: #a78bfa;
  --primary-light: rgba(139, 92, 246, 0.14);
  --primary-border: rgba(139, 92, 246, 0.30);

  // ── Status ──
  --green: #4ec87a;
  --green-light: rgba(78, 200, 122, 0.10);
  --green-border: rgba(78, 200, 122, 0.22);

  --amber: #f0a030;
  --amber-light: rgba(240, 160, 48, 0.10);
  --amber-border: rgba(240, 160, 48, 0.22);

  --red: #f06060;
  --red-light: rgba(240, 96, 96, 0.10);
  --red-border: rgba(240, 96, 96, 0.22);

  // ── Radii ──
  --r: 7px;
  --r-lg: 11px;
  --r-xl: 15px;

  // ── Shadows ──
  --shadow: 0 4px 24px rgba(5, 10, 40, 0.55);
  --shadow-sm: 0 2px 10px rgba(5, 10, 40, 0.40);

  // ── Fonts ──
  --font-ui: 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  --font-mono: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
}

// ── Legacy SCSS-variable aliases ──
// kept so existing partials (Sidebar.scss, PagePlaceholder.scss, etc.)
// that reference $variables keep compiling unchanged.
$bg-page: var(--bg-page);
$bg-main: var(--bg-main);
$bg-subtle: var(--bg-subtle);
$bg-active: var(--bg-active);

$border-subtle: var(--border-subtle);
$border-strong: var(--border-strong);

$text-primary: var(--text-primary);
$text-secondary: var(--text-secondary);
$text-tertiary: var(--text-tertiary);

$primary: var(--primary);
$primary-2: var(--primary-2);
$primary-light: var(--primary-light);
$primary-border: var(--primary-border);

$green: var(--green);
$amber: var(--amber);
$red: var(--red);

$font-mono: var(--font-mono);
$font-ui: var(--font-ui);

$radius: var(--r);
$radius-lg: var(--r-lg);
$radius-xl: var(--r-xl);

$shadow: var(--shadow);
$shadow-sm: var(--shadow-sm);

// ── Layout sizing ──
$header-height: 56px;
$footer-height: 40px;
$sidebar-width: 240px;

























// ═══════════════════════════════════════════════
// styles/_mixins.scss — Reusable SCSS mixins
// SemcoEval
// ═══════════════════════════════════════════════

@mixin flex-center {
  display: flex;
  align-items: center;
}

@mixin flex-between {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

@mixin mono {
  font-family: $font-mono;
}

@mixin label-caps {
  font-size: 0.6875rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  color: $text-tertiary;
  @include mono;
}

@mixin glass-panel {
  background: $bg-main;
  border-radius: $radius;
  outline: 1px solid $border-subtle;
  transition: outline-color 0.3s ease, box-shadow 0.3s ease;

  &:hover {
    outline-color: $border-strong;
    box-shadow: $shadow, inset 0 1px 0 rgba(255, 255, 255, 0.04);
  }
}

@mixin scrollbar {
  &::-webkit-scrollbar {
    width: 3px;
    height: 3px;
  }

  &::-webkit-scrollbar-track {
    background: transparent;
  }

  &::-webkit-scrollbar-thumb {
    background: $border-strong;
    border-radius: 99px;
  }
}

@mixin truncate {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}


















// ═══════════════════════════════════════════════
// styles/global.scss — Base reset & body styles
// SemcoEval
// ═══════════════════════════════════════════════
@use 'variables' as *;

*,
*::before,
*::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

html {
  font-size: 15px;

  @media (min-width: 1600px) {
    font-size: 16px;
  }
}

body {
  font-family: $font-ui;
  background: $bg-page;
  color: $text-primary;
  min-height: 100vh;
}

// ── Scrollbars ──
::-webkit-scrollbar {
  width: 3px;
  height: 3px;
}

::-webkit-scrollbar-track {
  background: transparent;
}

::-webkit-scrollbar-thumb {
  background: $border-strong;
  border-radius: 99px;
}

// ── Numeric / tabular text helper (used across dashboards) ──
.n {
  font-family: $font-mono;
  font-variant-numeric: tabular-nums;
}
















@use '../../styles/variables' as *;
@use '../../styles/mixins' as m;

.app-header {
  height: $header-height;
  flex-shrink: 0;
  background: $bg-main;
  border-bottom: 1px solid $border-subtle;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 1.25rem;
  gap: 0.75rem;
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  z-index: 50;

  /* ---------- brand ---------- */
  &__brand {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    text-decoration: none;
    flex-shrink: 0;
  }

  &__brand-mark {
    display: grid;
    place-items: center;
    width: 30px;
    height: 30px;
    border-radius: 8px;
    color: #fff;
    background: linear-gradient(135deg, $primary, $primary-2);
    box-shadow: 0 2px 10px rgba(139, 92, 246, 0.35);
  }

  &__brand-name {
    font-size: 1rem;
    font-weight: 700;
    letter-spacing: -0.02em;
    color: $text-primary;
  }

  /* ---------- user cluster ---------- */
  &__user {
    position: relative;
  }

  &__avatar {
    width: 32px;
    height: 32px;
    border-radius: 50%;
    display: grid;
    place-items: center;
    background: linear-gradient(135deg, $primary, $primary-2);
    color: #fff;
    font-size: 0.8125rem;
    font-weight: 700;
    border: 2px solid transparent;
    cursor: pointer;
    @include m.mono;
    transition: border-color 0.15s, box-shadow 0.15s;

    &:hover {
      border-color: $primary-border;
      box-shadow: 0 0 0 3px $primary-light;
    }

    &--open {
      border-color: $primary;
      box-shadow: 0 0 0 3px $primary-light;
    }
  }

  &__dropdown {
    position: absolute;
    top: calc(100% + 10px);
    right: 0;
    width: 230px;
    background: $bg-main;
    border: 1px solid $border-strong;
    border-radius: $radius-lg;
    box-shadow: $shadow;
    overflow: hidden;
    z-index: 200;
    animation: dropIn 0.14s cubic-bezier(0.16, 1, 0.3, 1);
  }

  &__drop-user {
    display: flex;
    align-items: center;
    gap: 0.625rem;
    padding: 0.875rem 0.875rem 0.75rem;
  }

  &__drop-avatar {
    width: 34px;
    height: 34px;
    border-radius: 50%;
    display: grid;
    place-items: center;
    background: linear-gradient(135deg, $primary, $primary-2);
    color: #fff;
    font-weight: 700;
    font-size: 0.8125rem;
    flex-shrink: 0;
    @include m.mono;
  }

  &__drop-info {
    min-width: 0;
    flex: 1;
  }

  &__drop-name {
    font-size: 0.8125rem;
    font-weight: 600;
    color: $text-primary;
    @include m.truncate;
  }

  &__drop-role {
    font-size: 0.75rem;
    color: $text-tertiary;
    margin-top: 1px;
    @include m.truncate;
    @include m.mono;
  }

  &__drop-divider {
    height: 1px;
    background: $border-subtle;
  }

  &__drop-item {
    display: flex;
    align-items: center;
    gap: 0.5625rem;
    width: 100%;
    padding: 0.625rem 0.875rem;
    background: transparent;
    border: none;
    color: $text-secondary;
    font-size: 0.8125rem;
    cursor: pointer;
    text-align: left;
    transition: background 0.1s, color 0.1s;

    svg {
      color: $text-tertiary;
      transition: color 0.1s;
    }

    &:hover {
      background: $bg-subtle;
      color: $red;

      svg {
        color: $red;
      }
    }
  }
}

@keyframes dropIn {
  from {
    opacity: 0;
    transform: translateY(-6px) scale(0.97);
  }

  to {
    opacity: 1;
    transform: translateY(0) scale(1);
  }
}

















@use '../../styles/variables' as *;
@use '../../styles/mixins' as m;

.app-footer {
  height: $footer-height;
  flex-shrink: 0;
  background: $bg-main;
  border-top: 1px solid $border-subtle;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 1.25rem;
  position: fixed;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 40;

  &__version {
    font-size: 0.75rem;
    color: $text-tertiary;
    background: $bg-subtle;
    border: 1px solid $border-subtle;
    border-radius: 99px;
    padding: 0.0625rem 0.4375rem;
    @include m.mono;
  }

  &__copyright {
    font-size: 0.75rem;
    color: $text-tertiary;
    white-space: nowrap;
    @include m.mono;
  }
}















@use '../../styles/variables' as *;
@use '../../styles/mixins' as m;

$sidebar-width-collapsed: 60px;

@keyframes activeSlideIn {
  from {
    opacity: 0;
    transform: translateX(-6px);
  }

  to {
    opacity: 1;
    transform: translateX(0);
  }
}

@keyframes labelFadeIn {
  from {
    opacity: 0;
    transform: translateX(-4px);
  }

  to {
    opacity: 1;
    transform: translateX(0);
  }
}

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
  border-radius: $radius;
  outline: 1px solid $border-subtle;
  padding: 0.75rem 0.5625rem 1.125rem;
  transition:
    width 0.22s cubic-bezier(0.4, 0, 0.2, 1),
    outline-color 0.3s ease,
    box-shadow 0.3s ease;
  @include m.scrollbar;

  &:hover {
    outline-color: $border-strong;
    box-shadow: $shadow, inset 0 1px 0 rgba(255, 255, 255, 0.04);
  }

  &--collapsed {
    width: $sidebar-width-collapsed;
    padding-left: 0.375rem;
    padding-right: 0.375rem;
    align-items: center;
  }

  &__top {
    flex: 1;
    display: flex;
    flex-direction: column;
    min-height: 0;
    width: 100%;
  }

  /* ---------- collapse toggle ---------- */
  &__collapse {
    display: flex;
    align-items: center;
    justify-content: space-between;
    width: 100%;
    padding: 0.4375rem 0.625rem;
    margin-bottom: 0.75rem;
    border: none;
    border-radius: 9px;
    background: transparent;
    color: $text-tertiary;
    font-size: 0.75rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.08em;
    cursor: pointer;
    @include m.mono;
    transition: background 0.13s ease, color 0.13s ease;

    &:hover {
      background: $bg-subtle;
      color: $text-primary;
    }
  }

  &--collapsed &__collapse {
    justify-content: center;
  }

  &__collapse-ic {
    transition: transform 0.25s cubic-bezier(0.4, 0, 0.2, 1);
    flex-shrink: 0;

    &--flipped {
      transform: rotate(180deg);
    }
  }

  /* ---------- nav sections ---------- */
  &__nav {
    display: flex;
    flex-direction: column;
    gap: 1.25rem;
    margin-top: 0.25rem;
    width: 100%;
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
    font-size: 0.6875rem;
    font-weight: 700;
    letter-spacing: 0.1em;
    text-transform: uppercase;
    color: $text-tertiary;
    padding: 0.25rem 0.625rem 0.375rem;
    animation: labelFadeIn 0.18s ease-out;
  }

  /* ---------- nav items ---------- */
  &__icon {
    display: grid;
    place-items: center;
    flex-shrink: 0;
    color: inherit;
    opacity: 0.65;
    transition: opacity 0.15s, filter 0.15s;
  }

  &__item {
    position: relative;
    display: flex;
    align-items: center;
    gap: 0.625rem;
    padding: 0.5rem 0.625rem;
    border-radius: 9px;
    font-size: 0.8125rem;
    font-weight: 500;
    color: $text-secondary;
    text-decoration: none;
    border: 1px solid transparent;
    background: transparent;
    width: 100%;
    cursor: pointer;
    overflow: hidden;
    white-space: nowrap;
    transition:
      background 0.15s ease,
      color 0.15s ease,
      border-color 0.15s ease,
      padding 0.25s cubic-bezier(0.4, 0, 0.2, 1),
      width 0.25s cubic-bezier(0.4, 0, 0.2, 1);

    &:hover {
      background: $bg-subtle;
      color: $text-primary;
      border-color: $border-subtle;

      .app-sidebar__icon {
        opacity: 1;
      }
    }

    &--active {
      background: linear-gradient(135deg, rgba(139, 92, 246, 0.16) 0%, rgba(167, 139, 250, 0.08) 100%);
      color: $primary-2;
      border-color: $primary-border;
      font-weight: 600;
      animation: activeSlideIn 0.18s ease-out;

      &::before {
        content: '';
        position: absolute;
        left: 0;
        top: 20%;
        bottom: 20%;
        width: 3px;
        border-radius: 0 3px 3px 0;
        background: linear-gradient(180deg, $primary, $primary-2);
        box-shadow: 0 0 8px rgba(139, 92, 246, 0.5);
      }

      .app-sidebar__icon {
        opacity: 1;
        color: $primary;

        svg {
          filter: drop-shadow(0 0 4px rgba(139, 92, 246, 0.45));
        }
      }

      &:hover {
        background: linear-gradient(135deg, rgba(139, 92, 246, 0.2) 0%, rgba(167, 139, 250, 0.1) 100%);
        color: $primary-2;
      }
    }

    &--button {
      text-align: left;
    }

    &--collapsed {
      justify-content: center;
      padding: 0.5625rem !important;
      width: 38px !important;
      gap: 0;
      margin: 0 auto;

      &.app-sidebar__item--active::before {
        top: 25%;
        bottom: 25%;
      }
    }
  }

  /* ---------- footer ---------- */
  &__footer {
    border-top: 1px solid $border-subtle;
    margin-top: 0.875rem;
    padding-top: 0.625rem;
    display: flex;
    flex-direction: column;
    gap: 0.125rem;
    width: 100%;
  }
}

















@use '../styles/variables' as *;

.main-layout {
  height: 100vh;
  display: flex;
  flex-direction: column;
  background: $bg-page;

  &__body {
    flex: 1;
    margin-top: $header-height;
    margin-bottom: $footer-height;
    overflow-y: auto;
    min-height: 0; // allow flex child to scroll
  }
}














@use '../styles/variables' as *;
@use '../styles/mixins' as m;

.workspace-layout {
  height: 100%;
  display: flex;
  overflow: hidden; // sidebar stays put; only content area scrolls
  gap: 0.5rem;
  padding: 0.5rem;
  background: $bg-page;

  @media (min-width: 1900px) {
    gap: 0.75rem;
    padding: 0.75rem;
  }

  &__content {
    flex: 1;
    height: 100%;
    overflow-y: auto;
    background: $bg-main;
    border-radius: $radius;
    outline: 1px solid $border-subtle;
    padding: 1.75rem 2rem 3rem;
    transition: outline-color 0.3s ease, box-shadow 0.3s ease;
    @include m.scrollbar;

    &:hover {
      outline-color: $border-strong;
      box-shadow: $shadow, inset 0 1px 0 rgba(255, 255, 255, 0.04);
    }
  }
}

















@use '../../styles/variables' as *;

.page-placeholder {
  &__title {
    font-size: 1.75rem;
    font-weight: 700;
    color: $text-primary;
    letter-spacing: -0.02em;
  }

  &__subtitle {
    margin-top: 0.375rem;
    color: $text-secondary;
    font-size: 0.96875rem;
  }

  &__box {
    margin-top: 1.5rem;
    border: 1.5px dashed $border-strong;
    border-radius: $radius-lg;
    padding: 3rem 1.5rem;
    text-align: center;
    color: $text-tertiary;
    font-size: 0.9375rem;
    background: $bg-subtle;
  }
}
