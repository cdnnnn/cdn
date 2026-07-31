@use '../../../styles/variables' as *;

.history {
  display: flex;
  flex-direction: column;
  gap: 16px;

  /* ---------- header ---------- */
  &__header {
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding-bottom: 18px;
    margin-bottom: 2px;
    border-bottom: 1px solid $border-subtle;
  }

  &__eyebrow {
    display: flex;
    align-items: center;
    gap: 8px;
    font-family: $font-mono;
    font-size: 0.6875rem;
    font-weight: 700;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    color: $primary;
    margin: 0 0 6px;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $primary;
    }
  }

  &__title {
    margin: 0;
    font-size: 21px;
    font-weight: 800;
    letter-spacing: -0.02em;
    color: $text-primary;
  }

  &__subtitle {
    margin: 3px 0 0;
    font-size: 0.84375rem;
    color: $text-secondary;
  }

  &__new-btn {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    font-family: $font-body;
    font-size: 0.8125rem;
    font-weight: 600;
    padding: 9px 14px;
    border-radius: 8px;
    border: 1px solid $primary;
    background: $primary;
    color: #fff;
    cursor: pointer;
    white-space: nowrap;
    transition: background 0.14s ease, border-color 0.14s ease;

    &:hover {
      background: $primary-hover;
      border-color: $primary-hover;
    }
  }

  &__split {
    display: grid;
    grid-template-columns: 340px 1fr;
    border: 1px solid $border-default;
    border-radius: 14px;
    overflow: hidden;
    background: $bg-main;
    min-height: 560px;
  }

  /* ---------- left list ---------- */
  &__list {
    border-right: 1px solid $border-default;
    background: $bg-subtle;
    display: flex;
    flex-direction: column;
    min-height: 0;
  }

  &__search {
    display: flex;
    align-items: center;
    gap: 9px;
    padding: 12px 14px;
    border-bottom: 1px solid $border-default;
    color: $text-tertiary;

    input {
      flex: 1;
      border: none;
      background: transparent;
      font-family: $font-body;
      font-size: 0.8125rem;
      color: $text-primary;
      outline: none;

      &::placeholder {
        color: $text-tertiary;
      }
    }
  }

  &__rows {
    overflow-y: auto;
    flex: 1;
  }

  &__row {
    width: 100%;
    display: flex;
    flex-direction: column;
    gap: 4px;
    text-align: left;
    padding: 13px 16px;
    border: none;
    border-bottom: 1px solid $border-subtle;
    background: transparent;
    cursor: pointer;

    &:hover {
      background: $bg-inset;
    }

    &--active {
      background: $bg-main;
      border-left: 3px solid $primary;
      padding-left: 13px;

      &:hover {
        background: $bg-main;
      }
    }
  }

  &__row-name {
    font-size: 0.8125rem;
    font-weight: 700;
    letter-spacing: -0.01em;
    color: $text-primary;
  }

  &__row-meta {
    display: flex;
    align-items: center;
    justify-content: space-between;
    font-size: 0.6875rem;
    color: $text-tertiary;
  }

  &__row-type {
    background: $bg-inset;
    border-radius: 999px;
    padding: 2px 8px;
    font-weight: 600;
  }

  /* ---------- right detail ---------- */
  &__detail {
    padding: 24px 28px;
    overflow-y: auto;
    min-height: 0;
  }

  &__detail-head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
    margin-bottom: 18px;
  }

  &__detail-title {
    margin: 0 0 4px;
    font-size: 0.9375rem;
    font-weight: 700;
    letter-spacing: -0.01em;
    color: $text-primary;
  }

  &__detail-sub {
    margin: 0;
    font-size: 0.75rem;
    color: $text-tertiary;
  }

  &__status-badge {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 4px 10px;
    border-radius: 999px;
    background: $success-subtle;
    color: $success;
    font-size: 0.75rem;
    font-weight: 600;
    white-space: nowrap;
  }

  &__stats {
    display: flex;
    border: 1px solid $border-default;
    border-radius: 10px;
    overflow: hidden;
    margin-bottom: 22px;
  }

  &__stat {
    flex: 1;
    padding: 12px 16px;
    border-right: 1px solid $border-subtle;
    display: flex;
    flex-direction: column;
    gap: 2px;

    &:last-child {
      border-right: none;
    }
  }

  &__stat-value {
    font-size: 1.25rem;
    font-weight: 800;
    letter-spacing: -0.02em;
    color: $text-primary;
    line-height: 1;
  }

  &__stat-label {
    font-size: 0.625rem;
    font-weight: 600;
    letter-spacing: 0.05em;
    text-transform: uppercase;
    color: $text-tertiary;
    margin-top: 2px;
  }

  /* ---------- results table ---------- */
  &__table-wrap {
    overflow-x: auto;
  }

  &__table {
    width: 100%;
    border-collapse: collapse;
    font-size: 0.8125rem;

    th {
      text-align: left;
      padding: 9px 12px;
      color: $text-tertiary;
      font-size: 0.6875rem;
      font-weight: 600;
      text-transform: uppercase;
      letter-spacing: 0.05em;
      border-bottom: 1px solid $border-default;
    }

    td {
      padding: 10px 12px;
      border-bottom: 1px solid $border-subtle;
      color: $text-secondary;
    }

    tr:last-child td {
      border-bottom: none;
    }
  }

  &__row-winner {
    background: $primary-light;
  }

  &__cell-strong {
    font-weight: 700;
    color: $text-primary;
  }

  &__score {
    font-weight: 700;
    color: $primary;
  }

  &__rank {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 22px;
    height: 22px;
    border-radius: 50%;
    background: $bg-inset;
    color: $text-secondary;
    font-size: 0.71875rem;
    font-weight: 700;

    &--1 {
      background: $primary;
      color: #fff;
    }
  }

  &__empty {
    padding: 24px;
    text-align: center;
    color: $text-tertiary;
    font-size: 0.8125rem;
  }
}
