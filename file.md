@use '../../../styles/variables' as *;

.reports-page {
  display: flex;
  flex-direction: column;
  height: calc(100vh - 166px);
  min-height: 0;
  gap: 16px;

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

  /* ---------- scrollable body ---------- */
  &__body {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding-right: 4px;
    margin-right: -4px;
  }

  &__list {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 14px;
  }

  /* ---------- report card ---------- */
  &__card {
    display: flex;
    flex-direction: column;
    gap: 12px;
    padding: 22px 24px;
    border: 1px solid $border-subtle;
    border-radius: 14px;
    background: $bg-main;
    box-shadow: $shadow-xs;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &:hover {
      border-color: $border-strong;
      box-shadow: $shadow-sm;
    }
  }

  &__card-head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
  }

  &__date {
    font-family: $font-mono;
    font-size: 0.71875rem;
    font-weight: 600;
    color: $text-tertiary;
  }

  &__actions {
    display: flex;
    align-items: center;
    gap: 6px;
  }

  &__btn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-family: $font-body;
    font-size: 0.75rem;
    font-weight: 600;
    padding: 6px 11px;
    border-radius: 7px;
    border: 1px solid $border-default;
    background: $bg-main;
    color: $text-primary;
    cursor: pointer;
    transition: border-color 0.14s ease;

    &:hover {
      border-color: $text-primary;
    }
  }

  &__icon-btn {
    display: grid;
    place-items: center;
    width: 28px;
    height: 28px;
    border-radius: 7px;
    border: 1px solid $border-default;
    background: $bg-main;
    color: $text-secondary;
    cursor: pointer;
    transition: border-color 0.14s ease, color 0.14s ease;

    &:hover {
      border-color: $text-primary;
      color: $text-primary;
    }
  }

  &__title-text {
    font-size: 1.0625rem;
    font-weight: 700;
    letter-spacing: -0.01em;
    line-height: 1.35;
    color: $text-primary;
  }

  &__summary {
    margin-top: -6px;
    font-size: 0.84375rem;
    line-height: 1.6;
    color: $text-secondary;
  }

  /* ---------- verdict box ---------- */
  &__verdict {
    display: flex;
    gap: 11px;
    padding: 13px 15px;
    background: $primary-light;
    border-radius: 10px;

    strong {
      display: block;
      font-size: 0.75rem;
      font-weight: 700;
      color: $primary;
      margin-bottom: 3px;
    }

    p {
      font-size: 0.8125rem;
      line-height: 1.5;
      color: $text-primary;
    }
  }

  &__verdict-icon {
    flex-shrink: 0;
    width: 30px;
    height: 30px;
    border-radius: 9px;
    background: $bg-main;
    color: $primary;
    display: grid;
    place-items: center;
  }

  /* ---------- footer ---------- */
  &__footer {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
    margin-top: 2px;
    padding-top: 13px;
    border-top: 1px solid $border-subtle;
  }

  &__top-model {
    display: flex;
    align-items: baseline;
    gap: 6px;
  }

  &__top-label {
    font-size: 0.71875rem;
    color: $text-tertiary;
  }

  &__top-value {
    font-size: 0.8125rem;
    font-weight: 700;
    color: $text-primary;
  }

  &__metrics {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
  }

  &__metric-tag {
    font-size: 0.6875rem;
    font-weight: 500;
    color: $text-secondary;
    background: $bg-subtle;
    border: 1px solid $border-subtle;
    border-radius: 6px;
    padding: 3px 8px;
  }

  /* ---------- responsive ---------- */
  @media (max-width: 800px) {
    &__list {
      grid-template-columns: 1fr;
    }
  }
}
