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
    bottom: 34px;
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
