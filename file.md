@use '../../../styles/variables' as *;

.history {
  display: flex;
  flex-direction: column;
  gap: 18px;
  min-height: 0;
  // .main-layout__body / .workspace-layout / .workspace-layout__content form an
  // unbroken flex chain sized to the viewport (header/footer are fixed and offset
  // via margin), so height: 100% here resolves to the visible content area.
  // workspace-layout__content also has a 3rem bottom padding, which would
  // otherwise leave a large gap below .history — pull most of that back in,
  // keeping just a small 0.75rem breathing-room strip at the true bottom edge,
  // and giving the two scrollable panels below a real height to scroll within.
  height: calc(100% + 3rem - 0.75rem);
  margin-bottom: calc(-3rem + 0.75rem);

  /* ---------- header — matches run-eval__header's eyebrow/title/subtitle + meta-pill pattern ---------- */
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

  &__header-right {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 10px;
    margin-bottom: 3px;
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

  /* ---------- generic buttons ---------- */
  &__btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
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
    transition: background 0.12s ease, border-color 0.12s ease, color 0.12s ease, box-shadow 0.12s ease;

    &:hover {
      border-color: $primary;
      box-shadow: $shadow-sm;
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

    &--sm {
      padding: 6px;

      svg {
        width: 15px;
        height: 15px;
      }
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

  /* ---------- default two-column layout: 400px list + detail, each independently scrollable ---------- */
  &__body {
    flex: 1;
    display: flex;
    gap: 16px;
    min-height: 0;
  }

  &__list-panel {
    flex: 0 0 400px;
    max-width: 400px;
    min-height: 0;
    overflow-y: auto;
  }

  /* ---------- list ---------- */
  &__list {
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  &__item {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 10px;
    padding: 14px 16px;
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: $radius-lg;
    cursor: pointer;
    transition: border-color 0.12s ease, box-shadow 0.12s ease, background 0.12s ease;

    &:hover {
      border-color: $primary;
      box-shadow: $shadow-sm;
    }

    &--active {
      border-color: $primary;
      background: $primary-light;
      box-shadow: $shadow-sm;
    }

    // Running evaluations get a thin light that continuously travels
    // around the card's border, rather than a flat pulsing glow.
    &--running {
      position: relative;
      border-color: $border-subtle;

      &::before {
        content: '';
        position: absolute;
        inset: 0;
        border-radius: inherit;
        padding: 1.5px;
        background: conic-gradient(
          from var(--history-angle, 0deg),
          transparent 0deg,
          transparent 265deg,
          rgba($primary, 0.9) 300deg,
          $primary 320deg,
          rgba($primary, 0.9) 340deg,
          transparent 360deg
        );
        -webkit-mask: linear-gradient(#fff 0 0) content-box, linear-gradient(#fff 0 0);
        -webkit-mask-composite: xor;
        mask-composite: exclude;
        animation: history-border-travel 2.6s linear infinite;
        pointer-events: none;
      }

      &.history__item--active {
        border-color: $primary;
        background: $primary-light;
        box-shadow: $shadow-sm;
      }
    }
  }

  @property --history-angle {
    syntax: '<angle>';
    inherits: false;
    initial-value: 0deg;
  }

  @keyframes history-border-travel {
    to {
      --history-angle: 360deg;
    }
  }

  &__icon {
    width: 36px;
    height: 36px;
    background: $primary-light;
    border-radius: $radius-md;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-shrink: 0;

    svg {
      width: 18px;
      height: 18px;
      color: $primary;
      stroke-width: 1.5;
    }
  }

  &__item--active &__icon {
    background: $bg-main;
  }

  &__content {
    flex-basis: calc(100% - 46px);
    flex-grow: 1;
    min-width: 0;

    h4 {
      font-size: 14px;
      font-weight: 500;
      margin-bottom: 4px;
      color: $text-primary;
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
    }
  }

  &__meta {
    display: flex;
    align-items: center;
    gap: 10px;
    font-size: 12px;
    color: $text-tertiary;
    overflow: hidden;

    span:last-child {
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
    }
  }

  &__type {
    flex-shrink: 0;
    padding: 2px 8px;
    background: $primary-light;
    color: $primary;
    border-radius: 4px;
    font-weight: 500;
  }

  &__item--active &__type {
    background: $bg-main;
  }

  /* ---------- status badge (list row + detail header) ---------- */
  &__status-badge {
    flex-shrink: 0;
    font-size: 0.625rem;
    font-weight: 700;
    letter-spacing: 0.02em;
    border-radius: 999px;
    padding: 2px 8px;
    white-space: nowrap;

    &--green {
      color: $success;
      background: $success-subtle;
    }

    &--blue {
      color: $primary;
      background: $primary-light;
    }

    &--amber {
      color: $warning;
      background: $warning-subtle;
    }

    &--violet {
      color: $violet;
      background: $violet-light;
    }

    &--danger {
      color: $danger;
      background: $danger-subtle;
    }
  }

  &__item--active &__status-badge--green,
  &__item--active &__status-badge--blue,
  &__item--active &__status-badge--amber,
  &__item--active &__status-badge--violet,
  &__item--active &__status-badge--danger {
    background: $bg-main;
  }

  &__status-badge--live {
    display: inline-flex;
    align-items: center;
    gap: 4px;
  }

  &__status-dot {
    flex-shrink: 0;
    width: 6px;
    height: 6px;
    border-radius: 50%;
    background: currentColor;
    animation: history-dot-pulse 1.4s ease-in-out infinite;
  }

  @keyframes history-dot-pulse {
    0%,
    100% {
      opacity: 1;
      transform: scale(1);
    }
    50% {
      opacity: 0.35;
      transform: scale(0.7);
    }
  }

  &__detail-date &__status-badge {
    margin-right: 6px;
    vertical-align: 1px;
  }

  &__results {
    display: flex;
    gap: 16px;
    flex-shrink: 0;
  }

  &__stat {
    display: flex;
    flex-direction: column;
  }

  &__stat-label {
    font-size: 10px;
    text-transform: uppercase;
    letter-spacing: 0.4px;
    color: $text-tertiary;
    margin-bottom: 2px;
  }

  &__stat-value {
    font-size: 13px;
    font-weight: 500;
    color: $text-primary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    max-width: 84px;

    &--highlight {
      color: $primary;
    }
  }

  &__actions {
    display: flex;
    gap: 6px;
    flex-shrink: 0;
    margin-left: auto;
  }

  /* ---------- detail panel ---------- */
  &__detail-panel {
    flex: 1;
    min-width: 0;
    min-height: 0;
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 16px;
    box-shadow: $shadow-xs;
    padding: 26px 28px;
    overflow-y: auto;
  }

  &__detail-head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
    padding-bottom: 18px;
    margin-bottom: 22px;
    border-bottom: 1px solid $border-subtle;
  }

  &__detail-head-left {
    display: flex;
    flex-direction: column;
    gap: 8px;
    min-width: 0;
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

  /* ---------- type badge (sidebar + detail) ---------- */
  &__type-badge {
    width: fit-content;
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

  /* ---------- summary cards — matches reference .results-summary-cards / .summary-card ---------- */
  &__summary-cards {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 16px;
    margin-bottom: 24px;
  }

  &__summary-card {
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: $radius-lg;
    padding: 20px;
    display: flex;
    gap: 14px;
    min-width: 0;

    &--winner {
      // Reference uses a fixed warm amber gradient/border for the winner
      // card in both themes (it's a celebratory accent, not a surface
      // token), so this intentionally does not switch with dark mode.
      background: linear-gradient(135deg, #fffbeb 0%, #fef3c7 100%);
      border-color: #fde68a;
    }
  }

  &__summary-icon {
    width: 44px;
    height: 44px;
    background: $bg-subtle;
    border-radius: $radius-md;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-shrink: 0;

    svg {
      width: 22px;
      height: 22px;
      color: $text-secondary;
      stroke-width: 1.5;
    }
  }

  &__summary-card--winner &__summary-icon {
    background: rgba(180, 83, 9, 0.12);

    svg {
      color: #b45309;
    }
  }

  &__summary-content {
    min-width: 0;
  }

  &__summary-label {
    font-size: 11px;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    color: $text-tertiary;
    margin-bottom: 4px;
  }

  &__summary-value {
    font-size: 16px;
    font-weight: 600;
    color: $text-primary;
    margin-bottom: 2px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__summary-score {
    font-size: 13px;
    color: $text-secondary;
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

  /* ---------- loading spinner (used by History.tsx's Loader2 icons) ---------- */
  &__spin {
    animation: history-spin 0.8s linear infinite;
  }

  @keyframes history-spin {
    to {
      transform: rotate(360deg);
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
    height: auto;
    margin-bottom: 0;

    &__header {
      flex-direction: column;
      align-items: flex-start;
    }

    &__header-right {
      margin-bottom: 0;
    }

    &__body {
      flex-direction: column;
    }

    &__list-panel {
      flex: 0 0 auto;
      max-width: none;
      max-height: 16rem;
    }

    &__detail-panel {
      max-height: none;
    }

    &__summary-cards {
      grid-template-columns: 1fr;
    }
  }

  @media (max-width: 520px) {
    &__header-right {
      width: 100%;
      justify-content: space-between;
    }

    &__detail-panel {
      padding: 18px 16px;
    }

    &__detail-head {
      flex-direction: column;
    }
  }
}
